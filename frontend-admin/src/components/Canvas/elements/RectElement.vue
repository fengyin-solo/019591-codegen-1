<template>
  <div class="rect-element" :style="rectStyle"></div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({ element: { type: Object, required: true } })

const rectStyle = computed(() => ({
  width: '100%',
  height: '100%',
  backgroundColor: props.element.fillColor || '#ffffff',
  border: `${props.element.strokeWidth || 1}px solid ${props.element.strokeColor || '#000000'}`
}))
</script>

<style scoped>
.rect-element { box-sizing: border-box; }
</style>
