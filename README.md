# vpn-gate-node-check

一个用于检测和展示 VPN Gate SSTP 节点状态的自动化项目。该项目会从 VPN Gate 数据源抓取节点信息，筛选出支持 TCP 入口的 SSTP 节点，并通过 Cloudflare Worker 发起连通性检测；最后生成静态公开数据与前端页面，并部署到 GitHub Pages。

## 项目简介

本项目的核心目标是：

- 自动获取 VPN Gate 原始中继节点
- 过滤出可用于 SSTP 的节点
- 并发检测节点是否真实可用
- 按国家分组并统计可用节点
- 生成静态页面和订阅文本，便于直接使用

它特别适合用于：

- 不定时更新的 VPN 节点状态监控
- 生成可导入的 edgetunnel 配置
- 为静态站点提供公开的节点列表
- 方便在 GitHub Pages 上部署展示

## 功能特性

### 1. 自动抓取 VPN Gate 节点

脚本会优先访问官方 VPN Gate API；若接口失败，则自动回退到 GitHub 镜像 JSON。这样可以显著降低数据源中断带来的影响。

### 2. 过滤出可用的 SSTP 节点

并非所有 VPN Gate 节点都能用于 SSTP。脚本会解析 OpenVPN 配置数据，筛选出具备 TCP 入口的中继节点，并丢弃 UDP-only 或不符合要求的条目。

### 3. 实时检测可用性

脚本会并发请求已部署的 Cloudflare Worker，判断节点是否能够真正连接成功。最终以 Worker 返回的 `success` 字段作为可用性判定依据，而不是仅凭 HTTP 请求成功。

### 4. 国家分组与统计

检测成功的节点会按国家分组，统计：

- 可用节点数
- 国家数量
- 住宅/数据中心估算数量
- 节点延迟等信息

### 5. 生成多种输出内容

脚本会生成以下静态输出：

- `public/data.json`：结构化检测结果
- `public/index.html`：可视化展示页
- `public/chains.txt`：edgetunnel 链式代理清单
- `public/hosts.txt`：自定义优选 IP 清单
- `public/sub.txt`：完整的 vless 订阅内容

### 6. 自动部署到 GitHub Pages

通过 GitHub Actions 定时执行脚本，并自动发布到 GitHub Pages，确保页面数据持续更新。

## 项目结构

```text
vpn-gate-node-check/
├── .github/
│   └── workflows/
│       └── check.yml
├── web/
│   └── index.html
├── .gitignore
├── README.md
├── requirements.txt
├── vpngate.py
└── public/   # 运行脚本后生成，部署页面使用
```

## 核心文件说明

### `vpngate.py`

这是整个项目的核心脚本，负责：

- 获取数据源
- 解析 VPN Gate 原始 CSV / JSON
- 过滤 SSTP 节点
- 去重
- 并发检测可用性
- 写出最终数据文件

### `web/index.html`

前端展示模板，用于：

- 读取 `data.json`
- 按国家展示节点列表
- 支持筛选住宅/数据中心/未知节点
- 提供复制 `sstp://` 链接等交互

### `.github/workflows/check.yml`

GitHub Actions 工作流，配置如下：

- 每 30 分钟执行一次
- 自动安装依赖
- 执行 Python 脚本
- 自动启用 GitHub Pages
- 上传 `public` 目录
- 部署静态页面

### `requirements.txt`

当前仅依赖：

```txt
requests
```

## 自动化流程

整个流程如下：

1. 拉取 VPN Gate 原始节点
2. 过滤出满足 SSTP 条件的节点
3. 去重处理
4. 通过 Cloudflare Worker 检查节点可用性
5. 按国家汇总并生成 JSON
6. 写出前端页面和订阅文件
7. GitHub Actions 自动部署到 Pages

## 运行方式

### 本地运行

```bash
pip install -r requirements.txt
python vpngate.py
```

脚本支持通过环境变量覆盖默认配置，例如：

```bash
export CHECK_WORKER="https://check.helei.kdns.fr/check?sstp=vpn:vpn@"
export CHECK_CONCURRENCY="32"
export CHECK_TIMEOUT="90"
python vpngate.py
```

### 常用环境变量

- `VPNGATE_API`：VPN Gate 官方数据源地址
- `VPNGATE_MIRROR`：回退镜像地址
- `CHECK_WORKER`：检测 Worker 地址
- `CHECK_CONCURRENCY`：并发检测数量
- `CHECK_TIMEOUT`：单个检测超时时间
- `PUBLIC_DIR`：输出目录
- `MAX_CHECK_NODES`：限制测试节点数量（用于本地调试）

## 生成结果说明

### `data.json`

该文件包含：

- 生成时间
- 数据源信息
- 统计摘要
- 按国家分组后的节点信息
- 每个节点的延迟、出口、类型等字段

### `chains.txt`

用于 edgetunnel 的链式代理配置，按国家分组并优先展示住宅 IP 节点。

### `hosts.txt`

适合粘贴到 edgetunnel 的“自定义优选 IP”区域，方便循环分配入口地址。

### `sub.txt`

完整订阅文本，适合直接粘贴到服务端的订阅链接配置中，客户端可以自动轮换节点。

## 部署说明

项目使用 GitHub Pages 作为展示站点，部署地址为：

```text
https://jerylihub.github.io/gate/
```

GitHub Actions 会在每次定时任务成功后更新页面内容。

## 注意事项

- 项目依赖 VPN Gate 公开数据源与 Cloudflare Worker 检测服务，因此数据和可用性受外部服务状态影响。
- 只有 `success = true` 的节点会被视为可用。
- 如果官方数据源和镜像源都失效，或 Worker 全部异常，脚本会返回失败状态，避免生成错误数据。
- 住宅/数据中心类型是估算值，主要用于排序和分类展示，不是严格的地理或 ISP 认证。

## 适用场景

- 需要持续监控 VPN Gate 节点状态
- 希望以静态页面形式展示节点列表
- 需要生成 edgetunnel 订阅或优选 IP 文本
- 希望低成本部署在 GitHub Pages 上

## 许可证

本项目未声明明确许可证，默认按仓库实际设置执行。若需要正式商业或分发使用，请先确认仓库中的许可要求。

## 结论

`vpn-gate-node-check` 是一个非常适合 GitHub Pages 的自动化节点检测与展示项目。它将外部 VPN 节点抓取、筛选、检测、分组、展示和部署串联在一起，适合用于高频更新、低运维成本的节点状态监控场景。

---

如果你需要，我也可以继续为这个仓库补上一份更偏“项目介绍型”的英文版 README，或者进一步整理成适合 GitHub 展示风格的精简版说明。
