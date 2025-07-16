<template>
  <div>
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
        animationDelay: star.animationDelay
      }" />
    </div>

    <main class="container">
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

const language = ref('en') // 'en' or 'fr'

const instruments = ref(
  instrumentsData.map((i) => ({
    ...i,
    enabled: false,
  }))
)

const audioElements = new Map()

const toggleInstrument = (index) => {
  const instrument = instruments.value[index]
  instrument.enabled = !instrument.enabled

  const audio = audioElements.get(instrument.name_en)
  if (!audio) return

  audio.muted = !instrument.enabled
}

const toggleLanguage = () => {
  language.value = language.value === 'en' ? 'fr' : 'en'
}

function detectBrowserLanguage() {
  const lang = navigator.language || navigator.userLanguage || 'en'
  if (lang.startsWith('fr')) return 'fr'
  return 'en'
}

onMounted(() => {
  language.value = detectBrowserLanguage()

  instruments.value.forEach((instrument) => {
    const audio = new Audio(`/${instrument.file}`)
    audio.loop = true
    audio.volume = 1
    audio.muted = true
    audio.play()
    audioElements.set(instrument.name_en, audio)
  })
  generateBackgroundStars()
})

onBeforeUnmount(() => {
  audioElements.forEach((audio) => {
    audio.pause()
    audio.src = ''
  })
})

// Fancy background stars
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
</script>

<style>
body {
  margin: 0;
  font-family: system-ui, sans-serif;
  background: #000;
  color: #fff;
  overflow-x: hidden;
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

.card {
  border: 2px solid #444;
  border-radius: 8px;
  padding: 1rem;
  text-align: center;
  background: rgba(255, 255, 255, 0.05);
  cursor: pointer;
  transition: all 0.2s ease;
}

.card:hover {
  background: rgba(255, 255, 255, 0.1);
}

.card.active {
  border-color: #4caf50;
  background: rgba(76, 175, 80, 0.2);
}

.name {
  font-weight: bold;
  font-size: 1.2rem;
}

.planet {
  font-size: 0.9rem;
  opacity: 0.7;
}

.planet-image {
  width: 100%;
  height: 120px;
  object-fit: contain;
  margin-bottom: 0.5rem;
  user-select: none;
  pointer-events: none;
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
