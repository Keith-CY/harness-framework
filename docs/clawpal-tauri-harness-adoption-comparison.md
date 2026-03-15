# ClawPal 落地 Tauri Harness 规范的两种路径对比

更新时间：2026-03-16

## 0. 执行摘要

- 决策问题：如果要让 `lay2dev/clawpal` 实践一套 Tauri Harness 规范，应该优先在原仓库改造，还是让 AI Agent 从零重启新项目。
- 我的结论：优先选“原仓库改造”，把 AI Agent 用在归档、拆分、补约束和补 runbook 上，而不是先整体重写。
- 判断置信度：中高。
- 主要原因：`clawpal` 已经有真实产品复杂度、测试链路、release 流程和 agent-first 架构收口方向，缺的是规范化和边界治理，不是从零发明产品。
- 前提假设：默认投入为 1 位熟悉 Tauri/Rust/TypeScript 的负责人，配合 AI coding agent；目标是让仓库达到可持续 harness 状态，而不是同步做大规模产品重设计。

## 1. 分析基线

本分析的主基线是：

- `docs/tauri-harness-system-design.md`
- `docs/harness-engineering-executable-playbook.md`
- `docs/harness-engineering-summary-outline.md`

其中：

- `docs/tauri-harness-system-design.md` 提供了 Tauri 项目专用的 harness 设计要求
- 另外两份文档提供更上层的 Harness Engineering 背景和落地原则

因此，本文不是泛泛讨论“AI 时代怎么做工程”，而是按 Tauri 项目的 harness 目标来判断 `clawpal` 的两条演进路径。

这套基线的核心要求可以归纳为：

1. 仓库要有稳定的 agent 入口文档和固定文档结构。
2. 文档要成为 system of record，而不是零散补充。
3. 运行、调试、验证、回滚要有明确 runbook。
4. 架构边界要同时体现在文档和自动化机制里。
5. 任务要小步交付，并且每次交付都附验证证据。
6. 对 Tauri 项目而言，还应明确 UI / Command / Domain / Adapter 四层边界，并补齐 contract test、packaged app smoke test、artifact 收集入口。

### 1.1 评估方法与范围

本次判断基于 2026-03-16 对以下材料的静态审阅，而不是本地运行 `clawpal`：

- GitHub 仓库元数据、文件树、语言分布
- `README.md`
- `docs/tauri-harness-system-design.md`
- `agents.md`
- `design.md`
- `cc.md`
- `cc-architecture-refactor-v1.md`
- `docs/testing/business-flow-test-matrix.md`
- `.github/workflows/ci.yml`
- `.github/workflows/e2e.yml`
- `.github/workflows/pr-build.yml`
- 代表性源码文件体量，例如 `src/App.tsx`、`src-tauri/src/commands/mod.rs`

因此，本文对“结构成熟度”和“改造工作量”的判断是方向性估算，不是经过本地构建与实机回归后得出的精确结论。

### 1.2 轻量评估尺子

为了避免“高/中/低”过于主观，这里用一个 6 项 checklist 判断仓库离 Tauri harness 目标有多近：

| 维度 | 基线要求 | ClawPal 当前判断 |
|------|----------|------------------|
| Agent 入口 | 有固定入口文档，agent 能先读什么一目了然 | 部分满足：有 `agents.md`，但未标准化成 `AGENTS.md` |
| 统一命令入口 | 有 `justfile` / `cargo xtask` 一类的固定入口 | 缺口明显：当前以 README、脚本、workflow 为主，缺统一本地入口 |
| 文档归档 | 结构化的 architecture / decisions / plans / runbooks | 部分满足：`docs/plans/` 很强，architecture/runbook/decision 分散 |
| 验证链路 | 启动、测试、E2E、发布有稳定命令和 gate | 满足度较高：CI/E2E/coverage/release 已存在 |
| 边界清晰度 | UI / Command / Domain / Adapter 分层清晰 | 部分满足：已有 core/cli/gui 分层，但 command 层仍偏重，且存在大文件和集中路由 |
| Tauri 专项验证 | command contract tests、packaged smoke、artifact 收集 | 部分满足：已有 CLI JSON contract 和打包流程，但缺显式 packaged smoke 与统一 artifact 入口 |
| 证据式交付 | 任务、PR、回滚、失败处理有固定证据格式 | 部分满足：已有 test matrix 和 release 文档，但仓库级模板和入口仍未归一 |

## 2. ClawPal 当前状态摘要

截至 2026 年 3 月 16 日，对 `https://github.com/lay2dev/clawpal` 的判断是：

先用一句话定义对象：`clawpal` 是一个基于 Tauri 的 OpenClaw 桌面伴侣应用，覆盖安装、配置、Doctor、回滚、远程 SSH 管理和多平台打包发布，不是一个简单的单机 GUI 小工具。

### 已经具备的基础

- 不是从零开始的仓库，已经有明确产品和真实发布流程。
- 已有 `agents.md`、`docs/plans/`、`docs/testing/business-flow-test-matrix.md`、`prompts/README.md`。
- 已有 CI、E2E、coverage、PR build、release workflow。
- 已有明显的 agent-first 架构演进痕迹：
  - `clawpal-core`
  - `clawpal-cli`
  - `src-tauri`
  - `GUI -> CLI/Core -> Agent fallback` 的三层设计文档
- 已有针对 agent 工具边界和 prompt 的约束，而不是完全放任 arbitrary shell。

### 与目标规范仍有差距的地方

- 缺少规范化的 `AGENTS.md` 入口，当前是小写 `agents.md`。
  这不是纯命名洁癖。很多 agent 工作流默认把 `AGENTS.md` 当作仓库入口，小写文件能否被自动发现并不稳定。
- 缺少清晰归档的 `docs/architecture/`、`docs/decisions/`、`docs/runbooks/` 目录，现有信息分散在：
  - `design.md`
  - `cc.md`
  - `cc-architecture-refactor-v1.md`
  - `cc-ssh-refactor-v1.md`
  - `docs/plans/`
- 缺少 `tauri-harness-system-design.md` 推荐的统一命令入口，例如 `justfile` 或 `cargo xtask`。
- 缺少显式的 `harness/fixtures/`、`harness/artifacts/` 之类目录来承接最小 fixture 与失败产物。
- 代码边界还不够“agent 可读”：
  - `src/App.tsx` 约 1,787 行
  - `src-tauri/src/commands/mod.rs` 约 10,546 行
  大文件本身不是罪，但它通常意味着 agent 需要在更大上下文里做修改，误伤范围更大；而 `cc*.md` 里也能看到持续拆分和收口仍在进行。
- 工具链约定不完全统一：README 以 Bun 为主，但 PR build 仍使用 `npm ci` 且保留 `package-lock.json`。
- 已有 contract test 雏形和打包验证，但还没有形成 `tauri-harness-system-design.md` 里强调的“command contract + packaged app smoke + artifacts”三件套。

结论先说在前面：`clawpal` 已经有相当多 harness 脚手架，但还没把这些脚手架归一成一个稳定、低歧义、可持续的 agent 工作入口。

## 3. 路径一：在原有 codebase 上改造

### 可行性

高。

原因不是“现状完美”，而是它已经具备三件最贵的东西：

1. 真实业务边界已经跑出来了。
2. 真实验证链路已经存在了。
3. 真实架构方向已经开始收敛了。

也就是说，现在缺的主要不是“产品发明”，而是“把已有结构整理成一套稳定 harness”。

### 难度

中高。

难点不在功能实现，而在“归一化”：

- 把分散文档归档成固定目录和固定入口
- 把当前隐含的架构约束写成可见规则
- 继续拆大文件，降低 agent 修改时的误伤半径
- 统一验证命令、证据格式、PR 入口、任务模板

这类工作比从零搭脚手架更枯燥，但风险更低，收益也更直接。

### 工作量

如果目标是“真正实践这套规范”，不是只补文档，在“1 位熟悉 Tauri/Rust/TypeScript 的负责人 + AI agent”这个默认投入下，我会按两个口径估算：

- 最小合规版：约 1.5 到 2.5 工程周
- 可持续运行版：约 4 到 6 工程周

建议拆成四个阶段：

#### Phase 1: 仓库入口归一

约 2 到 4 天。

- 新增正式 `AGENTS.md`
- 建立 `docs/architecture/`
- 建立 `docs/decisions/`
- 建立 `docs/runbooks/`
- 把现有 `design.md`、`cc*.md`、`docs/plans/*` 重新分类

#### Phase 2: 验证与流程归一

约 3 到 5 天。

- 落地 `justfile` 或 `cargo xtask`
- 统一本地开发、构建、测试、打包命令
- 统一 Bun / npm 策略
- 增加任务模板、PR 模板、验证证据模板
- 把 `docs/testing/business-flow-test-matrix.md` 升级为标准 gate 文档
- 补 `packaged app smoke` 和 `artifacts` 汇总入口

#### Phase 3: 代码可读性和 agent legibility 改造

约 1.5 到 3 周。

- 拆分 `src/App.tsx`
- 继续拆 `src-tauri/src/commands/mod.rs`
- 进一步收口 GUI / core / remote helper 的边界
- 为高风险模块补 architecture note 和 change guide

#### Phase 4: 把标准写进机制

约 4 到 7 天。

- 让 PR 必须附测试/截图/日志证据
- 对关键目录加 ownership / review rule
- 对高风险调用链加静态检查或约束测试
- 为 runbook 增加失败诊断路径和回滚路径

### 优点

- 保留当前产品积累、平台兼容经验和发布流程。
- 可以避免为 Local / Docker / WSL2 / Remote SSH / Doctor / Rollback 这些真实复杂度重新付一遍学习和验证成本。
- 可以边改造 harness 边继续发版本。
- AI agent 可以立刻介入做分阶段整理，而不是先花大量时间补产品理解。

### 风险

- 容易只补文档，不真正动边界。
- 若没有明确 phase owner，文档整理和代码拆分可能长期挂起。
- 旧结构和新结构会在一段时间内并存，需要纪律约束。

## 4. 路径二：用 AI Agent 重新启动一个项目

### 可行性

中。

如果目标只是做一个“更干净、从第一天就符合 harness 规范的 Tauri 新仓库”，当然可行。  
但如果目标是“替代当前 `clawpal` 的真实能力”，可行性会明显下降。

原因很简单：现在 `clawpal` 不是一个简单的 CRUD app，它已经覆盖：

- 多安装目标
- 本地与远程实例管理
- SSH
- Doctor
- History / rollback
- CLI / core / GUI 三层边界
- 发布和多平台构建

这些东西从零重建时，AI agent 可以加速 scaffold，但不会替你买回真实边界知识。

### 难度

高到很高。

最难的不是“写出新代码”，而是以下三件事：

1. 重新发现现在仓库里已经踩过的坑
2. 重新建立跨平台和远程场景的验证链路
3. 在功能追平前处理双线维护和迁移切换

如果 scope 不缩，clean-slate 项目的常见风险是：前期脚手架和主流程推进很快，后期却被兼容性、迁移和验收成本显著拉慢。

### 工作量

这里必须分成两个版本看。在同样的人员假设下：

- 干净新架构 MVP：约 6 到 8 工程周
- 接近当前 `clawpal` 能力的可替代版本：约 10 到 16 工程周

典型工作包会是：

#### Phase 1: 新仓库架构与规范脚手架

约 1 周。

- 建仓
- AGENTS / docs / runbooks / ADR 模板
- CI / PR / release 骨架
- 基础 Tauri + Rust + frontend 结构

#### Phase 2: 重新实现核心能力

约 3 到 5 周。

- core domain
- CLI
- Tauri command layer
- UI shell
- 配置读写、历史、doctor、install orchestration

#### Phase 3: 补复杂场景

约 2 到 4 周。

- Remote SSH
- Docker / WSL2 / 本地路径差异
- 多平台打包和发布
- 回归测试矩阵

#### Phase 4: 切换与迁移

约 1 到 2 周。

- 数据/配置兼容
- 旧版本迁移策略
- 并行验证
- cutover

### 优点

- 从第一天就可以按 harness 规范设计仓库结构。
- 没有历史包袱，架构可以更整齐。
- 适合在“明确砍 scope”的前提下打造一个更小、更硬的新产品线。

### 风险

- 容易低估现有产品已经覆盖的边界复杂度。
- 会产生较长的双线维护窗口。
- 很可能把大量时间花在“重新追平现有能力”，而不是提升 harness 质量。
- AI agent 产出虽然快，但如果产品知识和回归矩阵不完整，返工会非常多。

## 5. 正面对比

| 维度 | 原有 codebase 改造 | AI Agent 重启项目 |
|------|------------------|------------------|
| 可行性 | 高 | 中 |
| 难度 | 中高 | 高 / 很高 |
| 到“最小实践规范”的速度 | 快 | 中 |
| 到“接近当前能力”的速度 | 快很多 | 慢很多 |
| 业务连续性 | 高 | 低到中 |
| 架构整洁度上限 | 中高 | 高 |
| 迁移风险 | 低到中 | 高 |
| 对 AI agent 的利用方式 | 用于整理、拆分、补文档、补约束 | 用于重建大量代码与测试 |
| 我建议的优先级 | 第一选择 | 仅作为特定条件下的第二选择 |

## 6. 建议结论

### 推荐路径

优先选择：**在原有 `clawpal` codebase 上改造**。

这不是保守，而是更符合当前事实：

- `clawpal` 已经有不少 harness 元素
- 已经有真实产品复杂度
- 已经有三层架构收口方向
- 当前差距主要集中在“规范化、归档化、边界化”，而不是“完全缺失”

### 什么时候才应该重启

只有在下面条件同时成立时，我才会建议走“AI Agent 重启项目”：

1. 你准备主动砍掉当前一大半 scope
2. 你接受 2 到 4 个月的并行演进窗口
3. 你把目标从“替代当前 ClawPal”改成“做下一代更小的产品”

否则，重启大概率是在花更大代价，重新买回已经有的东西。

## 7. 更务实的执行建议

最现实的方案不是“二选一”，而是：

**主线做原仓库改造，局部用 AI agent 以 greenfield 方式重建子模块。**

具体建议：

1. 先在现仓库补 `AGENTS.md`、`docs/architecture/`、`docs/decisions/`、`docs/runbooks/`。
2. 选两个最影响 agent 可读性的地方优先拆：
   - `src/App.tsx`
   - `src-tauri/src/commands/mod.rs`
3. 把现有 `cc*.md` 和 `docs/plans/*` 提炼成长期有效的 ADR 和 runbook，而不是只保留审查记录。
4. 统一 Bun / npm 约定，避免 agent 在包管理器上来回摇摆。
5. 对高风险流转加“证据型 PR 要求”，例如测试输出、截图、日志、trace。

如果要在现仓库里做局部 greenfield，优先挑这类模块：

1. 输入输出边界清晰，能单独回归验证。
2. 高变更频率，但不直接绑定最危险的平台细节。
3. 现在已经形成大文件或高耦合热点。

按这个标准，优先候选通常是：

- 前端 shell / route / app-state 组织层
- Tauri command routing 层
- prompt 模板与 tool-schema 归档层

不建议第一批就 greenfield 重写的，是这些边界复杂且平台风险高的区域：

- SSH / remote transport
- 回滚与本地配置读写
- 多平台 release / signing / packaging

换句话说，更合理的表述不是“绝对不要重启”，而是：在当前假设下，**优先把 ClawPal 现有积累整理成一个真正可持续的 harness 仓库**，只在边界清晰的子模块上使用 greenfield 重建策略。

## 8. 参考来源

- 当前仓库基线：
  - `docs/harness-engineering-executable-playbook.md`
  - `docs/harness-engineering-summary-outline.md`
- ClawPal 仓库：
  - https://github.com/lay2dev/clawpal
- 关键参考文件：
  - https://raw.githubusercontent.com/lay2dev/clawpal/main/README.md
  - https://raw.githubusercontent.com/lay2dev/clawpal/main/agents.md
  - https://raw.githubusercontent.com/lay2dev/clawpal/main/design.md
  - https://raw.githubusercontent.com/lay2dev/clawpal/main/cc-architecture-refactor-v1.md
  - https://raw.githubusercontent.com/lay2dev/clawpal/main/docs/testing/business-flow-test-matrix.md
  - https://raw.githubusercontent.com/lay2dev/clawpal/main/.github/workflows/ci.yml
  - https://raw.githubusercontent.com/lay2dev/clawpal/main/.github/workflows/e2e.yml
  - https://raw.githubusercontent.com/lay2dev/clawpal/main/.github/workflows/pr-build.yml
