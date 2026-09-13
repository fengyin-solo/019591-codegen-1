<template>
  <div class="barcode-element">
    <canvas ref="canvasRef"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import JsBarcode from 'jsbarcode'

const props = defineProps({ element: { type: Object, required: true } })
const canvasRef = ref(null)

const render = () => {
  if (!canvasRef.value) return
  try {
    JsBarcode(canvasRef.value, props.element.content || '123456789', {
      format: props.element.format || 'CODE128',
      displayValue: props.element.showText !== false,
      width: 2,
      height: Math.max(30, props.element.height - 20),
      margin: 10,
      background: '#ffffff',
      lineColor: '#000000'
    })
  } catch (e) {
    console.error('Barcode render error:', e)
  }
}

watch(() => [props.element.content, props.element.format, props.element.showText], render, { deep: true })
onMounted(render)
</script>

<style scoped>
.barcode-element { width: 100%; height: 100%; display: flex; align-items: center; justify-content: center; background: #fff; overflow: hidden; }
.barcode-element canvas { max-width: 100%; max-height: 100%; }
</style>
