<template>
  <div class="quiz-app">
    <!-- 科目选择器 -->
    <div class="subject-selector">
      <el-cascader v-model="selectedSubjects" :options="formattedSubjects" :props="cascaderProps" placeholder="请选择科目"
        size="large" clearable @change="handleSubjectChange" />
    </div>

    <!-- 答题卡 -->
    <el-card class="quiz-card" shadow="hover">
      <!-- 顶部进度条 -->
      <div class="quiz-progress">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: progressPercentage }"></div>
        </div>
        <span class="progress-text">
          第 {{ currentIndex + 1 }} 题 / 共 {{ questions.length }} 题 ({{ progressPercentage }})
        </span>
      </div>

      <!-- 题目内容 -->
      <div class="quiz-content">
        <div class=" quiz-meta">
          <el-tag type="info">
            {{
              showDafult === 1 ? '单选' :
                currentQuestion.type === 2 ? "填空" : "问答"
            }}
          </el-tag>

          <el-tag v-if="currentQuestion.difficulty" :type="currentQuestion.difficulty === 'easy' ? 'success' :
            currentQuestion.difficulty === 'medium' ? 'warning' : 'danger'
            ">
            {{
              currentQuestion.difficulty === 'easy' ? '简单' :
                currentQuestion.difficulty === 'medium' ? '中等' : '困难'
            }}
          </el-tag>
        </div>

        <h2 class="question-text" v-html="renderMath(currentQuestion.question)"></h2>

        <!-- 选项区域 -->
        <!-- 单选题 -->
        <div class="options-grid" v-if="showDafult">
          <div v-for="(text, key) in currentQuestion.options" :key="key" class="option-item" :class="{
            'selected': selected === key,
            'correct': selected && key === currentQuestion.answer,
            'wrong': selected && key === selected && key !== currentQuestion.answer
          }" @click="selectOption(key, currentQuestion.answer)">
            <span class="option-key">{{ key }}</span>
            <span class="option-text" v-html="renderMath(text)"></span>
            <span v-if="selected" class="answer-icon">
              <el-icon v-if="key === currentQuestion.answer">
                <CircleCheck />
              </el-icon>
              <el-icon v-else-if="key === selected && key !== currentQuestion.answer">
                <CircleClose />
              </el-icon>
            </span>
          </div>
        </div>

        <!-- 填空题 -->
        <div class="input-text" v-if="currentQuestion.type === 2">
          <el-input v-model="selected" :rows="6" type="textarea" placeholder="请输入答案" />
        </div>

        <!-- 非填空题 只展示解析即可 -->

        <!-- 解析区域 -->
        <div v-if="showExplanation" class="explanation-section">
          <h3>题目解析</h3>
          <div class="explanation-content">
            <data class="question-text" v-html="renderMath(currentQuestion.explanation)"></data>
            <div v-if="currentQuestion.knowledgePoints" class="knowledge-points">
              <h4>相关知识点：</h4>
              <el-tag v-for="(point, index) in currentQuestion.knowledgePoints" :key="index" class="knowledge-tag">
                {{ point }}
              </el-tag>
            </div>
          </div>
        </div>
      </div>

      <!-- 导航按钮 -->
      <div class="quiz-navigation">
        <el-button class="nav-button" :icon="ArrowLeft" :disabled="currentIndex === 0" @click="prevQuestion"
          size="large">
          上一题
        </el-button>

        <el-button type="primary" @click="toggleExplanation" size="large">
          {{ showExplanation ? '隐藏解析' : '显示解析' }}
        </el-button>

        <el-button class="save-button" @click="saveInput" type="primary" size="large" v-if="currentQuestion.type === 2">
          提交答案
        </el-button>

        <el-button class="nav-button" type="primary" :icon="ArrowRight"
          :disabled="currentIndex === questions.length - 1" @click="nextQuestion" size="large">
          下一题
        </el-button>
      </div>
    </el-card>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { ArrowLeft, ArrowRight, CircleCheck, CircleClose } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import katex from 'katex'
import 'katex/dist/katex.min.css'
import 'element-plus/es/components/message/style/css'

// 科目数据
const subjects = ref([
  {
    value: 'test'
  },
  {
    value: '省考', children: [
      { value: '马克思主义基本原理' },
      { value: '中国近现代史纲要' },
      { value: '数据结构' },
      { value: 'C++程序设计' },
      { value: 'Java语言程序设计' },
      { value: '概率论与数理统计' },
    ]
  },

  {
    value: '统考',
    children: [
      {
        value: '高等数学',
        children: [
          { value: '2023年10月真题' }
        ]
      },
    ]
  }
])

// 格式化科目数据以适应级联选择器
const formattedSubjects = computed(() => {
  return subjects.value.map(subject => ({
    // 第一级数据
    value: subject.id || subject.value,
    label: subject.name || subject.value,
    children: subject.children ? subject.children.map(child => ({
      // 第二级数据
      value: child.id || child.value,
      label: child.name || child.value,
      children: child.children ? child.children.map(grandChild => ({
        // 第三级数据
        value: grandChild.id || grandChild.value,
        label: grandChild.name || grandChild.value,
      })) : null
    })) : null
  }))
})

const showDafult = computed(() => {
  if (!currentQuestion.value.hasOwnProperty('type')) {
    return 1
  };
  return currentQuestion.value.type;
})

const cascaderProps = {
  expandTrigger: 'hover',
  value: 'value',
  label: 'label',
  children: 'children'
}

const selectedSubject = ref() // 默认选择
const selectedSubjects = ref([])

// 题目数据
const questions = ref([
  {
    id: 1,
    question: '默认题目',
    options: {
      A: '默认选项A',
      B: '默认选项B',
      C: '默认选项C',
      D: '默认选项D'
    },
    answer: 'A',
    explanation: '这是默认题目的解析',
    difficulty: 'easy'
  }
])

// 根据选择的科目加载题目
const handleSubjectChange = async (value) => {
  try {
    isLoading.value = true
    currentIndex.value = 0
    selected.value = null

    // 获取最后一级的科目值
    const api = `/questions/${value.join("/")}.json`
    const response = await fetch(api)
    if (!response.ok) throw new Error('题目加载失败')
    console.log(api);


    const data = await response.json()
    questions.value = data

    // 如果没有题目，显示默认提示
    if (data.length === 0) {
      questions.value = [{
        id: 0,
        question: '当前科目暂无题目',
        options: { A: '请选择其他科目' },
        answer: 'A',
        explanation: '请尝试选择其他科目',
        difficulty: 'easy'
      }]
      ElMessage.warning('当前科目暂无题目')
    } else {
      ElMessage.success(`已加载 ${data.length} 道题目`)
    }
  } catch (error) {
    console.error('加载题目失败:', error)
    questions.value = [{
      id: 0,
      question: '题目加载失败',
      options: { A: '请稍后重试' },
      answer: 'A',
      explanation: error.message,
      difficulty: 'easy'
    }]
    ElMessage.error('题目加载失败，请稍后重试')
  } finally {
    isLoading.value = false
  }
}

// 渲染数学公式的函数
const renderMath = (text) => {
  if (!text) return ''

  // 使用正则表达式匹配行内公式和块级公式
  const inlineRegex = /\$(.*?)\$/g
  const blockRegex = /\$\$(.*?)\$\$/g

  let html = text
    // 替换行内公式
    .replace(inlineRegex, (match, equation) => {
      try {
        return katex.renderToString(equation, {
          throwOnError: false,
          displayMode: false
        })
      } catch (e) {
        console.error('KaTeX渲染错误:', e)
        return match
      }
    })
    // 替换块级公式
    .replace(blockRegex, (match, equation) => {
      try {
        return katex.renderToString(equation, {
          throwOnError: false,
          displayMode: true
        })
      } catch (e) {
        console.error('KaTeX渲染错误:', e)
        return match
      }
    })

  return html
}

// 答题状态
const currentIndex = ref(0)
const selected = ref(null)
const showExplanation = ref(false)
const isLoading = ref(false)

// 计算属性
const currentQuestion = computed(() => questions.value[currentIndex.value])
const progressPercentage = computed(() => {
  return Math.round((currentIndex.value + 1) / questions.value.length * 100) + '%'
})

// 选择题答案判断
function selectOption(key, answer) {
  if (!selected.value) {
    selected.value = key

    // 自动跳转下一题（仅在答对时）
    if (key === answer) {
      setTimeout(() => {
        if (currentIndex.value < questions.value.length - 1) {
          nextQuestion()
        } else {
          ElMessage.success('恭喜完成所有题目！')
        }
      }, 800) // 0.8秒延迟让用户看到反馈
    }
  }
}

// 填空题答案判断
function saveInput() {
  if (selected.value === currentQuestion.value.answer) {
    ElMessage.success("回答正确")
    setTimeout(() => {
      if (currentIndex.value < questions.value.length - 1) {
        nextQuestion()
      } else {
        ElMessage.success('恭喜完成所有题目！')
      }
    }, 800) // 0.8秒延迟让用户看到反馈
  } else {
    ElMessage.error('回答错误')
  }

}

function nextQuestion() {
  if (currentIndex.value < questions.value.length - 1) {
    currentIndex.value++
    selected.value = null
    showExplanation.value = false
    // 滚动到题目顶部
    document.querySelector('.quiz-content')?.scrollTo(0, 0)
  }
}

function prevQuestion() {
  if (currentIndex.value > 0) {
    currentIndex.value--
    selected.value = null
    showExplanation.value = false
    document.querySelector('.quiz-content')?.scrollTo(0, 0)
  }
}

function toggleExplanation() {
  showExplanation.value = !showExplanation.value
}

// 初始化加载默认科目题目
onMounted(() => {
  handleSubjectChange([selectedSubject.value])
})
</script>

<style lang="less" scoped>
// 定义变量
@primary-color: #4a90e2;
@success-color: #75d047;
@warning-color: #e6a23c;
@danger-color: #f56c6c;
@border-radius-base: 4px;
@transition-duration: 0.3s;

// 混入(Mixins)
.border-radius(@radius: @border-radius-base) {
  border-radius: @radius;
}

.transition(@property: all, @duration: @transition-duration, @timing: ease) {
  transition: @property @duration @timing;
}

.box-shadow(@x: 0, @y: 2px, @blur: 8px, @color: rgba(0, 0, 0, 0.1)) {
  box-shadow: @x @y @blur @color;
}

// 主容器样式
.quiz-app {
  max-width: 800px;
  margin: 0 auto;
  padding: 24px;
  background-color: #f5f7fa;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.subject-selector {
  width: 100%;
  max-width: 800px;
  margin-bottom: 20px;

  .el-cascader {
    width: 100% !important;
  }
}

.quiz-card {
  width: 100%;
  .border-radius(12px);
  overflow: hidden;
  .transition();
  .box-shadow();
}

// 进度条样式
.quiz-progress {
  margin-bottom: 28px;

  .progress-bar {
    height: 8px;
    background-color: #e4e7ed;
    .border-radius();
    margin-bottom: 8px;
    overflow: hidden;

    .progress-fill {
      height: 100%;
      background: linear-gradient(90deg, @primary-color, lighten(@primary-color, 10%));
      .border-radius();
      .transition(width);
    }
  }

  .progress-text {
    font-size: 14px;
    color: #606266;
    display: block;
    text-align: center;
  }
}

// 题目内容
.quiz-content {
  .question-text {
    font-size: 18px;
    line-height: 1.6;
    color: #303133;
    margin-bottom: 24px;
    font-weight: 500;
  }

  .options-grid {
    display: grid;
    grid-template-columns: 1fr;
    gap: 12px;
    margin-bottom: 32px;

    .option-item {
      padding: 16px 20px;
      border: 1px solid #e4e7ed;
      .border-radius(8px);
      cursor: pointer;
      .transition();
      display: flex;
      align-items: center;
      position: relative;
      background-color: white;

      &:hover {
        background-color: #f8fafc;
        border-color: #c0c4cc;
      }

      &.selected {
        background-color: #f0f7ff;
        border-color: lighten(@primary-color, 20%);
      }

      &.correct {
        background-color: #f0f9eb;
        border-color: lighten(@success-color, 20%);
      }

      &.wrong {
        background-color: #fef0f0;
        border-color: lighten(@danger-color, 20%);
      }

      .option-key {
        font-weight: bold;
        margin-right: 12px;
        color: @primary-color;
        min-width: 20px;
      }

      .option-text {
        flex: 1;
        text-align: left;
      }

      .answer-icon {
        margin-left: 8px;
        font-size: 18px;

        .el-icon {
          vertical-align: middle;
        }
      }
    }
  }
}

// 解析区域
.explanation-section {
  margin-top: 30px;
  padding: 15px;
  background-color: #f8f8f8;
  .border-radius();

  h3 {
    margin-top: 0;
    margin-bottom: 15px;
    color: #333;
  }

  .knowledge-tag {
    margin-right: 8px;
    margin-bottom: 8px;
  }
}

// 导航按钮
.quiz-navigation {
  display: flex;
  justify-content: space-between;
  margin-top: 24px;

  .nav-button {
    min-width: 120px;
    .transition();

    &:active {
      transform: scale(0.98);
    }
  }

  .save-button {
    background-color: #75d047;
  }
}

// 数学公式样式
:deep(.katex) {
  font-size: 1.1em;
}

:deep(.katex-display) {
  margin: 0.5em 0;
  overflow: auto hidden;
  padding: 0.2em 0;
}
</style>