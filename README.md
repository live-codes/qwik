# Qwik in the browser — proof of concept

A single, dependency-free HTML page that compiles and runs a multi-file Qwik project
entirely in the browser. This is step 1 of adding Qwik support to LiveCodes; it exists to
prove out the compile + link + run pipeline before it gets wired into the playground.

**Open `index.html` in a browser.** That's it — no build, no install, no server. (It works
straight off `file://`; the only network access is to the jsDelivr CDN.)

Everything is pinned to `@builder.io/qwik@1.15.0`.

## What it does

- Loads the Qwik **optimizer** (Rust, compiled to WebAssembly) from the CDN
- Compiles the whole project — including the `$` transforms (`component$`, `onClick$`, ...)
- **Links the project's files together**, rewriting imports between them so no server is needed
- Runs the linked entry in a sandboxed `<iframe>`, keeping Qwik's lazy chunk loading
- Forwards `console.*` and runtime errors from the frame back to the page

The editor holds a small project (`src/app.tsx`, `src/counter.tsx`, `src/components/label.tsx`).
Add, rename, remove and pick the entry file from the list above the editor; `ctrl/cmd + enter`
re-runs. The **Compiled JS** tab shows every module the optimizer emitted and, for each one,
what its relative imports resolved to.

Verified working: imports across directories, explicit import extensions, plain `.ts`
non-component modules, shared components, per-instance signal state, lazy chunk loading at
runtime, and compile-error reporting.

## Pipeline

```
project files (TSX/TS)
   │  optimizer.transform_modules()            ← wasm, runs in-browser
   ▼
N modules + M lazily-imported chunks           ← optimizer never bundles
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

## Findings worth knowing

The non-obvious things that cost time, in rough order of how much they cost:

1. **`mode` defaults to `"lib"`, which silently skips the `$` transform.** With the default,
   `component$`/`onClick$` come out completely untouched — no error, no warning, just a
   module that does nothing useful. Pass `mode: "prod"` (`EmitMode` is
   `'dev' | 'prod' | 'lib'`). This is set as the default inside the optimizer's JS wrapper's
   `convertOptions()`, so it is easy to miss.

2. **The optimizer never bundles.** With N input files it returns N modules (plus one module
   per `$` closure under the `smart`/`single` entry strategies) and **leaves relative imports
   between your files untouched**. There is no flag that makes it produce a single bundled
   output — so a resolver is unavoidable for multi-file projects.

   Worth knowing how the strategies differ here:

   | entry strategy | user files | `$` closures | relative imports to resolve |
   | --- | --- | --- | --- |
   | `inline` | one module each | inlined into the parent | yours only |
   | `single` / `smart` | one module each | own module, `import()`ed | yours **and** the chunks |

   We use `smart` (Qwik's default, what a real build uses) because the chunks stay
   dynamically imported, so **lazy loading survives** — the linked output still `import()`s
   them, just from data URLs. Note the optimizer also *copies* your relative imports into the
   generated chunk modules that use them, so the same specifier gets resolved more than once.

3. **`bindings/*` is not a package export.** `@builder.io/qwik/bindings/qwik.wasm.mjs` is in
   the published tarball but absent from the `exports` map, so it cannot be imported as a
   subpath — by URL only. For the same reason the raw wasm binding is called directly
   (`binding.transform_modules(...)`) instead of `createOptimizer()`, which skips the
   optimizer's Node/Deno/Bun platform detection entirely.

4. **`data:` URLs are the only portable way to resolve the modules — not blob URLs.**
   LiveCodes compiles in the app's origin and renders in a *different* origin (the result
   page), so a compiler cannot mint a blob URL the result page can fetch; `isBare()` in
   `compiler/import-map.ts` excludes `blob:` for exactly this reason. `data:` URLs work
   everywhere, and this is already the established LiveCodes mechanism — `replaceSFCImports`
   inlines nested Vue/Svelte components the same way.

   A pleasant consequence: because nothing needs a blob URL, the result iframe no longer needs
   `allow-same-origin`. The sandbox is now `allow-scripts allow-modals allow-popups allow-forms`
   only, so user code cannot reach the host page.

5. **A parent embeds its children, so links must be built bottom-up — which means cycles
   cannot be represented.** The linker walks from the entry and rewrites each relative
   specifier into the data URL of the module it resolves to; a module that is still being
   linked is a cycle, and is reported with the full chain. Cycles are genuinely possible in a
   component graph (even a single mutual import), so this is a real constraint, not a
   theoretical one.

   It also means base64 nesting grows ~1.33× per level of import depth. Fine for a playground;
   worth watching if deep graphs ever show up.

6. **Resolution needs a few guesses, because the optimizer renames files.** `.tsx` inputs come
   out as `.js`, so `./counter`, `./counter.tsx` and `./counter.js` must all land on
   `src/counter.js`. Try exact, then `.js`/`.jsx`/`.ts`/`.tsx`, then a source extension
   swapped for `.js`, then `/index.*`.

7. **`entryStrategy: "inline"` is not needed for "no server".** It was the original choice
   because it avoids relative chunk imports, but the linker handles those anyway — so the
   strategy is now a laziness trade-off rather than a feasibility one.

8. **The runtime needs exactly four import-map entries**, and two of them must be the *same*
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

9. **`render()` cannot take a bare component.** `render(el, Component)` hits
   `jsx(Component, null)`, and Qwik's `jsx` reads `props.children` eagerly → *"Cannot read
   properties of null (reading 'children')"*. Use `_jsxC(Component, null, 3, key)`, which is
   exactly what the optimizer emits for `<Component />`.

10. **`qwikloader.js` is required, or nothing is interactive.** Qwik does not attach
    per-element DOM listeners. The renderer stores handlers on the element context
    (`element._qc_.li`) and pushes used event names onto `window.qwikevents`; a single global
    delegate — `dist/qwikloader.js` — dispatches to them. Without that script the page renders
    perfectly and ignores every click.

11. **The optimizer reports errors instead of throwing.** It returns `diagnostics` with
    `category: 'error' | 'warning' | 'sourceError'`, a `file`, and `highlights[].startLine/startCol`.
    On a parse error, `modules` comes back empty. Note `isEntry` is never populated — with
    `inline` there is exactly one module, so use `modules[0]`.

12. **The compiled module keeps the user's original imports.** `import { component$, useSignal }
    from '@builder.io/qwik'` survives the transform (dead but still linked), so the runtime
    mapped into the import map must actually export those names. It does — `core.prod.mjs` is
    self-contained apart from `build` and `preloader`, and exports the internal helpers the
    compiler emits (`componentQrl`, `inlinedQrl`, `useLexicalScope`, `_fnSignal`, `_jsxQ`,
    `_jsxC`, ...).

## Known limitations

- **Import cycles are rejected.** See finding 5. The linker reports the cycle rather than
  guessing.
- **Bare imports other than the Qwik runtime fail**, with a message pointing at the file that
  asked for them. There is no CDN fallback here (that is LiveCodes' `createImportMap`).
- **No `@builder.io/qwik-city`**, no router, no SSR, no resumability — client rendering only.
  Resumability needs a server-rendered document, which is out of scope.
- **Specifier rewriting is regex-based**, matching LiveCodes' own `importsPattern` approach.
  A module specifier lookalike inside a string literal or comment would be rewritten too.
- **Diagnostics do not name the file.** The optimizer's diagnostics carry a `file`, which the
  error reporter currently drops; worth adding.
- **`_jsxC`, the `bindings/` URL and the diagnostics shape are version-coupled** to the
  pinned Qwik version.

## Path to LiveCodes

The compile and link halves map onto LiveCodes almost one-to-one:

| PoC | LiveCodes |
| --- | --- |
| `compile()` | `src/livecodes/languages/qwik/lang-qwik-compiler.ts` |
| `linkModules()` | the `replaceSFCImports` pattern in `compiler/import-map.ts` |
| `importMap` + qwikloader + iframe | result-page rendering (`src/livecodes/result/`), which already injects an import map |
| the editor / console / file UI | throw away — the playground already provides both |

Concretely:

1. **`src/livecodes/languages/qwik/`** — `lang-qwik.ts` (a `LanguageSpecs`: `name: 'qwik'`,
   compiler factory, `extensions`, `editor: 'script'`, `editorLanguage: 'typescript'`,
   `editorSupport.compilerOptions` with `jsx: preserve` + `jsxImportSource: '@builder.io/qwik'`),
   `lang-qwik-compiler.ts`, `index.ts`; register it in `languages/languages.ts`.
2. **Compiler bundle.** Either bundle the wasm glue into
   [`@live-codes/browser-compilers`](https://github.com/live-codes/browser-compilers) (the way
   Astro's compiler is published as `@hatemhosny/astro-internal`), or keep the 275 KB wrapper
   tiny and fetch the 3.4 MB `.wasm` from the CDN — the latter is closer to what this PoC does.
   Reference anything CDN-hosted from `src/livecodes/vendors.ts`.
3. **Runtime.** Map the four `@builder.io/qwik*` specifiers in the result page's import map and
   inject `qwikloader.js` there. This is a result-page concern, not a compiler concern.
4. **Standard checklist** from
   [`docs/docs/contribution/adding-languages.mdx`](../livecodes/docs/docs/contribution/adding-languages.mdx):
   `sdk/models.ts` names/aliases, Monaco/CodeMirror/Prism support, `language-info.html`,
   starter template, e2e tests, docs, vendor licenses.
5. **The compiler can run in the worker** — the wasm optimizer works fine off the main thread.

### The one thing that is *not* a drop-in

**LiveCodes has no multi-file model.** `ContentConfig` has exactly four slots — `markup`,
`style`, `script`, `tests` (`src/sdk/models.ts`) — and there is no virtual filesystem: a
repo-wide search for a `files` field finds nothing, and relative specifiers between plain
JS/TS files are deliberately dropped by `createImportMap()` and survive to runtime unresolved.

The linker above is small and lifts cleanly, but *exposing several files to the user* is a
LiveCodes architecture change, not a language-adaptor change. Three options:

- **Ship Qwik single-file** and let the linker handle only `$` chunks. Smallest step, but it
  gives up the multi-file authoring that makes a component framework pleasant.
- **Add a files model** to `ContentConfig` (e.g. `files?: Record<string, string>`) and feed the
  compiler a file map. This is what the PoC assumes, and it would benefit every framework,
  not just Qwik — but it touches config, storage, UI, import/export and the SDK.
- **Lean on `config.imports`**, which already maps specifiers to URLs, letting users pull in
  other modules by URL. No new architecture, but not really "multiple files in a project".

That choice is a product decision worth making before the Qwik adaptor is written, since it
determines whether `lang-qwik-compiler.ts` takes one string or a file map.
