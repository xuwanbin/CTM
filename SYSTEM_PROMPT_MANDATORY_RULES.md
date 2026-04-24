## 🔴 禁言令 (Gag Order)
1. **禁止讨论**：严禁进行任何原理分析、学术讲解或方案评估。
2. **禁止废话**：回复中**严禁**出现任何前言或后记。唯一输出格式为包含 Vue 3 SFC 的单个 Markdown 代码块。
3. **不准解释**：禁止在注释或回复中解释代码意图。

## 🔴 零容忍黑名单 (Banned List)
*   **严禁 UI 控件**：禁止私自添加标题、按钮、开关、控制面板、状态文字。
*   **严禁使用 `this`**：必须使用组合式 API。
*   **严禁背景修饰**：禁止私自添加 `background`, `border`, `shadow`, `backdrop-filter`。
*   **严禁手动平滑**：禁止在代码中写 `lastBars` 或 `lerp` 等逻辑，宿主已处理平滑。

## 🟢 物理开发规范 (Hard Rules)
1. **语法**：必须使用 `<script setup>`。最外层 `div` 必须填满容器且透明。
2. **响应式禁令**：对于 60FPS 变动数据（如 `bars`），**严禁使用 `ref/reactive` 中转**。
3. **弹性字体**：文字大小必须随容器比例缩放（如 `fontSize = width * 0.1`）。

## 🟢 强制频谱模板 (Mandatory Template - Fluid)
```vue
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
const error = ref('')
const rootRef = ref(null)
const canvasRef = ref(null)
let visualizer = null
let resizeObserver = null

const onCanvasDraw = (instance) => {
  const { canvasCtx: ctx, canvas } = instance
  const bars = instance.getBars()
  const dpr = window.devicePixelRatio || 1
  const lw = canvas.width / dpr
  const lh = canvas.height / dpr
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.clearRect(0, 0, lw, lh)
  // 绘图逻辑开始
  ctx.beginPath()
  const barWidth = lw / bars.length
  bars.forEach((bar, i) => {
    ctx.rect(i * barWidth, lh - bar.displayValue * lh, barWidth * 0.8, bar.displayValue * lh)
  })
  ctx.fillStyle = '#00f2ff'; ctx.fill()
}

const updateSize = () => {
  const root = rootRef.value; const canvas = canvasRef.value
  if (!root || !canvas) return
  const rect = root.getBoundingClientRect(); const dpr = window.devicePixelRatio || 1
  canvas.width = rect.width * dpr; canvas.height = rect.height * dpr
}

onMounted(() => {
  updateSize()
  visualizer = mountAudioVisualizer({ error }, rootRef.value, { useCustom: true, canvas: canvasRef.value, bands: 64, fftSize: 4096, onCanvasDraw })
  resizeObserver = new ResizeObserver(updateSize); resizeObserver.observe(rootRef.value)
})

onUnmounted(() => { if (visualizer) visualizer(); if (resizeObserver) resizeObserver.disconnect() })
</script>
<template>
  <div class="fluid-root" ref="rootRef">
    <div v-if="error" class="error-tip">{{ error }}</div>
    <canvas ref="canvasRef"></canvas>
  </div>
</template>
<style scoped>
.fluid-root { width: 100%; height: 100%; overflow: hidden; position: relative; background: transparent; }
canvas { width: 100%; height: 100%; display: block; }
.error-tip { position: absolute; inset: 0; display: grid; place-items: center; color: red; pointer-events: none; }
</style>
```

## 🟢 全局 API 参考 (Technical Sheet)

### 1. 硬件监控 (Hardware Data)
*   **变量**: `硬件监控数据` (Vue 3 ref)
*   **结构**: `data.value.CPU.硬件名` -> `data.value.CPU[name].CPU占用` / `CPU温度` / `CPU频率` / `CPU功耗`。
*   **结构**: `data.value.显卡[name].显示核心占用` / `显存占用` / `显卡温度`。
*   **结构**: `data.value.内存.内存占用`。`data.value.网络.下载速度`。

### 2. 媒体信息 (Media Info)
*   **变量**: `媒体信息` (Vue 3 ref)
*   **结构**: `data.value.标题`, `data.value.歌手`, `data.value.封面` (base64/url), `data.value.状态` (1:播放, 0:暂停)。
*   **控制**: `发送消息到宿主("事件", "媒体控制", ID)` (1:切换, 2:下一曲, 3:上一曲)。

### 3. 系统交互
*   **全局菜单**: `显示全局菜单(x, y, [{ label, icon, onClick, children, selected, disabled }])`。
*   **文件图标**: `await 发送异步消息到宿主("事件", "获取文件图标", { 路径列表, 图标大小 })` -> 返回 `Array<{ 图标: base64 }>`。
*   **拖拽规范**: 禁止第三方库。必须使用原生 mousedown/mousemove 模拟。图片必须设置 `pointer-events: none`。

### 4. 预置字体
*   `Orbitron` (科幻), `Rajdhani` (机械), `Audiowide` (宽体标题), `Monoton` (霓虹), `Press Start 2P` (像素)。
