<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

/**
 * Game state and logic:
 * - Random target in [min, max] created on mount and on restart
 * - Validates integer input within range
 * - Tracks guesses with attempt number, value, and hint
 * - Feedback shown after each valid guess
 */

const min = 1
const max = 100

// Reactive state
const target = ref<number>(0)
const currentGuess = ref<string>('') // using string to simplify input handling
const attempts = ref<number>(0)
const guesses = ref<Array<{ attempt: number; value: number; hint: 'low' | 'high' | 'correct' }>>([])
const status = ref<'idle' | 'low' | 'high' | 'correct'>('idle')
const error = ref<string>('')

// PUBLIC_INTERFACE
function newTarget(): number {
  /** Generate a random integer between min and max inclusive. */
  const range = max - min + 1
  return Math.floor(Math.random() * range) + min
}

// PUBLIC_INTERFACE
function resetGame(): void {
  /** Reset all game state and pick a new target. */
  target.value = newTarget()
  currentGuess.value = ''
  attempts.value = 0
  guesses.value = []
  status.value = 'idle'
  error.value = ''
}

// PUBLIC_INTERFACE
function validateGuess(raw: string): { ok: boolean; value?: number; message?: string } {
  /**
   * Validate that the guess is an integer within [min, max].
   * Returns parsing result and validation message if invalid.
   */
  if (raw.trim() === '') {
    return { ok: false, message: 'Please enter a number.' }
  }
  // Ensure numeric and integer
  const num = Number(raw)
  if (!Number.isFinite(num) || !Number.isInteger(num)) {
    return { ok: false, message: 'Enter a whole number (no decimals).' }
  }
  if (num < min || num > max) {
    return { ok: false, message: `Enter a number between ${min} and ${max}.` }
  }
  return { ok: true, value: num }
}

// PUBLIC_INTERFACE
function submitGuess(): void {
  /** Handle Guess button click: validate, update state, and set feedback. */
  error.value = ''
  const res = validateGuess(currentGuess.value)
  if (!res.ok) {
    error.value = res.message || 'Invalid input.'
    return
  }
  const guess = res.value as number
  attempts.value += 1

  if (guess === target.value) {
    status.value = 'correct'
    guesses.value.push({ attempt: attempts.value, value: guess, hint: 'correct' })
  } else if (guess < target.value) {
    status.value = 'low'
    guesses.value.push({ attempt: attempts.value, value: guess, hint: 'low' })
  } else {
    status.value = 'high'
    guesses.value.push({ attempt: attempts.value, value: guess, hint: 'high' })
  }
}

// Derived state
const feedbackMessage = computed(() => {
  switch (status.value) {
    case 'low':
      return 'Too low — try a higher number.'
    case 'high':
      return 'Too high — try a lower number.'
    case 'correct':
      return `Correct! You guessed the number in ${attempts.value} ${attempts.value === 1 ? 'attempt' : 'attempts'}.`
    default:
      return `Guess a number between ${min} and ${max}.`
  }
})

const isFinished = computed(() => status.value === 'correct')

onMounted(() => {
  resetGame()
})
</script>

<template>
  <div class="container-center">
    <main class="card" role="main" aria-labelledby="game-title">
      <div class="card-header">
        <h1 id="game-title" class="card-title">Number Guessing Game</h1>
        <p class="card-subtitle">Try to guess the secret number between {{ min }} and {{ max }}.</p>
      </div>

      <div class="card-body">
        <!-- Range indicator and attempts -->
        <div class="meta" aria-live="polite">
          <span class="badge" title="Range">
            <strong>Range:</strong> {{ min }} - {{ max }}
          </span>
          <span class="badge" title="Attempts">
            <strong>Attempts:</strong> {{ attempts }}
          </span>
        </div>

        <!-- Input and submit -->
        <form class="input-row" @submit.prevent="submitGuess" novalidate>
          <div>
            <input
              class="input"
              :class="{ error: !!error }"
              id="guess"
              name="guess"
              type="number"
              inputmode="numeric"
              :min="min"
              :max="max"
              step="1"
              placeholder="Enter your guess"
              v-model="currentGuess"
              :disabled="isFinished"
              :aria-invalid="!!error"
              aria-describedby="guess-help guess-error"
            />
            <p id="guess-help" class="helper">Enter a whole number between {{ min }} and {{ max }}.</p>
            <p id="guess-error" class="helper error" v-if="error">{{ error }}</p>
          </div>

          <button class="button" type="submit" :disabled="isFinished">
            Guess
          </button>
        </form>

        <!-- Feedback -->
        <div
          class="feedback"
          :class="{
            success: status === 'correct',
            error: status === 'high' || status === 'low'
          }"
          aria-live="polite"
          role="status"
        >
          {{ feedbackMessage }}
        </div>

        <!-- History -->
        <section class="history" aria-labelledby="history-title" v-if="guesses.length">
          <h3 id="history-title">Guess History</h3>
          <ul class="history-list">
            <li v-for="g in guesses" :key="g.attempt" class="history-item">
              <span>
                <strong>#{{ g.attempt }}</strong>
                — You guessed <strong>{{ g.value }}</strong>
              </span>
              <em style="opacity:0.8">
                {{
                  g.hint === 'correct'
                    ? 'Correct'
                    : g.hint === 'low'
                      ? 'Too low'
                      : 'Too high'
                }}
              </em>
            </li>
          </ul>
        </section>

        <!-- Actions -->
        <div class="actions">
          <button class="button ghost" type="button" @click="resetGame">
            {{ isFinished ? 'Play Again' : 'Restart' }}
          </button>
          <button class="button secondary" type="button" v-if="isFinished" @click="resetGame">
            Play Again
          </button>
        </div>
      </div>
    </main>
  </div>
</template>

<script lang="ts">
export default {
  name: 'HomeView',
  computed: {
    // expose min/max for template
    min(): number {
      return 1
    },
    max(): number {
      return 100
    }
  }
}
</script>

<style scoped>
/* Scoped styles are minimal because theme is in global assets */
</style>
