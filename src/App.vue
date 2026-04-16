<template>
  <div
    class="flex flex-col items-center min-h-screen p-4 md:p-8 transition-all duration-500"
    :class="[
      darkMode ? 'bg-slate-900 text-slate-100' : 'bg-slate-50 text-slate-800',
      !isCollapsed && isMobile ? 'overflow-hidden' : '',
    ]"
  >
    <transition name="fade">
      <div
        v-if="!isCollapsed && isMobile"
        class="fixed inset-0 bg-black/60 z-[40] backdrop-blur-sm"
        @click="isCollapsed = true"
      ></div>
    </transition>

    <div class="flex w-full max-w-6xl gap-8 relative">
      <transition name="slide">
        <aside
          v-show="!isCollapsed"
          class="flex flex-col transition-all z-[50] shadow-2xl"
          :class="[
            isMobile
              ? 'fixed inset-y-0 left-0 w-72 rounded-none p-6'
              : 'w-72 rounded-[2rem] p-6 sticky top-8 h-[calc(100vh-4rem)]',
            darkMode
              ? 'bg-slate-800 border-r border-slate-700'
              : 'bg-white border border-slate-200',
          ]"
        >
          <div class="flex items-center justify-between mb-8">
            <h3
              class="font-black text-xl tracking-tight italic text-indigo-500"
            >
              导航目录.
            </h3>
            <button
              @click="isCollapsed = true"
              class="p-2 rounded-xl hover:bg-slate-200 dark:hover:bg-slate-700 transition-colors"
            >
              ✕
            </button>
          </div>
          <div
            class="grid grid-cols-5 gap-3 overflow-y-auto custom-scroll pr-2"
          >
            <div
              v-for="(q, i) in displayQuestions"
              :key="'nav-' + i"
              class="aspect-square flex items-center justify-center text-xs font-bold rounded-xl cursor-pointer transition-all border-2"
              :class="getCircleClass(i)"
              @click="jumpToQuestion(i)"
            >
              {{ getDisplayNumber(i) }}
            </div>
          </div>
        </aside>
      </transition>

      <main class="flex-1 w-full max-w-3xl flex flex-col gap-6">
        <header
          class="rounded-[1.5rem] shadow-sm p-3 flex items-center justify-between transition-all border"
          :class="
            darkMode
              ? 'bg-slate-800 border-slate-700 text-white'
              : 'bg-white border-slate-200 text-slate-800'
          "
        >
          <div class="flex items-center gap-3">
            <button
              @click.stop="isCollapsed = !isCollapsed"
              class="p-3 rounded-2xl transition-all active:scale-90"
              :class="
                darkMode
                  ? 'bg-slate-700 text-indigo-400 hover:bg-slate-600'
                  : 'bg-indigo-50 text-indigo-600 hover:bg-indigo-100'
              "
            >
              <span class="text-xl">{{ isCollapsed ? "☰" : "⬅" }}</span>
            </button>
            <h3>Quiz</h3>
            <div class="hidden sm:block">
              <span
                class="text-[10px] font-black uppercase tracking-[0.2em] text-indigo-500"
                >{{ selectedExam || "同步中..." }}</span
              >
              <h1 class="font-bold text-sm opacity-80">
                {{ isExamMode ? "模拟考试模式" : "自主练习模式" }}
              </h1>
            </div>
          </div>

          <div
            v-if="isExamMode"
            class="px-5 py-2 bg-rose-500 text-white rounded-2xl flex items-center gap-2 shadow-lg shadow-rose-500/20"
          >
            <span class="animate-pulse text-lg">⏱</span>
            <span class="font-mono font-black">{{
              formatTime(examTimer)
            }}</span>
          </div>

          <div class="flex items-center gap-2">
            <button
              @click="showStats = true"
              class="p-3 rounded-2xl transition-all active:scale-90 hover:bg-indigo-500/10 text-indigo-500"
            >
              📊
            </button>
            <button
              @click="toggleDarkMode"
              class="p-3 rounded-2xl transition-all active:scale-90 hover:bg-amber-500/10 text-amber-500"
            >
              {{ darkMode ? "☀️" : "🌙" }}
            </button>

            <div class="relative">
              <button
                @click.stop="showMenu = !showMenu"
                class="p-3 rounded-2xl transition-all active:scale-90"
                :class="darkMode ? 'bg-slate-700' : 'bg-slate-100'"
              >
                ⋮
              </button>
              <transition name="menu">
                <div
                  v-if="showMenu"
                  class="absolute right-0 mt-4 w-52 rounded-2xl shadow-2xl border overflow-hidden z-[200]"
                  :class="
                    darkMode
                      ? 'bg-slate-800 border-slate-700'
                      : 'bg-white border-slate-200'
                  "
                >
                  <button
                    @click="clearCache"
                    class="w-full px-5 py-4 text-left hover:bg-rose-500/10 transition-colors flex items-center gap-3"
                  >
                    <span class="grayscale">🗑️</span>
                    <span class="font-bold text-rose-500">重置答题进度</span>
                  </button>
                  <button
                    @click="clearFavorites"
                    class="w-full px-5 py-4 text-left hover:bg-indigo-500/10 transition-colors border-t border-slate-100 dark:border-slate-700 flex items-center gap-3"
                  >
                    <span class="grayscale">⭐</span>
                    <span class="font-bold">清空我的收藏</span>
                  </button>
                </div>
              </transition>
            </div>
          </div>
        </header>

        <section
          v-if="!isExamMode"
          class="grid grid-cols-1 md:grid-cols-2 gap-4"
        >
          <div class="flex flex-col gap-2">
            <label class="text-[10px] font-black uppercase opacity-40 ml-4"
              >题库中心 Subject Library</label
            >
            <div class="flex gap-2">
              <select
                v-model="selectedExam"
                @change="handleExamChange"
                class="flex-1 appearance-none border-2 rounded-2xl p-3 px-5 font-bold outline-none transition-all cursor-pointer"
                :class="
                  darkMode
                    ? 'bg-slate-800 border-slate-700 text-slate-100'
                    : 'bg-white border-slate-200 focus:border-indigo-500 text-slate-800'
                "
              >
                <option
                  v-for="exam in exams"
                  :key="exam"
                  :value="exam"
                  :class="darkMode ? 'bg-slate-800' : 'bg-white'"
                >
                  {{ exam }}
                </option>
              </select>
              <select
                v-model="selectedSubject"
                @change="handleSubjectChange"
                class="flex-1 appearance-none border-2 rounded-2xl p-3 px-5 font-bold outline-none transition-all cursor-pointer"
                :class="
                  darkMode
                    ? 'bg-slate-800 border-slate-700 text-slate-100'
                    : 'bg-white border-slate-200 focus:border-indigo-500 text-slate-800'
                "
              >
                <option
                  v-for="sub in subSubjects[selectedExam]"
                  :key="sub"
                  :value="sub"
                  :class="darkMode ? 'bg-slate-800' : 'bg-white'"
                >
                  {{ sub }}
                </option>
              </select>
            </div>
          </div>
          <div class="flex flex-col gap-2">
            <label class="text-[10px] font-black uppercase opacity-40 ml-4"
              >筛选模式 Filter Mode</label
            >
            <div class="flex gap-2">
              <button
                v-for="f in [
                  { id: 'all', l: '全部', c: 'indigo' },
                  { id: 'wrong', l: '错题', c: 'rose' },
                  { id: 'fav', l: '收藏', c: 'amber' },
                ]"
                :key="f.id"
                @click="setFilter(f.id)"
                class="flex-1 py-3 rounded-2xl text-xs font-black transition-all"
                :class="getFilterClass(f)"
              >
                {{ f.l }}
              </button>
            </div>
          </div>
        </section>

        <div
          class="rounded-[2.5rem] shadow-xl p-6 md:p-12 flex flex-col transition-all min-h-[500px] border relative"
          :class="
            darkMode
              ? 'bg-slate-800 border-slate-700'
              : 'bg-white border-slate-100'
          "
          @touchstart="handleTouchStart"
          @touchend="handleTouchEnd"
        >
          <div v-if="currentQuestion" class="flex-1 flex flex-col">
            <div class="flex items-center justify-between mb-10">
              <div class="flex items-center gap-3">
                <span
                  class="px-4 py-1.5 rounded-full text-[10px] font-black tracking-widest text-white shadow-sm"
                  :class="isMulti ? 'bg-indigo-500' : 'bg-emerald-500'"
                  >{{ typeLabel }}</span
                >
                <span class="text-xs font-mono font-bold opacity-30"
                  >{{ currentIndex + 1 }} / {{ displayQuestions.length }}</span
                >
              </div>
              <div
                class="w-32 h-1.5 bg-slate-100 dark:bg-slate-700 rounded-full overflow-hidden"
              >
                <div
                  class="h-full bg-indigo-500 transition-all duration-700"
                  :style="{ width: progressPercentage }"
                ></div>
              </div>
            </div>

            <h2
              class="text-xl md:text-2xl font-bold leading-snug mb-10"
              v-html="renderMath(currentQuestion.question)"
            ></h2>

            <div v-if="hasOptions" class="grid gap-4">
              <div
                v-for="(text, key) in currentQuestion.options"
                :key="currentIndex + '-' + key"
                @click="handleSelect(key)"
                class="group flex items-center gap-5 p-5 rounded-3xl border-2 cursor-pointer transition-all active:scale-[0.98] relative overflow-hidden"
                :class="getOptionClass(key)"
              >
                <div
                  class="w-10 h-10 shrink-0 rounded-2xl border-2 flex items-center justify-center font-black transition-all"
                  :class="getOptionCircleClass(key)"
                >
                  {{ key }}
                </div>
                <div
                  class="text-sm md:text-base font-semibold leading-relaxed"
                  v-html="renderMath(text)"
                ></div>
              </div>
            </div>

            <button
              v-if="
                isMulti &&
                (isExamMode ||
                  answerStatus[currentQuestion.originalIndex] !== 'correct')
              "
              @click="submitMulti"
              class="mt-10 w-full py-5 rounded-[1.5rem] font-black tracking-widest bg-indigo-600 text-white shadow-xl shadow-indigo-600/20 hover:bg-indigo-700 active:scale-95 transition-all"
            >
              提交答案 SUBMIT
            </button>

            <transition name="fade">
              <div
                v-if="showExplanation && !isExamMode"
                class="mt-10 p-8 rounded-[2rem] bg-indigo-50 dark:bg-indigo-900/40 border-2 border-indigo-100 dark:border-indigo-800"
              >
                <div class="flex items-center gap-2 mb-4">
                  <span class="text-xl">💡</span>
                  <span
                    class="font-black text-indigo-600 dark:text-indigo-400 tracking-tighter uppercase"
                    >题目解析 Explanation</span
                  >
                </div>
                <div
                  class="text-sm leading-loose opacity-80 mb-6 font-medium"
                  v-html="
                    renderMath(
                      currentQuestion.explanation || '该题目暂无详细解析',
                    )
                  "
                ></div>
                <div
                  class="flex items-center gap-3 p-4 bg-white dark:bg-slate-800 rounded-2xl border border-indigo-100 dark:border-indigo-800"
                >
                  <span class="text-xs font-bold opacity-50">正确答案:</span>
                  <span class="font-mono font-black text-emerald-500 text-lg">{{
                    currentQuestion.answer
                  }}</span>
                </div>
              </div>
            </transition>
          </div>

          <footer
            class="grid grid-cols-4 gap-3 mt-12 pt-8 border-t border-slate-100 dark:border-slate-700"
          >
            <button
              @click="prevQuestion"
              :disabled="currentIndex === 0"
              class="py-5 rounded-2xl font-black bg-slate-100 dark:bg-slate-700 text-slate-500 dark:text-slate-300 disabled:opacity-20 hover:bg-indigo-500 hover:text-white transition-all active:scale-90 text-xs uppercase"
            >
              上一题
            </button>
            <button
              v-if="!isExamMode"
              @click="toggleExplanation"
              class="py-5 rounded-2xl font-black transition-all active:scale-90 text-xs uppercase"
              :class="
                showExplanation
                  ? 'bg-indigo-500 text-white'
                  : 'bg-indigo-100 text-indigo-600 dark:bg-indigo-900/60 dark:text-indigo-300'
              "
            >
              解析
            </button>
            <button
              v-else
              @click="finishExam"
              class="py-5 rounded-2xl font-black bg-rose-600 text-white shadow-lg shadow-rose-600/20 active:scale-90 text-xs uppercase"
            >
              交卷
            </button>
            <button
              @click="toggleFavorite"
              class="py-5 rounded-2xl font-black transition-all active:scale-90"
              :class="
                isFavorited
                  ? 'bg-amber-400 text-white shadow-lg shadow-amber-400/20'
                  : 'bg-slate-100 dark:bg-slate-700 text-slate-400'
              "
            >
              {{ isFavorited ? "★" : "☆" }}
            </button>
            <button
              @click="nextQuestion"
              :disabled="currentIndex === displayQuestions.length - 1"
              class="py-5 rounded-2xl font-black bg-slate-100 dark:bg-slate-700 text-slate-500 dark:text-slate-300 disabled:opacity-20 hover:bg-indigo-500 hover:text-white transition-all active:scale-90 text-xs uppercase"
            >
              下一题
            </button>
          </footer>
        </div>
      </main>
    </div>

    <transition name="fade">
      <div
        v-if="showStats"
        class="fixed inset-0 z-[300] flex items-center justify-center p-4 backdrop-blur-xl bg-black/60"
        @click="showStats = false"
      >
        <div
          class="relative w-full max-w-2xl rounded-[3rem] shadow-2xl p-8 md:p-14 transition-all border"
          :class="
            darkMode
              ? 'bg-slate-800 border-slate-700 text-white'
              : 'bg-white border-white/20 text-slate-800'
          "
          @click.stop
        >
          <div class="flex justify-between items-start mb-10">
            <div>
              <h2 class="text-4xl font-black italic tracking-tighter">
                学习报告.
              </h2>
              <p
                class="text-xs font-bold opacity-40 uppercase tracking-widest mt-2"
              >
                {{ selectedSubject }}
              </p>
            </div>
            <button
              @click="showStats = false"
              class="p-4 rounded-full bg-slate-100 dark:bg-slate-700 hover:rotate-90 transition-all text-slate-500"
            >
              ✕
            </button>
          </div>

          <div class="grid grid-cols-2 md:grid-cols-4 gap-4 mb-10">
            <div
              v-for="(v, l) in {
                总题量: stats.total,
                已完成: stats.done,
                正确率: stats.accuracy + '%',
                收藏: favoritesCount,
              }"
              class="p-5 rounded-[2rem] border transition-colors"
              :class="
                darkMode
                  ? 'bg-slate-900/50 border-slate-700 text-slate-100'
                  : 'bg-slate-50 border-slate-100 text-slate-800'
              "
            >
              <div
                class="text-[9px] font-black opacity-30 tracking-widest mb-1"
              >
                {{ l }}
              </div>
              <div class="text-2xl font-black tracking-tight">{{ v }}</div>
            </div>
          </div>

          <div class="space-y-8">
            <div
              class="h-4 w-full bg-slate-100 dark:bg-slate-900 rounded-full overflow-hidden shadow-inner"
            >
              <div
                class="h-full bg-gradient-to-r from-indigo-500 to-purple-500 transition-all duration-1000"
                :style="{ width: stats.progress + '%' }"
              ></div>
            </div>
            <div class="flex gap-4">
              <div
                class="flex-1 p-8 rounded-[2.5rem] bg-emerald-500/10 border border-emerald-500/20 text-center"
              >
                <div class="text-emerald-500 text-5xl font-black">
                  {{ stats.correct }}
                </div>
                <div
                  class="text-[10px] font-black opacity-40 mt-3 tracking-widest"
                >
                  正确 CORRECT
                </div>
              </div>
              <div
                class="flex-1 p-8 rounded-[2.5rem] bg-rose-500/10 border border-rose-500/20 text-center"
              >
                <div class="text-rose-500 text-5xl font-black">
                  {{ stats.wrong }}
                </div>
                <div
                  class="text-[10px] font-black opacity-40 mt-3 tracking-widest"
                >
                  错误 WRONG
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </transition>

    <transition name="menu">
      <div
        v-if="message.text"
        class="fixed bottom-10 left-1/2 -translate-x-1/2 px-8 py-4 rounded-2xl shadow-2xl z-[500] font-black text-sm tracking-widest text-white bg-indigo-600"
      >
        {{ message.text }}
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from "vue";
import katex from "katex";
import "katex/dist/katex.min.css";

// --- 响应式状态 ---
const darkMode = ref(false);
const isCollapsed = ref(true);
const isMobile = ref(false);
const showMenu = ref(false);
const showStats = ref(false);
const message = ref({ text: "", timeout: null });
const exams = ref([]);
const subSubjects = ref({});
const selectedExam = ref("");
const selectedSubject = ref("");
const questions = ref([]);
const currentIndex = ref(0);
const selected = ref(null);
const showExplanation = ref(false);
const answerStatus = ref([]);
const favorites = ref({});
const filterMode = ref("all");
const isExamMode = ref(false);
const examTimer = ref(3600);
const examInterval = ref(null);
const examAnswers = ref({});

// --- 动态题库加载 ---
async function fetchLibraryIndex() {
  try {
    const res = await fetch("/questions/index.json");
    const data = await res.json();
    subSubjects.value = data;
    exams.value = Object.keys(data);
    if (exams.value.length) {
      selectedExam.value = exams.value[0];
      if (subSubjects.value[selectedExam.value].length) {
        selectedSubject.value = subSubjects.value[selectedExam.value][0];
        handleSubjectChange();
      }
    }
  } catch (e) {
    showMessage("⚠️ INDEX LOAD FAILED");
  }
}

function handleExamChange() {
  if (subSubjects.value[selectedExam.value].length) {
    selectedSubject.value = subSubjects.value[selectedExam.value][0];
    handleSubjectChange();
  }
}

function handleSubjectChange() {
  if (!selectedExam.value || !selectedSubject.value) return;
  fetch(`/questions/${selectedExam.value}/${selectedSubject.value}.json`)
    .then((r) => r.json())
    .then((d) => {
      questions.value = d;
      loadData();
      currentIndex.value = 0;
      resetState();
    })
    .catch(() => {
      questions.value = [];
      showMessage("⚠️ FILE LOAD FAILED");
    });
}

// --- 计算属性 ---
const processedQuestions = computed(() =>
  questions.value.map((q, i) => ({ ...q, originalIndex: i })),
);
const displayQuestions = computed(() => {
  if (isExamMode.value) return processedQuestions.value;
  if (filterMode.value === "wrong")
    return processedQuestions.value.filter(
      (q) => answerStatus.value[q.originalIndex] === "wrong",
    );
  if (filterMode.value === "fav")
    return processedQuestions.value.filter(
      (q) => favorites.value[q.originalIndex],
    );
  return processedQuestions.value;
});
const currentQuestion = computed(
  () => displayQuestions.value[currentIndex.value] || null,
);
const hasOptions = computed(() => !!currentQuestion.value?.options);
const isMulti = computed(
  () => currentQuestion.value?.answer?.length > 1 && hasOptions.value,
);
const typeLabel = computed(() => (isMulti.value ? "多选题" : "单选题"));
const isFavorited = computed(
  () =>
    currentQuestion.value &&
    favorites.value[currentQuestion.value.originalIndex],
);
const progressPercentage = computed(() =>
  displayQuestions.value.length
    ? ((currentIndex.value + 1) / displayQuestions.value.length) * 100 + "%"
    : "0%",
);
const stats = computed(() => {
  const t = questions.value.length,
    d = answerStatus.value.filter((s) => s !== null).length;
  const c = answerStatus.value.filter((s) => s === "correct").length,
    w = answerStatus.value.filter((s) => s === "wrong").length;
  return {
    total: t,
    done: d,
    correct: c,
    wrong: w,
    accuracy: d ? Math.round((c / d) * 100) : 0,
    progress: t ? Math.round((d / t) * 100) : 0,
  };
});
const favoritesCount = computed(
  () => Object.values(favorites.value).filter(Boolean).length,
);

// --- 交互逻辑 ---
function handleSelect(key) {
  if (!currentQuestion.value) return;
  const oIdx = currentQuestion.value.originalIndex;

  if (isMulti.value) {
    answerStatus.value[oIdx] = null;
    if (!Array.isArray(selected.value)) selected.value = [];
    const idx = selected.value.indexOf(key);
    idx > -1 ? selected.value.splice(idx, 1) : selected.value.push(key);
    return;
  }

  if (isExamMode.value) {
    examAnswers.value[oIdx] = key;
    selected.value = key;
    setTimeout(nextQuestion, 300);
    return;
  }
  if (answerStatus.value[oIdx] === "correct") return;
  selected.value = key;
  if (key === currentQuestion.value.answer) {
    answerStatus.value[oIdx] = "correct";
    showMessage("✅ 回答正确");
    setTimeout(nextQuestion, 600);
  } else {
    answerStatus.value[oIdx] = "wrong";
    showMessage("❌ 回答错误");
  }
}

function submitMulti() {
  if (!selected.value?.length) return showMessage("请至少选择一个");
  const oIdx = currentQuestion.value.originalIndex;
  const user = selected.value.sort().join(""),
    correct = currentQuestion.value.answer.split("").sort().join("");
  if (isExamMode.value) {
    examAnswers.value[oIdx] = user;
    setTimeout(nextQuestion, 300);
  } else {
    if (user === correct) {
      answerStatus.value[oIdx] = "correct";
      showMessage("✅ 回答正确");
      setTimeout(nextQuestion, 600);
    } else {
      answerStatus.value[oIdx] = "wrong";
      showMessage("❌ 回答错误");
    }
  }
}

// --- 样式逻辑 ---
function getOptionClass(key) {
  const isSel = Array.isArray(selected.value)
    ? selected.value.includes(key)
    : selected.value === key;
  const s = currentQuestion.value
    ? answerStatus.value[currentQuestion.value.originalIndex]
    : null;

  if (s === "correct" && currentQuestion.value.answer.includes(key))
    return "bg-emerald-500 border-emerald-500 text-white shadow-lg";
  if (isSel) {
    if (s === "wrong")
      return "bg-rose-500 border-rose-500 text-white shadow-lg";
    return "bg-indigo-600 border-indigo-600 text-white shadow-lg";
  }
  return darkMode.value
    ? "bg-slate-700/50 border-slate-700 hover:bg-slate-700 hover:border-indigo-500 text-slate-200"
    : "bg-white border-slate-100 hover:border-indigo-300 text-slate-700";
}

function getOptionCircleClass(key) {
  const isSel = Array.isArray(selected.value)
    ? selected.value.includes(key)
    : selected.value === key;
  if (isSel) return "border-white/50 text-white";
  return darkMode.value
    ? "border-slate-600 text-slate-500"
    : "border-slate-200 text-slate-400";
}

function getFilterClass(f) {
  const active = filterMode.value === f.id;
  if (active)
    return `bg-${f.c}-500 text-white shadow-lg shadow-${f.c}-500/30 scale-105`;
  return darkMode.value
    ? "bg-slate-800 border border-slate-700 text-slate-400"
    : "bg-white border border-slate-200 text-slate-500";
}

function getCircleClass(i) {
  const oIdx = displayQuestions.value[i].originalIndex,
    s = answerStatus.value[oIdx];
  if (i === currentIndex.value)
    return "border-indigo-500 bg-indigo-500 text-white scale-110 shadow-lg";
  if (s === "correct")
    return "border-emerald-500 text-emerald-500 bg-emerald-500/10";
  if (s === "wrong") return "border-rose-500 text-rose-500 bg-rose-500/10";
  return darkMode.value
    ? "border-transparent bg-slate-700 text-slate-500 opacity-80"
    : "border-transparent bg-slate-100 text-slate-400 opacity-60";
}

// --- 通用逻辑 ---
function clearCache() {
  if (confirm("彻底重置进度？")) {
    answerStatus.value = Array(questions.value.length).fill(null);
    saveData();
    showMenu.value = false;
    showMessage("已重置进度");
  }
}
function clearFavorites() {
  if (confirm("清空收藏夹？")) {
    favorites.value = {};
    saveData();
    showMenu.value = false;
    showMessage("已清空收藏");
  }
}
function toggleDarkMode() {
  darkMode.value = !darkMode.value;
  localStorage.setItem("darkMode", darkMode.value);
}
function toggleFavorite() {
  favorites.value[currentQuestion.value.originalIndex] = !isFavorited.value;
  saveData();
}
function toggleExplanation() {
  showExplanation.value = !showExplanation.value;
}
function showMessage(t) {
  clearTimeout(message.value.timeout);
  message.value.text = t;
  message.value.timeout = setTimeout(() => (message.value.text = ""), 1500);
}
function formatTime(s) {
  return `${Math.floor(s / 60)}:${(s % 60).toString().padStart(2, "0")}`;
}
function renderMath(t) {
  return t
    ? t
        .replace(/\$(.*?)\$/g, (_, e) =>
          katex.renderToString(e, { throwOnError: false }),
        )
        .replace(/\n/g, "<br>")
    : "";
}
function nextQuestion() {
  if (currentIndex.value < displayQuestions.value.length - 1) {
    currentIndex.value++;
    resetState();
  }
}
function prevQuestion() {
  if (currentIndex.value > 0) {
    currentIndex.value--;
    resetState();
  }
}
function resetState() {
  const oIdx = currentQuestion.value?.originalIndex;
  selected.value = isExamMode.value ? examAnswers.value[oIdx] || null : null;
  showExplanation.value = false;
}
function jumpToQuestion(i) {
  currentIndex.value = i;
  resetState();
  if (isMobile.value) isCollapsed.value = true;
}
function getDisplayNumber(i) {
  return displayQuestions.value[i].originalIndex + 1;
}
function setFilter(m) {
  filterMode.value = m;
  currentIndex.value = 0;
  resetState();
}

// --- 考试逻辑 ---
function startExam() {
  isExamMode.value = true;
  examAnswers.value = {};
  examTimer.value = 3600;
  currentIndex.value = 0;
  resetState();
  examInterval.value = setInterval(() => {
    examTimer.value > 0 ? examTimer.value-- : finishExam();
  }, 1000);
}
function finishExam() {
  clearInterval(examInterval.value);
  questions.value.forEach((q, i) => {
    const a = examAnswers.value[i];
    if (a === q.answer) answerStatus.value[i] = "correct";
    else if (a) answerStatus.value[i] = "wrong";
  });
  isExamMode.value = false;
  showStats.value = true;
  resetState();
}

// --- 生命周期 & 持久化 ---
const KEY = (s) => `v7_quiz_${s}`;
function saveData() {
  if (selectedSubject.value)
    localStorage.setItem(
      KEY(selectedSubject.value),
      JSON.stringify({ status: answerStatus.value, favs: favorites.value }),
    );
}
function loadData() {
  const d = JSON.parse(
    localStorage.getItem(KEY(selectedSubject.value)) || "{}",
  );
  answerStatus.value = d.status || Array(questions.value.length).fill(null);
  favorites.value = d.favs || {};
}

let tX = 0;
onMounted(() => {
  darkMode.value = localStorage.getItem("darkMode") === "true";
  isMobile.value = window.innerWidth < 768;
  fetchLibraryIndex();
  window.addEventListener("click", () => (showMenu.value = false));
});
function handleTouchStart(e) {
  tX = e.touches[0].clientX;
}
function handleTouchEnd(e) {
  const diff = e.changedTouches[0].clientX - tX;
  if (Math.abs(diff) > 80) diff > 0 ? prevQuestion() : nextQuestion();
}
watch([answerStatus, favorites], saveData, { deep: true });
</script>

<style>
/* 字体与基础 */
body {
  font-family:
    "Inter",
    system-ui,
    -apple-system,
    sans-serif;
  -webkit-tap-highlight-color: transparent;
}

/* 动画过渡 */
.slide-enter-active,
.slide-leave-active {
  transition: all 0.4s cubic-bezier(0.16, 1, 0.3, 1);
}

.slide-enter-from,
.slide-leave-to {
  transform: translateX(-100%);
  opacity: 0;
}

.menu-enter-from,
.menu-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

.menu-enter-active,
.menu-leave-active {
  transition: all 0.2s ease-out;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* 滚动条美化 */
.custom-scroll::-webkit-scrollbar {
  width: 4px;
}

.custom-scroll::-webkit-scrollbar-thumb {
  background: rgba(99, 102, 241, 0.3);
  border-radius: 10px;
}

.custom-scroll::-webkit-scrollbar-track {
  background: transparent;
}

/* 其他交互 */
button:disabled {
  cursor: not-allowed;
  filter: grayscale(1);
}

select {
  appearance: none;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 24 24' stroke='%236366f1'%3E%3Cpath stroke-linecap='round' stroke-linejoin='round' stroke-width='2' d='M19 9l-7 7-7-7'%3E%3C/path%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 1rem center;
  background-size: 1em;
}
</style>
