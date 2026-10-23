# @hmh-3080/atomvm-wasm

Single-threaded [AtomVM](https://www.atomvm.net/) WebAssembly runtime for running Elixir and
Erlang in the browser.

The runtime is built **without Emscripten pthreads**, so it does not use `SharedArrayBuffer`,
does not require `Cross-Origin-Opener-Policy` / `Cross-Origin-Embedder-Policy`, and works in normal
(non-isolated) pages and cross-origin iframes. Popup-based OAuth keeps working.

## Files

- `wasm/AtomVM.mjs` — Emscripten JavaScript loader.
- `wasm/AtomVM.wasm` — AtomVM WebAssembly binary.
- `wasm/bundle.avm` — Elixir 1.17.3 / OTP 27 bytecode bundle (Elixir and Erlang).

`bundle.avm` is the pinned bundle from
[`live-codes/elixir-browser-eval@6b354da`](https://github.com/live-codes/elixir-browser-eval/tree/6b354dab35fb7454ba2812f935b52b398ca1bec8/wasm).

## Usage

Serve the files from the same directory and load `AtomVM.mjs` as an ES module. The loader accepts
the usual Emscripten options:

```js
import init from './AtomVM.mjs';

const Module = await init({
  locateFile: (name) => new URL(name, import.meta.url).href,
  noInitialRun: true,
  preRun: [({ FS }) => FS.writeFile('/data/bundle.avm', bundleBytes)],
});

Module.callMain(['/data/bundle.avm']);
```

LiveCodes uses it through the `beam-wasm` worker at
`src/livecodes/languages/beam-wasm/lang-beam-wasm-worker.raw.js`.

## Build

The build starts from
[`software-mansion-labs/FissionVM`](https://github.com/software-mansion-labs/FissionVM) at commit
`6c3208c7b3dbc7dacc35a19f8de1fa80b358ac73`, applies
`patches/fissionvm-no-pthread.patch`, and compiles with Emscripten 3.1.74 and mbedTLS 3.6.3.1.

```bash
npm run build
```

The script uses a local `emcmake` when available, otherwise Docker with
`emscripten/emsdk:3.1.74`. It writes `wasm/AtomVM.mjs` and `wasm/AtomVM.wasm`, then downloads the
pinned `bundle.avm` and verifies the SHA-256 checksums in `scripts/check.sh`.

The patch contains:

- single-threaded main loop and idle handling (`RUN_IDLE`, `single_thread_idle`);
- Emscripten build without `-pthread` / `-sPROXY_TO_PTHREAD`, with `callMain` exported;
- direct dispatch instead of `emscripten_dispatch_to_thread`;
- no-op pthread/signal paths when `AVM_NO_SMP` is set;
- the missing `erlang:md5/1` NIF backed by mbedTLS, required by OTP's compiler.

## Checks

```bash
npm run check
```

## Upgrading

The build inputs are overridable environment variables, so an upgrade does not require editing the
script:

| Variable | Default |
| --- | --- |
| `FISSIONVM_REF` | pinned FissionVM commit |
| `MBEDTLS_REF` | `v3.6.3.1` |
| `EMSDK_IMAGE` | `emscripten/emsdk:3.1.74` |
| `BUNDLE_URL` | pinned `live-codes/elixir-browser-eval` bundle |

```bash
FISSIONVM_REF=<new-commit> MBEDTLS_REF=<new-tag> BUNDLE_URL=<new-url> npm run build
npm run check
```

Then bump the version in `package.json`, publish with `npm publish --access public`, and update the
package version in LiveCodes (`src/livecodes/vendors.ts`). Run the LiveCodes Elixir/Erlang Playwright
suites before merging.

If FissionVM changed the files touched by `patches/fissionvm-no-pthread.patch`, rebase the patch
onto the new commit, rebuild, and confirm the output still has `callMain` and no
`SharedArrayBuffer`/pthread references (`npm run check` does this).

## License

AtomVM and FissionVM are licensed under Apache-2.0. Elixir and Erlang/OTP are Apache-2.0. Popcorn
is MIT. See `LICENSE` and the upstream projects for details.
