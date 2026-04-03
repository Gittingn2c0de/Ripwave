# MATH.md — Ripple Agent System
## Complete Mathematical Specification v0.1

---

## 1. Agent State Vector

Each agent A at grid position (x, y) carries the following state:

```
A = {
  role         ∈ {0, 1, 2}     // 0=CORE, 1=MID, 2=SHELL
  energy       ∈ [0, 1]        // computation capacity
  memory       ∈ [0, 1]        // accumulated experience
  memoryAccess ∈ ℤ+            // frame of last memory access
  memoryOutcome∈ [0, 1]        // success rate of tasks using this memory
  taskLoad     ∈ [0, 1]        // current task load
  taskId       ∈ ℤ             // active task identifier (-1 if none)
  signal       ∈ [0, 1]        // current ripple intensity received
  rippleTaskId ∈ ℤ             // task id of dominant ripple
}
```

---

## 2. Role Determination by Locus

Role is computed every frame. Not fixed.

Let d(A, T) = Euclidean distance from agent A to nearest active task source T:

```
d(A, T) = √((A.x - T.x)² + (A.y - T.y)²)

role(A) = 
  0 (CORE)   if d(A, T) < r_core
  1 (MID)    if r_core ≤ d(A, T) < r_mid
  2 (SHELL)  if d(A, T) ≥ r_mid

Constants:
  r_core = 10  (cells)
  r_mid  = 28  (cells)
```

When multiple tasks are active, an agent takes the most active role 
(lowest index) from any task within range.

---

## 3. Ripple Propagation

### 3.1 Ripple Emission

A CORE agent emits a ripple when:
- It has active task load > 0
- Its energy > 0.3
- Frame counter t mod 25 = 0

Emitted ripple R:
```
R = {
  origin: (x, y)
  taskId: A.taskId
  intensity: A.taskLoad · A.energy
  age: 0
}
```

### 3.2 Ripple Intensity Function

For a ripple originating at (rx, ry), the intensity at observer position 
(ox, oy) at ripple age t is:

```
I(d, t) = I₀ · e^(-λd) · e^(-μt) · cos(ωt - kd)

where:
  d  = √((ox - rx)² + (oy - ry)²)   // distance from origin
  I₀ = initial emission intensity
  λ  = 0.08    // spatial decay constant
  μ  = 0.025   // temporal decay constant  
  ω  = 0.12    // angular frequency (oscillation speed)
  k  = 0.18    // wave number (spatial frequency)

Ripple is alive while age < 200 frames.
Intensity clipped to [0, 1].
```

The cos term creates the traveling wave appearance — alternating 
bright and dark rings expanding from origin.

### 3.3 Constructive Interference

When multiple ripples reach the same agent simultaneously:

```
// Intensities merge — always constructive
I_combined = I₁ + I₂ - I₁·I₂

// Task identities preserved separately
agent.carries = [taskId₁, taskId₂, ...]

// Dominant task = highest intensity contributor
agent.rippleTaskId = argmax(Iₙ)
```

No destructive interference. Tasks must coexist and flow.
Different task streams diverge at shell boundary by memory affinity.

---

## 4. Agent Update Rules by Role

### 4.1 CORE Agent Update

```
// Find nearest task T at distance d
if d < r_core:
  A.taskLoad = min(1, A.taskLoad + 0.08)
  A.taskId   = T.id
  A.energy  -= A.taskLoad · 0.006

else:
  A.taskLoad = max(0, A.taskLoad - 0.05)
  A.taskId   = -1

// Passive energy recovery
A.energy = min(1, A.energy + 0.002)
```

### 4.2 MID Agent Update

```
// Signal from ripple field
A.signal = A.rippleIntensity · 0.7

// Inherit task load from adjacent CORE neighbors
coreSignal = max(neighbor.taskLoad for neighbor in adjacent if neighbor.role == CORE)
A.taskLoad = max(A.taskLoad · 0.92, coreSignal · 0.6)

// Energy cost of signal passing
A.energy -= A.signal · 0.001

// Passive recovery
A.energy = min(1, A.energy + 0.004)
```

### 4.3 SHELL Agent Update

```
// Memory write from ripple
if A.rippleIntensity > θ_write:
  A.memory += α · A.rippleIntensity · (1 - A.memory)   // logistic
  A.memoryAccess   = t
  A.memoryOutcome  = min(1, A.memoryOutcome + 0.02)

Constants:
  θ_write = 0.12   // minimum ripple intensity to trigger write
  α       = 0.12   // learning rate
```

---

## 5. Selective Forgetting

### 5.1 Utility Score

For shell agent A at frame t:

```
recency   = max(0, 1 - (t - A.memoryAccess) · 0.0005)
frequency = A.memoryOutcome
outcome   = A.memoryOutcome

U(A) = 0.4 · frequency + 0.4 · recency + 0.2 · outcome
```

### 5.2 Forgetting Rule

```
if U(A) < θ_forget:
  A.memory = max(0, A.memory - δ_fast · A.memory)   // prune
else:
  A.memory = max(0, A.memory - δ_slow · A.memory)   // preserve

Constants:
  θ_forget = 0.25      // utility threshold
  δ_fast   = 0.009     // aggressive pruning rate
  δ_slow   = 0.00004   // near-permanent preservation rate
```

High-utility memories approach permanence (δ_slow → 0).
Low-utility memories are aggressively pruned (δ_fast).
The shell self-curates toward experienced-route knowledge.

---

## 6. Memory-Informed Routing

When a new task T spawns at position (tx, ty):

```
// Query shell for experienced route
for each shell agent S (sampled every 3 cells for performance):
  
  d_st = √((S.x - tx)² + (S.y - ty)²)
  
  score(S, T) = S.memory · S.memoryOutcome · (1 / (1 + d_st · 0.01))

bestScore = max(score(S, T)) over all shell agents S

if bestScore > θ_route:
  // Experienced route — task dispatched toward known cluster
  route = EXPERIENCED
else:
  // Novel task — broad distribution across available CORE
  route = NOVEL

Constants:
  θ_route = 0.38
```

---

## 7. Energy Dynamics

```
CORE:
  consumption: E -= taskLoad · 0.006 per frame
  recovery:    E += 0.002 per frame
  floor/ceil:  E ∈ [0, 1]

MID:
  consumption: E -= signal · 0.001 per frame
  recovery:    E += 0.004 per frame

SHELL:
  consumption: E -= (memory_write · 0.001) per frame
  recovery:    E += 0.005 per frame

Shell agents are energy-efficient by design.
They should never burn out — memory must always be available.
```

---

## 8. Visual Encoding

```
CORE agent color:
  taskColor = color associated with A.taskId
  intensity = A.taskLoad · A.energy
  R = (0.6 + 0.4 · taskColor.r) · intensity + bloom(0.15)
  G = (0.6 + 0.4 · taskColor.g) · intensity + bloom(0.15)
  B = (0.6 + 0.4 · taskColor.b) · intensity + bloom(0.15)

MID agent color:
  sig = A.signal · 0.8 + A.taskLoad · 0.4
  R = sig · 0.05
  G = sig · 0.85
  B = sig · 1.0
  // Cyan signal carrier

SHELL agent color:
  taskColor = color of dominant ripple task
  R = A.memory · 0.48 + A.rippleIntensity · taskColor.r · 0.7
  G = A.memory · 0.10 + A.rippleIntensity · taskColor.g · 0.5
  B = A.memory · 1.00 + A.rippleIntensity · taskColor.b · 0.8
  // Deep purple base, task-colored ripple overlay

Ripple overlay (all roles):
  if A.rippleIntensity > 0.08:
    R = max(R, ripple · 0.95)
    G = max(G, ripple · 0.92)
    B = max(B, ripple · 0.75)
  // Warm white traveling wave

Ambient flicker:
  flicker = baseGlow · (0.5 + 0.5 · sin(t · 0.05 + phase))
  // Field is never fully dark
```

---

## 9. Parameter Reference

| Parameter | Symbol | Value | Description |
|-----------|--------|-------|-------------|
| Core radius | r_core | 10 | Role boundary: CORE |
| Mid radius | r_mid | 28 | Role boundary: MID/SHELL |
| Spatial decay | λ | 0.08 | Ripple amplitude vs distance |
| Temporal decay | μ | 0.025 | Ripple amplitude vs age |
| Angular frequency | ω | 0.12 | Ripple oscillation speed |
| Wave number | k | 0.18 | Ripple spatial frequency |
| Learning rate | α | 0.12 | Memory write rate |
| Write threshold | θ_write | 0.12 | Min ripple to write memory |
| Route threshold | θ_route | 0.38 | Memory score for experienced routing |
| Forget threshold | θ_forget | 0.25 | Utility threshold for pruning |
| Fast decay | δ_fast | 0.009 | Low-utility memory pruning rate |
| Slow decay | δ_slow | 0.00004 | High-utility memory decay rate |
| Ripple lifetime | — | 200 frames | Max ripple age |
| Emit interval | — | 25 frames | CORE ripple emission frequency |

---

## 10. Complexity

```
Per frame:
  Role update:    O(cols · rows)           — full grid scan
  Ripple update:  O(ripples · scan_area)   — bounded by ripple age
  Agent update:   O(cols · rows)           — full grid scan
  Render:         O(cols · rows · CELL²)   — pixel fill

Total: O(N) where N = cols · rows

At CELL=5, 1920×1080: ~82,000 agents
At CELL=2, 1920×1080: ~518,000 agents
At CELL=1, 1920×1080: ~2,000,000 agents (requires WebGL)

For 10M+ agents: port to WebGL compute shaders or NumPy/CUDA
```

---

## 11. Theoretical Basis

The architecture draws from:

- **Cellular automata** — local rules, global emergence (Conway, Gray-Scott)
- **Reaction-diffusion systems** — Turing pattern formation
- **Cortical column architecture** — layer-specific roles in neocortex
  - Layer 4: receives input (CORE)
  - Layer 2/3: lateral communication (MID)  
  - Layer 5/6: output and feedback (SHELL)
- **Swarm intelligence** — stigmergic coordination without central control
- **Delay-tolerant networking** — store-and-forward under intermittent connectivity

The ripple memory model was independently derived. The selective forgetting 
utility function is original. The role-by-locus dynamic topology is original.

---

*Ripple v0.1 — Greenrobe Studio — MIT License*
