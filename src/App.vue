<template>
  <div>
    <button v-if="!loading" @click="toggleLanguage">
      {{ language === 'en' ? 'EN' : 'FR' }}
    </button>

    <div v-if="loading" class="loading-screen">
      <p>Loading audio assets...</p>
    </div>

    <div v-else class="starry-sky">
      <div v-for="(star, i) in backgroundStars" :key="'bg-star-' + i" class="title-star" :style="{
        top: star.top + '%',
        left: star.left + '%',
        width: star.size + 'px',
        height: star.size + 'px',
        background: star.color,
        animationDelay: star.animationDelay
      }" />
    </div>

    <main v-if="!loading" class="container">
      <div class="grid">
        <TravelerCard v-for="(instrument, index) in instruments" :key="instrument.name_en" :instrument="instrument"
          :language="language" :onToggle="() => toggleInstrument(index)" />
      </div>
    </main>
  </div>
</template>

<script setup>
import TravelerCard from './components/TravelerCard.vue'
import { ref, onMounted, onBeforeUnmount } from 'vue'
import instrumentsData from './assets/instruments.json'

const language = ref('en')
const loading = ref(true)

const instruments = ref(
  instrumentsData.map((i) => ({
    ...i,
    enabled: false,
  }))
)

// Audio setup
let audioCtx = null
const instrumentNodes = new Map() // { name_en => { type: 'buffer', buffer: AudioBuffer } }
const playingSources = new Map() // { name_en => AudioBufferSourceNode }

function decodeAudioDataAsync(ctx, arrayBuffer) {
  return new Promise((resolve, reject) => {
    ctx.decodeAudioData(
      arrayBuffer,
      (decodedData) => resolve(decodedData),
      (err) => reject(err)
    )
  })
}

function detectBrowserLanguage() {
  const lang = navigator.language || navigator.userLanguage || 'en'
  if (lang.startsWith('fr')) return 'fr'
  return 'en'
}

// Called on first user interaction to unlock AudioContext
function ensureAudioStarted() {
  if (audioCtx.state === 'suspended') {
    audioCtx.resume()
  }
}

// Play audio buffer for given instrument (looped)
function playInstrument(name_en) {
  if (!instrumentNodes.has(name_en)) return

  stopInstrument(name_en) // stop existing source if any

  const source = audioCtx.createBufferSource()
  source.buffer = instrumentNodes.get(name_en).buffer
  source.loop = true
  source.connect(audioCtx.destination)
  source.start(0)
  playingSources.set(name_en, source)
}

function stopInstrument(name_en) {
  const source = playingSources.get(name_en)
  if (source) {
    source.stop()
    source.disconnect()
    playingSources.delete(name_en)
  }
}

const toggleInstrument = (index) => {
  const instrument = instruments.value[index]
  instrument.enabled = !instrument.enabled

  if (instrument.enabled) {
    playInstrument(instrument.name_en)
  } else {
    stopInstrument(instrument.name_en)
  }
}

const toggleLanguage = () => {
  language.value = language.value === 'en' ? 'fr' : 'en'
}

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
  generateBackgroundStars()

  audioCtx = new (window.AudioContext || window.webkitAudioContext)()

  try {
    for (const inst of instruments.value) {
      const res = await fetch(inst.file)
      const arrayBuffer = await res.arrayBuffer()
      const decoded = await decodeAudioDataAsync(audioCtx, arrayBuffer)
      instrumentNodes.set(inst.name_en, {
        type: 'buffer',
        buffer: decoded,
      })
    }
  } catch (e) {
    console.warn('Audio preloading error:', e)
  } finally {
    loading.value = false
  }

  // Unlock AudioContext on first user gesture
  window.addEventListener('pointerdown', ensureAudioStarted, { once: true })
  window.addEventListener('touchstart', ensureAudioStarted, { once: true })
  window.addEventListener('click', ensureAudioStarted, { once: true })
})

onBeforeUnmount(() => {
  playingSources.forEach((source) => {
    source.stop()
    source.disconnect()
  })
  playingSources.clear()
  instrumentNodes.clear()
  if (audioCtx) {
    audioCtx.close()
  }
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
