# WorldMonitor 全局架构原理分析报告

> 基于源码 `/workspace/worldmonitor/` 分析生成  
> 项目版本：v2.8.0 | 分析日期：2026-06-06  
> 仓库：https://github.com/koala73/worldmonitor

---

## 第 1 章：项目概览

### 1.1 项目定位

WorldMonitor 是一款**实时全球情报仪表盘**（Real-time Global Intelligence Dashboard），以单一 TypeScript SPA 代码库实现以下核心能力：

- **500+ 精选新闻源**，覆盖 15 个类别，AI 合成简报
- **双地图引擎**：3D 地球（globe.gl + Three.js）与 WebGL 平面地图（deck.gl + MapLibre GL），支持 56 种地图图层
- **跨流关联分析**：军事、经济、灾难、升级信号交叉关联
- **国家情报指数（CII）**：12 个信号类别的综合风险评分
- **金融雷达**：92 个股票交易所、大宗商品、加密货币、7 信号市场综合指数
- **本地 AI**：支持 Ollama 本地运行，无需 API 密钥
- **6 站点变体**：world / tech / finance / commodity / happy / energy
- **原生桌面应用**：Tauri 2（macOS / Windows / Linux）
- **24 种语言**，含 RTL 支持

### 1.2 技术栈总览

| 类别 | 技术 |
|------|------|
| **前端框架** | Vanilla TypeScript（无 React/Vue），Preact 辅助 |
| **构建工具** | Vite 6.x |
| **地图引擎** | globe.gl + Three.js（3D），deck.gl 9.x + MapLibre GL 5.x（2D） |
| **AI/ML** | Ollama / Groq / OpenRouter（服务端），Transformers.js + ONNX Runtime Web（浏览器端） |
| **API 契约** | Protocol Buffers（sebuf 框架），276 protos，34+ services |
| **桌面** | Tauri 2 (Rust) + Node.js sidecar |
| **部署** | Vercel Edge Functions（60+），Railway relay，Tauri，PWA |
| **缓存** | Redis (Upstash)，3 级缓存，CDN，Service Worker |
| **后端** | Convex Cloud（联系表单/等待列表） |
| **测试** | node:test，Playwright（E2E），Vitest（Convex） |
| **代码质量** | Biome（lint/format），TypeScript 5.7，markdownlint |
| **CI/CD** | GitHub Actions（12 workflows） |

### 1.3 项目规模

- **Stars**: 55,876 | **Forks**: 8,964
- **Commits**: 4,072+
- **仓库大小**: ~98 MB
- **源文件数**: 3,409 个文件
- **主要语言**: TypeScript（99%+）

---

## 第 2 章：目录结构

### 2.1 顶层目录树

```
worldmonitor/
├── api/                    # Vercel Edge Functions（自包含 JS）
│   ├── _*.js               # 共享辅助（CORS, rate-limit, API key, relay）
│   └── <domain>/           # 域名端点（aviation, climate, conflict, cyber, ...）
├── blog-site/              # 静态博客（构建到 public/blog/）
├── convex/                 # Convex 后端（联系表单, 等待列表）
├── data/                   # 静态数据文件（gamma-irradiators, localities, telegram channels）
├── deploy/                 # 部署配置
├── docker/                 # Dockerfile + nginx 配置
├── docs/                   # Mintlify 文档站点（80+ .mdx 文件）
├── e2e/                    # Playwright E2E 测试
├── plans/                  # 开发计划文档
├── pro-test/               # Pro 变体测试
├── proto/                  # Protobuf 服务定义（sebuf 框架）
│   ├── sebuf/http/         # sebuf HTTP 注解
│   └── worldmonitor/       # 按域名组织的服务定义
├── public/                 # 静态资源（favicon, PWA 图标）
├── scripts/                # 种子脚本（200+ .mjs）, 构建辅助, Railway relay
├── server/                 # 服务端代码（打包到 Edge Functions）
│   ├── _shared/            # Redis, rate-limit, LLM, 缓存工具
│   ├── gateway.ts          # 域名网关工厂
│   ├── router.ts           # 路由匹配
│   └── worldmonitor/       # 域名处理器（镜像 proto 结构）
├── shared/                 # 跨平台 JSON 配置（markets, RSS domains）
├── src/                    # 浏览器 SPA（TypeScript）
│   ├── app/                # 应用编排管理器
│   ├── bootstrap/          # Chunk 重载恢复
│   ├── components/         # Panel 子类 + 地图组件
│   ├── config/             # 变体, 面板, 图层, 市场配置
│   ├── generated/          # Proto 生成的 client/server stubs
│   ├── locales/            # i18n 翻译文件
│   ├── services/           # 按域名组织的业务逻辑
│   ├── types/              # TypeScript 类型定义
│   ├── utils/              # 共享工具（circuit-breaker, theme, URL state）
│   └── workers/            # Web Workers（analysis, ML, vector DB）
├── src-tauri/              # Tauri 桌面壳（Rust）
│   └── sidecar/            # Node.js sidecar API 服务器
├── tests/                  # 单元/集成测试（node:test）
├── todos/                  # 任务跟踪
├── workers/                # Cloudflare Workers
├── package.json            # 项目配置与依赖
├── vite.config.ts          # Vite 构建配置
├── tsconfig.json           # TypeScript 配置
├── vercel.json             # Vercel 部署配置
├── docker-compose.yml      # Docker Compose 配置
├── Makefile                # 构建辅助
├── middleware.ts           # Vercel Edge Middleware
├── index.html              # 主入口 HTML
├── settings.html           # 设置页面
├── live-channels.html      # 直播频道页面
├── mcp-grant.html          # MCP 授权页面
└── ARCHITECTURE.md         # 架构文档
```

### 2.2 各目录职责说明

| 目录 | 职责 | 部署位置 |
|------|------|---------|
| `src/` | 浏览器 SPA 前端代码 | Vercel 静态托管 / Tauri WebView |
| `api/` | Vercel Edge Functions | Vercel Edge Network |
| `server/` | 服务端业务逻辑，被打包到 Edge Functions | Vercel Edge（打包后） |
| `scripts/` | 数据种子脚本、Railway relay 服务 | Railway |
| `proto/` | API 契约定义（Protocol Buffers） | 仅开发时 |
| `src-tauri/` | Tauri 2 桌面壳（Rust）+ sidecar | 桌面端 |
| `convex/` | Convex 后端函数 | Convex Cloud |
| `shared/` | 跨平台配置 | 嵌入各个构建产物 |
| `docs/` | Mintlify 文档 | Mintlify（通过 Vercel 代理） |
| `tests/` | 单元/集成测试 | CI 环境 |
| `e2e/` | Playwright E2E 测试 | CI 环境 |

---

## 第 3 章：前端架构

### 3.1 入口与初始化流程

**入口文件**: [src/main.ts](file:///workspace/worldmonitor/src/main.ts)

初始化顺序：
1. Sentry 错误追踪初始化
2. Vercel Analytics 初始化
3. 动态 Meta 标签设置
4. 运行时 Fetch 补丁（桌面端 sidecar 重定向）
5. 主题应用
6. 创建 `App` 实例

**App.init() 8 阶段初始化**:

```
阶段 1: Storage + i18n
  ├── IndexedDB 初始化
  ├── 语言检测（浏览器/本地存储）
  └── 语言包加载

阶段 2: ML Worker
  ├── ONNX 模型准备
  ├── Embeddings (MiniLM-L6)
  ├── Sentiment（情感分析）
  └── Summarization（摘要）

阶段 3: Sidecar（仅桌面端）
  └── 等待桌面 sidecar 就绪

阶段 4: Bootstrap
  ├── 双层并发水合
  ├── Fast tier（3s 超时）
  └── Slow tier（5s 超时）

阶段 5: Layout
  └── PanelLayoutManager 渲染地图和面板

阶段 6: UI
  ├── SignalModal（信号弹窗）
  ├── IntelligenceGapBadge（情报缺口徽章）
  ├── BreakingNewsBanner（突发新闻横幅）
  └── 关联引擎

阶段 7: Data
  ├── 并行 loadAllData()
  └── 视口条件 primeVisiblePanelData()

阶段 8: Refresh
  └── 变体特定轮询间隔（startSmartPollLoop）
```

### 3.2 组件模型

**Panel 基类**: [src/components/Panel.ts](file:///workspace/worldmonitor/src/components/Panel.ts)

所有面板继承自 `Panel` 基类：
- 渲染通过 `setContent(html)` 方法（150ms 防抖）
- 事件委托在稳定的 `this.content` 元素上
- 支持可调整大小的行/列跨度（持久化到 localStorage）
- 86 个面板类，按域名集群组织

**面板集群划分**（来自 vite.config.ts 的 `PANEL_CLUSTER`）:

| 集群 | 面板数 | 领域 |
|------|--------|------|
| panels-markets | ~18 | 股票、加密货币、市场广度、ETF 流 |
| panels-energy | ~12 | 能源、大宗商品、管道、石油库存 |
| panels-defense | ~7 | 军事、航空、防御专利 |
| panels-news | ~10 | 新闻、简报、Telegram、GDELT |
| panels-economy | ~14 | 宏观、消费者价格、国债、制裁 |
| panels-intel | ~20 | 国家情报、关联、预测、监控 |
| panels-risk | ~15 | 灾难、气候、健康、互联网中断 |

### 3.3 状态管理

无外部状态库。使用 **AppContext** 中心可变对象：

```
AppContext {
  mapReferences       // 地图实例引用
  panelInstances      // 面板实例
  panelSettings       // 面板配置
  layerSettings       // 图层设置
  cachedData          // 所有缓存数据（news, markets, predictions, clusters, intelligence）
  inFlightRequests    // 进行中的请求追踪
  uiComponents        // UI 组件引用
}
```

**URL 状态同步**: `src/utils/urlState.ts` 双向同步（250ms 防抖）

### 3.4 多页面入口

| 入口 | HTML 文件 | 用途 |
|------|-----------|------|
| 主应用 | index.html | 仪表盘主界面 |
| 设置 | settings.html | 用户设置页面 |
| 直播频道 | live-channels.html | 直播频道页面 |
| MCP 授权 | mcp-grant.html | MCP 授权页面 |

### 3.5 服务层架构

`src/services/` 目录按域名组织，包含 40+ 服务模块：

- **核心服务**: bootstrap, runtime, auth-state, user-identity, clerk
- **数据服务**: aviation, climate, conflict, cyber, economic, energy, health, infrastructure, intelligence, maritime, military, natural, news, prediction, resilience, sanctions, scenario, shipping, supply-chain, trade, unrest, wildfire
- **功能服务**: ai-classify-queue, alerts, billing, brief-*, correlation-engine, entitlements, followed-countries, mcp-*, signal-aggregator, forecast, settings-manager, widget-store
- **基础设施**: convex-client, ml-worker, geo, terrain, sensor-data, cached-risk-scores

---

## 第 4 章：地图引擎

### 4.1 双引擎架构

地图容器 `MapContainer` 负责切换两种地图引擎：

```
MapContainer
├── DeckGLMap (2D/WebGL)
│   ├── deck.gl 9.x
│   ├── MapLibre GL 5.x
│   ├── PMTiles 协议
│   └── Supercluster 聚合
│
└── GlobeMap (3D)
    ├── globe.gl 2.x
    ├── Three.js
    └── 单一 htmlElementsData 数组（_kind 判别器）
```

**切换逻辑**:
- 桌面端默认使用 DeckGLMap
- 移动端根据 WebGL 支持选择 DeckGL 或 SVG fallback
- 用户可手动切换 2D/3D 模式
- 切换时保存快照、销毁实例、重建

### 4.2 56 种地图图层类型

图层定义位于 [src/config/map-layer-definitions.ts](file:///workspace/worldmonitor/src/config/map-layer-definitions.ts)，每个图层指定：

- **渲染器支持**: flat（deck.gl）/ globe（globe.gl）/ both
- **Premium 状态**: 免费/Premium
- **变体过滤**: 哪些变体启用
- **i18n 键**: 多语言标签
- **图标**: 图层图标

**deck.gl 图层类型**:
- ScatterplotLayer（散点图）
- GeoJsonLayer（地理JSON）
- PathLayer（路径）
- IconLayer（图标）
- PolygonLayer（多边形）
- ArcLayer（弧线）
- HeatmapLayer（热力图）
- H3HexagonLayer（H3六边形）

### 4.3 底图与瓦片

- **PMTiles 协议**: 自托管底图瓦片（protomaps/basemaps）
- **Service Worker 缓存**: PMTiles 范围请求缓存（30天）
- **CDN 集成**: Cloudflare + R2 存储

### 4.4 变体与图层关联

`getLayersForVariant()` 函数根据变体和渲染器返回对应的图层定义列表。不同变体启用不同的图层集合。

---

## 第 5 章：AI/ML 层

### 5.1 架构总览

```
浏览器端（Web Workers）
├── ml.worker.ts
│   ├── @xenova/transformers
│   ├── ONNX Runtime Web
│   ├── MiniLM-L6 → Embeddings（语义向量）
│   ├── Sentiment Analysis（情感分析）
│   ├── Summarization（文本摘要）
│   └── NER（命名实体识别）
│
├── analysis.worker.ts
│   ├── 新闻聚类（Jaccard 相似度）
│   └── 跨域关联检测
│
└── vector-db.ts
    └── IndexedDB 向量存储（语义搜索）

服务端
├── Ollama（本地，无需 API key）
├── Groq（云端）
├── OpenRouter（云端）
└── Anthropic Claude（@anthropic-ai/sdk）
```

### 5.2 Transformers.js 浏览器端推理

- **Embeddings**: MiniLM-L6 v2，384 维向量
- **Sentiment**: 情感分析模型
- **Summarization**: 文本摘要
- **NER**: 命名实体识别
- **Worker 内向量存储**: 用于标题记忆的向量数据库

### 5.3 LLM 集成

`server/_shared/llm.ts` 提供统一的 LLM 调用接口：
- 支持多提供商切换
- Prompt 处理与清洗
- 结果缓存

---

## 第 6 章：API 契约层（Protocol Buffers）

### 6.1 sebuf 框架

**sebuf** = Protocol Buffers + HTTP 注解，自定义框架：

```
proto/ 定义
    ↓ buf generate
├── src/generated/client/   (TypeScript RPC 客户端 stubs)
├── src/generated/server/   (TypeScript 服务端消息类型)
└── docs/api/               (OpenAPI v3 规范)
```

**HTTP 注解** (`proto/sebuf/http/annotations.proto`):
- `HttpMethod`: GET / POST / PUT / DELETE / PATCH
- `HttpConfig`: 路径模板 + HTTP 方法
- `ServiceConfig`: 服务级默认配置
- `(sebuf.http.query)`: GET 查询参数标记
- `(sebuf.http.path)`: 路径参数标记

### 6.2 服务域名划分

| 域名 | 服务 | 示例 RPC |
|------|------|---------|
| intelligence | 情报分析 | getCountryIntelBrief, searchGdeltDocuments, classifyEvent |
| economic | 经济数据 | getBisCredit, getEcbFxRates, getNationalDebt, getEnergyPrices |
| market | 金融市场 | analyzeStock, listCryptoMarketQuotes, getGoldIntelligence |
| news | 新闻聚合 | getFeedDigest, summarizeArticle, listNewsFeed |
| climate | 气候环境 | getCo2Monitoring, listClimateDisasters, listClimateAnomalies |
| military | 军事跟踪 | listMilitaryBases, getMilitaryFlights, getTheaterPosture |
| conflict | 冲突监测 | listUcdpEvents, getConflictIntel |
| cyber | 网络威胁 | listCyberThreats |
| maritime | 海事情报 | listVesselPositions, getChokepointHistory |
| aviation | 航空 | listAirportDelays, getAviationPrices |
| health | 健康 | listDiseaseOutbreaks, listAirQualityAlerts |
| natural | 自然灾害 | listNaturalEvents |
| wildfire | 野火 | listFireDetections |
| prediction | 预测市场 | listPredictionMarkets |
| sanctions | 制裁 | listSanctionsPressure, lookupEntity |
| resilience | 韧性评估 | getResilienceScores |
| trade | 贸易 | listComtradeFlows |
| supply_chain | 供应链 | getCountryProducts, getMultiSectorCostShock |
| infrastructure | 基础设施 | listSubmarineCables, listPipelines |
| displacement | 流离失所 | getDisplacementSummary |
| positive_events | 正面事件 | listPositiveGeoEvents |
| unrest | 社会动荡 | listUnrestEvents |
| research | 研究 | getResearchPapers |
| scenario | 情景模拟 | runScenario, getScenarioStatus |
| shipping | 航运 | getShippingRoutes |
| leads | 线索 | (企业情报) |
| giving | 捐赠 | (慈善数据) |
| seismology | 地震 | listEarthquakes |

### 6.3 Gateway Factory 处理管道

`server/gateway.ts` 的 10 步处理流程：

```
1. Origin 检查 → 403（如不允许）
2. CORS 头设置
3. OPTIONS 预检 → 204
4. 安全头剥离（移除上游代理的头）
5. 内部 MCP 验证
6. API 密钥验证
7. 授权检查（Premium 路径）
8. IP 速率限制
9. 路由匹配（静态 Map → 动态 {param} 扫描）
10. 处理器执行 + 错误边界 → ETag（FNV-1a）+ 缓存头
```

### 6.4 Edge Function 自包含约束

- 每个 `api/*.js` 文件必须是自包含的
- 禁止 `node:` 内置模块导入
- 禁止跨目录 `../server/` 或 `../src/` 导入
- CI 通过 `tests/edge-functions.test.mjs` 验证
- Pre-push hook 执行 esbuild bundle 检查

---

## 第 7 章：数据源层

### 7.1 数据管道架构

```
外部数据源（65+ providers）
    ↓
Railway Seed Scripts（scripts/seed-*.mjs）
    ↓ fetch + transform
Redis（Upstash）
    ↓ atomicPublish (SET NX + seed-meta)
    ↓
/api/bootstrap（批量读取 Redis keys）
    ↓ 双层水合（fast 3s + slow 5s）
浏览器 SPA
    ↓ getHydratedData(key)
面板组件
```

### 7.2 Seed 脚本分类

200+ 种子脚本覆盖以下数据域：

| 类别 | 示例脚本 |
|------|---------|
| 金融市场 | seed-market-quotes, seed-crypto-quotes, seed-fx-rates, seed-fear-greed |
| 宏观经济 | seed-economy, seed-imf-growth, seed-national-debt, seed-fao-food-price-index |
| 能源 | seed-energy-spine, seed-fuel-prices, seed-oil-inventories, seed-gie-gas-storage |
| 军事 | seed-military-bases, seed-military-flights, seed-military-cii, seed-thermal-escalation |
| 冲突 | seed-conflict-intel, seed-ucdp-events, seed-gdelt-intel |
| 气候 | seed-climate-disasters, seed-climate-anomalies, seed-co2-monitoring, seed-fire-detections |
| 健康 | seed-disease-outbreaks, seed-health-air-quality |
| 基础设施 | seed-submarine-cables, seed-pipelines-gas, seed-pipelines-oil, seed-storage-facilities |
| 贸易 | seed-trade-flows, seed-comtrade-bilateral-hs4, seed-supply-chain-trade |
| 海事 | seed-portwatch, seed-chokepoint-flows |
| 网络 | seed-cyber-threats, seed-internet-outages |
| 预测 | seed-prediction-markets, seed-forecasts |
| 韧性 | seed-resilience-scores, seed-resilience-static |
| 制裁 | seed-sanctions-pressure |
| 航空 | seed-aviation, seed-military-flights |

### 7.3 AIS Relay 持续种子循环

Railway relay 服务 `scripts/ais-relay.cjs` 运行持续循环：
- 市场数据（股票、大宗商品、加密货币、稳定币、ETF 流）
- 航空（国际航班延误）
- 正面事件
- GPSJAM（GPS 干扰）
- 风险评分（CII）
- UCDP 事件

### 7.4 数据新鲜度监控

`api/health.js` 检查每个 bootstrap 和独立 key：
- 读取 `seed-meta:<key>` 中的 `{ fetchedAt, recordCount }`
- 对比 `maxStaleMin` 阈值
- 级联组处理回退链（live → stale → backup）
- 返回状态：OK / STALE / WARN / EMPTY

### 7.5 RSS 新闻源

500+ 精选 RSS 源，覆盖 15 个类别。通过 `api/rss-proxy.js`（生产）和 `rssProxyPlugin()`（开发）代理。域名白名单包含 100+ 域名（BBC、Guardian、NPR、CNN、Al Jazeera、Reuters、SCMP 等）。

---

## 第 8 章：桌面应用层

### 8.1 Tauri 2 架构

```
Tauri Shell (Rust)
├── 应用生命周期管理
├── 系统托盘
├── IPC 命令
│   ├── 密钥管理（平台 keyring）
│   ├── Sidecar 控制（启动/停止/探针）
│   └── 窗口管理（3 个可信窗口）
│
└── Node.js Sidecar
    ├── 动态端口运行
    ├── 动态加载 Edge Function 处理器
    ├── 从 keyring 注入密钥
    └── monkey-patch fetch → 强制 IPv4
```

### 8.2 密钥管理

- 密钥存储在平台 keyring（macOS Keychain / Windows Credential Manager / Linux keyring）
- 通过 Tauri IPC 注入 sidecar 环境变量
- 限定于白名单环境变量 key
- 永不以明文存储

### 8.3 Fetch 拦截与重定向

`installRuntimeFetchPatch()` 替换桌面渲染器的 `window.fetch`：
- 所有 `/api/*` 请求路由到 sidecar
- 携带 `Authorization: Bearer <token>`（5 分钟 TTL）
- Sidecar 失败时回退到云端 API
- 401 时自动刷新 token 并重试

### 8.4 窗口管理

三个可信窗口：
- **主窗口**: 仪表盘
- **设置窗口**: 用户设置
- **直播频道窗口**: 直播频道

---

## 第 9 章：多站点变体

### 9.1 6 变体定义

| 变体 | 域名 | 主题 |
|------|------|------|
| full | worldmonitor.app | 完整全球情报 |
| tech | tech.worldmonitor.app | 科技情报 |
| finance | finance.worldmonitor.app | 金融市场 |
| commodity | commodity.worldmonitor.app | 大宗商品 |
| happy | happy.worldmonitor.app | 正面新闻 |
| energy | energy.worldmonitor.app | 能源情报 |

### 9.2 变体检测机制

`src/config/variant.ts`:
1. **构建时**: `VITE_VARIANT` 环境变量
2. **桌面端**: localStorage 覆盖（`VITE_VARIANT` 或 `wm-variant`）
3. **Web 端**: hostname 检测（`tech.worldmonitor.app` → tech）

### 9.3 变体控制

每个变体控制：
- 默认面板集合
- 启用的地图图层
- 刷新间隔
- 主题色
- UI 文本
- 变体切换时重置所有设置为默认

### 9.4 变体元数据

`src/config/variant-meta.ts` 定义每个变体的：
- SEO 标题、描述、关键词
- 功能列表
- 站点名称和简称
- URL
- 分类（用于 PWA manifest）

---

## 第 10 章：国际化

### 10.1 技术方案

- **框架**: i18next 25.x + i18next-browser-languagedetector
- **语言文件**: `src/locales/*.json`（24 种语言）
- **检测顺序**: localStorage → 浏览器设置 → 英语 fallback
- **动态加载**: 非英语语言包懒加载，英语静态导入

### 10.2 支持的语言

支持 24 种语言，包括：
- 英语 (en)、中文 (zh)、阿拉伯语 (ar)、法语 (fr)、西班牙语 (es)
- 德语 (de)、日语 (ja)、韩语 (ko)、俄语 (ru)、葡萄牙语 (pt)
- 印地语 (hi)、土耳其语 (tr)、匈牙利语 (hu)、克罗地亚语 (hr) 等

### 10.3 RTL 支持

- 阿拉伯语等 RTL 语言的特殊 CSS 重写 (`src/styles/rtl-overrides.css`)
- 翻译文件结构与英语对齐
- 变体特定的翻译

---

## 第 11 章：缓存与部署

### 11.1 四层缓存架构

```
Bootstrap Seed（Railway 定时写入 Redis）
    ↓ miss
In-Memory Cache（每个 Vercel 实例，短 TTL）
    ↓ miss
Redis（Upstash，跨实例，cachedFetchJson 并发合并）
    ↓ miss
Upstream API Fetch（结果缓存回 Redis + seed-meta 写入）
```

### 11.2 Cache Tier 策略

| Tier | s-maxage | 用途 |
|------|----------|------|
| fast | 300s | 实时事件流、航班状态 |
| medium | 600s | 市场报价、股票分析 |
| slow | 1800s | ACLED 事件、网络威胁 |
| static | 7200s | 人道主义摘要、ETF 流 |
| daily | 86400s | 关键矿物、静态参考数据 |
| no-store | 0 | 船舶快照、飞机追踪 |

### 11.3 ETag / 条件请求

- `server/gateway.ts` 对每个响应体计算 FNV-1a 哈希
- 返回为 `ETag` 头
- 客户端发送 `If-None-Match`
- 内容未变时返回 `304 Not Modified`

### 11.4 CDN 集成

- `CDN-Cache-Control` 头给 Cloudflare edge 更长的 TTL
- CF 可通过 ETag 重新验证，无需完整载荷传输

### 11.5 部署拓扑

| 服务 | 平台 | 角色 |
|------|------|------|
| SPA + Edge Functions | Vercel | 静态文件、API 端点、中间件 |
| AIS Relay | Railway | WebSocket 代理、种子循环、RSS 代理 |
| Redis | Upstash | 缓存、速率限制、seed-meta 新鲜度 |
| Convex | Convex Cloud | 联系表单、等待列表 |
| 文档 | Mintlify | 公共文档（通过 Vercel 代理） |
| 桌面应用 | Tauri 2.x | macOS/Windows/Linux 原生应用 |
| 容器镜像 | GHCR | 多架构 Docker 镜像 |

### 11.6 PWA 配置

- Workbox Service Worker（`vite-plugin-pwa`）
- 缓存策略：NetworkFirst（导航）、NetworkOnly（API）、CacheFirst（字体、PMTiles）
- Web Push 支持
- 自动更新 + 跳过等待
- 离线支持

---

## 第 12 章：构建系统

### 12.1 npm Scripts 总览

| 类别 | 命令 | 说明 |
|------|------|------|
| 开发 | `npm run dev` | 启动开发服务器（默认 full 变体） |
| 变体开发 | `npm run dev:tech` | tech 变体开发 |
| 构建 | `npm run build:full` | 生产构建 |
| 类型检查 | `npm run typecheck` | `tsc --noEmit` |
| 代码检查 | `npm run lint` | Biome lint + safe-html 检查 |
| 测试 | `npm run test:data` | `node:test` 单元/集成测试 |
| E2E | `npm run test:e2e:full` | Playwright E2E |
| 桌面 | `npm run desktop:dev` | Tauri 开发模式 |
| 桌面构建 | `npm run desktop:build:full` | Tauri 生产构建 |

### 12.2 构建流程

```
prebuild
├── build:openapi → 复制 OpenAPI 规范到 public/
└── build:agent-skills → 构建 agent skills 索引

build:full
├── build:openapi
├── build:agent-skills
├── build:blog（blog-site → public/blog/）
└── tsc + vite build（VITE_VARIANT=full）
```

### 12.3 Vite 构建优化

- **代码分割**: 手动分块（maplibre, deck-stack, transformers, onnxruntime, d3, i18n, sentry, clerk）
- **面板集群**: 按域名分块（panels-markets, panels-energy, panels-defense, panels-news, panels-economy, panels-intel, panels-risk）
- **懒加载**: MapContainer 动态导入，WebGL 栈延迟加载
- **Brotli 预压缩**: 构建时自动生成 `.br` 文件
- **PWA**: Service Worker 缓存策略

### 12.4 CI/CD 工作流

| Workflow | 触发条件 | 检查内容 |
|----------|---------|---------|
| typecheck.yml | PR, push to main | `tsc --noEmit`（src + api） |
| lint-code.yml | PR, push to main | Biome lint + sebuf API 契约 |
| lint.yml | PR (markdown 变更) | markdownlint |
| test.yml | PR, push to main | 单元/集成测试 + docs-stats |
| proto-check.yml | PR (proto 变更) | 生成代码与提交一致 |
| pro-bundle-freshness.yml | PR (pro bundle 变更) | Pro bundle 新鲜度 |
| feed-validation.yml | PR (feed 变更), daily | RSS feed 可达性 |
| contributor-trust.yml | PR | 首次贡献者信任门禁 |
| deploy-gate.yml | Test/Typecheck 完成 | 聚合 smoke gate 状态 |
| convex-deploy.yml | Push to main, manual | 部署 Convex 后端 |
| build-desktop.yml | Release tag, push, manual | 多平台 Tauri 构建 + 签名 |
| docker-publish.yml | Release, manual | 多架构 Docker 镜像推送 |
| test-linux-app.yml | Manual | Linux AppImage 无头测试 |

### 12.5 Pre-push Hook

`.husky/pre-push` 在每次 `git push` 前执行：
1. TypeScript 检查（`tsc --noEmit`）
2. CJS 语法验证
3. Edge function esbuild bundle 检查
4. Edge function 导入护栏测试
5. Markdown lint
6. MDX lint（Mintlify 兼容性）
7. 版本同步检查

---

## 附录：关键文件索引

| 文件 | 用途 |
|------|------|
| [ARCHITECTURE.md](file:///workspace/worldmonitor/ARCHITECTURE.md) | 官方架构文档 |
| [package.json](file:///workspace/worldmonitor/package.json) | 项目配置与依赖 |
| [vite.config.ts](file:///workspace/worldmonitor/vite.config.ts) | 构建配置 |
| [tsconfig.json](file:///workspace/worldmonitor/tsconfig.json) | TypeScript 配置 |
| [vercel.json](file:///workspace/worldmonitor/vercel.json) | Vercel 部署配置 |
| [middleware.ts](file:///workspace/worldmonitor/middleware.ts) | Edge Middleware |
| [src/main.ts](file:///workspace/worldmonitor/src/main.ts) | 前端入口 |
| [src/App.ts](file:///workspace/worldmonitor/src/App.ts) | 应用主类 |
| [src/components/MapContainer.ts](file:///workspace/worldmonitor/src/components/MapContainer.ts) | 地图容器 |
| [src/components/Panel.ts](file:///workspace/worldmonitor/src/components/Panel.ts) | Panel 基类 |
| [src/config/map-layer-definitions.ts](file:///workspace/worldmonitor/src/config/map-layer-definitions.ts) | 图层定义 |
| [server/gateway.ts](file:///workspace/worldmonitor/server/gateway.ts) | API 网关工厂 |
| [server/router.ts](file:///workspace/worldmonitor/server/router.ts) | 路由匹配 |
| [src-tauri/src/main.rs](file:///workspace/worldmonitor/src-tauri/src/main.rs) | Tauri 入口 |
| [Makefile](file:///workspace/worldmonitor/Makefile) | 构建辅助 |
| [docker-compose.yml](file:///workspace/worldmonitor/docker-compose.yml) | Docker 编排 |

---

*报告生成时间：2026-06-06 | 基于源码分析，非推测性描述*