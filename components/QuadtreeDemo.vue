<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'

const canvasRef = ref<HTMLCanvasElement | null>(null)
const stats = ref('Cells: 1 · Depth: 0 · Roads: 0')

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
  const canvas = canvasRef.value
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

function getPos(e: PointerEvent) {
  const canvas = canvasRef.value!
  const rect = canvas.getBoundingClientRect()
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  return {
    x: (e.clientX - rect.left) * scaleX,
    y: (e.clientY - rect.top) * scaleY,
  }
}

function beginDraw(e: PointerEvent) {
  if (activePointerId !== null) return
  activePointerId = e.pointerId
  canvasRef.value?.setPointerCapture(e.pointerId)
  e.preventDefault()
  startPt = getPos(e)
  currentPt = startPt
  drawing = true
  render()
}

function updateDraw(e: PointerEvent) {
  if (!drawing || e.pointerId !== activePointerId) return
  e.preventDefault()
  currentPt = getPos(e)
  render()
}

function endDraw(e: PointerEvent) {
  if (!drawing || e.pointerId !== activePointerId) return
  e.preventDefault()
  const end = getPos(e)
  if (startPt) {
    const dist = Math.hypot(end.x - startPt.x, end.y - startPt.y)
    if (dist > 8) insertRoad(startPt.x, startPt.y, end.x, end.y)
  }
  canvasRef.value?.releasePointerCapture(e.pointerId)
  activePointerId = null
  drawing = false
  startPt = null
  currentPt = null
  render()
}

function cancelDraw(e: PointerEvent) {
  if (activePointerId !== null && e.pointerId !== activePointerId) return
  if (activePointerId !== null)
    canvasRef.value?.releasePointerCapture(activePointerId)
  activePointerId = null
  drawing = false
  startPt = null
  currentPt = null
  render()
}

function reset() {
  const canvas = canvasRef.value
  if (!canvas) return
  root = { x: 0, y: 0, w: canvas.width, h: canvas.height, depth: 0, children: null }
  roads = []
  drawing = false
  activePointerId = null
  startPt = null
  currentPt = null
  render()
}

function resizeCanvas() {
  const canvas = canvasRef.value
  const wrap = canvas?.parentElement
  if (!canvas || !wrap) return
  const w = Math.floor(wrap.clientWidth)
  const h = Math.max(180, Math.min(300, Math.floor(w * 0.52)))
  if (canvas.width === w && canvas.height === h) return
  canvas.width = w
  canvas.height = h
  reset()
}

let ro: ResizeObserver | null = null

onMounted(() => {
  const canvas = canvasRef.value
  if (!canvas) return
  ctx = canvas.getContext('2d')
  resizeCanvas()
  ro = new ResizeObserver(resizeCanvas)
  ro.observe(canvas.parentElement!)
})

onUnmounted(() => ro?.disconnect())
</script>

<template>
  <div
    class="quadtree-demo"
    @pointerdown.stop
    @pointermove.stop
    @pointerup.stop
    @pointercancel.stop
    @touchstart.stop
    @touchmove.stop
    @touchend.stop
  >
    <div class="quadtree-header">
      <span class="quadtree-title">PM Quadtree — Draw a Road</span>
      <span class="quadtree-stats">{{ stats }}</span>
    </div>
    <canvas
      ref="canvasRef"
      class="quadtree-canvas"
      @pointerdown="beginDraw"
      @pointermove="updateDraw"
      @pointerup="endDraw"
      @pointercancel="cancelDraw"
      @pointerleave="cancelDraw"
    />
    <div class="quadtree-footer">
      <span>Tap &amp; drag to draw a road</span>
      <button type="button" @click.stop="reset">Reset</button>
    </div>
  </div>
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
  white-space: nowrap;
}

.quadtree-canvas {
  display: block;
  width: 100%;
  height: auto;
  border: 1px solid #30363d;
  border-radius: 4px;
  background: #161b22;
  cursor: crosshair;
  touch-action: none;
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
  padding: 4px 10px;
  border-radius: 4px;
  font-size: 10px;
  font-family: inherit;
  touch-action: manipulation;
}
</style>
