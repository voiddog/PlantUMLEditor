<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue'
import plantumlEncoder from 'plantuml-encoder'
import { Codemirror } from 'vue-codemirror'
import { oneDark } from '@codemirror/theme-one-dark'
import { java } from '@codemirror/lang-java'
import panzoom from 'panzoom'

// State
const savedCode = localStorage.getItem('plantuml-code')
const defaultCode = `@startuml
actor User
participant "First Class" as A
participant "Second Class" as B
participant "Last Class" as C

User -> A: DoWork
activate A

A -> B: Create Request
activate B

B -> C: DoWork
activate C
C --> B: WorkDone
destroy C

B --> A: RequestCreated
deactivate B

A --> User: Done
deactivate A
@enduml`

const code = ref(savedCode || defaultCode)

const imgUrl = ref('')
const previewContainer = ref(null)
let panzoomInstance = null

// Resizing logic
const savedWidth = localStorage.getItem('plantuml-editor-width')
const editorWidth = ref(savedWidth ? parseFloat(savedWidth) : 50) // Percentage
const isResizing = ref(false)

const startResize = () => {
  isResizing.value = true
  document.addEventListener('mousemove', onResize)
  document.addEventListener('mouseup', stopResize)
  document.body.style.cursor = 'col-resize'
  document.body.style.userSelect = 'none' // Prevent text selection while dragging
}

const onResize = (e) => {
  if (!isResizing.value) return
  const container = document.querySelector('.glass-content')
  const rect = container.getBoundingClientRect()
  const x = e.clientX - rect.left
  
  let newWidth = (x / rect.width) * 100
  
  // Constraints (min 20%, max 80%)
  if (newWidth < 20) newWidth = 20
  if (newWidth > 80) newWidth = 80
  
  editorWidth.value = newWidth
}

const stopResize = () => {
  isResizing.value = false
  document.removeEventListener('mousemove', onResize)
  document.removeEventListener('mouseup', stopResize)
  document.body.style.cursor = ''
  document.body.style.userSelect = ''
  // Save width preference
  localStorage.setItem('plantuml-editor-width', editorWidth.value)
}

// Debounce logic
let timeout = null
const debounceDelay = 500 // ms

const updatePreview = () => {
  try {
    const encoded = plantumlEncoder.encode(code.value)
    imgUrl.value = `https://www.plantuml.com/plantuml/svg/${encoded}`
  } catch (e) {
    console.error('PlantUML Encoding Error:', e)
  }
}

// Watch code changes with debounce
watch(code, (newVal) => {
  // Save code changes
  localStorage.setItem('plantuml-code', newVal)

  if (timeout) clearTimeout(timeout)
  timeout = setTimeout(() => {
    updatePreview()
  }, debounceDelay)
}, { immediate: true })

onMounted(() => {
  if (previewContainer.value) {
    // Initialize panzoom on the image container or the image itself
    // We'll target a wrapper inside
    const imgWrapper = previewContainer.value.querySelector('.img-wrapper')
    if (imgWrapper) {
       panzoomInstance = panzoom(imgWrapper, {
         maxZoom: 5,
         minZoom: 0.1,
         bounds: true,
         boundsPadding: 0.1
       })
    }
  }
})

onUnmounted(() => {
  // Cleanup listeners just in case
  document.removeEventListener('mousemove', onResize)
  document.removeEventListener('mouseup', stopResize)
})

// Extensions for CodeMirror
const extensions = [oneDark, java()]

</script>

<template>
  <div class="layout">
    <header class="glass-header">
      <h1>PlantUML Editor</h1>
    </header>
    
    <main class="glass-content">
      <div class="editor-pane glass-panel" :style="{ width: editorWidth + '%' }">
        <div class="panel-header">Code</div>
        <div class="editor-wrapper">
          <Codemirror
            v-model="code"
            placeholder="Write PlantUML code here..."
            :style="{ height: '100%' }"
            :autofocus="true"
            :indent-with-tab="true"
            :tab-size="2"
            :extensions="extensions"
          />
        </div>
      </div>

      <div class="resizer" @mousedown="startResize"></div>

      <div class="preview-pane glass-panel" :style="{ width: (100 - editorWidth) + '%' }">
        <div class="panel-header">Preview</div>
        <div class="preview-viewport" ref="previewContainer">
          <div class="img-wrapper">
            <img :src="imgUrl" alt="PlantUML Diagram" @error="console.error('Image Load Error')" />
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
.layout {
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding: 20px;
  gap: 20px;
}

.glass-header {
  background: var(--glass-bg);
  backdrop-filter: blur(var(--glass-blur));
  -webkit-backdrop-filter: blur(var(--glass-blur));
  border: 1px solid var(--glass-border);
  border-radius: 16px;
  padding: 10px 20px;
  box-shadow: var(--glass-shadow);
  color: var(--text-color);
  flex-shrink: 0;
}

.glass-header h1 {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 600;
  letter-spacing: 1px;
}

.glass-content {
  display: flex;
  flex: 1;
  gap: 0; /* Remove gap, handled by resizer/padding if needed, but here we want flush for resizing control */
  min-height: 0;
  position: relative;
}

.glass-panel {
  /* flex: 1; removed in favor of explicit width */
  background: var(--glass-bg);
  backdrop-filter: blur(var(--glass-blur));
  -webkit-backdrop-filter: blur(var(--glass-blur));
  border: 1px solid var(--glass-border);
  border-radius: 16px;
  box-shadow: var(--glass-shadow);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  transition: width 0.1s ease-out; /* Smooth resizing visual, optional, might conflict with drag */
}

/* Disable transition during drag for performance/responsiveness */
.resizer:active ~ .glass-panel,
.glass-panel:has(+ .resizer:active) {
  transition: none;
}
/* Note: :has support is good but fallback or JS class is safer. 
   For simplicity, we can remove the transition on the panel entirely if it feels laggy, 
   or bind a class :class="{ 'no-transition': isResizing }" 
*/
.glass-panel {
  transition: none; /* Let's disable for instant feedback */
}


.resizer {
  width: 10px;
  cursor: col-resize;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
  margin: 0 5px; /* Visual spacing */
}

.resizer::after {
  content: '';
  width: 4px;
  height: 40px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 2px;
}

.resizer:hover::after, .resizer:active::after {
  background: rgba(255, 255, 255, 0.5);
}

.panel-header {
  padding: 10px 15px;
  background: rgba(255, 255, 255, 0.05);
  border-bottom: 1px solid var(--glass-border);
  font-weight: 500;
  color: var(--accent-color);
  text-transform: uppercase;
  font-size: 0.85rem;
  letter-spacing: 1px;
}

.editor-wrapper {
  flex: 1;
  overflow: hidden; 
}

/* Customize codemirror to fit glass theme */
:deep(.cm-editor) {
  height: 100%;
  background: transparent;
}
:deep(.cm-scroller) {
  font-family: 'Fira Code', monospace;
  font-size: 14px;
}
:deep(.cm-gutters) {
  background: rgba(0, 0, 0, 0.2);
  border-right: 1px solid var(--glass-border);
  color: rgba(255, 255, 255, 0.5);
}

.preview-viewport {
  flex: 1;
  overflow: hidden; /* Hide overflow for panzoom context */
  position: relative;
  background: rgba(0, 0, 0, 0.1);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: grab;
}

.preview-viewport:active {
  cursor: grabbing;
}

.img-wrapper {
  /* panzoom transforms this */
  display: flex;
  align-items: center;
  justify-content: center;
}

img {
  max-width: none; /* Allow full size for zooming */
  display: block;
}
</style>
