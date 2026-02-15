<template>
  <div class="game-container" :class="{ 'game-over': gameState === 'over' }">
    <!-- Start Screen -->
    <div v-if="gameState === 'start'" class="screen">
      <h1>Mental Battle</h1>
      <p>Бесконечный драйв</p>
      <button @click="startGame" class="btn-main">СТАРТ</button>
    </div>

    <!-- Playing Screen -->
    <div v-else-if="gameState === 'playing'" class="screen">
      <div class="stats">
        <div class="score">Очки: {{ score }}</div>
        <div class="timer" :class="{ 'timer-low': timeLeft < 2 }">
          {{ timeLeft.toFixed(1) }}s
        </div>
      </div>

      <div class="problem">
        {{ problem.a }} {{ problem.op }} {{ problem.b }} = ?
      </div>

      <div class="input-display">{{ userInput || '?' }}</div>

      <div class="numpad">
        <button v-for="n in 9" :key="n" @click="appendDigit(n)">{{ n }}</button>
        <button @click="clearInput">C</button>
        <button @click="appendDigit(0)">0</button>
        <button @click="submitAnswer" class="btn-submit">➔</button>
      </div>
    </div>

    <!-- Game Over Screen -->
    <div v-else-if="gameState === 'over'" class="screen">
      <h2>КОНЕЦ ИГРЫ</h2>
      <div class="final-score">Ваш счет: {{ score }}</div>
      <button @click="startGame" class="btn-main">ЕЩЕ РАЗ</button>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onUnmounted } from 'vue'

const gameState = ref('start') // start, playing, over
const score = ref(0)
const userInput = ref('')
const timeLeft = ref(5)
const problem = reactive({ a: 0, b: 0, op: '+', answer: 0 })

let timerInterval = null

const generateProblem = () => {
  const a = Math.floor(Math.random() * 50) + 1
  const b = Math.floor(Math.random() * 50) + 1
  const op = Math.random() > 0.5 ? '+' : '-'
  
  problem.a = a
  problem.b = b
  problem.op = op
  problem.answer = op === '+' ? a + b : a - b
  
  // Reset timer based on score (gets faster)
  timeLeft.value = Math.max(2, 5 - Math.floor(score.value / 10))
}

const startGame = () => {
  score.value = 0
  userInput.value = ''
  gameState.value = 'playing'
  generateProblem()
  startTimer()
}

const startTimer = () => {
  clearInterval(timerInterval)
  timerInterval = setInterval(() => {
    timeLeft.value -= 0.1
    if (timeLeft.value <= 0) {
      endGame()
    }
  }, 100)
}

const endGame = () => {
  clearInterval(timerInterval)
  gameState.value = 'over'
}

const appendDigit = (n) => {
  if (userInput.value.length < 4) {
    userInput.value += n
  }
}

const clearInput = () => {
  userInput.value = ''
}

const submitAnswer = () => {
  if (parseInt(userInput.value) === problem.answer) {
    score.value++
    userInput.value = ''
    generateProblem()
  } else {
    endGame()
  }
}

onUnmounted(() => clearInterval(timerInterval))
</script>

<style>
:root {
  --bg: #121212;
  --text: #e0e0e0;
  --accent: #4caf50;
  --danger: #f44336;
}

body {
  margin: 0;
  padding: 0;
  background-color: var(--bg);
  color: var(--text);
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  overflow: hidden;
  user-select: none;
}

.game-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  transition: background-color 0.3s;
}

.game-over {
  background-color: #2c0000;
}

.screen {
  width: 100%;
  max-width: 400px;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 20px;
}

h1 { font-size: 3rem; margin-bottom: 0.5rem; letter-spacing: -1px; }

.stats {
  display: flex;
  justify-content: space-between;
  width: 100%;
  font-size: 1.2rem;
  font-weight: bold;
  margin-bottom: 2rem;
}

.timer-low { color: var(--danger); }

.problem {
  font-size: 3.5rem;
  font-weight: 800;
  margin-bottom: 1rem;
}

.input-display {
  font-size: 2.5rem;
  height: 3rem;
  border-bottom: 2px solid var(--accent);
  width: 150px;
  margin-bottom: 2rem;
}

.numpad {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  width: 100%;
}

button {
  background: #1e1e1e;
  border: 1px solid #333;
  color: white;
  padding: 20px;
  font-size: 1.5rem;
  border-radius: 12px;
  cursor: pointer;
  touch-action: manipulation;
}

button:active { background: #333; }

.btn-main {
  background: var(--accent);
  color: black;
  font-weight: bold;
  width: 200px;
  padding: 15px;
  margin-top: 2rem;
}

.btn-submit { background: var(--accent); color: black; }
.final-score { font-size: 2rem; margin: 1rem 0; }
</style>
