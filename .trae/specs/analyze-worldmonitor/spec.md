# WorldMonitor 源码分析与测试 Spec

## Why
需要对 koala73/worldmonitor（实时全球情报仪表盘，TypeScript + Vite + Tauri 2，55k+ stars，4k+ commits，~98MB 仓库）进行完整的源码拉取、架构分析和运行测试，为二次开发提供全面的技术理解基础。

## What Changes
- 从 GitHub 克隆 worldmonitor 源码到沙箱工作目录
- 生成详尽的 Markdown 格式全局架构原理分析报告（包括项目结构、技术栈、模块划分、数据流、API 契约、构建系统等）
- 在沙箱中执行安装和运行测试，生成详尽的测试报告

## Impact
- Affected specs: 无（新建任务）
- Affected code: 无修改，仅分析
- 产出物：
  - `/workspace/worldmonitor/` — 克隆的源码
  - `/workspace/worldmonitor_architecture_report.md` — 架构分析报告
  - `/workspace/worldmonitor_test_report.md` — 测试报告

## ADDED Requirements

### Requirement: 源码拉取
系统 SHALL 从 `https://github.com/koala73/worldmonitor.git` 克隆完整源码到 `/workspace/worldmonitor/`。

#### Scenario: 克隆成功
- **WHEN** 执行 `git clone` 命令
- **THEN** 源码完整存在于 `/workspace/worldmonitor/` 目录下
- **AND** 包含所有分支、标签和提交历史

### Requirement: 架构分析报告生成
系统 SHALL 对 worldmonitor 源码进行全面架构分析，生成 `/workspace/worldmonitor_architecture_report.md`。

报告必须包含以下章节：
1. **项目概览** — 项目定位、功能特性、技术栈总览
2. **目录结构** — 顶层目录树及每个目录的职责说明
3. **前端架构** — Vite 构建配置、入口文件、路由/页面结构、组件体系
4. **地图引擎** — globe.gl (Three.js) 和 deck.gl (MapLibre GL) 双引擎架构
5. **AI/ML 层** — Ollama/Groq/OpenRouter 集成、Transformers.js 浏览器端推理
6. **API 契约层** — Protocol Buffers 定义（276 protos，34 services）、sebuf HTTP 注解
7. **数据源层** — 65+ 外部数据源、500+ 新闻源、新鲜度监控
8. **桌面应用层** — Tauri 2 (Rust) + Node.js sidecar 架构
9. **多站点变体** — 6 个变体（world/tech/finance/commodity/happy/energy）单代码库机制
10. **国际化** — 24 种语言、RTL 支持
11. **缓存与部署** — Redis 三级缓存、Vercel Edge Functions、Railway relay、PWA
12. **构建系统** — npm scripts、构建流程、CI/CD

**分段读取规则**（强制执行）：
- 对于超过 2000 行的文件，必须制定子计划分段读取（如 Read offset=0 limit=500, offset=500 limit=500...），完整覆盖文件全部内容后再进行分析
- 禁止仅凭截取的首部分进行猜测
- 每个分段读取完成后，记录已读取的行范围和分析结论

#### Scenario: 架构分析成功
- **WHEN** 源码克隆完成
- **THEN** 生成完整的架构分析报告
- **AND** 报告覆盖所有 12 个必需章节
- **AND** 所有大型文件均已分段完整读取

### Requirement: 安装运行测试
系统 SHALL 在沙箱中执行 `npm install` 和 `npm run typecheck`（或等价测试命令），生成 `/workspace/worldmonitor_test_report.md`。

测试报告必须包含：
1. **环境信息** — Node.js 版本、npm 版本、操作系统信息
2. **安装过程** — 安装命令、耗时、安装的包数量、任何警告或错误
3. **类型检查** — `npm run typecheck` 结果、错误数量与分类
4. **构建测试** — `npm run build:full`（如环境支持）结果
5. **开发服务器启动测试** — `npm run dev` 是否能成功启动
6. **已知限制** — 无法在沙箱中运行的测试（如 Tauri 桌面端、需要 API key 的功能）

#### Scenario: 安装成功
- **WHEN** 执行 `npm install`
- **THEN** 所有依赖成功安装
- **AND** 记录安装日志

#### Scenario: 类型检查
- **WHEN** 执行 `npm run typecheck`
- **THEN** 记录类型检查结果（通过/失败及具体错误）