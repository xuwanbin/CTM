<template>
    <div ref="container" class="spectrum-host">
        <canvas ref="canvasEl" class="spectrum-canvas"></canvas>
    </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const container = ref(null)
const canvasEl = ref(null)
const error = ref('')
let destroy = null
let pulse = 0
let tick = 0
const getValue = (bar) => {
    if (!bar) return 0
    if (Number.isFinite(bar.displayValue)) return bar.displayValue
    if (Number.isFinite(bar.value)) return bar.value
    if (Number.isFinite(bar.rawValue)) return bar.rawValue
    if (Array.isArray(bar.value) && Number.isFinite(bar.value[0])) return bar.value[0]
    return 0
}
const drawSpectrum = (instance) => {
    const { canvasCtx: ctx, canvas } = instance
    const bars = instance.getBars ? instance.getBars() : []
    const dpr = window.devicePixelRatio || 1
    const rect = canvas.getBoundingClientRect()
    const W = rect.width
    const H = rect.height
    if (canvas.width !== Math.max(1, W * dpr)) canvas.width = Math.max(1, W * dpr)
    if (canvas.height !== Math.max(1, H * dpr)) canvas.height = Math.max(1, H * dpr)
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
    tick += 0.016

    const n = bars.length || 1
    let low = 0, mid = 0, high = 0
    for (let i = 0; i < n; i++) {
        const v = Math.min(1, Math.max(0, getValue(bars[i])))
        if (i < n * 0.18) low += v
        else if (i < n * 0.62) mid += v
        else high += v
    }
    low /= Math.max(1, Math.floor(n * 0.18))
    mid /= Math.max(1, Math.floor(n * 0.44))
    high /= Math.max(1, Math.floor(n * 0.38))
    const totalEnergy = low + mid + high
    pulse = Math.max(pulse * 0.88, low)



    ctx.clearRect(0, 0, W, H)

    const currentScale = 0.8 + pulse * 0.2
    const contentWidth = W * currentScale
    const startX = (W - contentWidth) / 2
    const baseY = H * 0.85
    const barW = contentWidth / n

    // ── 频谱柱子（白色）──
    ctx.beginPath()
    for (let i = 0; i < n; i++) {
        const v = Math.min(1, Math.max(0, getValue(bars[i])))
        const h = 4 + Math.pow(v, 0.72) * (H * 0.6)
        const x = startX + i * barW + barW * 0.15
        const w = Math.max(1, barW * 0.7)
        ctx.rect(x, baseY - h, w, h)
    }
    const grad = ctx.createLinearGradient(0, baseY - H * 0.55, 0, baseY)
    grad.addColorStop(0, `rgba(255,255,255,${0.95 + high * 0.05})`)
    grad.addColorStop(0.4, `rgba(255,255,255,${0.75 + mid * 0.15})`)
    grad.addColorStop(1, `rgba(255,255,255,${0.45 + low * 0.15})`)
    ctx.fillStyle = grad
    ctx.fill()

    // ── 底部粒子（白色）──
    ctx.beginPath()
    for (let i = 0; i < n; i += 2) {
        const v = Math.min(1, Math.max(0, getValue(bars[i])))
        const x = startX + i * barW + barW * 0.5
        const y = baseY + 12 + Math.sin(tick * 2 + i * 0.16) * 4
        ctx.moveTo(x + v * 6, y)
        ctx.arc(x, y, 1.2 + v * 3, 0, Math.PI * 2)
    }
    ctx.fillStyle = `rgba(255,255,255,${0.15 + high * 0.45})`
    ctx.fill()
}
onMounted(() => {
    const api = window.mountAudioVisualizer
    if (!api || !container.value || !canvasEl.value) return
    destroy = api({ error }, container.value, {
        canvas: canvasEl.value,
        bands: 96,
        fftSize: 8192,
        fftOverlap: 4096,
        minFreq: 48,
        maxFreq: 16000,
        sensitivity: 32,
        attack: 200,
        decay: 100,
        outputSmoothingMs: 60,
        useCustom: true,
        onCanvasDraw: drawSpectrum
    })
})
onUnmounted(() => {
    if (typeof destroy === 'function') destroy()
    destroy = null
})
</script>
<style scoped>
.spectrum-host {
    width: 100%;
    height: 100%;
    overflow: hidden;
    pointer-events: none;
}

.spectrum-canvas {
    width: 100%;
    height: 100%;
    display: block;
}
</style>
