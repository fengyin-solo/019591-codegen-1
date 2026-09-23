<template>
  <canvas ref="canvasRef" class="canvas-ruler" />
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount, nextTick } from 'vue'
import { useCanvasStore } from '@/stores/canvas'

const props = defineProps({
  // 'h' 顶部横向刻度尺, 'v' 左侧纵向刻度尺
  direction: { type: String, default: 'h' },
  // 返回画布原点(0,0)在屏幕上的坐标 { x, y }，用于滚动/缩放后对齐刻度
  getCanvasOrigin: { type: Function, required: true }
})

const store = useCanvasStore()
const canvasRef = ref(null)
let resizeObserver = null

const draw = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  const cssW = canvas.clientWidth
  const cssH = canvas.clientHeight
  if (!cssW || !cssH) return

  const dpr = window.devicePixelRatio || 1
  const pixelW = Math.round(cssW * dpr)
  const pixelH = Math.round(cssH * dpr)
  if (canvas.width !== pixelW || canvas.height !== pixelH) {
    canvas.width = pixelW
    canvas.height = pixelH
  }

  const ctx = canvas.getContext('2d')
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.clearRect(0, 0, cssW, cssH)

  const isH = props.direction === 'h'
  const scale = store.scale
  const mmToDot = store.MM_TO_DOT
  const totalMm = isH ? store.canvasWidth : store.canvasHeight
  const totalPx = isH ? store.canvasPixelWidth : store.canvasPixelHeight
  const limit = isH ? cssW : cssH

  // 画布原点相对刻度尺原点的屏幕偏移（滚动/缩放后仍保持同步）
  const origin = props.getCanvasOrigin()
  const rect = canvas.getBoundingClientRect()
  const start = isH ? origin.x - rect.left : origin.y - rect.top

  // 标签间隔：保证缩放后标签不重叠
  const labelSteps = [1, 2, 5, 10, 20, 50, 100, 200]
  const labelStep = labelSteps.find(s => s * mmToDot * scale >= 40) || 200
  const showMinor = mmToDot * scale >= 3
  const showMid = 5 * mmToDot * scale >= 10

  ctx.strokeStyle = '#c0c4cc'
  ctx.fillStyle = '#909399'
  ctx.font = '9px sans-serif'
  ctx.lineWidth = 1

  for (let mm = 0; mm <= totalMm; mm++) {
    // 刻度按像素计算位置，并限制在画布范围内
    const px = mm * mmToDot
    if (px > totalPx) break
    const pos = Math.round(start + px * scale) + 0.5
    if (pos < 0 || pos > limit) continue

    const isLabel = mm % labelStep === 0
    const isMid = !isLabel && showMid && mm % 5 === 0
    if (!isLabel && !isMid && !showMinor) continue

    const len = isLabel ? 10 : isMid ? 7 : 4
    ctx.beginPath()
    if (isH) {
      ctx.moveTo(pos, cssH)
      ctx.lineTo(pos, cssH - len)
    } else {
      ctx.moveTo(cssW, pos)
      ctx.lineTo(cssW - len, pos)
    }
    ctx.stroke()

    if (isLabel && pos >= 8 && pos <= limit - 8) {
      if (isH) {
        ctx.textAlign = 'center'
        ctx.textBaseline = 'top'
        ctx.fillText(String(mm), pos, 1)
      } else {
        ctx.save()
        ctx.translate(cssW / 2, pos)
        ctx.rotate(-Math.PI / 2)
        ctx.textAlign = 'center'
        ctx.textBaseline = 'middle'
        ctx.fillText(String(mm), 0, 0)
        ctx.restore()
      }
    }
  }
}

// 滚动画面时由父组件调用，保持刻度与元件位置同步
const redraw = () => draw()

watch(
  [() => store.scale, () => store.canvasWidth, () => store.canvasHeight],
  () => nextTick(draw)
)

onMounted(() => {
  resizeObserver = new ResizeObserver(() => draw())
  resizeObserver.observe(canvasRef.value)
  draw()
})

onBeforeUnmount(() => {
  if (resizeObserver) resizeObserver.disconnect()
})

defineExpose({ redraw })
</script>

<style lang="scss" scoped>
.canvas-ruler {
  display: block;
  width: 100%;
  height: 100%;
  background: #f5f7fa;
}
</style>
