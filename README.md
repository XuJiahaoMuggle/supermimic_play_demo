# SuperMimic Browser Playback

A fully client-side sim2sim demo for SuperMimic motion tracking, running
MuJoCo physics and the trained policy directly in the browser.

**Live demo:** <https://xujiahaomuggle.github.io/supermimic_play_demo/>

## What it is

This repository hosts the static build of the SuperMimic browser demo. There is
no Python server, WebSocket backend, or inference service at runtime.

- **MuJoCo WebAssembly** — rigid-body simulation at 200 Hz
- **ONNX Runtime Web** — actor inference at 50 Hz
- **Three.js** — scene rendering
- **BONES-SEED** — reference motion clips

The browser runs four physics substeps per control step, matching the training
setup (`step_dt = 0.02 s`).

## Demo controls

| Input | Action |
| --- | --- |
| `Space` | Play / pause |
| `R` | Reset the current clip |
| `ArrowRight` | Advance one control step |
| Drag | Orbit the camera |
| Scroll | Zoom |

The side panel also exposes:

- **Checkpoint** — select the exported actor
- **Motion** — filter and choose a BONES-SEED clip
- **Inference backend** — `Auto / WebGPU` or `WASM`
- **Target clip** — official position target or torque-aware clipping

## Current deployment

The current page was exported from:

```text
runs/bones_seed/bones_seed_bin_0_2s/last.pt  (it=27000)
```

It contains one ONNX actor and 156 motion clips. The exported actor uses a
10-frame proprio history and a 640-dimensional motion goal.

## Repository layout

```text
index.html                    prebuilt demo entry point
assets/                       bundled JavaScript and CSS
manifest.json                 checkpoints and motion metadata
models/g1/                    G1 MJCF and mesh assets
policies/                     exported ONNX actor(s)
motions/motions.bin           packed reference motion clips
ort/                          ONNX Runtime Web wasm loader
wasm/                         MuJoCo WebAssembly
.github/workflows/pages.yml   GitHub Pages deployment
.nojekyll                     disable Jekyll processing on Pages
```

This is a deployment repository: the files above are generated artifacts. The
source code lives in the main SuperMimic repository under `demos/play_demo/`.

## Local preview

From the repository root:

```bash
python -m http.server 8081
```

Then open <http://localhost:8081/>.

The demo uses relative asset paths, so it works when served from a subpath as
well as from the site root.

## Updating the demo

Build the static assets in the SuperMimic source repository:

```bash
python -m demos.play_demo.export_web \
  --checkpoint runs/bones_seed/bones_seed_bin_0_2s/last.pt \
  --output demos/play_demo/web/public

cd demos/play_demo/web
npm ci
npm run build
```

Then replace the prebuilt files in this repository with `web/dist`:

```bash
rm -rf assets
cp -a /path/to/supermimic/demos/play_demo/web/dist/. .
```

Commit and push the updated files. The GitHub Actions workflow will publish the
new build to Pages.

The export intentionally packs only the selected motion clips. Adding an entire
multi-gigabyte motion library would make the static site too large for Pages.

## GitHub Pages

The workflow is in `.github/workflows/pages.yml` and uses:

- `actions/configure-pages`
- `actions/upload-pages-artifact` with `path: .`
- `actions/deploy-pages`

In the repository settings, configure:

```text
Settings -> Pages -> Build and deployment -> Source: GitHub Actions
```

If `configure-pages` reports:

```text
Resource not accessible by integration
```

enable Pages once in the repository settings, then re-run the workflow.

## Troubleshooting

### The browser still shows an old version

GitHub Pages and the browser cache assets for several minutes. Use a hard
refresh or an incognito window:

```text
Ctrl+Shift+R  /  Cmd+Shift+R
```

### ONNX Runtime reports `initWasm() failed`

Check that these files return HTTP 200:

```text
/ort/ort-wasm-simd-threaded.jsep.mjs
/ort/ort-wasm-simd-threaded.jsep.wasm
```

The demo builds absolute asset URLs from `window.location.href`, so ONNX Runtime
resolves its wasm files from the deployed subpath rather than from
`/assets/ort/`.

GitHub Pages cannot set `Cross-Origin-Opener-Policy` or
`Cross-Origin-Embedder-Policy` response headers. ONNX Runtime is configured with
a single wasm thread, but if a browser still blocks wasm initialization, serve
the same build from Cloudflare Pages or Netlify with these headers, or add a
`coi-serviceworker` shim.

### WebGPU is unavailable

Use the `WASM` inference backend in the side panel. WebGPU availability depends
on the browser, GPU, and platform.

## Asset size

The deployed build is about 105 MB:

- ONNX actor: about 35 MB
- ONNX Runtime wasm: about 30 MB
- motion archive: about 12 MB
- MuJoCo wasm: about 11 MB
- G1 meshes: the remainder

All files are below GitHub's 100 MB per-file limit and the total is below the
GitHub Pages site limit. Do not use Git LFS for Pages assets: GitHub Pages does
not resolve LFS pointers to the real files.

## Credits

This demo builds on:

- [MuJoCo](https://mujoco.org/) and MuJoCo WebAssembly
- [ONNX Runtime Web](https://onnxruntime.ai/docs/tutorials/web/)
- [Three.js](https://threejs.org/)
- SuperMimic and BONES-SEED motion data

Motion, robot, and model assets remain subject to their original licenses and
terms of use. Check those terms before redistributing or republishing this demo.
