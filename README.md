# okx-quant-platform

> 一个接入 **OKX（欧易）** 交易所的量化交易平台练手项目。
> 个人学习性质，用于把「策略 → 风控 → 下单 → 回测」这条链路完整跑通一遍。

[![Python](https://img.shields.io/badge/Python-3.11%2B-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.109-green.svg)](https://fastapi.tiangolo.com/)
[![Next.js](https://img.shields.io/badge/Next.js-14-black.svg)](https://nextjs.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## ⚠️ 先看这里

这是一个 **学习/练手项目，不是可以直接实盘交易的系统**。

- 默认运行在**模拟盘**，`LIVE_TRADING_ENABLED` 默认为 `false`
- 交易相关代码**尚未经过充分的实盘验证**，请勿直接用真钱跑
- 代码里的风控、订单持久化、部分策略逻辑还**没有全部实现**（详见下方「当前进度」）
- API Key 请**务必在 OKX 后台只勾选「读取」和「交易」权限，不要开「提现」**

如果你只是想跑起来看看效果，用模拟盘就完全够用了。

---

## 这个项目想解决什么

写策略本身不难，难的是策略之外的那一圈东西：请求怎么签名、模拟盘头怎么带、订单状态怎么同步、亏多了怎么自动停手、下单超时了怎么避免重复下单、密钥怎么不泄露。

这个项目就是把这些「周边工程」都写一遍，让注意力能集中在策略逻辑上。

---

## 技术栈

| 层 | 选型 |
|---|---|
| 后端 | Python 3.11+ / FastAPI / SQLAlchemy 2.0 (async) |
| 数据库 | PostgreSQL 16 |
| 缓存·队列 | Redis 7（Celery + APScheduler 预留） |
| 前端 | Next.js 14 (App Router) / React 18 / Tailwind CSS |
| 容器 | Docker Compose |
| 测试 | pytest + pytest-asyncio |

---

## 快速开始

### 1. 准备环境

需要 Docker 和 Docker Compose。

```bash
git clone https://github.com/<你的用户名>/okx-quant-platform.git
cd okx-quant-platform
cp .env.example .env
```

### 2. 配置 OKX 模拟盘 API Key

1. 登录 [OKX 网页版](https://www.okx.com/account/my-api)，切到**模拟盘**标签页
2. 创建 API Key，权限只勾 **读取 / 交易**，**不要勾提现**
3. 拿到 `API Key` / `Secret Key` / `Passphrase` 三件套，填进 `.env`：

```ini
OKX_API_KEY=你的key
OKX_SECRET_KEY=你的secret
OKX_PASSPHRASE=你的passphrase
OKX_SIMULATED=true
```

> 模拟盘的 Key 只能访问模拟盘，真实资金账户的 Key 是另一套，两者别搞混。
> 建议再把服务器的固定 IP 绑到 API Key 上。

### 3. 启动

```bash
docker compose up -d
```

- 后端：<http://localhost:8001> （API 文档在 `/docs`）
- 前端：<http://localhost:3000>
- 健康检查：`curl http://localhost:8001/api/health`

### 4. 本地开发（不用 Docker）

```bash
# 后端
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8001

# 前端
cd frontend
npm install
npm run dev
```

---

## 项目结构

```
quant-okx-platform/
├── backend/
│   ├── app/
│   │   ├── core/          # 配置、日志、异常、安全工具
│   │   ├── exchange/okx/  # OKX 接入层：签名、REST、WebSocket
│   │   ├── database/      # SQLAlchemy 模型与 Pydantic schema
│   │   ├── market/        # 行情服务（规划中）
│   │   ├── account/       # 账户服务（规划中）
│   │   ├── trade/         # 下单服务、执行引擎、订单路由
│   │   ├── strategy/      # 策略基类、管理器、示例策略
│   │   ├── risk/          # 风控规则、风控管理器、熔断器（规划中）
│   │   ├── backtest/      # 回测引擎、撮合、指标计算
│   │   ├── api/routes/    # FastAPI 路由
│   │   ├── workers/       # 定时任务调度
│   │   └── tests/         # 单元测试
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── app/               # Next.js 页面
│   ├── components/
│   └── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## 核心模块说明

### OKX 接入层 `backend/app/exchange/okx/`

| 文件 | 作用 |
|---|---|
| `signer.py` | REST 签名：`HMAC-SHA256(timestamp + method + requestPath + body)` → Base64 |
| `rest.py` | REST 公共/私有接口封装，模拟盘自动带 `x-simulated-trading: 1` |
| `websocket.py` | 公共行情 / 私有账户频道，含心跳、重连、重订阅 |
| `client.py` | 统一对外客户端 |

**签名规则**（OKX V5）：

```
preHash = timestamp + method + requestPath + body
signature = Base64(HMAC-SHA256(preHash, SecretKey))
```

注意 GET 请求的 query string 属于 `requestPath`，不属于 body。请求头需要 `OK-ACCESS-KEY` / `OK-ACCESS-SIGN` / `OK-ACCESS-TIMESTAMP` / `OK-ACCESS-PASSPHRASE` 四个。

### 策略系统 `backend/app/strategy/`

`StrategyBase` 定义了策略的生命周期钩子：`on_start` / `on_stop` / `on_kline` / `on_tick` / `on_order_update` / `on_trade` / `generate_signal`。

内置三个示例：

- **SMA 均线交叉** (`sma_cross.py`) — 快慢均线金叉买、死叉卖
- **现货网格** (`grid.py`) — 在价格区间内布网格，触发后反向挂单
- **手动下单** (`manual.py`) — 由外部手动触发信号

### 风控系统 `backend/app/risk/`

所有订单在下发前**必须**经过 `RiskManager`，目前实现的规则：

| 规则 | 说明 |
|---|---|
| `LiveTradingGuard` | 实盘模式但 `LIVE_TRADING_ENABLED=false` 时直接拒单 |
| `MaxOrderValueRule` | 单笔名义价值上限 |
| `MaxDailyLossRule` | 单日亏损上限 |
| `MaxPositionRule` | 单币种仓位上限 |
| `DuplicateOrderRule` | 同一信号短时间内不重复下单 |
| `KillSwitchRule` | 全局急停开关 |

`rules.py` 里还写了 `LeverageRule` / `ReduceOnlyRule` / `PriceBandRule`，但**尚未接入 `RiskManager` 的规则列表**。

一键急停：

```bash
curl -X POST http://localhost:8001/api/risk/kill-switch/on
```

### 回测系统 `backend/app/backtest/`

`BacktestEngine` 负责历史数据回放，`BacktestBroker` 模拟撮合（含手续费与滑点），`Metrics` 计算收益指标：

总收益率、最大回撤、胜率、盈亏比、夏普比率、交易明细、权益曲线。

---

## 环境变量说明

完整清单见 [`.env.example`](.env.example)，几个关键的：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `OKX_SIMULATED` | `true` | 模拟盘开关，模拟盘请求会自动带 `x-simulated-trading: 1` |
| `LIVE_TRADING_ENABLED` | `false` | **实盘总开关，默认关闭** |
| `OKX_BASE_URL` | `https://www.okx.com` | 不同地区站点地址不同，按需改 |
| `MAX_ORDER_USDT` | `50` | 单笔最大下单金额 |
| `MAX_DAILY_LOSS_USDT` | `100` | 单日最大亏损 |
| `MAX_POSITION_USDT` | `500` | 单币种最大仓位 |
| `MAX_LEVERAGE` | `3` | 最大杠杆 |
| `SECRET_KEY` | — | JWT 签名密钥，**必须改成随机值** |
| `ENCRYPTION_KEY` | — | API Key 加密密钥，**必须改** |

---

## 运行测试

```bash
cd backend
pip install -r requirements.txt
pytest app/tests -v
```

单元测试覆盖：OKX 签名与请求头、模拟盘 header、风控放行与拦截、`client_order_id` 生成、SMA 策略信号、网格价格生成、回测撮合与指标计算。

---

## 当前进度

按开发阶段划分的完成情况：

| 阶段 | 内容 | 状态 |
|---|---|---|
| 1 | 项目骨架、Docker Compose、配置、README | ✅ 完成 |
| 2 | OKX 签名、REST 客户端、server time / instruments / ticker / balance | ✅ 完成 |
| 3 | 下单、撤单、订单查询、`client_order_id`、订单入库 | ⚠️ 部分（订单未落库） |
| 4 | 公共/私有 WebSocket、心跳、重连、重订阅 | ⚠️ 部分（消息未落库） |
| 5 | 风控管理器与核心规则、kill switch | ⚠️ 部分（部分规则未接入） |
| 6 | 策略基类、管理器、SMA / 网格 / 手动策略 | ✅ 骨架完成 |
| 7 | 回测引擎、撮合、手续费滑点、指标统计 | ✅ 完成 |
| 8 | 前端 Dashboard / 策略页 | ⚠️ 部分（页面较少） |
| 9 | 实盘保护、密钥加密存储、日志脱敏 | ⚠️ 部分（**待补齐**） |

### 已知待补齐的坑

这些是我自己整理的 TODO，欢迎提 PR：

- [ ] `Market` / `Account` 服务层、`OrderRouter`、熔断器（`circuit_breaker.py`）尚未创建
- [ ] 订单 / 成交 / 持仓 / 权益快照 **还没有真正写入数据库**
- [ ] API Key 加密存储（Fernet）与日志脱敏尚未实现
- [ ] `get_server_time` / `get_fills` / `get_order` 等接口待补
- [ ] `LeverageRule` / `ReduceOnlyRule` / `PriceBandRule` 尚未接入风控管理器
- [ ] 前端缺少 Market / Trade / Risk / Backtest 页面
- [ ] `alembic` 迁移脚本目录为空
- [ ] 单元测试之外还缺集成测试与异常测试

---

## 常见错误

| 现象 | 原因 | 处理 |
|---|---|---|
| `50102` Timestamp request expired | 本机时间和 OKX 服务器差太多 | 校准系统时间（Windows 开启「自动设置时间」） |
| 签名错误 `Invalid sign` | SecretKey 填错，或用了另一个环境的 Key | 确认 Key / Secret / Passphrase 三件套同属一套环境 |
| 返回空数据 | 模拟盘 Key 访问了实盘接口 | 确认 `OKX_SIMULATED=true` 且请求带了 `x-simulated-trading: 1` |
| 余额不足 | 模拟盘账户没有 USDT 余额 | 到模拟盘后台手动充值/领币 |
| 产品不存在 | `inst_id` 写错 | 现货是 `BTC-USDT`，合约是 `BTC-USDT-SWAP`，用 `GET /api/market/instruments` 查 |
| 请求限速 | 短时间请求太频繁 | 退避重试，OKX 单 IP 有速率限制 |
| WebSocket 登录失败 | 用了实盘 Key 连私有频道 | 模拟盘 Key 请配合模拟盘环境使用 |

---

## 安全建议

- API Key **只勾读取和交易权限，绝对不开提现**
- 建议在 OKX 后台绑定服务器固定 IP
- 密钥只放 `.env`（已被 `.gitignore` 忽略），**不要写进代码或提交到 Git**
- 浏览器端不保存 OKX 密钥
- 用完记得把 `SECRET_KEY` 和 `ENCRYPTION_KEY` 换成随机值

---

## 免责声明

本项目仅供学习交流使用，不构成任何投资建议。量化交易有风险，实盘使用前请自行充分测试，风险自负。

---

## License

[MIT](LICENSE)
