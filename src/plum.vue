<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { isDark } from './toggleDark'

interface Point {
  x: number
  y: number
}

interface Branch {
  start: Point
  length: number
  theta: number
  depth: number
  hueOffset: number
}

type Pattern = 'plum' | 'spiral' | 'crystal'
type Palette = 'mist' | 'sunset' | 'aurora'

interface PatternConfig {
  spread: number
  twist: number
  decayMin: number
  decayMax: number
  depthLimit: number
  splitChance: number
  minChildren: number
  maxChildren: number
}

interface PaletteConfig {
  baseHue: number
  hueStep: number
  saturation: number
  alpha: number
  lightnessDark: [number, number]
  lightnessLight: [number, number]
}

const canvasRef = ref<HTMLCanvasElement | null>(null)
const width = ref(600)
const height = ref(600)

const pattern = ref<Pattern>('plum')
const palette = ref<Palette>('mist')
const density = ref(42)
const speed = ref(90)
const flowEnabled = ref(false)
const flowStrength = ref(45)
const timeMotion = ref(false)
const motionAmount = ref(30)
const pointer = ref<Point | null>(null)
const timePhase = ref(0)

const patternOptions: Array<{ label: string; value: Pattern }> = [
  { label: 'Twin Plum', value: 'plum' },
  { label: 'Spiral Bloom', value: 'spiral' },
  { label: 'Crystal Vein', value: 'crystal' },
]

const paletteOptions: Array<{ label: string; value: Palette }> = [
  { label: 'Sepia Mist', value: 'mist' },
  { label: 'Copper Dusk', value: 'sunset' },
  { label: 'Ink Night', value: 'aurora' },
]

const patternConfigs: Record<Pattern, PatternConfig> = {
  plum: {
    spread: 0.45,
    twist: 0,
    decayMin: 0.88,
    decayMax: 1.05,
    depthLimit: 70,
    splitChance: 0.62,
    minChildren: 1,
    maxChildren: 2,
  },
  spiral: {
    spread: 0.36,
    twist: 0.15,
    decayMin: 0.86,
    decayMax: 0.98,
    depthLimit: 80,
    splitChance: 0.58,
    minChildren: 1,
    maxChildren: 3,
  },
  crystal: {
    spread: 0.2,
    twist: 0,
    decayMin: 0.9,
    decayMax: 1.03,
    depthLimit: 90,
    splitChance: 0.78,
    minChildren: 2,
    maxChildren: 3,
  },
}

const paletteConfigs: Record<Palette, PaletteConfig> = {
  mist: {
    baseHue: 36,
    hueStep: 2.4,
    saturation: 26,
    alpha: 0.24,
    lightnessDark: [58, 78],
    lightnessLight: [30, 57],
  },
  sunset: {
    baseHue: 16,
    hueStep: 4.2,
    saturation: 70,
    alpha: 0.24,
    lightnessDark: [56, 74],
    lightnessLight: [35, 61],
  },
  aurora: {
    baseHue: 214,
    hueStep: 3.4,
    saturation: 38,
    alpha: 0.2,
    lightnessDark: [58, 76],
    lightnessLight: [29, 50],
  },
}

const tasks: Branch[] = []
let ctx: CanvasRenderingContext2D | null = null
let rafId: number | null = null
let autoRedrawId: number | null = null
let pointerRedrawId: number | null = null
let redrawTimerId: number | null = null
let taskIndex = 0
let drawnSegments = 0
let isMounted = false

const canvasStyle = computed(() => ({
  width: `${width.value}px`,
  height: `${height.value}px`,
}))

const speedLabel = computed(() => `${speed.value} 步/帧`)
const densityLabel = computed(() => `${density.value}%`)
const flowLabel = computed(() => `${flowStrength.value}%`)
const motionLabel = computed(() => `${motionAmount.value}%`)
const patternLabel = computed(() => patternOptions.find((item) => item.value === pattern.value)?.label ?? '')
const paletteLabel = computed(() => paletteOptions.find((item) => item.value === palette.value)?.label ?? '')
const segmentLimit = computed(() => {
  const base = 2200 + density.value * 42
  const timeFactor = timeMotion.value ? 0.65 : 1
  return Math.floor(base * timeFactor)
})
const frameBudget = computed(() => {
  const base = speed.value
  return timeMotion.value ? Math.max(28, Math.floor(base * 0.65)) : base
})

function randomBetween(min: number, max: number) {
  return min + Math.random() * (max - min)
}

function clamp(value: number, min: number, max: number) {
  return Math.min(max, Math.max(min, value))
}

function normalizeAngle(angle: number) {
  let value = angle
  while (value > Math.PI) value -= Math.PI * 2
  while (value < -Math.PI) value += Math.PI * 2
  return value
}

function getPaletteLightness(depth: number, depthLimit: number) {
  const config = paletteConfigs[palette.value]
  const ratio = clamp(depth / depthLimit, 0, 1)
  const range = isDark.value ? config.lightnessDark : config.lightnessLight
  return range[0] + ratio * (range[1] - range[0])
}

function getStrokeColor(depth: number, depthLimit: number, hueOffset: number) {
  const config = paletteConfigs[palette.value]
  const hue = (config.baseHue + depth * config.hueStep + hueOffset + 360) % 360
  const lightness = getPaletteLightness(depth, depthLimit)
  return `hsla(${hue}, ${config.saturation}%, ${lightness}%, ${config.alpha})`
}

function getEndPoint(branch: Branch): Point {
  return {
    x: branch.start.x + branch.length * Math.cos(branch.theta),
    y: branch.start.y + branch.length * Math.sin(branch.theta),
  }
}

function isInBounds(point: Point) {
  return point.x >= 0 && point.x <= width.value && point.y >= 0 && point.y <= height.value
}

function drawLine(start: Point, end: Point, depth: number, depthLimit: number, hueOffset: number) {
  if (!ctx) return
  ctx.beginPath()
  ctx.lineCap = 'round'
  ctx.lineWidth = clamp(2.5 - depth * 0.03 + Math.random() * 0.35, 0.45, 2.7)
  ctx.strokeStyle = getStrokeColor(depth, depthLimit, hueOffset)
  ctx.moveTo(start.x, start.y)
  ctx.lineTo(end.x, end.y)
  ctx.stroke()
}

function clearCanvas() {
  if (!ctx) return
  ctx.clearRect(0, 0, width.value, height.value)
}

function pendingTaskCount() {
  return tasks.length - taskIndex
}

function resetTasks() {
  tasks.length = 0
  taskIndex = 0
  drawnSegments = 0
}

function popTask() {
  if (taskIndex >= tasks.length) return null
  const task = tasks[taskIndex]
  taskIndex += 1

  if (taskIndex > 512 && taskIndex * 2 > tasks.length) {
    tasks.splice(0, taskIndex)
    taskIndex = 0
  }

  return task
}

function queueStep(branch: Branch) {
  if (pendingTaskCount() + drawnSegments >= segmentLimit.value) return
  tasks.push(branch)
}

function getFlowInfluence(start: Point, theta: number) {
  if (!flowEnabled.value || !pointer.value) return 0
  const dx = pointer.value.x - start.x
  const dy = pointer.value.y - start.y
  const distance = Math.hypot(dx, dy)
  const radius = Math.min(width.value, height.value) * 0.6
  if (distance > radius) return 0
  const target = Math.atan2(dy, dx)
  const angleDelta = normalizeAngle(target - theta)
  const falloff = 1 - distance / radius
  return angleDelta * 0.28 * (flowStrength.value / 100) * falloff
}

function getTimeInfluence(depth: number, childIndex: number) {
  if (!timeMotion.value) return 0
  return Math.sin(timePhase.value + depth * 0.16 + childIndex * 0.95) * 0.12 * (motionAmount.value / 100)
}

function step(branch: Branch) {
  if (drawnSegments >= segmentLimit.value || branch.length < 1.6) return

  const config = patternConfigs[pattern.value]
  const end = getEndPoint(branch)

  drawLine(branch.start, end, branch.depth, config.depthLimit, branch.hueOffset)
  drawnSegments += 1

  if (!isInBounds(end) || branch.depth >= config.depthLimit || drawnSegments >= segmentLimit.value) return

  const shouldSplit = branch.depth < 3 || Math.random() < config.splitChance
  if (!shouldSplit) return

  const densityFactor = density.value / 45
  const childCount = Math.floor(randomBetween(config.minChildren, config.maxChildren + 1))

  for (let i = 0; i < childCount; i += 1) {
    const length = clamp(
      branch.length * randomBetween(config.decayMin, config.decayMax) * randomBetween(0.9, 1.08) * densityFactor,
      2,
      Math.max(width.value, height.value) * 0.1
    )

    const swing = randomBetween(-config.spread, config.spread)
    const drift = pattern.value === 'spiral' ? branch.depth * 0.003 : 0
    const field = getFlowInfluence(end, branch.theta)
    const timeBend = getTimeInfluence(branch.depth, i)
    const theta = branch.theta + swing + config.twist + drift + field + timeBend

    queueStep({
      start: end,
      length,
      theta,
      depth: branch.depth + 1,
      hueOffset: branch.hueOffset + randomBetween(-16, 16),
    })
  }
}

function frame() {
  if (timeMotion.value) {
    timePhase.value += 0.02
  }
  let budget = frameBudget.value
  while (budget > 0 && pendingTaskCount() > 0) {
    const task = popTask()
    if (!task) break
    step(task)
    budget -= 1
  }
}

function stopAnimation() {
  if (rafId !== null) {
    cancelAnimationFrame(rafId)
    rafId = null
  }
}

function clearAutoRedraw() {
  if (autoRedrawId !== null) {
    clearTimeout(autoRedrawId)
    autoRedrawId = null
  }
}

function clearPointerRedraw() {
  if (pointerRedrawId !== null) {
    clearTimeout(pointerRedrawId)
    pointerRedrawId = null
  }
}

function clearRedrawTimer() {
  if (redrawTimerId !== null) {
    clearTimeout(redrawTimerId)
    redrawTimerId = null
  }
}

function scheduleRedraw(delay = 90) {
  clearRedrawTimer()
  redrawTimerId = window.setTimeout(() => {
    redrawTimerId = null
    redraw()
  }, delay)
}

function scheduleAutoRedraw() {
  clearAutoRedraw()
  if (!timeMotion.value || !isMounted) return
  autoRedrawId = window.setTimeout(() => {
    autoRedrawId = null
    if (!isMounted) return
    redraw()
  }, 320)
}

function runAnimation() {
  if (rafId !== null) return

  const tick = () => {
    frame()
    if (pendingTaskCount() === 0) {
      rafId = null
      scheduleAutoRedraw()
      return
    }
    rafId = requestAnimationFrame(tick)
  }

  rafId = requestAnimationFrame(tick)
}

function seedBranches() {
  const side = Math.min(width.value, height.value)
  const length = side * 0.04

  if (pattern.value === 'plum') {
    queueStep({
      start: { x: width.value * 0.24, y: height.value },
      length,
      theta: -Math.PI / 2,
      depth: 0,
      hueOffset: -20,
    })
    queueStep({
      start: { x: width.value * 0.76, y: 0 },
      length,
      theta: Math.PI / 2,
      depth: 0,
      hueOffset: 20,
    })
    return
  }

  if (pattern.value === 'spiral') {
    const center = { x: width.value / 2, y: height.value / 2 }
    for (let i = 0; i < 4; i += 1) {
      const theta = (Math.PI / 2) * i
      queueStep({
        start: center,
        length,
        theta,
        depth: 0,
        hueOffset: i * 30,
      })
    }
    return
  }

  const center = { x: width.value / 2, y: height.value / 2 }
  for (let i = 0; i < 6; i += 1) {
    const theta = (Math.PI / 3) * i
    queueStep({
      start: center,
      length: side * 0.03,
      theta,
      depth: 0,
      hueOffset: i * 18,
    })
  }
}

function redraw() {
  if (!ctx) return
  stopAnimation()
  clearAutoRedraw()
  clearRedrawTimer()
  resetTasks()
  clearCanvas()
  seedBranches()
  runAnimation()
}

function schedulePointerRedraw() {
  if (rafId !== null) return
  clearPointerRedraw()
  pointerRedrawId = window.setTimeout(() => {
    pointerRedrawId = null
    if (flowEnabled.value && !timeMotion.value) redraw()
  }, 180)
}

function getCanvasPoint(event: MouseEvent): Point | null {
  if (!canvasRef.value) return null
  const rect = canvasRef.value.getBoundingClientRect()
  if (rect.width <= 0 || rect.height <= 0) return null
  return {
    x: ((event.clientX - rect.left) / rect.width) * width.value,
    y: ((event.clientY - rect.top) / rect.height) * height.value,
  }
}

function onCanvasMove(event: MouseEvent) {
  const next = getCanvasPoint(event)
  pointer.value = next
  if (flowEnabled.value && !timeMotion.value) {
    schedulePointerRedraw()
  }
}

function onCanvasLeave() {
  pointer.value = null
}

function exportPng() {
  if (!canvasRef.value) return
  const source = canvasRef.value
  const temp = document.createElement('canvas')
  temp.width = source.width
  temp.height = source.height
  const tempCtx = temp.getContext('2d')
  if (!tempCtx) return

  tempCtx.fillStyle = isDark.value ? '#050505' : '#ffffff'
  tempCtx.fillRect(0, 0, temp.width, temp.height)
  tempCtx.drawImage(source, 0, 0)

  const link = document.createElement('a')
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-')
  link.download = `plum-${pattern.value}-${timestamp}.png`
  link.href = temp.toDataURL('image/png')
  link.click()
}

function setCanvasSize() {
  const viewportSide = Math.min(window.innerWidth - 36, window.innerHeight - 280, 760)
  const side = clamp(viewportSide, 250, 760)
  width.value = Math.floor(side)
  height.value = Math.floor(side)
}

function syncCanvasResolution() {
  if (!ctx || !canvasRef.value) return
  const dpr = clamp(window.devicePixelRatio || 1, 1, 2)
  canvasRef.value.width = Math.floor(width.value * dpr)
  canvasRef.value.height = Math.floor(height.value * dpr)
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
}

function handleResize() {
  setCanvasSize()
  syncCanvasResolution()
  redraw()
}

onMounted(() => {
  isMounted = true
  const context = canvasRef.value?.getContext('2d')
  if (!context) return
  ctx = context

  setCanvasSize()
  syncCanvasResolution()
  redraw()

  window.addEventListener('resize', handleResize, { passive: true })
})

onUnmounted(() => {
  isMounted = false
  stopAnimation()
  clearAutoRedraw()
  clearPointerRedraw()
  clearRedrawTimer()
  resetTasks()
  window.removeEventListener('resize', handleResize)
})

watch([pattern, palette, flowEnabled, () => isDark.value], redraw)
watch([density, flowStrength, motionAmount], () => scheduleRedraw(120))
watch(timeMotion, (enabled) => {
  if (!enabled) {
    clearAutoRedraw()
  }
  redraw()
})
</script>

<template>
  <section class="poster" :class="{ dark: isDark }">
    <header class="poster-head">
      <p class="catalog">Archive 2026 · Generative Botany</p>
      <h2 class="work-title">{{ patternLabel }}</h2>
      <p class="work-meta">Palette {{ paletteLabel }} · Click canvas to regenerate</p>
    </header>

    <canvas ref="canvasRef" :style="canvasStyle" class="canvas" @click="redraw" @mousemove="onCanvasMove" @mouseleave="onCanvasLeave"></canvas>

    <section class="controls">
      <label>
        Pattern
        <select v-model="pattern">
          <option v-for="item in patternOptions" :key="item.value" :value="item.value">
            {{ item.label }}
          </option>
        </select>
      </label>

      <label>
        Palette
        <select v-model="palette">
          <option v-for="item in paletteOptions" :key="item.value" :value="item.value">
            {{ item.label }}
          </option>
        </select>
      </label>

      <label>
        Density {{ densityLabel }}
        <input v-model.number="density" type="range" min="28" max="72" step="1" />
      </label>

      <label>
        Speed {{ speedLabel }}
        <input v-model.number="speed" type="range" min="40" max="180" step="10" />
      </label>

      <label>
        Flow {{ flowLabel }}
        <input v-model.number="flowStrength" type="range" min="0" max="100" step="1" :disabled="!flowEnabled" />
      </label>

      <label class="toggle-box">
        <span>Flow Field</span>
        <input v-model="flowEnabled" type="checkbox" />
      </label>

      <label>
        Motion {{ motionLabel }}
        <input v-model.number="motionAmount" type="range" min="0" max="100" step="1" :disabled="!timeMotion" />
      </label>

      <label class="toggle-box">
        <span>Time Motion</span>
        <input v-model="timeMotion" type="checkbox" />
      </label>
    </section>

    <footer class="actions">
      <button type="button" @click="redraw">Regenerate</button>
      <button type="button" @click="exportPng">Export PNG</button>
    </footer>
  </section>
</template>

<style scoped>
.poster {
  position: relative;
  max-width: 1040px;
  width: 100%;
  margin: 0 auto;
  display: grid;
  gap: 12px;
  padding: 8px 12px 24px;
}

.poster-head {
  text-align: left;
  padding: 6px 8px 2px;
}

.catalog {
  margin: 0 0 5px;
  color: var(--muted);
  letter-spacing: 0.14em;
  text-transform: uppercase;
  font-size: 0.68rem;
}

.work-title {
  margin: 0;
  font-size: clamp(1.9rem, 4.6vw, 3rem);
  letter-spacing: 0.02em;
  line-height: 0.97;
  font-family: 'Cormorant Garamond', 'Noto Serif SC', 'Source Han Serif SC', serif;
  font-weight: 600;
}

.work-meta {
  margin: 6px 0 0;
  color: var(--muted);
  font-size: 0.82rem;
  letter-spacing: 0.02em;
}

.canvas {
  border: 1px solid #cdb691;
  border-radius: 4px;
  max-width: calc(100vw - 24px);
  background:
    radial-gradient(circle at 23% 19%, #ffffff 0%, #faf2e4 54%, #f0e4ce 100%),
    repeating-linear-gradient(0deg, #00000003 0 1px, transparent 1px 3px);
  box-shadow:
    inset 0 0 0 1px #fffbf2a0,
    0 18px 34px #2f24131c;
  cursor: crosshair;
}

.controls {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 8px 10px;
  padding: 10px 0;
  border-top: 1px solid #d8c8ac;
  border-bottom: 1px solid #d8c8ac;
}

.controls label {
  display: grid;
  gap: 5px;
  font-size: 0.72rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: var(--muted);
}

.controls select,
.controls input[type='range'] {
  width: 100%;
}

.controls select {
  height: 30px;
  border: none;
  border-bottom: 1px solid #c8b392;
  border-radius: 0;
  padding: 0 2px;
  color: inherit;
  background: transparent;
  font-family: 'IBM Plex Sans', 'Noto Sans SC', 'PingFang SC', sans-serif;
}

.controls input[type='range'] {
  accent-color: #b88a44;
}

.controls input[type='checkbox'] {
  width: 16px;
  height: 16px;
  accent-color: #b88a44;
}

.toggle-box {
  grid-template-columns: 1fr auto;
  align-items: center;
}

.actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.actions button {
  min-width: 128px;
  height: 36px;
  border-radius: 999px;
  border: 1px solid #b89a6d;
  background: #fbf5e9;
  color: inherit;
  cursor: pointer;
  text-transform: uppercase;
  letter-spacing: 0.09em;
  font-size: 0.67rem;
}

.actions button:hover {
  background: #fff9ef;
}

.poster.dark .canvas {
  border-color: #5b4b33;
  background:
    radial-gradient(circle at 23% 19%, #1d1812 0%, #17130f 52%, #100d0a 100%),
    repeating-linear-gradient(0deg, #ffffff05 0 1px, transparent 1px 3px);
  box-shadow:
    inset 0 0 0 1px #fff5de14,
    0 18px 34px #00000052;
}

.poster.dark .controls {
  border-color: #4f402d;
}

.poster.dark .controls label,
.poster.dark .catalog,
.poster.dark .work-meta {
  color: #bba98c;
}

.poster.dark .controls select {
  border-bottom-color: #5a4c38;
}

.poster.dark .controls input[type='range'],
.poster.dark .controls input[type='checkbox'] {
  accent-color: #c8a56a;
}

.poster.dark .actions button {
  border-color: #68563d;
  background: #1c1712;
}

.poster.dark .actions button:hover {
  background: #251e16;
}

@media (max-width: 680px) {
  .poster {
    padding-top: 2px;
  }

  .controls {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .actions {
    justify-content: stretch;
    display: grid;
    grid-template-columns: 1fr 1fr;
  }

  .actions button {
    min-width: 0;
  }
}

@media (max-width: 520px) {
  .controls {
    grid-template-columns: 1fr;
  }

  .actions {
    grid-template-columns: 1fr;
  }
}

</style>
