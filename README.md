<div align="center">

# 县域卡车物流生态云平台

**County Truck Logistics Ecosystem Cloud Platform**

把「货主找车」从打电话变成线上闭环的开源微服务系统

[![Java](https://img.shields.io/badge/Java-17-orange?style=flat-square&logo=openjdk)](https://adoptium.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.5-brightgreen?style=flat-square&logo=springboot)](https://spring.io/projects/spring-boot)
[![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-2023.0.1-blue?style=flat-square)](https://spring.io/projects/spring-cloud)
[![Vue](https://img.shields.io/badge/Vue-3.5-42b883?style=flat-square&logo=vuedotjs)](https://vuejs.org/)
[![MySQL](https://img.shields.io/badge/MySQL-5.7%20%7C%208.0-4479a1?style=flat-square&logo=mysql)](https://www.mysql.com/)
[![Tests](https://img.shields.io/badge/tests-480%20passed-success?style=flat-square)](#-测试)

**简体中文** | [English](#english)

</div>

---

## 📖 这是什么

一套面向**县域物流**场景的完整微服务系统，覆盖「货主发单 → 司机接单 → 运输执行 → 电子回单 → 财务对账」全流程。

不是概念演示或 CRUD 脚手架，而是一个**可真实部署运行**的工程：

| 指标 | 数值 |
|---|---|
| 微服务 | 11 个 + 1 网关 + 1 公共模块 |
| 对外接口 | 159 个 |
| 数据表 | 43 张 |
| Java 主源码 | 329 个文件 / 约 27,000 行 |
| 前端页面 | 32 个（Vue） |
| Python（AI） | 13 个文件 / 约 2,000 行 |
| **自动化测试** | **480 项，全部通过** |

---

## ✨ 核心特性

### 交易撮合

- **货源检索**：车型 / 线路 / 时间窗 / 重量区间 / 关键词多条件组合
- **双引擎可切换**：`CargoSearchEngine` 接口下提供 MySQL（零依赖）与 Lucene 9（中文分词 + BM25 相关性排序）两套实现，通过 `platform.search.type` 一行配置切换
- **报价与竞价**：一口价 / 限时竞价双模式，竞价强制单调递增
- **并发防重复成单**：应用层状态校验 + 乐观锁 + 数据库唯一索引**三重防护**，有真并发测试覆盖

### 运输管理

- **四节点打卡**：装货 → 出发 → 到达 → 签收，状态机强制合法流转
- **轨迹削峰**：位置上报只入队不落库（毫秒级返回），后台批量消费落库
- **异常报警**：偏航 / 停车超时 / 运输超时 / 失联四类规则引擎，阈值可配
- **防里程欺诈**：相邻定位点距离 > 50km 判定为定位漂移，不计入里程

### 结算与风控

- **运费自动计算**：起步价 + 超里程 × 车型系数 × 重量系数（纯函数，可单测）
- **双向对账**：以 `order_flow` 流水为唯一依据，差异逐单列明
- **支付验签 + 幂等**：支付网关重复通知不会重复记账
- **跨企业越权防护**：开票等资金操作按付款方过滤

### 生态服务

维修 / 加油 / 保险 / ETC / 配件商城五类服务，佣金在**核销时**计提（非下单时），避免未核销订单取消后的冲正。

### 数据与监管

- 数据驾驶舱、运价指数、供需匹配度分析
- **政府监管看板只读**：无任何写接口，不暴露个人信息
- 报表导出（CSV 带 BOM，Excel 直接打开中文不乱码）

---

## 🏗️ 技术架构

```
                          ┌──────────────┐
   企业端 Web ──┐          │              │
   司机端 uni-app├──►  网关 Gateway (8080) ──► 11 个微服务 ──► MySQL
   监管端 Web ──┘      鉴权 · 限流 · 路由      │                   │
                                              │                   ├─ Redis（令牌黑名单/限流/Stream）
                                              │                   └─ MinIO / 本地存储（对象存储）
                                              ▼
                                     ai-gateway (9010) ──► ai-service (Python, 9020)
```

### 技术选型

| 层次 | 选型 | 版本 |
|---|---|---|
| 运行时 | JDK | 17 |
| 框架 | Spring Boot | 3.2.5 |
| 微服务 | Spring Cloud | 2023.0.1 |
| ORM | MyBatis-Plus | 3.5.5 |
| 数据库 | MySQL | 5.7 / 8.0 |
| 缓存与队列 | Redis | 5.0+ |
| 检索 | Lucene | 9.11.1（Apache-2.0） |
| 对象存储 | MinIO / 本地磁盘 | minio-java 8.5.10 |
| 令牌 | JJWT | 0.12.5 |
| 报表 | Apache POI / EasyExcel | 5.2.5 / 3.3.4 |
| 前端 | Vue 3 + TypeScript + Element Plus | 3.5 / 5.6 / 2.8 |
| 移动端 | uni-app | — |
| AI | FastAPI + Pydantic | 0.110 / 2.6 |
| 可观测 | Micrometer + Prometheus | 1.12.5 |

### 为什么不用更重的技术栈

县域场景最怕「系统太重、运维跟不上」，因此做了这些取舍：

- **不用 Elasticsearch**：需要独立集群（≥1GB 堆）+ 守护进程。Lucene 以进程内库形态提供中文分词与相关性排序，零额外进程
- **不用 Nacos**：静态路由即可，单县单实例部署不值得引入注册中心
- **Kafka → Redis Stream**：消息语义完全对齐（消费组 + 不 ACK 重投 + 积压可监控），省掉独立中间件
- **不强制 K8s**：提供 Docker Compose 方案，单机可跑

> 这些都是**通过 `CargoSearchEngine` / `MessageQueue` / `ObjectStorage` 适配层隔离**的，
> 随时可替换为 ES / Kafka / MinIO 正式实现，业务代码零改动。

---

## 🚀 快速开始

完整教程见 **[docs/环境搭建说明.md](docs/环境搭建说明.md)**（12 章手把手，每步都有验证方法）。

### 环境要求

| 软件 | 版本 | 说明 |
|---|---|---|
| JDK | **17** | 必须 17，8 和 21 都会构建失败 |
| Maven | 3.8+ | 路径不能含空格与中文 |
| MySQL | 5.7 / 8.0 | 字符集必须 `utf8mb4` |
| Redis | 5.0+ | Windows 用 [microsoftarchive/redis](https://github.com/microsoftarchive/redis/releases) |
| Node.js | 18+ | 推荐 20 / 22 |

> **内存要求 8GB 以上**，推荐 16GB（需同时运行 11 个服务 + 中间件 + 前端）。

### 三步启动

```bash
# 1. 导入数据库（43 张表 + 初始配置）
mysql -uroot -p county_truck < sql/01_schema.sql

# 2. 编译（项目自带构建脚本，含模块一致性自检）
python tools/build.py --install

# 3. 启动后端（按依赖顺序，逐个终端执行）
mvn -pl services/user-auth spring-boot:run
mvn -pl services/cargo    spring-boot:run
# ... 其余服务同理，网关最后启动
mvn -pl gateway spring-boot:run
```

启动前端：

```bash
cd web
npm install
npm run dev      # http://localhost:5173
```

### 服务端口

| 服务 | 端口 | 服务 | 端口 |
|---|---|---|---|
| gateway | 8080 | tms | 9006 |
| user-auth | 9001 | settlement | 9007 |
| file | 9002 | ecosystem | 9008 |
| capacity | 9003 | member | 9009 |
| cargo | 9004 | ai-gateway | 9010 |
| dispatch | 9005 | datacenter | 9011 |

### 第一个账号

系统**不预置任何默认账号**（安全考虑），请先注册：

```bash
curl -X POST http://127.0.0.1:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testcompany","password":"123456","phone":"13800001111","userType":"COMPANY"}'
```

> `ADMIN` / `supervision` 角色**不能通过注册创建**（防越权提权），需在数据库中改 `user_type`。

---

## 📁 项目结构

```
county-platform/
├── common/                   公共模块
│   ├── adapter/               适配层（MessageQueue / ObjectStorage / StorageFactory）
│   ├── config/                公共配置
│   ├── constant/              常量与错误码
│   ├── crypto/                AES 加解密
│   ├── jwt/                   令牌解析
│   ├── page/                  统一分页
│   └── statemachine/          状态机基类
├── gateway/                   网关（鉴权 / 限流 / 路由 / CORS）
├── services/
│   ├── user-auth/             注册登录、企业司机认证
│   ├── file/                  文件上传下载、预签名
│   ├── capacity/              车辆、证件、司机信用
│   ├── cargo/                 货源、报价、竞价、消息
│   ├── dispatch/              智能调度推荐
│   ├── tms/                   运单、轨迹、报警
│   ├── settlement/            运费、结算、对账、发票
│   ├── ecosystem/             维修/加油/保险/配件
│   ├── member/                会员、积分、等级
│   ├── ai-gateway/            AI 能力网关
│   └── datacenter/            数据看板、报表、监管
├── ai-service/                AI 算法服务（Python + FastAPI）
├── web/                       企业端 / 管理端 / 监管端（Vue 3）
├── driver-app/                司机端（uni-app）
├── enterprise-app/            企业小程序端（规划中）
├── sql/01_schema.sql          数据库脚本
├── tools/build.py             构建脚本（--install / --tests / --only）
├── tools/s3-stub/             S3 协议桩服务（用于离线验证 MinIO 接入）
├── deploy/                    Docker Compose / Nginx / K8s / 备份脚本
└── docs/                      全部文档
```

---

## 🧪 测试

```bash
python tools/build.py --tests      # 全量测试
mvn -pl services/cargo test        # 单模块
mvn -pl services/cargo test -Dtest=LuceneCargoSearchEngineTest
```

**当前状态：14 模块 BUILD SUCCESS，480 项测试 Failures=0 / Errors=0 / Skipped=0**

测试重点覆盖：

| 领域 | 覆盖内容 |
|---|---|
| 并发安全 | 2 线程同时成单，断言恰好 1 成功 1 失败 |
| 资金一致性 | 双向对账差异检出、支付验签、重复通知幂等 |
| 证件边界 | 29/30/31 天 / 当日 / 已过期 的临期判定 |
| 检索正确性 | 中文分词、相关性排序、多条件取交集、区间闭边界、分页不漏不重 |
| 安全 | 跨企业越权、预签名越权、路径穿越、直传票据伪造 |
| 业务规则 | 佣金幂等与核销时点、库存防超卖、油价优惠取更低不叠加 |

### 特殊测试开关

MinIO 接入测试需要 S3 兼容服务端，未提供环境变量时**自动跳过**（不影响常规构建）：

```bash
python tools/s3-stub/s3_stub_server.py --port 19000
S3_STUB_URL=http://127.0.0.1:19000 python tools/build.py --only common --tests
```

---

## 📚 文档

| 文档 | 说明 |
|---|---|
| [环境搭建说明](docs/环境搭建说明.md) | 从零部署，12 章手把手，新手可照做 |
| [功能使用说明](docs/功能使用说明.md) | 5 个角色 × 全部页面，含完整业务流程 |
| [对外 API 文档](docs/api/对外API文档_V1.0.md) | 全部接口的入参、返回、示例 |
| [内部接口清单](docs/api/内部接口清单.md) | 服务间调用接口（不对外） |
| [部署手册](docs/deploy/部署手册.md) | 生产环境部署 |
| [一致性审计报告](docs/计划与代码一致性审计报告_2026-10-11.md) | **计划文档 vs 代码的逐项核查结论** |
| [实施计划与进度](县域卡车物流生态云平台_可行实施计划与进度跟踪_V1.0.md) | 85 项任务、9 阶段、进度台账 |

---

## 🔐 配置与安全

### 必须替换的默认密钥

```yaml
platform:
  jwt:
    secret: ${JWT_SECRET:}                    # 生产必填，泄漏=任何人可伪造登录
  crypto:
    aes-key: ${PLATFORM_AES_KEY:}              # 32 位 hex，泄漏=敏感字段可解密
```

> ⚠️ **修改 `aes-key` 会导致历史加密数据无法解密**，请谨慎操作。

### 数据安全设计

- 敏感字段（手机号、身份证、银行卡）**AES 加密存储**
- 证件与回单通过**短期预签名 URL** 分发，桶不开匿名读
- 监管端**只读**，无写接口，不暴露个人信息
- 内部接口统一 `/api/internal/**` 前缀，网关限制仅服务间可访问

---

## 📊 项目状态

**已完成并经测试验证**：

- ✅ 货源发布、多条件检索（Lucene 中文分词）
- ✅ 报价 / 竞价 / 成单（含并发防重）
- ✅ 运单状态机、轨迹上报与削峰
- ✅ 异常报警规则引擎
- ✅ 电子回单 PDF 渲染
- ✅ 运费计算、结算对账、发票申请
- ✅ 车辆证件管理与临期拦截
- ✅ 生态服务五类（维修/加油/保险/ETC/配件）
- ✅ 会员积分与等级体系
- ✅ 数据看板、报表导出、政府监管看板
- ✅ AI 网关（报价/客服/风控/分析）

**进行中 / 待完善**：

- ⬜ 微信小程序端（需 AppID 与微信开发者工具）
- ⬜ 部分业务的前端页面入口
- ⬜ 人脸核身、短信通道、真实支付渠道（需三方资质）

> 详细的完成度核查（含已完成项的真实状态说明）见
> [一致性审计报告](docs/计划与代码一致性审计报告_2026-10-11.md)。
> **我们坚持文档里的 ✅ 必须有「有调用方 + 有入口 + 方法体非空」三重证据支撑。**

---

## 🤝 参与共建

非常欢迎，尤其是有**县域物流实操经验**的同行：

- 🐛 **提 Issue**：发现 bug 或功能缺失
- 💡 **提建议**：产品设计与流程优化
- 🔧 **提 PR**：代码改进
- 📖 **补文档**：尤其是行业术语与业务流程说明

我们特别希望听到：
- 县域货运的真实痛点（纸面上的流程和实际跑的不一样）
- 中小物流企业的 IT 能力边界（决定系统该做成什么样）
- 现有系统对接的坑

---

## 📄 License

**本项目暂未指定开源许可证。**

在添加许可证文件前，默认适用「保留所有权利」条款 —— 即代码仅供内部研究与参考，
**未经许可不得商用或再分发**。

如需商用或二次开发，请联系项目方授权。

> 依赖组件的许可证：
> Spring Boot / Spring Cloud / MyBatis-Plus / Lucene（Apache-2.0）、
> minio-java（Apache-2.0）、
> Vue / Element Plus（MIT）、
> FastAPI（MIT）、
> **OpenPDF 2.0.3（LGPL-3.0 / MPL-2.0 双许可）** —— 商业使用前建议法务确认，
> 详见 [OpenPDF License FAQ](https://openpdf.softwareland.de/license-faq/)。

---

<div align="center">

**如果这个项目对你有帮助，请给个 ⭐ Star**

这能帮助更多人找到它，也让我们知道哪些功能值得优先完善。

</div>

---

<a name="english"></a>

## English

**County Truck Logistics Ecosystem Cloud Platform** — an open-source microservices system for county-level freight logistics in China.

Covers the full loop: cargo posting → driver bidding → transport execution → electronic receipt → financial reconciliation.

- **11 microservices** + gateway + shared module
- **159 REST endpoints**, **43 database tables**
- **480 automated tests**, all passing
- **Java 17 / Spring Boot 3.2.5 / Spring Cloud 2023 / MyBatis-Plus / MySQL / Redis / Lucene / Vue 3 / uni-app / FastAPI**

Design philosophy: **optimize for low operational overhead in county-level deployments** — Elasticsearch replaced with in-process Lucene, Kafka with Redis Streams, no mandatory Kubernetes. All alternatives are isolated behind adapter interfaces (`MessageQueue` / `ObjectStorage` / `CargoSearchEngine`) so they can be swapped without touching business logic.

📖 See [docs/环境搭建说明.md](docs/环境搭建说明.md) for the deployment guide (Chinese).

**License:** Not yet specified — all rights reserved. Contact the authors for commercial use.

</div>

---

## 🏢 关于我们

<p align="center">
  <a href="http://www.net188.net">
    <img src="http://www.net188.net/images/logo1.png" alt="Net188 Logo" width="200" />
  </a>
</p>

<p align="center">
  <strong>Net188 · 互联网技术服务</strong>
</p>

<p align="center">
  专注于跨平台应用开发、AI Agent 集成与大模型应用落地。<br/>
  提供从产品设计、开发实施到部署运维的全栈技术解决方案。
</p>

<p align="center">
  🌐 <a href="http://www.net188.net"><strong>www.net188.net</strong></a>
</p>

---
