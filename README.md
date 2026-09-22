# Qwik in the browser — proof of concept

A single, dependency-free HTML page that compiles and runs a multi-file Qwik project
entirely in the browser.

**Open `index.html` in a browser.** That's it — no build, no install, no server. (It works
straight off `file://`; the only network access is to the jsDelivr CDN.)

Everything is pinned to `@builder.io/qwik@1.15.0`.

> **Status: proof of concept, deliberately outside the LiveCodes codebase.** Its job was to
> find out where Qwik's compiler and runtime differ from the frameworks LiveCodes already
> supports. It is not intended to be integrated as-is — see
> [Part 4](#part-4-what-this-means-for-a-qwik-adaptor) for what changed now that
> [multi-file support](https://github.com/live-codes/livecodes/pull/934) exists on the
> `multi-file-support` branch.

## What it does

- Loads the Qwik **optimizer** (Rust, compiled to WebAssembly) from the CDN
- Compiles the whole project — including the `$` transforms (`component$`, `onClick$`, ...)
- **Links the project's files together**, rewriting imports between them so no server is needed
- Runs the linked entry in a sandboxed `<iframe>`
- Forwards `console.*` and runtime errors from the frame back to the page

The editor holds a small project (`src/app.tsx`, `src/counter.tsx`, `src/components/label.tsx`).
Add, rename, remove and pick the entry file from the list above the editor; `ctrl/cmd + enter`
re-runs. The **Compiled JS** tab shows every module the optimizer emitted and, for each one,
what its relative imports resolved to.

Verified working: imports across directories, explicit import extensions, plain `.ts`
non-component modules, shared components, per-instance signal state, lazy chunk loading at
runtime, non-ASCII source, and compile-error reporting for unresolved, bare and circular
imports.

## Pipeline

```
project files (TSX/TS)
   │  optimizer.transform_modules()            ← wasm, runs in-browser
   ▼
N modules + M lazily-imported chunks           ← the optimizer never bundles
   │  link from the entry: every "./x" → data: URL of the module it resolves to
   ▼
one self-contained entry module
   │  import(<entry data url>) from <script type="module">
   ▼
import map resolves "@builder.io/qwik*" → CDN runtime
   │  _jsxC(App, null, 3, key) → render()
   ▼
DOM in the iframe   ← qwikloader.js dispatches events
```

---

## Findings

### Part 1: Qwik's optimizer and runtime

The non-obvious things that cost the most time, in rough order:

1. **`mode` defaults to `"lib"`, which silently skips the `$` transform.** With the default,
   `component$`/`onClick$` come out completely untouched — no error, no warning, just a module
   that does nothing useful. Pass `mode: "prod"` (`EmitMode` is `'dev' | 'prod' | 'lib'`). That
   default lives inside the optimizer JS wrapper's `convertOptions()`, so it is easy to miss.

2. **`bindings/*` is not a package export.** `@builder.io/qwik/bindings/qwik.wasm.mjs` ships in
   the tarball but is absent from the `exports` map, so it cannot be imported as a subpath — by
   URL only. For the same reason the raw wasm binding is called directly
   (`binding.transform_modules(...)`) instead of `createOptimizer()`, which skips the
   optimizer's Node/Deno/Bun platform detection entirely.

3. **The runtime needs exactly four import-map entries**, and two of them must be the *same*
   URL:

   ```
   @builder.io/qwik              → dist/core.prod.mjs
   @builder.io/qwik/jsx-runtime  → dist/core.prod.mjs   ← same file, one module instance
   @builder.io/qwik/build        → dist/build/index.prod.mjs
   @builder.io/qwik/preloader    → dist/preloader.mjs
   ```

   The package maps `jsx-runtime` to `core`'s file, so pointing both at one URL guarantees a
   single runtime instance — Qwik keeps module-level state (signals, container state) that
   breaks if it is duplicated.

4. **`render()` cannot take a bare component.** `render(el, Component)` hits
   `jsx(Component, null)`, and Qwik's `jsx` reads `props.children` eagerly → *"Cannot read
   properties of null (reading 'children')"*. Use `_jsxC(Component, null, 3, key)`, which is
   exactly what the optimizer emits for `<Component />`.

   This only bites when you render a module's `default` export from *outside* it — as this PoC
   does. Inside a compiled file, writing `<App />` is compiled to `_jsxC(App, null, 3, key)`
   for you. See [Part 4](#the-renderel-component-pitfall-disappears).

5. **`qwikloader.js` is required, or nothing is interactive.** Qwik does not attach per-element
   DOM listeners. The renderer stores handlers on the element context (`element._qc_.li`) and
   pushes used event names onto `window.qwikevents`; a single global delegate —
   `dist/qwikloader.js` — dispatches to them. Without that script the page renders perfectly and
   ignores every click.

6. **The optimizer reports errors instead of throwing.** It returns `diagnostics` with
   `category: 'error' | 'warning' | 'sourceError'`, a `file`, and
   `highlights[].startLine/startCol`. On a parse error, `modules` comes back empty.

7. **`isEntry` is never populated.** Every module has `isEntry: false`, even for the file passed
   as the entry. Don't use it to find the entry module.

8. **The compiled module keeps the user's original imports.** `import { component$, useSignal }
   from '@builder.io/qwik'` survives the transform (dead but still linked), so the runtime
   mapped into the import map must actually export those names. It does — `core.prod.mjs` is
   self-contained apart from `build` and `preloader`, and exports the internal helpers the
   compiler emits (`componentQrl`, `inlinedQrl`, `useLexicalScope`, `_fnSignal`, `_jsxQ`,
   `_jsxC`, ...).

9. **Output is version-coupled to the runtime.** The emitted code imports underscore-prefixed
   internals, so the optimizer and the runtime build must be the same version. A version bump
   moves both, and `_jsxC` in particular is an internal API.

### Part 2: Multi-file, as this PoC does it

**The optimizer never bundles.** That single fact shapes everything else. With N input files it
returns N modules (plus, under `smart`/`single`, one module per `$` closure) and **leaves
relative imports between your files untouched**. No flag produces a single bundled output, so a
resolver is unavoidable.

| entry strategy | user files | `$` closures | relative imports to resolve |
| --- | --- | --- | --- |
| `inline` | one module each | inlined into the parent | yours only |
| `single` / `smart` | one module each | own module, `import()`ed | yours **and** the chunks |

This PoC's linker (`linkModules`) walks the module graph from the entry and rewrites each
relative specifier into the data URL of the module it resolves to. Two consequences fall out of
doing it that way:

- **A parent embeds its children, so links must be built bottom-up — which means cycles cannot
  be represented.** A module that is still being linked is a cycle; the linker reports the full
  chain rather than emitting something broken. Cycles are genuinely possible in a component
  graph (even a single mutual import).
- **Base64 nesting grows ~1.33× per level of import depth**, because each child's data URL is
  embedded verbatim in its parent's.

Both are properties of *this* linker, not of the approach — see Part 3.

Resolution needs a few guesses, because the optimizer renames files: `.tsx` inputs come out as
`.js`, so `./counter`, `./counter.tsx` and `./counter.js` must all land on `src/counter.js`. Try
exact, then `.js`/`.jsx`/`.ts`/`.tsx`, then a source extension swapped for `.js`, then
`/index.*`.

### Part 3: How LiveCodes solves the same problem

Now that [`multi-file-support`](https://github.com/live-codes/livecodes/pull/934) exists, this
is the approach that matters. Same two ingredients (import maps + data URLs), but wired **flat**
rather than recursively, in `src/livecodes/result/multi-file-result-page.ts`:

1. Every script file's compiled output becomes a data URL, cached in `fileUrls` so each file is
   base64-encoded **exactly once**.
2. Relative specifiers are rewritten to a *bare* specifier, `./x` → `~/x`:

   > `// importmaps cannot override relative imports in data URLs`
   > — `multi-file-result-page.ts:343`

   This is the crux. Relative specifiers resolve against the referrer, and a data URL has no
   meaningful base — so a relative import inside a data URL is simply unresolvable. `~/x` is a
   bare specifier, and bare specifiers *can* be remapped by an import map.
3. All the `~/x` → data URL pairs go into the result page's import map, which is injected into
   `<head>` before anything is imported.

Because the mapping is flat, the shape of the graph stops mattering:

| | this PoC (recursive inline) | LiveCodes (flat remap) |
| --- | --- | --- |
| cycles | impossible | fine |
| base64 nesting | ~1.33× per depth | each file encoded once |
| ordering constraint | bottom-up | none |
| relative specifier in a data URL | pre-rewritten by the parent | `~/x` bare specifier |

The flat design is strictly better, and it is why the "cycles cannot be represented" limitation
in Part 2 is **not** a constraint on multi-file Qwik in general — it is a constraint on how this
PoC chose to link. Should the PoC be picked up again, switching it to the flat approach is a
small change (rewrite `./x` → `~/x`, build one import map, encode each module once) and would
remove the cycle limitation outright.

### Part 4: What this means for a Qwik adaptor

#### The entry strategy is forced to `inline`

LiveCodes compiles **one file at a time and stores exactly one output module per file**
(`core.ts:1652` — `compiledFiles.push({ ...file, compiled: compileResult.code })`). I
reproduced that per-file call to see what each strategy emits:

| strategy | modules emitted per file | relative specifiers out |
| --- | --- | --- |
| `inline` | **1** | only the user's own (`./counter`, `./components/label.tsx`) |
| `smart` | 2–3 | `./counter.tsx_Counter_component_9bHJINVHHaU`, … |

Under `smart` the chunk modules are (a) not in `config.files`, so `findFile()` never finds them,
and (b) referenced by *relative* specifiers, which import maps cannot override. So `inline` —
with its clean 1:1 file→module mapping — is the only strategy that fits as-is. `inline` also
makes `isEntry` and the emitted module path irrelevant, since only `compileResult.code` is used.

**The cost: Qwik loses lazy loading.** `inline` collapses every `$` closure into its file, so
`onClick$` handlers ship eagerly instead of in a lazily-fetched chunk. It will not behave like a
Vite build. That is consistent with the deliberate "no rollup/rolldown" decision, but it is the
one Qwik feature that cannot survive — worth deciding consciously, since resumability is Qwik's
whole pitch. (Resumability proper already needs SSR, which is out of scope regardless.)

If laziness is wanted later there is a supported escape hatch: use `smart`, have the compiler
rewrite its own chunk specifiers to `~/…`, and publish the chunk data URLs through
`CompileInfo.imports`. That works because `compileInfo.imports` is spread last inside
`userImports` (`multi-file-result-page.ts:640`), so it overrides the bogus CDN entry
`createImportMap` would otherwise generate for a `~/…` specifier — and `compileInfo` is threaded
through and merged across files (`core.ts:1650,1653`). Feasible, but it means the compiler
reimplementing part of the result page's job.

#### The `render(el, Component)` pitfall disappears

The `_jsxC` workaround in Part 1 finding 4 is an artifact of this PoC rendering `mod.default`
from outside the module. In a Vite-like project the entry is an ordinary `.tsx` file, and the
optimizer compiles `<App />` into `_jsxC(App, null, 3, key)` itself. An adaptor needs no such
trick — it just compiles the file.

#### The hooks needed already exist

No result-page changes are required; the same extension points Vue and Svelte use are enough:

- **`compiler.imports`** — map the four `@builder.io/qwik*` specifiers (same shape as
  `lang-vue.ts:29-32`). Two must be the *same* URL, per Part 1 finding 3. This also overrides
  the CDN fallback: `createImportMap` would otherwise send `@builder.io/qwik` to esm.sh and
  produce a second, incompatible runtime instance. `compilerImports` is spread after
  `userImports` (`multi-file-result-page.ts:676`), so it wins.
- **`compiler.scripts`** — inject `qwikloader.js` (Part 1 finding 5). `deferScripts` exists if
  load order calls for it.
- **`options.filename`** — arrives per compile (`core.ts:1649`, `CompileOptions.filename`), so
  the adaptor can pass the real path to the optimizer as `path` and get `.ts` vs `.tsx` parsing
  right.

A Vite-shaped template would be `mainFile: 'index.html'` with
`<script type="module" src="src/entry.tsx">`, following the same `Template` shape as
`templates/multifile/solid.ts`.

#### Two smaller decisions

- **Naming.** Follow the `solid.tsx` precedent and add the language as e.g. `qwik.tsx` rather
  than reusing `.tsx`. `files[].language` is authoritative per file, but `fileLanguages` is a
  project-wide extension→language override — so `{ tsx: 'qwik' }` would clobber React `.tsx`
  files and foreclose the "multiple frameworks in one project" use case.
- **Compiler delivery.** `bindings/*` is not exported, so the wasm glue is fetch-by-URL only
  (Part 1 finding 2): either bundle it into
  [`@live-codes/browser-compilers`](https://github.com/live-codes/browser-compilers) the way
  Astro's compiler is published, or keep the 275 KB wrapper thin and pull the 3.4 MB `.wasm`
  from a CDN. Either way it must reference the same pinned version as `compiler.imports`
  (Part 1 finding 9).

---

## PoC limitations

- **Import cycles are rejected.** A property of this linker's recursive inlining; LiveCodes'
  flat approach does not have it (Part 3).
- **Bare imports other than the Qwik runtime fail**, with a message pointing at the file that
  asked for them. There is no CDN fallback here (that is LiveCodes' `createImportMap`).
- **No `@builder.io/qwik-city`**, no router, no SSR, no resumability — client rendering only.
- **Specifier rewriting is regex-based**, matching LiveCodes' own `importsPattern` approach. A
  specifier lookalike inside a string literal or comment would be rewritten too.
- **Compile diagnostics do not name the file.** The optimizer's diagnostics carry a `file`,
  which the error reporter drops.
- **No dedicated result-page origin.** The iframe has no `allow-same-origin` (good), but there
  is no separate origin or postMessage protocol behind it as in LiveCodes.

## If this PoC is picked up again

In rough order of value:

1. **Switch the linker to the flat approach** (Part 3). Removes the cycle limitation and the
   base64 nesting, and matches what LiveCodes actually does.
2. **Report the `file` field on compile diagnostics.**
3. **Try `smart` + `CompileInfo.imports`** to see whether lazy loading can be kept.

Anything beyond that is integration work, which stays out of scope while this is a PoC.
