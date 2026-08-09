# 内置代码 Wiki 构建器（Code Wiki Builder）

> 本文件是 L3 知识层"领域知识库"的生成器。方法论源自 [openwiki](https://github.com/langchain-ai/openwiki) 的 agent 文档工程（`src/agent/prompts/code.ts`），内化为本 skill 的**自包含流程**——不依赖任何外部 CLI，直接由当前 agent 用 Read/Grep/Glob/Write 产出。
>
> 产出落在目标仓库的 `openwiki/` 目录，采用 OKF v0.1 front matter。**日后若安装了 openwiki CLI，可无缝接力刷新**（front matter、`.last-update.json` 完全兼容），已有产出不浪费。
>
> 与 [templates.md](templates.md) 的代码 wiki 模板组配合使用：本文件讲**怎么建**，templates 讲**长什么样**。

## 目录

- [0. 设计哲学：为什么这样建](#0-设计哲学为什么这样建)
- [1. 产出结构与 OKF](#1-产出结构与-okf)
- [2. 七步构建工作流（首次生成）](#2-七步构建工作流首次生成)
- [3. 证据门禁（写每页前的必检项）](#3-证据门禁写每页前的必检项)
- [4. 骨架评审清单（skeleton critic）](#4-骨架评审清单skeleton-critic)
- [5. 任务路由表（quickstart 的灵魂）](#5-任务路由表quickstart-的灵魂)
- [6. OKF front matter](#6-okf-front-matter)
- [7. 概念链接纪律](#7-概念链接纪律)
- [8. Mermaid 图表](#8-mermaid-图表)
- [9. 更新纪律（增量刷新）](#9-更新纪律增量刷新)
- [10. QA 自检](#10-qa-自检)
- [11. 安全规则](#11-安全规则)
- [12. wiki brief（INSTRUCTIONS.md）](#12-wiki-briefinstructionsmd)

---

## 0. 设计哲学：为什么这样建

三条原则贯穿全流程。理解了 why，步骤才有判断力，遇到没见过的项目结构也能自己调整。

**① 源码锚定（Source-grounded）**——每条重要论断必须有源文件/测试/docs/git 证据，不臆造文件、模块、API、行为。
*为什么*：遗留项目最大的坑是"看着有引用、运行时不可达"。只有锚定源码才能区分真相与噪音；臆造一条，就会误导后续每一个 agent 在错误前提上继续搭建（级联故障）。

**② 路径压缩（Path compression）**——wiki 的首要优化目标是缩短"工程意图 → 拥有文件/符号/相关系统/focused tests/最窄验证命令"的路径。
*为什么*：wiki 是给 agent 当**记忆**用的。它要的不是"读懂项目"，而是"改 X 时去哪、验什么、踩什么坑"。少一次仓库级搜索就是实打实提效。这也是为什么 quickstart 必须有任务路由表（见 §5）。

**③ 概念即节点、链接即边（Concept-as-node, link-as-edge）**——标准 Markdown 链接表达概念间的**运行时关系**，不是导航装饰。
*为什么*：关系网络让 agent 能从一个概念跳到相关概念，比平铺目录更接近"理解"。孤立的一页 wiki 价值有限；连成网才有复利。

---

## 1. 产出结构与 OKF

```
openwiki/
├── INSTRUCTIONS.md        # wiki brief：用户撰写的范围/优先级（构建器读但不重写，见 §12）
├── quickstart.md          # 入口：总览 + 任务路由表 + 指向各主要概念
├── .last-update.json      # 上次成功刷新的 gitHead（增量更新用，见 §9）
├── _skeleton.md           # 临时骨架文件，构建完成后删除
├── architecture/
│   └── overview.md        # 运行时结构、主要模块、执行流
├── domain/                # 业务概念页（按真实领域边界，非目录树）
├── workflows/             # 运行时链路（请求流/生命周期/状态机）
├── operations/            # runbook：部署/配置/凭据/排障
├── integrations/          # 外部对接点（第三方服务、SDK）
└── testing.md             # 测试策略 + focused tests 索引（可选独立页）
```

**关键纪律**：
- 目录边界跟随仓库的**真实组件/领域**，不要镜像目录树（目录树用 `tree` 就能看，wiki 要提供目录树给不了的"关系与意图"）。
- 每个 substantive 服务 / API / 工作流必须有**专属页或目录**；不要把所有东西塞进一个文件，也不要每个文件一页。
- 多个服务组成一个大系统时，建一个目录收纳，目录内每个服务一页。
- `index.md`、`log.md` 是 OKF 保留文档，不给概念 front matter；目录索引由后续维护时手动或脚本生成（本构建器不强求，quickstart 已是事实入口）。

---

## 2. 七步构建工作流（首次生成）

**严格按序**。每步有明确产出，不跳步——跳步的典型后果是写出一堆只有 manifest 级理解的薄页。

### 步骤 1：盘点清单（inventory）→ 写骨架
用 Grep/Glob/Read 盘点（不写正文，只收集）：
- manifest-backed 的服务 / app / package / workspace（package.json、Cargo.toml、pyproject.toml、go.mod 等）
- runtime 与 build entrypoints（main、cli、server 入口、路由注册）
- public surfaces（导出的 API、对外契约）
- 主要 domain（业务领域划分）
- data / schema ownership（谁拥有哪个数据模型/表/状态）
- operational services（部署、定时任务、消息消费）
- 现有 docs（README、docs/、已有 AGENTS.md——用于**发现意图与术语**，不作实现证据）
- 代表性 tests

**产出** → `openwiki/_skeleton.md`：计划写哪些页、每页记什么、互相怎么连。这是骨架，不是正文。

### 步骤 2：排序
按 ①运行时重要性 ②依赖中心性 ③近期变更活跃度（git log）④public surface ⑤test 占有 排序。排序**只决定探索顺序，不决定是否覆盖**——任何 substantive 组件都要覆盖，再小也得有归属。

### 步骤 3：聚类成系统
用 import / symbol / 运行时调用 / 共享数据 / tests / git history 把相关文件聚成"系统"与"跨系统工作流"。**禁止把目录树复制进 wiki**——那是文件清单，不是理解。

### 步骤 4：完成骨架
在 `_skeleton.md` 定下**完整** wiki 结构（目录 + 文件 + 每页计划内容）。确保每个 substantive 服务 / API 端点 / 主要工作流都在结构里。
> 自检：一个 agent 或人，能否**仅靠这套 wiki** 就完整理解仓库的系统与工作流？不能 → 结构不全，回去补。

### 步骤 5：骨架评审（skeleton critic）
对着 [§4 骨架评审清单](#4-骨架评审清单skeleton-critic) 逐条自审，每条问题建一个 TODO 并解决。这是内化版的 `skeleton_critic` 子代理——openwiki 用独立子代理做，这里用结构化自审清单达到同等效果。

### 步骤 6：满足证据门禁后填充每页
**先收集证据、后写正文**。对每页执行 [§3 证据门禁](#3-证据门禁写每页前的必检项)，门禁未过不写。
> 最容易踩的坑：读完 manifest / README / 目录列表 / 头几行 import 就开始写——这些只是 discovery 线索，**不是实现证据**。必须在源码里追到真实行为、调用边界、失败用例才能写。

### 步骤 7：unknown-unknown 扫描 + reconcile
写完后扫三类盲区：①未被覆盖的 manifest-backed / 高优 cluster；②未引用的"一跳依赖"（某页讲的主角的直接上下游是否都有交代）；③写作过程中暴露的跨系统工作流。发现真实缺口就扩大计划补上。
最后把最终 wiki 树对回完整 inventory 核对：覆盖率 / 源码锚定 / 术语一致 / 导航 / 关系链接。

### 步骤 8：写 quickstart.md（最后写）
入口页：高层总览 + 指向每个主要概念的链接 + **任务路由表**（§5）。最后写是因为它要索引全部已完成的页。

### 步骤 9：QA 自检
执行 [§10 QA 自检](#10-qa-自检)，修掉答不全的页。

**完成后删除 `_skeleton.md`**（它是临时脚手架，不进版本库的话加进 `.gitignore` 或直接删）。

---

## 3. 证据门禁（写每页前的必检项）

对每个主要 component / domain，**写正文前**必须用 Read 检视以下证据。少一项都会导致"看着对、实际错"的页面。

- [ ] runtime entrypoint 与注册/组合面（这个组件从哪里被启动/注册）
- [ ] entrypoint 背后的**主实现**（不止读签名，读实现体）
- [ ] 重要 public types / schemas / config
- [ ] 持久化 / 缓存 / 队列 / 状态管理代码
- [ ] **≥1 上游 caller + ≥1 下游 dependency**（跨至少一个调用边界，双向各追一跳）
- [ ] 代表性 focused tests——含断言与失败用例（测试是理解"作者到底在意什么"的最佳证据）
- [ ] 相关生成契约 / ops 配置 / migration（如适用）

**明确不算实现证据**（只能当 discovery 线索）：manifest、README、目录列表、import 语句、composition root 开头几行。

> 落地判断：读完这一组证据，你能否解释"这个组件做什么、为什么存在、谁依赖它、改它会牵动什么、怎么验证"？能 → 可以写；不能 → 继续读源码。

---

## 4. 骨架评审清单（skeleton critic）

步骤 5 逐条核对。每条不满足就建 TODO 修复后再进入步骤 6。

- [ ] 每个 manifest-backed 服务 / app / package / workspace 都有专属页或明确归属？
- [ ] 是否有 substantive 工作流没有任何页覆盖？（请求流、生命周期、关键状态机）
- [ ] 结构是按**领域/系统**聚类的，还是照搬了目录树？（后者要重做）
- [ ] 每页是否都有明确的"这页记什么"的一句话描述？
- [ ] quickstart 的任务路由表能否覆盖项目的主要变更意图类别？
- [ ] 是否存在 thin stub（只列文件名、无实质解释）？合并或充实。
- [ ] 概念之间的关系是否在骨架里就标出了 link（避免写完才发现是孤岛）？

---

## 5. 任务路由表（quickstart 的灵魂）

`quickstart.md` 必须含一张紧凑路由表。这是 wiki 对 agent 最有价值的部分——把"我想改 X"直接映射到"去哪、动什么、验什么"。

| 变更意图 | 相关 wiki 页 | source entry points | 关键 symbols/types | focused tests | 最小验证命令 |
|---|---|---|---|---|---|
| 加一个 API 端点 | workflows/request-flow.md | src/server/routes.ts | registerRoute, Router | tests/routes/*.test.ts | `pnpm test routes` |
| 改领域模型 | domain/order.md | src/domain/order.ts | Order, OrderStatus | tests/domain/order.test.ts | `pnpm test order` |

**纪律**：
- 路由的是**真实存在的变更类别**（由源码/历史支持），不是臆想的一次性需求。
- source 用**稳定路径与符号名**，不用行号（行号会漂移）。
- 验证命令给**最窄的非破坏性检查**（focused test / 单模块构建），不是全量；把昂贵的集成/发布检查单独标注为"条件性"。

---

## 6. OKF front matter

每个**概念页**（非 `index.md` / `log.md`）顶部必须有 OKF v0.1 YAML front matter。`type` 是唯一必填，其余按需。

```yaml
---
type: Service                 # 必填，自由概念类：Service / API Endpoint / Domain Concept / Workflow / Runbook / Data Model …
title: 订单服务               # 可选，显示名
description: 处理订单全生命周期的核心服务，含状态机与定价回退。  # 1-2 句，检索优化
resource: src/domain/order.ts # 可选，底层资产 canonical 路径
tags: [order, state-machine, pricing]  # 可选，领域 facet
timestamp: 2026-08-09         # 可选，最近一次有意义变更
# openwiki 扩展（evidence-backed 时才填，不要编）
openwiki:
  roles: [domain, workflow]           # architecture/delivery/domain/integration/operations/repository/testing/workflow
  source_paths: [src/domain/order.ts]
  symbols: [Order, OrderStatus, calcPrice]
  test_paths: [tests/domain/order.test.ts]
  invariants: [订单状态只能单向前进，禁止从已完成回退]
  validation_commands: [pnpm test order]
---
```

- `type` 是自由的人类概念类，不限定固定枚举。
- `description` 对检索最有用，写清楚、写具体。
- `openwiki` 扩展是 **evidence-backed 的路由元数据**（有源码证据才填），不是凑数要求。**绝不**把 secret / 暴露 secret 的命令放进元数据。
- 每个 invariant 尽量链接到能证明它的最窄测试。

---

## 7. 概念链接纪律

标准 Markdown 链接 = 关系边。链接要**落在解释关系的那句话里**，用周围文字说明关系含义：

> 订单服务 [`dispatches to`](domain/pricing.md) 定价模块计算最终价格；订单状态机 [`is persisted by`](domain/order-store.md) 订单仓库。

- 用动词短语表达关系：`dispatches to` / `depends on` / `shares infrastructure with` / `is configured through` / `is surfaced by` / `is secured by`。
- 每个 substantive 概念至少 **2 条 grounded 关系链接**。某页仍孤立 → 补真实关系、并入更宽概念、或解释它为何真的独立。
- **不要**为增图密度造链接，**不要**自动加反向链接（反向链接只在有助于理解目标概念时才加）。
- 一个概念一个 **canonical 页**，别处用链接而非复制正文。
- quickstart 指向所有主要概念的链接算导航，不计入关系审计。

---

## 8. Mermaid 图表

图比文字更清晰时才加——少而精，不为装饰。

| 场景 | 图类型 |
|---|---|
| 请求 / 运行时流 | `sequenceDiagram` |
| 生命周期 / 状态机 | `stateDiagram-v2` |
| 数据模型 | `erDiagram` |
| 控制流分支 | `flowchart` |

**纪律**：
- 每个参与者 / 状态 / 实体 / 关系必须由**检视过的源码**支撑，不凭空发明。
- 加在"讲请求流 / 调用序列 / 生命周期 / 数据模型"的页上；导航页、参考表、配置页跳过。
- 写完自检语法（缩进、`end` 配对、节点名无特殊字符）。若不确定语法，宁可退化为带缩进的文字描述，也不要留坏图——坏图比没图更糟。

---

## 9. 更新纪律（增量刷新）

巡检模式 / wiki 保鲜时用。**核心是增量、低 churn**：只改被变更波及的页。

1. 读 `openwiki/.last-update.json` 取上次 `gitHead`（无则按首次生成处理）。
2. `git rev-parse HEAD` 取当前 commit。
3. `git log <lastHead>..HEAD --name-status --oneline` 圈定变更窗口（无 git 则对比源码与 wiki 的修改时间）。
4. 建 **docs impact plan**：`source change → docs affected → edit needed → why`。无法 tie 到某项源码/工作流/产品/文档变更的页，**不动**。
5. 只改需要改的页。**无相关变更 → no-op**（不动文件，明确说"wiki 已最新"）——这避免定时任务的无意义 churn。
6. **不做 formatting-only 编辑**（不改表格格式、不规整空行、不润色措辞），除非该处本来就要为准确性改动。
7. **Backlog**：evidence-blocked 的延期项进 `quickstart.md` 的 `## Backlog` 节，每条带 source anchor + 一句原因；证据够了就补完并移出 backlog，不让 backlog 静默膨胀。
8. **刷新成功且内容确有变更 → 必须更新 `.last-update.json`**：把 `gitHead` 设为 `git rev-parse HEAD` 的**当前值**、`timestamp` 设为本次刷新时间。这是下次增量刷新的**基线锚点**——漏掉它，下次巡检会以旧 head 算变更窗口、误判"无变更"而跳过本该刷新的页（静默漏更）。**即便是降级路径**（上一次 `gitHead` 是占位哈希 / 不在 git 历史中、改为按源码逐条核对），核对 + 修正完成后**仍必须 stamp 当前 HEAD**，让基线从此闭合，不再悬空。

> 与 SKILL.md 巡检模式的关系：巡检模式产出 `docs/context-drift-report.md`（核对 AGENTS.md 体系的事实性漂移）；本节是 `openwiki/` 代码 wiki 的增量刷新。两者互补，可同一次巡检一并做。

---

## 10. QA 自检

步骤 9 用。内化版 `wiki_question_finder` + `wiki_answer_verifier`（openwiki 用两个子代理，这里用结构化自检）。

1. **自拟 5–10 个问题**：模拟未来 agent 真实会问的——以"我想改 / 排查 / 扩展 X"为主，覆盖主要变更意图。例如："加一个新的 provider 要动哪些文件、注册在哪、怎么测？""订单卡在 PENDING 是哪个状态机分支？"
2. **逐个验证 wiki 能否回答**，每个判 PASS / PARTIAL / FAIL：
   - PASS：仅靠 wiki（不必读 wiki 外源码）即可定位文件、symbols、关系、验证方式。
   - PARTIAL / FAIL：记下缺什么 → 回对应页补 → 重验该问题，直到 PASS。
3. **优先批量修完一批再重验**，不要修一个验一个。

这是质量门：QA 通不过的 wiki 不算完成。

**收尾硬检查（写完 wiki / 增量刷新后必过）**：
- [ ] **`.last-update.json` 已闭合**：本次若有任何内容变更，`gitHead` 是否已 stamp 为 `git rev-parse HEAD` 当前值？（漏 stamp = 下次巡检以旧 head 算窗口、静默跳过本该刷新的页——这是 §9 step8 最易漏的一步，务必逐次核对。）

---

## 11. 安全规则

- **不读、不文档化** secret / credential / token / private key / `.env`。
- `.env.example` 等示例配置**仅在含占位符**时可读；含真实密钥则跳过。
- secret-bearing 文件若相关，只记"此类配置存在、非敏感设置应在哪里描述"，不记值。

---

## 12. wiki brief（INSTRUCTIONS.md）

构建器**首次运行**时创建 `openwiki/INSTRUCTIONS.md`——一份用户撰写的范围与优先级简报。构建器**读它但不重写它**（它控制元数据，不是生成文档）。

模板见 [../assets/wiki-instructions-template.md](../assets/wiki-instructions-template.md)。默认涵盖：项目一句话定位、必须覆盖的领域、优先级（哪些子系统最重要）、风格偏好（语言、详略）。

后续 init / update 运行都先读 INSTRUCTIONS.md 校准范围；用户改它即可调整 wiki 取向，无需改 skill。
