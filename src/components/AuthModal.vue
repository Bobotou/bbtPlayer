<script setup lang="ts">
import { ref } from 'vue'
import { supabase } from '../lib/supabase'

const email = ref('')
const password = ref('')
const loading = ref(false)
const errorMessage = ref('')
const successMessage = ref('')
const isLoginMode = ref(true)

const emit = defineEmits(['close'])

const handleSubmit = async () => {
  loading.value = true
  errorMessage.value = ''
  successMessage.value = ''

  try {
    let error
    if (isLoginMode.value) {
      // Login Logic
      const { error: signInError } = await supabase.auth.signInWithPassword({
        email: email.value,
        password: password.value,
      })
      error = signInError
    } else {
      // Register Logic
      const { error: signUpError } = await supabase.auth.signUp({
        email: email.value,
        password: password.value,
      })
      error = signUpError
    }

    if (error) {
      errorMessage.value = error.message
    } else {
      if (isLoginMode.value) {
        successMessage.value = 'Access Granted. Welcome back.'
        setTimeout(() => {
          emit('close')
        }, 1500)
      } else {
        successMessage.value = 'Registration successful! Check your email for confirmation.'
        setTimeout(() => {
          isLoginMode.value = true // Switch to login after registration
        }, 3000)
      }
    }
  } catch (e: any) {
    errorMessage.value = 'An unexpected error occurred.'
    console.error(e)
  } finally {
    loading.value = false
  }
}

const toggleMode = () => {
  isLoginMode.value = !isLoginMode.value
  errorMessage.value = ''
  successMessage.value = ''
}
</script>

<template>
  <div class="modal-overlay" @click.self="$emit('close')">
    <div class="modal-card glass-card">
      <button class="close-btn" @click="$emit('close')">&times;</button>
      <h2>{{ isLoginMode ? 'Access System' : 'Join the Cyberverse' }}</h2>

      <div class="form-group">
        <label>Email ID</label>
        <input type="email" v-model="email" placeholder="cyber@punk.net" />
      </div>

      <div class="form-group">
        <label>Access Code</label>
        <input type="password" v-model="password" placeholder="********" />
      </div>

      <div class="message error" v-if="errorMessage">{{ errorMessage }}</div>
      <div class="message success" v-if="successMessage">{{ successMessage }}</div>

      <button class="action-btn" @click="handleSubmit" :disabled="loading">
        {{ loading ? 'Connecting...' : isLoginMode ? 'Login' : 'Initialize' }}
      </button>

      <p class="toggle-text">
        {{ isLoginMode ? 'New user?' : 'Already have an account?' }}
        <span @click="toggleMode">{{ isLoginMode ? 'Create Identity' : 'Login' }}</span>
      </p>
    </div>
  </div>
</template>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(5px);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-card {
  position: relative;
  width: 400px;
  padding: 2.5rem;
  background: rgba(15, 15, 21, 0.7);
  border: 1px solid rgba(0, 255, 255, 0.3);
  box-shadow: 0 0 40px rgba(0, 255, 255, 0.2);
  border-radius: 16px;
  color: white;
  text-align: center;
}

.close-btn {
  position: absolute;
  top: 10px;
  right: 15px;
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.5);
  font-size: 1.5rem;
  cursor: pointer;
  transition: color 0.3s;
}

.close-btn:hover {
  color: #00ffff;
}

h2 {
  margin-bottom: 2rem;
  font-family: 'Inter', sans-serif;
  font-weight: 700;
  background: linear-gradient(90deg, #00ffff, #ff00ff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.form-group {
  margin-bottom: 1.5rem;
  text-align: left;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  font-size: 0.8rem;
  color: rgba(0, 255, 255, 0.8);
  font-weight: 600;
  text-transform: uppercase;
}

input {
  width: 100%;
  padding: 12px;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  color: white;
  font-family: 'Inter', sans-serif;
  transition: all 0.3s ease;
}

input:focus {
  outline: none;
  border-color: #00ffff;
  box-shadow: 0 0 15px rgba(0, 255, 255, 0.2);
  background: rgba(0, 255, 255, 0.05);
}

.action-btn {
  width: 100%;
  padding: 12px;
  margin-top: 1rem;
  background: linear-gradient(90deg, #00ffff, #0088ff);
  border: none;
  border-radius: 8px;
  color: #000;
  font-weight: 700;
  font-size: 1rem;
  text-transform: uppercase;
  cursor: pointer;
  transition:
    transform 0.2s,
    box-shadow 0.2s;
}

.action-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 20px rgba(0, 255, 255, 0.4);
}

.action-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
  transform: none;
}

.message {
  margin-bottom: 1rem;
  font-size: 0.9rem;
  padding: 10px;
  border-radius: 6px;
}

.error {
  background: rgba(255, 0, 0, 0.2);
  color: #ff5555;
  border: 1px solid rgba(255, 0, 0, 0.3);
}

.success {
  background: rgba(0, 255, 0, 0.2);
  color: #55ff55;
  border: 1px solid rgba(0, 255, 0, 0.3);
}

.toggle-text {
  margin-top: 1rem;
  font-size: 0.9rem;
  color: rgba(255, 255, 255, 0.6);
}

.toggle-text span {
  color: #00ffff;
  cursor: pointer;
  text-decoration: underline;
  margin-left: 5px;
}

.toggle-text span:hover {
  text-shadow: 0 0 5px #00ffff;
}
</style>
