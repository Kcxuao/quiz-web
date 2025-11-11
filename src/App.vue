<template>
  <div class="flex flex-col items-center min-h-screen bg-gray-100 p-4 md:p-8">
    <!-- 移动端遮罩层 -->
    <div v-if="!isCollapsed && isMobile" class="fixed inset-0 bg-black/30 z-40" @click="isCollapsed = true"></div>

    <div class="flex w-full max-w-6xl gap-4 relative">
      <!-- 左侧题号栏 -->
      <transition name="slide">
        <aside v-show="!isCollapsed" class="bg-white shadow-md p-4 flex flex-col"
          :class="isMobile ? 'fixed inset-0 left-0 top-0 z-50 rounded-none w-4/5 h-screen' : 'w-64 rounded-2xl'">
          <div class="flex justify-between items-center mb-3">
            <h3 class="font-semibold text-gray-700">题号</h3>
            <button class="text-gray-500 hover:text-gray-700 p-1 rounded" @click="isCollapsed = true">
              {{ isMobile ? '✕' : '←' }}
            </button>
          </div>
          <div class="grid gap-2 overflow-y-auto pr-1 custom-scroll" :class="isMobile ? 'grid-cols-4' : 'grid-cols-5'"
            :style="{ maxHeight: isMobile ? 'calc(100vh - 120px)' : '60vh' }">
            <div v-for="(q, i) in displayQuestions" :key="i"
              class="w-9 h-9 flex items-center justify-center text-sm font-medium rounded-full cursor-pointer transition-all relative"
              :class="{
                'bg-blue-500 text-white': getOriginalIndex(i) === currentIndex,
                'bg-green-500 text-white': answerStatus[getOriginalIndex(i)] === 'correct',
                'bg-red-500 text-white': answerStatus[getOriginalIndex(i)] === 'wrong',
                'bg-gray-100 hover:bg-gray-200 text-gray-600': !answerStatus[getOriginalIndex(i)] && getOriginalIndex(i) !== currentIndex
              }" @click="jumpToQuestion(getOriginalIndex(i))">
              {{ getOriginalIndex(i) + 1 }}
              <span v-if="isWrongOnly && favorites[getOriginalIndex(i)]" class="absolute top-0 right-0 text-xs">⭐</span>
            </div>
          </div>
        </aside>
      </transition>

      <!-- 折叠按钮 -->
      <button v-show="isCollapsed && isMobile"
        class="fixed right-4 bottom-4 z-30 bg-blue-500 text-white rounded-full w-12 h-12 flex items-center justify-center shadow-lg text-xl font-bold hover:bg-blue-600"
        @click="isCollapsed = false">
        ☰
      </button>

      <!-- 主答题区域 -->
      <main class="flex-1 w-full md:w-auto">
        <div class="bg-white rounded-2xl shadow-lg p-4 md:p-8 h-full flex flex-col">
          <!-- 科目选择和筛选 -->
          <div class="space-y-3 mb-6">
            <div class="flex gap-2 justify-center">
              <select v-model="selectedExam" class="border border-gray-300 rounded-lg p-2 flex-1"
                @change="onExamChange">
                <option v-for="exam in exams" :key="exam" :value="exam">{{ exam }}</option>
              </select>

              <select v-model="selectedSubject" class="border border-gray-300 rounded-lg p-2 flex-1"
                @change="handleSubjectChange">
                <option v-for="sub in subSubjects[selectedExam]" :key="sub" :value="sub">{{ sub }}</option>
              </select>
            </div>

            <!-- 筛选按钮 -->
            <div class="flex gap-2 justify-center flex-wrap">
              <button class="px-3 py-1 rounded-lg text-sm transition-all"
                :class="!isWrongOnly ? 'bg-blue-500 text-white' : 'bg-gray-200 text-gray-700 hover:bg-gray-300'"
                @click="toggleWrongOnly(false)">
                全部题目
              </button>
              <button class="px-3 py-1 rounded-lg text-sm transition-all"
                :class="isWrongOnly ? 'bg-red-500 text-white' : 'bg-gray-200 text-gray-700 hover:bg-gray-300'"
                @click="toggleWrongOnly(true)">
                ❌ 错题集 ({{ wrongCount }})
              </button>
              <button class="px-3 py-1 rounded-lg text-sm transition-all"
                :class="showFavorites ? 'bg-yellow-500 text-white' : 'bg-gray-200 text-gray-700 hover:bg-gray-300'"
                @click="toggleShowFavorites">
                ⭐ 收藏 ({{ favoritesCount }})
              </button>
            </div>
          </div>

          <!-- 统计面板 -->
          <div class="grid grid-cols-3 gap-2 md:gap-4 mb-4 text-center">
            <div class="bg-green-50 text-green-700 rounded-xl py-2 text-sm md:text-base font-semibold">
              ✅ 正确：{{ correctCount }}
            </div>
            <div class="bg-red-50 text-red-700 rounded-xl py-2 text-sm md:text-base font-semibold">
              ❌ 错误：{{ wrongCount }}
            </div>
            <div class="bg-blue-50 text-blue-700 rounded-xl py-2 text-sm md:text-base font-semibold">
              📊 {{ accuracy }}%
            </div>
          </div>

          <!-- 进度条 -->
          <div class="w-full mb-4">
            <div class="h-2 bg-gray-200 rounded-full overflow-hidden">
              <div class="h-full bg-blue-500 transition-all" :style="{ width: progressPercentage }"></div>
            </div>
            <p class="text-xs md:text-sm text-gray-500 mt-1 text-center">
              第 {{ currentIndex + 1 }} / {{ displayQuestions.length }} 题 · {{ progressPercentage }}
            </p>
          </div>

          <!-- 题目内容 -->
          <div class="flex-1 overflow-y-auto pr-1 custom-scroll">
            <div class="space-y-4">
              <div class="flex flex-wrap items-center justify-between gap-2">
                <div class="flex flex-wrap items-center gap-2">
                  <span class="px-3 py-1 rounded-full text-xs font-semibold" :class="{
                    'bg-blue-100 text-blue-700': showDafult === 1,
                    'bg-yellow-100 text-yellow-700': currentQuestion.type === 2
                  }">
                    {{ showDafult === 1 ? '单选' : currentQuestion.type === 2 ? '填空' : '问答' }}
                  </span>
                  <span v-if="currentQuestion.difficulty" class="px-3 py-1 rounded-full text-xs font-semibold" :class="{
                    'bg-green-100 text-green-700': currentQuestion.difficulty === 'easy',
                    'bg-yellow-100 text-yellow-700': currentQuestion.difficulty === 'medium',
                    'bg-red-100 text-red-700': currentQuestion.difficulty === 'hard'
                  }">
                    {{
                      currentQuestion.difficulty === 'easy'
                        ? '简单'
                        : currentQuestion.difficulty === 'medium'
                          ? '中等'
                          : '困难'
                    }}
                  </span>
                </div>
                <!-- 收藏按钮 -->
                <button class="text-2xl transition-all hover:scale-110"
                  :class="isFavorited ? 'text-yellow-500' : 'text-gray-300'" @click="toggleFavorite">
                  ⭐
                </button>
              </div>

              <h2 class="text-lg md:text-xl font-semibold text-gray-800" v-html="renderMath(currentQuestion.question)">
              </h2>

              <!-- 单选题选项 -->
              <div v-if="showDafult" class="grid grid-cols-1 sm:grid-cols-2 gap-2 md:gap-3 mt-4">
                <div v-for="(text, key) in currentQuestion.options" :key="key"
                  class="flex items-start gap-3 p-3 rounded-xl border cursor-pointer transition-all text-sm md:text-base"
                  :class="{
                    'bg-blue-50 border-blue-400': selected === key,
                    'bg-green-50 border-green-400': selected && key === currentQuestion.answer,
                    'bg-red-50 border-red-400': selected && key === selected && key !== currentQuestion.answer,
                    'hover:bg-gray-50 border-gray-200': !selected
                  }" @click="selectOption(key, currentQuestion.answer)">
                  <span class="font-semibold flex-shrink-0">{{ key }}.</span>
                  <span v-html="renderMath(text)"></span>
                </div>
              </div>

              <!-- 填空题 -->
              <div v-if="currentQuestion.type === 2" class="mt-4">
                <textarea v-model="selected" rows="4" class="w-full border border-gray-300 rounded-lg p-2 text-sm"
                  placeholder="请输入答案"></textarea>
                <button class="mt-2 w-full md:w-auto bg-green-500 text-white rounded-lg px-4 py-2 hover:bg-green-600"
                  @click="saveInput">提交答案</button>
              </div>

              <!-- 解析 -->
              <div v-if="showExplanation" class="bg-gray-50 border border-gray-200 rounded-2xl p-4 mt-6">
                <h3 class="font-semibold text-gray-700 mb-2">题目解析</h3>
                <div v-html="renderMath(currentQuestion.answer)" class="text-gray-700 text-sm md:text-base"></div>
                <div v-if="currentQuestion.knowledgePoints" class="mt-3 flex flex-wrap gap-2">
                  <span v-for="(point, index) in currentQuestion.knowledgePoints" :key="index"
                    class="px-2 py-1 bg-blue-100 text-blue-700 rounded-full text-xs">
                    {{ point }}
                  </span>
                </div>
              </div>
            </div>
          </div>

          <!-- 底部按钮 -->
          <div class="flex justify-between gap-2 mt-6 pt-4 border-t border-gray-100">
            <button
              class="flex-1 bg-gray-200 px-3 md:px-4 py-2 rounded text-sm md:text-base hover:bg-gray-300 disabled:opacity-50"
              :disabled="currentIndex === 0" @click="prevQuestion">上一题</button>
            <button
              class="flex-1 bg-blue-500 text-white px-3 md:px-4 py-2 rounded text-sm md:text-base hover:bg-blue-600"
              @click="toggleExplanation">{{ showExplanation ? '隐藏' : '解析' }}</button>
            <button
              class="flex-1 bg-gray-200 px-3 md:px-4 py-2 rounded text-sm md:text-base hover:bg-gray-300 disabled:opacity-50"
              :disabled="currentIndex === displayQuestions.length - 1" @click="nextQuestion">下一题</button>
          </div>
        </div>
      </main>
    </div>

    <!-- 提示框 -->
    <div v-if="message.text"
      class="fixed bottom-6 left-1/2 transform -translate-x-1/2 bg-black text-white px-4 py-2 rounded shadow z-20">
      {{ message.text }}
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import katex from 'katex'
import 'katex/dist/katex.min.css'

// ====== 科目数据 ======
const exams = ['省考']
const subSubjects = {
  '省考': [
    '创新思维', '计算机网络安全与管理', '论文写作', '中国近现代史纲要',
    'C++程序设计', 'Java语言程序设计', '马克思主义基本原理', '数据结构', '概率论与数理统计'
  ]
}

const selectedExam = ref('省考')
const selectedSubject = ref(subSubjects[selectedExam.value][0])

// ====== 题目数据 ======
const questions = ref([])
const currentIndex = ref(0)
const selected = ref(null)
const showExplanation = ref(false)
const isCollapsed = ref(true)
const isMobile = ref(false)
const answerStatus = ref([])

// ====== 错题集和收藏功能 ======
const favorites = ref({}) // 收藏的题目 { questionIndex: true }
const isWrongOnly = ref(false) // 是否只显示错题
const showFavorites = ref(false) // 是否只显示收藏

// ====== 提示信息 ======
const message = ref({ text: '', timeout: null })
function showMessage(text) {
  clearTimeout(message.value.timeout)
  message.value.text = text
  message.value.timeout = setTimeout(() => message.value.text = '', 2000)
}

// ====== 计算属性 ======
const currentQuestion = computed(() => {
  const idx = getOriginalIndex(currentIndex.value)
  return questions.value[idx] || {}
})

// 显示的题目列表（根据筛选条件）
const displayQuestions = computed(() => {
  if (!isWrongOnly.value && !showFavorites.value) {
    return questions.value
  }
  if (isWrongOnly.value) {
    return questions.value.filter((_, i) => answerStatus.value[i] === 'wrong')
  }
  if (showFavorites.value) {
    return questions.value.filter((_, i) => favorites.value[i])
  }
  return questions.value
})

// 获取原始索引（用于在过滤后的列表中找到原始位置）
function getOriginalIndex(displayIndex) {
  let count = 0
  for (let i = 0; i < questions.value.length; i++) {
    let isInDisplayList = true
    if (isWrongOnly.value) {
      isInDisplayList = answerStatus.value[i] === 'wrong'
    } else if (showFavorites.value) {
      isInDisplayList = favorites.value[i]
    }
    if (isInDisplayList) {
      if (count === displayIndex) return i
      count++
    }
  }
  return 0
}

// 检查当前题是否被收藏
const isFavorited = computed(() => {
  const idx = getOriginalIndex(currentIndex.value)
  return favorites.value[idx] || false
})

const progressPercentage = computed(() => displayQuestions.value.length ? Math.round((currentIndex.value + 1) / displayQuestions.value.length * 100) + '%' : '0%')
const showDafult = computed(() => currentQuestion.value.options ? 1 : 2)
const correctCount = computed(() => answerStatus.value.filter(s => 'correct' === s).length)
const wrongCount = computed(() => answerStatus.value.filter(s => 'wrong' === s).length)
const favoritesCount = computed(() => Object.values(favorites.value).filter(v => v).length)
const accuracy = computed(() => {
  const total = correctCount.value + wrongCount.value
  return total ? ((correctCount.value / total) * 100).toFixed(1) : 0
})

// ====== 本地存储相关 ======
const STORAGE_KEY = (exam, subject) => `quiz_${exam}_${subject}`
const FAVORITES_KEY = (exam, subject) => `favorites_${exam}_${subject}`

function saveData() {
  const key = STORAGE_KEY(selectedExam.value, selectedSubject.value)
  const favKey = FAVORITES_KEY(selectedExam.value, selectedSubject.value)
  localStorage.setItem(key, JSON.stringify({
    answerStatus: answerStatus.value,
    currentIndex: currentIndex.value
  }))
  localStorage.setItem(favKey, JSON.stringify(favorites.value))
}

function loadData() {
  const key = STORAGE_KEY(selectedExam.value, selectedSubject.value)
  const favKey = FAVORITES_KEY(selectedExam.value, selectedSubject.value)
  const saved = localStorage.getItem(key)
  const savedFav = localStorage.getItem(favKey)

  if (saved) {
    const data = JSON.parse(saved)
    answerStatus.value = data.answerStatus || Array(questions.value.length).fill(null)
    currentIndex.value = data.currentIndex || 0
  } else {
    answerStatus.value = Array(questions.value.length).fill(null)
  }

  if (savedFav) {
    favorites.value = JSON.parse(savedFav)
  } else {
    favorites.value = {}
  }
}

// ====== 方法 ======
function renderMath(text) {
  if (!text) return ''
  return text.replace(/\$(.*?)\$/g, (_, eq) => katex.renderToString(eq, { throwOnError: false }))
}

function handleSubjectChange() {
  fetch(`/questions/${selectedExam.value}/${selectedSubject.value}.json`)
    .then(res => res.json())
    .then(data => {
      questions.value = data
      currentIndex.value = 0
      selected.value = null
      loadData()
      showMessage(`已加载 ${data.length} 道题`)
    }).catch(() => showMessage('加载题目失败'))
}

function onExamChange() {
  selectedSubject.value = subSubjects[selectedExam.value][0]
  handleSubjectChange()
}

function selectOption(key, answer) {
  if (!selected.value) {
    selected.value = key
    const originalIdx = getOriginalIndex(currentIndex.value)
    if (key === answer) {
      answerStatus.value[originalIdx] = 'correct'
      showMessage('回答正确')
      saveData()
      setTimeout(nextQuestion, 800)
    } else {
      answerStatus.value[originalIdx] = 'wrong'
      showMessage('回答错误')
      saveData()
    }
  }
}

function saveInput() {
  const originalIdx = getOriginalIndex(currentIndex.value)
  if (selected.value === currentQuestion.value.answer) {
    answerStatus.value[originalIdx] = 'correct'
    showMessage('正确')
  } else {
    answerStatus.value[originalIdx] = 'wrong'
    showMessage('错误')
  }
  saveData()
}

function toggleFavorite() {
  const originalIdx = getOriginalIndex(currentIndex.value)
  favorites.value[originalIdx] = !favorites.value[originalIdx]
  saveData()
  showMessage(favorites.value[originalIdx] ? '已收藏' : '已取消收藏')
}

function toggleWrongOnly(value) {
  isWrongOnly.value = value
  showFavorites.value = false
  currentIndex.value = 0
}

function toggleShowFavorites() {
  showFavorites.value = !showFavorites.value
  isWrongOnly.value = false
  currentIndex.value = 0
}

function nextQuestion() {
  if (currentIndex.value < displayQuestions.value.length - 1) {
    currentIndex.value++
    selected.value = null
    showExplanation.value = false
  }
}

function prevQuestion() {
  if (currentIndex.value > 0) {
    currentIndex.value--
    selected.value = null
    showExplanation.value = false
  }
}

function toggleExplanation() {
  showExplanation.value = !showExplanation.value
}

function jumpToQuestion(originalIdx) {
  // 找到原始索引在显示列表中的位置
  let displayIdx = 0
  for (let i = 0; i < originalIdx; i++) {
    let isInDisplayList = true
    if (isWrongOnly.value) {
      isInDisplayList = answerStatus.value[i] === 'wrong'
    } else if (showFavorites.value) {
      isInDisplayList = favorites.value[i]
    }
    if (isInDisplayList) displayIdx++
  }

  currentIndex.value = displayIdx
  selected.value = null
  showExplanation.value = false
  if (isMobile.value) {
    isCollapsed.value = true
  }
}

function checkMobile() {
  isMobile.value = window.innerWidth < 768
  if (isMobile.value) {
    isCollapsed.value = true
  } else {
    isCollapsed.value = false
  }
}

// 监听答题状态变化自动保存
watch([answerStatus, favorites], () => {
  saveData()
}, { deep: true })

onMounted(() => {
  checkMobile()
  window.addEventListener('resize', checkMobile)
  handleSubjectChange()
})
</script>

<style scoped>
.slide-enter-active,
.slide-leave-active {
  transition: all 0.3s ease;
}

.slide-enter-from,
.slide-leave-to {
  transform: translateX(-100%);
  opacity: 0;
}

.custom-scroll::-webkit-scrollbar {
  width: 6px;
}

.custom-scroll::-webkit-scrollbar-thumb {
  background-color: rgba(0, 0, 0, 0.2);
  border-radius: 3px;
}

.custom-scroll::-webkit-scrollbar-thumb:hover {
  background-color: rgba(0, 0, 0, 0.3);
}
</style>