<template>
  <div class="app">
    <div class="header">
      <div class="file-info">
        <span class="file-name">{{ currentFile || 'Markdown Util' }}</span>
      </div>
    </div>
    <div class="container">
      <div class="sidebar">
        <h3>大纲</h3>
        <div v-if="outline.length === 0" class="empty-outline">
          暂无大纲
        </div>
        <div v-else>
          <div 
            v-for="(item, index) in outline" 
            :key="index"
            :class="['outline-item', `level-${item.level}`]"
            @click="scrollToSection(item.level, item.text)"
          >
            {{ item.text }}
          </div>
        </div>
      </div>
      <div class="main">
        <div 
          v-if="!content" 
          class="drop-area"
          :class="{ active: isDragOver }"
          @dragover.prevent="handleDragOver"
          @dragleave.prevent="handleDragLeave"
          @drop.prevent="handleDrop"
        >
          <div class="icon">📄</div>
          <p>拖拽Markdown文件到此处</p>
          <p>或点击选择文件</p>
          <input 
            type="file" 
            accept=".md,.markdown" 
            style="display: none;" 
            ref="fileInput"
            @change="handleFileSelect"
          />
        </div>
        <div v-else class="editor-container">
          <textarea 
            class="editor"
            v-model="content"
            @input="updateOutline"
            placeholder="在此编辑Markdown内容..."
          ></textarea>
          <div class="preview" v-html="renderedContent"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'
import MarkdownIt from 'markdown-it'
import hljs from 'highlight.js'
import 'highlight.js/styles/github.css'

const content = ref('')
const currentFile = ref('')
const isDragOver = ref(false)
const outline = ref<Array<{ level: number; text: string }>>([])
const fileInput = ref<HTMLInputElement | null>(null)

// 初始化markdown-it实例
const md = new MarkdownIt({
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return hljs.highlight(str, { language: lang }).value
      } catch (__) {}
    }
    return hljs.highlightAuto(str).value
  }
})

// 渲染Markdown内容
const renderedContent = computed(() => {
  if (!content.value) return ''
  return md.render(content.value)
})

// 处理拖拽事件
const handleDragOver = (e: DragEvent) => {
  isDragOver.value = true
}

const handleDragLeave = (e: DragEvent) => {
  isDragOver.value = false
}

const handleDrop = (e: DragEvent) => {
  isDragOver.value = false
  if (e.dataTransfer?.files.length) {
    const file = e.dataTransfer.files[0]
    if (file.type === 'text/markdown' || file.name.endsWith('.md')) {
      readFile(file)
    }
  }
}

const handleFileSelect = (e: Event) => {
  const target = e.target as HTMLInputElement
  if (target.files?.length) {
    const file = target.files[0]
    readFile(file)
  }
}

// 读取文件内容
const readFile = (file: File) => {
  currentFile.value = file.name
  const reader = new FileReader()
  reader.onload = (e) => {
    if (e.target?.result) {
      content.value = e.target.result as string
      updateOutline()
    }
  }
  reader.readAsText(file, 'utf-8')
}

// 更新大纲
const updateOutline = () => {
  const lines = content.value.split('\n')
  const newOutline: Array<{ level: number; text: string }> = []
  
  lines.forEach(line => {
    const match = line.match(/^(#{1,6})\s+(.*)$/)
    if (match) {
      const level = match[1].length
      const text = match[2]
      newOutline.push({ level, text })
    }
  })
  
  outline.value = newOutline
}

// 滚动到对应章节
const scrollToSection = (level: number, text: string) => {
  const previewElement = document.querySelector('.preview')
  if (previewElement) {
    // 查找对应的标题元素
    const headingElements = previewElement.querySelectorAll(`h${level}`)
    for (const element of headingElements) {
      if (element.textContent === text) {
        element.scrollIntoView({ behavior: 'smooth', block: 'start' })
        break
      }
    }
  }
}

// 点击drop area触发文件选择
const dropArea = ref<HTMLElement | null>(null)

onMounted(() => {
  const area = document.querySelector('.drop-area')
  if (area) {
    area.addEventListener('click', () => {
      fileInput.value?.click()
    })
  }
})
</script>

<style scoped>
.app {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.empty-outline {
  color: var(--text-secondary);
  padding: 20px 0;
  text-align: center;
}
</style>