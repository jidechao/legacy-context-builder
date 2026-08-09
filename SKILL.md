---
name: legacy-context-builder
description: 为老项目 / 历史遗留（legacy / 屎山）项目系统性构建 AI 上下文工程，使其变为 Claude Code、Codex、Cursor 等 Coding Agent 可维护的系统；也用于已有 AGENTS.md/文档的项目做过期巡检与漂移修正（知识保鲜）。四层模型：代码层（死代码/幽灵代码识别与架构瘦身）→ 规范层（接口契约与模块边界）→ 知识层（AGENTS.md 分层体系）→ 验证层（自动化测试与 MR 门禁），并在规范层内置 Karpathy AI 编码规则作为代理行为准则（生成 `docs/conventions/agent-rules.md` 并接入根 AGENTS.md）；以 AGENTS.md 为唯一上下文来源、CLAUDE.md 仅作 `@AGENTS.md` 导入桩（兼容 Codex/Cursor 等非 Claude Agent）。当用户接手缺文档的老项目、想让 AI 帮忙维护但 AI 频繁误改、需要给 AI 补足上下文、梳理哪些代码是死的（看着有引用其实运行时不可达）、搭建多 Agent/多人通用的项目知识体系，或怀疑现有上下文文档与代码脱节时，都应使用本 skill——即使没有明说"构建上下文"也适用。区别于：从零搭建新项目、单点重构某个函数、或单纯跑 knip/depcheck。触发词：老项目、遗留项目、屎山、接手老项目、历史债、给 AI 补上下文、AI 上下文工程、AGENTS.md、幽灵代码、死代码梳理、上下文过期、文档和代码对不上。
---

# Legacy Context Builder — 遗留系统 AI 上下文工程构建器

## 核心认知（两篇文章的共同结论）

> AI 在遗留系统中失效的根因不是模型能力，而是**上下文缺口（Context Gap）**。
> 重构的目标不是代码，是上下文。给 AI 足够的上下文，它会给你足够的惊喜。

四类典型上下文缺口（勘察时逐项排查）：

| 上下文类型 | 典型缺失内容 | 对 AI 的影响 |
|---|---|---|
| 业务背景 | 兼容代码对应的历史需求、已下线功能残留 | 误判代码有效性，提出错误删改方案 |
| 架构契约 | 模块间通信协议、状态机约定、接口版本演进 | 生成不符合约定的代码，破坏调用链 |
| 运行时真相 | 哪些分支在真实流量中可达、哪些配置已废弃 | 基于静态引用做错误假设，方案无法运行 |
| 技术债务 | 改造做了一半的模块、临时方案转正的副作用 | 在脆弱地基上继续搭建，引发级联故障 |

## 四层上下文工程模型

```
┌──────────────────────────────────────────────────────────┐
│  L4 验证层 │ 自动化测试 + 视觉回归 + MR 门禁              │
│  L3 知识层 │ AGENTS.md 分层体系 + 代码wiki(内置构建器)    │
│  L2 规范层 │ 接口契约 + 设计约束 + 编码规范               │
│  L1 代码层 │ 死代码清理 + 架构瘦身 + 依赖显化(codegraph)  │
└──────────────────────────────────────────────────────────┘
```

**核心原则：下层是上层的基础，不可跨层建设。** 代码层不清，规范层是空中楼阁；规范层不定，知识层缺乏锚点；验证层不立，AI 产出没有安全网。

能力演进阶梯：AI 可读(Readable) → AI 可写(Writable) → AI 可测(Testable) → AI 可自治(Autonomous)

## 执行工作流

运行本 skill 时按以下阶段执行。**每个阶段结束设置人工确认门禁**——遗留系统的"运行时真相"和"业务背景"无法纯靠静态分析获得，必须与人确认。

### Phase 0：勘察与缺口诊断

1. 盘点项目基本面：技术栈、目录结构、git 历史（`git log --format='%an %ad %s' --date=short | head -100`）、现有文档（README/docs/已有 AGENTS.md、CLAUDE.md）、CI 配置、测试现状。**逐条核对现有文档（尤其 README）中的事实性陈述与代码现实是否矛盾**——文档撒谎是遗留项目的常态，每个矛盾点都是一条高价值上下文，必须写入诊断产物。（首次建设模式下，文档漂移只入报告 catalog、**不就地改原文**——就地修正是巡检模式职责；避免在建设期改动非己方产品文档。）
2. **运行模式判定**：检查项目是否已有 AGENTS.md / CLAUDE.md / docs 上下文体系——
   - **首次建设模式**：无或基本无上下文文档 → 走下方完整 L1-L4 流程
   - **巡检模式（Drift Check）**：已有上下文体系 → 不重建，转入文末"巡检模式"，产出上下文漂移报告并就地修正文档
3. **探测 codegraph 可用性**（见“工具集成”节）：跑 `codegraph --version` 检测是否安装——已安装则 Phase 1 **自动建索引**并用于 L1/L2；未安装则降级为手工静态分析，不阻塞、不强求用户预装、不逐项问授权。代码 wiki 由本 skill **内置构建器**生成，不依赖外部工具。
4. 对照"四类上下文缺口"逐项评估，输出 **缺口诊断报告**（保存为 `docs/context-gap-report.md`）：
   - 每个层级（L1-L4）的现状评分：缺失 / 部分 / 完备
   - 死代码嫌疑清单（零引用符号、环境判断残留、多套并存通信机制）
   - 文档与现实矛盾清单（README/注释/TODO 与代码实际行为的冲突）
   - 建议的建设顺序与工作量估算
5. **门禁**：与用户确认建设范围（全量四层 or 只做 L1+L3 快速见效路径）。

### Phase 1：L1 代码层 — 让代码与运行态一致

目标：消除"幽灵代码"噪音。AI 无法通过静态引用判断代码是否真实运行。

1. **建立代码图谱（自动检测 → 可用即建 → 不可用降级，完整策略见“工具集成”）**：
   - **已安装**（Phase 0 探测命中）：在项目根**后台**跑 `codegraph init` 建索引（`run_in_background`，数百文件约数秒～十几秒，不阻塞勘察）→ `codegraph status` 验证 → 提醒用户把 `.codegraph/` 加 `.gitignore`。之后 L1 全程用图谱：死代码静态扫描走 `codegraph_explore`（有 MCP，传 `projectPath`）或 CLI `codegraph show`/`explore` 查调用图与 blast radius；依赖显化也从图谱导出。
   - **未安装**：降级为 Grep（引用计数）+ Glob（结构盘点）+ Read 手工构建调用链与依赖分析，在 `dependency-map.md` 与缺口报告注明 **“本地未检测到 codegraph，已降级为手工静态分析，置信度较低”**。不报错、不阻塞；后续用户装了 codegraph 可随时 `codegraph init` 补建、用 `codegraph_explore` 复核升级。
2. **死代码三层识别**：
   - 静态扫描：IDE/工具链引用分析 +（有 codegraph 时）codegraph 调用图（blast radius 查零入边）；无 codegraph 时用 grep 引用计数兜底。标记零引用符号
   - 运行时追踪：日志埋点/动态插桩/覆盖率数据，确认路径真实可达性。**遗留项目常无此基建，不可用时不要硬凑**——降级为"静态图谱 + 注释/roadmap 旁证"，在报告中注明"无运行时数据，置信度落到业务确认层"
   - 业务确认（**幽灵代码裁决的主战场，运行时追踪缺失时几乎全靠它**）：对静态无法定性的项，先用"预留 vs 废弃"启发式自判（见 [references/playbooks.md](references/playbooks.md) Playbook 1），判不了的 **标 ⏳ 写入报告、不阻塞流程**，集中到最近的阶段门禁批量请用户确认，AI 记录每项依据
3. **依赖显化**：有 codegraph 时从图谱导出、无则用 Grep+Glob+Read 手工归纳模块依赖关系，生成 `docs/architecture/dependency-map.md`，并在其中注明数据来源（图谱 / 手工）与置信度。
4. **架构瘦身评估**：对每个复杂模块问两个问题——当前业务是否真实依赖这一复杂度？未来 6-12 个月是否有明确需求？产出瘦身建议清单（不执行删除，只出方案）。
5. **落盘**：每确认一块死代码/过度设计，在对应模块 AGENTS.md 记录"某功能已下线，相关兼容代码已移除/将移除"，防止 AI 后续误判。

> **L1 铁律：只诊断、不删除。** L1 阶段对任何"疑似死代码"只产出**带证据的清理建议**，绝不执行删除或修改生产代码（删依赖、删文件、改导出都算）。删除权完整留给后续阶段（L2 收敛 + 门禁人工确认 + L4 测试护栏到位）后再分批执行。
>
> *为什么这条是本 skill 的核心安全纪律*：静态"零引用"**不等于**死代码。遗留系统里，一类典型陷阱是**配置/Provider/注册层**——它可能没有直接 import 者，但它的效果散落在各处。真实反面案例（评估实测）：某 agent 把 React i18n 基础设施（`I18nProvider` + `i18n` 初始化配置 + `LanguageSelector`）当"零引用死代码"删除，因为静态看无人 import 这些文件；但全仓 `useTranslation()` 调用散落于 App/对话框/卡片等十几个组件——删掉 Provider+配置后 i18n 未初始化，应用照常启动，**所有 UI 文案却退回原始 key（静默损坏，无报错无崩溃）**。"看着没引用"的 Provider/配置/注册类符号，必须追**谁在消费它注入的能力**（运行时效果），不能只数 import。运行时可达性与业务意图只有人能确认——这正是三层识别里"运行时追踪 + 业务确认"不可跳过的原因。

### Phase 2：L2 规范层 — 设定约束边界

目标：把隐性团队共识显性化为 AI 可执行的契约。

1. **模块边界梳理**：划分"场景外壳 + 共享能力层"，明确哪些模块禁止隐式状态穿透，产出 `docs/conventions/module-boundaries.md`。
2. **接口契约提取**：有 codegraph 时基于图谱、无则基于 Grep+Read，梳理模块间通信协议、状态机约定、接口版本，产出 `docs/conventions/contracts.md`。
3. **编码规范沉淀**：从现存代码中归纳（而非凭空发明）命名约定、目录组织、错误处理模式，产出 `docs/conventions/coding-style.md`。
4. **代理行为准则落地（Karpathy 编码规则）**：上下文工程解决"AI 看不懂现状"，行为准则解决"AI 守不住纪律"——两者互补，缺一不可。从本 skill 的 [assets/agent-rules.md](assets/agent-rules.md) 取 Karpathy 十条规则，落盘为项目 `docs/conventions/agent-rules.md`，作为 AI 代理行为底线。落地纪律：
   - **先查后写**：项目若已有等价规则文件（`AGENTS-RULES.md` / `docs/conventions/agent-rules.md` / 根 AGENTS.md 全局规则），**不要覆盖**，改为核对差异、合并补充（沿用本 skill"不就地改非己方文档"的纪律——就地修正非己方产品文档属巡检模式职责，建设期只入报告/新建）。
   - **保留原文**：默认保留规则正文（权威、可检索）；团队可在各章追加"本项目补充：…"而不要删改原条目。
   - **速查版入索引**：把十条规则的中文速查精简版写入根 AGENTS.md"全局规则"节，完整版留在 `agent-rules.md`，遵循 Progressive Disclosure（模板见 [references/templates.md](references/templates.md)）。
5. **规范代码化建议**：能 lint 化的给 lint 规则配置，能 token 化的给 token 清单——规范不是文档，而是可检测的约束。（行为准则中可 lint 化的条目，如"禁止未经说明引入新依赖"，也归入此步。）
6. **门禁**：与用户确认规范清单，标注"已有共识"vs"新建约束"，并确认是否采用 Karpathy 规则（默认采用，团队有自有规则可关闭）。

### Phase 3：L3 知识层 — 构建 AI 可读的上下文体系

目标：解决"为什么这么做"。这是 AI 从执行者进化为协作者的关键。

**为什么 AGENTS.md 是唯一来源、CLAUDE.md 只做零内容指针桩**：AGENTS.md 是跨 Coding Agent 的事实标准（Codex、Amp、Cursor、Jules 等均读取），CLAUDE.md 仅 Claude Code 读取。因此**所有 agent 关键内容（技术栈、常用命令、架构契约、运行时真相、已下线清单、落盘规范）必须落在 AGENTS.md 体系**——绝不能让任何关键上下文只存在于 CLAUDE.md（否则 Codex/Cursor 永远看不到，等于没写）。CLAUDE.md 降级为**零内容指针桩** `@AGENTS.md`，**与每个 AGENTS.md 配对放置**（根 + 各模块目录）：内容仍只在 AGENTS.md（单一来源、零漂移）；配对桩的存在仅为让 Claude Code 沿目录层级自动加载模块文档——Claude Code 自动加载 CLAUDE.md 但不自动加载 AGENTS.md，配对桩把模块上下文也纳入自动注入，无需 agent 主动跟索引。桩只一行、不含内容，重命名 AGENTS.md 时同步改路径即可。

1. **生成代码 wiki**（本 skill 内置构建器，不依赖外部 CLI）：按 [references/code-wiki-builder.md](references/code-wiki-builder.md) 的七步工作流，由当前 agent 用 Read/Grep/Glob/Write 直接产出**源码锚定**的代码 wiki 到 `openwiki/` 目录。首次运行先创建 `openwiki/INSTRUCTIONS.md`（wiki brief，模板见 [assets/wiki-instructions-template.md](assets/wiki-instructions-template.md)）校准范围与优先级，再盘点→骨架→证据门禁→填充→QA 自检。方法论源自 openwiki（langchain-ai/openwiki）的 agent 文档工程并已内化——openwiki CLI 可用时作可选加速器，不可用完全不影响产出。这是 L3 的领域知识底座，与 AGENTS.md 分层体系互补（AGENTS.md 是索引与运行必需，wiki 是深度领域知识与"改 X 去哪/验什么"的路径压缩）。
2. **构建 AGENTS.md 分层体系**（Progressive Disclosure 渐进式披露，禁止全部堆根目录）：
   ```
   project-root/
   ├── AGENTS.md            # 唯一来源：技术栈+命令+全局规则+索引（~70-90 行）
   ├── CLAUDE.md            # Claude Code 入口：@AGENTS.md 导入桩 + Claude 专属（< 30 行）
   ├── src/core/
   │   ├── AGENTS.md        # 核心模块：架构契约 + 状态机约定
   │   └── CLAUDE.md        # 指针桩：@AGENTS.md（Claude Code 沿目录层级自动注入）
   ├── src/feature-*/
   │   ├── AGENTS.md        # 业务模块：领域术语 + 链路说明 + 历史决策
   │   └── CLAUDE.md        # 指针桩：@AGENTS.md
   └── docs/                    # runtime-truth / conventions / testing / ops
   ```
   - 根目录 AGENTS.md **放运行必需 + 索引**：顶部技术栈 + 常用命令（任何 agent 第一眼就该看到如何构建/跑测），再全局规则，最后索引（路径 + 1~2 句摘要）。深度细节下沉模块级；**跨模块全局硬知识**（决策/运行时真相/已下线/技术债）若不属于单一模块，落 `docs/runtime-truth.md` 并在根索引指向——不要塞进 CLAUDE.md
   - **全局规则节接入行为准则**：写入 Karpathy 十条规则的中文速查版（一句话/条）并指向 Phase 2 生成的 `docs/conventions/agent-rules.md` 完整版——这是 AI 代理行为底线，与项目上下文并列存于根索引（速查版见 templates.md 根 AGENTS.md 模板）
   - **CLAUDE.md 与每个 AGENTS.md 配对**：根 CLAUDE.md = `@AGENTS.md` 导入桩 + Claude 专属；**每个有 AGENTS.md 的模块目录再放一个零内容 CLAUDE.md 指针桩**（仅 `@AGENTS.md` + 注释），让 Claude Code 沿目录层级自动注入模块文档（Claude Code 自动加载 CLAUDE.md 但不自动加载 AGENTS.md）。桩不含任何内容，防漂移。模板见 templates.md。
   - 模板见 [references/templates.md](references/templates.md)
3. **补齐四类上下文**：对照 Phase 0 缺口报告，把业务背景、架构契约、运行时真相、技术债务分别落盘到最近的模块 AGENTS.md。
4. **写入知识落盘规范**（放进根 AGENTS.md）：
   - 对话中产生的可复用规则/排障结论，必须就近落盘
   - 索引过期时主动修正
   - 变更驱动更新：架构调整/规范变更/接口升级必须在同一 MR 同步更新 AGENTS.md
5. **门禁**：请用户补充"只有人知道"的业务背景和运行时真相——这是缺口报告中标红的部分。

### Phase 4：L4 验证层 — 锁定 AI 产出质量

目标：没有验证层，AI 产出就是不可信黑盒。测试策略核心不是"写多少"，而是"测什么"。

1. **测试工程分期**（详见 [references/playbooks.md](references/playbooks.md)）：
   - 一期：测试骨架（框架 + Mock 环境 + fixtures）
   - 二期：核心链路覆盖（梳理 P0/P1 用例）
   - 三期：复杂场景补齐（多模块联动、异常链路）
   - 四期：MR 流水线集成（失败即阻断合入）
2. **视觉回归**（前端项目适用）：全页截图（宽松像素偏差，抓布局异常）+ 组件级截图（严格偏差，抓精确样式问题），baseline 随仓库存储。
3. **MR 智能门禁建议**：规范校验（AI 评审规则文件）+ 测试校验 + 影响面校验（用 codegraph 依赖图谱分析 blast radius，高风险变更标记人工 review）。
4. **建立上下文漂移巡检**：建议每月运行一次"代码变更 vs AGENTS.md 差异扫描"，生成上下文漂移报告。

### 收尾：产出验收清单

- [ ] `docs/context-gap-report.md` 缺口诊断报告（巡检模式则为 `docs/context-drift-report.md` 漂移报告）
- [ ] codegraph 接入：已装 → `.codegraph/` 索引并用于 L1/L2；未装 → 降级产出 `docs/architecture/dependency-map.md` 手工依赖分析并注明置信度
- [ ] `openwiki/` 代码 wiki（内置构建器产出：quickstart + 任务路由表 + OKF front matter + 概念关系网；详见 references/code-wiki-builder.md）
- [ ] 根 AGENTS.md（技术栈+命令+全局规则+索引，~70-90 行，**唯一来源**）+ CLAUDE.md（`@AGENTS.md` 导入桩 <30 行）+ 模块级 AGENTS.md，**且每个有 AGENTS.md 的目录都配一个零内容 CLAUDE.md 指针桩（`@AGENTS.md`）——桩数 = AGENTS.md 数**
- [ ] 跨模块全局硬知识落 `docs/runtime-truth.md`（决策/运行时真相/已下线/技术债），根索引指向；**无任何关键内容仅存于 CLAUDE.md**
- [ ] `docs/conventions/` 规范文档组（module-boundaries / contracts / coding-style / **agent-rules** — Karpathy AI 行为准则，并已在根 AGENTS.md 全局规则节接入）
- [ ] 测试骨架 + P0/P1 用例清单 + MR 门禁配置建议
- [ ] 上下文保鲜机制（落盘规范 + 巡检节奏）已写入根 AGENTS.md
- [ ] 能力阶段评估：当前处于 Readable / Writable / Testable / Autonomous 哪一级

## 巡检模式（重复运行 / 知识保鲜）

上下文的最大敌人是腐烂。当项目已有上下文体系（Phase 0 第 2 步判定）时再次运行本 skill，进入巡检模式——目标不是重建，而是保鲜：

1. **收集变更证据**：`git log --oneline --since="<上次巡检日期>"` 与 `git diff`（无 git 仓库则对比文档与源码的修改时间），圈定变更范围。
2. **逐项核对文档事实性陈述**：现有 AGENTS.md / CLAUDE.md / docs 中的每条"事实"（架构契约、已下线清单、运行时真相、构建命令、规范条目、索引链接）在当前代码中是否仍成立。特别关注：索引指向的文件是否还存在、"已下线"清单是否有新增、命令是否还能跑。另核对根 AGENTS.md 全局规则节是否仍指向 `agent-rules.md`、该文件是否存在且未被误删。
3. **产出漂移报告** `docs/context-drift-report.md`：过期项（文档与代码矛盾）/ 缺失项（新代码无文档覆盖）/ 健康项（核对无误），模板见 [references/templates.md](references/templates.md) 末节。
4. **就地修正**：按"变更驱动更新"原则直接修正文档，报告里记录每处修正的依据（哪个 commit / 哪段代码）。
5. **代码 wiki 增量刷新**：若项目有 `openwiki/`，按 [references/code-wiki-builder.md](references/code-wiki-builder.md) §9 更新纪律增量刷新——读 `.last-update.json` 取上次 gitHead，`git log <head>..HEAD --name-status` 算变更窗口，建 docs impact plan，只改被波及的页，无变更则 no-op。openwiki CLI 可用时亦可 `openwiki code --update --print` 替代。
6. **门禁**：涉及"运行时真相"类过期（如某开关线上值变了）无法从代码证实的，列入人工确认清单，不擅自改写。

巡检模式也应该出现在每个项目的保鲜机制设计中：建议用户把"每月跑一次本 skill 巡检"固化为团队节奏。

## 工具集成

### codegraph（colbymchenry/codegraph）— L1/L2 的图谱引擎

本地优先的语义代码图谱：符号、调用边、依赖关系一次索引，替代 grep+Read 逐文件爬行。

**接入策略：自动探测 → 可用即建索引 → 不可用降级。** 不阻塞、不强求用户预装、不逐项问授权——codegraph 能显著提升 L1/L2 精度，但本 skill 在任何环境都能跑通：装了就用、没装就降级。

**① 探测（Phase 0，非破坏性）**：跑 `codegraph --version`（Windows/Unix 通用）。返回版本号 = 已安装；命令未找到 / 非零退出 = 未安装。把结论（已安装 / 未安装 / 失败原因）记入缺口报告。

**② 已安装 → 自动建索引 + 使用**：

```bash
codegraph --version      # 探测（非破坏）
codegraph init           # 项目根建 .codegraph/ 索引（后台跑，不阻塞勘察）
codegraph status         # 验证索引
codegraph show <sym>     # 查看符号定义与源码（-f 消歧义）
codegraph explore "…"    # 一次取相关符号源码 + 调用路径 + blast radius
```

- 在目标项目根执行；`codegraph init` 以项目根为 cwd，不能简单 `cd` 时用子 shell 或工具的路径参数。
- 索引数百文件约数秒～十几秒，**用后台任务跑（`run_in_background`）** 避免阻塞勘察，完成后 `codegraph status` 校验。
- 建索引后 L1/L2 全程用图谱：**有 codegraph MCP 优先 `codegraph_explore`**（传 `projectPath` 指向该项目）一次获取符号源码 + 调用路径 + blast radius；无 MCP 用 CLI `codegraph show <sym>` / `codegraph explore`。
- `.codegraph/` 是本地产物，**主动提醒用户加 `.gitignore`**（不要提交）。

**③ 未安装 → 降级手工静态分析**（不报错、不阻塞、不强求安装）：

- 用 Grep（引用计数）+ Glob（结构盘点）+ Read 手工构建调用链与依赖分析，动态分派（工厂/反射）手工追踪。
- 在 `dependency-map.md` 与缺口报告注明 **“本地未检测到 codegraph，已降级为手工静态分析，置信度较低”**。
- 用户后续装了 codegraph，可随时 `codegraph init` 补建、用 `codegraph_explore` 复核升级既有产物。

> 探测或 `codegraph init` 失败（权限、原生依赖缺失、路径含特殊字符等）按“未安装”降级处理，在报告记录失败原因，不要卡住流程。

### 代码 wiki — L3 的知识库（本 skill 内置构建器）

L3 的领域知识库由**本 skill 内置的代码 wiki 构建器**生成，不依赖任何外部 CLI。方法论源自 openwiki（langchain-ai/openwiki）的 agent 文档工程，完整流程见 [references/code-wiki-builder.md](references/code-wiki-builder.md)：先骨架后填充、证据门禁、路径压缩 + 任务路由表、OKF front matter、概念关系网、Mermaid、增量更新、QA 自检。产出落仓库 `openwiki/` 目录，采用 OKF v0.1 格式。

**为什么内置而非外部工具**：接手遗留项目时环境往往不齐（openwiki CLI 未装 / 网络受限 / 模型 provider 未配），而代码 wiki 是 L3 的关键产出——不能因缺一个工具卡住。内置构建器把方法论变成 agent 可直接执行的步骤，Read/Grep/Glob/Write 即可落地。这也是本 skill 与 openwiki 的分工：openwiki 是通用 wiki 引擎，本 skill 的内置构建器专为"遗留系统 AI 上下文工程"调校，与 L1 代码层（死代码/依赖图）、L2 规范层、AGENTS.md 体系联动。

**openwiki CLI 作可选加速器**：环境已装 openwiki 时可用它替代内置流程，产出格式完全兼容（同样的 `openwiki/` 目录、OKF front matter、`.last-update.json`），日后装上也能无缝接力刷新已有 wiki：

```bash
openwiki --init                 # 从零生成（等价内置构建器首次生成）
openwiki code --update --print  # 增量刷新（等价内置 §9 更新纪律）
```

并提供 CI 模板（定时跑 update 自动开文档 PR）。但这**不是必需的**——内置构建器在任何环境都能产出同等质量的 wiki。

## 度量指标（建设前后对比，写入验收报告）

| 指标 | 测量方式 |
|---|---|
| AI 方案一次通过率 | 抽样 10 个真实任务，记录方案无需人工纠正即通过的比例（目标 >85%） |
| 核心链路发布故障率 | 发版后核心链路问题数 |
| 需求交付周期 | AI 自主完成比例（目标 70%+） |
| 回归测试耗时 | 人工半天 → 流水线分钟级 |

## 关键纪律

1. **不可跨层建设**——L1 未清不动 L2，L2 未定不动 L3。
2. **模糊地带必须问人**——运行时真相和业务背景是 AI 无法自行获取的，宁可多问不可幻觉。
3. **规范从代码归纳，不凭空发明**——遗留系统的规范首先是描述现状，其次才是约束未来。
4. **知识就近落盘**——索引在根，细节在模块，过期即修。
5. **思考留给人**——落地执行、记录沉淀交给 AI；架构判断和模糊决策必须人工确认。

详细方法论背景见 [references/methodology.md](references/methodology.md)；模板见 [references/templates.md](references/templates.md)；分期执行细则见 [references/playbooks.md](references/playbooks.md)；AI 代理行为准则素材见 [assets/agent-rules.md](assets/agent-rules.md)。
