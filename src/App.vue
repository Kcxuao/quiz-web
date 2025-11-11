<template>
  <div class="flex flex-col items-center min-h-screen bg-gray-100 p-4 md:p-8">
    <div class="flex w-full max-w-6xl gap-4">
      <!-- 左侧题号栏 -->
      <transition name="slide">
        <aside v-show="!isCollapsed" class="w-64 bg-white rounded-2xl shadow-md p-4 flex flex-col">
          <div class="flex justify-between items-center mb-3">
            <h3 class="font-semibold text-gray-700">题号</h3>
            <button class="text-gray-500 hover:text-gray-700 p-1 rounded" @click="isCollapsed = true">←</button>
          </div>
          <div class="grid grid-cols-5 gap-2 overflow-y-auto pr-1 custom-scroll" style="max-height: 60vh;">
            <div v-for="(q, i) in questions" :key="i"
              class="w-9 h-9 flex items-center justify-center text-sm font-medium rounded-full cursor-pointer transition-all"
              :class="{
                'bg-blue-500 text-white': i === currentIndex,
                'bg-green-500 text-white': answerStatus[i] === 'correct',
                'bg-red-500 text-white': answerStatus[i] === 'wrong',
                'bg-gray-100 hover:bg-gray-200 text-gray-600': !answerStatus[i] && i !== currentIndex
              }" @click="jumpToQuestion(i)">
              {{ i + 1 }}
            </div>
          </div>
        </aside>
      </transition>

      <!-- 折叠按钮 -->
      <button v-show="isCollapsed"
        class="fixed left-3 bottom-3 z-30 bg-blue-500 text-white rounded-full w-10 h-10 flex items-center justify-center shadow"
        @click="isCollapsed = false">
        →
      </button>

      <!-- 主答题区域 -->
      <main class="flex-1">
        <div class="bg-white rounded-2xl shadow-lg p-6 md:p-8 h-full flex flex-col">
          <!-- 科目选择 -->
          <div class="flex gap-2 justify-center mb-6">
            <select v-model="selectedExam" class="border border-gray-300 rounded-lg p-2" @change="onExamChange">
              <option v-for="exam in exams" :key="exam" :value="exam">{{ exam }}</option>
            </select>

            <select v-model="selectedSubject" class="border border-gray-300 rounded-lg p-2"
              @change="handleSubjectChange">
              <option v-for="sub in subSubjects[selectedExam]" :key="sub" :value="sub">{{ sub }}</option>
            </select>
          </div>

          <!-- 统计面板 -->
          <div class="grid grid-cols-3 gap-4 mb-4 text-center">
            <div class="bg-green-50 text-green-700 rounded-xl py-2 font-semibold">
              ✅ 正确：{{ correctCount }}
            </div>
            <div class="bg-red-50 text-red-700 rounded-xl py-2 font-semibold">
              ❌ 错误：{{ wrongCount }}
            </div>
            <div class="bg-blue-50 text-blue-700 rounded-xl py-2 font-semibold">
              📊 正确率：{{ accuracy }}%
            </div>
          </div>

          <!-- 进度条 -->
          <div class="w-full mb-4">
            <div class="h-2 bg-gray-200 rounded-full overflow-hidden">
              <div class="h-full bg-blue-500 transition-all" :style="{ width: progressPercentage }"></div>
            </div>
            <p class="text-sm text-gray-500 mt-1 text-center">
              第 {{ currentIndex + 1 }} / {{ questions.length }} 题 · {{ progressPercentage }}
            </p>
          </div>

          <!-- 题目内容 -->
          <div class="flex-1 overflow-y-auto pr-1 custom-scroll">
            <div class="space-y-4">
              <div class="flex items-center gap-2">
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

              <h2 class="text-xl font-semibold text-gray-800" v-html="renderMath(currentQuestion.question)"></h2>

              <!-- 单选题选项 -->
              <div v-if="showDafult" class="grid grid-cols-1 sm:grid-cols-2 gap-3 mt-4">
                <div v-for="(text, key) in currentQuestion.options" :key="key"
                  class="flex items-start gap-3 p-3 rounded-xl border cursor-pointer transition-all" :class="{
                    'bg-blue-50 border-blue-400': selected === key,
                    'bg-green-50 border-green-400': selected && key === currentQuestion.answer,
                    'bg-red-50 border-red-400': selected && key === selected && key !== currentQuestion.answer,
                    'hover:bg-gray-50 border-gray-200': !selected
                  }" @click="selectOption(key, currentQuestion.answer)">
                  <span class="font-semibold">{{ key }}.</span>
                  <span v-html="renderMath(text)"></span>
                </div>
              </div>

              <!-- 填空题 -->
              <div v-if="currentQuestion.type === 2" class="mt-4">
                <textarea v-model="selected" rows="4" class="w-full border border-gray-300 rounded-lg p-2"
                  placeholder="请输入答案"></textarea>
                <button class="mt-2 bg-green-500 text-white rounded-lg px-4 py-2" @click="saveInput">提交答案</button>
              </div>

              <!-- 解析 -->
              <div v-if="showExplanation" class="bg-gray-50 border border-gray-200 rounded-2xl p-4 mt-6">
                <h3 class="font-semibold text-gray-700 mb-2">题目解析</h3>
                <div v-html="renderMath(currentQuestion.answer)" class="text-gray-700"></div>
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
          <div class="flex justify-between mt-6 pt-4 border-t border-gray-100">
            <button class="bg-gray-200 px-4 py-2 rounded" :disabled="currentIndex === 0"
              @click="prevQuestion">上一题</button>
            <button class="bg-blue-500 text-white px-4 py-2 rounded" @click="toggleExplanation">{{ showExplanation ?
              '隐藏解析' : '显示解析' }}</button>
            <button class="bg-gray-200 px-4 py-2 rounded" :disabled="currentIndex === questions.length - 1"
              @click="nextQuestion">下一题</button>
          </div>
        </div>
      </main>
    </div>

    <!-- 提示框 -->
    <div v-if="message.text"
      class="fixed bottom-6 left-1/2 transform -translate-x-1/2 bg-black text-white px-4 py-2 rounded shadow">
      {{ message.text }}
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
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
const isCollapsed = ref(false)
const answerStatus = ref([])

// ====== 提示信息 ======
const message = ref({ text: '', timeout: null })
function showMessage(text) {
  clearTimeout(message.value.timeout)
  message.value.text = text
  message.value.timeout = setTimeout(() => message.value.text = '', 2000)
}

// ====== 计算属性 ======
const currentQuestion = computed(() => questions.value[currentIndex.value] || {})
const progressPercentage = computed(() => questions.value.length ? Math.round((currentIndex.value + 1) / questions.value.length * 100) + '%' : '0%')
const showDafult = computed(() => currentQuestion.value.options ? 1 : 2)
const correctCount = computed(() => answerStatus.value.filter(s => 'correct' === s).length)
const wrongCount = computed(() => answerStatus.value.filter(s => 'wrong' === s).length)
const accuracy = computed(() => {
  const total = correctCount.value + wrongCount.value
  return total ? ((correctCount.value / total) * 100).toFixed(1) : 0
})

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
      answerStatus.value = Array(data.length).fill(null)
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
    if (key === answer) {
      answerStatus.value[currentIndex.value] = 'correct'
      showMessage('回答正确')
      setTimeout(nextQuestion, 800)
    } else {
      answerStatus.value[currentIndex.value] = 'wrong'
      showMessage('回答错误')
    }
  }
}

function saveInput() {
  if (selected.value === currentQuestion.value.answer) {
    answerStatus.value[currentIndex.value] = 'correct'
    showMessage('正确')
  } else {
    answerStatus.value[currentIndex.value] = 'wrong'
    showMessage('错误')
  }
}

function nextQuestion() {
  if (currentIndex.value < questions.value.length - 1) {
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
function toggleExplanation() { showExplanation.value = !showExplanation.value }
function jumpToQuestion(i) { currentIndex.value = i; selected.value = null; showExplanation.value = false }

onMounted(() => {
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
