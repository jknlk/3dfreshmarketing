# Pseudocode

Language-neutral pseudocode for each system in Fresh Market. Units are metres. +x points right, +z points back toward the entrance, and y points up.

---

## 1. App start-up

```
ON PAGE LOAD:
    IF browser has no WebGL2:
        mode ← "2d"                       // list view, same cart and checkout
    ELSE:
        mount 3D Experience (client-only)

    IF touch device OR low memory OR GPU name matches (intel|mali|adreno|...):
        quality ← "balanced"
    ELSE:
        quality ← "high"

    unlock audio on first click or key press
    preload startup models + label atlas from the HTML (in parallel with the JS)
    show Loader (server-rendered, so it is the first paint):
        a CSS 3D pallet gets one carton per slice of real asset progress
        the wordmark fills bottom-up in step
    WHEN store ready AND pallet full:
        stretch-wrap glints across the pallet, pallet is wheeled off
        play intro: wordmark flies into the top bar
        enterStore()                      // scripted walk in through the doors
```

## 2. Store layout (data, not hand-placed meshes)

```
STORE      = box(x: -16..16, z: -14..14, height 4.6)
GONDOLAS   = 7 double-sided units at fixed x positions, z from -9 to 1, 5 shelf heights
AISLES     = one per category, centred between neighbouring gondolas
FRIDGES    = units along the back wall (z = -14), 5 shelves each
PROMOS     = 2 islands { position, featured productIds, headline, deal }
PRODUCE    = rectangle near the entrance
LANES      = 4 checkouts near the front, numbered right-to-left as the shopper faces them

OBSTACLES  = padded rectangles for every gondola, fridge row, promo, produce and lane
```

Every other system reads from this one data model: the 3D scene, minimap, collision, routing, price tags and search.

## 3. Planogram (where every pack goes)

```
FUNCTION packSize(product):
    h ← product.realHeight
    IF product has 3D model: RETURN (h·model.w/h, h, h·model.d/h)
    w ← min(0.42, h · photoAspect)        // beverages capped at 0.62·h (multipack photos)
    MATCH product.shape:
        box   → depth ← w · (product.depth OR 0.35)
        pouch → depth ← w · 0.3
        round → depth ← w · (product.depth OR 1)
    RETURN (w, h, depth)

FUNCTION fillRun(products, startOffset, origin, alongDir, faceNormal, length, zone):
    cursor ← 0
    FOR k, cycling through products from startOffset, WHILE cursor < length:
        (w, h, d) ← packSize(product)
        facings ← clamp(round(0.5 / (w + gap)), 2, 6)       // block of facings
        REPEAT facings times:
            IF cursor + w > length: STOP
            emit Placement {
                productId, zone,
                position = origin + alongDir·(cursor + w/2) − faceNormal·(inset + d/2) + up·h/2,
                normal   = faceNormal, size = (w, h, d),
                blockStart = (first facing of block)       // price tag hangs here
            }
            cursor += w + gap
        cursor += blockGap

BUILD PLANOGRAM:
    FOR each gondola, FOR each side (west / east):
        category ← aisle served by that side
        FOR each shelf level: fillRun(category products, offset by level, along +z)
    FOR each gondola endcap: fillRun(one featured product per level)
    FOR each fridge unit, FOR each shelf: fillRun(beverages)
    FOR each promo island, FOR 3 stepped tiers, FOR 4 sides: fillRun(featured products)

backRow(placement)  = placement moved back by its own depth   // decorative, not clickable
PRIMARY[product]    = facing with lowest score(zone rank + |y − 1.15|)  // eye level first
viewpointFor(p)     = resolveCollision(p.position + p.normal · 1.25), looking at p
```

## 4. Collision and routing

```
FUNCTION resolveCollision(x, z, r = playerRadius):
    clamp (x, z) inside STORE shrunk by r
    FOR each obstacle grown by r:
        IF point inside:
            push to the nearest edge (smallest of the four distances)
    RETURN (x, z)

FUNCTION planRoute(from, to):
    IF straight segment is clear (sample every 0.2 m): RETURN [from, to]
    candidates ← for each cross-aisle (front, back):
                   [from, (from.x, crossZ), (to.x, crossZ), to]
    RETURN shortest candidate whose legs are all clear (else the shortest overall)
```

## 5. Player pose and camera flights

```
player = { x, z, yaw, pitch, eye: 1.62, lean, flying }   // mutable object, NOT React state

FUNCTION flyTo(target x, z, optional yaw/pitch OR lookAt point, lean):
    cancel current flight
    (x, z) ← resolveCollision(x, z)
    IF lookAt: compute yaw and pitch toward it
    route ← planRoute(player, target)
    IF route is direct:
        tween player → target over min(2.6, 0.7 + dist·0.11) s, ease power3.inOut
    ELSE:
        curve ← Catmull-Rom spline through route points
        tween t: 0 → 1 over min(4.5, 0.9 + curveLength·0.12) s:
            player.position ← curve(t)
            heading ← blend from start yaw to travel direction during first 12%
            on arrival (last 28%): blend heading → final yaw, pitch → final pitch, lean in
    IF reduced motion: duration ≈ 0 (cut)
    RETURN promise resolved on complete OR interrupt
```

## 6. Per-frame shopper controls

```
EVERY FRAME (dt):
    read keys:  forward/back (W S ↑ ↓), strafe (A D), turn (Q E ← →), run (Shift)
    read drag:  yaw/pitch deltas
    read wheel: add decaying forward impulse          // trackpad swipe = push the trolley

    IF any manual input AND player.flying: cancelFlight()

    velocity ← lerp(velocity, inputDir · speed · (run ? 1.8 : 1), smoothing)
    (x, z)   ← resolveCollision(player + rotate(velocity, yaw) · dt)
    camera   ← (x, eye + headBob(speed), z) rotated by yaw, pitch, plus lean

    trolley target ← in front of shopper
                     | slid aside while inspecting
                     | parked at counter during checkout
    trolley ← damp toward target (feels weighted)
    basketAnchor ← trolley basket world position
```

Click handling:

```
ON floor click (point):           flyTo(point)
ON minimap click (map → world):   flyTo(world point)
ON aisle chip (n):                flyTo(aisle n entrance, facing into the aisle)
```

## 7. Rendering products (instanced, culled, LOD)

```
library ← load label atlas + branded GLBs once
FOR each product:
    IF has model: parts ← selected GLB nodes, normalised to a unit cube
    ELSE IF shape = box: parts ← box with photo on front/back, edge strips on sides
    ELSE: parts ← loft(silhouette profile)
        // rows bottom→top; each row an ellipse between left/right silhouette edges
        // pouches pinch flat toward top and bottom seams; caps fan to centre
        // UVs project the photo onto the front from the atlas rect
    build three detail levels:
        FULL  = the model or detailed loft          (within 3.5 m)
        LIGHT = light loft (models fall back to it)  (to 13 m)
        CARD  = flat photo card                     (beyond)

matrix(placement) = translate(position) · rotateY(yawFrom(normal)) · scale(size)

PER PRODUCT, EVERY FRAME:
    full, light, card ← empty lists
    FOR each facing:
        IF bounding sphere outside camera frustum: skip
        IF distance < 3.5 m: full.add(facing)
        ELSE IF distance < 13 m: light.add(facing)
        ELSE: card.add(facing)
        IF distance < BACKSTOCK_DISTANCE: backRow.add(facing)
    IF visible sets changed: rewrite instance matrices and counts
    draw FULL parts × full, LIGHT parts × (light + backRow), CARD × card

ON hover facing:   show glow shell (slightly enlarged back-face mesh)
ON click facing:   quickAdd(facingIndex)
```

## 8. Cart, toast and flights

```
cart = persisted map productId → qty      (localStorage "fresh-market-cart")

FUNCTION addToTrolley(productId, qty = 1, facing = null):
    IF checkout in progress: RETURN
    cart[productId] += qty                 // update first, so Undo is exact
    IF 3D AND facing known:
        launchFlight(productId, from = facing.position)
    ELSE: play "pop"
    toast ← { productId, qty: (same product as current toast ? toast.qty : 0) + qty, facing }

FUNCTION undoAdd():
    cart[toast.productId] −= toast.qty     // removed if ≤ 0
    remove that product's in-air flights; hide toast

FlightPack (per flight):
    hop up off the shelf, then quadratic Bézier: from → raised midpoint → basketAnchor
    on land: remove flight; pack now drawn in TrolleyContents

TrolleyContents:
    units ← cart expanded to one entry per unit, minus units still in the air
    lay out in rows across the basket, then stack in layers, slightly askew
```

## 9. Deals and totals

```
DEALS = [
    { type: multibuy, qty: 3, price: 99, products: [cola cans...] },
    { type: buyXgetY, buy: 2, free: 1,  products: [snacks...] },
]

FUNCTION computeTotals(cart):
    subtotal ← Σ price · qty
    FOR each deal:
        units ← every unit in cart belonging to deal, as prices, sorted high → low
        IF multibuy:
            groups  ← floor(len(units) / qty)
            bundled ← cheapest groups·qty units
            saving  ← max(0, Σ bundled − groups · price)
        IF buyXgetY:
            groups  ← floor(len(units) / (buy + free))
            saving  ← Σ cheapest groups·free units
    total ← subtotal − Σ savings
    RETURN { subtotal, savings[], total, count }
```

## 10. Search and inspector

```
SearchPalette (opened with "/" or Ctrl K):
    results ← fuzzy match query over name, brand, category
    ON choose(product): findProduct(product.id)

FUNCTION findProduct(id):    inspectPlacement(PRIMARY[id])

FUNCTION inspectPlacement(facing):
    IF checkout in progress: RETURN
    open Inspector(product); close search/cart
    viewpoint ← viewpointFor(facing, distance 1.35 (1.7 for promos))
    aim slightly right of the pack on wide screens (panel covers the left)
    flyTo(viewpoint, lookAt = pack, lean in)
```

## 11. Self-checkout state machine

```
stages: idle → walking → unloading → scanning → review → paying → done → idle
                       ↘ empty (trolley had nothing)

FUNCTION startCheckout(lane = nearest by x):
    units ← cart expanded to one productId per unit (frozen)
    stage ← units ? "walking" : "empty"
    IF list view: stage ← "scanning"; RETURN
    stand ← lane front − 2 m (offset aside on wide screens so the panel doesn't cover the kiosk)
    await flyTo(stand, lookAt = counter)
    IF still walking to the same lane: stage ← "unloading"

CheckoutAnimator (per frame):
    unloading: every Δt, next unit jumps from basketAnchor → counter grid slot (4×3 per layer)
               unloaded += 1; when all unloaded → "scanning"
    scanning:  every Δt, next unit slides across the scanner → beep + red flash → bag slot
               scanned += 1; when all scanned → "review"
    (visual units capped; any extra units are counted as scanned at the end)

TillScreen (canvas texture, redrawn on change):
    header + step indicator, item list with thumbnails, running totalsFor(scanned units),
    savings lines, big total; card reader shows amount / approved

review → user presses Pay → "paying" (simulated) → "done": receipt prints
endCheckout(clearCart = true): reset stage, units, counters; empty cart
```

## 12. Quality and performance

```
TIERS:
    high     → DPR 1–1.75, soft shadows (baked), post FX: AO, bloom, tone map, grade, vignette, SMAA
    balanced → DPR 1–1.25, soft shadows (baked), no post FX, renderer tone mapping
    battery  → DPR 0.75–1, no shadows, no post FX

shadows rendered once (BakeShadows): fixtures are static
environment map built from in-scene light panels (Lightformers), rendered once
fog hides the far end; camera far plane 80 m

PerformanceMonitor ON decline (with flip-flop limit):
    IF autoQuality: quality ← next lower tier
AdaptiveDpr while autoQuality
frameloop paused in 2D mode
```

## 13. Asset pipeline (offline)

```
assets:products  (seed → generated catalogue)
    FOR each seed item { q or barcode, shape, price, optional model }:
        candidates ← Open Food/Beauty/Products Facts search (rate-limited, cached, retry on 503)
        relevant   ← candidates containing all query words (fallback: any word)
        choose     ← prefer plain light background (measure border lightness), then search rank
        download full-resolution front image
        cutout:
            IF studio shot: flood-fill background from the border
            ELSE: IS-Net salient-object mask (ONNX)
            IF matte solidity low: fall back to next method, finally the uncut photo
            remove haze, trim; for boxes refill each row between outer edges
            paint transparent pixels with the pack's average colour (no edge bleed)
        silhouette ← per-row [left, right] extents, gaps filled, median-filtered
        save ≤768px WebP for the inspector / 2D view
    pack all cutouts into one atlas.webp; record UV rect per product
    write catalogue JSON { currency: INR, atlas, categories, products[...] }

assets:models
    download listed CC-BY Sketchfab models → record credits
    optimise: dedupe, weld, simplify packs toward a triangle budget,
              (packs retry with looser error tolerances until within budget),
              textures ≤1K WebP (≤512 px on packs), meshopt compression → public/models/*.glb
```
