<!-- ColorPaletteGenerator.vue -->
<script setup lang="ts">
import { ref, reactive, watch } from 'vue'

interface Color {
  h: number
  s: number
  l: number
  shades?: Color[]
  rowIndex?: number
  index?: number
}

interface ColorFormats {
  hsl: string
  rgb: string
  hex: string
}

// State
const baseColor = reactive<Color>({ h: 45, s: 70, l: 60 })
const palette = ref<Color[]>([])
const selectedColor = ref<number | null>(null)
const selectedShade = ref<Color | null>(null)
const hexInput = ref('')

// Color conversion utilities
const HSLToRGB = (h: number, s: number, l: number) => {
  s /= 100
  l /= 100
  const k = (n: number) => (n + h / 30) % 12
  const a = s * Math.min(l, 1 - l)
  const f = (n: number) =>
    l - a * Math.max(-1, Math.min(k(n) - 3, Math.min(9 - k(n), 1)))
  return {
    r: Math.round(255 * f(0)),
    g: Math.round(255 * f(8)),
    b: Math.round(255 * f(4))
  }
}

const RGBToHex = (r: number, g: number, b: number) => {
  const toHex = (c: number) => {
    const hex = c.toString(16)
    return hex.length === 1 ? "0" + hex : hex
  }
  return "#" + toHex(r) + toHex(g) + toHex(b)
}

const hexToHSL = (hex: string) => {
  hex = hex.replace(/^#/, '')

  const r = parseInt(hex.slice(0, 2), 16) / 255
  const g = parseInt(hex.slice(2, 4), 16) / 255
  const b = parseInt(hex.slice(4, 6), 16) / 255

  const max = Math.max(r, g, b)
  const min = Math.min(r, g, b)
  let h = 0
  let s
  const l = (max + min) / 2

  if (max === min) {
    h = s = 0
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

  return {
    h: Math.round(h * 360),
    s: Math.round(s * 100),
    l: Math.round(l * 100)
  }
}

const getColorFormats = (color: Color): ColorFormats => {
  const rgb = HSLToRGB(color.h, color.s, color.l)
  return {
    hsl: `hsl(${color.h}, ${color.s}%, ${color.l}%)`,
    rgb: `rgb(${rgb.r}, ${rgb.g}, ${rgb.b})`,
    hex: RGBToHex(rgb.r, rgb.g, rgb.b)
  }
}

// Methods
const generatePalette = () => {
  const newPalette: Color[] = []
  let currentHue = baseColor.h

  for (let i = 0; i < 8; i++) {
    newPalette.push({
      h: currentHue % 360,
      s: baseColor.s,
      l: baseColor.l,
      shades: []
    })

    if (i === 1) currentHue += 90
    else if (i === 7) currentHue += 60
    else currentHue += 30
  }

  palette.value = newPalette
}

const generateShades = (color: Color, index: number) => {
  const newShades: Color[] = []
  const lightnesses = [80, 70, 60, 50, 40]

  // Base lightness variations
  lightnesses.forEach(l => {
    newShades.push({ h: color.h, s: color.s, l })
  })

  // Higher saturation
  lightnesses.forEach(l => {
    newShades.push({
      h: color.h,
      s: Math.min(100, color.s + 20),
      l
    })
  })

  // Lower saturation
  lightnesses.forEach(l => {
    newShades.push({
      h: color.h,
      s: Math.max(0, color.s - 20),
      l
    })
  })

  if (palette.value[index]) {
    palette.value[index].shades = newShades
    selectedColor.value = index
    selectedShade.value = null
  }
}

const handleHexInput = () => {
  if (/^#?[0-9A-Fa-f]{6}$/.test(hexInput.value)) {
    const hsl = hexToHSL(hexInput.value)
    Object.assign(baseColor, hsl)
  }
}

const selectShade = (shade: Color & { rowIndex: number; index: number }) => {
  selectedShade.value = shade
}

const copyColor = async (color: Color, format: keyof ColorFormats) => {
  const formats = getColorFormats(color)
  await navigator.clipboard.writeText(formats[format])
}

// Watch for changes in baseColor
watch(
  () => ({ ...baseColor }),
  () => generatePalette(),
  { immediate: true }
)
</script>

<template>
  <div class="p-6 max-w-4xl mx-auto">
    <!-- Base Color Input Card -->
    <div class="bg-white rounded-lg shadow-sm mb-6">
      <div class="p-6">
        <div class="space-y-4">
          <div class="flex gap-4 items-end">
            <div class="flex-1">
              <label class="block text-sm font-medium mb-2">Base Color (HEX)</label>
              <input v-model="hexInput" placeholder="#FF5733" class="w-full px-3 py-2 border rounded-lg"
                @input="handleHexInput" />
            </div>
            <div class="w-16 h-10 rounded-lg shadow-sm"
              :style="{ backgroundColor: `hsl(${baseColor.h}deg ${baseColor.s}% ${baseColor.l}%)` }" />
          </div>

          <h2 class="text-xl font-bold mb-4">HSL Controls</h2>
          <div class="grid grid-cols-3 gap-4">
            <div>
              <label class="block text-sm font-medium mb-2">Hue ({{ baseColor.h }}°)</label>
              <input type="range" v-model.number="baseColor.h" min="0" max="360" class="w-full" />
            </div>
            <div>
              <label class="block text-sm font-medium mb-2">Saturation ({{ baseColor.s }}%)</label>
              <input type="range" v-model.number="baseColor.s" min="0" max="100" class="w-full" />
            </div>
            <div>
              <label class="block text-sm font-medium mb-2">Lightness ({{ baseColor.l }}%)</label>
              <input type="range" v-model.number="baseColor.l" min="0" max="100" class="w-full" />
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Main Palette -->
    <div class="mb-8">
      <h2 class="text-xl font-bold mb-4">Main Palette</h2>
      <div class="flex gap-4 mb-4">
        <div v-for="(color, index) in palette" :key="index"
          class="w-16 h-16 cursor-pointer rounded-lg shadow-sm transition-transform hover:scale-105"
          :style="{ backgroundColor: `hsl(${color.h}deg ${color.s}% ${color.l}%)` }"
          @click="generateShades(color, index)" />
      </div>
    </div>

    <!-- Shades Section -->
    <template v-if="selectedColor !== null && palette[selectedColor]?.shades">
      <div>
        <h2 class="text-xl font-bold mb-4">Shades for Color {{ selectedColor + 1 }}</h2>
        <div class="space-y-6">
          <!-- Base lightness variations -->
          <div class="flex items-center gap-4">
            <div class="w-32 text-sm">Base lightness variations:</div>
            <div class="grid grid-cols-5 gap-2">
              <div v-for="(shade, index) in palette[selectedColor].shades?.slice(0, 5)" :key="index"
                class="w-10 h-10 cursor-pointer rounded-lg shadow-sm transition-transform hover:scale-105"
                :style="{ backgroundColor: `hsl(${shade.h}deg ${shade.s}% ${shade.l}%)` }"
                @click="selectShade({ ...shade, rowIndex: 0, index })" />
            </div>
          </div>

          <!-- Higher saturation -->
          <div class="flex items-center gap-4">
            <div class="w-32 text-sm">Higher saturation:</div>
            <div class="grid grid-cols-5 gap-2">
              <div v-for="(shade, index) in palette[selectedColor].shades?.slice(5, 10)" :key="index"
                class="w-10 h-10 cursor-pointer rounded-lg shadow-sm transition-transform hover:scale-105"
                :style="{ backgroundColor: `hsl(${shade.h}deg ${shade.s}% ${shade.l}%)` }"
                @click="selectShade({ ...shade, rowIndex: 1, index })" />
            </div>
          </div>

          <!-- Lower saturation -->
          <div class="flex items-center gap-4">
            <div class="w-32 text-sm">Lower saturation:</div>
            <div class="grid grid-cols-5 gap-2">
              <div v-for="(shade, index) in palette[selectedColor].shades?.slice(10, 15)" :key="index"
                class="w-10 h-10 cursor-pointer rounded-lg shadow-sm transition-transform hover:scale-105"
                :style="{ backgroundColor: `hsl(${shade.h}deg ${shade.s}% ${shade.l}%)` }"
                @click="selectShade({ ...shade, rowIndex: 2, index })" />
            </div>
          </div>
        </div>
      </div>

      <!-- Color Values Section -->
      <div class="mt-6 space-y-6">
        <!-- Selected Main Color -->
        <div v-if="selectedColor !== null && palette[selectedColor]">
          <h3 class="text-lg font-semibold mb-2">Selected Main Color</h3>
          <div class="grid grid-cols-4 gap-4">
            <template v-if="palette[selectedColor]">
              <input readonly :value="getColorFormats(palette[selectedColor]).hsl"
                class="px-3 py-2 border rounded-lg" />
              <input readonly :value="getColorFormats(palette[selectedColor]).rgb"
                class="px-3 py-2 border rounded-lg" />
              <input readonly :value="getColorFormats(palette[selectedColor]).hex"
                class="px-3 py-2 border rounded-lg" />
              <div class="flex gap-2">
                <button v-for="format in ['hsl', 'rgb', 'hex'] as const" :key="format"
                  class="flex-1 px-3 py-2 text-sm border rounded-lg hover:bg-gray-100"
                  @click="copyColor(palette[selectedColor], format)">
                  {{ format.toUpperCase() }}
                </button>
              </div>
            </template>
          </div>
        </div>

        <!-- Selected Shade -->
        <div v-if="selectedShade">
          <h3 class="text-lg font-semibold mb-2">Selected Shade</h3>
          <div class="grid grid-cols-4 gap-4">
            <input readonly :value="getColorFormats(selectedShade).hsl" class="px-3 py-2 border rounded-lg" />
            <input readonly :value="getColorFormats(selectedShade).rgb" class="px-3 py-2 border rounded-lg" />
            <input readonly :value="getColorFormats(selectedShade).hex" class="px-3 py-2 border rounded-lg" />
            <div class="flex gap-2">
              <button v-for="format in ['hsl', 'rgb', 'hex'] as const" :key="format"
                class="flex-1 px-3 py-2 text-sm border rounded-lg hover:bg-gray-100"
                @click="copyColor(selectedShade, format)">
                {{ format.toUpperCase() }}
              </button>
            </div>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
input[type="range"] {
  -webkit-appearance: none;
  width: 100%;
  height: 6px;
  background: #ddd;
  border-radius: 3px;
  outline: none;
}

input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  background: #666;
  border-radius: 50%;
  cursor: pointer;
}

input[type="range"]::-moz-range-thumb {
  width: 16px;
  height: 16px;
  background: #666;
  border-radius: 50%;
  cursor: pointer;
}
</style>
