---
title: 语音模式架构
created: 2026-04-10
updated: 2026-05-04
type: concept
tags: [voice, stt, tts, architecture]
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

## 流程

```text
用户按 Ctrl+B 开始录音
    ↓
sounddevice 采集音频 → WAV 临时文件
    ↓
再按 Ctrl+B 停止录音
    ↓
STT 转文字（6 个 Provider 可选，源：tools/transcription_tools.py）:
  - local: faster-whisper（本地，无需 API Key，默认）
  - local_command: 任意本地 STT 二进制
  - groq: Whisper via Groq（免费额度）
  - openai: Whisper via OpenAI
  - mistral: Voxtral Transcribe API
  - xai: xAI Grok STT（ITN + diarization + 21 语言）
    ↓
转录文本作为用户消息发送给 LLM
    ↓
LLM 回复（自动注入简洁指令："respond concisely, 2-3 sentences max"）
    ↓
TTS 语音播报（10 个内置 Provider，源：tools/tts_tool.py BUILTIN_TTS_PROVIDERS）:
  - edge（默认）
  - elevenlabs（流式，边生成边播放）
  - openai
  - minimax
  - xai
  - mistral
  - gemini（Google Gemini TTS）
  - neutts（自托管）
  - kittentts（本地 CPU，~25MB）
  - piper（v0.12.0 新增，本地 VITS，44 语言）
  
  此外可在 `tts.providers.<name>` 下配置自定义 command provider。
```

## STT 配置

```yaml
# config.yaml
stt:
  provider: local   # local | local_command | groq | openai | mistral | xai
                    # 自动模式优先级：local > groq > openai > mistral > xai
  model: base       # faster-whisper 模型大小（base ~150MB，首次自动下载）
```

```bash
# .env
GROQ_API_KEY=...              # Groq Whisper（免费）
VOICE_TOOLS_OPENAI_KEY=...    # OpenAI Whisper
MISTRAL_API_KEY=...           # Mistral Voxtral
XAI_API_KEY=...               # xAI Grok STT
```

## TTS 配置

TTS Provider 选择和语音设置通过 `tools/tts_tool.py` 管理，支持 ElevenLabs 的流式播报——LLM 生成一句就播一句，不用等完整回复。

### TTS Provider 演进

| Provider | 引入版本 | 备注 |
|----------|---------|------|
| edge | 原有 | 微软 Edge TTS（默认） |
| ElevenLabs | 原有 | 流式播报 |
| OpenAI | 原有 | |
| MiniMax | 原有 | v0.12.0 修正 endpoint 为 `v1/text_to_speech` |
| Mistral | v0.10.0 | |
| **Google Gemini TTS** | v0.10.0 | |
| **xAI TTS** | v0.10.0 | 随 xAI Responses API 升级引入 |
| NeuTTS（自托管） | 原有 | |
| **KittenTTS（本地）** | v2026.4.18+ | 本地 CPU 运行，~25MB，默认声音 Jasper |
| **Piper（本地）** | **v0.12.0 新增** | OHF-Voice/piper1-gpl，VITS 神经网络，44 语言，`pip install piper-tts` |

v0.12.0 还引入了 **Pluggable TTS provider registry**：在 `tts.providers.<name>` 下可以挂自定义 `type: command` provider，规则与内置 BUILTIN_TTS_PROVIDERS 隔离（用户配置无法覆盖内置名）。

这些 provider 也可通过 Nous Tool Gateway 统一访问（无需自备 API key）。

### STT Provider 演进

| Provider | 说明 |
|----------|------|
| local（faster-whisper） | 原有，默认；模型自动下载到 `~/.hermes/cache/` |
| local_command | 调用任意本地 STT 二进制（v0.12.0 时已有） |
| Groq Whisper（免费） | 原有 |
| OpenAI Whisper | 原有 |
| **Mistral Voxtral** | v0.12.0 阶段已并入 |
| **xAI Grok STT** | v2026.4.18 引入，支持 ITN、可选 diarization、21 语言 |

> 注：源码 `tools/transcription_tools.py` 模块 docstring 自述 "six providers"，与 `_get_provider()` 的分支一致（local / local_command / groq / openai / mistral / xai）。

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
| `tools/voice_mode.py`（1017 行）| 录音、STT 调度、音频播放 |
| `tools/tts_tool.py`（2185 行）| TTS Provider 路由、流式播报、Piper 缓存 |
| `tools/transcription_tools.py`（911 行）| STT Provider 统一接口 |
| `cli.py` | Push-to-talk 键绑定（Ctrl+B） |
