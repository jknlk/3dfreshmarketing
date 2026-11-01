# Build prompt

This is the brief used to build Fresh Market. It is written so that a developer, or an AI coding assistant, could build the same product from it.

---

## The brief

> Build a production-quality **walk-in 3D supermarket website** called **Fresh Market**. The shopper explores a realistic grocery store in first person, pushing a trolley, and buys real branded products. The experience should feel like a polished retail product, not a tech demo. It must clearly beat existing "virtual store" sites on visual quality, speed and ease of use.
>
> **Stack:** Next.js (App Router) + React + TypeScript, React Three Fiber + drei for 3D, GSAP for motion, Zustand for state, Tailwind for the UI. Currency is **INR**. Design the catalogue behind an interface so Shopify or Medusa can plug in later.

### 1. The store

- A single hall about 32 × 28 m with a 4.6 m ceiling, entered through sliding doors. It opens with a short scripted walk-in from outside.
- **6 numbered aisles**, each a department with its own colour and tagline: Beverages, Chips & Snacks, Chocolates & Biscuits, Breakfast & Pantry, Personal Care, Home Care.
- 7 double-sided **gondolas** with 5 shelf levels, plus **endcaps** that feature each aisle's hero products.
- A back wall of glass-door **drinks fridges**.
- 2 **promo islands** built as stepped pyramids of featured packs ("Ice-cold deal: 3 for ₹99", "Snack attack: Buy 2, get 1 free").
- A **produce** area, overhead **aisle signs**, shelf-edge **price tags** and department colour strips.
- **4 self-checkout lanes**, each with an unloading shelf, a scanner, a touchscreen, a card reader, a bagging area and a receipt printer.

### 2. Products

- About 75 **real products** (Coca-Cola, Pepsi, Lay's, Pringles, Nutella, Maggi, Dove, Ariel and so on) with real names, sizes, barcodes and prices in INR.
- **Every product is a real 3D pack. Never use flat photo cards.**
  - Use a branded 3D model where a CC-BY one exists.
  - Otherwise rebuild the pack from its photo: remove the background, trace the silhouette, and build bottles, cans and jars as round or oval solids, bags as pinched pouches and cartons as boxes, with the photo projected onto the front.
- Lay out shelves like a real **planogram**: blocks of 2–6 facings per product, with a back-stock row so shelves look full from any angle.
- Each pack's physical size comes from its real height and photo proportions.

### 3. Moving around

- Desktop: `WASD` / arrows to walk, `Q E` or drag to turn, `Shift` to run. Scrolling or swiping on a trackpad pushes the trolley forward with momentum.
- Touch: drag to look, tap to move.
- **Click the floor, the minimap or an aisle chip** to glide there. Routes must follow the aisles, never pass through fixtures, and end facing the right way.
- Collision keeps the shopper inside the store and outside every fixture.
- The trolley follows the shopper with a little weight, slides aside while you browse a shelf, and parks in front of you at the till.
- Respect `prefers-reduced-motion`: moves become instant cuts.

### 4. Shopping

- Hovering a pack makes it glow. **Clicking adds one straight to the trolley**: the pack hops off the shelf in an arc and lands in the basket, where it stays visibly stacked.
- A toast confirms the add, counts repeat clicks, and offers **Undo** (exact) and **Details**.
- **Details / Inspector:** walk up to the shelf, lean in, and show the name, brand, size, price, any active deal and a quantity picker.
- **Search palette** (`/` or `Ctrl K`): fuzzy search. Choosing a result walks you to that product's most browsable facing (eye-level shelf first) and opens it.
- **Cart drawer:** lines, quantities, deal savings and total. The cart persists in localStorage.
- **Deals engine:** multibuy ("3 for ₹99") and buy-X-get-Y apply across *mixed* products in a group. The cheapest units are always the ones discounted, so the shopper never pays more than shelf price.

### 5. Self-checkout

- Click a lane number, or Trolley → *Go to self-checkout*. The shopper walks to the lane and stands square-on to the kiosk.
- The trolley empties onto the counter one item at a time. Each item crosses the scanner (beep, red flash) into the bag.
- The kiosk screen is a live canvas texture showing the item list, running total, savings and step indicator. A side panel mirrors it.
- Pay on the card reader (simulated). A receipt feeds out, then the cart clears.
- An empty trolley produces a friendly "nothing to scan" state, not an error.

### 6. Interface

- A clean, warm, premium retail look: Inter font, generous spacing, soft shadows.
- Top bar with the brand, aisle chips, search, a trolley count badge and a list-view toggle.
- Minimap with the player's position and heading. Click it to travel.
- A loader with real progress, and an intro where the wordmark flies into the top bar.
- Settings: quality tier, auto quality, sound, and **Credits & licences** for every CC-BY model.
- A **2D list view** with the same cart and checkout. It is used automatically when WebGL2 is unavailable.

### 7. Performance (hard requirements)

- Smooth on an **integrated laptop GPU (Intel Iris Xe)** at the default tier.
- Draw every product with **instancing**: one draw per product part, not per facing.
- **Frustum-cull** each facing every frame. Switch between detailed and light meshes by distance. Far-away branded models fall back to the light rebuilt pack.
- The camera pose lives in a mutable object outside React. **Walking must never re-render React.**
- **Bake shadows once** (fixtures don't move). Build reflections from in-scene light panels, not an HDR download.
- Quality tiers:
  - **High:** ambient occlusion, bloom, colour grade, reflective floor.
  - **Balanced:** baked soft shadows, no post-processing. Default for integrated and mobile GPUs.
  - **Battery:** no shadows, lower device pixel ratio (DPR).
- A performance monitor steps the tier down automatically when the frame rate drops.
- Compress models (meshopt, ≤1K WebP textures). Put all labels in a single texture atlas.

### 8. Asset pipeline

- `assets:search`: search Sketchfab for downloadable models.
- `assets:models`: download the listed CC-BY models, record credits, then optimise them.
- `assets:products`: resolve the seed catalogue against Open Food / Beauty / Products Facts, choose the cleanest packshot, cut out its background, trace its silhouette, and write the label atlas plus the generated catalogue JSON.
- The repository ships with generated assets, so running the site needs **no API keys**.

### 9. Acceptance criteria

- [ ] You can walk from the door to any aisle, add products, and complete a checkout with keyboard, mouse or touch only.
- [ ] No product appears as a flat card, and nothing clips through shelves.
- [ ] Undo always restores the exact previous cart.
- [ ] Deal totals are correct for mixed baskets.
- [ ] 60 fps on a dedicated GPU, and smooth on Iris Xe at Balanced.
- [ ] Works without WebGL2 (list view) and with reduced motion.
- [ ] Every third-party model is credited in the app.
