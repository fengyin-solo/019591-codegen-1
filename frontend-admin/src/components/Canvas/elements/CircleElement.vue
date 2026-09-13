<template>
  <div class="circle-element" :style="circleStyle"></div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({ element: { type: Object, required: true } })

const circleStyle = computed(() => ({
  width: '100%',
  height: '100%',
  backgroundColor: props.element.fillColor || '#ffffff',
  border: `${props.element.strokeWidth || 1}px solid ${props.element.strokeColor || '#000000'}`,
  borderRadius: '50%'
}))
</script>

<style scoped>
.circle-element { box-sizing: border-box; }
</style>
