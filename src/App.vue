<template>
  <div>
    <div v-if="isLoading" class="loading-screen">
      Loading audio, please wait...
    </div>

    <div v-else>
      <button @click="toggleLanguage">
        {{ language === 'en' ? 'EN' : 'FR' }}
      </button>

      <div class="starry-sky">
        <div v-for="(star, i) in backgroundStars" :key="'bg-star-' + i" class="title-star" :style="{
          top: star.top + '%',
          left: star.left + '%',
          width: star.size + 'px',
          height: star.size + 'px',
          background: star.color,
          animationDelay: star.animationDelay,
        }" />
      </div>

      <main class="container">
        <div class="grid">
          <TravelerCard v-for="(instrument, index) in instruments" :key="instrument.name_en" :instrument="instrument"
            :language="language" :volume="volumes[index]" :onToggle="() => toggleInstrument(index)"
            :onVolumeChange="(vol) => setVolume(index, vol)" />
        </div>
      </main>
    </div>
  </div>
</template>

<script setup>
import TravelerCard from './components/TravelerCard.vue'
import { ref, onMounted, onBeforeUnmount, watch } from 'vue'
import instrumentsData from './assets/instruments.json'

const language = ref('en') // 'en' or 'fr'

const instruments = ref(
  instrumentsData.map((i) => ({
    ...i,
    enabled: false,
  }))
)

let audioCtx = null
const instrumentBuffers = new Map() // name_en => AudioBuffer
const sources = new Map() // name_en => AudioBufferSourceNode
const gains = new Map() // name_en => GainNode
const volumes = ref(instruments.value.map(() => 100)) // 100% default volume
const gainNodes = new Map()

const isLoading = ref(true)  // <-- loading flag

// Helper to decode audio data as a Promise
function decodeAudioDataAsync(ctx, arrayBuffer) {
  return new Promise((resolve, reject) => {
    ctx.decodeAudioData(arrayBuffer, resolve, reject)
  })
}

async function setupAudio() {
  audioCtx = new (window.AudioContext || window.webkitAudioContext)()

  // Load and decode all audio files
  for (const inst of instruments.value) {
    const res = await fetch(inst.file)
    const arrayBuffer = await res.arrayBuffer()
    const decoded = await decodeAudioDataAsync(audioCtx, arrayBuffer)
    instrumentBuffers.set(inst.name_en, decoded)
  }

  // Create a source and gain node per instrument, start all muted and looped
  for (const inst of instruments.value) {
    const source = audioCtx.createBufferSource()
    source.buffer = instrumentBuffers.get(inst.name_en)
    source.loop = true

    const gainNode = audioCtx.createGain()
    gainNode.gain.value = 0 // start muted

    source.connect(gainNode).connect(audioCtx.destination)
    source.start(0)

    sources.set(inst.name_en, source)
    gains.set(inst.name_en, gainNode)
  }
}

// Unlock audio on first user interaction
function ensureAudioStarted() {
  if (audioCtx && audioCtx.state === 'suspended') {
    audioCtx.resume()
  }
}

const setVolume = (index, vol) => {
  volumes.value[index] = vol

  const inst = instruments.value[index]
  const gainNode = gains.get(inst.name_en)
  if (!gainNode) return

  gainNode.gain.value = (inst.enabled ? vol / 100 : 0)
}

const toggleInstrument = (index) => {
  const inst = instruments.value[index]
  inst.enabled = !inst.enabled

  const gainNode = gains.get(inst.name_en)
  if (!gainNode) return

  gainNode.gain.value = inst.enabled ? volumes.value[index] / 100 : 0
}

const toggleLanguage = () => {
  language.value = language.value === 'en' ? 'fr' : 'en'
}

function detectBrowserLanguage() {
  const lang = navigator.language || navigator.userLanguage || 'en'
  if (lang.startsWith('fr')) return 'fr'
  return 'en'
}

// Starry sky background
const backgroundStars = ref([])

function random(min, max) {
  return Math.random() * (max - min) + min
}

function generateBackgroundStars() {
  const count = 250
  backgroundStars.value = []

  for (let i = 0; i < count; i++) {
    const colorChance = Math.random()
    let color = 'white'
    if (colorChance < 0.2) color = '#ffa500' // orange
    else if (colorChance < 0.4) color = '#ffff99' // yellowish

    backgroundStars.value.push({
      top: random(0, 100),
      left: random(0, 100),
      size: random(0.5, 2),
      color,
      animationDelay: `${random(0, 3)}s`,
    })
  }
}

onMounted(async () => {
  language.value = detectBrowserLanguage()

  await setupAudio()

  generateBackgroundStars()

  isLoading.value = false // <-- done loading

  // Unlock audio on first user interaction (mobile requirement)
  const unlockAudio = () => {
    ensureAudioStarted()
    window.removeEventListener('touchstart', unlockAudio)
    window.removeEventListener('click', unlockAudio)
  }
  window.addEventListener('touchstart', unlockAudio, { once: true })
  window.addEventListener('click', unlockAudio, { once: true })
})

onBeforeUnmount(() => {
  // Stop all audio sources
  sources.forEach((source) => {
    source.stop(0)
  })
  audioCtx && audioCtx.close()
})
</script>



<style>
body {
  margin: 0;
  font-family: system-ui, sans-serif;
  background: #000;
  color: #fff;
  overflow-x: hidden;
}

.loading-screen {
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 1.4rem;
  color: #aaa;
  background: #000;
}

button {
  position: fixed;
  top: 15px;
  right: 15px;
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  z-index: 1000;
  display: flex;
  align-items: center;
  color: #ffffff78;
  transition: all 0.3s ease;
}

button:hover {
  color: #ffffffcc;
}

.container {
  margin: 0 auto;
  padding: 1rem;
  position: relative;
  z-index: 1;
}

.grid {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 1rem;
  padding-bottom: 0.5rem;
  margin-top: 80px;
  position: relative;
  z-index: 1;
}

/* Starry sky background with flickering colored stars */
.starry-sky {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  z-index: 0;
  background: radial-gradient(ellipse at center, #000 0%, #000 100%);
}

.title-star {
  position: absolute;
  border-radius: 50%;
  opacity: 0.6;
  animation: flicker 2s infinite ease-in-out;
}

@keyframes flicker {

  0%,
  100% {
    opacity: 0.2;
  }

  50% {
    opacity: 1;
  }
}

@media (max-width: 768px) {
  .card {
    width: 100%;
    height: auto;
  }

  .container {
    padding: 1rem 0.5rem;
  }
}
</style>
