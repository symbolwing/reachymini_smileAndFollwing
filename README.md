# Reachy Mini 摇头微笑识别

基于 Web 摄像头的微笑检测应用，用于控制 [Reachy Mini](https://www.pollen-robotics.com/reachy-mini) 机器人：检测微笑时摆动天线，同时跟踪头部朝向，并将微笑状态同步至 [DigiShow](https://www.digi-show.com/) 灯光控制系统。

## 功能概览

| 模块 | 说明 |
|------|------|
| **微笑检测** | 使用 MediaPipe Face Mesh 实时分析面部关键点，计算微笑分数并可视化 |
| **天线控制** | 检测到微笑时，左右天线以正弦波交替摆动；停止微笑后复位 |
| **头部跟踪** | 根据人脸位置控制机器人头部 Yaw / Pitch，实现「摇头」跟随 |
| **DigiShow 联动** | 通过 WebSocket 发送开关量，将微笑状态映射到 DigiShow 通道 1 |
| **多连接模式** | 支持本地 REST 直连、WebRTC 远程连接、纯本地测试三种模式 |

## 系统架构

```
┌─────────────┐     摄像头      ┌──────────────────┐
│  用户面部   │ ──────────────► │  MediaPipe       │
└─────────────┘                 │  Face Mesh       │
                                └────────┬─────────┘
                                         │ 微笑分数 / 人脸位置
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
            ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
            │ 天线摆动     │    │ 头部跟踪     │    │ DigiShow WS  │
            │ (REST/WebRTC)│    │ (REST/WebRTC)│    │ dgss 协议    │
            └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
                   │                   │                   │
                   └───────────────────┼───────────────────┘
                                       ▼
                              ┌─────────────────┐
                              │  Reachy Mini    │
                              └─────────────────┘
```

## 连接模式

### 1. Lite 本地直连（推荐）

适用于 Reachy Mini Lite 通过 USB 连接本地电脑的场景。

- **前提**：Reachy Mini Control 应用已启动，daemon 运行在 `http://localhost:8000`
- **协议**：REST API（`/api/move/set_target`、`/api/state/full` 等）
- **验证**：可在浏览器打开 `http://localhost:8000/docs` 确认 daemon 状态

### 2. WebRTC 远程连接

适用于无线版或远程控制场景。

- **前提**：机器人 daemon 已注册到信令服务器
- **认证**：
  - 本地运行：需在登录页填入 [Hugging Face Token](https://huggingface.co/settings/tokens)（read 权限）
  - 部署到 HF Space：支持 OAuth 自动登录
- **SDK**：[reachy_mini JS SDK](https://github.com/pollen-robotics/reachy_mini) v1.7.2+

### 3. 仅本地模式

不连接机器人，仅测试摄像头微笑检测与 DigiShow 联动。

## 快速开始

### 环境要求

- 现代浏览器（Chrome / Edge / Firefox，需支持 WebRTC 与 `getUserMedia`）
- 摄像头权限
- （可选）Reachy Mini + 运行中的 daemon
- （可选）DigiShow 软件，WebSocket 服务默认端口 `50000`

### 运行方式

本项目为纯静态页面，无需构建步骤：

```bash
# 方式一：直接用浏览器打开
open index.html

# 方式二：本地 HTTP 服务（推荐，避免部分浏览器限制）
python3 -m http.server 8080
# 然后访问 http://localhost:8080
```

### 使用流程

1. 选择连接模式并完成机器人连接（或选择「仅本地模式」）
2. 允许浏览器访问摄像头
3. （可选）在「DigiShow 连接」面板填入 WebSocket 地址并点击连接
4. 调整「灵敏度」滑块以适配不同用户的微笑习惯
5. 对着摄像头微笑 — 机器人天线开始摆动，DigiShow 通道 1 输出 ON

## 可调参数

| 面板 | 参数 | 范围 | 默认值 |
|------|------|------|--------|
| 微笑检测 | 灵敏度（阈值） | 0.1 – 0.9 | 0.45 |
| 头部跟踪 | Yaw 范围 | 10° – 60° | 35° |
| 头部跟踪 | Pitch 范围 | 5° – 35° | 20° |
| 头部跟踪 | 平滑度 | 0.05 – 0.5 | 0.15 |
| 天线控制 | 摆动幅度 | 10° – 90° | 40° |
| 天线控制 | 摆动速度 | 2 – 12 Hz | 6 Hz |

## DigiShow 协议

微笑状态通过 WebSocket 以 CSV 格式发送：

```
dgss,1,66,0,0,{bValue}
```

- `bValue = 1`：微笑中（通道 1 ON）
- `bValue = 0`：未微笑（通道 1 OFF）

默认连接地址：`ws://127.0.0.1:50000`

## 微笑检测原理

基于 MediaPipe 468 点面部网格，综合以下指标计算 0–1 微笑分数：

- **嘴角宽度比**：嘴角间距 / 脸宽
- **嘴角上扬比**：嘴角相对嘴中心的垂直偏移
- **嘴部开合比**：上下唇间距 / 嘴宽

当分数 ≥ 设定阈值时判定为「微笑中」，触发天线摆动与 DigiShow 信号。

## 项目结构

```
.
├── index.html    # 主应用（UI + 全部业务逻辑）
├── style.css     # 深色主题样式
└── README.md
```

## 技术栈

- **面部识别**：[MediaPipe Face Mesh](https://google.github.io/mediapipe/solutions/face_mesh.html)（CDN）
- **机器人控制**：[Reachy Mini JS SDK](https://github.com/pollen-robotics/reachy_mini) v1.7.2+
- **前端**：原生 HTML / CSS / JavaScript（ES Module），无构建依赖

## 常见问题

**无法连接 daemon（REST 模式）**

1. 确认 Reachy Mini 已通过 USB 连接
2. 确认 Reachy Mini Control 应用已启动
3. 检查端口 8000 是否被占用
4. 访问 `http://localhost:8000/docs` 验证 API 是否可用

**WebRTC 认证失败**

1. 在登录页 Token 输入框填入有效的 HF Token
2. Token 需具备 read 权限
3. OAuth 仅在部署到 Hugging Face Space 后可用，localhost 需手动填 Token

**摄像头无法访问**

- 检查浏览器权限设置
- 建议使用 HTTPS 或 `localhost` 访问（部分浏览器限制非安全上下文）

**DigiShow 连接失败**

- 确认 DigiShow 软件已启动且 WebSocket 服务已开启
- 检查地址与端口是否正确（默认 `ws://127.0.0.1:50000`）

## 许可证

本项目为 Reachy Mini 生态的配套演示应用，机器人 SDK 与 MediaPipe 遵循各自的开源许可证。
