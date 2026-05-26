# voice-debrief

面试复盘 skill for Claude Code — 上传录音或转录文本，自动转录、整理问答、生成总评，保存到本地或飞书文档。

An interview debrief skill for Claude Code — upload a recording or transcript, get automatic transcription, Q&A summary, and evaluation, saved locally or to Feishu docs.

---

## 安装 / Install

```bash
npx skills add realwooolf/voice-debrief
```

---

## 使用方法 / Usage

把录音文件路径发给 Claude，说「帮我整理这场面试」：

Paste the recording file path and say "帮我整理这场面试" (help me debrief this interview):

```
帮我整理这场面试 /Users/xxx/Desktop/面试录音.mp3
```

已有 PDF 转录文本也可以直接发过来，skill 会跳过转录阶段。

You can also pass an existing PDF transcript — the skill will skip transcription automatically.

---

## 前置依赖 / Prerequisites

```bash
# Apple Silicon Mac（推荐 / Recommended）
pip install mlx-whisper
brew install ffmpeg

# Other platforms
pip install openai-whisper
brew install ffmpeg      # macOS
# sudo apt install ffmpeg  # Ubuntu
```

---

## 功能 / Features

- **自动转录 / Auto transcription**：本地运行 Whisper，支持 .wav / .m4a / .mp3 / .mp4，不上传任何数据
- **发言人识别 / Speaker detection**：根据对话内容自动区分面试官和候选人，完整对话中分别标注
- **多文件合并 / Multi-file merge**：多段录音自动按时间排序合并
- **三份输出 / Three outputs**：完整对话、问答整理、面试总评，总评同步输出到对话框
- **多维表格归档 / Bitable archive**：每场面试自动写入飞书多维表格，记录公司、岗位、轮次和三个文档链接，首次使用自动建表
- **多轮命名 / Multi-round naming**：同公司多次面试自动追加轮次后缀，如「天气」→「天气一面」「天气二面」，历史记录不乱
- **双路径保存 / Two save options**：本地 .md 文件，或飞书文档 + 多维表格

---

## 飞书配置 / Feishu Setup（可选 / Optional）

选择保存到飞书时需要配置，选本地保存可跳过。

Only required if you choose to save to Feishu. Skip if saving locally.

首次使用时 skill 会引导完成飞书自建应用配置（约 5 分钟），之后浏览器自动完成 OAuth 授权，无需重复操作。

On first use, the skill will guide you through setting up a Feishu app (~5 min). After that, OAuth is handled automatically.

---

## 平台支持 / Platform Support

| 平台 / Platform | 转录方式 / Engine | 速度 / Speed |
|---|---|---|
| Apple Silicon Mac (M1–M4) | mlx-whisper (GPU) | 41 min audio ≈ 3–5 min |
| Intel Mac / Windows / Linux | openai-whisper (CPU) | Slower, depends on hardware |
