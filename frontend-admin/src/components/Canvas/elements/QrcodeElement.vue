<template>
  <div class="qrcode-element">
    <canvas ref="canvasRef"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import QRCode from 'qrcode'

const props = defineProps({ element: { type: Object, required: true } })
const canvasRef = ref(null)

const render = async () => {
  if (!canvasRef.value) return
  try {
    await QRCode.toCanvas(canvasRef.value, props.element.content || 'https://example.com', {
      width: Math.min(props.element.width, props.element.height),
      margin: 2,
      errorCorrectionLevel: props.element.errorLevel || 'M',
      color: { dark: '#000000', light: '#ffffff' }
    })
  } catch (e) {
    console.error('QRCode render error:', e)
  }
}

watch(() => [props.element.content, props.element.errorLevel, props.element.width, props.element.height], render, { deep: true })
onMounted(render)
</script>

<style scoped>
.qrcode-element { width: 100%; height: 100%; display: flex; align-items: center; justify-content: center; background: #fff; }
.qrcode-element canvas { max-width: 100%; max-height: 100%; }
</style>
