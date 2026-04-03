# ARCHITECTURE.md — Ripple Agent System

## System Design v0.1

---

## Overview

Ripple is a **coordination primitive** — a foundational pattern for massively 
parallel agent systems where role, memory, and behavior emerge from position 
and local interaction rather than central instruction.

```
┌─────────────────────────────────────────────────────┐
│                    TASK INPUT                        │
│         (click, API, semantic mapper, sensor)        │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│                  ROLE TOPOLOGY                       │
│   Dynamic assignment: CORE / MID / SHELL             │
│   Computed every frame from task proximity           │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌──────────┐    ┌──────────────┐    ┌─────────────────┐
│  CORE    │───▶│     MID      │───▶│     SHELL       │
│ Compute  │    │  Coordinate  │    │  Memory/Learn   │
│ Process  │◀───│  Route       │◀───│  Store/Route    │
└──────────┘    └──────────────┘    └─────────────────┘
      │                                      ▲
      │           RIPPLE EMISSION            │
      └──────────────────────────────────────┘
              Task completion → wave outward
              Shell receives → writes memory
```

---

## Layer 1: Agent Grid

The fundamental data structure is a 2D grid of agents. Each agent is a 
lightweight object with ~10 float fields. No heap allocation during runtime.

```
Grid dimensions: floor(W / CELL) × floor(H / CELL)
Cell size:       5px default (tunable)
Agent count:     ~82,000 at 1920×1080 / CELL=5

At CELL=1:       ~2M agents (requires WebGL renderer)
Target scale:    10M+ agents (CUDA / compute shaders)
```

Agent state is updated in a single sequential pass per frame.
No agent communicates directly — all coordination is mediated by the 
shared field state (rippleIntensity values).

---

## Layer 2: Task System

Tasks are the system's inputs. A task has:

```typescript
interface Task {
  id:       number      // unique identifier
  x:        number      // grid position
  y:        number      // grid position  
  color:    [r, g, b]   // visual identity
  life:     number      // frames remaining
  maxLife:  number      // total duration
  completed: boolean
}
```

Tasks are spawned externally (click, API, auto-spawn) and placed at a 
grid position. The role topology reorganizes around each active task.

**Task completion** triggers:
1. A final high-intensity ripple emission
2. Memory outcome updates for nearby shell agents
3. Task removal from active set

---

## Layer 3: Ripple System

Ripples are the system's communication mechanism. They carry task identity 
outward from CORE to SHELL.

```typescript
interface Ripple {
  x:        number   // origin
  y:        number   // origin
  taskId:   number   // what task spawned this
  intensity: number  // initial magnitude
  age:      number   // frames since emission
  maxAge:   number   // lifetime (200 frames)
}
```

Ripples are processed by scanning a bounding box around the origin.
As age increases, the scan radius grows (approximating wave expansion).
This bounds the per-ripple computation cost.

**Performance note:** With 10+ concurrent tasks each emitting every 25 
frames, ripple count stays bounded under 100 in typical operation.

---

## Layer 4: Memory System

Shell memory is the system's learning layer. It persists across task cycles.

**Write path:**
```
Ripple reaches shell agent
  → intensity > θ_write
  → memory += α · intensity · (1 - memory)   // logistic
  → memoryAccess = currentFrame
  → memoryOutcome += 0.02
```

**Selective forget path (runs every frame):**
```
Compute utility U from frequency, recency, outcome
  → U < 0.25: fast prune  (memory · δ_fast)
  → U ≥ 0.25: slow decay  (memory · δ_slow)
```

**Route path (runs on task spawn):**
```
New task T spawns
  → Sample shell agents (every 3 cells for performance)
  → Score each: memory · outcome · (1 / distance)
  → If max score > θ_route: experienced route
  → Else: novel distribution
```

---

## Layer 5: Renderer

Current renderer: 2D Canvas pixel manipulation via Uint32Array.

```
Per frame:
  1. Create ImageData buffer
  2. Compute color for each agent based on role + state
  3. Fill CELL×CELL pixel block per agent
  4. putImageData to canvas
  5. Overlay task rings with arc() calls
```

**Performance:** ~60fps at 82K agents on modern hardware.

**Next renderer (planned):** WebGL2 compute shader approach:
- Agent state as texture2D
- Update pass: fragment shader computes new state
- Render pass: instanced quads per agent
- Target: 10M agents at 60fps

---

## Extension Points

### Adding task types

Tasks currently have uniform behavior. To add typed tasks:

```javascript
const TASK_TYPES = {
  COMPUTE:  { duration: 150, emitInterval: 20, energy: 0.008 },
  MEMORY:   { duration: 300, emitInterval: 40, energy: 0.003 },
  NETWORK:  { duration:  80, emitInterval: 15, energy: 0.012 },
}
```

Different types produce visually distinct ripple patterns and 
different shell memory outcomes.

### Semantic mapper (Mercury 2 / LLM integration)

To connect an LLM's output to the field:

```javascript
// Map model state to task types
function semanticMapper(modelState) {
  return {
    type:       classifyContent(modelState.tokens),
    intensity:  modelState.uncertainty,   // high noise = high intensity
    position:   topicToPosition(modelState.topic),
    duration:   modelState.complexity * 100,
  }
}

// Each diffusion step of Mercury 2 → one task cycle
onDiffusionStep(step) {
  const task = semanticMapper(step)
  spawnTask(task.position.x, task.position.y, task.type)
}
```

### External data ingestion

Any time-series data can drive the field:

```javascript
// Audio → tasks
analyser.getFloatFrequencyData(frequencyData)
for (const [freq, amplitude] of frequencyData.entries()) {
  if (amplitude > threshold) {
    spawnTask(freqToX(freq), amplitudeToY(amplitude))
  }
}

// Sensor data, API responses, user activity — same pattern
```

### Physical hardware target

The architecture is designed to run on embedded hardware:

```
Target: ARM Cortex-M7 or similar
Display: LED matrix or e-ink panel
Power:   Battery + solar
Compute: Local only, no cloud dependency
Stack:   C port of core update loop + display driver
```

The CELL size becomes the physical LED pitch.
At 5mm LED pitch, a 50×50 panel = 2500 agents.
Meaningful emergence starts around 10,000 agents.

---

## Relationship to Other Systems

### Veritas Protocol
The ripple memory architecture maps directly to Veritas's 
Verification Intelligence Layer:
- Tasks = labor claims
- CORE = active verification processors  
- SHELL = community labor history
- Ripples = verification completion events
- Selective forgetting = pruning outdated labor records

### Cortical columns (neocortex)
The three-layer role structure mirrors cortical column architecture:
- CORE ≈ Layer 4 (receives and processes input)
- MID ≈ Layer 2/3 (lateral coordination)
- SHELL ≈ Layer 5/6 (output, feedback, memory consolidation)

This convergence was not designed — it emerged from first principles.

---

## Known Limitations (v0.1)

- Sequential CPU update loop: bottleneck at >200K agents
- Ripple scan is O(ripples × scan_area): degrades with many concurrent tasks
- No persistence: memory resets on page reload
- No inter-instance communication: multiple tabs are isolated
- Role determination rescans full grid: could be event-driven

All solvable. This is v0.1.

---

*Ripple v0.1 — Greenrobe Studio — MIT License*
