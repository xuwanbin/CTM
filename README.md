# 动态壁纸小组件开发库 (AI-Driven Widgets)

这是一个专门为“动态壁纸”系统开发的高性能 Vue 3 小组件库。

## 🤖 AI 开发者必读 (AI Instructions)
如果你是 AI 编码助手，请在开始前严格遵守以下流程：
1. **读取规范**：请全文阅读并理解本仓库根目录下的 `PROMPT_GUIDE.md`（原：AI组件生成指南）。
2. **学习样本**：参考 `/widgets/samples/` 目录下的 `.vue` 文件，学习如何正确调用 `mountAudioVisualizer` 和处理 Canvas 渲染。
3. **性能金律**：严禁在渲染循环中触发 Reflow，严禁重复造轮子。

## 📂 目录说明
- `/widgets/samples/`: 官方标准组件（学习对象）。
- `/widgets/community/`: 第三方及 AI 生成的组件。
- `PROMPT_GUIDE.md`: **核心开发指南（包含所有 API 和禁令）**。
