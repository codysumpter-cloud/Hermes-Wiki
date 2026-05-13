---
title: 语音模式架构
created: 2026-04-10
updated: 2026-05-13
type: concept
tags: [voice, stt, tts, architecture, piper]
sources: [tools/voice_mode.py, tools/tts_tool.py, tools/transcription_tools.py, cli.py]
---

# 语音模式架构

## 概述

Hermes 支持 Push-to-talk 语音交互：用户按键录音 → STT 转文字 → LLM 处理 → TTS 语音播报回复。整个链路在 CLI 中完成，依赖可选的音频库。

## 依赖

```bash
pip install sounddevice numpy   # 或
pip install hermes-agent[voice]
```

音频库**按需懒加载**，不装也不影响文本模式。在无音频设备的环境（SSH、Docker、WSL）中自动检测并禁用。

> WSL 改进（v0.12.0，081f936）：`sd.query_devices()` 返回空列表但 `PULSE_SERVER` env 已设置时，仍判定为有音频可用（之前直接禁用，跑 WSL + PulseAudio 的用户没法用语音）。

## 流程

```text
用户按 Ctrl+B 开始录音
    ↓
sounddevice 采集音频 → WAV 临时文件
    ↓
再按 Ctrl+B 停止录音
    ↓
STT 转文字（3 个 Provider 可选）:
  - local: faster-whisper（本地，无需 API Key）
  - groq: Whisper via Groq（免费额度）
  - openai: Whisper via OpenAI
    ↓
转录文本作为用户消息发送给 LLM
    ↓
LLM 回复（自动注入简洁指令："respond concisely, 2-3 sentences max"）
    ↓
TTS 语音播报（10 个内置 Provider + 任意命令型自定义 provider）:
  - Edge TTS（默认、免费、无需 API key）
  - ElevenLabs（流式，边生成边播放）
  - OpenAI TTS
  - MiniMax TTS（含 voice cloning）
  - Mistral Voxtral TTS（多语种，原生 Opus）
  - Google Gemini TTS（30 prebuilt voices）
  - xAI TTS（Grok voices）
  - NeuTTS（本地）
  - KittenTTS（本地 25MB）
  - Piper（本地，OHF-Voice/piper1-gpl 神经 VITS，44 语种）—— v0.12.0+
  - 任意 `tts.providers.<name>` 命令型自定义 provider（v0.12.0+）
```

## STT 配置

```yaml
# config.yaml
stt:
  provider: local   # local | groq | openai（优先级：local > groq > openai）
  model: base       # faster-whisper 模型大小（base ~150MB，首次自动下载）
```

```bash
# .env
GROQ_API_KEY=...              # Groq Whisper（免费）
VOICE_TOOLS_OPENAI_KEY=...    # OpenAI Whisper
```

## TTS 配置

TTS Provider 选择和语音设置通过 `tools/tts_tool.py` 管理，支持 ElevenLabs 的流式播报——LLM 生成一句就播一句，不用等完整回复。

### 新增 TTS Provider（v0.10.0）

| Provider | 来源 |
|----------|------|
| ElevenLabs | 原有 |
| OpenAI | 原有 |
| **Google Gemini TTS** | v0.10.0 新增，通过 Gemini API |
| **xAI TTS** | v0.10.0 随 xAI Responses API 升级引入 |
| **KittenTTS（本地）** | v2026.4.18+ 引入，本地 CPU 运行，无需 GPU 和 API key，默认模型 `KittenML/kitten-tts-nano-0.8-int8`（25MB），默认声音 `Jasper`，其他声音由 KittenTTS 包提供（25-80MB 模型范围） |

这些 provider 也可通过 Nous Tool Gateway 统一访问（无需自备 API key）。

### v0.12.0+ TTS 更新

| 变更 | 说明 |
|------|------|
| **Piper 原生 provider** | `pip install piper-tts` 启用，OHF-Voice/piper1-gpl 神经 VITS，44 语种。模块级 `_piper_voice_cache` 复用 voice 实例。源码 `tools/tts_tool.py:1342-1500`。closes #8508（8d302e3） |
| **`tts.providers.<name>` 命令型 provider 注册表** | 用户可在 `~/.hermes/config.yaml` 声明 `type: command` 的命名 provider，Hermes 将文本写入 UTF-8 临时文件并执行配置好的 shell 命令（占位符替换 `{output_path}` / `{input_path}`）。已有 10 个内置 provider 名字被保留为 frozenset `BUILTIN_TTS_PROVIDERS`，命令型 provider 不能撞名。默认超时 120s，支持 `mp3/wav/ogg/flac`，默认 5000 字符限制。源码 `tools/tts_tool.py:309-470`（2facea7） |
| **gateway 音频路由集中化 + FLAC 支持** | `gateway/platforms/base.py:_AUDIO_EXTS` 加入 `.flac`，Telegram 对原生不支持的 `.wav`/`.flac` 自动 document fallback（aa7bf32） |
| **MiniMax TTS endpoint 更新** | API 升级到 `v1/text_to_speech`（6875471） |

### STT Provider 扩展（v2026.4.18+）

| Provider | 说明 |
|----------|------|
| Groq Whisper（免费） | 原有 |
| OpenAI Whisper | 原有 |
| Deepgram | 原有 |
| **xAI Grok STT** | 新增，POST `/v1/stt`，支持 ITN（Inverse Text Normalization）+ 可选 diarization |

## 语音模式特殊行为

- LLM 收到语音输入时，系统自动注入前缀指令要求简短回复
- 该前缀仅用于 API 调用，**不持久化到会话历史**（通过 `persist_user_message` 参数保存原始转录文本）
- 持续语音模式下遇到持久错误（如 429）会自动停止，防止错误 → 录音 → 错误的死循环

## 相关页面

- [[cli-architecture]] — CLI 中的语音模式集成
- [[auxiliary-client-architecture]] — STT/TTS 使用 auxiliary 模型配置

## 关键源码

| 文件 | 职责 |
|------|------|
| `tools/voice_mode.py`（1018 行）| 录音、STT 调度、音频播放 |
| `tools/tts_tool.py`（2225 行）| TTS Provider 路由（10 内置 + 命令型自定义）、流式播报 |
| `tools/transcription_tools.py`（921 行）| STT Provider 统一接口 |
| `cli.py` | Push-to-talk 键绑定（Ctrl+B） |
