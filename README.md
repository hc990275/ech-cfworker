# ECH - WebSocket Proxy Server

> 基于 Cloudflare Workers 构建的轻量级 WebSocket 代理服务。内置全栈管理面板、动态 Token 鉴权、全球优选 IP 订阅生成能力。

## ✨ 核心特性

### 🔐 动态 Token 鉴权体系
- 基于远程 `token.json` 的多 Token 鉴权，支持有效期控制
- 内存级缓存（60s TTL）加速鉴权响应
- 内置兜底配置，远程拉取失败时自动降级

### 🌐 双协议 WebSocket 代理
- **二进制协议**：完整 VLESS 协议头解析（版本/UUID/命令/地址类型/端口），UUID 即为鉴权 Token，兼容 v2rayNG / NekoBox 等客户端
- **文本协议**：ECH 自定义协议（CONNECT / DATA / CLOSE 指令），支持全格式地址解析
- 首包自动检测：`ArrayBuffer` 走二进制协议，文本走 ECH 协议，两套协议透明共存
- 内置 Cloudflare 网络容错重试机制

### 🏠 自助主页控制台 (`/`)
- 玻璃拟物风格的服务器运行时长计时器
- Token 到期时间查询入口（精确到秒），**支持按备注模糊搜索**
- 优选 IP 面板：智能推荐最佳地区（支持一键勾选美/日/港/台/新等），并且**自动屏蔽无效或不可用地区**
- 无限流量美化展示，并可一键生成 Base64 订阅配置链接

### ⚙️ 管理后台 (`/admin`)
- **全新卡片式响应布局**：在手机设备上（<768px）优雅地自动折叠为独立的卡片式阅读体验，拒绝传统表格的横向滚动折行
- **模态弹窗编辑**：独立居中 Modal 弹窗替代原生列表行内编辑，界面更清爽美观
- **智能搜索过滤**：新增实时关键字搜索，支持通过 Token、UUID 或备注快速筛选海量记录
- **全参数高度定制化**：支持每个 Token 独立设置订阅路径（`subPath`）、指定下发地区（`regions`）、以及每地区 IP 取用上限（`ipLimit`）
- **灵活有效期流转**：内置预设有效周期快速按钮（1天/1周/1月/1年/永久有效）与自定义天数支持，由北京时间精确兜底计算
- 一站式 GitHub 配置同步，完美解决 SHA 热冲突，杜绝 Cloudflare 边缘缓存延误

### 🌍 优选 IP 智能订阅系统
- 优选 IP 数据采集逻辑源自极客项目 `CF-Worker-BestIP-collector`
- 按后台设定的 `subPath`（如：`/111`）生成极其干净纯粹的极简订阅直出端点，客户端配置零门槛
- 节点别名高度定制识别（例：`🇯🇵 日本 01`），自动使用分配的优选 IP，SNI/Host 保持 Worker 原生域

## 📁 目录结构

```
ceh/
├── ech.js          # 主程序（截至 v1.7.0）- Cloudflare Worker 核心入口
├── ech_v1.js       # v1.5.0 历史备份
├── ech_v2.js       # v1.5.0 历史备份
├── _worker.js      # check.mjs 配套的 Worker
├── check.mjs       # 辅助检测模块
├── test.js         # 测试脚本
├── token.json      # Token 配置模版
├── CHANGELOG.md    # 版本变更记录
└── README.md       # 本文档
```

## 🚀 部署指南

### 1. 创建 Worker
在 Cloudflare Dashboard 创建一个新的 Worker，将 `ech.js` 的内容粘贴到编辑器中并部署。

### 2. 配置环境变量

在 Worker 设置 → 环境变量页面配置以下参数：

| 变量名 | 必填 | 说明 |
|--------|------|------|
| `ADMIN_PASSWORD` | ⚠️ 强烈推荐 | 管理后台密码。未设置时 `/admin` 返回 403 |
| `GITHUB_TOKEN` | 选填 | GitHub PAT（需 repo 写入权限），用于管理面板一键推送配置 |
| `TOKEN_JSON_URL` | 选填 | 远程 `token.json` 的直链地址，系统从此 URL 拉取 Token 列表 |

### 3. 配置 token.json

在 GitHub 仓库中维护如下格式的 JSON 配置：

```json
{
  "global": {
    "SERVER_START_TIME": "2024-01-01T00:00:00.000Z"
  },
  "tokens": [
    {
      "token": "d7a4b0d1...",
      "remark": "My Phone",
      "subPath": "111",
      "regions": ["JP", "US", "HK"],
      "ipLimit": 5,
      "expire": "2025-01-01T00:00:00.000Z"
    }
  ]
}
```

*   `global`：全局配置，可存放服务器启动时间等信息
*   `tokens`：Token 数组
    *   `token`：(必填) Token 标识，建议使用 UUID
    *   `remark`：(可选) 备注信息，用于备忘和**主页快捷查询**
    *   `subPath`：(可选) **管理员指定的自定义订阅路径**（例如填 `111`，订阅直连 URL 为 `https://你的域名/111`）
    *   `regions`：(可选) 分配给该用户的优选地区，未设置时默认使用 `US,JP,HK,TW,SG,KH,KR,PH`
    *   `ipLimit`：(可选) 用户每个地区获取的优选 IP 数量上限，默认 5
    *   `expire`：(可选) 过期时间 (ISO 8601)，无此字段则永久有效

## 📡 API 参考

### 公开接口（无需管理密码）

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/check-token` | Token / 备注查询。Body: `{ "token": "xxx" }`，支持按备注模糊匹配 |
| `GET` | `/api/regions` | 返回可用地区列表 |
| `POST` | `/api/ipsel` | 根据地区筛选优选 IP 并生成订阅。Body: `{ "token": "xxx", "regions": ["JP","US"], "limit": 10 }` |
| `GET` | `/sub/{token}?regions=JP,US` | 订阅直出（Base64），每地区上限取 Token 配置的 `ipLimit` |

### 管理接口（需 Authorization Header = ADMIN_PASSWORD）

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/tokens` | 读取全部 Token 配置 |
| `PUT` | `/api/tokens` | 写入 Token 配置到 GitHub |

## 🛠 技术架构

- **运行时**：Cloudflare Workers（V8 Isolate）
- **语言**：纯 JavaScript（ES Module）
- **存储**：GitHub 仓库 JSON 文件（通过 REST API 读写）
- **缓存**：内存级 Token 配置缓存（60s TTL）
- **UI**：内嵌 HTML + Vanilla JS，无外部依赖
- **网络**：`cloudflare:sockets` TCP 连接 + WebSocket 双向桥接

## 📋 版本历史

详见 [CHANGELOG.md](./CHANGELOG.md)

**当前版本：v1.7.2**
