# ripwave

**A massively parallel agent coordination system where computation is its own visualization.**

Built by [Lonlon](https://github.com/Gittingn2c0de) / 
Open sourced because good ideas should spread.

---

## What is this

Ripwave is a particle agent system built on a simple premise:

> Each pixel is an agent. Agents coordinate locally. Global behavior emerges.

Every agent has a role determined by its position relative to active tasks:

- **CORE** — active computation
- **MID** — signal routing and coordination  
- **SHELL** — memory, storage, continuous learning

Information travels as ripples. When a task completes, a wave propagates outward. Shell agents at the boundary receive it, store it, and hold it — selectively — based on utility. The system learns what matters and forgets what doesn't.

The visual output isn't a representation of what's happening. **It is what's happening.**

---

## Why open source

Because the right move with a coordination primitive is to let people build on it — not hoard it.

If you build something with this lmk would be nice. I'm @psychcamel on X.

---

## Quick start

No build step. No dependencies. One file.

```bash
git clone https://github.com/psychcamel/ripple
cd ripple
open demos/ripple_v0.1.html
```

Or just open `ripple_v0.1.html` directly in any modern browser.

Click anywhere on the field to spawn a task. Watch roles shift. Watch memory accumulate in the shell ring. Hit BURST to spawn 5 concurrent tasks and observe constructive interference.

---

## Core concepts

### Role-by-locus

An agent's role is not fixed. It is computed every frame based on proximity to the nearest active task source:

```
d < r_core (10 cells)  →  CORE
d < r_mid  (28 cells)  →  MID  
d ≥ r_mid              →  SHELL
```

As tasks move, the role topology shifts with them. An agent can be CORE for one task and SHELL for another simultaneously.

### Ripple propagation

When a CORE agent completes a task cycle, it emits a ripple. The ripple carries the task's identity outward:

```
I(d, t) = I₀ · e^(-λd) · e^(-μt) · cos(ωt - kd)
```

A traveling wave that decays in space and time. The wavefront is visible as a bright ring expanding from the task center.

### Selective forgetting

Shell agents don't forget uniformly. Memory decays based on utility:

```
U = 0.4 · frequency + 0.4 · recency + 0.2 · outcome

if U < 0.25  →  fast pruning  (δ = 0.009)
if U ≥ 0.25  →  slow decay    (δ = 0.00004)
```

High-utility memories are essentially permanent. The shell self-curates. Over time, the spatial distribution of shell memory reflects the system's actual experience — dense where it's been, light where it hasn't.

### Constructive interference

When two ripples meet, they combine — intensities merge, task identities are preserved separately. Concurrent task streams diverge at the shell boundary toward different memory regions. You can watch the system sort itself.

---

## Architecture

See [ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full system design.  
See [MATH.md](MATH.md) for all equations with derivations.

---

## Applications

What people have built or are building:

- **Ambient compute display** — real task processing visualized as a living field
- **LLM interface layer** — model diffusion steps mapped to agent field behavior
- **Generative visuals** — input data transformed through agent rules)
- **Live music visualization** — audio streams as typed task input

If you're building something, open an issue and we'll add it here.

---

## Extending

The system is designed to be extended at three points:

**1. Task types** — define new task signatures with different visual behavior and routing rules

**2. Agent rules** — modify the update functions for CORE, MID, and SHELL independently

**3. Semantic mapper** — translate external data (audio, text, sensor input) into task types the field can ingest

See [extending.md](docs/extending.md) for implementation guides.

---

## Roadmap

- [ ] WebGL renderer for 10M+ agent scale
- [ ] Mercury 2 semantic task mapper (LLM diffusion → field tasks)
- [ ] Python/NumPy port for Colab-scale simulation
- [ ] Voice mode interface layer
- [ ] Physical hardware spec (embedded, battery-powered panel)

---

## License

MIT. Build whatever you want. Credit appreciated, not required.

---

## Built by

Lonlon — independent builder, Germany for now lol.  


---

*"The visual output isn't a representation of what's happening. It is what's happening."*
