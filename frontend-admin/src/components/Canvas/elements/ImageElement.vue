<template>
  <div class="image-element" @drop="handleDrop" @dragover.prevent>
    <img v-if="element.imageData" :src="element.imageData" alt="图片" />
    <div v-else class="placeholder" @click="triggerUpload">
      <el-icon :size="24"><Picture /></el-icon>
      <span>点击添加</span>
      <input ref="fileInput" type="file" accept="image/*" @change="handleFileChange" style="display: none" />
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useCanvasStore } from '@/stores/canvas'
import { ElMessage } from 'element-plus'

const props = defineProps({ element: { type: Object, required: true } })
const store = useCanvasStore()
const fileInput = ref(null)

const triggerUpload = () => {
  fileInput.value?.click()
}

const handleFileChange = (e) => {
  const file = e.target.files[0]
  if (file) loadImage(file)
}

const handleDrop = (e) => {
  e.preventDefault()
  e.stopPropagation()
  const file = e.dataTransfer.files[0]
  if (file && file.type.startsWith('image/')) {
    loadImage(file)
  }
}

const loadImage = (file) => {
  const reader = new FileReader()
  reader.onload = (e) => {
    store.updateElement(props.element.id, { imageData: e.target.result })
    ElMessage.success('图片已加载')
  }
  reader.readAsDataURL(file)
}
</script>

<style scoped>
.image-element { width: 100%; height: 100%; overflow: hidden; background: #f5f7fa; border-radius: 2px; }
.image-element img { width: 100%; height: 100%; object-fit: contain; }
.placeholder {
  width: 100%; height: 100%; display: flex; flex-direction: column;
  align-items: center; justify-content: center; color: #909399;
  gap: 4px; font-size: 11px; cursor: pointer; border: 1px dashed #dcdfe6;
}
.placeholder:hover { border-color: #409eff; color: #409eff; }
</style>
