# Hermes + OpenCode 智能体整合方案

## 概述

本实验项目旨在整合 **Hermes Agent**（消息网关型 AI 助手）与 **OpenCode**（软件工程 AI 助手），实现跨平台的智能体协作。

| 智能体 | 核心能力 | 入口 |
|--------|---------|------|
| **Hermes** | 消息网关（微信/Telegram/Discord 等）、75+ 技能、28+ 工具 | `hermes chat` / `hermes gateway` |
| **OpenCode** | 代码编辑、文件操作、Bash 执行、Git 操作、子代理委托 | `opencode` CLI |

## 设计原则

- **CLI 优先**：两个智能体都在 CLI 环境中运行，直接通过 CLI 相互调用，不引入额外协议层
- **无侵入**：不修改 Hermes 或 OpenCode 源码，纯外部集成
- **渐进式**：从单向调用到双向协作，逐步增加集成深度

## 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层                             │
│  微信 / Telegram / Discord / Terminal                     │
└───────────┬─────────────────────────────┬───────────────┘
            │                             │
            ▼                             ▼
┌─────────────────────┐     ┌─────────────────────────────┐
│   Hermes Gateway    │     │     OpenCode CLI            │
│   (消息网关服务)      │     │     (软件工程助手)           │
└──────────┬──────────┘     └──────────────┬──────────────┘
           │                               │
           │     CLI 相互调用                │
           │  ┌─────────────────────────┐   │
           └──┤  hermes chat -q "..."   ├───┘
              │  opencode chat -q "..." │
              └─────────────────────────┘
```

---

## 阶段一：Hermes → OpenCode（消息通道桥接，MVP）

### 目标

通过 Hermes 的消息网关，让用户能在微信上与 OpenCode 交互。

### 流程

```
微信消息 → Hermes Gateway → opencode chat -q "..." → 处理结果 → 回复回微信
```

### 实现

在 Hermes 中创建一个 `opencode` 技能，CLI 调用 OpenCode。

```yaml
# Hermes skill 配置
skills:
  - name: opencode
    command: opencode chat -q "{{query}}" --quiet
    platforms: [weixin, telegram, discord]
    prefix: /opencode
    timeout: 300
```

### 示例

```
用户 (微信): /opencode 当前项目结构是怎样的？
    ↓
Hermes gateway: → opencode chat -q "当前项目结构是怎样的？" --quiet
    ↓
OpenCode: 探索目录，返回结果
    ↓
Hermes gateway: 将结果发回用户微信
```

**注意**：Hermes 需配置 `WEIXIN_HOME_CHANNEL` 为自己微信号，并确保 gateway 运行中。

---

## 阶段二：OpenCode → Hermes（消息发送能力，已实现 ✅）

### 目标

OpenCode 通过调用 Hermes Gateway 的 Weixin API 向微信发送消息。

### 实现

创建了两个工具脚本：

| 脚本 | 用途 | 调用方式 |
|------|------|----------|
| `tools/send-wechat` | Python 脚本，直接调用 Hermes 的 `send_weixin_direct` API | `python tools/send-wechat <消息>` |
| `tools/wechat-send` | Bash 包装器，自动使用 Hermes venv | `./tools/wechat-send <消息>` |

OpenCode 在需推送通知时，通过 Bash 工具调用：

```bash
./tools/wechat-send "重构已完成，改动如下：..."
```

### 实现细节

`send-wechat` 使用 Hermes Gateway 内置的 `send_weixin_direct` 函数（`gateway/platforms/weixin.py`），通过 iLink Bot API 直发微信消息，不经过 Hermes 会话循环。

### 验证

```bash
$ ./tools/wechat-send "我是opencode"
OK: message sent to o9cq80zB-xxxxx@im.wechat
```

---

## 阶段三：双向委托协作

### 目标

两个智能体在复杂任务中相互委托，各司其职。

### 分工模型

| 任务类型 | 负责智能体 | 方式 |
|---------|-----------|------|
| 代码工程（编辑/搜索/重构） | **OpenCode** | 直接执行 |
| 消息推送与通知 | **Hermes** | `hermes chat -q "发消息:..."` |
| 用户交互与对话管理 | **Hermes** | 多平台会话管理 |
| 文件系统操作 | **OpenCode** | 直接执行 |
| 外部 API 调用 | **Hermes** | 丰富的技能生态 |
| Git 版本控制 | **OpenCode** | 直接执行 |

### 委托机制

CLI 调用天然支持双向委托，不依赖额外协议：

- **Hermes → OpenCode**：Hermes skill 中配置 `opencode chat -q` 命令
- **OpenCode → Hermes**：OpenCode 的 Task/Bash 工具执行 `hermes chat -q` 命令

### 流程示例

```
用户 (微信): 帮我重构 src/utils.py，完成后通知我
    ↓
Hermes: 理解意图，调用 opencode skill
    ↓
Hermes → opencode chat -q "重构 src/utils.py，完成后回复'重构完成'"
    ↓
OpenCode: 执行重构任务
    ↓
OpenCode → hermes chat -q "请给用户微信发送消息：重构完成，改动如下：..."
    ↓
Hermes gateway: 发送微信消息给用户
```

---

## 阶段四（可选）：共享工作区与协议文件

### 目标

实现异步解耦的协作模式。

### 方案

使用共享目录 `agent-inbox/` 作为通信层：

```
agent-inbox/
├── requests/       # Hermes → OpenCode 请求
│   └── 001.json
├── responses/      # OpenCode → Hermes 响应
│   └── 001.json
└── notifications/  # OpenCode → Hermes 通知
    └── 微信推送.json
```

- Hermes 写入请求到 `requests/`，OpenCode 轮询处理
- OpenCode 写入通知到 `notifications/`，Hermes gateway 读取并推送
- 适合不需要实时响应的后台任务

---

## 技术选型

| 组件 | 方案 |
|------|------|
| 进程间通信 | CLI 相互调用（`hermes chat -q` / `opencode chat -q`） |
| 异步通信 | 共享文件（`agent-inbox/` 目录协议） |
| 会话存储 | Hermes 原生会话管理 |
| 消息路由 | Hermes Gateway 技能机制 |
| 通知推送 | Hermes gateway 微信通道 |

## 实现状态

| 阶段 | 状态 | 说明 |
|------|------|------|
| 一：Hermes → OpenCode | ✅ 内置 | Hermes 已有 `opencode` built-in skill，通过 `opencode run` 调用 |
| 二：OpenCode → Hermes | ✅ 已实现 | `tools/send-wechat` + `tools/wechat-send` 可直接发送微信消息 |
| 三：双向委托协作 | ⏳ 待完善 | 需测试完整闭环（微信发指令 → OpenCode 执行 → 微信通知结果） |
| 四：共享工作区 | 📋 可选 | 按需推进 |

## 下一步

1. 测试 Hermes → OpenCode 方向：在微信中发送 `/opencode <任务>` 触发 OpenCode
2. 测试完整闭环：微信发指令 → OpenCode 执行 → 微信通知结果
3. 编写集成测试用例，覆盖双向调用场景
