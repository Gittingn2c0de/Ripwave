# Getting Started

## Run it in 30 seconds

```bash
git clone https://github.com/psychcamel/ripple
cd ripple
open demos/ripple_v0.1.html
```

No npm. No build step. No dependencies. One HTML file.

---

## Controls

| Action | Result |
|--------|--------|
| Click anywhere | Spawn task at that position |
| SPAWN TASK | Spawn random task |
| BURST ×5 | Spawn 5 concurrent tasks |
| AUTO ON/OFF | Toggle automatic task spawning |
| DISRUPT | Inject energy chaos into random cluster |
| RESET | Clear all state |
| PAUSE | Freeze simulation |

---

## What you're watching

**White/yellow agents** — CORE. Actively processing a task. Color tinted by task identity.

**Cyan agents** — MID ring. Routing signals between CORE and SHELL. 

**Purple agents** — SHELL. Memory layer. Brightness = accumulated memory. Watch these get denser over time.

**Traveling white rings** — Ripples. Emitted by CORE on task completion. Carry task identity outward to SHELL.

**HUD top-right** — Live stats: agent count, FPS, active tasks, shell memory average, ripple count.

**SHELL MEMORY log** — Shell agents that hit significant memory thresholds. Gets richer the longer you run.

**TASK STREAM log** — Each spawned task. ⚡ROUTED means shell memory recognized a similar task and fast-tracked it.

---

## Tuning parameters

All key parameters are at the top of the script:

```javascript
const R_CORE = 10;      // Core role radius (cells)
const R_MID  = 28;      // Mid role radius (cells)
const LAMBDA  = 0.08;   // Ripple spatial decay
const MU      = 0.025;  // Ripple temporal decay
const OMEGA   = 0.12;   // Ripple oscillation speed
const K       = 0.18;   // Ripple wave number
const ALPHA   = 0.12;   // Memory learning rate
const THETA_WRITE  = 0.12;   // Min ripple to write memory
const THETA_ROUTE  = 0.38;   // Memory score for experienced routing
const DELTA_FAST   = 0.009;  // Low-utility memory pruning rate
const DELTA_SLOW   = 0.00004; // High-utility memory decay rate
const THETA_FORGET = 0.25;   // Utility threshold
```

Try:
- Increasing `R_CORE` for larger task influence zones
- Decreasing `LAMBDA` for longer-range ripples
- Increasing `ALPHA` for faster memory accumulation
- Decreasing `THETA_FORGET` for more aggressive pruning

---

## Performance

Default cell size is 5px — roughly 82,000 agents at 1080p.

To increase agent count, reduce `CELL`:

```javascript
const CELL = 3;   // ~230,000 agents — still smooth on modern hardware
const CELL = 2;   // ~518,000 agents — may drop below 60fps
const CELL = 1;   // ~2,000,000 agents — needs WebGL renderer
```

For 10M+ agents see the WebGL roadmap in ARCHITECTURE.md.

---

## Next steps

- Read [MATH.md](../MATH.md) for the full equations
- Read [ARCHITECTURE.md](../ARCHITECTURE.md) for extension points
- Read [extending.md](extending.md) to build on top of this
- Open an issue if you build something
