<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'

// Theme
const theme = ref<'light' | 'dark'>('light')

function initTheme() {
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
  theme.value = prefersDark ? 'dark' : 'light'
}

function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}

// Listen for system theme changes
let mediaQuery: MediaQueryList | null = null

onMounted(() => {
  initTheme()
  mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
  mediaQuery.addEventListener('change', (e) => {
    theme.value = e.matches ? 'dark' : 'light'
  })
})

onUnmounted(() => {
  if (mediaQuery) {
    mediaQuery.removeEventListener('change', () => {})
  }
})

// Indicator type
const indicatorType = ref<'circle' | 'square'>('circle')

// Timer state
const inputHours = ref(0)
const inputMinutes = ref(0)
const inputSeconds = ref(0)

const originalTime = ref(0) // in seconds
const remainingTime = ref(0) // in seconds
const isRunning = ref(false)
const hasStarted = ref(false)

let intervalId: number | null = null

// Computed display values
const displayHours = computed(() => Math.floor(remainingTime.value / 3600))
const displayMinutes = computed(() => Math.floor((remainingTime.value % 3600) / 60))
const displaySeconds = computed(() => remainingTime.value % 60)

const formattedTime = computed(() => {
  const h = String(displayHours.value).padStart(2, '0')
  const m = String(displayMinutes.value).padStart(2, '0')
  const s = String(displaySeconds.value).padStart(2, '0')
  return `${h}:${m}:${s}`
})

const canStart = computed(() => {
  if (hasStarted.value) return remainingTime.value > 0
  return inputHours.value > 0 || inputMinutes.value > 0 || inputSeconds.value > 0
})

// Progress calculations
const ringRadius = 140
const ringCircumference = 2 * Math.PI * ringRadius

const progressPercent = computed(() => {
  if (!hasStarted.value || originalTime.value === 0) {
    return 1 // Full when not started
  }
  return remainingTime.value / originalTime.value
})

const progressOffset = computed(() => {
  return ringCircumference * (1 - progressPercent.value)
})

// Square indicator path (perimeter-based, starting from top-left, going clockwise)
const squareSize = 280
const squareStrokeWidth = 12
const squareInset = squareStrokeWidth / 2
const squarePerimeter = (squareSize - squareStrokeWidth) * 4

const squarePath = computed(() => {
  const s = squareSize - squareStrokeWidth
  const offset = squareInset
  // Start from top-left, go clockwise: right, down, left, up
  return `M ${offset} ${offset} L ${offset + s} ${offset} L ${offset + s} ${offset + s} L ${offset} ${offset + s} Z`
})

const squareProgressOffset = computed(() => {
  return squarePerimeter * (1 - progressPercent.value)
})

// Update page title when running
watch([isRunning, formattedTime], ([running, time]) => {
  if (running) {
    document.title = `${time} - Timer`
  } else {
    document.title = 'Timer'
  }
})

// Timer functions
function startTimer() {
  if (!hasStarted.value) {
    // First start - calculate total seconds from input
    const totalSeconds = inputHours.value * 3600 + inputMinutes.value * 60 + inputSeconds.value
    if (totalSeconds <= 0) return
    originalTime.value = totalSeconds
    remainingTime.value = totalSeconds
    hasStarted.value = true
  }

  if (remainingTime.value <= 0) return

  isRunning.value = true
  intervalId = window.setInterval(() => {
    if (remainingTime.value > 0) {
      remainingTime.value--
    }
    if (remainingTime.value <= 0) {
      pauseTimer()
    }
  }, 1000)
}

function pauseTimer() {
  isRunning.value = false
  if (intervalId !== null) {
    clearInterval(intervalId)
    intervalId = null
  }
}

function resetTimer() {
  pauseTimer()
  remainingTime.value = originalTime.value
}

function deleteTimer() {
  pauseTimer()
  hasStarted.value = false
  remainingTime.value = 0
  originalTime.value = 0
  inputHours.value = 0
  inputMinutes.value = 0
  inputSeconds.value = 0
}

// Cleanup on unmount
onUnmounted(() => {
  if (intervalId !== null) {
    clearInterval(intervalId)
  }
})
</script>

<template>
  <div class="app" :class="theme">
    <div class="top-controls">
      <div class="indicator-select">
        <select v-model="indicatorType">
          <option value="circle">Circle</option>
          <option value="square">Square</option>
        </select>
      </div>
      <button class="theme-toggle" @click="toggleTheme" :title="`Switch to ${theme === 'light' ? 'dark' : 'light'} mode`">
        <span v-if="theme === 'light'">&#9790;</span>
        <span v-else>&#9788;</span>
      </button>
    </div>

    <main class="container">
      <div class="indicator-wrapper">
        <!-- Circle indicator -->
        <svg v-if="indicatorType === 'circle'" class="progress-indicator progress-circle" viewBox="0 0 320 320">
          <circle
            class="progress-bg"
            cx="160"
            cy="160"
            :r="ringRadius"
            fill="none"
            stroke-width="12"
          />
          <circle
            class="progress-fill"
            cx="160"
            cy="160"
            :r="ringRadius"
            fill="none"
            stroke-width="12"
            :stroke-dasharray="ringCircumference"
            :stroke-dashoffset="progressOffset"
            stroke-linecap="round"
          />
        </svg>

        <!-- Square indicator -->
        <svg v-else class="progress-indicator progress-square" :viewBox="`0 0 ${squareSize} ${squareSize}`">
          <path
            class="progress-bg"
            :d="squarePath"
            fill="none"
            :stroke-width="squareStrokeWidth"
          />
          <path
            class="progress-fill"
            :d="squarePath"
            fill="none"
            :stroke-width="squareStrokeWidth"
            :stroke-dasharray="squarePerimeter"
            :stroke-dashoffset="squareProgressOffset"
            stroke-linecap="square"
          />
        </svg>

        <div class="indicator-content">
          <h1>Timer</h1>

          <!-- Input form (shown when timer hasn't started) -->
          <div v-if="!hasStarted" class="input-section">
            <div class="time-inputs">
              <div class="input-group">
                <label for="hours">Hours</label>
                <input
                  id="hours"
                  type="number"
                  v-model.number="inputHours"
                  min="0"
                  max="99"
                />
              </div>
              <div class="input-group">
                <label for="minutes">Minutes</label>
                <input
                  id="minutes"
                  type="number"
                  v-model.number="inputMinutes"
                  min="0"
                  max="59"
                />
              </div>
              <div class="input-group">
                <label for="seconds">Seconds</label>
                <input
                  id="seconds"
                  type="number"
                  v-model.number="inputSeconds"
                  min="0"
                  max="59"
                />
              </div>
            </div>
            <button class="btn btn-primary" @click="startTimer" :disabled="!canStart">
              Start
            </button>
          </div>

          <!-- Timer display (shown when timer has started) -->
          <div v-else class="timer-section">
            <div class="timer-display" :class="{ finished: remainingTime === 0 }">
              {{ formattedTime }}
            </div>
            <div class="controls">
              <button
                v-if="!isRunning && remainingTime > 0"
                class="btn btn-primary"
                @click="startTimer"
              >
                {{ remainingTime === originalTime ? 'Start' : 'Resume' }}
              </button>
              <button
                v-if="isRunning"
                class="btn btn-secondary"
                @click="pauseTimer"
              >
                Pause
              </button>
              <button
                class="btn btn-secondary"
                @click="resetTimer"
                :disabled="remainingTime === originalTime"
              >
                Reset
              </button>
              <button class="btn btn-danger" @click="deleteTimer">
                Delete
              </button>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  height: 100%;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

#app {
  height: 100%;
}

.app {
  min-height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 0.75rem;
  transition: background-color 0.3s, color 0.3s;
  position: relative;
}

/* Light theme */
.app.light {
  background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%);
  color: #1e293b;
}

.app.light .btn {
  background: linear-gradient(180deg, #f1f5f9 0%, #e2e8f0 100%);
  color: #1e293b;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05), 0 1px 2px rgba(0, 0, 0, 0.1);
  border: 1px solid rgba(0, 0, 0, 0.05);
}

.app.light .btn-primary {
  background: linear-gradient(180deg, #3b82f6 0%, #2563eb 100%);
  color: white;
  border: 1px solid rgba(0, 0, 0, 0.1);
  box-shadow: 0 2px 8px rgba(37, 99, 235, 0.3), 0 1px 2px rgba(0, 0, 0, 0.1);
}

.app.light .btn-danger {
  background: linear-gradient(180deg, #ef4444 0%, #dc2626 100%);
  color: white;
  border: 1px solid rgba(0, 0, 0, 0.1);
  box-shadow: 0 2px 8px rgba(220, 38, 38, 0.3), 0 1px 2px rgba(0, 0, 0, 0.1);
}

.app.light input,
.app.light select {
  background: white;
  border: 1px solid #cbd5e1;
  color: #1e293b;
  box-shadow: inset 0 1px 2px rgba(0, 0, 0, 0.05);
}

.app.light .indicator-wrapper {
  background: rgba(255, 255, 255, 0.6);
  box-shadow:
    0 25px 50px -12px rgba(0, 0, 0, 0.1),
    0 0 0 1px rgba(255, 255, 255, 0.8) inset;
}

.app.light .progress-bg {
  stroke: #e2e8f0;
}

.app.light .progress-fill {
  stroke: #3b82f6;
  filter: drop-shadow(0 0 8px rgba(59, 130, 246, 0.4));
}

/* Dark theme */
.app.dark {
  background: linear-gradient(135deg, #0f172a 0%, #1e293b 100%);
  color: #f1f5f9;
}

.app.dark .btn {
  background: linear-gradient(180deg, #334155 0%, #1e293b 100%);
  color: #f1f5f9;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2), 0 1px 2px rgba(0, 0, 0, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.app.dark .btn-primary {
  background: linear-gradient(180deg, #60a5fa 0%, #3b82f6 100%);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 2px 12px rgba(59, 130, 246, 0.4), 0 1px 2px rgba(0, 0, 0, 0.2);
}

.app.dark .btn-danger {
  background: linear-gradient(180deg, #f87171 0%, #ef4444 100%);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 2px 12px rgba(239, 68, 68, 0.4), 0 1px 2px rgba(0, 0, 0, 0.2);
}

.app.dark input,
.app.dark select {
  background: #1e293b;
  border: 1px solid #475569;
  color: #f1f5f9;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.3);
}

.app.dark .indicator-wrapper {
  background: rgba(30, 41, 59, 0.6);
  box-shadow:
    0 25px 50px -12px rgba(0, 0, 0, 0.5),
    0 0 0 1px rgba(255, 255, 255, 0.05) inset;
}

.app.dark .progress-bg {
  stroke: #334155;
}

.app.dark .progress-fill {
  stroke: #60a5fa;
  filter: drop-shadow(0 0 12px rgba(96, 165, 250, 0.5));
}

/* Top controls */
.top-controls {
  position: absolute;
  top: 1rem;
  right: 1rem;
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.indicator-select select {
  padding: 0.4rem 0.6rem;
  font-size: 0.8rem;
  font-family: 'Inter', sans-serif;
  border-radius: 0.5rem;
  cursor: pointer;
  transition: all 0.2s;
}

.indicator-select select:focus {
  outline: none;
  border-color: #3b82f6;
}

.theme-toggle {
  background: transparent;
  border: none;
  font-size: 1.25rem;
  cursor: pointer;
  padding: 0.4rem;
  opacity: 0.7;
  transition: opacity 0.2s, transform 0.2s;
}

.theme-toggle:hover {
  opacity: 1;
  transform: scale(1.1);
}

.app.light .theme-toggle {
  color: #1e293b;
}

.app.dark .theme-toggle {
  color: #f1f5f9;
}

/* Container */
.container {
  text-align: center;
  width: 100%;
  max-width: 1800px;
  padding: 0 0.5rem;
}

/* Indicator wrapper */
.indicator-wrapper {
  position: relative;
  width: min(85vw, 85vh, 1800px);
  height: min(85vw, 85vh, 1800px);
  min-width: 280px;
  min-height: 280px;
  margin: 0 auto;
  border-radius: 50%;
  backdrop-filter: blur(10px);
  transition: border-radius 0.3s, box-shadow 0.3s;
}

.indicator-wrapper:has(.progress-square) {
  border-radius: 1.5rem;
}

.progress-indicator {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.progress-circle {
  transform: rotate(-90deg);
}

.progress-bg {
  transition: stroke 0.3s;
}

.progress-fill {
  transition: stroke-dashoffset 0.3s ease, stroke 0.3s;
}

.indicator-content {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 75%;
  max-width: 1200px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

h1 {
  font-size: clamp(1.25rem, 4vw, 4rem);
  margin-bottom: clamp(0.75rem, 2vw, 3rem);
  font-weight: 500;
  letter-spacing: -0.02em;
  opacity: 0.9;
}

/* Input section */
.input-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: clamp(0.75rem, 3vw, 3rem);
}

.time-inputs {
  display: flex;
  gap: clamp(0.5rem, 3vw, 3rem);
  justify-content: center;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: clamp(0.25rem, 0.5vw, 0.75rem);
}

.input-group label {
  font-size: clamp(0.6rem, 1.5vw, 1.5rem);
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  opacity: 0.6;
}

.input-group input {
  width: clamp(55px, 15vw, 200px);
  padding: clamp(0.4rem, 2vw, 1.5rem);
  font-size: clamp(1rem, 4vw, 3rem);
  font-family: 'JetBrains Mono', monospace;
  text-align: center;
  border-radius: clamp(0.5rem, 1vw, 1rem);
  transition: all 0.2s;
}

.input-group input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.2);
}

/* Hide number input spinners */
.input-group input::-webkit-outer-spin-button,
.input-group input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.input-group input[type=number] {
  -moz-appearance: textfield;
}

/* Timer display */
.timer-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: clamp(0.75rem, 3vw, 3rem);
}

.timer-display {
  font-size: clamp(2.5rem, 12vw, 12rem);
  font-family: 'JetBrains Mono', monospace;
  font-weight: 500;
  letter-spacing: -0.02em;
}

.timer-display.finished {
  opacity: 0.4;
}

/* Controls */
.controls {
  display: flex;
  gap: clamp(0.4rem, 1.5vw, 1.5rem);
  justify-content: center;
  flex-wrap: wrap;
}

/* Buttons */
.btn {
  padding: clamp(0.4rem, 1.5vw, 1.25rem) clamp(0.6rem, 2vw, 2.5rem);
  font-size: clamp(0.75rem, 2vw, 2rem);
  font-family: 'Inter', sans-serif;
  font-weight: 500;
  border-radius: clamp(0.5rem, 1vw, 1rem);
  cursor: pointer;
  transition: all 0.15s;
}

.btn:hover:not(:disabled) {
  transform: translateY(-1px);
}

.btn:active:not(:disabled) {
  transform: translateY(0) scale(0.98);
}

.btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

/* Responsive - small screens */
@media (max-width: 400px) {
  .time-inputs {
    flex-direction: column;
    align-items: center;
  }

  .top-controls {
    top: 0.75rem;
    right: 0.75rem;
    gap: 0.5rem;
  }
}

/* Responsive - large screens */
@media (min-width: 800px) {
  .indicator-select select {
    padding: 0.5rem 0.75rem;
    font-size: 0.9rem;
  }

  .theme-toggle {
    font-size: 1.5rem;
  }
}
</style>
