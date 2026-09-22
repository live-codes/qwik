# Qwik in the browser — proof of concept

A single, dependency-free HTML page that compiles and runs Qwik code entirely in the
browser. This is step 1 of adding Qwik support to LiveCodes; it exists to prove out the
compile + run pipeline before it gets wired into the playground.

**Open `index.html` in a browser.** That's it — no build, no install, no server. (It works
straight off `file://`; the only network access is to the jsDelivr CDN.)

Everything is pinned to `@builder.io/qwik@1.15.0`.

## What it does

- Loads the Qwik **optimizer** (Rust, compiled to WebAssembly) from the CDN
- Compiles the TSX in the editor — including the `$` transforms (`component$`, `onClick$`, ...)
- Runs the compiled module against the Qwik runtime in a sandboxed `<iframe>`
- Forwards `console.*` and runtime errors from the frame back to the page

Verified working: signals, `useStore`, child components with props, `onInput$`, list
rendering, compile-error reporting, and re-running after edits.

## Pipeline

```
editor (TSX)
   │  optimizer.transform_modules()          ← wasm, runs in-browser
   ▼
single ES module (entryStrategy: "inline")
   │  blob URL → <script type="module"> in the result iframe
   ▼
import map resolves "@builder.io/qwik" → CDN runtime
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

2. **`entryStrategy: "inline"` is what makes "no server" possible.** The default (`smart`)
   and `single` emit separate chunk modules referenced by relative dynamic imports
   (`import("./app.tsx_App_component_x")`). Those need a resolver/bundler to serve. `inline`
   inlines every `$` closure into one module, so there is nothing relative left to resolve.

3. **`bindings/*` is not a package export.** `@builder.io/qwik/bindings/qwik.wasm.mjs` is in
   the published tarball but absent from the `exports` map, so it cannot be imported as a
   subpath — by URL only. For the same reason the raw wasm binding is called directly
   (`binding.transform_modules(...)`) instead of `createOptimizer()`, which skips the
   optimizer's Node/Deno/Bun platform detection entirely.

4. **The runtime needs exactly four import-map entries**, and two of them must be the *same*
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

5. **`render()` cannot take a bare component.** `render(el, Component)` hits
   `jsx(Component, null)`, and Qwik's `jsx` reads `props.children` eagerly → *"Cannot read
   properties of null (reading 'children')"*. Use `_jsxC(Component, null, 3, key)`, which is
   exactly what the optimizer emits for `<Component />`.

6. **`qwikloader.js` is required, or nothing is interactive.** Qwik does not attach
   per-element DOM listeners. The renderer stores handlers on the element context
   (`element._qc_.li`) and pushes used event names onto `window.qwikevents`; a single global
   delegate — `dist/qwikloader.js` — dispatches to them. Without that script the page renders
   perfectly and ignores every click.

7. **The optimizer reports errors instead of throwing.** It returns `diagnostics` with
   `category: 'error' | 'warning' | 'sourceError'` plus `highlights[].startLine/startCol`.
   On a parse error, `modules` comes back empty.

8. **`isEntry` is never populated.** Every module has `isEntry: false`. With `inline` there is
   exactly one module, so use `modules[0]`.

9. **The compiled module keeps the user's original imports.** `import { component$, useSignal }
   from '@builder.io/qwik'` survives the transform (dead but still linked), so the runtime
   mapped into the import map must actually export those names. It does — `core.prod.mjs` is
   self-contained apart from `build` and `preloader`, and exports the internal helpers the
   compiler emits (`componentQrl`, `inlinedQrl`, `useLexicalScope`, `_fnSignal`, `_jsxQ`,
   `_jsxC`, ...).

## Known limitations

- **Single file only.** `inline` inlines `$` closures, it does not bundle modules, so
  `import './foo'` between user files is not resolved. LiveCodes already has
  `replaceSFCImports` / import-map machinery for exactly this.
- **Client-only rendering.** No SSR, no resumability — the Qwik "resumable" story needs a
  server-rendered document, which is out of scope here.
- **Sandbox is weak.** The result iframe needs `allow-same-origin` for the blob-URL module
  import, so it shares the parent's origin. LiveCodes' dedicated result-page origin +
  postMessage protocol is the real answer.
- **`_jsxC` and the `bindings/` URL are version-coupled.** Both are tied to the pinned
  Qwik version; a version bump needs re-checking.

## Path to LiveCodes

The PoC maps onto LiveCodes almost one-to-one:

| PoC | LiveCodes |
| --- | --- |
| `compile()` | `src/livecodes/languages/qwik/lang-qwik-compiler.ts` |
| `importMap` + qwikloader + iframe | result-page rendering (`src/livecodes/result/`), which already builds import maps |
| the editor / console UI | throw away — the playground already provides both |

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
   inject `qwikloader.js` there. This is result-page concern, not compiler concern.
4. **Standard checklist** from
   [`docs/docs/contribution/adding-languages.mdx`](../livecodes/docs/docs/contribution/adding-languages.mdx):
   `sdk/models.ts` names/aliases, Monaco/CodeMirror/Prism support, `language-info.html`,
   starter template, e2e tests, docs, vendor licenses.
5. **Compiler runs in the worker.** The wasm optimizer works fine off the main thread, which is
   where LiveCodes' compile pipeline expects to run it.
