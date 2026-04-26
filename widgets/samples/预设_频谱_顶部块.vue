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

    const SHOW_FPS = false

    /**
     * 顶峰横条状态
     */
    let peakValues = []
    let peakHoldUntil = []
    let lastPeakTime = 0

    const readBarValue = (bar) => {
        if (!bar) return 0

        if (Number.isFinite(bar.rawValue)) return bar.rawValue
        if (Number.isFinite(bar.displayValue)) return bar.displayValue
        if (Array.isArray(bar.value) && Number.isFinite(bar.value[0])) return bar.value[0]

        return Number.isFinite(bar.value) ? bar.value : 0
    }

    const clamp01 = (v) => {
        if (!Number.isFinite(v)) return 0
        return Math.max(0, Math.min(1, v))
    }

    const ensurePeakArrays = (count) => {
        if (peakValues.length === count && peakHoldUntil.length === count) return

        peakValues = new Array(count).fill(0)
        peakHoldUntil = new Array(count).fill(0)
        lastPeakTime = performance.now()
    }

    destroyFn = window.mountAudioVisualizer({ error }, container.value, {
        useCustom: true,
        canvas,

        fftSize: 4096,
        fftOverlap: 2048,
        minFreq: 32,
        maxFreq: 16500,
        bands: 48,
        maxFPS: 60,
        attack: 200,
        decay: 100,
        sensitivity: 35,

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

            ensurePeakArrays(bars.length)

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

            // 柱子最小绘制高度
            const minDrawHeight = 0.5 * pr

            // 柱子之间的间距
            const barGap = 1

            // 主柱体高度比例，越小顶部留白越多
            const mainHeightRatio = 0.9

            // 顶峰横条停留时间，单位 ms
            const PEAK_HOLD_MS = 300

            // 顶峰横条下降速度，单位：归一化高度 / 秒
            // 越大下降越快
            const PEAK_FALL_SPEED = 0.75

            // 顶峰横条高度
            const PEAK_BAR_HEIGHT = Math.max(2 * pr, 3 * pr)

            // 顶峰横条左右加宽
            const PEAK_BAR_EXTRA_WIDTH = 1.4 * pr

            // 顶峰横条透明度
            const PEAK_BAR_ALPHA = 0.95

            // 顶峰横条发光强度
            const PEAK_GLOW_BLUR = 8 * pr

            // 主柱体高度压缩曲线，低于 1 会增强弱音
            const VALUE_POWER = 0.72

            const now = performance.now()
            const dt = lastPeakTime > 0 ? Math.min(0.05, (now - lastPeakTime) / 1000) : 0.016
            lastPeakTime = now

            const mainHeight = Math.max(1, height * mainHeightRatio)
            const baselineY = height
            const slotWidth = width / bars.length

            let mainCanvas = instance._mainCanvas
            let mainCtx = instance._mainCtx

            if (!mainCanvas) {
                mainCanvas = document.createElement('canvas')
                mainCtx = mainCanvas.getContext('2d')
                instance._mainCanvas = mainCanvas
                instance._mainCtx = mainCtx
            }

            if (mainCanvas.width !== width || mainCanvas.height !== height) {
                mainCanvas.width = width
                mainCanvas.height = height
            }

            mainCtx.setTransform(1, 0, 0, 1, 0, 0)
            mainCtx.clearRect(0, 0, width, height)

            // ===== 离屏 canvas 绘制主柱体 =====
            mainCtx.fillStyle = `rgba(${barColor}, 1)`

            for (let i = 0; i < bars.length; i++) {
                const raw = clamp01(readBarValue(bars[i]))
                const normalized = Math.pow(raw, VALUE_POWER)

                const barH = normalized * mainHeight
                if (barH < minDrawHeight) continue

                const slotLeft = i * slotWidth
                const slotRight = (i + 1) * slotWidth

                const x0 = Math.ceil(slotLeft + barGap * 0.5)
                const x1 = Math.floor(slotRight - barGap * 0.5)

                const w = Math.max(1, x1 - x0)
                const h = Math.round(barH)
                const y = Math.round(baselineY - h)

                mainCtx.fillRect(x0, y, w, h)
            }

            // ===== 主画布清空 =====
            ctx.setTransform(1, 0, 0, 1, 0, 0)
            ctx.clearRect(0, 0, width, height)

            // ===== 主柱体 =====
            ctx.drawImage(mainCanvas, 0, 0)

            // ===== 顶部可下落横条 =====
            ctx.save()
            ctx.shadowColor = `rgba(${barColor}, 0.85)`
            ctx.shadowBlur = PEAK_GLOW_BLUR

            for (let i = 0; i < bars.length; i++) {
                const raw = clamp01(readBarValue(bars[i]))
                const normalized = Math.pow(raw, VALUE_POWER)

                /**
                 * 当前柱体高度对应的顶部位置。
                 * normalized 越大，currentPeak 越高。
                 */
                const currentPeak = normalized

                /**
                 * 顶峰横条逻辑：
                 * 1. 当前音量超过历史顶峰：横条立即上升到新顶部；
                 * 2. 当前音量低于历史顶峰：横条先停留；
                 * 3. 停留时间结束后：横条按 PEAK_FALL_SPEED 下落。
                 */
                if (currentPeak >= peakValues[i]) {
                    peakValues[i] = currentPeak
                    peakHoldUntil[i] = now + PEAK_HOLD_MS
                } else if (now > peakHoldUntil[i]) {
                    peakValues[i] = Math.max(
                        currentPeak,
                        peakValues[i] - PEAK_FALL_SPEED * dt
                    )
                }

                const peak = clamp01(peakValues[i])
                const peakY = Math.round(baselineY - peak * mainHeight)

                const slotLeft = i * slotWidth
                const slotRight = (i + 1) * slotWidth

                const x0 = Math.ceil(slotLeft + barGap * 0.5)
                const x1 = Math.floor(slotRight - barGap * 0.5)

                const w = Math.max(1, x1 - x0)

                /**
                 * 横条只画在顶部，不画整段区域。
                 */
                ctx.fillStyle = `rgba(${barColor}, ${PEAK_BAR_ALPHA})`
                ctx.fillRect(
                    x0,
                    peakY - PEAK_BAR_HEIGHT * 0.5,
                    w,
                    PEAK_BAR_HEIGHT
                )
            }

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
