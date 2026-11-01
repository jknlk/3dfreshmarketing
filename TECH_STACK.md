# Tech stack

## Runtime (in the browser)

| Layer | Choice | Version | Why |
|---|---|---|---|
| Framework | **Next.js** (App Router) | 15.5 | Routing, image/font handling and production builds. The 3D scene is loaded client-side only (`ssr: false`). |
| UI library | **React** | 19.1 | The heads-up display (HUD) and the 3D scene share one component model. |
| 3D engine | **three.js** | 0.180 | WebGL2 renderer, instanced meshes, GLB loading, Catmull-Rom curves for walking paths. |
| React renderer for three.js | **@react-three/fiber** | 9.3 | Declarative scene graph and a per-frame `useFrame` loop. |
| 3D helpers | **@react-three/drei** | 10.7 | `Environment` + `Lightformer` (reflections without an HDR download), `PerformanceMonitor`, `AdaptiveDpr`, `BakeShadows`, `Preload`, GLTF loading. |
| Post-processing | **@react-three/postprocessing** + **postprocessing** | 3.0 / 6.37 | N8AO ambient occlusion, bloom, neutral tone mapping, colour grade, vignette, SMAA. High tier only. |
| Animation | **GSAP** | 3.13 | Camera flights, the scripted walk-in, intro wordmark and UI motion. |
| State | **Zustand** (+ `persist`) | 5.0 | Cart (saved to localStorage) and UI state. The camera pose deliberately lives *outside* React. |
| Styling | **Tailwind CSS** | 4 | HUD panels, drawers, toasts and the 2D catalogue. |
| Font | **Inter** (variable, via Fontsource) | 5.2 | UI typography. Static WOFF files are used for canvas text on signs and till screens. |
| Audio | **Web Audio API** | — | Synthesised beeps and pops. It unlocks on the first click or key press. |
| Language | **TypeScript** | 5 | Strict types across the catalogue, planogram and state. |

## Asset pipeline (Node scripts, run at build time)

| Tool | Used for |
|---|---|
| **Sketchfab Data API** | Searching and downloading CC-BY 3D models (trolley, fridges, branded packs). Credits are recorded automatically. |
| **Open Food / Beauty / Products Facts API** | Resolving each catalogue item to a real product: name, brand, barcode, front packshot. |
| **IS-Net** (`isnet-general-use`, Apache-2.0) via **onnxruntime-node** | Removing the background from packshots that aren't plain studio shots. |
| **sharp** | Image decode/resize, flood-fill cutouts, silhouette tracing, WebP encoding, label-atlas packing. |
| **glTF-Transform** + **meshoptimizer** | Model optimisation: dedupe, weld, simplify toward a triangle budget, meshopt compression, textures capped at 1K WebP. |

## Tooling

- **ESLint 9** with `eslint-config-next`
- `tsc --noEmit` for type checks
- npm scripts: `dev`, `build`, `start`, `lint`, `typecheck`, `assets:search`, `assets:models`, `assets:products`

## Data

- **Catalogue:** a seed file (product search query or barcode, pack shape, price) is turned into a generated catalogue JSON. It holds 75 products in 6 categories and prices in INR.
- **Label atlas:** a single WebP texture shared by every rebuilt 3D pack, so the whole store uses one texture for labels.
- **Models:** about 30 optimised GLBs (packs, trolley, drinks fridge, display case, fruit boxes).

## Planned

- **Commerce backend:** Shopify or Medusa behind a `CatalogProvider` interface (prices, stock, offers).
- **Payments:** a real payment gateway behind a server action, replacing the simulated pay step.
- **Product media:** retailer or brand media-kit images in place of community packshots.

## Target hardware

The baseline is a laptop with an integrated GPU (Intel Iris Xe class) on the **Balanced** tier. **High** adds post-processing for dedicated GPUs, and **Battery** drops shadows and lowers the device pixel ratio (DPR).
