## SaaS-Molly 服务器端（Rebuild-XZ）

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue)](https://www.python.org/)
[![aiohttp](https://img.shields.io/badge/aiohttp-3.8%2B-green)](https://docs.aiohttp.org/)
[![Status](https://img.shields.io/badge/Status-Active-brightgreen)](./)

SaaS-Molly 的服务端为 ESP/玩具设备提供 WebSocket 会话、语音管线（VAD/ASR/LLM/TTS）、意图解析、记忆、以及 OTA 接口。默认支持通过 `/xiaozhi/v1/` 建立 WebSocket 连接，通过 `/xiaozhi/ota/` 进行 OTA 初始化和地址发现，并提供若干简易 REST API 端点用于可用性探测与集成演示。


## 目录

- **功能概览**
- **系统架构**
- **目录结构**
- **环境要求与安装**
- **配置说明**
- **启动方式**
- **设备接入与协议**
- **REST API**
- **OTA 接口**
- **模块说明**
- **常见问题**
- **开发与贡献**
- **许可证**


## 功能概览

- **语音管线**: 内置 VAD、ASR（FunASR 可本地/服务端两种模式）、LLM（Ali LLM 兼容 OpenAI 接口）、TTS（EdgeTTS/GPT-SoVITS v2）。
- **会话编排**: `DeviceSessionHandler` 统一处理文本/音频/IOT/传感器事件，异步驱动 LLM 与 TTS，边处理边推流。
- **设备管理**: 通过 WebSocket 维护设备会话，支持 `device-id`/`client-id` 鉴别以及 IoT 描述符/状态注入。
- **OTA 支持**: 提供独立 OTA 发现接口，返回设备应连接的 WebSocket 地址与固件信息字段示例。
- **插件扩展**: 压力传感器/马达/蜂鸣器等插件能力，可扩展回调与意图动作。


## 系统架构

```
设备 <—WebSocket—> 服务端会话层(DeviceSessionHandler)
                 ├─ VAD (adapters.vad_unit.VADUnit)
                 ├─ ASR (adapters.asr_unit.AcousticDecoder)
                 ├─ LLM (adapters.llm_unit.LanguageReasoner)
                 ├─ TTS (adapters.tts_unit.TTSUnit)
                 ├─ Memory (adapters.memory_unit.MemoryUnit)
                 ├─ Intent (adapters.intent_unit.PurposeResolver)
                 └─ Plugins (plugins.ExtensionHub + pressure/motor/buzzer)

HTTP:
- /api/*          基础探活与示例 API（见下文）
- /xiaozhi/ota/   OTA 地址发现与返回
```


## 目录结构

```
rebuild-xz/
  adapters/           # VAD/ASR/LLM/TTS/WS/OTA 适配器
  api/                # REST API 路由
  config/             # 配置与注册表
  orchestrator/       # 会话编排、OTA 处理、网关等
  plugins/            # 传感器与硬件插件
  models/             # 本地模型目录（如 FunASR）
  dual_server.py      # 同时启动 WebSocket + API + OTA 的主入口
  run_ota.py          # 独立 OTA 服务启动脚本
  ota_server.py       # OTA 服务实现（更详细版本）
  requirements.txt    # Python 依赖
  ...
```


## 环境要求与安装

- **Python**: 3.9+（建议 3.10/3.11）
- **依赖**: 见 `requirements.txt`
- **音频工具**:
  - `pydub` 依赖 FFmpeg，建议安装 FFmpeg（Windows 可安装官方包并将其 `bin` 目录加入 PATH）

安装步骤：

```bash
# 1) 创建虚拟环境（可选）
python -m venv .venv
# Windows PowerShell: .\.venv\Scripts\Activate.ps1
# macOS/Linux: source .venv/bin/activate

# 2) 安装依赖
pip install -r requirements.txt
```


## 配置说明

配置文件：`config/settings.toml`

- `system.service_port`: 主服务端口（默认 8000），用于 WebSocket 与 REST API
- `system.ota_port`: OTA 服务端口（默认 8002）
- `modules.*`: 选择具体实现，如 `voice_activity_detector`, `speech_recognizer`, `language_model`, `speech_synthesizer`, `memory_engine`, `intent_resolver`
- `asr.FunASR`: FunASR 的本地/服务器模式配置
- `llm.AliLLM`: OpenAI 兼容配置（`base_url`、`model_name`、`api_key` 等）
- `tts.EdgeTTS` 与 `tts.GPT_SOVITS_V2`: 语音合成引擎配置
- `intent.function_call.plugins`: 内置示例插件列表
- `wakeup_words`: 唤醒词列表

请根据实际密钥与服务地址，修改 `llm.AliLLM.api_key` 等字段。


## 启动方式

- **一体化启动（推荐）**：同时启动 WebSocket+API 与 OTA 接口

```bash
python dual_server.py
```

默认：
- WebSocket/REST API 监听 `0.0.0.0:${service_port}`（默认 8000）
- OTA 监听 `0.0.0.0:${ota_port}`（默认 8002）

- **仅启动 OTA 服务（独立）**：

```bash
python run_ota.py
```


## 设备接入与协议

- **WebSocket 路径**: `ws://<server_host>:<service_port>/xiaozhi/v1/`
- **握手 Header/Query**: 需包含 `device-id` 与 `client-id`，否则将被拒绝
- **握手步骤**：
  1. 连接建立后，服务端先下发简要 `hello`
  2. 设备发送 `{"type":"hello", ...}`
  3. 服务端返回完整 `hello`（含音频参数）并进入工作流

- **音频参数（服务端返回示例）**：
```json
{
  "type": "hello",
  "version": 1,
  "transport": "websocket",
  "session_id": "<uuid>",
  "audio_params": {
    "format": "opus",
    "sample_rate": 16000,
    "channels": 1,
    "frame_duration": 60
  }
}
```

- **文本消息类型（部分）**：
  - `hello`: 设备端握手
  - `listen`: 控制拾音与检测
    - `{ "type":"listen", "state":"start|stop|detect", "mode":"auto|manual" }`
  - `detect`: 直接触发固定欢迎语进入对话
  - `abort`: 中断当前处理流程
  - `iot`: 注入 IoT 设备描述符/状态
  - `pressure_sensor`: 压力传感器事件
  - `tts`: 直接请求 TTS

- **二进制消息**：
  - 当 `listen.state=start` 后，发送音频二进制帧（Opus/PCM，依你设备端实现），服务端自动积累并在静默或 stop 时触发识别 + 对话流程

- **服务端常见下行消息**：
  - `stt`: 语音转文本结果 `{ "type":"stt", "text":"...", "session_id":"..." }`
  - `tts`: TTS 状态 `{ "type":"tts", "state":"start|end", "tts_type":"edge|gpt_sovits_v2" }`
  - 设备可按需扩展处理 `llm` 中间态与分片 TTS 音频等（见 `orchestrator/device_session.py`）

最小连接示例（伪代码）：
```text
GET ws://SERVER:8000/xiaozhi/v1/
Headers:
  device-id: demo-device-001
  client-id: demo-client-001

<- {"type":"hello","version":1,...}
-> {"type":"hello","device":"SaaS-Molly"}
<- {"type":"hello","audio_params":{...}}
-> {"type":"listen","state":"detect","text":"你好助手"}
<- {"type":"stt","text":"嘿，你好呀"}
<- {"type":"tts","state":"start"}
... 音频播放 ...
<- {"type":"tts","state":"end"}
```


## REST API

默认根路径与示例（文件：`api/routes.py`）：

- `GET /api/device` → `{ "device": "ok" }`
- `GET /api/agent` → `{ "agent": "ok" }`
- `GET /api/models` → `{ "models": "ok" }`
- `GET /api/ttsVoice` → `{ "ttsVoice": "ok" }`
- `GET /api/admin` → `{ "admin": "ok" }`
- `GET /api/user` → `{ "user": "ok" }`
- `GET /api/dummy` → `{ "dummy": true }`

这些接口主要用于可用性探测与调通演示，可按需扩展。


## OTA 接口

- **路径**：`/xiaozhi/ota/`
- **端口**：默认 `ota_port=8002`（独立 OTA 服务）或由 `dual_server.py` 并行启动

- `GET /xiaozhi/ota/`
  - 作用：健康检查，返回设备应连接的 WebSocket 地址（基于 `system.service_port`）
  - 响应：`text/plain`

- `POST /xiaozhi/ota/`
  - 作用：设备上报基础信息后，服务端返回时间/固件/目标 WebSocket 地址
  - Header：可包含 `device-id`
  - Body（示例）：
```json
{
  "application": { "version": "1.0.0" }
}
```
  - 响应（示例）：
```json
{
  "server_time": { "timestamp": "1736500000000", "timezone_offset": "480" },
  "firmware": { "version": "1.0.0", "url": "http://<ip>/firmware/1.0.0.bin" },
  "websocket": { "url": "ws://<ip>:8001/xiaozhi/v1/" }
}
```
  - 说明：POST 中 WebSocket 端口可能固定返回 `8001` 以兼容既有设备固件；GET 则使用配置的 `service_port`。可根据你的设备侧约定进行调整（见 `orchestrator/ota_handler.py`）。


## 模块说明（节选）

- `adapters.vad_unit.VADUnit`：语音活动检测
- `adapters.asr_unit.AcousticDecoder`：FunASR 本地/服务端两种工作模式
- `adapters.llm_unit.LanguageReasoner`：对接 Ali LLM（OpenAI 兼容）
- `adapters.tts_unit.TTSUnit`：EdgeTTS 或 GPT-SoVITS v2 合成
- `adapters.memory_unit.MemoryUnit`：简单记忆占位实现
- `adapters.intent_unit.PurposeResolver`：意图路由与插件函数调用
- `plugins.*`：`pressure_sensor`、`motor_control`、`buzzer_control` 等
- `orchestrator/device_session.py`：设备会话全流程处理的核心


## 常见问题

- **无法播放 TTS/处理音频**：确认已安装 FFmpeg，并且其路径在系统 PATH 中；Windows 建议安装官方 FFmpeg 发行版。
- **ASR（FunASR）本地模式无法加载模型**：检查 `config/settings.toml` 中 `asr.FunASR.model_dir` 是否存在，并确保有相应权重。
- **LLM 无响应/报 401**：检查 `llm.AliLLM.api_key`、`base_url` 和 `model_name` 设置；网络需可访问对应域名。
- **设备连接被拒绝**：WebSocket 握手必须携带 `device-id` 与 `client-id`（可放在 Header 或 Query）。
- **OTA 返回端口与主服务端口不一致**：见上文说明，属于兼容性设计，可根据设备固件约定调整实现。


## 开发与贡献

- 建议启用虚拟环境并使用 `pytest` 运行示例测试（仓库内包含若干集成/单测样例，可参考命名为 `test_*.py` 的文件）。
- 代码风格：遵循清晰命名、显式类型注解（如适用）、早返回与错误优先处理；避免深层嵌套。
- 重要位置：
  - WebSocket 会话入口：`dual_server.py` 与 `orchestrator/device_session.py`
  - REST/OTA 路由：`api/routes.py`、`orchestrator/ota_handler.py`
  - 模块装配：`dual_server.py.initialize_modules()` 与 `config/settings.toml`

本项目结构清晰，适合根据你的设备协议进行裁剪与扩展：可在 `adapters/` 新增编解码/能力模块，或在 `plugins/` 扩展硬件与行为。


## 许可证

当前仓库未附带许可证文件。若需开源或分发，请补充 `LICENSE` 并在此处注明授权协议（例如 MIT、Apache-2.0 等）。 
