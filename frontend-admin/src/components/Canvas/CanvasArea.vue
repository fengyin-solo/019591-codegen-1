<template>
  <div class="canvas-area card">
    <div class="canvas-stage">
      <div class="canvas-wrapper" ref="wrapperRef" @click.self="clearSelection" @drop="handleDrop" @dragover="handleDragOver">
        <div class="canvas-container" ref="containerRef" :style="canvasContainerStyle">
          <canvas ref="canvasRef" :width="store.canvasPixelWidth" :height="store.canvasPixelHeight" class="export-canvas" />
          <div class="edit-area">
            <div
              v-for="element in visibleElements"
              :key="element.id"
              class="canvas-element"
              :class="{ selected: isSelected(element.id), 'multi-selected': isMultiSelected(element.id) }"
              :style="getElementStyle(element)"
              @mousedown="handleElementMouseDown($event, element)"
              @dblclick="handleDoubleClick(element)"
            >
              <component :is="getElementComponent(element.type)" :element="element" :ref="el => setElementRef(element.id, el)" />
              <div v-if="isSelected(element.id)" class="resize-handles">
                <div v-for="handle in resizeHandles" :key="handle" :class="['resize-handle', handle]" @mousedown.stop="startResize($event, element, handle)" />
              </div>
            </div>
          </div>
          <!-- 吸附参考线：坐标与画布像素坐标系一致，仅编辑期显示，不参与导出 -->
          <div v-if="showGuides" class="snap-guides">
            <div
              v-for="(gx, i) in guides.v"
              :key="'gv' + i"
              class="snap-guide snap-guide-v"
              :style="{ left: gx + 'px', width: guideThickness + 'px' }"
            />
            <div
              v-for="(gy, i) in guides.h"
              :key="'gh' + i"
              class="snap-guide snap-guide-h"
              :style="{ top: gy + 'px', height: guideThickness + 'px' }"
            />
          </div>
        </div>
      </div>

      <!-- 毫米刻度尺（固定覆盖在滚动区上，尺面随滚动/缩放平移） -->
      <div class="ruler ruler-corner"><span>mm</span></div>
      <div class="ruler ruler-h">
        <div class="ruler-track" ref="rulerHTrackRef" :style="{ width: rulerTrackWidth + 'px' }">
          <div
            v-for="(tick, i) in horizontalTicks"
            :key="'ht' + i"
            class="ruler-tick"
            :class="tick.level"
            :style="{ left: tick.pos + 'px' }"
          >
            <span v-if="tick.label !== ''" class="ruler-label">{{ tick.label }}</span>
          </div>
        </div>
      </div>
      <div class="ruler ruler-v">
        <div class="ruler-track" ref="rulerVTrackRef" :style="{ height: rulerTrackHeight + 'px' }">
          <div
            v-for="(tick, i) in verticalTicks"
            :key="'vt' + i"
            class="ruler-tick"
            :class="tick.level"
            :style="{ top: tick.pos + 'px' }"
          >
            <span v-if="tick.label !== ''" class="ruler-label">{{ tick.label }}</span>
          </div>
        </div>
      </div>
    </div>
    <div class="canvas-info">
      <span>画布: {{ store.canvasWidth }}mm × {{ store.canvasHeight }}mm</span>
      <span>像素: {{ store.canvasPixelWidth }} × {{ store.canvasPixelHeight }} px（1mm = {{ store.MM_TO_DOT }}px）</span>
      <span class="tip">提示: Ctrl+点击多选元件，拖动/缩放时靠近边线或中线自动吸附</span>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted, onBeforeUnmount, watch } from 'vue'
import { useCanvasStore } from '@/stores/canvas'
import TextElement from './elements/TextElement.vue'
import RectElement from './elements/RectElement.vue'
import CircleElement from './elements/CircleElement.vue'
import LineElement from './elements/LineElement.vue'
import ImageElement from './elements/ImageElement.vue'
import BarcodeElement from './elements/BarcodeElement.vue'
import QrcodeElement from './elements/QrcodeElement.vue'
import TableElement from './elements/TableElement.vue'
import JsBarcode from 'jsbarcode'
import QRCode from 'qrcode'
import { ElMessage } from 'element-plus'

const store = useCanvasStore()
const canvasRef = ref(null)
const wrapperRef = ref(null)
const containerRef = ref(null)
const rulerHTrackRef = ref(null)
const rulerVTrackRef = ref(null)
const elementRefs = ref({})

const RULER_SIZE = 20
// 吸附判定阈值（屏幕像素），除以缩放比换算为画布像素，保证任何比例下手感一致
const SNAP_THRESHOLD_SCREEN_PX = 6

const resizeHandles = ['nw', 'n', 'ne', 'e', 'se', 's', 'sw', 'w']
let isDragging = false
let isResizing = false
let dragStartX = 0
let dragStartY = 0
let elementStartX = 0
let elementStartY = 0
let elementStartW = 0
let elementStartH = 0
let currentHandle = ''
let currentElementId = null

const visibleElements = computed(() => store.elements.filter(el => el.visible))

const canvasContainerStyle = computed(() => ({
  width: `${store.canvasPixelWidth}px`,
  height: `${store.canvasPixelHeight}px`,
  transform: `scale(${store.scale})`,
  transformOrigin: 'top left'
}))

const componentMap = { text: TextElement, rect: RectElement, circle: CircleElement, line: LineElement, image: ImageElement, barcode: BarcodeElement, qrcode: QrcodeElement, table: TableElement }
const getElementComponent = (type) => componentMap[type] || 'div'

const setElementRef = (id, el) => { if (el) elementRefs.value[id] = el }

const getElementStyle = (el) => ({
  left: `${el.x}px`,
  top: `${el.y}px`,
  width: `${el.width}px`,
  height: `${el.height}px`,
  transform: el.rotation ? `rotate(${el.rotation}deg)` : undefined
})

const isSelected = (id) => store.selectedElementId === id
const isMultiSelected = (id) => store.selectedElementIds.includes(id) && store.selectedElementIds.length > 1
const clearSelection = () => store.clearSelection()

// ============ 毫米刻度尺 ============
// 尺面（屏幕像素）长度，刻度只画到画布边界为止
const rulerTrackWidth = computed(() => store.canvasPixelWidth * store.scale)
const rulerTrackHeight = computed(() => store.canvasPixelHeight * store.scale)

/**
 * 按像素位置生成毫米刻度：
 * 刻度位置 = 毫米数 × MM_TO_DOT（画布像素）× scale（屏幕比例），
 * 与元件坐标共用同一套 mm→px 口径，刻度范围不超过画布尺寸。
 */
function buildTicks(canvasMm, scale) {
  const pxPerMm = store.MM_TO_DOT * scale
  // 数字刻度间距：保证屏幕上至少约 40px，避免缩小后文字挤在一起
  const labelCandidates = [2, 5, 10, 20, 50, 100]
  let labelStep = 100
  for (const c of labelCandidates) {
    if (c * pxPerMm >= 40) { labelStep = c; break }
  }
  // 小刻度：优先 1mm，屏幕上小于 3px 时退化为 2mm/5mm
  let minorStep = labelStep >= 10 ? labelStep / 10 : 1
  if (minorStep * pxPerMm < 3) minorStep = labelStep / 5
  if (minorStep * pxPerMm < 3) minorStep = labelStep / 2

  const divisions = Math.round(labelStep / minorStep)
  const totalMinor = Math.floor(canvasMm / minorStep)
  const trackLength = canvasMm * store.MM_TO_DOT * scale
  const ticks = []
  for (let i = 0; i <= totalMinor; i++) {
    const mm = i * minorStep
    const pos = mm * store.MM_TO_DOT * scale
    if (pos > trackLength + 0.001) break // 限制在画布内
    let level = 'minor'
    let label = ''
    if (i % divisions === 0) {
      level = 'major'
      label = String(Math.round(mm))
    } else if (divisions % 2 === 0 && i % (divisions / 2) === 0) {
      level = 'mid'
    }
    ticks.push({ pos, label, level })
  }
  return ticks
}

const horizontalTicks = computed(() => buildTicks(store.canvasWidth, store.scale))
const verticalTicks = computed(() => buildTicks(store.canvasHeight, store.scale))

// 根据滚动位置与画布在视口中的实际位置平移尺面
let rulerRafId = 0
const updateRulers = () => {
  rulerRafId = 0
  const wrapper = wrapperRef.value
  const container = containerRef.value
  if (!wrapper || !container) return
  const wr = wrapper.getBoundingClientRect()
  const cr = container.getBoundingClientRect()
  const offsetX = cr.left - wr.left - RULER_SIZE
  const offsetY = cr.top - wr.top - RULER_SIZE
  if (rulerHTrackRef.value) rulerHTrackRef.value.style.transform = `translateX(${offsetX}px)`
  if (rulerVTrackRef.value) rulerVTrackRef.value.style.transform = `translateY(${offsetY}px)`
}
const scheduleRulerUpdate = () => {
  if (!rulerRafId) rulerRafId = requestAnimationFrame(updateRulers)
}

let resizeObserver = null
onMounted(() => {
  nextTick(updateRulers)
  wrapperRef.value?.addEventListener('scroll', scheduleRulerUpdate, { passive: true })
  window.addEventListener('resize', scheduleRulerUpdate)
  resizeObserver = new ResizeObserver(scheduleRulerUpdate)
  if (wrapperRef.value) resizeObserver.observe(wrapperRef.value)
  if (containerRef.value) resizeObserver.observe(containerRef.value)
})

onBeforeUnmount(() => {
  if (rulerRafId) cancelAnimationFrame(rulerRafId)
  wrapperRef.value?.removeEventListener('scroll', scheduleRulerUpdate)
  window.removeEventListener('resize', scheduleRulerUpdate)
  resizeObserver?.disconnect()
})

// 切换比例 / 画布尺寸后，尺面刻度与偏移量重新同步
watch(
  () => [store.scale, store.canvasPixelWidth, store.canvasPixelHeight],
  () => nextTick(scheduleRulerUpdate)
)

// ============ 吸附参考线 ============
const showGuides = ref(false)
const guides = ref({ v: [], h: [] })
// 参考线画在被 scale 的画布容器内，按比例反向补偿线宽，保证屏幕上始终为 1px
const guideThickness = computed(() => 1 / store.scale)

// 吸附目标：画布边线/中线 + 其他可见元件的边线/中心线（画布像素坐标）
function collectSnapTargets(excludeId) {
  const xs = [0, store.canvasPixelWidth / 2, store.canvasPixelWidth]
  const ys = [0, store.canvasPixelHeight / 2, store.canvasPixelHeight]
  for (const other of store.elements) {
    if (!other.visible || other.id === excludeId) continue
    xs.push(other.x, other.x + other.width, other.x + other.width / 2)
    ys.push(other.y, other.y + other.height, other.y + other.height / 2)
  }
  return { xs, ys }
}

// 在阈值内找差值最小的一组对齐线，返回需要补偿的位移 delta 与目标线坐标
function findSnap(movingLines, targetLines, threshold) {
  let best = null
  for (const moving of movingLines) {
    for (const target of targetLines) {
      const delta = target - moving
      if (Math.abs(delta) <= threshold && (!best || Math.abs(delta) < Math.abs(best.delta))) {
        best = { delta, target }
      }
    }
  }
  return best
}

const handleDoubleClick = (element) => {
  if (element.type === 'text') {
    const ref = elementRefs.value[element.id]
    if (ref && ref.startEdit) ref.startEdit()
  }
}

const handleElementMouseDown = (e, element) => {
  if (element.locked) return
  e.preventDefault()
  store.selectElement(element.id, e.ctrlKey || e.metaKey)
  isDragging = true
  currentElementId = element.id
  dragStartX = e.clientX
  dragStartY = e.clientY
  elementStartX = element.x
  elementStartY = element.y
  document.addEventListener('mousemove', handleMouseMove)
  document.addEventListener('mouseup', handleMouseUp)
}

const startResize = (e, element, handle) => {
  e.preventDefault()
  isResizing = true
  currentHandle = handle
  currentElementId = element.id
  dragStartX = e.clientX
  dragStartY = e.clientY
  elementStartX = element.x
  elementStartY = element.y
  elementStartW = element.width
  elementStartH = element.height
  document.addEventListener('mousemove', handleMouseMove)
  document.addEventListener('mouseup', handleMouseUp)
}

const handleMouseMove = (e) => {
  const dx = (e.clientX - dragStartX) / store.scale
  const dy = (e.clientY - dragStartY) / store.scale
  const el = store.elements.find(item => item.id === currentElementId)
  if (!el) return

  const threshold = SNAP_THRESHOLD_SCREEN_PX / store.scale
  const targets = collectSnapTargets(currentElementId)
  const nextGuides = { v: [], h: [] }

  if (isDragging && currentElementId) {
    // 限制不超出画布
    let newX = Math.max(0, Math.min(store.canvasPixelWidth - el.width, Math.round(elementStartX + dx)))
    let newY = Math.max(0, Math.min(store.canvasPixelHeight - el.height, Math.round(elementStartY + dy)))

    // 只在判定为吸附时修正落点，并画出对应参考线
    const snapX = findSnap([newX, newX + el.width, newX + el.width / 2], targets.xs, threshold)
    if (snapX) {
      newX = Math.max(0, Math.min(store.canvasPixelWidth - el.width, Math.round(newX + snapX.delta)))
      nextGuides.v.push(snapX.target)
    }
    const snapY = findSnap([newY, newY + el.height, newY + el.height / 2], targets.ys, threshold)
    if (snapY) {
      newY = Math.max(0, Math.min(store.canvasPixelHeight - el.height, Math.round(newY + snapY.delta)))
      nextGuides.h.push(snapY.target)
    }
    store.updateElement(currentElementId, { x: newX, y: newY })
  } else if (isResizing && currentElementId) {
    let newX = elementStartX, newY = elementStartY, newW = elementStartW, newH = elementStartH

    if (currentHandle.includes('e')) newW = Math.max(10, elementStartW + dx)
    if (currentHandle.includes('w')) { newW = Math.max(10, elementStartW - dx); newX = elementStartX + dx }
    if (currentHandle.includes('s')) newH = Math.max(10, elementStartH + dy)
    if (currentHandle.includes('n')) { newH = Math.max(10, elementStartH - dy); newY = elementStartY + dy }

    // 限制不超出画布
    newX = Math.max(0, Math.round(newX))
    newY = Math.max(0, Math.round(newY))
    newW = Math.min(store.canvasPixelWidth - newX, Math.round(newW))
    newH = Math.min(store.canvasPixelHeight - newY, Math.round(newH))

    // 横向：仅被拖动的那条边及中心参与吸附，吸附只调整该边落点
    if (currentHandle.includes('w') || currentHandle.includes('e')) {
      const isWest = currentHandle.includes('w')
      const movingLines = isWest ? [newX, newX + newW / 2] : [newX + newW, newX + newW / 2]
      const snap = findSnap(movingLines, targets.xs, threshold)
      if (snap) {
        if (isWest) {
          const candX = Math.round(newX + snap.delta)
          const candW = Math.round(newW - snap.delta)
          if (candX >= 0 && candW >= 10) { newX = candX; newW = candW; nextGuides.v.push(snap.target) }
        } else {
          const candW = Math.round(newW + snap.delta)
          if (candW >= 10 && newX + candW <= store.canvasPixelWidth) { newW = candW; nextGuides.v.push(snap.target) }
        }
      }
    }
    // 纵向
    if (currentHandle.includes('n') || currentHandle.includes('s')) {
      const isNorth = currentHandle.includes('n')
      const movingLines = isNorth ? [newY, newY + newH / 2] : [newY + newH, newY + newH / 2]
      const snap = findSnap(movingLines, targets.ys, threshold)
      if (snap) {
        if (isNorth) {
          const candY = Math.round(newY + snap.delta)
          const candH = Math.round(newH - snap.delta)
          if (candY >= 0 && candH >= 10) { newY = candY; newH = candH; nextGuides.h.push(snap.target) }
        } else {
          const candH = Math.round(newH + snap.delta)
          if (candH >= 10 && newY + candH <= store.canvasPixelHeight) { newH = candH; nextGuides.h.push(snap.target) }
        }
      }
    }

    store.updateElement(currentElementId, { x: newX, y: newY, width: newW, height: newH })
  }

  // 画布里没有元件（或未进行拖动/缩放）时不画参考线
  guides.value = nextGuides
  showGuides.value = store.elements.some(item => item.visible)
}

const handleMouseUp = () => {
  isDragging = false
  isResizing = false
  currentElementId = null
  showGuides.value = false
  guides.value = { v: [], h: [] }
  document.removeEventListener('mousemove', handleMouseMove)
  document.removeEventListener('mouseup', handleMouseUp)
}

const handleDragOver = (e) => {
  e.preventDefault()
  e.dataTransfer.dropEffect = 'copy'
}

const handleDrop = (e) => {
  e.preventDefault()
  e.stopPropagation()
  const data = e.dataTransfer.getData('application/json')
  if (!data) return

  try {
    const item = JSON.parse(data)
    const container = containerRef.value
    const rect = container.getBoundingClientRect()
    let x = (e.clientX - rect.left) / store.scale
    let y = (e.clientY - rect.top) / store.scale

    const defaultSize = {
      text: { width: 100, height: 24 },
      rect: { width: 80, height: 60 },
      circle: { width: 60, height: 60 },
      line: { width: 100, height: 4 },
      image: { width: 80, height: 80 },
      barcode: { width: 150, height: 60 },
      qrcode: { width: 80, height: 80 },
      table: { width: 200, height: 120 }
    }

    const size = defaultSize[item.type] || { width: 100, height: 40 }

    // 计算位置并限制在画布内
    x = Math.max(0, Math.min(store.canvasPixelWidth - size.width, Math.round(x - size.width / 2)))
    y = Math.max(0, Math.min(store.canvasPixelHeight - size.height, Math.round(y - size.height / 2)))

    store.addElement({
      type: item.type,
      ...item.defaultProps,
      x, y,
      ...size
    })
  } catch (err) {
    console.error('Drop error:', err)
  }
}

const renderCanvas = async () => {
  await nextTick()
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  ctx.fillStyle = '#ffffff'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  for (const el of store.elements) {
    if (!el.visible) continue
    ctx.save()
    ctx.translate(el.x + el.width / 2, el.y + el.height / 2)
    if (el.rotation) ctx.rotate((el.rotation * Math.PI) / 180)
    ctx.translate(-el.width / 2, -el.height / 2)
    await renderElement(ctx, el)
    ctx.restore()
  }
}

const renderElement = async (ctx, el) => {
  switch (el.type) {
    case 'text':
      ctx.fillStyle = el.color || '#000'
      ctx.font = `${el.italic ? 'italic ' : ''}${el.bold ? 'bold ' : ''}${el.fontSize || 14}px ${el.fontFamily || 'Arial'}`
      ctx.textBaseline = 'top'
      ctx.fillText(el.content || '', 0, 0)
      break
    case 'rect':
      if (el.fillColor && el.fillColor !== 'transparent') { ctx.fillStyle = el.fillColor; ctx.fillRect(0, 0, el.width, el.height) }
      if (el.strokeWidth) { ctx.strokeStyle = el.strokeColor || '#000'; ctx.lineWidth = el.strokeWidth; ctx.strokeRect(0, 0, el.width, el.height) }
      break
    case 'circle':
      ctx.beginPath()
      ctx.ellipse(el.width / 2, el.height / 2, el.width / 2, el.height / 2, 0, 0, Math.PI * 2)
      if (el.fillColor && el.fillColor !== 'transparent') { ctx.fillStyle = el.fillColor; ctx.fill() }
      if (el.strokeWidth) { ctx.strokeStyle = el.strokeColor || '#000'; ctx.lineWidth = el.strokeWidth; ctx.stroke() }
      break
    case 'line':
      ctx.beginPath(); ctx.moveTo(0, el.height / 2); ctx.lineTo(el.width, el.height / 2)
      ctx.strokeStyle = el.strokeColor || '#000'; ctx.lineWidth = el.strokeWidth || 2; ctx.stroke()
      break
    case 'image':
      if (el.imageData) {
        const img = new Image(); img.src = el.imageData
        await new Promise(r => { img.onload = r; img.onerror = r })
        ctx.drawImage(img, 0, 0, el.width, el.height)
      }
      break
    case 'barcode':
      try {
        const bcCanvas = document.createElement('canvas')
        JsBarcode(bcCanvas, el.content || '123456', { format: el.format || 'CODE128', displayValue: el.showText !== false })
        ctx.drawImage(bcCanvas, 0, 0, el.width, el.height)
      } catch (e) { console.error(e) }
      break
    case 'qrcode':
      try {
        const qrCanvas = document.createElement('canvas')
        await QRCode.toCanvas(qrCanvas, el.content || 'https://example.com', { width: el.width, errorCorrectionLevel: el.errorLevel || 'M' })
        ctx.drawImage(qrCanvas, 0, 0, el.width, el.height)
      } catch (e) { console.error(e) }
      break
    case 'table': {
      const rows = el.rows || 3
      const cols = el.cols || 3
      const bw = el.borderWidth || 1
      const bc = el.borderColor || '#000000'
      const cellW = el.width / cols
      const cellH = el.height / rows
      const padding = 4
      ctx.strokeStyle = bc
      ctx.lineWidth = bw
      ctx.strokeRect(bw / 2, bw / 2, el.width - bw, el.height - bw)
      for (let r = 1; r < rows; r++) {
        ctx.beginPath()
        ctx.moveTo(0, r * cellH)
        ctx.lineTo(el.width, r * cellH)
        ctx.stroke()
      }
      for (let c = 1; c < cols; c++) {
        ctx.beginPath()
        ctx.moveTo(c * cellW, 0)
        ctx.lineTo(c * cellW, el.height)
        ctx.stroke()
      }
      const fontSize = el.cellFontSize || 12
      const fontFamily = el.cellFontFamily || 'Arial'
      const textAlign = el.cellTextAlign || 'center'
      ctx.fillStyle = el.cellFontColor || '#000000'
      ctx.font = `${fontSize}px ${fontFamily}`
      ctx.textAlign = textAlign
      ctx.textBaseline = 'middle'
      for (let r = 0; r < rows; r++) {
        for (let c = 0; c < cols; c++) {
          const text = (el.cells && el.cells[r] && el.cells[r][c]) || ''
          if (text) {
            let x
            if (textAlign === 'left') x = c * cellW + padding
            else if (textAlign === 'right') x = (c + 1) * cellW - padding
            else x = c * cellW + cellW / 2
            const y = r * cellH + cellH / 2
            ctx.fillText(text, x, y)
          }
        }
      }
      break
    }
  }
}

const exportToBMP = (canvas, filename = 'label.bmp') => {
  const ctx = canvas.getContext('2d')
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  const pixels = imageData.data
  const width = canvas.width
  const height = canvas.height
  const rowSize = Math.ceil(width / 8)
  const paddedRowSize = Math.ceil(rowSize / 4) * 4
  const pixelDataSize = paddedRowSize * height
  const headerSize = 62
  const fileSize = headerSize + pixelDataSize
  const buffer = new ArrayBuffer(fileSize)
  const view = new DataView(buffer)
  view.setUint8(0, 0x42); view.setUint8(1, 0x4D)
  view.setUint32(2, fileSize, true); view.setUint32(6, 0, true); view.setUint32(10, headerSize, true)
  view.setUint32(14, 40, true); view.setInt32(18, width, true); view.setInt32(22, -height, true)
  view.setUint16(26, 1, true); view.setUint16(28, 1, true); view.setUint32(30, 0, true)
  view.setUint32(34, pixelDataSize, true); view.setUint32(38, 2835, true); view.setUint32(42, 2835, true)
  view.setUint32(46, 2, true); view.setUint32(50, 2, true)
  view.setUint8(54, 0); view.setUint8(55, 0); view.setUint8(56, 0); view.setUint8(57, 0)
  view.setUint8(58, 255); view.setUint8(59, 255); view.setUint8(60, 255); view.setUint8(61, 0)
  let offset = headerSize
  for (let y = 0; y < height; y++) {
    let byte = 0, bitIndex = 7
    for (let x = 0; x < width; x++) {
      const i = (y * width + x) * 4
      const gray = 0.299 * pixels[i] + 0.587 * pixels[i+1] + 0.114 * pixels[i+2]
      byte |= ((gray > 128 ? 1 : 0) << bitIndex); bitIndex--
      if (bitIndex < 0 || x === width - 1) { view.setUint8(offset++, byte); byte = 0; bitIndex = 7 }
    }
    for (let p = Math.ceil(width / 8); p < paddedRowSize; p++) view.setUint8(offset++, 0)
  }
  const blob = new Blob([buffer], { type: 'image/bmp' })
  const link = document.createElement('a')
  link.href = URL.createObjectURL(blob); link.download = filename; link.click()
}

const exportCanvas = async (type) => {
  await renderCanvas()
  const canvas = canvasRef.value
  if (type === 'bmp') {
    exportToBMP(canvas, 'label.bmp')
    ElMessage.success('BMP 导出成功')
  } else {
    const link = document.createElement('a')
    link.href = canvas.toDataURL('image/png')
    link.download = 'label.png'
    link.click()
    ElMessage.success('PNG 导出成功')
  }
}

defineExpose({ exportCanvas })
</script>

<style lang="scss" scoped>
.canvas-area { flex: 1; display: flex; flex-direction: column; overflow: hidden; min-width: 0; }

.canvas-stage { position: relative; flex: 1; min-height: 0; display: flex; }

.canvas-wrapper {
  flex: 1; overflow: auto; background: #e4e7ed;
  background-image: linear-gradient(45deg, #d0d0d0 25%, transparent 25%), linear-gradient(-45deg, #d0d0d0 25%, transparent 25%), linear-gradient(45deg, transparent 75%, #d0d0d0 75%), linear-gradient(-45deg, transparent 75%, #d0d0d0 75%);
  background-size: 20px 20px; background-position: 0 0, 0 10px, 10px -10px, -10px 0px;
  padding: 24px; display: flex; justify-content: flex-start; align-items: flex-start;
}

.canvas-container { position: relative; background: #fff; box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15); overflow: hidden; flex-shrink: 0; }
.export-canvas { position: absolute; top: 0; left: 0; visibility: hidden; pointer-events: none; }
.edit-area { position: relative; width: 100%; height: 100%; }

.canvas-element {
  position: absolute; cursor: move; border: 1px solid transparent; box-sizing: border-box;
  &:hover { border-color: #409eff; }
  &.selected { border-color: #409eff; border-width: 2px; }
  &.multi-selected { border-color: #67c23a; border-width: 2px; }
}

.resize-handles .resize-handle {
  position: absolute; width: 8px; height: 8px; background: #409eff; border: 1px solid #fff; border-radius: 2px;
  &.nw { top: -4px; left: -4px; cursor: nw-resize; }
  &.n { top: -4px; left: 50%; transform: translateX(-50%); cursor: n-resize; }
  &.ne { top: -4px; right: -4px; cursor: ne-resize; }
  &.e { top: 50%; right: -4px; transform: translateY(-50%); cursor: e-resize; }
  &.se { bottom: -4px; right: -4px; cursor: se-resize; }
  &.s { bottom: -4px; left: 50%; transform: translateX(-50%); cursor: s-resize; }
  &.sw { bottom: -4px; left: -4px; cursor: sw-resize; }
  &.w { top: 50%; left: -4px; transform: translateY(-50%); cursor: w-resize; }
}

// 吸附参考线
.snap-guides { position: absolute; inset: 0; pointer-events: none; z-index: 20; }
.snap-guide { position: absolute; background: #f5222d; }
.snap-guide-v { top: 0; bottom: 0; }
.snap-guide-h { left: 0; right: 0; }

// 毫米刻度尺
.ruler {
  position: absolute; z-index: 10; pointer-events: none;
  background: #f5f7fa; overflow: hidden;
  color: #909399; font-size: 9px; line-height: 1;
}
.ruler-corner {
  top: 0; left: 0; width: 20px; height: 20px; z-index: 11;
  border-right: 1px solid #dcdfe6; border-bottom: 1px solid #dcdfe6;
  display: flex; align-items: center; justify-content: center;
  span { font-size: 8px; color: #c0c4cc; }
}
.ruler-h { top: 0; left: 20px; right: 0; height: 20px; border-bottom: 1px solid #dcdfe6; }
.ruler-v { left: 0; top: 20px; bottom: 0; width: 20px; border-right: 1px solid #dcdfe6; }

.ruler-track { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }

.ruler-h .ruler-tick {
  position: absolute; top: 0; width: 1px; background: #c0c4cc;
  &.minor { height: 4px; }
  &.mid { height: 7px; }
  &.major { height: 9px; background: #909399; }
}
.ruler-h .ruler-label {
  position: absolute; top: 10px; left: 2px; white-space: nowrap; color: #909399;
}

.ruler-v .ruler-tick {
  position: absolute; left: 0; height: 1px; width: 4px; background: #c0c4cc;
  &.mid { width: 7px; }
  &.major { width: 9px; background: #909399; }
}
.ruler-v .ruler-label {
  position: absolute; left: 0; top: 2px; width: 20px; text-align: center; white-space: nowrap; color: #909399;
}

.canvas-info {
  padding: 8px 16px; background: #f5f7fa; border-top: 1px solid #e4e7ed;
  display: flex; gap: 24px; font-size: 12px; color: #909399;
  .tip { margin-left: auto; color: #409eff; }
}
</style>
