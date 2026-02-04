<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue'
import AuthModal from './AuthModal.vue'

const props = defineProps<{
  coverImage?: string
  audioSrc?: string
  title?: string
  artist?: string
}>()

interface Track {
  id: number
  title: string
  artist: string
  src: string
  cover: string
}

const defaultTrack: Track = {
  id: 1,
  title: props.title || 'Neon Fractal',
  artist: props.artist || 'Cyberia',
  src:
    props.audioSrc ||
    'https://raw.githubusercontent.com/muhammederdem/mini-player/master/mp3/1.mp3',
  cover: props.coverImage || new URL('../assets/cover.png', import.meta.url).href,
}

const audioRef = ref<HTMLAudioElement | null>(null)
const isPlaying = ref(false)
const progress = ref(0)
const currentTime = ref('0:00')
const duration = ref('0:00')

// Advanced State
const playlist = ref<Track[]>([defaultTrack])
const currentTrackIndex = ref(0)
const volume = ref(0.8)
const isMuted = ref(false)
const prevVolume = ref(0.8)
const showPlaylist = ref(false)
const showAuth = ref(false)
const errorMsg = ref('')

const currentTrack = computed(() => playlist.value[currentTrackIndex.value] || defaultTrack)

// Theme State
const currentTheme = ref('cyberpunk')
const themes = {
  cyberpunk: { primary: '#00ffff', secondary: '#ff00ff', bg: 'rgba(0,0,0,0.5)' },
  gold: { primary: '#ffd700', secondary: '#ffA500', bg: 'rgba(20,20,0,0.5)' },
  aurora: { primary: '#00ff7f', secondary: '#9370db', bg: 'rgba(0,20,20,0.5)' },
}

// Audio Context & Visualizer & EQ
let audioCtx: AudioContext | null = null
let analyser: AnalyserNode | null = null
let source: MediaElementAudioSourceNode | null = null
// EQ Nodes
let bassNode: BiquadFilterNode | null = null
let midNode: BiquadFilterNode | null = null
let trebleNode: BiquadFilterNode | null = null

const canvasRef = ref<HTMLCanvasElement | null>(null)
let animationId: number

// EQ State
const bassGain = ref(0)
const midGain = ref(0)
const trebleGain = ref(0)
const showEq = ref(false)

const toggleTheme = () => {
  const keys = Object.keys(themes)
  const nextIndex = (keys.indexOf(currentTheme.value) + 1) % keys.length
  currentTheme.value = keys[nextIndex] || 'cyberpunk'
  applyTheme(currentTheme.value)
}

const applyTheme = (themeName: string) => {
  const theme = themes[themeName as keyof typeof themes]
  document.documentElement.style.setProperty('--primary-color', theme.primary)
  document.documentElement.style.setProperty('--secondary-color', theme.secondary)
}

const initAudioContext = () => {
  if (audioCtx) return
  const AudioContext = window.AudioContext || (window as any).webkitAudioContext
  audioCtx = new AudioContext()
  analyser = audioCtx.createAnalyser()
  analyser.fftSize = 256

  // Create EQ Nodes
  bassNode = audioCtx.createBiquadFilter()
  bassNode.type = 'lowshelf'
  bassNode.frequency.value = 200 // Hz

  midNode = audioCtx.createBiquadFilter()
  midNode.type = 'peaking'
  midNode.frequency.value = 1000
  midNode.Q.value = 1

  trebleNode = audioCtx.createBiquadFilter()
  trebleNode.type = 'highshelf'
  trebleNode.frequency.value = 3000

  if (audioRef.value) {
    try {
      source = audioCtx.createMediaElementSource(audioRef.value)
      // Connect Graph: Source -> Bass -> Mid -> Treble -> Analyser -> Destination
      source.connect(bassNode)
      bassNode.connect(midNode)
      midNode.connect(trebleNode)
      trebleNode.connect(analyser)
      analyser.connect(audioCtx.destination)
    } catch (e) {
      console.warn('MediaElementSource already attached', e)
    }
  }

  drawVisualizer()
}

const updateEq = (type: 'bass' | 'mid' | 'treble', event: Event) => {
  const input = event.target as HTMLInputElement
  const val = parseFloat(input.value)

  if (!audioCtx) initAudioContext()

  if (type === 'bass') {
    bassGain.value = val
    if (bassNode) bassNode.gain.value = val
  } else if (type === 'mid') {
    midGain.value = val
    if (midNode) midNode.gain.value = val
  } else if (type === 'treble') {
    trebleGain.value = val
    if (trebleNode) trebleNode.gain.value = val
  }
}

const drawVisualizer = () => {
  if (!canvasRef.value || !analyser) return

  const canvas = canvasRef.value
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const bufferLength = analyser.frequencyBinCount
  const dataArray = new Uint8Array(bufferLength)

  const draw = () => {
    animationId = requestAnimationFrame(draw)
    analyser!.getByteFrequencyData(dataArray)

    // Clear with semi-transparent background for trail effect? No, just clear
    ctx.clearRect(0, 0, canvas.width, canvas.height)

    // Get current theme colors from CSS vars
    const primary =
      getComputedStyle(document.documentElement).getPropertyValue('--primary-color').trim() ||
      '#00ffff'

    // Circular Visualizer
    const centerX = canvas.width / 2
    const centerY = canvas.height / 2
    const radius = 80

    ctx.beginPath()
    ctx.arc(centerX, centerY, radius, 0, 2 * Math.PI)
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.1)'
    ctx.stroke()

    const bars = bufferLength
    const step = (Math.PI * 2) / bars

    for (let i = 0; i < bars; i++) {
      const barHeight = ((dataArray[i] || 0) / 255) * 50
      const angle = i * step

      const x1 = centerX + Math.cos(angle) * radius
      const y1 = centerY + Math.sin(angle) * radius
      const x2 = centerX + Math.cos(angle) * (radius + barHeight)
      const y2 = centerY + Math.sin(angle) * (radius + barHeight)

      ctx.beginPath()
      ctx.moveTo(x1, y1)
      ctx.lineTo(x2, y2)

      // Use HSL for rainbow or Primary Color
      // Let's use Primary color but vary lightness or alpha
      ctx.strokeStyle = primary
      ctx.lineWidth = 2
      ctx.stroke()
    }
  }
  draw()
}

const togglePlay = async () => {
  if (!audioRef.value) return

  // Ensure AudioContext is running
  if (!audioCtx) {
    initAudioContext()
  }
  if (audioCtx && audioCtx.state === 'suspended') {
    await audioCtx.resume()
  }

  if (isPlaying.value) {
    audioRef.value.pause()
  } else {
    try {
      await audioRef.value.play()
      errorMsg.value = ''
    } catch (e) {
      console.error('Playback failed:', e)
      errorMsg.value = 'Playback failed. Try a local file.'
      isPlaying.value = false // Revert state
      return
    }
  }
  isPlaying.value = !isPlaying.value
}

const updateProgress = () => {
  if (!audioRef.value) return
  const { currentTime: current, duration: total } = audioRef.value
  if (isNaN(total)) return
  progress.value = (current / total) * 100
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
  nextTrack()
}

const updateVolume = (event: Event) => {
  const input = event.target as HTMLInputElement
  const val = parseFloat(input.value)
  volume.value = val
  if (audioRef.value) audioRef.value.volume = val
  isMuted.value = val === 0
}

const toggleMute = () => {
  if (isMuted.value) {
    volume.value = prevVolume.value || 0.5
    isMuted.value = false
  } else {
    prevVolume.value = volume.value
    volume.value = 0
    isMuted.value = true
  }
  if (audioRef.value) audioRef.value.volume = volume.value
}

const nextTrack = () => {
  let nextIndex = currentTrackIndex.value + 1
  if (nextIndex >= playlist.value.length) nextIndex = 0
  playTrack(nextIndex)
}

const prevTrack = () => {
  let prevIndex = currentTrackIndex.value - 1
  if (prevIndex < 0) prevIndex = playlist.value.length - 1
  playTrack(prevIndex)
}

const playTrack = (index: number) => {
  currentTrackIndex.value = index
  isPlaying.value = true
  setTimeout(() => {
    if (audioRef.value) audioRef.value.play()
  }, 50)
}

const handleDrop = (e: DragEvent) => {
  const files = e.dataTransfer?.files
  if (!files) return

  for (let i = 0; i < files.length; i++) {
    const file = files[i]
    if (!file) continue
    if (file.type.startsWith('audio/')) {
      const url = URL.createObjectURL(file)
      playlist.value.push({
        id: Date.now() + i,
        title: file.name.replace(/\.[^/.]+$/, ''),
        artist: 'Local File',
        src: url,
        cover: defaultTrack.cover,
      })
    }
  }
  if (!isPlaying.value && playlist.value.length > 1) {
    // Optional
  }
}

const handleKeydown = (e: KeyboardEvent) => {
  if (e.code === 'Space') {
    e.preventDefault()
    togglePlay()
  } else if (e.code === 'ArrowRight') {
    if (audioRef.value) audioRef.value.currentTime += 5
  } else if (e.code === 'ArrowLeft') {
    if (audioRef.value) audioRef.value.currentTime -= 5
  } else if (e.code === 'ArrowUp') {
    e.preventDefault()
    volume.value = Math.min(1, volume.value + 0.1)
    if (audioRef.value) audioRef.value.volume = volume.value
  } else if (e.code === 'ArrowDown') {
    e.preventDefault()
    volume.value = Math.max(0, volume.value - 0.1)
    if (audioRef.value) audioRef.value.volume = volume.value
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
  applyTheme('cyberpunk')
  if (audioRef.value) {
    audioRef.value.volume = volume.value
  }
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  if (animationId) cancelAnimationFrame(animationId)
  if (audioCtx) audioCtx.close()
})
</script>

<template>
  <div class="player-container" @dragover.prevent @drop.prevent="handleDrop">
    <AuthModal v-if="showAuth" @close="showAuth = false" />
    <div class="glass-card" :class="{ 'playlist-open': showPlaylist }">
      <!-- Playlist Drawer -->
      <transition name="slide">
        <div class="playlist-drawer" v-if="showPlaylist">
          <h3>Playlist</h3>
          <ul class="track-list">
            <li
              v-for="(track, index) in playlist"
              :key="track.id"
              :class="{ active: index === currentTrackIndex }"
              @click="playTrack(index)"
            >
              <span class="track-num">{{ index + 1 }}</span>
              <div class="track-meta-list">
                <span class="track-title-list">{{ track.title }}</span>
                <span class="track-artist-list">{{ track.artist }}</span>
              </div>
              <div class="playing-indicator" v-if="index === currentTrackIndex && isPlaying">
                <span></span><span></span><span></span>
              </div>
            </li>
          </ul>
        </div>
      </transition>

      <!-- Main Player View -->
      <div class="main-player">
        <div class="top-bar">
          <button class="icon-btn" @click="showPlaylist = !showPlaylist" title="Playlist">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="20"
              height="20"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <line x1="8" y1="6" x2="21" y2="6"></line>
              <line x1="8" y1="12" x2="21" y2="12"></line>
              <line x1="8" y1="18" x2="21" y2="18"></line>
              <line x1="3" y1="6" x2="3.01" y2="6"></line>
              <line x1="3" y1="12" x2="3.01" y2="12"></line>
              <line x1="3" y1="18" x2="3.01" y2="18"></line>
            </svg>
          </button>
          <div class="volume-container">
            <button @click="toggleMute" class="icon-btn small">
              <svg
                v-if="isMuted || volume === 0"
                xmlns="http://www.w3.org/2000/svg"
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"></polygon>
                <line x1="23" y1="9" x2="17" y2="15"></line>
                <line x1="17" y1="9" x2="23" y2="15"></line>
              </svg>
              <svg
                v-else
                xmlns="http://www.w3.org/2000/svg"
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"></polygon>
                <path d="M19.07 4.93a10 10 0 0 1 0 14.14M15.54 8.46a5 5 0 0 1 0 7.07"></path>
              </svg>
            </button>
            <input
              type="range"
              min="0"
              max="1"
              step="0.01"
              :value="volume"
              @input="updateVolume"
              class="volume-slider"
            />
          </div>
          <button class="icon-btn" @click="showAuth = true" title="Register">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="20"
              height="20"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
              <circle cx="12" cy="7" r="4"></circle>
            </svg>
          </button>
        </div>

        <div class="cover-art-wrapper">
          <div class="cover-art">
            <canvas ref="canvasRef" width="300" height="300" class="visualizer-canvas"></canvas>
            <img :src="currentTrack.cover" alt="Album Cover" :class="{ 'fast-spin': isPlaying }" />
            <div class="glow-effect"></div>
          </div>
        </div>

        <div class="track-info">
          <h2>{{ currentTrack.title }}</h2>
          <p>{{ currentTrack.artist }}</p>
          <p v-if="errorMsg" class="error-text">{{ errorMsg }}</p>
        </div>

        <div class="extra-controls">
          <button class="icon-btn small" @click="showEq = !showEq" :class="{ active: showEq }">
            EQ
          </button>
        </div>

        <div class="eq-panel" v-if="showEq">
          <div class="eq-band">
            <span>Bass</span>
            <input
              type="range"
              min="-10"
              max="10"
              step="1"
              :value="bassGain"
              @input="(e) => updateEq('bass', e)"
            />
          </div>
          <div class="eq-band">
            <span>Mid</span>
            <input
              type="range"
              min="-10"
              max="10"
              step="1"
              :value="midGain"
              @input="(e) => updateEq('mid', e)"
            />
          </div>
          <div class="eq-band">
            <span>High</span>
            <input
              type="range"
              min="-10"
              max="10"
              step="1"
              :value="trebleGain"
              @input="(e) => updateEq('treble', e)"
            />
          </div>
        </div>

        <div class="controls">
          <div class="progress-bar-wrapper">
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

          <div class="media-controls">
            <button class="icon-btn medium" @click="prevTrack">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                width="24"
                height="24"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <polygon points="19 20 9 12 19 4 19 20"></polygon>
                <line x1="5" y1="19" x2="5" y2="5"></line>
              </svg>
            </button>

            <button class="play-btn" @click="togglePlay">
              <div v-if="isPlaying" class="icon pause"></div>
              <div v-else class="icon play"></div>
            </button>

            <button class="icon-btn medium" @click="nextTrack">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                width="24"
                height="24"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <polygon points="5 4 15 12 5 20 5 4"></polygon>
                <line x1="19" y1="5" x2="19" y2="19"></line>
              </svg>
            </button>
          </div>
        </div>
      </div>

      <audio
        ref="audioRef"
        :src="currentTrack.src"
        @timeupdate="updateProgress"
        @ended="onEnded"
        @loadedmetadata="updateProgress"
        crossorigin="anonymous"
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
  perspective: 1000px;
}

.glass-card {
  position: relative;
  width: 350px;
  height: 550px; /* Fixed height for consistent layout */
  padding: 1.5rem;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 24px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.37);
  display: flex;
  overflow: hidden;
  transition: width 0.4s ease;
}

.glass-card.playlist-open {
  width: 600px; /* Expand for playlist */
}

/* Playlist Drawer */
.playlist-drawer {
  width: 250px; /* Fixed width */
  height: 100%;
  border-right: 1px solid rgba(255, 255, 255, 0.1);
  padding-right: 1rem;
  margin-right: 1rem;
  overflow-y: auto;
  flex-shrink: 0;
}

.playlist-drawer h3 {
  font-size: 1.2rem;
  margin-bottom: 1rem;
  color: #00ffff;
  text-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
}

.track-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.track-list li {
  display: flex;
  align-items: center;
  padding: 0.8rem;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.2s;
  margin-bottom: 0.5rem;
}

.track-list li:hover {
  background: rgba(255, 255, 255, 0.1);
}

.track-list li.active {
  background: rgba(0, 255, 255, 0.15);
  border: 1px solid rgba(0, 255, 255, 0.3);
}

.track-num {
  font-size: 0.8rem;
  opacity: 0.5;
  width: 20px;
}

.track-meta-list {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.track-title-list {
  font-size: 0.9rem;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.error-text {
  color: #ff5555;
  font-size: 0.8rem;
  margin-top: 5px;
}

.track-artist-list {
  font-size: 0.75rem;
  opacity: 0.6;
}

.playing-indicator span {
  display: inline-block;
  width: 3px;
  height: 10px;
  background: #00ffff;
  margin: 0 1px;
  animation: bounce 1s infinite;
}

.playing-indicator span:nth-child(2) {
  animation-delay: 0.2s;
}
.playing-indicator span:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes bounce {
  0%,
  100% {
    transform: scaleY(1);
  }
  50% {
    transform: scaleY(2);
  }
}

/* Main Player */
.main-player {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;
  height: 100%;
  min-width: 0; /* Prevent flex item overflow */
}

.top-bar {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 40px;
}

.volume-container {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: rgba(0, 0, 0, 0.2);
  padding: 5px 10px;
  border-radius: 20px;
}

.volume-slider {
  width: 60px;
  height: 3px;
  -webkit-appearance: none;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 2px;
}

.volume-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 10px;
  height: 10px;
  background: white;
  border-radius: 50%;
  cursor: pointer;
}

.cover-art-wrapper {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  position: relative;
}

.cover-art {
  position: relative;
  width: 220px;
  height: 220px;
  border-radius: 50%;
  box-shadow: 0 0 30px rgba(0, 255, 255, 0.3);
  display: flex;
  justify-content: center;
  align-items: center;
}

.visualizer-canvas {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 380px;
  height: 380px;
  z-index: -1;
  pointer-events: none;
}

.cover-art img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 50%;
  transition: transform 0.5s ease;
  z-index: 1;
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
  margin: 1rem 0;
}

.track-info h2 {
  font-size: 1.4rem;
  font-weight: 700;
  margin: 0;
  background: linear-gradient(90deg, #00ffff, #ff00ff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 250px;
}

.track-info p {
  margin: 0.3rem 0 0;
  opacity: 0.7;
  font-size: 0.9rem;
}

.controls {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.2rem;
  margin-bottom: 0.5rem;
}

.progress-bar-wrapper {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.75rem;
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
  transition: transform 0.2s;
}

.progress-bar::-webkit-slider-thumb:hover {
  transform: scale(1.2);
}

.media-controls {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.icon-btn {
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.7);
  cursor: pointer;
  transition: all 0.2s;
  padding: 5px;
  border-radius: 50%;
}

.icon-btn:hover {
  color: white;
  background: rgba(255, 255, 255, 0.1);
}

.icon-btn.medium {
  transform: scale(1.2);
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
  box-shadow: 0 0 15px rgba(0, 0, 0, 0.2);
}

.play-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  box-shadow: 0 0 25px rgba(0, 255, 255, 0.4);
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

/* EQ Panel */
.extra-controls {
  margin-bottom: 0.5rem;
}

.icon-btn.small {
  font-size: 0.8rem;
  padding: 2px 8px;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.icon-btn.active {
  background: rgba(0, 255, 255, 0.2);
  border-color: #00ffff;
  color: #00ffff;
  text-shadow: 0 0 5px rgba(0, 255, 255, 0.5);
}

.eq-panel {
  display: flex;
  gap: 1rem;
  background: rgba(0, 0, 0, 0.3);
  padding: 10px;
  border-radius: 12px;
  margin-bottom: 1rem;
  animation: fadeIn 0.3s ease;
}

.eq-band {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.3rem;
  font-size: 0.7rem;
}

.eq-band input[type='range'] {
  -webkit-appearance: none; /* Vertical sliders are tricky, stick to horizontal or use transform */
  width: 60px;
  height: 3px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 2px;
}

.eq-band input[type='range']::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 10px;
  height: 10px;
  background: #00ffff;
  border-radius: 50%;
  cursor: pointer;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-5px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* CSS Variables usage */
.track-info h2 {
  background: linear-gradient(
    90deg,
    var(--primary-color, #00ffff),
    var(--secondary-color, #ff00ff)
  );
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.playing-indicator span {
  background: var(--primary-color, #00ffff);
}

.icon-btn.active {
  border-color: var(--primary-color, #00ffff);
  color: var(--primary-color, #00ffff);
  text-shadow: 0 0 5px var(--primary-color, #00ffff);
}

.eq-band input[type='range']::-webkit-slider-thumb {
  background: var(--primary-color, #00ffff);
}

.playlist-drawer h3 {
  color: var(--primary-color, #00ffff);
  text-shadow: 0 0 10px var(--primary-color, #00ffff);
}

.slide-enter-from,
.slide-leave-to {
  width: 0;
  padding-right: 0;
  margin-right: 0;
  opacity: 0;
}
</style>
