# WorldMonitor 二次开发问题备忘

> 记录时间：2026-06-06
> 状态：**待改进**（用户实际使用中发现，待二次开发处理）
> 适用范围：worldmonitor 二次开发

---

## 问题 1：UI 卡死 / 程序假死

### 现象
实际使用一段时间后，UI 出现卡死，程序进入假死状态。

### 推测原因（待源码验证）
- [ ] **WebGL 资源泄漏** — `DeckGLMap` / `GlobeMap` 切换时未释放 Three.js scene / GL context
- [ ] **Z-index 面板堆积** — 86 个面板同时运行，部分面板订阅事件未取消
- [ ] **Web Worker 消息堆积** — `ml.worker.ts` / `analysis.worker.ts` 的 postMessage 队列阻塞
- [ ] **IndexedDB 事务未关闭** — `vector-db.ts` 的持续事务泄漏
- [ ] **setInterval 未清理** — `startSmartPollLoop()` 的轮询定时器在面板销毁时未清理
- [ ] **Backpressure 缺失** — 高频数据流（AIS、飞机追踪）无流控

### 排查方向
1. Chrome DevTools Performance → 录制卡死前 30s
2. 检查 `MapContainer.ts` 的 `rehydrateActiveMap()` 是否在切换时正确销毁旧实例
3. 检查面板 `setContent(html)` 的 150ms debounce 是否在面板卸载时清理 timer
4. 检查 `runtime.ts` 的 `installRuntimeFetchPatch` 是否替换 `window.fetch` 后无法还原
5. 用 `performance.memory.usedJSHeapSize` 监控内存增长

### 改进方向
- 实现面板的 `destroy()` 生命周期，统一释放资源
- 用 WeakRef 跟踪面板持有的 DOM/WebGL 资源
- 给 Web Worker 加消息队列上限 + 背压
- IndexedDB 事务使用 try/finally 强制关闭
- 添加"性能监控面板"显示 FPS、内存、活跃 Worker 数

---

## 问题 2：刷新机制不明，UI 无法设置/手动刷新

### 现象
- 刷新策略对用户不透明
- UI 中找不到开关或手动刷新按钮
- 不知道哪些数据在更新、什么时候更新

### 推测原因（待源码验证）
- [ ] `startSmartPollLoop()` 的轮询间隔硬编码在 `variant.ts` 中
- [ ] `refresh-scheduler.ts` 的指数退避策略对用户不可见
- [ ] `App.init()` 阶段 8 之后才启动轮询，初始状态不显示
- [ ] 缺少 `/api/health` 的前端展示（虽然后端有，但前端未渲染）
- [ ] 缺少"立即刷新"按钮

### 排查方向
1. 阅读 `src/app/refresh-scheduler.ts` 全部逻辑
2. 阅读 `src/config/variant.ts` 中的刷新间隔配置
3. 检查 `api/health.js` 是否有对应前端面板
4. 检查 `IntuitionSignalModal` / `SignalModal` 是否有手动刷新入口

### 改进方向
- **增加"健康状态"面板**：显示每个数据源的 last fetched、staleness、status
- **增加全局手动刷新按钮**：工具栏右上角
- **增加每面板刷新控制**：面板头部下拉菜单 → "立即刷新"、"暂停"、"调整间隔"
- **增加刷新策略可视化**：时间线显示上次/下次刷新
- **键盘快捷键**：`Ctrl+R` 立即刷新当前可见面板，`Ctrl+Shift+R` 全量刷新
- **设置页暴露所有刷新参数**：允许用户自定义各域刷新间隔

---

## 问题 3：AI 总结判断逻辑不明，存在重复/低质量输出

### 现象
- 每个子模块的 AI 总结"节制标准"（过滤阈值）不明
- 经常出现：
  - 重复信息（同一事件多个面板重复总结）
  - 无关紧要的内容（次要新闻占用主位置）
  - 不是最新/最重要的（被旧信息覆盖）

### 推测原因（待源码验证）
- [ ] `brief-llm-core.ts` 的 dedup 逻辑阈值不公开
- [ ] `importance-score` 评分模型未明确（`tests/importance-score-parity.test.mjs` 存在但逻辑散落）
- [ ] `feed-date-ranking-uses-effective.test.mts` 提示"effective date"概念，但前端不显示
- [ ] 各域面板的"显著阈值"硬编码，无统一规范
- [ ] LLM 提示词（prompt）针对每个域独立维护，无版本管理

### 排查方向
1. 阅读 `src/services/brief-llm-core.ts` 和 `src/shared/brief-llm-core.ts`
2. 阅读 `scripts/_insights-brief.mjs` 看 prompt 模板
3. 阅读 `tests/importance-score-parity.test.mjs` 看评分公式
4. 阅读 `tests/brief-composer-rule-dedup.test.mjs` 看 dedup 规则
5. 阅读 `tests/feed-date-ranking-uses-effective.test.mts` 看 effective date 机制
6. 列出所有 86 个面板的"显示阈值"配置

### 改进方向（按优先级）
1. **统一"重要性评分"框架**：每个事件 0-100 分，分数阈值由用户配置
2. **统一"时效性"维度**：使用 effective date 排序，旧的自动降权
3. **跨面板 dedup**：基于 embedding 相似度去重（已有 vector-db 基础）
4. **可解释的 AI 总结**：每条总结附带"为什么这条重要"理由
5. **用户反馈循环**：每条总结可"👍/👎"反馈，影响后续权重
6. **Prompt 版本管理**：所有 prompt 模板集中存储 + 版本号
7. **A/B 评估**：保留 2-3 套 prompt 配置，定期对比质量
8. **节流可配置**：在设置页暴露各域的"最大显示数"、"最小重要性"、"时效窗口"

---

## 问题 4：API/新闻源不稳定，国内环境大部分不可用

### 现象
- 部分 API 不稳定（间歇性超时/限流）
- **国内环境下大量新闻源不可用**（重点问题）：
  - Telegram 群（.json 配置文件存在但 API 不可达）
  - Twitter/X（无官方 API）
  - 大量西方媒体 RSS（部分被 GFW 屏蔽）
  - Google News RSS
  - 部分政府/智库站点

### 推测原因（待源码验证）
- [ ] `data/telegram-channels.json` 配置了频道但无代理
- [ ] `vite.config.ts` 的 `rssProxyPlugin` 没有国内镜像
- [ ] `vercel.json` 的 RSS rewrite 走国外源
- [ ] `api/rss-proxy.js` 的白名单不含中文/亚洲媒体
- [ ] `RSS_PROXY_ALLOWED_DOMAINS` 在 vite 配置中重复定义，容易不一致

### 当前已知的不可用源（按重要性）
| 类别 | 源 | 国内可达性 | 替代方案 |
|------|------|------|----------|
| 社交媒体 | Telegram | 不可达 | 需 VPN/代理/或换源 |
| 社交媒体 | Twitter/X | 不可达 | 需代理或换 Nitter |
| 新闻聚合 | Google News | 受限 | 改用 Bing/今日头条 |
| 西方媒体 | BBC, Guardian, CNN | 受限 | 改用国内转译或换源 |
| 财经 | Yahoo Finance | 部分可达 | 用东方财富/同花顺 |
| 政府/智库 | CSIS, CFR, Brookings | 可达但慢 | 直连 |
| 卫星/遥感 | 部分 | 可达 | 直连 |

### 排查方向
1. 阅读 `api/rss-proxy.js` 和 `vite.config.ts` 的 `rssProxyPlugin`
2. 阅读 `shared/rss-allowed-domains.json`（如存在）
3. 阅读 `data/telegram-channels.json` 看配置的频道
4. 阅读 `vercel.json` 的 `/rss/*` rewrites
5. 阅读 `scripts/validate-rss-feeds.mjs` 看验证逻辑

### 改进方向（重要）
1. **国内新闻源补充**：
   - 财新、新浪、网易、腾讯、澎湃、观察者网、虎嗅、36氪
   - 央视新闻、人民日报、新华社（官方）
   - 第一财经、华尔街见闻、东方财富、同花顺
   - 知乎热榜、微博热搜、百度热搜
2. **多源 fallback**：每个 topic 至少配 2-3 个源，主源失败时自动切换
3. **代理层抽象**：
   - 实现 `RssFetcher` 接口，支持 `direct` / `proxy` / `mirror` 三种模式
   - 提供配置文件 `.env.local` 让用户配置可用代理
4. **国内/国外双轨内容**：
   - 检测用户地理位置（IP 库），自动选择源池
   - 变体增加 `cn` 站点（"worldmonitor.app/cn" 或独立子域）
5. **Telegram 替代**：
   - 用 Telegram Bot API 通过代理轮询
   - 或转用国内可达的类似服务（如：可对接 RSSHub 自部署实例）
6. **缓存强化**：每个源在 Redis 缓存 5-30 分钟，避免重复请求失败
7. **健康监控**：失败的源自动降级 + 通知用户"X 源当前不可用"
8. **离线模式**：用户可下载最近一次成功的快照，应急使用

---

## 问题 5：项目定位偏差——"仪表盘" vs "实时预警 + 战略分析"

### 现象
- 当前项目更像"信息仪表盘"——一股脑展示所有数据
- 与项目名 "world monitor" 应有的定位不符
- **应有定位**：及时给出实时突发提醒 + 建设性战略分析
- 部分信息源国内不可达加剧展示混乱

### 根本问题
1. **展示 ≠ 洞察**：把所有信息铺开不等于让用户获得情报
2. **缺乏优先级**：所有数据同等对待，"信号/噪声"比例低
3. **缺乏行动建议**：只显示"发生了什么"，不显示"这意味着什么/该怎么做"
4. **缺乏时间敏感性**：突发事件与背景信息混在一起

### 重新定位（建议）
**WorldMonitor 应该从"信息聚合器"进化为"情报助手"**：

#### 5.1 实时突发提醒（Breaking Alerts）—— 核心 1
- **显眼的警报横幅**：在视图顶部固定位置，红/黄/绿三色编码
- **多级严重度**：
  - 🔴 CRITICAL：直接影响全球安全/经济（如：核设施事件、央行紧急动作）
  - 🟠 HIGH：重大地缘事件（军事冲突升级、领导人突发声明）
  - 🟡 MEDIUM：值得关注（重要经济数据、灾害事件）
  - 🟢 LOW：背景信息（一般新闻）
- **多媒体警报**：声音、桌面通知、Web Push、邮件/短信
- **快速行动入口**：每个警报附带"深度分析"按钮
- **时间戳清晰**：明确显示"X 分钟前发生"而非模糊时间

#### 5.2 战略分析（Strategic Analysis）—— 核心 2
- **AI 战略简报**（区别于新闻摘要）：
  - 不是"事件 A 发生了"
  - 而是"事件 A 发生意味着：1) X 行业将受影响 2) Y 国家可能跟进 3) 历史相似情况下的演变路径"
- **多源交叉验证**：同一事件从政治/经济/军事 3 视角解读
- **预测性分析**：
  - 基于情景的"如果 X 发生，会怎样"分支
  - 历史相似性检索（已有 vector-db 基础）
- **行动建议**：
  - 投资者视角：受影响资产、避险方向
  - 决策者视角：外交/经济杠杆
  - 普通用户视角：实际影响（油价、汇率、供应链）

#### 5.3 信息架构重组
**新首页布局建议**：

```
┌─────────────────────────────────────────────────────────┐
│  🔴 3 个 CRITICAL 警报  |  🟠 7 个 HIGH 警报        [▶]  │  ← 警报条
├─────────────────────────────────────────────────────────┤
│                                                          │
│  📊 战略简报（AI 生成）                                  │  ← 战略层
│  ┌──────────────────────┬──────────────────────┐       │
│  │ 今日重点 1：地缘     │ 今日重点 2：经济      │       │
│  │ - 含义 1 / 2 / 3     │ - 含义 1 / 2 / 3     │       │
│  │ - 历史相似 / 预测    │ - 历史相似 / 预测     │       │
│  └──────────────────────┴──────────────────────┘       │
│                                                          │
│  🗺️ 实时地图（带事件热力图）                            │  ← 视觉层
│                                                          │
│  📰 分类面板网格（按重要性排序，非按时间）              │  ← 细节层
│  [军事] [经济] [气候] [健康] [技术] ...                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### 5.4 用户体验改进
- **"简报模式"** vs **"深度模式"** 切换
  - 简报：只看 CRITICAL + HIGH 警报 + 战略简报
  - 深度：全量数据 + 自定义面板布局
- **"今日重点"** 推送：每天早/中/晚三次主动推送
- **"周报/月报"**：自动生成战略复盘
- **个性化**：用户可声明关注国家/行业，系统智能加权

---

## 待办优先级排序（建议）

| 优先级 | 改进项 | 影响范围 | 工作量估计 |
|--------|--------|----------|------------|
| 🔴 P0 | 修复 UI 卡死 bug | 全部 | 1-2 周（需先定位根因） |
| 🔴 P0 | 暴露刷新控制（手动/可视化） | UX | 1 周 |
| 🟠 P1 | 国内新闻源补充 + 多源 fallback | 数据完整性 | 2-3 周 |
| 🟠 P1 | 重新定位：实时警报 + 战略分析 | 产品形态 | 4-6 周（最大改动） |
| 🟡 P2 | AI 总结可配置 + 可解释 | 内容质量 | 2-3 周 |
| 🟢 P3 | 性能监控面板 | 可维护性 | 1 周 |

---

## 待用户确认的问题

- [ ] 二次开发是否只针对国内环境？还是要做双轨？
- [ ] 是否要新增"中国变体"独立子域（如 cn.worldmonitor.app）？
- [ ] UI 卡死的具体场景（用多久、做什么操作后卡死）能否录制视频？
- [ ] 战略分析的目标用户是谁？（投资者/研究者/普通公众？）
- [ ] 是否要引入付费功能（如更专业的战略简报）？
- [ ] 是否需要多语言（中英双语）UI？

---

## 相关源码索引（待阅读验证）

| 问题 | 关键文件 |
|------|---------|
| UI 卡死 | `src/components/MapContainer.ts`, `src/components/Panel.ts`, `src/app/refresh-scheduler.ts`, `src/workers/*.ts` |
| 刷新机制 | `src/app/refresh-scheduler.ts`, `src/config/variant.ts`, `api/health.js`, `api/bootstrap.js` |
| AI 总结 | `src/services/brief-llm-core.ts`, `src/shared/brief-llm-core.ts`, `scripts/_insights-brief.mjs`, `src/services/ai-classify-queue.ts` |
| API 不稳定 | `api/rss-proxy.js`, `vite.config.ts` (`rssProxyPlugin`), `vercel.json`, `data/telegram-channels.json` |
| 项目定位 | `src/main.ts`, `src/App.ts`, `index.html`, `src/config/variant-meta.ts` |

---

*备忘更新：2026-06-06 — 用户实际使用中反馈，待二次开发处理*
