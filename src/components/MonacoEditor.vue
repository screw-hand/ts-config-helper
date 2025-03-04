<template>
  <div ref="codeEditBox" class="codeEditBox"></div>
</template>
​
<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref, watch, defineExpose, nextTick } from 'vue'
import { debounce } from '@utils'
import useDataStore from '../store/data'
import * as monaco from 'monaco-editor'
// import json5 from 'json5'
export interface EditorOptions {
  theme: 'darkTheme' | 'lightTheme'
}
export interface Props {
  modelValue?: string
  language: string
  options?: EditorOptions
  width?: string
  height?: string
}
const emit = defineEmits(['update:modelValue', 'change', 'editor-mounted', 'cursorLineChange'])
const props = defineProps<Props>()
const dataStore = useDataStore()
let editor: monaco.editor.IStandaloneCodeEditor | null = null
const codeEditBox = ref()
let shouldEmitChange = true
const init = () => {
  // 定义主题
  monaco.editor.defineTheme('darkTheme', {
    base: 'vs-dark',
    inherit: true,
    rules: [],
    colors: {
      'editor.background': '#101014'
    }
  })
  monaco.editor.defineTheme('lightTheme', {
    base: 'vs',
    inherit: true,
    rules: [],
    colors: {
      'editor.background': '#ffffff'
    }
  })
  monaco.languages.json.jsonDefaults.setDiagnosticsOptions({
    validate: true,
    enableSchemaRequest: true,
    allowComments: false,
    schemas: [
      {
        fileMatch: ['*'],
        uri: 'https://json.schemastore.org/tsconfig'
      }
    ]
  })
  editor = monaco.editor.create(codeEditBox.value, {
    value: props.modelValue,
    automaticLayout: false,
    domReadOnly: true,
    scrollbar: {
      verticalScrollbarSize: 8,
      horizontalScrollbarSize: 8
    },
    suggest: {
      preview: true
    },
    insertSpaces: true,
    ...props.options
  })
  editor.onDidChangeCursorPosition(
    debounce((e) => {
      if (!editor) return
      const selection = editor.getSelection()
      if (selection && Math.abs(selection.startLineNumber - selection.endLineNumber) !== 0) return
      const lineNumber: number = e.position.lineNumber
      const model = editor.getModel()
      if (!model) return
      const allLineContent = model.getLinesContent()
      const cursorLineContent = model.getLineContent(lineNumber)
      const cursorBeforeLineContent = allLineContent.slice(0, lineNumber - 1).join('')
      emit('cursorLineChange', { cursorBeforeLineContent, cursorLineContent })
    }, 200)
  )
  // // 绑定“Ctrl+Z”键为撤销操作
  // editor.addCommand(monaco.KeyMod.CtrlCmd | monaco.KeyCode.KeyZ, function () {
  //   // console.log('撤销')
  //   editor?.trigger('keyboard', 'undo', null)
  // })

  // // 绑定“Ctrl+Y”键为重做操作
  // editor.addCommand(monaco.KeyMod.CtrlCmd | monaco.KeyCode.KeyY, function () {
  //   // console.log('重做')
  //   editor?.trigger('keyboard', 'redo', null)
  // })
  // 监听值的变化
  editor.onDidChangeModelContent(() => {
    if (!shouldEmitChange) return
    const value = editor!.getValue() //给父组件实时返回最新文本
    dataStore.config = value
    emit('update:modelValue', value)
    emit('change', value)
  })
  emit('editor-mounted', editor)
  // sync value
  dataStore.config = props.modelValue || ''
}

watch(
  () => props.modelValue,
  (newValue) => {
    if (editor) {
      let value = editor.getValue()
      if (newValue !== value) {
        shouldEmitChange = false
        const model = editor.getModel()
        const position = editor.getPosition()
        if (model) {
          // editor.pushUndoStop()
          editor.setValue((newValue as string) || '')
          position && editor.setPosition(position)
        }
        nextTick(() => (shouldEmitChange = true))
      }
    }
  }
)
watch(
  () => props.options,
  (newOptions) => {
    if (editor) {
      editor.updateOptions(newOptions || {})
    }
  },
  { deep: true }
)
// watch(
//   () => props.theme,
//   (themeName) => monaco.editor.setTheme(themeName)
// )
function resize() {
  if (!editor) return
  editor.layout()
}
onMounted(init)
onBeforeUnmount(() => editor && editor.dispose())
defineExpose({
  resize
})
</script>

<style scoped>
.codeEditBox {
  width: v-bind(width);
  height: v-bind(height);
}
:deep(.monaco-editor .margin),
:deep(.monaco-editor-background) {
  transition: background-color 0.3s var(--n-bezier), background-color 0.3s var(--n-bezier),
    box-shadow 0.3s var(--n-bezier), border-color 0.3s var(--n-bezier);
}
</style>
