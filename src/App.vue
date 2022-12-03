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
          Write like a pro.
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
              <template v-slot:append-icon>
                <v-tooltip text="Copy from the right">
                  <template v-slot:activator="{ props }">
                    <v-icon v-bind="props" icon="mdi-select" @click.stop="onSelectForInput(THEME)"
                      class="hover-primary"></v-icon>
                  </template>
                </v-tooltip>
              </template>
            </v-textarea>
          </div>

          <div v-if="selectedType.outline">
            <h3 class="mb-3 no-select">
              {{ selectedType.outline.title }}
            </h3>

            <v-textarea variant="outlined" ref="outlineRef" auto-grow v-model="formData.outline" :rows="1" :max-rows="2"
              :counter="selectedType.outline.maxlength" :counter-value="() => formData.outline.length"
              :placeholder="selectedType.outline.placeholder" color="primary">
              <template v-slot:append-inner>
                <v-tooltip text="Copy from the right">
                  <template v-slot:activator="{ props }">
                    <v-icon v-bind="props" icon="mdi-select" @click.stop="onSelectForInput(OUTLINE)"
                      class="hover-primary"></v-icon>
                  </template>
                </v-tooltip>
              </template>
            </v-textarea>
          </div>

          <div v-if="formData.type !== DEEPL" class="mb-3 pr-15">
            <h3 class="mb-3 no-select">Tone</h3>
            <div v-for="mood in MOODS" :key="mood" class="mood-wrap">
              <v-btn class="mb-1 mr-1" :variant="formData.mood === mood ? 'tonal' : 'text'"
                :color="formData.mood === mood ? 'primary' : ''" size="small" @click="onToggleMood(mood)">
                {{ mood }}
              </v-btn>
            </div>
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
          <div v-if="(costTime && loading)" class="text-subtitle-2 text-center">
            It will take about {{costTime}} seconds.
            <br>
            <p v-if="timeCount">{{timeCount}} seconds passed...</p>
          </div>
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
          @selectionChange="onArticleSelect" @update:content="onContentChange"
          class="flex-1 pa-4 pb-0 border-0 overflow-y-auto" id="quill-editor" draggable="false"
          @textChange="onTextChange" />
        <!-- <textarea id="article" name="" cols="30" rows="10" v-model="article" placeholder="Your article" class="pa-5"
          @change="onContentChange" @select="onArticleSelect" @click="onContentClick"
          @keydown="onContentKeyboardEvent"></textarea> -->

        <div class="article-tools d-flex justify-space-between align-center pa-4 bg-white">

          <v-btn class="mr-3" variant="tonal" color="primary" @click="onCopyAll">
            Copy!
          </v-btn>

          <v-spacer></v-spacer>
          <v-tooltip text="clear all" location="top">
            <template v-slot:activator="{ props }">
              <v-btn icon="mdi-delete" size="small" class="mr-3" variant="tonal" @click="onClearArticle"
                v-bind="props">
              </v-btn>
            </template>
          </v-tooltip>

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
const OUTLINE = "outline"
const DEEPL = 'Translate to English'
const ESSAY_OUTLINE = 'Essay Outline'
const OUTLINE_TO_PARAGRAPH = 'Outline to Paragraph'
const KEYWORDS_TO_PARAGRAPH = 'Keywords to Paragraph'
const TONE_REWRITE = 'Tone Rewrite'
const POSITIVE = 'positive'
const EXCITED = 'excited'
const GENTLE = 'gentle'
const FORMAL = 'formal'
const CASUAL = 'casual'
const WITTY = 'witty'
const MOODS = [POSITIVE, EXCITED, GENTLE, FORMAL, CASUAL, WITTY]
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
    type: ESSAY_OUTLINE,
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
    type: OUTLINE_TO_PARAGRAPH,
    theme: {
      title: "Essay Topic",
      placeholder: "e.g. The essay is about the benefits of baking your own bread at home. It shows that baking can reduce stress.",
      help: "How to give a effective brief.",
      maxlength: 200,
      rows: 3
    },
    outline: {
      title: "Outline",
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
    type: KEYWORDS_TO_PARAGRAPH,
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
    type: TONE_REWRITE,
    theme: {
      title: "Text to rewrite",
      placeholder: 'Add the text you want to rewrite.',
      help: "How to give a effective brief.",
      maxlength: 1000
    },
    genPrompt: (mood, text) => {
      let prompt = ''
      if (!mood) prompt = 'completely summarize and rewrite differently every single sentence below completely based on your understanding'
      else prompt = rewriteConfigs[mood].prompt
      if (text) prompt += `\nQuote: ${text}\nRewrite:`
      return prompt
    },
  },
]

/** rewrite时，每种mood对应不同的prompt和openaiConfig */
const rewriteConfigs = {
  [POSITIVE]: {
    config: {
      temperature: 1.1,
      max_tokens: 600,
      top_p: 0.9,
      frequency_penalty: 0.5,
      presence_penalty: 0.5
    },
    prompt: 'completely summarize and rewrite reconstruct the qoute and every sentence below completely based on your understanding, using very very positive words, encouraging words, optimistic words',
  },
  [EXCITED]: {
    config: {
      temperature: 1.1,
      max_tokens: 600,
      top_p: 0.9,
      frequency_penalty: 0.5,
      presence_penalty: 0.5
    },
    prompt: 'completely rewrite reconstruct the qoute and every sentence below completely based on your understanding, using very exciting words, surprising words, amazing words, new words, like you are very excited',
  },
  [GENTLE]: {
    config: {
      temperature: 1.2,
      max_tokens: 600,
      top_p: 1,
      frequency_penalty: 0.9,
      presence_penalty: 0.9
    },
    prompt: 'completely summarize and rewrite refine reconstruct sentences below completely based on your understanding, using very gentle words, nice words, beautiful words, and kind words for 2nd grade students',
  },
  [FORMAL]: {
    config: {
      temperature: 1.2,
      max_tokens: 600,
      top_p: 0.9,
      frequency_penalty: 0.9,
      presence_penalty: 0.9
    },
    prompt: 'completely summarize and rewrite the quote below and merge and reconstruct sentences below completely based on your understanding, using very formal words, professional words, adavanced words, and academic words, pretending you are a professor',
  },
  [CASUAL]: {
    config: {
      temperature: 1.2,
      max_tokens: 600,
      top_p: 1,
      frequency_penalty: 0.9,
      presence_penalty: 0.9
    },
    prompt: 'completely rewrite differently refine every single sentence differently below completely based on your understanding, using very casual words, street words, and easy words for 2nd grade students',
  },
  [WITTY]: {
    config: {
      temperature: 1.2,
      max_tokens: 600,
      top_p: 1,
      frequency_penalty: 0.9,
      presence_penalty: 0.9
    },
    prompt: 'completely summarize and rewrite differently every single sentence below completely based on your understanding, using very witty words, funny words, and joke words, like you are telling a joke',
  },
}


const TASK_TYPES = TASKS.map((t) => t.type)
let quill = ref(null) //vue的quillRef，仅有部分api
let quillInstance = ref(null) //原生的quill对象，包含所有api
const dialog = ref(false)
const loading = ref(false)
const costTime = ref(0) //请求预计花费的时间
const counter = ref(null)  //请求中计时花费了多少秒，每秒更新
const timeCount = ref(0)
const form = ref(null)
const themeRef = ref(null)
const outlineRef = ref(null)
const suggestions = ref([])
const selectingForCopy = ref(false)
const selectingForCopyType = ref(THEME)
const article = ref('')
const openAISettingForm = ref(null)
const mousePosition = ref({ index: 0, length: 0 })
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
  outline: "",
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
  if (selectedType.value.outline && !formData.outline.trim()) return true
  return false
})
const isNeedReset = computed(() => formData.theme || formData.outline)

watch(() => formData.type, onReset)

watch(article, () => {
  selectingForCopy.value = false
})

watch(loading, (n, o) => {
  if(n && !o) suggestions.value.length = 0
})

onMounted(() => {
  let hash = location.hash
  let k1, k2
  if (hash) {
    hash = hash.slice(1)
    let split = hash.split('&').map(c => c.split('='))

    k1 = split.find(c => c[0] === 'k1')?.[1]
    if (k1) {
      settingForm.apiKey = k1
      localStorage.setItem("apiKey", k1)
    }

    k2 = split.find(c => c[0] === 'k2')?.[1]
    if (k2) {
      settingForm.authKey = k2
      localStorage.setItem("authKey", k2)
    }
  }
  if (!k1) {
    let apiKey = localStorage.getItem("apiKey")
    if (apiKey) settingForm.apiKey = apiKey
  }
  if (!k2) {
    let authKey = localStorage.getItem("authKey")
    if (authKey) settingForm.authKey = authKey
  }
  generateOpenai()
  quillInstance = quill.value.getQuill()
})

/** 估算请求需要花费的时间，单位：秒 */
function computeWaitTime(type, input) {
  let len = input.replaceAll(/\n+|\s+/g, ' ').split(' ').filter(o => !!o).length
  if (type === TONE_REWRITE) return Math.max(parseInt(len / 15), 2)
  else if (type === DEEPL) return 0
  else return 12
}

function getContent() {
  return quill.value.getText()
}

function generateOpenai() {
  const configuration = new Configuration({ apiKey: settingForm.apiKey })
  openai.value = new OpenAIApi(configuration)
}

function onReset() {
  formData.theme = ""
  formData.outline = ""
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
  let extraConfig
  if (selectedType.value.type === TONE_REWRITE) {
    extraConfig = {
      temperature: 1.2,
      max_tokens: 600,
      top_p: 1,
      frequency_penalty: 0.9,
      presence_penalty: 0.9
    }
    if (formData.mood) extraConfig = {
      ...extraConfig,
      ...rewriteConfigs[formData.mood].config
    }
  }
  let p = openai.value.createCompletion({
    model: "text-davinci-003",
    prompt: selectedType.value?.genPrompt(formData.mood, formData.theme),
    temperature: 0.1,
    max_tokens: 600,
    top_p: 1.0,
    frequency_penalty: 0.0,
    presence_penalty: 0.0,
    n: 2,
    ...extraConfig
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
    auth_key: settingForm.authKey,
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
  costTime.value = 0
  timeCount.value = 0
  let t = computeWaitTime(formData.type, formData.theme)
  if (formData.type === DEEPL) {
    if (!settingForm.authKey) {
      dialog.value = true
      return
    }
    loading.value = true
    costTime.value = t
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
    costTime.value = t
    counter.value = setInterval(()=> {timeCount.value++}, 1000)
    requestAI().then(res => {
      suggestions.value = res.data.choices.map(c => c.text.trim())
    }).finally(() => {
      loading.value = false
      clearInterval(counter.value)
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

function onClearArticle() {
  setText('')
}

/**
 * 文章内容改变时记录光标位置
 */
function onContentChange() {
  setTimeout(() => {
    mousePosition.value = quillInstance.getSelection()
  })
}

function onTextChange({ delta: { ops } }) {
  if (ops.find(o => o.insert === '\n')) { //当按下回车换行时，quill有个bug是光标的index并不会+1，但是后面的回车又是正常的，要处理一下
    mousePosition.value.index++
  }
}

/**
 * copy suggestion并自动填充到文章中
 * @param {*} index 被copy项的suggestions的index
 */
function onCopySuggestion(i) {
  let value = suggestions.value[i]
  const { index, length } = mousePosition.value

  if (length) {  //如果之前已经选中了文章中的某一段，那就替换
    let t = article.value.slice(0, index) + value + article.value.slice(index + length)
    setText(t)
  } else {  //否则，应该把内容插入到记录下的光标位置去
    let t = article.value.slice(0, index) + value + article.value.slice(index)
    setText(t)
  }
  quillInstance.setSelection(index, value.length)
  mousePosition.value.index = index + value.length
  suggestions.value.splice(i, 1)
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
  if (!range) return
  mousePosition.value = range
  if (range.length === 0) return

  const content = getContent()
  if (!selectingForCopy.value) return
  formData[selectingForCopyType.value] = content.slice(range.index, range.index + range.length)
  if (selectingForCopyType.value === THEME) themeRef.value?.focus()
  if (selectingForCopyType.value === OUTLINE) outlineRef.value?.focus()
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
      let hash = '#'
      if (settingForm.apiKey) hash += 'k1=' + settingForm.apiKey
      if (settingForm.authKey) hash += '&k2=' + settingForm.authKey
      if (hash !== '#') location.hash = hash
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
  flex-shrink: 0;
}

.mood-wrap {
  display: inline-block;
  width: 85px;
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
  word-break: break-word;
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
