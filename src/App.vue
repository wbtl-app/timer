<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'

// Theme
function getSystemTheme(): 'light' | 'dark' {
  return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'
}

function getStoredTheme(): string | null {
  return localStorage.getItem('wbtl-theme')
}

function setTheme(theme: 'light' | 'dark' | 'system') {
  if (theme === 'system') {
    document.documentElement.removeAttribute('data-theme')
    localStorage.removeItem('wbtl-theme')
  } else {
    document.documentElement.setAttribute('data-theme', theme)
    localStorage.setItem('wbtl-theme', theme)
  }
}

function toggleTheme() {
  const current = document.documentElement.getAttribute('data-theme') as 'light' | 'dark' | null
  const system = getSystemTheme()

  if (!current) {
    // Currently using system, switch to opposite
    setTheme(system === 'dark' ? 'light' : 'dark')
  } else {
    // Currently using manual, check if it matches system
    if (current === system) {
      // Matches system, switch to opposite
      setTheme(current === 'dark' ? 'light' : 'dark')
    } else {
      // Doesn't match system, go back to system
      setTheme('system')
    }
  }
}

// Indicator type
const indicatorType = ref<'circle' | 'square'>('circle')

// Ring color
const ringColor = ref<'blue' | 'green' | 'red' | 'purple' | 'orange'>('blue')

// Flashing mode for alarm
const flashingMode = ref<'none' | 'fade' | 'slow' | 'fast'>('none')

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

// Local storage
const STORAGE_KEY = 'timer-settings'

interface StoredSettings {
  indicatorType: 'circle' | 'square'
  ringColor: 'blue' | 'green' | 'red' | 'purple' | 'orange'
  flashingMode: 'none' | 'fade' | 'slow' | 'fast'
  lastTimerHours: number
  lastTimerMinutes: number
  lastTimerSeconds: number
}

function saveSettings() {
  const settings: StoredSettings = {
    indicatorType: indicatorType.value,
    ringColor: ringColor.value,
    flashingMode: flashingMode.value,
    lastTimerHours: inputHours.value,
    lastTimerMinutes: inputMinutes.value,
    lastTimerSeconds: inputSeconds.value
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(settings))
}

function loadSettings() {
  const stored = localStorage.getItem(STORAGE_KEY)
  if (stored) {
    try {
      const settings: StoredSettings = JSON.parse(stored)
      indicatorType.value = settings.indicatorType ?? 'circle'
      ringColor.value = settings.ringColor ?? 'blue'
      flashingMode.value = settings.flashingMode ?? 'none'
      inputHours.value = settings.lastTimerHours ?? 0
      inputMinutes.value = settings.lastTimerMinutes ?? 0
      inputSeconds.value = settings.lastTimerSeconds ?? 0
    } catch {
      // Invalid stored data, use defaults
    }
  }
}

// Watch settings and save to localStorage
watch([indicatorType, ringColor, flashingMode, inputHours, inputMinutes, inputSeconds], () => {
  saveSettings()
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
    document.title = 'Timer - wbtl.app'
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
      startAlarm()
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

// Alarm state
const isAlarming = ref(false)
let alarmTimeoutId: number | null = null
let alarmIntervalId: number | null = null

function startAlarm() {
  if (flashingMode.value === 'none') return
  isAlarming.value = true

  // All alarm modes run for 10 seconds unless user interacts
  alarmTimeoutId = window.setTimeout(() => {
    stopAlarm()
  }, 10000)
}

function stopAlarm() {
  isAlarming.value = false
  if (alarmTimeoutId !== null) {
    clearTimeout(alarmTimeoutId)
    alarmTimeoutId = null
  }
  if (alarmIntervalId !== null) {
    clearInterval(alarmIntervalId)
    alarmIntervalId = null
  }
}

// Stop alarm on any click in the window
function handleWindowClick() {
  if (isAlarming.value) {
    stopAlarm()
  }
}

// Listen for system theme changes
let mediaQuery: MediaQueryList | null = null

onMounted(() => {
  // Initialize theme from storage
  const stored = getStoredTheme()
  if (stored) {
    setTheme(stored as 'light' | 'dark')
  }

  loadSettings()

  mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
  mediaQuery.addEventListener('change', () => {
    // Only react to system changes if no manual theme is set
    if (!getStoredTheme()) {
      document.documentElement.removeAttribute('data-theme')
    }
  })
})

// Cleanup on unmount
onUnmounted(() => {
  if (intervalId !== null) {
    clearInterval(intervalId)
  }
  stopAlarm()
  if (mediaQuery) {
    mediaQuery.removeEventListener('change', () => {})
  }
})
</script>

<template>
  <div class="app" :class="[ringColor, { alarming: isAlarming, ['alarm-' + flashingMode]: isAlarming }]" @click="handleWindowClick">
    <header>
      <div class="header-left">
        <div class="tool-icon">
          <svg viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="13" r="8"/>
            <path d="M12 9v4l2 2"/>
            <path d="M9 2h6"/>
            <path d="M12 2v2"/>
          </svg>
        </div>
        <div class="tool-name">Timer</div>
      </div>
      <div class="header-right">
        <a href="https://wbtl.app" class="back-link">wbtl.app</a>
        <div class="theme-toggle">
          <svg viewBox="0 0 24 24" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="5"/>
            <path d="M12 1v2M12 21v2M4.22 4.22l1.42 1.42M18.36 18.36l1.42 1.42M1 12h2M21 12h2M4.22 19.78l1.42-1.42M18.36 5.64l1.42-1.42"/>
          </svg>
          <div class="toggle-switch" @click="toggleTheme" @keydown.enter="toggleTheme" @keydown.space.prevent="toggleTheme" role="button" tabindex="0" aria-label="Toggle dark mode"></div>
          <svg viewBox="0 0 24 24" stroke-linecap="round" stroke-linejoin="round">
            <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
          </svg>
        </div>
      </div>
    </header>

    <div class="config-bar">
      <div class="setting-select">
        <label>Shape</label>
        <select v-model="indicatorType">
          <option value="circle">Circle</option>
          <option value="square">Square</option>
        </select>
      </div>
      <div class="setting-select">
        <label>Color</label>
        <select v-model="ringColor">
          <option value="blue">Blue</option>
          <option value="green">Green</option>
          <option value="red">Red</option>
          <option value="purple">Purple</option>
          <option value="orange">Orange</option>
        </select>
      </div>
      <div class="setting-select">
        <label>Alarm</label>
        <select v-model="flashingMode">
          <option value="none">No Flash</option>
          <option value="fade">Fade</option>
          <option value="slow">Slow Flash</option>
          <option value="fast">Fast Flash</option>
        </select>
      </div>
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
:root {
  --bg: #ffffff;
  --bg-secondary: #f5f5f7;
  --text: #1d1d1f;
  --text-secondary: #6e6e73;
  --accent: #0071e3;
  --accent-hover: #0077ed;
  --border: #d2d2d7;
  --toggle-bg: #e5e5e5;
  --toggle-knob: #ffffff;
  --tool-gradient-start: #ff6b6b;
  --tool-gradient-end: #ee5a5a;
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --bg: #000000;
    --bg-secondary: #1c1c1e;
    --text: #f5f5f7;
    --text-secondary: #98989d;
    --accent: #2997ff;
    --accent-hover: #40a9ff;
    --border: #38383a;
    --toggle-bg: #3a3a3c;
    --toggle-knob: #ffffff;
  }
}

[data-theme="dark"] {
  --bg: #000000;
  --bg-secondary: #1c1c1e;
  --text: #f5f5f7;
  --text-secondary: #98989d;
  --accent: #2997ff;
  --accent-hover: #40a9ff;
  --border: #38383a;
  --toggle-bg: #3a3a3c;
  --toggle-knob: #ffffff;
}

[data-theme="light"] {
  --bg: #ffffff;
  --bg-secondary: #f5f5f7;
  --text: #1d1d1f;
  --text-secondary: #6e6e73;
  --accent: #0071e3;
  --accent-hover: #0077ed;
  --border: #d2d2d7;
  --toggle-bg: #e5e5e5;
  --toggle-knob: #ffffff;
}

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
  background: var(--bg);
  color: var(--text);
  transition: background-color 0.3s, color 0.3s;
}

/* Header */
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.5rem;
  background: var(--bg-secondary);
  border-bottom: 1px solid var(--border);
}

.header-left {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.tool-icon {
  width: 40px;
  height: 40px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, var(--tool-gradient-start), var(--tool-gradient-end));
}

.tool-icon svg {
  width: 24px;
  height: 24px;
}

.tool-name {
  font-size: 1.25rem;
  font-weight: 600;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.back-link {
  color: var(--text-secondary);
  text-decoration: none;
  font-size: 0.9rem;
  transition: color 0.2s ease;
}

.back-link:hover {
  color: var(--accent);
}

/* Theme Toggle */
.theme-toggle {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.theme-toggle svg {
  width: 16px;
  height: 16px;
  stroke: var(--text-secondary);
  fill: none;
  stroke-width: 2;
}

.toggle-switch {
  position: relative;
  width: 44px;
  height: 24px;
  background: var(--toggle-bg);
  border-radius: 12px;
  cursor: pointer;
  transition: background 0.2s ease;
}

.toggle-switch::after {
  content: '';
  position: absolute;
  top: 2px;
  left: 2px;
  width: 20px;
  height: 20px;
  background: var(--toggle-knob);
  border-radius: 50%;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
  transition: transform 0.2s ease;
}

[data-theme="dark"] .toggle-switch::after,
:root:not([data-theme="light"]) .toggle-switch::after {
  transform: translateX(20px);
}

@media (prefers-color-scheme: light) {
  :root:not([data-theme="dark"]) .toggle-switch::after {
    transform: translateX(0);
  }
}

/* Config Bar */
.config-bar {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1.5rem;
  padding: 1rem 1.5rem;
  background: var(--bg-secondary);
  border-bottom: 1px solid var(--border);
}

.setting-select {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.setting-select label {
  font-size: 0.8rem;
  font-weight: 500;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.setting-select select {
  padding: 0.4rem 0.6rem;
  font-size: 0.85rem;
  font-family: 'Inter', sans-serif;
  border-radius: 0.5rem;
  cursor: pointer;
  transition: all 0.2s;
  background: var(--bg);
  border: 1px solid var(--border);
  color: var(--text);
}

.setting-select select:focus {
  outline: none;
  border-color: var(--accent);
}

/* Main container */
.container {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  width: 100%;
  max-width: 1800px;
  margin: 0 auto;
  padding: 1rem;
}

/* Indicator wrapper */
.indicator-wrapper {
  position: relative;
  width: min(80vw, 70vh, 1800px);
  height: min(80vw, 70vh, 1800px);
  min-width: 280px;
  min-height: 280px;
  margin: 0 auto;
  border-radius: 50%;
  background: var(--bg-secondary);
  box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.1);
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
  stroke: var(--border);
  transition: stroke 0.3s;
}

.progress-fill {
  transition: stroke-dashoffset 0.3s ease, stroke 0.3s;
}

/* Ring colors */
.app.blue .progress-fill { stroke: #3b82f6; filter: drop-shadow(0 0 8px rgba(59, 130, 246, 0.4)); }
.app.green .progress-fill { stroke: #22c55e; filter: drop-shadow(0 0 8px rgba(34, 197, 94, 0.4)); }
.app.red .progress-fill { stroke: #ef4444; filter: drop-shadow(0 0 8px rgba(239, 68, 68, 0.4)); }
.app.purple .progress-fill { stroke: #a855f7; filter: drop-shadow(0 0 8px rgba(168, 85, 247, 0.4)); }
.app.orange .progress-fill { stroke: #f97316; filter: drop-shadow(0 0 8px rgba(249, 115, 22, 0.4)); }

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
  background: var(--bg);
  border: 1px solid var(--border);
  color: var(--text);
}

.input-group input:focus {
  outline: none;
  border-color: var(--accent);
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
  background: var(--bg-secondary);
  border: 1px solid var(--border);
  color: var(--text);
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

.btn-primary {
  background: linear-gradient(180deg, var(--tool-gradient-start), var(--tool-gradient-end));
  color: white;
  border: 1px solid rgba(0, 0, 0, 0.1);
  box-shadow: 0 2px 8px rgba(255, 107, 107, 0.3);
}

.btn-danger {
  background: linear-gradient(180deg, #ef4444, #dc2626);
  color: white;
  border: 1px solid rgba(0, 0, 0, 0.1);
  box-shadow: 0 2px 8px rgba(220, 38, 38, 0.3);
}

/* Alarm animations */
@keyframes fade-alarm {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}

@keyframes slow-flash {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.2; }
}

@keyframes fast-flash {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.2; }
}

.app.alarming.alarm-fade .indicator-wrapper {
  animation: fade-alarm 2s ease-in-out infinite;
}

.app.alarming.alarm-slow .indicator-wrapper {
  animation: slow-flash 1s steps(2, end) infinite;
}

.app.alarming.alarm-fast .indicator-wrapper {
  animation: fast-flash 0.3s steps(2, end) infinite;
}

/* Responsive - small screens */
@media (max-width: 640px) {
  .tool-name {
    font-size: 1.1rem;
  }

  .header-right {
    gap: 1rem;
  }

  .back-link {
    font-size: 0.8rem;
  }

  .config-bar {
    flex-wrap: wrap;
    gap: 0.75rem;
    padding: 0.75rem 1rem;
  }

  .setting-select {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.25rem;
  }
}

@media (max-width: 400px) {
  .time-inputs {
    flex-direction: column;
    align-items: center;
  }

  header {
    padding: 0.75rem 1rem;
  }
}

/* Responsive - large screens */
@media (min-width: 800px) {
  .setting-select select {
    padding: 0.5rem 0.75rem;
    font-size: 0.9rem;
  }
}
</style>
