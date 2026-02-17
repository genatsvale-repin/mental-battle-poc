<template>
  <div class="game-wrapper" :class="currentStateClass">
    <div class="game-container">
      
      <!-- Start Screen -->
      <transition name="fade" mode="out-in">
        <div v-if="gameState === 'start'" class="screen start-screen" key="start">
          <div class="logo-area">
            <h1>MENTAL<br>BATTLE</h1>
            <div class="subtitle">INFINITE DRIVE</div>
          </div>
          
          <div class="hero-image">🚀</div>

          <button @click="startGame" class="btn-main pulse">
            СТАРТ
            <span class="btn-shine"></span>
          </button>
        </div>

        <!-- Playing Screen -->
        <div v-else-if="gameState === 'playing'" class="screen play-screen" key="playing">
          <div class="header-bar">
            <div class="stat-box score-box">
              <span class="label">СЧЁТ</span>
              <span class="value">{{ score }}</span>
            </div>
            <div class="stat-box timer-box" :class="{ 'timer-critical': timeLeft <= 5 }">
              <span class="label">ВРЕМЯ</span>
              <span class="value">{{ timeLeft.toFixed(1) }}</span>
            </div>
          </div>

          <div class="battle-arena">
             <div class="problem-card">
               <span class="operand">{{ problem.a }}</span>
               <span class="operator">{{ problem.op }}</span>
               <span class="operand">{{ problem.b }}</span>
               <span class="equals">=</span>
               <span class="answer-slot" :class="{ 'filled': userInput.length > 0 }">
                 {{ userInput || '?' }}
               </span>
             </div>
          </div>

          <div class="numpad-grid">
            <button v-for="n in [1, 2, 3, 4, 5, 6, 7, 8, 9]" :key="n" 
                    @click="appendDigit(n)" class="btn-num">
              {{ n }}
            </button>
            <button @click="backspace" class="btn-action btn-del">⌫</button>
            <button @click="appendDigit(0)" class="btn-num">0</button>
            <!-- Auto-submit is handled, but keep an enter button just in case or for visual symmetry -->
             <button @click="checkAnswerManually" class="btn-action btn-go">GO</button>
          </div>
        </div>

        <!-- Game Over Screen -->
        <div v-else-if="gameState === 'over'" class="screen result-screen" key="over">
          <div class="result-header">
            <h2>GAME OVER</h2>
            <p v-if="score > highScore" class="new-record">🏆 НОВЫЙ РЕКОРД! 🏆</p>
            <p v-else class="good-job">ОТЛИЧНАЯ ПОПЫТКА!</p>
          </div>

          <div class="score-card">
            <div class="final-score">{{ score }}</div>
            <div class="best-score">ЛУЧШИЙ: {{ highScore }}</div>
          </div>

          <button @click="startGame" class="btn-main">
            ЕЩЕ РАЗ
          </button>
        </div>
      </transition>

    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted, watch } from 'vue'

// --- Game Logic ---
const gameState = ref('start')
const score = ref(0)
const highScore = ref(parseInt(localStorage.getItem('mb-highscore') || '0'))
const userInput = ref('')
const timeLeft = ref(15.0)
const bonusTime = ref(2.0)
const problem = reactive({ a: 0, b: 0, op: '+', answer: 0 })

let timerInterval = null

// --- Difficulty Scaling ---
// As score increases, problems get harder
const getRange = (currentScore) => {
  if (currentScore < 10) return 10 // 0-9: numbers up to 10
  if (currentScore < 25) return 20 // 10-24: numbers up to 20
  if (currentScore < 50) return 50 // 25-49: numbers up to 50
  return 100 // 50+: numbers up to 100
}

const generateProblem = () => {
  const range = getRange(score.value)
  let a = Math.floor(Math.random() * range) + 1
  let b = Math.floor(Math.random() * range) + 1
  
  // Mix of + and -
  // After 20 points, maybe introduce simple multiplication? Let's stick to +/- for MVP consistency with "Mental Battle" basics
  const ops = score.value > 30 ? ['+', '-', '+', '-', '*'] : ['+', '-'] // 20% chance of * after 30
  const op = ops[Math.floor(Math.random() * ops.length)]

  if (op === '*') {
     // Simple multiplication table (2-9)
     a = Math.floor(Math.random() * 8) + 2
     b = Math.floor(Math.random() * 9) + 1
  } else if (op === '-') {
    // Ensure positive result for simplicity (kids mode)
    if (a < b) [a, b] = [b, a]
  }

  problem.a = a
  problem.b = b
  problem.op = op
  
  if (op === '+') problem.answer = a + b
  if (op === '-') problem.answer = a - b
  if (op === '*') problem.answer = a * b
}

const startGame = () => {
  score.value = 0
  bonusTime.value = 2.0
  inputShake.value = false
  timeLeft.value = 15.0 // Start with 15 seconds
  userInput.value = ''
  gameState.value = 'playing'
  generateProblem()
  startTimer()
}

const startTimer = () => {
  if (timerInterval) clearInterval(timerInterval)
  const tickRate = 100
  
  timerInterval = setInterval(() => {
    timeLeft.value -= (tickRate / 1000)
    if (timeLeft.value <= 0) {
      timeLeft.value = 0
      gameOver()
    }
  }, tickRate)
}

const gameOver = () => {
  clearInterval(timerInterval)
  gameState.value = 'over'
  if (score.value > highScore.value) {
    highScore.value = score.value
    localStorage.setItem('mb-highscore', score.value)
  }
}

// --- Input Handling ---
const inputShake = ref(false)

const appendDigit = (n) => {
  if (userInput.value.length < 4) {
    userInput.value += n
    checkAnswerAuto()
  }
}

const backspace = () => {
  userInput.value = userInput.value.slice(0, -1)
}

const checkAnswerAuto = () => {
  const ans = parseInt(userInput.value)
  if (isNaN(ans)) return

  const strAns = problem.answer.toString()
  const strUser = userInput.value
  
  if (strAns === strUser) {
    processCorrectAnswer()
  } else if (!strAns.startsWith(strUser)) {
    // If what user typed is not a prefix of the correct answer, it's immediately wrong
    processWrongAnswer()
  }
}

const checkAnswerManually = () => {
  const ans = parseInt(userInput.value)
  if (ans === problem.answer) {
    processCorrectAnswer()
  } else {
    processWrongAnswer()
  }
}

const processCorrectAnswer = () => {
  score.value++
  // Add current bonus time
  timeLeft.value += bonusTime.value 
  
  // Decrease bonus time by 0.1s for next answer, minimum 0.5s
  bonusTime.value = Math.max(0.5, bonusTime.value - 0.1)
  
  userInput.value = ''
  generateProblem()
}

const processWrongAnswer = () => {
  // -0.5 seconds penalty
  timeLeft.value = Math.max(0, timeLeft.value - 0.5)
  
  // Shake effect
  inputShake.value = true
  setTimeout(() => inputShake.value = false, 400)
  
  userInput.value = ''
}

const currentStateClass = computed(() => {
  return `state-${gameState.value}`
})

</script>

<style>
/* --- Design System --- */
@import url('https://fonts.googleapis.com/css2?family=Fredoka:wght@300;400;600;700&display=swap');

:root {
  --bg-dark: #FFF9F2; /* Warm Cream */
  --bg-light: #FFEBCC; /* Soft Peach */
  --primary: #FF7043; /* Friendly Orange */
  --secondary: #4DB6AC; /* Gentle Teal */
  --accent: #FBC02D; /* Sunny Yellow */
  --text-main: #4E342E; /* Dark Chocolate/Brown for readability */
  --glass: rgba(255, 255, 255, 0.6);
  --glass-border: rgba(255, 255, 255, 0.8);
  --font-main: 'Fredoka', sans-serif;
}

* { box-sizing: border-box; touch-action: manipulation; }

body {
  margin: 0;
  background: linear-gradient(135deg, var(--bg-light), var(--bg-dark));
  color: var(--text-main);
  font-family: var(--font-main);
  overflow: hidden;
  height: 100vh;
}

.game-wrapper {
  height: 100vh;
  width: 100vw;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
}

.game-container {
  width: 100%;
  max-width: 480px;
  height: 100%;
  position: relative;
  display: flex;
  flex-direction: column;
}

.screen {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  position: absolute;
  top: 0; 
  left: 0;
}

/* --- Start Screen --- */
.start-screen {
  justify-content: center;
  gap: 30px;
}

.logo-area {
  text-align: center;
}

h1 {
  font-size: 4rem;
  line-height: 0.9;
  margin: 0;
  color: var(--primary);
  text-shadow: 3px 3px 0px var(--accent);
  font-weight: 700;
  letter-spacing: 1px;
}

.subtitle {
  font-size: 1.5rem;
  color: var(--secondary);
  letter-spacing: 2px;
  margin-top: 10px;
  font-weight: 600;
  text-shadow: none;
}

.hero-image {
  font-size: 6rem;
  animation: float 3s ease-in-out infinite;
}

@keyframes float {
  0%, 100% { transform: translateY(0) rotate(0); }
  50% { transform: translateY(-15px) rotate(5deg); }
}

/* --- Buttons --- */
button {
  border: none;
  font-family: var(--font-main);
  cursor: pointer;
  outline: none;
  border-radius: 20px;
  transition: transform 0.1s;
}

button:active {
  transform: scale(0.92);
}

.btn-main {
  background: var(--primary);
  color: white;
  font-size: 2rem;
  font-weight: 700;
  padding: 18px 50px;
  border-bottom: 6px solid #D84315;
  box-shadow: 0 8px 15px rgba(255, 112, 67, 0.3);
}

/* --- Playing Screen --- */
.play-screen {
  justify-content: space-between;
  padding-bottom: 40px;
}

.header-bar {
  display: flex;
  width: 100%;
  justify-content: space-between;
  padding: 10px 0;
}

.stat-box {
  background: white;
  border: 2px solid var(--bg-light);
  border-radius: 18px;
  padding: 8px 16px;
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 110px;
  box-shadow: 0 4px 10px rgba(0,0,0,0.05);
}

.stat-box .label {
  font-size: 0.75rem;
  color: #8D6E63;
  font-weight: 600;
}

.stat-box .value {
  font-size: 1.8rem;
  font-weight: 700;
}

.score-box .value { color: var(--accent); }
.timer-box .value { color: var(--secondary); }
.timer-critical .value { color: var(--primary); }

.battle-arena {
  flex-grow: 1;
  display: flex;
  justify-content: center;
  align-items: center;
}

.problem-card {
  font-size: 4.5rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  gap: 15px;
  color: var(--text-main);
}

.operator { color: var(--primary); }
.equals { color: #8D6E63; }

.answer-slot {
  min-width: 120px;
  height: 90px;
  border-bottom: 6px solid #FFCCBC;
  display: flex;
  justify-content: center;
  align-items: center;
  color: var(--secondary);
}

.answer-slot.filled {
  border-bottom: 6px solid var(--secondary);
}

/* Numpad */
.numpad-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
  width: 100%;
  max-width: 360px;
}

.btn-num {
  background: white;
  color: var(--text-main);
  font-size: 2.2rem;
  font-weight: 600;
  padding: 18px 0;
  border-radius: 18px;
  box-shadow: 0 5px 0 #EAD9C6;
}

.btn-action {
  font-size: 1.6rem;
  font-weight: 700;
  border-radius: 18px;
}

.btn-del {
  background: #BCAAA4;
  color: white;
  box-shadow: 0 5px 0 #8D6E63;
}

.btn-go {
  background: var(--secondary);
  color: white;
  box-shadow: 0 5px 0 #00796B;
}

/* --- Result Screen --- */
.result-screen {
  justify-content: center;
  gap: 25px;
}

.result-header h2 {
  font-size: 3rem;
  color: var(--primary);
  margin: 0;
}

.score-card {
  background: white;
  padding: 30px;
  border-radius: 25px;
  border: 4px solid var(--bg-light);
  box-shadow: 0 10px 20px rgba(0,0,0,0.05);
}

.final-score {
  font-size: 5rem;
  font-weight: 700;
  color: var(--accent);
}

.best-score {
  margin-top: 10px;
  font-size: 1.2rem;
  opacity: 0.8;
}

/* --- Transitions --- */
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}

/* Mobile Responsiveness */
@media (max-height: 700px) {
  h1 { font-size: 3rem; }
  .problem-card { font-size: 3.5rem; }
  .btn-main { padding: 15px 40px; font-size: 1.5rem; }
  .btn-num { padding: 15px 0; font-size: 1.8rem; }
}
</style>
