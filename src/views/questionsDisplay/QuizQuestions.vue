<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { storeToRefs } from 'pinia'
import router from '@/router'
import singleSessionStore from '@/stores/singleSessionStore'
import parameterStore from '@/stores/parameterStore'
import allSessionsStore from '@/stores/allSessionsStore'
import QuestionText from '@/components/QuestionText.vue'
import {
  fetchQuizQuestions,
  transformQuestionData,
  loadErrorPage
} from './fetchQuestions/fetchQuestions'

export type Question = {
  id: number
  question: string
  answers: Record<string, string>
  correctAnswer: string
  selectedAnswer?: string
  isCorrect?: boolean
}

export type Session = {
  date: string
  sessionScore: number
  sessionCategory: string
  sessionQuestions: Question[]
}
export type QuestionData = {
  id: number
  question: string
  answers: Record<string, string>
  correct_answers: Record<string, string>
}

type SelectedAnswers = Record<number, string | undefined>

// to load everything on the page once questions are fetched from API
const dataLoaded = ref(false)
const questions = ref<Question[]>([])
const { category, limit } = storeToRefs(parameterStore())
const selectedAnswers = ref<SelectedAnswers>({})

// check if selected answer exists and is equal to correct answer
function correctCounter(count: number, question: Question) {
  if (question.selectedAnswer && question.selectedAnswer === question.correctAnswer) {
    question.isCorrect = true
    count += 1
  } else {
    question.isCorrect = false
  }
  return count
}

// move everything that's onSubmit in one function and then refactor
// computed is used for displaying
const percentageCount = computed(() => {
  let count = 0
  questions.value.forEach(question => {
    question.selectedAnswer = selectedAnswers.value[question.id]
    count = correctCounter(count, question)
  })
  const percentage = (count / questions.value.length) * 100
  return Math.round(percentage)
})

const { sessionScore, sessionCategory, sessionQuestions } = storeToRefs(singleSessionStore())

// replace previous session data with this one's
function updateSession() {
  sessionScore.value = percentageCount.value
  sessionCategory.value = category.value
  sessionQuestions.value = questions.value
}

// updated session object to allSessions array and navigate to result page
function onSubmit() {
  updateSession()
  const sessionObj = singleSessionStore().createSessionObject()
  allSessionsStore().addLastSession(sessionObj)
  router.push({ name: 'result' })
}

// fetch quiz questions data from the api and assign transformed data to questions value
async function initializeQuiz() {
  try {
    const questionsData = await fetchQuizQuestions(category.value, limit.value)
    questions.value = questionsData.map(transformQuestionData)
    dataLoaded.value = true
  } catch (error) {
    loadErrorPage(router, error)
  }
}

function getAnswerTestId(questionKey: number, answerIndex: string) {
  return `question-${questionKey}-${answerIndex}`
}
// Call initializeQuiz when the component is mounted
onMounted(initializeQuiz)
</script>

<template>
  <div
    class="mb-5 h-px border-t-0 bg-transparent bg-gradient-to-r from-transparent via-neutral-500 to-transparent opacity-25 dark:opacity-100"
  ></div>
  <div v-show="dataLoaded" class="m-5">
    <h1 class="text-xl pb-5 text-center">Category: {{ category }}</h1>
    <div v-for="(question, key) in questions" :key="key">
      <QuestionText :question="question" :obj-key="key" />
      <div class="mb-4">
        <div
          v-for="(answer, index) in question.answers"
          :key="index"
          class="flex items-center pl-4 py-1.5 border rounded-2xl border-gray-700 mb-1.5"
        >
          <label class="text-base" for="index" :data-testid="getAnswerTestId(key, index)">
            <input
              v-if="answer"
              type="radio"
              :id="index"
              :value="index"
              v-model="selectedAnswers[question.id]"
              class="radio radio-secondary radio-sm py-1"
            />
            {{ answer }}
          </label>
        </div>
      </div>
    </div>
    <div class="flex justify-center mb-5">
      <button
        :disabled="Object.keys(selectedAnswers).length !== questions.length"
        type="button"
        class="btn btn-secondary"
        @click="onSubmit"
      >
        Finish quiz
      </button>
    </div>
  </div>
</template>
