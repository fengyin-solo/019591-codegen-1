<template>
  <div class="app-container">
    <HeaderToolbar @export="handleExport" />
    <div class="main-content">
      <ElementPanel />
      <CanvasArea ref="canvasAreaRef" />
      <PropertyPanel />
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import HeaderToolbar from './components/Toolbar/HeaderToolbar.vue'
import ElementPanel from './components/Elements/ElementPanel.vue'
import CanvasArea from './components/Canvas/CanvasArea.vue'
import PropertyPanel from './components/Properties/PropertyPanel.vue'

const canvasAreaRef = ref(null)

const handleExport = (type) => {
  if (canvasAreaRef.value) {
    canvasAreaRef.value.exportCanvas(type)
  }
}
</script>

<style lang="scss" scoped>
.app-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  background: #f0f2f5;
}

.main-content {
  flex: 1;
  display: flex;
  overflow: hidden;
  padding: 16px;
  gap: 16px;
}
</style>
