<template>
  <v-app>
    <v-main class="flex main">
      <aside class="ai-drawer pa-4">
        <div class="justify-space-between align-center flex-row flex mb-3 w-100">
          <div class="text-h5 logo no-select font-weight-bold">AI Writer</div>
          <div>
            <v-dialog v-model="dialog" :width="540">
              <template v-slot:activator="{ props }">
                <v-icon icon="mdi-cog-outline" class="pointer hover-primary" @click="dialog = true" v-bind="props">
                </v-icon>
              </template>
              <v-card>
                <v-card-title>
                  <span class="text-h5">Settings</span>
                </v-card-title>
                <v-card-text>
                  <v-form ref="openAISettingForm" lazy-validation class="mt-4">
                    <v-text-field label="AI writer key" v-model="settingForm.apiKey">
                    </v-text-field>

                    <v-text-field label="Translator key" v-model="settingForm.authKey">
                    </v-text-field>
                  </v-form>
                </v-card-text>
                <v-card-actions>
                  <v-spacer></v-spacer>
                  <v-btn color="blue-darken-1" variant="text" @click="onSaveAPIKey">
                    Save
                  </v-btn>
                </v-card-actions>
              </v-card>
            </v-dialog>
          </div>
        </div>
        <p class="text-subtitle-2 font-weight-regular no-select">
          Let AI generate text suggestions for you.
        </p>
        <v-form ref="form" lazy-validation class="mt-4">
          <v-select variant="outlined" v-model="formData.type" :items="TASK_TYPES"
            :rules="[(v) => !!v || 'Item is required']" color="primary">
          </v-select>

          <div v-if="selectedType.theme">
            <h3 class="mb-3 no-select">
              {{ selectedType.theme.title }}
            </h3>

            <v-textarea variant="outlined" ref="themeRef" auto-grow v-model="formData.theme"
              :rows="selectedType.theme.rows || 1" :max-rows="4" :counter="selectedType.theme.maxlength"
              :counter-value="() => formData.theme.length" :placeholder="selectedType.theme.placeholder"
              color="primary">
              <template v-slot:append-inner>
                <v-tooltip text="Copy from the right">
                  <template v-slot:activator="{ props }">
                    <v-icon v-bind="props" icon="mdi-select" @click.stop="onSelectForInput(THEME)"
                      class="hover-primary"></v-icon>
                  </template>
                </v-tooltip>
              </template>
            </v-textarea>
          </div>

          <div v-if="selectedType.subheading">
            <h3 class="mb-3 no-select">
              {{ selectedType.subheading.title }}
            </h3>

            <v-textarea variant="outlined" ref="subheadingRef" auto-grow v-model="formData.subheading" :rows="1"
              :max-rows="2" :counter="selectedType.subheading.maxlength"
              :counter-value="() => formData.subheading.length" :placeholder="selectedType.subheading.placeholder"
              color="primary">
              <template v-slot:append-inner>
                <v-tooltip text="Copy from the right">
                  <template v-slot:activator="{ props }">
                    <v-icon v-bind="props" icon="mdi-select" @click.stop="onSelectForInput(SUBHEADING)"
                      class="hover-primary"></v-icon>
                  </template>
                </v-tooltip>
              </template>
            </v-textarea>
          </div>

          <div v-if="formData.type !== DEEPL" class="mb-3">
            <h3 class="mb-3 no-select">Tonality</h3>

            <v-btn class="mb-1 mr-1" :variant="formData.mood === mood ? 'tonal' : 'text'"
              :color="formData.mood === mood ? 'primary' : ''" v-for="mood in MOODS" :key="mood" size="small"
              @click="onToggleMood(mood)">
              {{ mood }}
            </v-btn>
          </div>

          <div class="btns">
            <v-btn :disabled="invalid" :loading="loading" @click="onCreate" color="primary">
              Magic
            </v-btn>
            <v-btn @click="onReset" class="ml-2" v-if="isNeedReset">
              Reset
            </v-btn>
          </div>
        </v-form>

        <div class="suggestions mt-5">
          <div class="text-subtitle-2 font-weight-regular" v-if="suggestions.length">
            Add suggestions to the editor
            <v-btn @click="suggestions.length = 0" class="ml-2" size="x-small" variant="tonal">
              Clear
            </v-btn>
          </div>
          <v-card v-for="(suggestion, index) in suggestions" class="pa-4 pb-0 my-4 pointer" :key="sug" elevation="2"
            hover @click="onCopySuggestion(index)">
            <p>{{ suggestion }}</p>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn color="primary" @click.stop="onCopySuggestionOnly(index)">Copy</v-btn>
            </v-card-actions>
          </v-card>
        </div>
      </aside>

      <div class="flex-1 content flex flex-column">
        <QuillEditor theme="snow" contentType="text" :options="quillOptions" ref="quill" v-model:content="article"
          @selectionChange="onArticleSelect" class="flex-1 pa-4 pb-0 border-0 overflow-y-auto" id="quill-editor"
          draggable="false" />
        <!-- <textarea id="article" name="" cols="30" rows="10" v-model="article" placeholder="Your article" class="pa-5"
          @change="onContentChange" @select="onArticleSelect" @click="onContentClick"
          @keydown="onContentKeyboardEvent"></textarea> -->

        <div class="article-tools d-flex justify-space-between align-center pa-4 bg-white">
          <v-btn class="mr-10" variant="tonal" color="primary" @click="onCopyAll">
            Copy!
          </v-btn>

          <v-spacer></v-spacer>

          <v-tooltip text="back" location="top">
            <template v-slot:activator="{ props }">
              <v-btn icon="mdi-arrow-u-left-top" size="small" class="mr-3" variant="tonal" @click="onBack"
                v-bind="props">
              </v-btn>
            </template>
          </v-tooltip>

          <v-tooltip text="forward" location="top">
            <template v-slot:activator="{ props }">
              <v-btn icon="mdi-arrow-u-right-top" size="small" class="mr-3" variant="tonal" @click="onForward"
                v-bind="props">
              </v-btn>
            </template>
          </v-tooltip>

          <v-chip class="no-select" close-icon="mdi-close-outline" color="primary" size="small">{{ wordsCount }} words
          </v-chip>
        </div>
      </div>

      <v-snackbar v-model="toast" location="top" :timeout="2000">
        {{ toastMessage }}
      </v-snackbar>
    </v-main>
  </v-app>
</template>

<script setup>
import { QuillEditor } from "@vueup/vue-quill"
import DeepL from 'deepl'
import { Configuration, OpenAIApi } from "openai"
import { computed, onMounted, reactive, ref, watch } from "vue"

const quillOptions = {
  modules: {
    toolbar: [
      [{ 'header': [1, 2, 3, false] }],
      ['bold', 'italic', 'underline', 'strike'],
      [{ 'color': [] }],
      [{ 'list': 'ordered' }, { 'list': 'bullet' }],
    ]
  },
}

const THEME = "theme"
const SUBHEADING = "subheading"
const DEEPL = 'Translate to English'
const MOODS = ['positive', 'excited', 'gentle', 'formal', 'casual', 'witty']
const TASKS = [
  {
    type: DEEPL,
    theme: {
      title: "Text to translate",
      placeholder: 'Add the text you want to translate.',
      help: "How to give a effective brief.",
      maxlength: 1000,
    },
  },
  {
    type: "Essay Outline",
    theme: {
      title: "Essay Title",
      placeholder: "e.g. The dramatic fall of cocoa prices.",
      help: "How to give a effective brief.",
      maxlength: 600,
    },
    genPrompt: (mood, text) => {
      let prompt = `Generate an outline for an essay`
      if (text) prompt += ` about ${text}:`
      return prompt
    }
  },
  {
    type: 'Outline to Paragraph',
    theme: {
      title: "The essay is about",
      placeholder: "e.g. The essay is about the benefits of baking your own bread at home. It shows that baking can reduce stress.",
      help: "How to give a effective brief.",
      maxlength: 200,
      rows: 3
    },
    subheading: {
      title: "Subheading",
      placeholder: "e.g. The science behind baking",
      help: "How to give a effective brief.",
      maxlength: 600,
    },
    genPrompt: (mood, text) => {
      let prompt = `Generate an essay talking`
      if (text) prompt += ` about ${text}`
      if (mood) prompt += ` in a ${mood} tone.\n`
      return prompt
    }
  },
  {
    type: 'Keywords to Paragraph',
    theme: {
      title: "Text to expand",
      placeholder: "Use keywords or short sentencenes.",
      help: "How to give a effective brief.",
      maxlength: 200,
    },
    genPrompt: (mood, text) => {
      let prompt = `Generate an essay talking`
      if (text) prompt += ` about ${text}`
      if (mood) prompt += ` in a ${mood} tone.\n`
      return prompt
    }
  },
  {
    type: "Tone Rewrite",
    theme: {
      title: "Text to rewrite",
      placeholder: 'Add the text you want to rewrite.',
      help: "How to give a effective brief.",
      maxlength: 1000
    },
    config: {
      temperature: 0.5,
      frequency_penalty: 0.2,
    },
    genPrompt: (mood, text) => {
      let prompt = `rewrite paragraph`
      if (text) prompt += ` below\n Original: ${text}.\n Rewrite:`
      if (mood) prompt = `in a ${mood} style ` + prompt
      return prompt
    }
  },
]
const TASK_TYPES = TASKS.map((t) => t.type)
let quill = ref(null) //vue的quillRef，仅有部分api
let quillInstance = ref(null) //原生的quill对象，包含所有api
const dialog = ref(false)
const loading = ref(false)
const form = ref(null)
const themeRef = ref(null)
const subheadingRef = ref(null)
const suggestions = ref([])
const selectingForCopy = ref(false)
const selectingForCopyType = ref(THEME)
const article = ref('')
const openAISettingForm = ref(null)
const deepLSettingForm = ref(null)
const mousePosition = ref(0)
const openai = ref(null)
const toast = ref(false)
const toastMessage = ref('')
const settingForm = reactive({
  apiKey: '',
  authKey: ''
})
const formData = reactive({
  type: TASKS[0].type,
  theme: "",
  subheading: "",
  mood: ''
})
const wordsCount = computed(() => {
  let t = article.value.replaceAll(/\n+|\s+/g, ' ')
  return t.split(' ').filter(o => !!o).length
})
const selectedType = computed(() =>
  TASKS.find((t) => t.type === formData.type)
)
const invalid = computed(() => {
  if (selectedType.value.theme && !formData.theme.trim()) return true
  if (selectedType.value.subheading && !formData.subheading.trim()) return true
  return false
})
const isNeedReset = computed(() => formData.theme || formData.subheading)

watch(() => formData.type, onReset)

watch(article, () => {
  selectingForCopy.value = false
})

onMounted(() => {
  let authKey = localStorage.getItem("authKey")
  if (authKey) {
    settingForm.authKey = authKey
  }
  let apiKey = localStorage.getItem("apiKey")
  if (apiKey) {
    settingForm.apiKey = apiKey
  }
  generateOpenai()
  quillInstance = quill.value.getQuill()
})

function getContent() {
  return quill.value.getText()
}

function generateOpenai() {
  const configuration = new Configuration({ apiKey: settingForm.apiKey })
  openai.value = new OpenAIApi(configuration)
}

function onReset() {
  formData.theme = ""
  formData.subheading = ""
  formData.mood = ""
}

/**
 * 切换ai语气
 * @param {string} mood
 */
function onToggleMood(mood) {
  if (formData.mood === mood) formData.mood = ''
  else formData.mood = mood
}

/**
 * 除了deepl的，都走这个请求
 */
function requestAI() {
  let p = openai.value.createCompletion({
    model: "text-davinci-002",
    prompt: selectedType.value?.genPrompt(formData.mood, formData.theme),
    temperature: 0.1,
    max_tokens: 600,
    top_p: 1.0,
    frequency_penalty: 0.0,
    presence_penalty: 0.0,
    n: 2,
    ...selectedType.value.config
  })
  p.catch(err => {
    toastMessage.value = err.message
    toast.value = true
  })
  return p
}

function requestTranslate() {
  let p = DeepL({
    free_api: true,
    text: formData.theme,
    target_lang: 'EN',
    source_lang: 'ZH',
    auth_key: settingForm.apiKey,
  })
  p.catch(err => {
    toastMessage.value = err.message
    toast.value = true
  })
  return p
}

/**
 * 发起请求
 */
function onCreate() {
  if (formData.type === DEEPL) {
    if (!settingForm.authKey) {
      dialog.value = true
      return
    }
    loading.value = true
    requestTranslate().then(res => {
      suggestions.value = res.data.translations.map(c => c.text.trim())
    }).finally(() => {
      loading.value = false
    })
  } else {
    if (!settingForm.apiKey) {
      dialog.value = true
      return
    }
    loading.value = true
    requestAI().then(res => {
      suggestions.value = res.data.choices.map(c => c.text.trim())
    }).finally(() => {
      loading.value = false
    })
  }
}

/**
 * 修改实际的文章数据，并同步修改ui表现
 * @param {*} text
 */
function setText(text) {
  article.value = text
  quill.value.setText(text)
}

/**
 * copy suggestion并自动填充到文章中
 * @param {*} index 被copy项的suggestions的index
 */
function onCopySuggestion(index) {
  let value = suggestions.value[index]
  if (mousePosition.value !== undefined) {
    let t = article.value.slice(0, mousePosition.value) + value + article.value.slice(mousePosition.value)
    setText(t)
  } else {
    setText(article.value + " " + value)
  }
  let temp = mousePosition.value
  quillInstance.setSelection(temp + value.length)
  mousePosition.value = temp + value.length
  suggestions.value.splice(index, 1)
}

/**
 * 仅仅copy suggestion，不自动填充到文章中
 * @param {*} index 被copy项的suggestions的index
 */
function onCopySuggestionOnly(index) {
  let value = suggestions.value[index]
  navigator.clipboard.writeText(value)
}

/**
 * 复制文章所有内容
 */
function onCopyAll() {
  let value = quill.value.getText()
  navigator.clipboard.writeText(value)
}

/**
 * 当文章的内容被用户选中时要做的事
 * @param {*} param quill的select事件的参数
 */
function onArticleSelect(param) {
  const { range } = param
  if (range.length === 0) {
    mousePosition.value = range.index
    return
  }
  const content = getContent()
  if (!selectingForCopy.value) return
  formData[selectingForCopyType.value] = content.slice(range.index, range.index + range.length)
  if (selectingForCopyType.value === THEME) themeRef.value?.focus()
  if (selectingForCopyType.value === SUBHEADING) subheadingRef.value?.focus()
  selectingForCopy.value = false
}

/**
 * 用户填写apiKey后保存到本地
 */
function onSaveAPIKey() {
  openAISettingForm.value.validate().then((res) => {
    if (res.valid) {
      dialog.value = false
      localStorage.setItem("authKey", settingForm.authKey || '')
      localStorage.setItem("apiKey", settingForm.apiKey || '')
      generateOpenai()
    }
  })
}

function onSelectForInput(type) {
  selectingForCopy.value = true
  selectingForCopyType.value = type
}

/**
 * 执行后退操作
 */
function onBack() {
  quillInstance.history.undo()
}

/**
 * 执行前进操作
 */
function onForward() {
  quillInstance.history.redo()
}
</script>

<style lang="scss">
html {
  overflow-y: hidden !important;
}

::selection {
  color: white;
  background-color: #7bb5a3;
}

.logo {
  color: #7bb5a3;
}

.main {
  height: calc(100vh - 64px);
  overflow: hidden;
}

.ai-drawer {
  width: 400px;
  background-color: #fcfcfc;
  overflow-y: auto;
}

.content {
  box-shadow: 0 0 2px 0 #ccc;
}

#article {
  overflow-y: auto;
  height: calc(100% - 72px);
  width: 100%;
  outline: none;
  resize: none;
  padding-bottom: 72px !important;
}

.article-tools {
  // position: fixed;
  width: calc(100vw - 400px);
  right: 0;
  bottom: 0;
}

.flex {
  display: flex;
}

.justify-center {
  justify-content: center;
}

.align-center {
  align-items: center;
}

.flex-1 {
  flex: 1;
}

.pointer {
  cursor: pointer;
}

.copy {
  cursor: copy;
}

.hover-primary:hover {
  color: #7bb5a3;
}

.no-select {
  user-select: none;
}
</style>

<style>
::-webkit-scrollbar {
  width: 7px;
  height: 7px;
}

::-webkit-scrollbar-track {
  width: 6px;
  background: rgba(16, 31, 28, 0.05);
  -webkit-border-radius: 2em;
  -moz-border-radius: 2em;
  border-radius: 2em;
}

::-webkit-scrollbar-thumb {
  background-color: rgba(144, 147, 153, 0.3);
  background-clip: padding-box;
  min-height: 28px;
  -webkit-border-radius: 2em;
  -moz-border-radius: 2em;
  border-radius: 2em;
  transition: background-color 0.3s;
  cursor: pointer;
}

::-webkit-scrollbar-thumb:hover {
  background-color: rgba(144, 147, 153, 0.4);
}

.ql-container {
  font-size: 16px !important;
}

.ql-editor {
  line-height: 1.5 !important;
}

.v-textarea--auto-grow .v-field__input {
  overflow: auto !important;
  mask-image: none !important;
  -webkit-mask-image: none !important;
}

.v-textarea--auto-grow .v-field__field {
  position: relative;
}

.v-textarea--auto-grow .v-field__field:after {
  content: '';
  width: 7px;
  height: 100%;
  background-color: rgb(252, 252, 252);
  position: absolute;
  top: 0;
  right: 0;
}
</style>
