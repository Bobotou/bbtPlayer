<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const props = defineProps<{
  coverImage?: string
  audioSrc?: string
  title?: string
  artist?: string
}>()

const audioRef = ref<HTMLAudioElement | null>(null)
const isPlaying = ref(false)
const progress = ref(0)
const currentTime = ref('0:00')
const duration = ref('0:00')

const togglePlay = () => {
  if (!audioRef.value) return

  if (isPlaying.value) {
    audioRef.value.pause()
  } else {
    audioRef.value.play()
  }
  isPlaying.value = !isPlaying.value
}

const updateProgress = () => {
  if (!audioRef.value) return

  const { currentTime: current, duration: total } = audioRef.value

  if (isNaN(total)) return

  progress.value = (current / total) * 100

  // Format times
  currentTime.value = formatTime(current)
  duration.value = formatTime(total)
}

const formatTime = (seconds: number) => {
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}

const seek = (event: Event) => {
  const input = event.target as HTMLInputElement
  if (!audioRef.value) return

  const time = (parseFloat(input.value) / 100) * audioRef.value.duration
  audioRef.value.currentTime = time
}

const onEnded = () => {
  isPlaying.value = false
  progress.value = 0
  currentTime.value = '0:00'
}
</script>

<template>
  <div class="player-container">
    <div class="glass-card">
      <div class="cover-art">
        <img
          :src="coverImage || '/src/assets/cover.png'"
          alt="Album Cover"
          :class="{ 'fast-spin': isPlaying }"
        />
        <div class="glow-effect"></div>
      </div>

      <div class="track-info">
        <h2>{{ title || 'Neon Fractal' }}</h2>
        <p>{{ artist || 'Cyberia' }}</p>
      </div>

      <div class="controls">
        <div class="progress-container">
          <span class="time">{{ currentTime }}</span>
          <input
            type="range"
            min="0"
            max="100"
            :value="progress"
            @input="seek"
            class="progress-bar"
          />
          <span class="time">{{ duration }}</span>
        </div>

        <button class="play-btn" @click="togglePlay">
          <div v-if="isPlaying" class="icon pause"></div>
          <div v-else class="icon play"></div>
        </button>
      </div>

      <audio
        ref="audioRef"
        :src="audioSrc || 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3'"
        @timeupdate="updateProgress"
        @ended="onEnded"
        @loadedmetadata="updateProgress"
      ></audio>
    </div>
  </div>
</template>

<style scoped>
.player-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  font-family: 'Inter', sans-serif;
  color: white;
}

.glass-card {
  position: relative;
  width: 320px;
  padding: 2rem;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 24px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.37);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.5rem;
  overflow: hidden;
}

.cover-art {
  position: relative;
  width: 200px;
  height: 200px;
  border-radius: 50%;
  box-shadow: 0 0 30px rgba(0, 255, 255, 0.3);
}

.cover-art img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 50%;
  transition: transform 0.5s ease;
}

.rotation {
  animation: rotate 10s linear infinite;
}

.glow-effect {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 100%;
  height: 100%;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(0, 255, 255, 0.2) 0%, rgba(255, 0, 255, 0) 70%);
  pointer-events: none;
  z-index: -1;
}

.track-info {
  text-align: center;
}

.track-info h2 {
  font-size: 1.5rem;
  font-weight: 700;
  margin: 0;
  background: linear-gradient(90deg, #00ffff, #ff00ff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.track-info p {
  margin: 0.5rem 0 0;
  opacity: 0.7;
  font-size: 0.9rem;
}

.controls {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

.progress-container {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.8rem;
  opacity: 0.8;
}

.progress-bar {
  flex: 1;
  -webkit-appearance: none;
  height: 4px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 2px;
  cursor: pointer;
}

.progress-bar::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 12px;
  height: 12px;
  background: #fff;
  border-radius: 50%;
  box-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
  cursor: pointer;
}

.play-btn {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 0.3s ease;
}

.play-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  box-shadow: 0 0 20px rgba(0, 255, 255, 0.4);
  transform: scale(1.05);
}

.icon {
  width: 0;
  height: 0;
  border-style: solid;
}

.icon.play {
  border-width: 10px 0 10px 18px;
  border-color: transparent transparent transparent white;
  margin-left: 4px; /* Optical center */
}

.icon.pause {
  width: 14px;
  height: 18px;
  border-width: 0 4px;
  border-style: double;
  border-color: white;
}

@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.fast-spin {
  animation: rotate 20s linear infinite;
}
</style>
