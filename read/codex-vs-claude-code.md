# Codex 与 Claude Code 的架构思路对比

本文比较两个仓库：

- 当前仓库 `codex`
- `C:\repos\CWYH\claude-code-main`

目标不是做逐文件源码审阅，而是从产品与架构设计角度回答三个问题：

1. 两者在整体思路上有哪些共同点。
2. 两者分别把复杂度压在了哪里。
3. 如果站在产品能力演进角度，各自最值得借鉴什么。

## 一、先给结论

如果用一句话概括两者差异：

- **Codex 更像平台化 agent runtime + 多前端控制面。**
- **Claude Code 更像能力高度集中在单体 CLI 中的产品化 agent。**

两者都很强，但强的方向不同：

- Codex 的优势在于边界清晰、服务化能力强、前后端解耦更彻底。
- Claude Code 的优势在于 CLI 内体验浓度高、工作流完整度高、用户可感知功能密度更高。

## 二、共同点：两者其实在解决同一类问题

尽管技术栈完全不同，两个仓库都围绕同一组核心问题展开。

### 1. 都不是简单聊天壳，而是真正的 agent loop

两边都不是“输入 prompt，打印回答”的薄封装，而是都在建设：

- 多轮会话状态
- 工具调用循环
- 文件读写与命令执行
- 权限与安全控制
- 长会话恢复与持久化

在 Claude Code 中，这个中心是 `src/QueryEngine.ts`；在 Codex 中，这个中心更分散但更稳定，主要落在 `codex-rs/core` 的线程、工具和 rollout 体系里。

### 2. 都把工具系统当成一等公民

两个项目都明白：coding agent 的价值主要取决于工具，而不是聊天框。

- Claude Code 有 `src/Tool.ts`、`src/tools.ts` 和大量 `src/tools/*`。
- Codex 有 `codex-rs/core` 内部工具编排，同时又在 `codex-rs/tools` 中抽离共享 tool schema 和 tool spec。

也就是说，两者都把“工具调用”视作主干，而不是附属功能。

### 3. 都在做扩展生态，而不是封闭产品

两个仓库都支持：

- MCP
- skills
- plugins 或插件化能力
- 更丰富的外部连接方式

这说明它们都不是只想成为单一官方客户端，而是在往 agent 平台方向演进。

### 4. 都在做长期会话而不是一次性执行

Claude Code 有 session / transcript / resume；Codex 有 thread / turn / item / rollout / fork / archive。

虽然抽象方式不同，但都说明一个事实：**真实工程场景需要长期协作对象，而不是一次性 prompt。**

## 三、关键差异：两者把复杂度压在了不同位置

## 1. 技术栈与运行时路线

### Codex

- 核心运行时：Rust workspace
- 分发包装：Node/npm
- 终端 UI：Ratatui
- 服务控制面：app-server JSON-RPC

### Claude Code

- 核心运行时：Bun + TypeScript
- 终端 UI：React + Ink
- feature gating：大量 `bun:bundle` 条件裁剪

### 差异本质

- Codex 选择 **native runtime 优先**，强调跨平台原生执行、协议稳定性、长期可维护边界。
- Claude Code 选择 **高迭代 CLI 产品优先**，强调在一个 TypeScript/Bun 单体里快速累积功能密度。

这不是简单的“Rust vs TypeScript”，而是 **平台工程优先级** 和 **CLI 产品优先级** 的差异。

## 2. 核心业务组织方式

### Codex：多 crate 分层收敛

从这些入口能看出它的组织方式：

- `codex-rs/cli/src/main.rs`
- `codex-rs/core/src/lib.rs`
- `codex-rs/app-server/README.md`
- `codex-rs/tools/README.md`

Codex 的做法是：

- 用 `cli` 聚合入口。
- 用 `core` 承载 runtime。
- 用 `protocol` / `app-server-protocol` 固化边界。
- 用独立 crate 逐步外提工具、状态、沙箱和连接器能力。

### Claude Code：单仓大模块集中编排

从这些入口能看出它的组织方式：

- `src/main.tsx`
- `src/QueryEngine.ts`
- `src/Tool.ts`
- `src/tools.ts`
- `src/commands.ts`

Claude Code 的做法是：

- 在主入口中并行初始化大量能力。
- 用 `QueryEngine.ts` 持有对话和 agent 核心状态。
- 用 `tools.ts` 与 `commands.ts` 把 CLI 中的能力直接注册起来。
- 大量依靠 feature flag 与动态 import 控制功能组合。

### 差异本质

- Codex 更强调 **边界稳定性**。
- Claude Code 更强调 **能力集中编排**。

前者更利于多前端和协议复用，后者更利于单产品快速堆出高密度体验。

## 3. 前端与服务边界

这是最关键的差异之一。

### Codex：前端不是唯一入口

Codex 明确存在多种消费方式：

- TUI：`codex-rs/tui`
- headless：`codex-rs/exec`
- IDE / rich client：`codex-rs/app-server`
- SDK：`sdk/typescript`、`sdk/python`

这说明它的产品思路是：**agent 能力先独立，前端只是消费层。**

### Claude Code：CLI 本身就是主要产品形态

Claude Code 也有 bridge、remote、server 等模块，但从整体组织和阅读重心看，主产品仍然是 CLI/TUI 自身：

- `src/main.tsx` 汇聚绝大多数启动逻辑
- `src/commands.ts` 和 `src/tools.ts` 直接体现功能密度
- `src/components/`、`src/screens/` 与核心行为耦合更紧

### 差异本质

- Codex 是“多前端共享一个 runtime”。
- Claude Code 更像“一个非常强的 CLI，顺便向外延伸”。

## 4. 协议思维与平台化深度

### Codex

Codex 在这一点上非常突出：

- `codex-rs/protocol` 明确做共享类型。
- `codex-rs/app-server-protocol` 明确做 v1/v2 wire contract。
- Python SDK 直接站在 app-server v2 上。

这说明它不是“把 CLI 输出包装一下”，而是在认真建设一个 **可长期演进的控制面协议**。

### Claude Code

Claude Code 也有很多内部类型和桥接协议，但整体观感更偏“围绕当前 CLI 产品组织”，而不是先抽成一个对外长期稳定的服务面。

### 差异本质

- Codex 更像平台 API 思维。
- Claude Code 更像产品 runtime 思维。

## 5. Feature Flag 与实验能力组织

### Claude Code

Claude Code 大量依赖 `bun:bundle` feature，例如：

- `COORDINATOR_MODE`
- `KAIROS`
- `VOICE_MODE`
- `WORKFLOW_SCRIPTS`
- `BUDDY`

这让它在一个代码库里可以快速切换和裁剪很多实验能力。

### Codex

Codex 也有 feature / experimental API / collaboration mode，但更多通过：

- crate 边界
- app-server experimental API 标注
- models manager / config / protocol 约束

来组织演进。

### 差异本质

- Claude Code 更适合快速叠实验。
- Codex 更适合把实验能力逐步沉淀成稳定接口。

## 四、产品能力视角：两者各自的长板与短板

## 1. Codex 的长板

### 1. 平台化能力强

Codex 已经形成了比较完整的“runtime + protocol + SDK + app-server”体系，这对 IDE 集成、企业接入、自动化嵌入都很有优势。

### 2. 安全与边界意识更强

从 sandbox、execpolicy、协议类型、thread model 可以看出，Codex 很重视长期可控性，尤其适合对合规、跨平台一致性和服务接入有要求的环境。

### 3. 前端可替换性更好

因为核心能力没有完全绑死在 TUI 上，所以后续扩展 Web、IDE、SDK、后台 agent 都更顺。

## 2. Codex 的短板

### 1. 用户可感知功能密度没有 Claude Code 那么“扑面而来”

仓库能力很多，但很多能力沉在协议、crate 或 config 里，用户第一视角不一定直接感知到。

### 2. 产品入口的心智稍微分散

TUI、exec、app-server、SDK 都存在，平台能力很强，但普通用户不一定立刻理解“什么时候该用哪一层”。

### 3. `core` 的认知负担仍然偏高

虽然已经在治理，但 `codex-core` 仍然是复杂度高地，新人理解成本不低。

## 3. Claude Code 的长板

### 1. CLI 产品完成度很高

从 `src/commands.ts`、`src/tools.ts`、`src/main.tsx` 可以看出，Claude Code 的 CLI 像一个“能力超密集的工作台”，很多功能直接摆在用户面前。

### 2. 工作流完整度强

多 agent、任务、计划、权限、IDE bridge、remote、voice、workflow、buddy 等能力，在一个终端产品里有很强的连续体验。

### 3. 实验速度快

feature flag + TypeScript 单仓大模块，使它可以快速长出新功能并测试用户反馈。

## 4. Claude Code 的短板

### 1. 核心复杂度更集中

`QueryEngine.ts`、`Tool.ts`、`commands.ts`、`main.tsx` 这类超大文件说明复杂度集中度很高，长期维护风险更大。

### 2. 前端与 runtime 的边界没 Codex 那么硬

这并不代表设计差，只是意味着当它继续往多前端、多 SDK、多服务控制面扩展时，重构成本可能更高。

### 3. 平台协议化程度略弱

从仓库形态看，它更像一个非常强的 CLI 产品，而不是一个已经清晰分出控制面协议和平台层的系统。

## 五、Codex 可以向 Claude Code 学什么

这里重点从产品能力角度讲，而不是单纯架构洁癖。

### 1. 更强的功能可见性

Codex 已经有很多能力，但普通用户不一定能直观看到。Claude Code 的强项是：

- 命令与工具入口更显性
- 用户更容易感觉“这个产品能做很多事”

Codex 可以加强：

- TUI 内的功能发现路径
- 常用工作流的显式入口
- 对 skills/plugins/MCP 能力的前台展示

### 2. 更完整的任务工作流包装

Claude Code 的产品感很强，一个重要原因是用户不需要先理解架构，再理解如何组合能力。Codex 可以补强：

- 更清晰的任务型入口
- 更明显的计划、执行、审查、恢复、回滚工作流
- 更顺手的多 agent 编排可视化反馈

### 3. 更强的实验功能前台化

Codex 已经有 experimental API 和 feature 体系，但普通用户侧的“试验能力体验”没有 Claude Code 那么集中。可以考虑让一些实验能力更易发现、更易启用和比较。

### 4. 更强的单产品叙事

Codex 当前更像平台，优点很大，但也容易让用户只看到“它很强”，却未必立刻形成“它如何改变我工作流”的具体心智。Claude Code 在这一点上更产品化。

## 六、Claude Code 可以向 Codex 学什么

### 1. 更明确的协议边界

Codex 的 `protocol` 与 `app-server-protocol` 给了一个很好的方向：当产品越来越多端化时，最好尽早把对外控制面与内部类型边界清楚建模。

### 2. 更明确的多前端共享 runtime

如果 Claude Code 未来继续强化 IDE、SDK、server mode 和后台能力，那么把一部分核心行为从 CLI 产品层抽成更稳定的 runtime 边界，会明显降低扩展成本。

### 3. 更早治理超大核心模块

Codex 虽然也有大核心问题，但它已经通过独立 crate 持续外提低耦合能力。Claude Code 如果要长期演进，越早拆出共享协议、工具模型、状态模型，越不容易陷入巨型单体维护成本。

### 4. 更强的安全基础设施前置

Codex 在 sandbox、execpolicy、跨平台执行控制上的架构意识更重。Claude Code 若继续扩大执行边界和企业场景，这部分值得更系统化。

## 七、对 Codex 的具体优化建议

下面只给对当前 `codex` 仓库的建议，优先考虑产品能力提升。

### 建议 1：强化 TUI 中的“能力导航层”

目标是让用户更快理解 Codex 不只是聊天终端，而是完整 agent 工作台。

可以考虑：

- 把常用模式按任务目标组织，而不是按底层能力组织。
- 在首屏或线程侧栏更明确展示：当前线程状态、可用技能、MCP、插件、审批模式、恢复入口。
- 为 exec、review、resume、fork、app-server 对应能力建立更统一的前台心智。

### 建议 2：把 app-server 优势更明显地反哺到交互产品

现在 Codex 的 app-server 很强，但这种优势更多体现在集成层。可以考虑让用户更直接感知到：

- 当前线程在协议层可做什么。
- 哪些结构化能力仅在 app-server/SDK 中可用。
- 如何把一个 TUI 会话无缝接到 IDE 或脚本中。

### 建议 3：继续降低 `codex-core` 的心智密度

不只是为了架构整洁，更是为了提升迭代速度。

优先级较高的方向：

- 继续把纯工具模型、纯协议适配、纯状态读写迁出 `core`。
- 保持 `core` 聚焦于线程生命周期、agent orchestration 和安全控制。
- 对外维护更清晰的“runtime API map”，方便新开发者定位能力边界。

### 建议 4：增强多 agent / 任务流的用户可观察性

Codex 已经具备多线程、fork、resume、collaboration 等基础，但产品前台的可感知度还可以更强。可以增强：

- 子任务树视图
- agent 间关系图
- 当前执行阶段与等待状态
- 工具调用、审批和回滚的更可读时间线

### 建议 5：把平台能力转换成更清晰的产品叙事

Codex 现在的真实优势是“平台化 agent runtime”。建议在文档、CLI 帮助和 TUI onboarding 中更直接讲清楚：

- 什么时候用 TUI
- 什么时候用 exec
- 什么时候用 app-server/SDK
- 它们之间如何衔接，而不是彼此平行存在

## 八、综合判断

从研究视角看，这两个仓库代表了两种都成立的路线：

- **Codex**：先把 runtime、协议、服务面和安全边界做扎实，再让多个入口共享能力。
- **Claude Code**：先把单产品工作流做到极致，再通过 bridge、remote、plugins、skills 把能力外延出去。

如果站在长期平台建设角度，我更看好 Codex 这种“runtime + protocol + 多前端”的结构，因为它更适合持续扩展到 IDE、SDK、企业集成和服务控制面。

如果站在短中期用户体验角度，Claude Code 这种“CLI 内能力高度集中”的路线能更快形成强烈产品感。

最理想的方向，其实是两者长板结合：

- 保留 Codex 的平台边界、协议建模和安全能力。
- 吸收 Claude Code 在功能可见性、工作流包装和前台体验浓度上的做法。

## 九、路径佐证

本文判断主要基于以下入口文件与文档：

- Codex：`codex-rs/cli/src/main.rs`、`codex-rs/core/src/lib.rs`、`codex-rs/app-server/README.md`、`codex-rs/tools/README.md`、`sdk/typescript/README.md`、`sdk/python/README.md`
- Claude Code：`src/main.tsx`、`src/QueryEngine.ts`、`src/Tool.ts`、`src/tools.ts`、`src/commands.ts`、`read/architecture.md`
