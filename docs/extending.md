# Extending Ripple

Three clean extension points. Everything else flows from these.

---

## 1. Custom task types

Tasks currently have uniform behavior. Typed tasks let different inputs 
produce different visual signatures and coordination patterns.

```javascript
// Define your task types
const TASK_TYPES = {
  COMPUTE: {
    duration:     150,
    emitInterval: 20,
    energyCost:   0.008,
    color:        [1.0, 0.9, 0.0],  // yellow
  },
  MEMORY: {
    duration:     300,
    emitInterval: 40,
    energyCost:   0.003,
    color:        [0.5, 0.2, 1.0],  // purple
  },
  SIGNAL: {
    duration:      80,
    emitInterval:  10,
    energyCost:    0.012,
    color:         [0.0, 1.0, 0.6],  // green
  },
}

// Spawn with type
function spawnTypedTask(x, y, type) {
  const def = TASK_TYPES[type]
  tasks.push({
    id: taskIdCounter++,
    x, y,
    color: def.color,
    life: def.duration,
    maxLife: def.duration,
    emitInterval: def.emitInterval,
    energyCost: def.energyCost,
    completed: false,
  })
}
```

---

## 2. Semantic mapper — connect any data source

The semantic mapper translates external data into task inputs.
This is the bridge between Ripple and the real world.

### Audio input

```javascript
const audioCtx = new AudioContext()
const analyser = audioCtx.createAnalyser()
const freqData = new Float32Array(analyser.frequencyBinCount)

function audioToTasks() {
  analyser.getFloatFrequencyData(freqData)
  
  for (let i = 0; i < freqData.length; i += 8) {
    const amplitude = (freqData[i] + 140) / 140  // normalize
    if (amplitude > 0.3) {
      const x = Math.floor((i / freqData.length) * cols)
      const y = Math.floor((1 - amplitude) * rows * 0.8 + rows * 0.1)
      spawnTask(x, y)
    }
  }
}

// Call audioToTasks() in your main loop
```

### LLM / Mercury 2 diffusion steps

```javascript
// Each diffusion step maps to agent field state
function onDiffusionStep(step) {
  // step.uncertainty: how noisy is the current state (0-1)
  // step.tokens: current token candidates
  // step.semanticCluster: topic classification

  const x = semanticToX(step.semanticCluster)
  const y = Math.floor(rows * 0.5)
  const taskType = uncertaintyToType(step.uncertainty)
  
  spawnTypedTask(x, y, taskType)
  
  // High uncertainty = many concurrent tasks = turbulent field
  if (step.uncertainty > 0.7) {
    spawnTypedTask(x + Math.random()*20-10, y + Math.random()*20-10, 'SIGNAL')
  }
}

function uncertaintyToType(u) {
  if (u > 0.6) return 'SIGNAL'   // noisy, fast
  if (u > 0.3) return 'COMPUTE'  // working
  return 'MEMORY'                 // converging, slow
}
```

### Sensor / IoT data

```javascript
// Any numeric stream → task at position
function streamToTask(value, channel, totalChannels) {
  const x = Math.floor((channel / totalChannels) * cols)
  const y = Math.floor((1 - value) * rows)
  spawnTask(x, y)
}

// Example: temperature sensors across a building
sensors.forEach((temp, index) => {
  const normalized = (temp - 15) / 25  // 15-40°C range
  streamToTask(normalized, index, sensors.length)
})
```

---

## 3. WebGL renderer — scale to 10M agents

The current Canvas 2D renderer hits its limit around 200K agents.
For 10M+ you need WebGL2.

### Architecture

```
Two textures (ping-pong):
  stateA: RGBA32F — encodes (energy, memory, taskLoad, role)
  stateB: RGBA32F — output of update pass

Update pass (fragment shader):
  For each texel (agent):
    - Sample neighbors
    - Apply role-specific update rules
    - Write new state to output texture

Render pass (vertex + fragment shader):
  - Instanced quad per agent (CELL × CELL pixels)
  - Color computed from state in vertex shader
  - No CPU readback needed
```

### Minimal WebGL2 setup

```javascript
const gl = canvas.getContext('webgl2')

// Create state textures
function createStateTexture(w, h) {
  const tex = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, tex)
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA32F, w, h, 0,
                gl.RGBA, gl.FLOAT, null)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST)
  return tex
}

// Update shader reads neighbors via texture sampling
// Full shader code: see examples/webgl-renderer/ (coming soon)
```

---

## 4. Persistence layer

Shell memory currently resets on page reload.
To persist across sessions:

```javascript
// Save shell memory state
function saveMemory() {
  const memoryMap = {}
  for (let j = 0; j < rows; j += 2) {
    for (let i = 0; i < cols; i += 2) {
      const a = agents[j][i]
      if (a.role === 2 && a.memory > 0.1) {
        memoryMap[`${i},${j}`] = {
          memory: a.memory,
          outcome: a.memoryOutcome,
          access: a.memoryAccess,
        }
      }
    }
  }
  localStorage.setItem('ripple_memory', JSON.stringify(memoryMap))
}

// Restore on init
function loadMemory() {
  const saved = localStorage.getItem('ripple_memory')
  if (!saved) return
  const memoryMap = JSON.parse(saved)
  for (const [key, val] of Object.entries(memoryMap)) {
    const [i, j] = key.split(',').map(Number)
    if (agents[j]?.[i]) {
      agents[j][i].memory = val.memory
      agents[j][i].memoryOutcome = val.outcome
      agents[j][i].memoryAccess = val.access
    }
  }
}
```

---

## 5. Physical hardware port (concept)

For embedded/LED panel deployment:

```c
// C port of core update loop
// Target: ARM Cortex-M7, 480MHz

typedef struct {
  float energy;
  float memory;
  float taskLoad;
  uint8_t role;
  uint8_t rippleIntensity;
} Agent;

Agent grid[GRID_W][GRID_H];

void update_shell(Agent* a, uint32_t frame) {
  if (a->rippleIntensity > THETA_WRITE_FIXED) {
    float rip = a->rippleIntensity / 255.0f;
    a->memory += ALPHA * rip * (1.0f - a->memory);
  }
  // selective forgetting...
  float utility = compute_utility(a, frame);
  float delta = utility < THETA_FORGET ? DELTA_FAST : DELTA_SLOW;
  a->memory = fmaxf(0.0f, a->memory - delta * a->memory);
}

// LED output: map agent state to RGB
void agent_to_led(Agent* a, uint8_t* r, uint8_t* g, uint8_t* b) {
  if (a->role == SHELL) {
    *r = (uint8_t)(a->memory * 122);
    *g = (uint8_t)(a->memory * 25);
    *b = (uint8_t)(a->memory * 255);
  }
  // CORE, MID...
}
```

---

## Contributing

Open an issue describing what you're building.  
PRs welcome — especially:
- WebGL renderer
- New semantic mappers
- Hardware ports
- Novel agent rule sets

*Ripple v0.1 — Greenrobe Studio — MIT License*
