<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const container = ref(null)
const error = ref('')
let destroyFn = null

const LETTERS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split('')

const clamp = (v, min, max) => (v < min ? min : v > max ? max : v)
const lerp = (a, b, t) => a + (b - a) * t

const readBandValue = (bar) => {
  if (!bar) return 0
  if (Number.isFinite(bar.rawValue)) return Math.max(0, bar.rawValue)
  if (Number.isFinite(bar.displayValue)) return Math.max(0, bar.displayValue)
  if (Array.isArray(bar.value) && Number.isFinite(bar.value[0])) return Math.max(0, bar.value[0])
  if (Number.isFinite(bar.value)) return Math.max(0, bar.value)
  return 0
}

const createState = (count, width, height) => ({
  width,
  height,
  count,
  values: new Float32Array(count),
  fps: 0,
  frames: 0,
  accumMs: 0,
  lastTs: 0
})

const ensureState = (instance, count, width, height) => {
  const state = instance._alphabet26State
  if (!state || state.count !== count || state.width !== width || state.height !== height) {
    instance._alphabet26State = createState(count, width, height)
  }
  return instance._alphabet26State
}

onMounted(() => {
  const canvas = document.createElement('canvas')
  canvas.style.width = '100%'
  canvas.style.height = '100%'
  canvas.style.display = 'block'
  container.value.appendChild(canvas)

  const SHOW_FPS = false

  destroyFn = window.mountAudioVisualizer({ error }, container.value, {
    useCustom: true,
    canvas,

    // 核心：固定 26 个频段
    fftSize: 2048,
    fftOverlap: 1024,
    minFreq: 32,
    maxFreq: 16000,
    bands: 26,

    maxFPS: 60,
    attack: 40,
    decay: 40,
    outputSmoothingMs: 18,
    sensitivity: 28,

    onCanvasDraw: (instance) => {
      const ctx = instance.canvasCtx
      const cv = instance.canvas
      const rw = cv.clientWidth || 300
      const rh = cv.clientHeight || 150
      const width = Math.max(1, rw | 0)
      const height = Math.max(1, rh | 0)

      if (cv.width !== width || cv.height !== height) {
        cv.width = width
        cv.height = height
      }

      const bars = instance.getBars?.()
      if (!bars || bars.length === 0) {
        ctx.clearRect(0, 0, width, height)
        return
      }

      const count = Math.min(26, bars.length)
      const state = ensureState(instance, count, width, height)

      // FPS
      if (SHOW_FPS) {
        const now = performance.now()
        if (!state.lastTs) state.lastTs = now
        const dt = now - state.lastTs
        state.lastTs = now
        if (dt > 0 && Number.isFinite(dt)) {
          state.accumMs += dt
          state.frames += 1
          if (state.accumMs >= 500) {
            state.fps = (state.frames * 1000) / state.accumMs
            state.accumMs = 0
            state.frames = 0
          }
        }
      }

      // 清屏
      ctx.setTransform(1, 0, 0, 1, 0, 0)
      ctx.clearRect(0, 0, width, height)

      // 背景可选：非常轻的暗层，不要可删
      // ctx.fillStyle = 'rgba(0,0,0,0.08)'
      // ctx.fillRect(0, 0, width, height)

      // ========== 平滑 26 个频段 ==========
      for (let i = 0; i < count; i++) {
        const raw = readBandValue(bars[i])
        const shaped = Math.pow(clamp(raw, 0, 1.5), 0.82)

        const prev = state.values[i]
        const next = shaped > prev
          ? lerp(prev, shaped, 0.28)
          : lerp(prev, shaped, 0.10)

        state.values[i] = next
      }

      // ========== 布局 ==========
      const paddingX = Math.max(16, width * 0.03)
      const usableW = Math.max(1, width - paddingX * 2)
      const slotW = usableW / count
      const centerY = height * 0.55

      // 字体大小：自动适应
      const fontSize = Math.min(height * 0.4, slotW * 1)
      const fontFamily = 'Consolas, Arial, sans-serif'

      ctx.textAlign = 'center'
      ctx.textBaseline = 'middle'

      for (let i = 0; i < count; i++) {
        const energy = clamp(state.values[i], 0, 1.2)
        const x = paddingX + slotW * (i + 0.5)

        // 亮度增强：核心逻辑
        const alpha = 0.22 + energy * 0.78

        // 附加一点轻微动态：不喜欢可删
        const scale = 1 + energy * 0.3
        const lift = energy * fontSize * 0.5

        // glow 强度
        const glow = 2 + energy * 18

        ctx.save()
        ctx.translate(x, centerY - lift)
        ctx.scale(scale, scale)

        ctx.font = `700 ${fontSize}px ${fontFamily}`
        ctx.fillStyle = `rgba(255,255,255,${alpha})`
        ctx.shadowBlur = glow
        ctx.shadowColor = `rgba(255,255,255,${0.15 + energy * 0.55})`

        ctx.fillText(LETTERS[i], 0, 0)
        ctx.restore()
      }



      if (SHOW_FPS) {
        ctx.save()
        ctx.font = '14px Consolas, monospace'
        ctx.fillStyle = 'rgba(120,255,160,0.95)'
        ctx.textAlign = 'right'
        ctx.textBaseline = 'top'
        ctx.fillText(`FPS ${state.fps.toFixed(1)}`, width - 10, 10)
        ctx.restore()
      }
    }
  })
})

onUnmounted(() => {
  if (destroyFn) destroyFn()
})
</script>

<template>
  <div class="root">
    <div ref="container" class="container"></div>
    <div v-if="error" class="error">
      {{ error }}
    </div>
  </div>
</template>

<style scoped>
.root {
  width: 100%;
  height: 100%;
  position: relative;
}
.root,
.container,
.container canvas {
  pointer-events: none;
}
.container {
  width: 100%;
  height: 100%;
  min-height: 100px;
  overflow: hidden;
}

.error {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  color: red;
  font-size: 1.8vmin;
}
</style>
