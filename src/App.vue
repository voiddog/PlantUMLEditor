<script setup>
import { ref, watch, onMounted, onUnmounted, nextTick } from 'vue'
import plantumlEncoder from 'plantuml-encoder'
import { Codemirror } from 'vue-codemirror'
import { oneDark } from '@codemirror/theme-one-dark'
import { java } from '@codemirror/lang-java'
import panzoom from 'panzoom'

// === Constants ===
const STORAGE_KEY_FILES = 'plantuml-files'
const STORAGE_KEY_ACTIVE = 'plantuml-active-id'
const STORAGE_KEY_WIDTH = 'plantuml-editor-width'

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

// === State ===
const files = ref([])
const activeFileId = ref(null)
const code = ref('') // Bound to CodeMirror
const imgUrl = ref('')
const previewContainer = ref(null)
const renameInput = ref(null)
let panzoomInstance = null

// === Initialization ===
const initFiles = () => {
  const savedFiles = localStorage.getItem(STORAGE_KEY_FILES)
  if (savedFiles) {
    try {
      files.value = JSON.parse(savedFiles)
    } catch (e) {
      console.error('Failed to parse saved files', e)
      files.value = []
    }
  }

  // Fallback / Migration
  if (files.value.length === 0) {
    const oldCode = localStorage.getItem('plantuml-code')
    files.value = [{
      id: Date.now().toString(),
      name: 'Diagram 1',
      content: oldCode || defaultCode,
      isRenaming: false
    }]
  }

  // Set Active File
  const savedActive = localStorage.getItem(STORAGE_KEY_ACTIVE)
  if (savedActive && files.value.find(f => f.id === savedActive)) {
    activeFileId.value = savedActive
  } else {
    activeFileId.value = files.value[0].id
  }
}

initFiles()

// === File Management Actions ===

const selectFile = (id) => {
  activeFileId.value = id
}

const addFile = () => {
  const newFile = {
    id: Date.now().toString(),
    name: `Diagram ${files.value.length + 1}`,
    content: defaultCode,
    isRenaming: false
  }
  files.value.push(newFile)
  selectFile(newFile.id)
}

const deleteFile = (id) => {
  if (!confirm('Are you sure you want to delete this diagram?')) return

  const index = files.value.findIndex(f => f.id === id)
  if (index === -1) return

  files.value.splice(index, 1)

  // If we deleted the last file, create a new default one
  if (files.value.length === 0) {
    addFile()
    return
  }

  // If we deleted the active file, switch to another
  if (activeFileId.value === id) {
    // Try to go to the previous one, or the first one
    const newIndex = Math.max(0, index - 1)
    activeFileId.value = files.value[newIndex].id
  }
}

const startRenaming = (file) => {
  file.isRenaming = true
  nextTick(() => {
    // Focus the input
    // Note: v-for ref logic in Vue 3 implies renameInput will be an array
    if (renameInput.value && Array.isArray(renameInput.value)) {
       // Find the input that corresponds to this file?
       // Actually simpler: Since only one is editing, focus the last mounted input or iterate.
       const input = renameInput.value.find(el => el.value === file.name) || renameInput.value[renameInput.value.length-1]
       if(input) input.focus()
    }
  })
}

const finishRenaming = (file) => {
  if (!file.name.trim()) {
    file.name = 'Untitled'
  }
  file.isRenaming = false
}

// === Reactive Logic ===

// 1. Sync Active File -> Editor Code
watch(activeFileId, (newId) => {
  const file = files.value.find(f => f.id === newId)
  if (file) {
    code.value = file.content
    localStorage.setItem(STORAGE_KEY_ACTIVE, newId)
  }
}, { immediate: true })

// 2. Sync Editor Code -> Active File Content
watch(code, (newCode) => {
  const file = files.value.find(f => f.id === activeFileId.value)
  if (file && file.content !== newCode) {
    file.content = newCode
  }
})

// 3. Persist Files (Debounced optional, but deep watch is fine for local text)
watch(files, (newFiles) => {
  // Strip transient state like 'isRenaming' before saving?
  // Ideally yes, but lazy way is saving it and ignoring on load.
  // Let's just save.
  localStorage.setItem(STORAGE_KEY_FILES, JSON.stringify(newFiles))
}, { deep: true })


// === Preview Logic (Debounced) ===
let timeout = null
const debounceDelay = 500

const updatePreview = () => {
  try {
    const encoded = plantumlEncoder.encode(code.value)
    imgUrl.value = `https://www.plantuml.com/plantuml/svg/${encoded}`
  } catch (e) {
    console.error('PlantUML Encoding Error:', e)
  }
}

watch(code, () => {
  if (timeout) clearTimeout(timeout)
  timeout = setTimeout(updatePreview, debounceDelay)
}, { immediate: true })


// === Layout & Resizing ===
const editorWidth = ref(localStorage.getItem(STORAGE_KEY_WIDTH) ? parseFloat(localStorage.getItem(STORAGE_KEY_WIDTH)) : 50)
const isResizing = ref(false)

const startResize = () => {
  isResizing.value = true
  document.addEventListener('mousemove', onResize)
  document.addEventListener('mouseup', stopResize)
  document.body.style.cursor = 'col-resize'
  document.body.style.userSelect = 'none'
}

const onResize = (e) => {
  if (!isResizing.value) return
  const container = document.querySelector('.glass-content')
  const rect = container.getBoundingClientRect()
  const x = e.clientX - rect.left
  let newWidth = (x / rect.width) * 100
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
    localStorage.setItem(STORAGE_KEY_WIDTH, editorWidth.value)
  }
  
  // === Lifecycle ===
  onMounted(() => {
    if (previewContainer.value) {
      const imgWrapper = previewContainer.value.querySelector('.img-wrapper')
      if (imgWrapper) {
         panzoomInstance = panzoom(imgWrapper, {
           maxZoom: 5,
           minZoom: 0.1,
           // zoomOnDoubleClick: false // Default is true, so we can just remove this line or set it to true explicitly if preferred.
           // bounds: true, // Removed to allow full panning regardless of container
           // boundsPadding: 0.1
         })
      }
    }
  })
  onUnmounted(() => {
  document.removeEventListener('mousemove', onResize)
  document.removeEventListener('mouseup', stopResize)
})

const extensions = [oneDark, java()]
</script>

<template>
  <div class="layout">
    <header class="glass-header">
      <h1>PlantUML Editor</h1>
    </header>

    <main class="glass-content">
      <div class="editor-pane glass-panel" :style="{ width: editorWidth + '%' }">
        <!-- New Tabs Header -->
        <div class="panel-header tabs-header">
          <div
            v-for="file in files"
            :key="file.id"
            class="tab-item"
            :class="{ active: file.id === activeFileId }"
            @click="selectFile(file.id)"
            @dblclick="startRenaming(file)"
          >
            <input
              v-if="file.isRenaming"
              v-model="file.name"
              @blur="finishRenaming(file)"
              @keyup.enter="finishRenaming(file)"
              ref="renameInput"
              class="rename-input"
            />
            <span v-else class="tab-name">{{ file.name }}</span>
            <span class="close-btn" @click.stop="deleteFile(file.id)">×</span>
          </div>
          <div class="add-btn" @click="addFile" title="New Diagram">+</div>
        </div>

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
  gap: 0;
  min-height: 0;
  position: relative;
}

.glass-panel {
  background: var(--glass-bg);
  backdrop-filter: blur(var(--glass-blur));
  -webkit-backdrop-filter: blur(var(--glass-blur));
  border: 1px solid var(--glass-border);
  border-radius: 16px;
  box-shadow: var(--glass-shadow);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.resizer {
  width: 10px;
  cursor: col-resize;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
  margin: 0 5px;
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
  height: 40px; /* Explicit height to match tabs-header */
  padding: 0 15px; /* Adjusted padding since we have explicit height */
  background: rgba(255, 255, 255, 0.05);
  border-bottom: 1px solid var(--glass-border);
  font-weight: 500;
  color: var(--accent-color);
  text-transform: uppercase;
  font-size: 0.85rem;
  letter-spacing: 1px;
  /* Flex for tabs */
  display: flex;
  align-items: center;
}

/* Tabs Specific Styles */
.tabs-header {
  padding: 0; /* Remove padding to let tabs fill height */
  overflow-x: auto;
  overflow-y: hidden;
  height: 40px;
  gap: 1px; /* Separator feel */
}

/* Scrollbar styling for tabs if many */
.tabs-header::-webkit-scrollbar {
  height: 4px;
}
.tabs-header::-webkit-scrollbar-thumb {
  background: rgba(255,255,255,0.2);
  border-radius: 2px;
}

.tab-item {
  display: flex;
  align-items: center;
  padding: 0 15px;
  height: 100%;
  cursor: pointer;
  border-right: 1px solid rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.6);
  transition: background 0.2s, color 0.2s;
  min-width: 100px;
  max-width: 200px;
  position: relative;
  user-select: none;
}

.tab-item:hover {
  background: rgba(255, 255, 255, 0.05);
  color: rgba(255, 255, 255, 0.9);
}

.tab-item.active {
  background: rgba(255, 255, 255, 0.1);
  color: var(--accent-color);
  font-weight: 600;
  box-shadow: inset 0 -2px 0 var(--accent-color); /* Bottom active indicator */
}

.tab-name {
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-right: 5px;
}

.rename-input {
  background: rgba(0,0,0,0.3);
  border: none;
  color: white;
  padding: 2px 5px;
  border-radius: 4px;
  width: 100%;
  outline: none;
  font-size: 0.85rem;
}

.close-btn {
  opacity: 0;
  font-size: 1.2rem;
  line-height: 1;
  margin-left: 5px;
  border-radius: 50%;
  width: 18px;
  height: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.tab-item:hover .close-btn {
  opacity: 0.7;
}

.close-btn:hover {
  opacity: 1 !important;
  background: rgba(255, 50, 50, 0.4);
  color: white;
}

.add-btn {
  padding: 0 15px;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 1.2rem;
  color: rgba(255, 255, 255, 0.5);
  transition: color 0.2s;
}

.add-btn:hover {
  color: var(--accent-color);
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
  overflow: hidden;
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
  display: flex;
  align-items: center;
  justify-content: center;
}

img {
  max-width: none;
  display: block;
}
</style>
