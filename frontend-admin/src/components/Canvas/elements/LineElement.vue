<template>
  <div class="line-element">
    <div class="line" :style="lineStyle"></div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({ element: { type: Object, required: true } })

const lineStyle = computed(() => ({
  height: `${props.element.strokeWidth || 2}px`,
  backgroundColor: props.element.strokeColor || '#000000'
}))
</script>

<style scoped>
.line-element { width: 100%; height: 100%; display: flex; align-items: center; }
.line { width: 100%; }
</style>
