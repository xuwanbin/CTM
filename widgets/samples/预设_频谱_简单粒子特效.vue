<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const rootRef = ref(null)
const canvasRef = ref(null)
const error = ref('')
let visualizer = null
let resizeObserver = null
const particles = []

const updateSize = () => {
  const canvas = canvasRef.value
  if (canvas && rootRef.value) {
    const rect = rootRef.value.getBoundingClientRect()
    const dpr = window.devicePixelRatio || 1
    canvas.width = rect.width * dpr
    canvas.height = rect.height * dpr
  }
}

onMounted(() => {
  updateSize()
  visualizer = mountAudioVisualizer({ error }, rootRef.value, {
    useCustom: true,
    canvas: canvasRef.value,
    onCanvasDraw: (instance) => {
      const { canvasCtx: ctx, canvas } = instance
      const bars = instance.getBars()
      const dpr = window.devicePixelRatio || 1
      const lw = canvas.width / dpr
      const lh = canvas.height / dpr

      ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
      ctx.clearRect(0, 0, lw, lh)

      const barWidth = lw / bars.length
      
      // 1. 绘制主体频谱（发光线条）
      ctx.beginPath()
      ctx.lineCap = 'round'
      ctx.lineWidth = barWidth * 0.5
      bars.forEach((bar, i) => {
        const x = i * barWidth + barWidth / 2
        const h = bar.displayValue * lh * 0.8
        ctx.moveTo(x, lh)
        ctx.lineTo(x, lh - h)
        
        // 2. 高能检测：生成粒子
        if (bar.displayValue > 0.5 && particles.length < 300) {
          particles.push({
            x: x,
            y: lh - h,
            vx: (Math.random() - 0.5) * 4,
            vy: -Math.random() * bar.displayValue * 12,
            size: Math.random() * 3 + 1,
            color: `hsla(${180 + bar.displayValue * 60}, 100%, 70%,`,
            life: 1.0
          })
        }
      })
      ctx.strokeStyle = '#00f2ff'
      ctx.shadowBlur = 15
      ctx.shadowColor = '#00f2ff'
      ctx.stroke()

      // 3. 粒子物理模拟与批量绘制
      ctx.shadowBlur = 0
      for (let i = particles.length - 1; i >= 0; i--) {
        const p = particles[i]
        p.x += p.vx
        p.y += p.vy
        p.vy += 0.2 // 重力
        p.life -= 0.015
        
        if (p.life <= 0 || p.y > lh) {
          particles.splice(i, 1)
          continue
        }

        ctx.beginPath()
        ctx.fillStyle = `${p.color}${p.life})`
        ctx.arc(p.x, p.y, p.size * p.life, 0, Math.PI * 2)
        ctx.fill()
      }
    }
  })
  resizeObserver = new ResizeObserver(updateSize)
  resizeObserver.observe(rootRef.value)
})

onUnmounted(() => {
  if (visualizer) visualizer()
  if (resizeObserver) resizeObserver.disconnect()
})
</script>

<template>
  <div class="fluid-root" ref="rootRef">
    <div v-if="error" class="error-mask">{{ error }}</div>
    <canvas ref="canvasRef"></canvas>
  </div>
</template>

<style scoped>
.fluid-root {
  width: 100%;
  height: 100%;
  overflow: hidden;
  position: relative;
  background: transparent;
}
canvas {
  width: 100%;
  height: 100%;
  display: block;
  filter: drop-shadow(0 0 5px rgba(0, 242, 255, 0.3));
}
.error-mask {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  color: #ff4444;
  font-family: "Orbitron", sans-serif;
  background: rgba(0,0,0,0.2);
  pointer-events: none;
}
</style>
