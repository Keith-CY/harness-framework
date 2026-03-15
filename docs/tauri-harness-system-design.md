# 基于 Tauri 项目的 Harness System 设计

## 1. 目标

为一个基于 Tauri 的桌面应用设计一套 `agent-first` 的 harness system，让工程师或 coding agent 可以在清晰边界内稳定完成以下工作：

- 理解前端、Rust 后端与原生能力之间的职责分层
- 启动本地开发环境并复现关键路径
- 对修改执行可重复、可定位、可收集证据的验证
- 在不破坏桌面端安全边界的前提下提高交付吞吐

这套 harness 的重点不是“让 agent 自动写更多代码”，而是“让系统更容易被理解、验证和恢复”。

## 2. 设计原则

### 2.1 先做边界清晰，再谈高自主

Tauri 项目天然跨越三种上下文：Web UI、Rust 命令层、操作系统能力。如果这三层混在一起，agent 很容易在错误层级做修改，或者只能依赖整机联调猜结果。

### 2.2 把跨端调用视为正式接口

`invoke` 不是普通函数调用，而是一个进程边界。所有跨端接口都应该有稳定命名、明确输入输出、统一错误模型和可验证契约。

### 2.3 默认把副作用隔离

文件系统、shell、通知、剪贴板、数据库、自动更新、窗口事件都属于高风险副作用。它们不应该散落在 UI 或业务逻辑里，而应该收敛到 adapter 层统一管理。

### 2.4 同时验证 dev 模式和 packaged app

很多 Tauri 问题只会在打包后暴露，例如资源路径、权限配置、签名、窗口行为、自动更新、平台差异。因此 harness 不能只围绕 `tauri dev` 设计。

## 3. 推荐架构

建议把项目拆成四层：

### 3.1 UI 层

目录建议：`src/`

职责：

- 页面、组件、状态管理
- 用户交互与渲染逻辑
- 调用统一的 Tauri client

约束：

- 不直接在组件中散落 `invoke("xxx")`
- 不直接访问原生能力
- 不拼接 command 名称和错误字符串

### 3.2 Command 层

目录建议：`src-tauri/src/commands/`

职责：

- 定义 Tauri command
- 参数校验
- 权限检查
- 错误映射
- 事件分发

约束：

- 保持薄层
- 不在这里堆业务编排
- 不直接写复杂文件系统或数据库逻辑

### 3.3 Domain / Service 层

目录建议：`src-tauri/src/domain/`

职责：

- 核心业务规则
- 应用级用例编排
- 与 Tauri 框架解耦的逻辑

约束：

- 尽量不依赖 `tauri::*`
- 输入输出尽量保持普通 Rust 类型
- 业务逻辑优先在这一层做单元测试

### 3.4 Adapter 层

目录建议：`src-tauri/src/adapters/`

职责：

- 文件系统
- shell
- 通知
- 剪贴板
- 数据库存储
- updater
- 外部 API

约束：

- 所有原生副作用从这里进入
- 提供 mock、fake 或 test double
- 记录清楚平台差异和失败模式

## 4. 推荐目录结构

```text
docs/
  architecture/
  plans/
  runbooks/
  tauri-harness-system-design.md
harness/
  fixtures/
  artifacts/
src/
  components/
  pages/
  lib/
    api/
src-tauri/
  src/
    commands/
    domain/
    adapters/
    errors/
tests/
  desktop/
  contracts/
  integration/
AGENTS.md
justfile
```

说明：

- `src/lib/api/` 统一封装前端对 Tauri command 的访问
- `tests/contracts/` 负责验证 command 输入输出契约
- `tests/desktop/` 负责关键桌面路径的 smoke test
- `harness/fixtures/` 存放最小稳定测试数据
- `harness/artifacts/` 收集日志、截图、trace、失败产物

## 5. Harness 入口命令

建议统一到 `justfile` 或 `cargo xtask`，不要把启动和验证命令散落到口头文档或临时 shell 片段里。

最低需要以下命令：

- `just doctor`
  - 检查 Rust、Node、平台依赖、WebView、环境变量、签名配置
- `just dev`
  - 启动前端开发服务器和 Tauri dev
- `just test-unit`
  - 前端单测、Rust 单测、domain 测试
- `just test-contracts`
  - 验证 command 的输入、输出、错误与权限边界
- `just test-integration`
  - 对 adapter 和最小 fixture 做集成验证
- `just smoke`
  - 启动桌面应用并走关键用户路径
- `just package-check`
  - 打包应用后执行 packaged app 冒烟验证
- `just artifacts`
  - 汇总日志、截图、trace、stdout/stderr

## 6. 验证链路设计

### 6.1 静态检查

每次改动至少能自动运行：

- 前端类型检查
- 前端 lint
- Rust `fmt`
- Rust `clippy`

目标是优先用机制阻止低级问题，而不是把这些问题留给 reviewer 或 agent 反复试错。

### 6.2 单元测试

重点覆盖：

- domain 层业务规则
- 错误映射
- 前端状态更新逻辑
- 数据转换逻辑

原则是：越核心、越纯粹的逻辑，越不要依赖完整桌面环境才能测。

### 6.3 Contract Test

这是 Tauri harness 的关键部分。每个 command 都应有契约验证：

- 输入 schema 是否符合预期
- 输出类型是否稳定
- 业务错误是否被映射成统一错误结构
- 权限不足时是否返回可识别错误
- 是否存在不应暴露给前端的内部错误细节

如果项目使用 `specta`、JSON schema 或自定义 type generation，可以把 contract test 和类型生成绑定在一起，减少前后端漂移。

### 6.4 Integration Test

针对 adapter 层验证：

- 文件读写
- 本地数据库
- 外部 API
- 平台能力调用

这里必须配最小 fixture，避免测试依赖真人机器上的随机状态。

### 6.5 Packaged App Smoke Test

至少验证以下路径：

- 应用能正常启动
- 首页渲染成功
- 一条核心 command 可以闭环执行
- 日志路径和资源路径正确
- 关键权限在打包后仍有效

这是最容易被团队忽略、但最值得单独保留的一条链路。

## 7. 日志与调试可见性

Tauri 项目如果缺乏统一日志，排查成本会非常高。建议做到：

- 前端日志、Rust 日志、command 调用日志进入统一目录
- 每次 command 调用带上 `request_id`
- 失败时自动输出 command 名称、参数摘要、错误码、耗时
- smoke test 失败时自动保存截图和日志包

建议在 `docs/runbooks/` 写清楚以下内容：

- 本地开发启动方式
- 如何查看前端日志
- 如何查看 Rust 日志
- 如何排查 command 调用失败
- 如何定位 packaged app 与 dev 模式差异

## 8. 安全与权限

Tauri harness 设计不能把安全当成“上线前再补”的事情。至少要把以下内容编码进机制：

- command 白名单和职责说明
- capability / permission 配置的变更审查
- 高风险 adapter 的测试覆盖
- shell、文件系统、更新能力的默认最小权限
- 任何新增原生能力都必须补文档和验证

原则很简单：agent 可以提高改动速度，但不能绕过桌面应用的权限边界。

## 9. 面向 Agent 的工作入口

建议在 `AGENTS.md` 中明确写出：

- 项目目录说明
- 标准启动命令
- 标准验证命令
- command、domain、adapter 的边界规则
- 新增 command 时必须补哪些测试
- 修改权限配置时必须附哪些验证证据
- 打包问题与平台问题的常见排查路径

对 agent 来说，最有价值的不是“大段背景介绍”，而是固定入口、固定命令和固定验收标准。

## 10. 最小可用版本

如果团队还没有完整 harness，不要一开始就铺太大。先补这四项：

1. 一个统一命令入口，例如 `justfile`
2. 一套 command contract tests
3. 一条 packaged app smoke test
4. 一份 `AGENTS.md` 和两份 runbook

有了这四项，agent 就已经可以在较低风险下处理大量 Tauri 日常改动。

## 11. 最终判断标准

这套 harness 是否有效，不看“AI 写了多少代码”，而看下面这些指标是否改善：

- command 漂移是否减少
- packaged app 问题是否更早暴露
- 新任务是否更容易被拆给 agent
- review 是否更多聚焦边界与风险，而不是补上下文
- 同类环境问题是否越来越少重复发生

如果这些指标在变好，这套 Tauri harness system 就是有效的。
