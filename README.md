# Fresh Market 3D: design docs

**Fresh Market** is an online grocery store you shop by walking through it. You push a trolley down real aisles, click packs off the shelf to drop them in the trolley, and pay at an animated self-checkout. Prices are in INR.

This repository holds the **design documentation** only: the build prompt, the pseudocode and the tech stack. The source code is kept private.

| Document | What's inside |
|---|---|
| [PROMPT.md](PROMPT.md) | The full product brief: features, UX rules, performance targets and acceptance criteria |
| [PSEUDOCODE.md](PSEUDOCODE.md) | How every system works, in language-neutral pseudocode: store layout, planogram, movement, rendering, cart, deals, checkout and the asset pipeline |
| [TECH_STACK.md](TECH_STACK.md) | Libraries, versions, tooling, and why each was chosen |

## At a glance

- **First-person walk-in store.** It has 6 aisles, 75 real products, wall fridges, promo islands, a produce area and 4 self-checkout lanes.
- **Real 3D packs.** Branded 3D models are used where they exist. Every other product is rebuilt in 3D from its photo.
- **Click to add.** The pack hops off the shelf into your trolley, and a toast offers *Undo* or *Details*.
- **Search** (`/` or `Ctrl K`) walks you to the shelf. The minimap and aisle chips move you around the store.
- **Deals.** "3 for ₹99" and "Buy 2, get 1 free" are applied across mixed products.
- **Self-checkout.** The trolley empties onto the counter, each item is scanned (beep and flash) into the bag, and a live till screen and receipt follow.
- **Runs on laptops.** It uses instancing, frustum culling, level of detail, and quality tiers that step down automatically. A 2D list view is the fallback when WebGL2 isn't available.

## Controls

| | |
|---|---|
| Walk | `W A S D`, or scroll / swipe to push the trolley |
| Turn | drag, `← →` or `Q E` |
| Run | hold `Shift` |
| Go somewhere | click the floor, the minimap or an aisle chip |
| Add to trolley | click any product |
| Search | `/` or `Ctrl K` |
| Check out | click a lane number above the self-checkouts |
