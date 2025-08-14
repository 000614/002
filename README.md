<h1 align="center">SaaS-Molly: 智能AI玩具服务端</h1>

<p align="center">
  一个为SaaS-Molly智能AI玩具设备设计的后端服务，提供了完整的语音交互、功能插件和设备管理能力。
</p>

<p align="center">
  <a href="#">English</a>
  · <a href="#">常见问题</a>
  · <a href="#">反馈问题</a>
  · <a href="#">部署文档</a>
  · <a href="#">更新日志</a>
</p>

<p align="center">
  <a href="#">
    <img alt="GitHub Release" src="https://img.shields.io/badge/release-v1.0.0-blue.svg?logo=github" />
  </a>
  <a href="#">
    <img alt="GitHub Contributors" src="https://img.shields.io/badge/contributors-1-orange.svg?logo=github" />
  </a>
  <a href="#">
    <img alt="Issues" src="https://img.shields.io/badge/issues-0-green.svg?color=0088ff" />
  </a>
  <a href="#">
    <img alt="GitHub pull requests" src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg?color=0088ff" />
  </a>
  <a href="#">
    <img alt="License" src="https://img.shields.io/badge/license-MIT-white.svg?labelColor=black" />
  </a>
  <a href="#">
    <img alt="stars" src="https://img.shields.io/badge/stars-100-yellow.svg?color=ffcb47&labelColor=black" />
  </a>
</p>

---

## 🚀 主要特性

*   **🤖 智能对话**: 集成先进的ASR、LLM和TTS技术，提供自然流畅的语音对话体验。
*   **🧩 功能插件**: 支持通过插件扩展功能，例如播放音乐、查询天气、控制家电等。
*   **🔧 模块化设计**: 项目采用高度模块化的架构，方便独立升级和维护各个功能组件。
*   **🔄 OTA更新**: 支持通过云端对设备进行固件的远程升级（OTA）。
*   **🔌 WebSocket通信**: 通过WebSocket实现服务端与设备之间的实时双向通信。

---

## 警告 ⚠️

1.  本项目功能未完善，且未通过网络安全测评，请勿在生产环境中使用。 如果您在公网环境中部署学习本项目，请务必做好必要的防护。

---

## 部署文档

### 📦 环境准备

确保您的开发环境中已经安装了 Python 3.10+ 和 pip。

1.  **克隆仓库**

    ```bash
    git clone https://github.com/your-username/SaaS-Molly.git
    cd SaaS-Molly
    ```

2.  **安装依赖**

    ```bash
    pip install -r requirements.txt
    ```

### ⚙️ 配置说明

所有的配置信息都在 `config/settings.toml` 文件中。在启动服务之前，请根据您的实际情况修改此文件。

-   **服务端口**: `system.service_port` 和 `system.ota_port`
-   **模块选择**: `module_selection` 部分用于选择使用的VAD, ASR, LLM, TTS等模块。
-   **API密钥**: 如果您选择使用需要API密钥的模块（如AliLLM），请确保在相应的配置部分填写您的密钥。

### ▶️ 启动服务

您可以根据需要选择不同的启动方式：

*   **统一服务模式 (推荐)**: 同时启动WebSocket、API和OTA服务。

    ```bash
    python dual_server.py
    ```

*   **分离服务模式**: 如果您希望将WebSocket/API服务和OTA服务分开部署，可以单独运行 `bootstrap.py`。

    ```bash
    python bootstrap.py
    ```

服务启动后，您将在控制台看到类似以下的输出：

```
[系统] 🚀 Rebuild-XZ 全部服务已启动，等待设备和API请求...
[系统] 🌐 WebSocket+API服务已在端口 8000 启动。
[OTA服务] OTA接口已在端口 8002 启动。
```

---

## 🤝 贡献

我们非常欢迎社区的贡献！如果您有任何好的想法或者发现了bug，请随时提交Pull Request或Issue。

## 🙏 鸣谢

本项目的灵感和部分代码参考了 [xiaozhi-esp32-server](https://github.com/xinnan-tech/xiaozhi-esp32-server) 项目，在此表示感谢。

## 📄 License

本项目采用 [MIT](LICENSE) 许可协议。
