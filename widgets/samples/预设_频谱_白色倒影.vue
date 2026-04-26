<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const container = ref(null)
const error = ref('')
let destroyFn = null

onMounted(() => {
  const canvas = document.createElement('canvas')
  canvas.style.width = '100%'
  canvas.style.height = '100%'
  canvas.style.display = 'block'
  container.value.appendChild(canvas)

  const SHOW_FPS = true

  const readBarValue = (bar) => {
    if (!bar) return 0
    // 优先读取原始值或显示值
    if (Number.isFinite(bar.rawValue)) return bar.rawValue
    if (Number.isFinite(bar.displayValue)) return bar.displayValue
    if (Array.isArray(bar.value) && Number.isFinite(bar.value[0])) return bar.value[0]
    return Number.isFinite(bar.value) ? bar.value : 0
  }

  destroyFn = window.mountAudioVisualizer({ error }, container.value, {
    useCustom: true,
    canvas,
    fftSize: 4096,
    fftOverlap: 2048,
    minFreq: 100,
    maxFreq: 16500,
    bands: 121,
    maxFPS: 60,
    attack: 200,
    decay: 100,
    sensitivity:32,

onCanvasDraw: (instance) => {
  const ctx = instance.canvasCtx
  const cv = instance.canvas
  const rw = cv.clientWidth || 300
  const rh = cv.clientHeight || 150
  const pr = 1

  const nextWidth = Math.max(1, Math.round(rw * pr))
  const nextHeight = Math.max(1, Math.round(rh * pr))

  if (cv.width !== nextWidth || cv.height !== nextHeight) {
    cv.width = nextWidth
    cv.height = nextHeight
  }

  const width = cv.width
  const height = cv.height

  const bars = instance.getBars?.()
  if (!bars || bars.length === 0) return

  // FPS 统计
  if (SHOW_FPS) {
    const now = performance.now()
    if (!instance._fpsState) {
      instance._fpsState = { lastTs: now, accumMs: 0, frames: 0, fps: 0 }
    }
    const fpsState = instance._fpsState
    const dt = now - fpsState.lastTs
    fpsState.lastTs = now
    if (dt > 0) {
      fpsState.accumMs += dt
      fpsState.frames += 1
      if (fpsState.accumMs >= 500) {
        fpsState.fps = (fpsState.frames * 1000) / fpsState.accumMs
        fpsState.accumMs = 0
        fpsState.frames = 0
      }
    }
  }

  // ===== 可调变量 =====
  const barColor = '255,255,255'
  const reflectionOpacity = 0.28
  const reflectionHeightRatio = 0.3
  const reflectionGap = 4 * pr
  const minDrawHeight = 0.5 * pr
  const barGap = 1

  const separatorThickness = Math.max(1.5 * pr, height * 0.005)
  const separatorOpacity = Math.min(1, reflectionOpacity + 0.25)

  const reflectionHeight = Math.max(0, height * reflectionHeightRatio)
  const mainHeight = Math.max(1, height - reflectionHeight - reflectionGap)
  const baselineY = mainHeight
  const slotWidth = width / bars.length
  const separatorY = baselineY + reflectionGap * 0.5

  let mirrorCanvas = instance._mirrorCanvas
  let mirrorCtx = instance._mirrorCtx

  if (!mirrorCanvas) {
    mirrorCanvas = document.createElement('canvas')
    mirrorCtx = mirrorCanvas.getContext('2d')
    instance._mirrorCanvas = mirrorCanvas
    instance._mirrorCtx = mirrorCtx
  }

  if (mirrorCanvas.width !== width || mirrorCanvas.height !== Math.max(1, Math.round(mainHeight))) {
    mirrorCanvas.width = width
    mirrorCanvas.height = Math.max(1, Math.round(mainHeight))
  }

  mirrorCtx.setTransform(1, 0, 0, 1, 0, 0)
  mirrorCtx.clearRect(0, 0, mirrorCanvas.width, mirrorCanvas.height)

  // ===== 1. 批量构建主柱体路径 (核心优化) =====
  mirrorCtx.setTransform(1, 0, 0, 1, 0, 0)
  mirrorCtx.clearRect(0, 0, mirrorCanvas.width, mirrorCanvas.height)
  mirrorCtx.fillStyle = `rgb(${barColor})`
  
  mirrorCtx.beginPath()
  for (let i = 0; i < bars.length; i++) {
    const val = readBarValue(bars[i])
    if (val <= 0) continue
    
    const barH = val * mainHeight
    if (barH < minDrawHeight) continue

    const slotLeft = i * slotWidth
    const x0 = (slotLeft + barGap * 0.5) | 0
    const x1 = ((i + 1) * slotWidth - barGap * 0.5) | 0
    const w = (x1 - x0) || 1
    const h = (barH + 0.5) | 0
    
    mirrorCtx.rect(x0, mainHeight - h, w, h)
  }
  mirrorCtx.fill() // 一次性提交 GPU 绘制

  // ===== 2. 主画布复合渲染 =====
  ctx.setTransform(1, 0, 0, 1, 0, 0)
  ctx.clearRect(0, 0, width, height)

  // 绘制主柱体
  ctx.drawImage(mirrorCanvas, 0, 0)

  // 绘制倒影 (镜像并拉伸压缩)
  ctx.save()
  ctx.setTransform(1, 0, 0, -1, 0, baselineY + reflectionGap + reflectionHeight)
  ctx.globalAlpha = reflectionOpacity
  // 将整个 mirrorCanvas 压缩绘制到 reflectionHeight 高度
  ctx.drawImage(mirrorCanvas, 0, 0, width, mainHeight, 0, 0, width, reflectionHeight)
  
  // 绘制倒影渐变遮罩 (缓存渐变对象)
  if (!instance._refGrd || instance._lastRefH !== reflectionHeight) {
    const g = ctx.createLinearGradient(0, 0, 0, reflectionHeight)
    g.addColorStop(0, `rgba(0,0,0,0)`)
    g.addColorStop(1, `rgba(0,0,0,${reflectionOpacity})`)
    instance._refGrd = g
    instance._lastRefH = reflectionHeight
  }
  
  ctx.globalCompositeOperation = 'destination-out'
  ctx.fillStyle = instance._refGrd
  ctx.fillRect(0, 0, width, reflectionHeight)
  ctx.restore()

  // 绘制分隔线 (贝塞尔曲线优化)
  ctx.save()
  ctx.fillStyle = `rgba(${barColor}, ${separatorOpacity})`
  ctx.beginPath()
  ctx.moveTo(0, separatorY)
  const cp1x = width * 0.25, cp2x = width * 0.75, midX = width * 0.5
  const offset = separatorThickness * 0.5
  ctx.quadraticCurveTo(cp1x, separatorY - offset * 1.1, midX, separatorY - offset)
  ctx.quadraticCurveTo(cp2x, separatorY - offset * 1.1, width, separatorY)
  ctx.quadraticCurveTo(cp2x, separatorY + offset * 1.1, midX, separatorY + offset)
  ctx.quadraticCurveTo(cp1x, separatorY + offset * 1.1, 0, separatorY)
  ctx.fill()
  ctx.restore()

  // 绘制 FPS
  if (SHOW_FPS && instance._fpsState) {
    ctx.save()
    ctx.font = '14px Consolas, monospace'
    ctx.fillStyle = 'rgba(50, 248, 0, 0.95)'
    ctx.textAlign = 'right'
    ctx.textBaseline = 'top'
    ctx.fillText(`FPS ${instance._fpsState.fps.toFixed(1)}`, width - 10, 10)
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
