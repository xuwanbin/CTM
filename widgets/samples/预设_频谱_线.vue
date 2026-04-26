<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const container = ref(null)
const error = ref('')
let destroyFn = null


const clamp = (v, min, max) => (v < min ? min : v > max ? max : v)

const mapLinear = (v, inMin, inMax, outMin, outMax) => {
  if (inMax === inMin) return outMin
  return outMin + (outMax - outMin) * ((v - inMin) / (inMax - inMin))
}

// 更接近 Lua 的“激励读取”：优先读 rawValue，再退回 display/value
const readBandExcitation = (bar) => {
  if (!bar) return 0

  if (Number.isFinite(bar.rawValue)) return Math.max(0, bar.rawValue)
  if (Number.isFinite(bar.displayValue)) return Math.max(0, bar.displayValue)

  if (Array.isArray(bar.value) && Number.isFinite(bar.value[0])) {
    return Math.max(0, bar.value[0])
  }

  if (Number.isFinite(bar.value)) return Math.max(0, bar.value)
  return 0
}

const buildWaveState = (pointCount, bandCount, height) => {
  const map1 = new Float32Array(pointCount)
  const map2 = new Float32Array(pointCount)
  const parentBand = new Uint16Array(pointCount)

  for (let i = 0; i < pointCount; i++) {
    parentBand[i] = Math.floor(
      mapLinear(i, 0, Math.max(1, pointCount - 1), 0, Math.max(0, bandCount - 1))
    )
  }

  // 对齐 Lua：中间给一段初始扰动
  const mid = Math.floor(pointCount / 2)
  const start = Math.max(0, mid - 7)
  const end = Math.min(pointCount - 1, mid + 7)
  for (let i = start; i <= end; i++) {
    map1[i] = -3
  }

  return {
    map1,
    map2,
    parentBand,
    useMap1: true,
    pointCount,
    lastHeight: height
  }
}

const ensureWaveState = (instance, pointCount, bandCount, height) => {
  const state = instance._waveState
  if (
    !state ||
    state.pointCount !== pointCount ||
    state.parentBand.length !== pointCount ||
    state.lastHeight !== height
  ) {
    instance._waveState = buildWaveState(pointCount, bandCount, height)
  }
  return instance._waveState
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
    fftSize: 8192,
    fftOverlap: 4096,
    minFreq: 15,
    maxFreq: 10000,
    bands: 256,

    maxFPS: 60,

    // FFT 层尽量少做额外“显示平滑”，把顺滑交给 wave 本身
    attack: 0,
    decay: 0, 
    outputSmoothingMs: 1,


    onCanvasDraw: (instance) => {
      const ctx = instance.canvasCtx
      const cv = instance.canvas
      const rw = cv.clientWidth || 300
      const rh = cv.clientHeight || 150
      const showFps = SHOW_FPS
      const nextWidth = Math.max(1, rw )
      const nextHeight = Math.max(1, rh )

      if (cv.width !== nextWidth || cv.height !== nextHeight) {
        cv.width = nextWidth
        cv.height = nextHeight
      }

      const width = cv.width
      const height = cv.height

      ctx.setTransform(1, 0, 0, 1, 0, 0)
      ctx.clearRect(0, 0, width, height)

      const bars = instance.getBars?.()
let fpsState = null

if (showFps) {
  const now = performance.now()

  if (!instance._fpsState) {
    instance._fpsState = {
      lastTs: now,
      accumMs: 0,
      frames: 0,
      fps: 0
    }
  }

  fpsState = instance._fpsState
  const dt = now - fpsState.lastTs
  fpsState.lastTs = now

  if (dt > 0 && Number.isFinite(dt)) {
    fpsState.accumMs += dt
    fpsState.frames += 1

    if (fpsState.accumMs >= 500) {
      fpsState.fps = (fpsState.frames * 1000) / fpsState.accumMs
      fpsState.accumMs = 0
      fpsState.frames = 0
    }
  }
}
      if (!bars || bars.length === 0) return

      // ===== 可调参数：对应 Lua 的核心参数 =====
      const pointCount = 256
      const expScaleFactor = 1.8
      const waveStiffness = Math.max(1, 1.08) // Lua 里 <=1 会被钳成 1
      const waveSpread = 4
      const waveScale = 2.2
const lineThickness = 3 

      const lineWidth = 2
      const fillWave = false
      const fillColor = 'rgba(255,255,255,0.10)'
      // =====================================

      const state = ensureWaveState(instance, pointCount, bars.length, height)
      const source = state.useMap1 ? state.map1 : state.map2
      const dest = state.useMap1 ? state.map2 : state.map1

      // ---- 核心更新：直接翻译 Lua 的双缓冲波形逻辑 ----
      for (let i = 0; i < pointCount; i++) {
        const bandIndex = state.parentBand[i]
        const excitation = Math.pow(readBandExcitation(bars[bandIndex]), expScaleFactor)

        const prevDest = dest[i]
        const left = source[i > 0 ? i - 1 : 0]
        const right = source[i < pointCount - 1 ? i + 1 : pointCount - 1]

        let next = prevDest + excitation
        next = (left + right) / waveStiffness - next
        next = next - next / waveSpread

        dest[i] = next
      }

      state.useMap1 = !state.useMap1

      const wave = dest
      const centerY = height * 0.6
      const halfHeight = height * 0.5
      const stepX = width / pointCount

      const valueToY = (v) => centerY + (halfHeight * v) / waveScale

      let yNext = valueToY(wave[0])

      ctx.beginPath()
      if (fillWave) {
        ctx.moveTo(0, height)
        ctx.lineTo(0, yNext)
      } else {
        ctx.moveTo(0, yNext)
      }

      for (let i = 0; i < pointCount; i++) {
        const xEnd = mapLinear(i, 0, Math.max(1, pointCount - 1), 0, width)
        const yVal = yNext
        const nextIndex = i + 1 < pointCount ? i + 1 : i
        yNext = valueToY(wave[nextIndex])
        const yEnd = (yVal + yNext) * 0.5

        ctx.quadraticCurveTo(
          xEnd - stepX * 0.5,
          yVal,
          xEnd,
          yEnd
        )
      }

      if (fillWave) {
        ctx.lineTo(width, height)
        ctx.closePath()
        ctx.fillStyle = fillColor
        ctx.fill()
      }

      // ---- 绘图优化：缓存渐变对象 ----
      if (!instance._cachedGradient || instance._lastGWidth !== width) {
        const g = ctx.createLinearGradient(0, 0, width, 0)
        g.addColorStop(0.0, '#E818BD')
        g.addColorStop(0.5, '#619DE1')
        g.addColorStop(1.0, '#12EDF7')
        instance._cachedGradient = g
        instance._lastGWidth = width
      }

      ctx.strokeStyle = instance._cachedGradient
      ctx.lineWidth = lineThickness
      ctx.lineJoin = 'round'
      ctx.lineCap = 'round'
      ctx.stroke()
if (showFps && fpsState) {
  ctx.save()

  const fpsText = `FPS ${fpsState.fps.toFixed(1)}`
  const padX = 10
  const padY = 8
  const fontSize = 14

  ctx.font = `${fontSize}px Consolas, monospace`
  ctx.textAlign = 'right'
  ctx.textBaseline = 'top'

  const metrics = ctx.measureText(fpsText)
  const boxW = metrics.width + 16 
  const boxH = fontSize + 10
  const boxX = width - padX - boxW
  const boxY = padY

  ctx.fillStyle = 'rgba(0,0,0,0.35)'
  ctx.fillRect(boxX, boxY, boxW, boxH)

  ctx.fillStyle = 'rgba(50, 248, 0, 0.95)'
  ctx.fillText(fpsText, width - padX - 8, boxY + 5 )

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

.container {
  width: 100%;
  height: 100%;
  min-height: 100px;
  overflow: hidden;
}
.root,
.container,
.container canvas {
  pointer-events: none;
}
.error {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  color: red;
  pointer-events: none;
  font-size: 1.8vmin;
}
</style>
