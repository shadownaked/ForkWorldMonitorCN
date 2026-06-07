# WorldMonitor 安装运行测试报告

> 测试日期：2026-06-06  
> 测试环境：远程沙箱（Linux x86_64）  
> 项目版本：v2.8.0

---

## 1. 环境信息

| 项目 | 值 |
|------|-----|
| 操作系统 | Ubuntu 24.04.3 LTS (Noble Numbat) |
| 内核版本 | Linux 6.18.5 x86_64 |
| Node.js | v24.15.0 |
| npm | 11.4.2 |
| 工作目录 | /workspace/worldmonitor |

---

## 2. 依赖安装

### 2.1 安装命令

```bash
npm install --ignore-scripts
```

### 2.2 安装结果

| 指标 | 值 |
|------|-----|
| 状态 | 成功 |
| 安装的包数量 | 918 个 |
| 安全漏洞 | 13 个 moderate severity |
| 安装方式 | 跳过 postinstall 脚本（sharp 二进制下载在沙箱中失败） |

### 2.3 已知问题

**sharp 二进制下载失败**：
- 错误：`Client network socket disconnected before secure TLS connection was established`
- 原因：沙箱中存在 HTTP 代理（`http://127.0.0.1:18080`）导致 sharp 的 libvips 二进制下载失败
- 解决：使用 `--ignore-scripts` 跳过安装后脚本
- 影响：sharp 是 blog-site 的依赖，不影响主项目功能

**blog-site 子项目依赖**：
- 需单独执行 `cd blog-site && npm install` 安装
- 安装成功（168 packages）

---

## 3. 类型检查

### 3.1 执行命令

```bash
npx tsc --noEmit
```

### 3.2 结果

| 指标 | 值 |
|------|-----|
| 状态 | **通过** |
| 错误数 | 0 |
| 退出码 | 0 |

TypeScript 类型检查完全通过，无任何类型错误。

---

## 4. 构建测试

### 4.1 执行命令

```bash
npm run build:full
```

### 4.2 构建结果

| 指标 | 值 |
|------|-----|
| 状态 | **成功** |
| 构建耗时 | 39.79 秒 |
| 构建产物 | dist/ 目录 |

### 4.3 构建产物分析

| 文件 | 大小 | Gzip |
|------|------|------|
| main.js | 720.80 kB | 208.19 kB |
| panels.js | 2,451.47 kB | 649.87 kB |
| MapContainer.js | 2,180.19 kB | 583.03 kB |
| maplibre.js | 1,106.78 kB | 297.70 kB |
| deck-stack.js | 1,047.88 kB | 287.27 kB |
| hls.js | 523.81 kB | 162.06 kB |
| sentry.js | 431.70 kB | 142.53 kB |
| transformers.js | 4,650.57 kB | 1,346.16 kB |

**PWA Service Worker**：
- 预缓存条目：134 个文件
- 预缓存大小：14,299.53 KiB

**警告**：
- 部分 chunk 超过 1200 kB 警告阈值（panels、MapContainer、maplibre、deck-stack、transformers）
- 这些是预期行为：项目已通过 `chunkSizeWarningLimit: 1200` 和手动分块策略优化

---

## 5. 开发服务器启动测试

### 5.1 执行命令

```bash
npm run dev
```

### 5.2 结果

| 指标 | 值 |
|------|-----|
| 状态 | **成功启动** |
| 启动耗时 | 645 ms |
| 监听地址 | http://localhost:3000/ |
| Vite 版本 | v6.4.2 |

### 5.3 注意事项

- `xdg-open` 错误（`spawn xdg-open ENOENT`）：这是因为沙箱没有图形浏览器，Vite 尝试自动打开浏览器失败。这是正常行为，不影响开发服务器功能。
- 开发服务器能正常响应 API 路由（通过 vite.config.ts 中的 sebufApiPlugin、rssProxyPlugin、polymarketPlugin 等插件）

---

## 6. 已知限制

以下功能无法在沙箱环境中完整测试：

| 限制项 | 原因 |
|--------|------|
| Tauri 桌面应用 | 需要 Rust 编译环境 + 图形显示 |
| 完整 E2E 测试 | 需要 Playwright 浏览器 |
| 需要 API Key 的功能 | 沙箱未配置 API 密钥（如 FRED、Yahoo Finance、OpenSky 等） |
| Redis 连接 | 沙箱未配置 Upstash Redis 连接 |
| Convex 后端 | 沙箱未配置 Convex 部署 |
| RSS Feed 验证 | 需要外网访问（部分 RSS 源可能被沙箱网络限制） |
| sharp 图像处理 | 二进制下载受沙箱代理限制 |
| Ollama 本地 AI | 沙箱未安装 Ollama |
| 实时数据流（AIS、WebSocket） | 需要 Railway relay 服务 |

---

## 7. 总结

| 测试项 | 结果 |
|--------|------|
| 源码克隆 | 通过 |
| 依赖安装 | 通过（使用 --ignore-scripts 绕过 sharp 问题） |
| TypeScript 类型检查 | 通过（0 错误） |
| 生产构建 | 通过（39.79s） |
| 开发服务器启动 | 通过（645ms 启动） |
| 整体评估 | **可正常开发与构建** |

项目在沙箱环境中可以正常进行 TypeScript 编译、Vite 开发服务器运行和生产构建。需要外部 API 密钥和外部服务（Redis、Railway、Convex）的功能无法在沙箱中测试，但这不影响代码分析和二次开发。

---

*报告生成时间：2026-06-06*