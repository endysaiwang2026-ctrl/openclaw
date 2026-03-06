# OpenClaw 项目结构速览（框架、代码组织、扩展点）

> 这份文档目标：帮助你先建立“主干认知”，再决定深入哪一层代码。

## 1) OpenClaw 的核心框架：Gateway 作为统一控制平面

OpenClaw 的主轴不是某一个渠道机器人，而是一个 **Gateway 控制平面**：

- 所有 CLI、Web 控制台、节点设备（macOS/iOS/Android/headless）都通过统一 WebSocket 协议连接网关。
- 消息渠道（Telegram / Slack / Discord / Signal / iMessage / WhatsApp 等）由网关统一管理。
- 业务能力（工具、路由、模型调用、会话等）在网关和插件层集中编排。

从代码入口可以看到这个设计：

- `src/index.ts`：启动时先做环境标准化、日志捕获、运行时检查，再构建 CLI。
- `src/cli/program/build-program.ts`：构建 Commander 程序。
- `src/cli/program/command-registry.ts`：核心命令惰性注册，按模块加载。
- `docs/concepts/architecture.md`：明确说明单网关 + WS 控制面的总体模型。

---

## 2) 代码结构怎么读（从“框架层”到“实现层”）

### 2.1 CLI 层（命令编排）

**关键文件**

- `src/index.ts`
- `src/cli/program/build-program.ts`
- `src/cli/program/command-registry.ts`

**理解重点**

1. 命令不是一次性全量加载，而是按主命令（如 `onboard`、`agent`、`message`、`status`）进行惰性 `import`。
2. `registerProgramCommands` 是 CLI 的“路由表入口”；你新增命令时，通常从这里追踪注册链路。
3. CLI 和实际业务逻辑分离：命令层负责参数与交互，业务落在 `src/commands/*`。

### 2.2 Gateway + 协议层（系统骨架）

**关键文件**

- `docs/concepts/architecture.md`
- `src/gateway/protocol/schema.ts`
- `src/gateway/protocol/schema/*`

**理解重点**

1. 协议是显式 schema 化的（请求/响应/事件），便于跨端一致性。
2. 首帧握手 + 鉴权 + role/capabilities 的约束，使“客户端”和“节点设备”共享协议但隔离权限。
3. 你如果要扩展跨端能力，通常要同时改：协议 schema + server method + 客户端调用方。

### 2.3 渠道层（内置渠道 + 插件渠道）

**关键文件**

- `src/channels/registry.ts`
- `src/channels/plugins/index.ts`

**理解重点**

1. `src/channels/registry.ts` 维护核心渠道的 ID、顺序、别名、展示元数据。
2. `normalizeAnyChannelId` 会从当前激活的插件注册表中解析渠道 ID，说明渠道已插件化接入。
3. 共享路径尽量依赖轻量 registry，不直接耦合“重型渠道实现”。

### 2.4 插件层（最重要的扩展机制）

**关键文件**

- `src/plugins/runtime.ts`
- `src/plugins/loader.ts`
- `src/plugins/registry.ts`
- `src/plugin-sdk/index.ts`

**理解重点**

1. 全局 active registry + version 机制支持缓存与热更新场景。
2. loader 负责发现插件、加载模块、配置校验、构建运行时。
3. 插件可注册多种能力：
   - tools（Agent 工具）
   - hooks（生命周期钩子）
   - channels（新渠道）
   - providers（模型/服务提供方）
   - gateway handlers（网关方法）
   - HTTP 路由 / CLI 命令 / services

---

## 3) 仓库组织（Monorepo）

从 `pnpm-workspace.yaml` 可见工作区结构：

- 根包 `.`（主 CLI + Gateway）
- `ui`
- `packages/*`
- `extensions/*`

这意味着：

1. **核心系统在 `src/`**：CLI、网关、路由、模型、通道、插件运行时。
2. **扩展能力在 `extensions/*`**：每个扩展可独立维护和发布。
3. **前端/应用端在 `ui`、`apps`**：作为控制面或节点侧实现。

常见扩展目录（示例）：

- `extensions/matrix`
- `extensions/msteams`
- `extensions/voice-call`
- `extensions/zalouser`

---

## 4) 可扩展性到底在哪里

如果你想评估“这个项目是否易扩展”，重点看下面四个入口：

### A. 新增插件能力（优先级最高）

通过 plugin API 与 SDK，你可以尽量不改核心，就增加工具、认证、网关方法、HTTP 集成。

### B. 新增消息渠道

在渠道插件体系下接入新渠道，复用现有会话、路由、权限、状态体系。

### C. 扩展 Gateway 协议

新增跨端能力时，在 schema 层做显式建模，保证 server/client/node 三端一致演进。

### D. 扩展 CLI 命令

CLI 命令模块化且惰性加载，新增命令通常是“局部增量”，而非大规模改动。

---

## 5) 推荐阅读路径（适合第一次进仓）

1. `src/index.ts` + `src/cli/program/*`（理解命令入口与程序装配）
2. `docs/concepts/architecture.md` + `src/gateway/protocol/schema/*`（理解控制平面协议）
3. `src/channels/registry.ts` + `src/channels/plugins/index.ts`（理解渠道抽象与规范化）
4. `src/plugins/loader.ts` + `src/plugins/registry.ts` + `src/plugin-sdk/index.ts`（理解真正扩展点）

如果你是要“二次开发”，第 4 步（插件系统）应尽早深入。

---

## 6) 一句话结论

OpenClaw 的可扩展性主要来自：

- **插件系统（`src/plugins` + `src/plugin-sdk`）**：功能扩展主通道；
- **渠道插件化（`src/channels/plugins`）**：渠道接入标准化；
- **协议 schema 化（`src/gateway/protocol/schema/*`）**：跨端能力可持续演进；
- **CLI 模块化注册（`src/cli/program/*`）**：工具链扩展成本低。
