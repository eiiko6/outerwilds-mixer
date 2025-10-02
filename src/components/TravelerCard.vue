<template>
  <div class="card" :class="{ active: instrument.enabled }" @click="onToggle" :style="activeStyle">
    <div class="planet-image-wrapper">
      <img :src="`./${instrument.name_en.toLowerCase()}0.png`" :alt="displayedName" class="planet-image base"
        crossorigin="anonymous" />

      <transition name="fade">
        <img v-if="instrument.enabled" ref="imgRef" :src="`./${instrument.name_en.toLowerCase()}1.png`"
          :alt="displayedName" class="planet-image overlay" @load="onImageLoad" crossorigin="anonymous" />
      </transition>
    </div>
    <h2 class="name">{{ displayedName }}</h2>
    <p class="planet">{{ getPlanet(instrument.name_en) }}</p>

    <div class="volume-slider" @click.stop>
      <!-- <label>Volume: {{ volume }}%</label> -->
      <input type="range" min="0" max="100" :value="volume" @input="onVolumeInput" />
    </div>
  </div>
</template>

<script setup>
import { ref, watch, computed } from 'vue'

const props = defineProps({
  instrument: Object,
  onToggle: Function,
  language: {
    type: String,
    default: 'en',
  },
  volume: {
    type: Number,
    default: 100,
  },
  onVolumeChange: Function,
})

function onVolumeInput(event) {
  const vol = Number(event.target.value)
  props.onVolumeChange && props.onVolumeChange(vol)
}

const imgRef = ref(null)
const dominantColor = ref(null)

function getPlanet(name_en) {
  const map = {
    Gabbro: "Giant's Deep",
    Riebeck: 'Timber Hearth',
    Chert: 'Hourglass Twins',
    Feldspar: 'Dark Bramble',
    Esker: 'Attlerock',
    Solanum: 'Quantum Moon',
    Prisoner: 'The Stranger',
  }
  return map[name_en] || 'Unknown'
}

const displayedName = computed(() => {
  return props.language === 'fr' ? props.instrument.name_fr : props.instrument.name_en
})

// RGB <-> HSL conversion helpers

function rgbToHsl(r, g, b) {
  r /= 255
  g /= 255
  b /= 255
  const max = Math.max(r, g, b), min = Math.min(r, g, b)
  let h, s, l = (max + min) / 2

  if (max === min) {
    h = s = 0 // achromatic
  } else {
    const d = max - min
    s = l > 0.5 ? d / (2 - max - min) : d / (max + min)
    switch (max) {
      case r: h = (g - b) / d + (g < b ? 6 : 0); break
      case g: h = (b - r) / d + 2; break
      case b: h = (r - g) / d + 4; break
    }
    h /= 6
  }
  return [h, s, l]
}

function hslToRgb(h, s, l) {
  let r, g, b

  if (s === 0) {
    r = g = b = l // achromatic
  } else {
    const hue2rgb = (p, q, t) => {
      if (t < 0) t += 1
      if (t > 1) t -= 1
      if (t < 1 / 6) return p + (q - p) * 6 * t
      if (t < 1 / 2) return q
      if (t < 2 / 3) return p + (q - p) * (2 / 3 - t) * 6
      return p
    }

    const q = l < 0.5 ? l * (1 + s) : l + s - l * s
    const p = 2 * l - q
    r = hue2rgb(p, q, h + 1 / 3)
    g = hue2rgb(p, q, h)
    b = hue2rgb(p, q, h - 1 / 3)
  }

  return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)]
}

function adjustColor(r, g, b) {
  let [h, s, l] = rgbToHsl(r, g, b)

  // Boost lightness if too dark
  if (l < 0.4) l = 0.4

  // Boost saturation if too low
  if (s < 0.6) s = 0.6

  return hslToRgb(h, s, l)
}

function onImageLoad() {
  if (!props.instrument.enabled) {
    dominantColor.value = null
    return
  }
  const img = imgRef.value
  if (!img) return

  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')
  canvas.width = img.naturalWidth
  canvas.height = img.naturalHeight
  ctx.drawImage(img, 0, 0)

  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  const data = imageData.data

  let r = 0, g = 0, b = 0, count = 0
  for (let i = 0; i < data.length; i += 40) {
    r += data[i]
    g += data[i + 1]
    b += data[i + 2]
    count++
  }
  r = Math.round(r / count)
  g = Math.round(g / count)
  b = Math.round(b / count)

  const [br, bg, bb] = adjustColor(r, g, b)
  dominantColor.value = `rgb(${br},${bg},${bb})`
}

watch(() => props.instrument.enabled, (enabled) => {
  if (!enabled) {
    dominantColor.value = null
  }
})

const activeStyle = computed(() => {
  if (!props.instrument.enabled || !dominantColor.value) return {}

  return {
    borderColor: dominantColor.value,
    backgroundColor: dominantColor.value.replace('rgb', 'rgba').replace(')', ', 0.2)'),
  }
})
</script>

<style scoped>
.card {
  width: 180px;
  height: 270px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  border: 2px solid #444;
  border-radius: 8px;
  padding: 1rem;
  text-align: center;
  background: rgba(255, 255, 255, 0.05);
  cursor: pointer;
  transition: all 0.5s ease;
  user-select: none;
}

.card:hover {
  background: rgba(255, 255, 255, 0.1);
}

.planet-image-wrapper {
  position: relative;
  width: 100%;
  height: 160px;
  margin-bottom: 0.5rem;
}

.planet-image {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 170px;
  max-width: 100%;
  max-height: 170px;
  object-fit: contain;
  margin-bottom: 0.5rem;
  pointer-events: none;
  user-select: none;
}

.planet-image.base {
  z-index: 0;
}

.planet-image.overlay {
  z-index: 1;
}

.name {
  font-weight: bold;
  font-size: 1.2rem;
  color: #fff;
  margin: 0;
  margin-top: 30px;
  padding: 0;
}

.planet {
  font-size: 0.9rem;
  opacity: 0.7;
  color: #ddd;
  margin: 0;
  margin-bottom: 10px;
  padding: 0;
}

.volume-slider {
  color: #ddd;
  font-size: 0.85rem;
  user-select: none;
}

.volume-slider input[type="range"] {
  width: 100%;
  margin-top: 5px;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
