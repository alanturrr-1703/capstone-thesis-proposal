<script setup lang="ts">
import { nextTick, onMounted, onUnmounted, ref, watch } from 'vue'

const canvasRef = ref<HTMLCanvasElement | null>(null)
const overlayCanvasRef = ref<HTMLCanvasElement | null>(null)
const stats = ref('Cells: 1 · Depth: 0 · Roads: 0')
const drawMode = ref(false)
const isTouch = ref(false)

const MAX_DEPTH = 6
const MIN_SIZE = 12

type Cell = {
  x: number
  y: number
  w: number
  h: number
  depth: number
  children: Cell[] | null
}

type Road = { x1: number; y1: number; x2: number; y2: number }

let root: Cell | null = null
let roads: Road[] = []
let drawing = false
let startPt: { x: number; y: number } | null = null
let currentPt: { x: number; y: number } | null = null
let ctx: CanvasRenderingContext2D | null = null
let activePointerId: number | null = null
let activeTouchId: number | null = null
let boundCanvas: HTMLCanvasElement | null = null

function cross(px: number, py: number, qx: number, qy: number, rx: number, ry: number) {
  return (qx - px) * (ry - py) - (qy - py) * (rx - px)
}

function segmentsIntersect(ax: number, ay: number, bx: number, by: number, cx: number, cy: number, dx: number, dy: number) {
  const d1 = cross(cx, cy, dx, dy, ax, ay, bx, by)
  const d2 = cross(cx, cy, dx, dy, bx, by, ax, ay)
  const d3 = cross(ax, ay, bx, by, cx, cy, dx, dy)
  const d4 = cross(ax, ay, bx, by, dx, dy, cx, cy)
  return ((d1 > 0 && d2 < 0) || (d1 < 0 && d2 > 0)) &&
    ((d3 > 0 && d4 < 0) || (d3 < 0 && d4 > 0))
}

function lineIntersectsRect(x1: number, y1: number, x2: number, y2: number, rx: number, ry: number, rw: number, rh: number) {
  const left = rx; const right = rx + rw; const top = ry; const bottom = ry + rh
  if (Math.min(x1, x2) > right || Math.max(x1, x2) < left ||
    Math.min(y1, y2) > bottom || Math.max(y1, y2) < top) return false
  const edges = [
    [left, top, right, top], [right, top, right, bottom],
    [right, bottom, left, bottom], [left, bottom, left, top],
  ]
  for (const [ex1, ey1, ex2, ey2] of edges) {
    if (segmentsIntersect(x1, y1, x2, y2, ex1, ey1, ex2, ey2)) return true
  }
  return x1 >= left && x1 <= right && y1 >= top && y1 <= bottom
}

function subdivide(node: Cell) {
  if (node.children || node.depth >= MAX_DEPTH || node.w < MIN_SIZE * 2 || node.h < MIN_SIZE * 2) return
  const hw = node.w / 2; const hh = node.h / 2
  node.children = [
    { x: node.x, y: node.y, w: hw, h: hh, depth: node.depth + 1, children: null },
    { x: node.x + hw, y: node.y, w: hw, h: hh, depth: node.depth + 1, children: null },
    { x: node.x, y: node.y + hh, w: hw, h: hh, depth: node.depth + 1, children: null },
    { x: node.x + hw, y: node.y + hh, w: hw, h: hh, depth: node.depth + 1, children: null },
  ]
}

function subdivideForRoad(node: Cell, x1: number, y1: number, x2: number, y2: number) {
  if (!lineIntersectsRect(x1, y1, x2, y2, node.x, node.y, node.w, node.h)) return
  subdivide(node)
  node.children?.forEach(c => subdivideForRoad(c, x1, y1, x2, y2))
}

function insertRoad(x1: number, y1: number, x2: number, y2: number) {
  roads.push({ x1, y1, x2, y2 })
  if (root) subdivideForRoad(root, x1, y1, x2, y2)
}

function countCells(node: Cell): number {
  if (!node.children) return 1
  return node.children.reduce((s, c) => s + countCells(c), 0)
}

function maxDepth(node: Cell): number {
  if (!node.children) return node.depth
  return Math.max(...node.children.map(maxDepth))
}

function activeCanvas() {
  return drawMode.value ? overlayCanvasRef.value : canvasRef.value
}

function drawNode(node: Cell) {
  if (!ctx) return
  const colors = ['#21262d', '#1a2332', '#152238', '#101d2e', '#0d1828', '#0a1420', '#070f18']
  const color = colors[Math.min(node.depth, colors.length - 1)]
  ctx.fillStyle = color
  ctx.fillRect(node.x + 0.5, node.y + 0.5, node.w - 1, node.h - 1)
  ctx.strokeStyle = `rgba(88, 166, 255, ${0.15 + node.depth * 0.08})`
  ctx.lineWidth = 1
  ctx.strokeRect(node.x + 0.5, node.y + 0.5, node.w - 1, node.h - 1)
  node.children?.forEach(drawNode)
}

function render() {
  const canvas = activeCanvas()
  if (!ctx || !canvas || !root) return
  ctx.clearRect(0, 0, canvas.width, canvas.height)
  drawNode(root)
  ctx.lineCap = 'round'
  for (const r of roads) {
    ctx.strokeStyle = '#f0883e'
    ctx.lineWidth = 3
    ctx.beginPath()
    ctx.moveTo(r.x1, r.y1)
    ctx.lineTo(r.x2, r.y2)
    ctx.stroke()
    ctx.fillStyle = '#f0883e'
    ctx.beginPath()
    ctx.arc(r.x1, r.y1, 3, 0, Math.PI * 2)
    ctx.arc(r.x2, r.y2, 3, 0, Math.PI * 2)
    ctx.fill()
  }
  if (drawing && startPt && currentPt) {
    ctx.strokeStyle = 'rgba(240, 136, 62, 0.6)'
    ctx.lineWidth = 2
    ctx.setLineDash([4, 4])
    ctx.beginPath()
    ctx.moveTo(startPt.x, startPt.y)
    ctx.lineTo(currentPt.x, currentPt.y)
    ctx.stroke()
    ctx.setLineDash([])
  }
  stats.value = `Cells: ${countCells(root)} · Depth: ${maxDepth(root)} · Roads: ${roads.length}`
}

function getPosFromClient(clientX: number, clientY: number) {
  const canvas = activeCanvas()!
  const rect = canvas.getBoundingClientRect()
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  return {
    x: (clientX - rect.left) * scaleX,
    y: (clientY - rect.top) * scaleY,
  }
}

function finishDraw(end: { x: number; y: number }) {
  if (startPt) {
    const dist = Math.hypot(end.x - startPt.x, end.y - startPt.y)
    if (dist > 8) insertRoad(startPt.x, startPt.y, end.x, end.y)
  }
  drawing = false
  startPt = null
  currentPt = null
  activePointerId = null
  activeTouchId = null
  render()
}

function blockEvent(e: Event) {
  e.preventDefault()
  e.stopPropagation()
}

function beginDrawAt(pt: { x: number; y: number }) {
  startPt = pt
  currentPt = pt
  drawing = true
  render()
}

function updateDrawAt(pt: { x: number; y: number }) {
  if (!drawing) return
  currentPt = pt
  render()
}

function onPointerDown(e: PointerEvent) {
  if (activeTouchId !== null || activePointerId !== null) return
  blockEvent(e)
  activePointerId = e.pointerId
  try { boundCanvas?.setPointerCapture(e.pointerId) } catch { /* iOS */ }
  beginDrawAt(getPosFromClient(e.clientX, e.clientY))
}

function onPointerMove(e: PointerEvent) {
  if (!drawing || e.pointerId !== activePointerId) return
  blockEvent(e)
  updateDrawAt(getPosFromClient(e.clientX, e.clientY))
}

function onPointerUp(e: PointerEvent) {
  if (!drawing || e.pointerId !== activePointerId) return
  blockEvent(e)
  try { boundCanvas?.releasePointerCapture(e.pointerId) } catch { /* iOS */ }
  finishDraw(getPosFromClient(e.clientX, e.clientY))
}

function findTouch(e: TouchEvent) {
  if (activeTouchId === null) return e.changedTouches[0] ?? e.touches[0]
  for (const t of e.changedTouches) {
    if (t.identifier === activeTouchId) return t
  }
  for (const t of e.touches) {
    if (t.identifier === activeTouchId) return t
  }
  return null
}

function onTouchStart(e: TouchEvent) {
  if (activePointerId !== null || e.touches.length !== 1) return
  blockEvent(e)
  const t = e.touches[0]
  activeTouchId = t.identifier
  beginDrawAt(getPosFromTouch(t))
}

function getPosFromTouch(t: Touch) {
  return getPosFromClient(t.clientX, t.clientY)
}

function onTouchMove(e: TouchEvent) {
  if (activeTouchId === null) return
  blockEvent(e)
  const t = findTouch(e) ?? e.touches[0]
  if (t) updateDrawAt(getPosFromTouch(t))
}

function onTouchEnd(e: TouchEvent) {
  if (activeTouchId === null) return
  blockEvent(e)
  const t = findTouch(e)
  if (t) finishDraw(getPosFromTouch(t))
  else if (currentPt) finishDraw(currentPt)
}

function onDocTouchMove(e: TouchEvent) {
  if (activeTouchId !== null && drawing) onTouchMove(e)
}

function onDocTouchEnd(e: TouchEvent) {
  if (activeTouchId !== null && drawing) onTouchEnd(e)
}

function reset() {
  const canvas = activeCanvas()
  if (!canvas) return
  root = { x: 0, y: 0, w: canvas.width, h: canvas.height, depth: 0, children: null }
  roads = []
  drawing = false
  activePointerId = null
  activeTouchId = null
  startPt = null
  currentPt = null
  render()
}

function sizeCanvas(container: HTMLElement, fullscreen = false) {
  const canvas = activeCanvas()
  if (!canvas) return
  const w = Math.floor(container.clientWidth)
  const h = fullscreen
    ? Math.floor(Math.min(window.innerHeight * 0.5, w * 0.7))
    : Math.max(160, Math.min(260, Math.floor(w * 0.52)))
  const savedRoads = [...roads]
  canvas.width = w
  canvas.height = h
  canvas.style.width = `${w}px`
  canvas.style.height = `${h}px`
  root = { x: 0, y: 0, w, h, depth: 0, children: null }
  roads = []
  for (const r of savedRoads) insertRoad(r.x1, r.y1, r.x2, r.y2)
  drawing = false
  activePointerId = null
  activeTouchId = null
  startPt = null
  currentPt = null
  render()
}

function bindCanvas(canvas: HTMLCanvasElement) {
  if (boundCanvas === canvas) return
  if (boundCanvas) unbindCanvas(boundCanvas)
  boundCanvas = canvas
  ctx = canvas.getContext('2d')
  const opts = { passive: false, capture: true } as AddEventListenerOptions
  canvas.addEventListener('pointerdown', onPointerDown, opts)
  canvas.addEventListener('pointermove', onPointerMove, opts)
  canvas.addEventListener('pointerup', onPointerUp, opts)
  canvas.addEventListener('pointercancel', onPointerUp, opts)
  canvas.addEventListener('touchstart', onTouchStart, opts)
  canvas.addEventListener('touchmove', onTouchMove, opts)
  canvas.addEventListener('touchend', onTouchEnd, opts)
  canvas.addEventListener('touchcancel', onTouchEnd, opts)
}

function unbindCanvas(canvas: HTMLCanvasElement) {
  const opts = { capture: true } as EventListenerOptions
  canvas.removeEventListener('pointerdown', onPointerDown, opts)
  canvas.removeEventListener('pointermove', onPointerMove, opts)
  canvas.removeEventListener('pointerup', onPointerUp, opts)
  canvas.removeEventListener('pointercancel', onPointerUp, opts)
  canvas.removeEventListener('touchstart', onTouchStart, opts)
  canvas.removeEventListener('touchmove', onTouchMove, opts)
  canvas.removeEventListener('touchend', onTouchEnd, opts)
  canvas.removeEventListener('touchcancel', onTouchEnd, opts)
  if (boundCanvas === canvas) boundCanvas = null
}

function setupActiveCanvas() {
  const canvas = activeCanvas()
  const container = drawMode.value
    ? document.getElementById('quadtree-draw-overlay')
    : canvas?.parentElement
  if (!canvas || !container) return
  bindCanvas(canvas)
  sizeCanvas(container, drawMode.value)
}

async function openDrawMode() {
  drawMode.value = true
  document.body.style.overflow = 'hidden'
  await nextTick()
  setupActiveCanvas()
}

async function closeDrawMode() {
  drawMode.value = false
  document.body.style.overflow = ''
  await nextTick()
  setupActiveCanvas()
}

let ro: ResizeObserver | null = null

watch(drawMode, async () => {
  await nextTick()
  setupActiveCanvas()
})

onMounted(() => {
  isTouch.value = 'ontouchstart' in window || navigator.maxTouchPoints > 0
  document.addEventListener('touchmove', onDocTouchMove, { passive: false })
  document.addEventListener('touchend', onDocTouchEnd, { passive: false })
  document.addEventListener('touchcancel', onDocTouchEnd, { passive: false })
  nextTick(() => setupActiveCanvas())
  ro = new ResizeObserver(() => {
    const container = drawMode.value
      ? document.getElementById('quadtree-draw-overlay')
      : canvasRef.value?.parentElement
    if (container && activeCanvas()) sizeCanvas(container, drawMode.value)
  })
  if (canvasRef.value?.parentElement) ro.observe(canvasRef.value.parentElement)
})

onUnmounted(() => {
  document.body.style.overflow = ''
  if (boundCanvas) unbindCanvas(boundCanvas)
  document.removeEventListener('touchmove', onDocTouchMove)
  document.removeEventListener('touchend', onDocTouchEnd)
  document.removeEventListener('touchcancel', onDocTouchEnd)
  ro?.disconnect()
})
</script>

<template>
  <div class="quadtree-demo">
    <div class="quadtree-header">
      <span class="quadtree-title">PM Quadtree — Draw a Road</span>
      <span class="quadtree-stats">{{ stats }}</span>
    </div>

    <button
      v-if="isTouch"
      type="button"
      class="draw-mode-btn"
      @click.stop.prevent="openDrawMode"
    >
      Tap here to draw (iPhone)
    </button>

    <canvas v-show="!isTouch" ref="canvasRef" class="quadtree-canvas" />

    <p v-if="isTouch" class="touch-hint">
      Slidev blocks touch on the small canvas. Use the button above for fullscreen draw mode.
    </p>

    <div class="quadtree-footer">
      <span>{{ isTouch ? 'Fullscreen draw mode' : 'Click & drag to draw' }}</span>
      <button type="button" @click.stop.prevent="reset">Reset</button>
    </div>
  </div>

  <Teleport to="body">
    <div
      v-if="drawMode"
      id="quadtree-draw-overlay"
      class="draw-overlay"
    >
      <div class="draw-overlay-header">
        <span>Drag finger to draw road</span>
        <button type="button" class="done-btn" @click.stop.prevent="closeDrawMode">Done</button>
      </div>
      <div class="draw-overlay-stats">{{ stats }}</div>
      <canvas ref="overlayCanvasRef" class="quadtree-canvas overlay-canvas" />
      <button type="button" class="reset-overlay-btn" @click.stop.prevent="reset">Reset</button>
    </div>
  </Teleport>
</template>

<style scoped>
.quadtree-demo {
  font-family: 'SF Mono', 'Fira Code', 'Consolas', monospace;
  background: #0f1419;
  border: 1px solid #30363d;
  border-radius: 6px;
  padding: 10px;
  touch-action: none;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}

.quadtree-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
  gap: 8px;
}

.quadtree-title {
  font-size: 11px;
  font-weight: 600;
  color: #58a6ff;
}

.quadtree-stats {
  font-size: 10px;
  color: #8b949e;
}

.draw-mode-btn {
  display: block;
  width: 100%;
  margin-bottom: 8px;
  padding: 14px;
  background: #238636;
  border: none;
  border-radius: 8px;
  color: #fff;
  font-size: 15px;
  font-weight: 700;
  font-family: inherit;
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

.touch-hint {
  font-size: 10px;
  color: #8b949e;
  margin: 6px 0;
  line-height: 1.4;
}

.quadtree-canvas {
  display: block;
  border: 1px solid #30363d;
  border-radius: 4px;
  background: #161b22;
  touch-action: none;
  -webkit-tap-highlight-color: transparent;
}

.quadtree-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 6px;
  font-size: 10px;
  color: #6e7681;
}

.quadtree-footer button {
  background: #21262d;
  border: 1px solid #30363d;
  color: #c9d1d9;
  padding: 8px 14px;
  border-radius: 4px;
  font-size: 12px;
  font-family: inherit;
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}
</style>

<style>
.draw-overlay {
  position: fixed;
  inset: 0;
  z-index: 99999;
  background: #0d1117;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: max(16px, env(safe-area-inset-top)) 16px max(16px, env(safe-area-inset-bottom));
  touch-action: none;
  overscroll-behavior: none;
}

.draw-overlay-header {
  width: 100%;
  max-width: 600px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  font-family: system-ui, sans-serif;
  font-size: 15px;
  color: #c9d1d9;
}

.draw-overlay-stats {
  width: 100%;
  max-width: 600px;
  font-family: monospace;
  font-size: 12px;
  color: #8b949e;
  margin-bottom: 10px;
}

.draw-overlay .overlay-canvas {
  width: calc(100% - 32px) !important;
  max-width: 600px;
  border: 2px solid #388bfd !important;
  border-radius: 8px;
  touch-action: none;
  -webkit-tap-highlight-color: transparent;
}

.done-btn {
  background: #388bfd;
  border: none;
  color: #fff;
  padding: 10px 18px;
  border-radius: 8px;
  font-size: 15px;
  font-weight: 600;
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

.reset-overlay-btn {
  margin-top: 14px;
  background: #21262d;
  border: 1px solid #30363d;
  color: #c9d1d9;
  padding: 12px 24px;
  border-radius: 8px;
  font-size: 14px;
  touch-action: manipulation;
}
</style>
