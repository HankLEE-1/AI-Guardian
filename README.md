<h1 align="center">AI-Guardian</h1>
<h3 align="center">下一代 AI 驱动的安全运营中枢</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.11+-blue?style=flat-square&logo=python" />
  <img src="https://img.shields.io/badge/FastAPI-0.110+-009688?style=flat-square&logo=fastapi" />
  <img src="https://img.shields.io/badge/React-18-61DAFB?style=flat-square&logo=react" />
  <img src="https://img.shields.io/badge/License-Apache%202.0-green?style=flat-square" />
  <img src="https://img.shields.io/badge/Docker-Ready-2496ED?style=flat-square&logo=docker" />
</p>

<p align="center">
  <b>AI-Guardian</b> 是一款以 <b>AI Agent</b> 为核心引擎的安全运营协作平台，围绕安全事件全生命周期，提供智能解析、资产关联、自动化研判、协同处置、经验沉淀与报告生成能力。<br/>
  通过证据驱动的分析闭环与多角色协作机制，帮助安全团队从「被动响应」进化为「主动防御」。
</p>

---

## 为什么选择 AI-Guardian

传统安全运营面临的核心矛盾：**告警爆炸 vs 人力有限**。AI-Guardian 的设计哲学是让 AI 承担 80% 的重复研判工作，让人聚焦于真正的决策。

| 痛点 | AI-Guardian 的解法 |
|:---|:---|
| 告警海量，人工逐条研判效率极低 | 内置 AI Agent 自动研判，基于证据链生成结构化分析结论 |
| 多厂商设备日志格式各异，无法统一处理 | 可视化正则规则引擎 + 设备级解析模板，适配任意日志格式 |
| 研判经验存在人脑中，人员流动即丢失 | STE 经验沉淀机制，闭环告警自动转化为可检索的研判知识库 |
| 处置流程靠口头协调，状态不透明 | 四阶段工作流（监测→研判→处置→闭环），支持认领、指派、强制解锁 |
| 报告编写全靠手工拼凑 | 模板化报告引擎，支持从告警数据一键生成 Markdown / Excel / CSV |
| 黑白名单管理死板，查询效率低 | CIDR 范围匹配 + IP 段批量导入导出，毫秒级判定 |

## 核心能力

### 🧠 AI Agent 引擎
- **自主任务建模**：Agent 根据告警上下文自动规划研判步骤
- **证据检索与关联**：自动拉取资产信息、威胁情报、历史相似告警
- **结构化反思**：对研判结果进行自我校验，发现矛盾时主动补查
- **经验提取与复用**：闭环告警可沉淀为 STE 经验，后续研判自动检索复用
- **多模型兼容**：支持 OpenAI / DeepSeek / 通义千问 / 智谱 AI / 硅基流动 / Ollama 等

### 📊 告警全生命周期管理
- **统一解析**：原始日志 → 正则提取 → 结构化字段 → 资产关联 → 威胁情报增强
- **去重与追踪**：每条告警生成唯一 `alert_hash`，支持跨时间窗口的精确去重
- **四阶段流转**：监测组同步 → 研判组认领分析 → 处置组执行封禁 → 闭环归档
- **实时协作**：消息中心推送、认领/释放机制、强制解锁、状态流转通知

### 🔗 资产与情报联动
- **资产中心**：个体资产 + 网段资产，支持 Excel 批量导入，自动关联告警源目 IP
- **威胁情报**：集成主流情报源，自动查询 IP / 域名信誉
- **IP 名单**：黑白名单 + CIDR 匹配，支持快速封禁与误封排查

### 📝 报告与输出
- **模板引擎**：消息模板、Excel 模板、CSV 模板，拖拽式字段拼接
- **报告中心**：支持从告警数据、运营总览一键生成结构化报告
- **Webhook 集成**：告警到达 / 状态变更时自动推送至企业微信、钉钉等

## 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    AI-Guardian Frontend                     │
│           React + TypeScript + Ant Design + Vite         │
└────────────────────────┬────────────────────────────────┘
                         │ REST API
┌────────────────────────┴────────────────────────────────┐
│                    AI-Guardian Backend                      │
│              FastAPI + SQLAlchemy + Pydantic              │
│                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │
│  │ 告警引擎  │ │ AI Agent │ │ 资产服务  │ │ 报告引擎   │  │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │
│  │ 规则解析  │ │ 情报服务  │ │ 工作流   │ │ 模板服务   │  │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘  │
└────────────────────────┬────────────────────────────────┘
              ┌──────────┴──────────┐
         ┌────┴────┐          ┌─────┴─────┐
         │ PostgreSQL│          │   Redis   │
         └─────────┘          └───────────┘
```

## 快速部署

### Docker 一键启动

```bash
git clone https://github.com/HankLEE-1/AI-Guardian.git
cd AI-Guardian
cp .env.example .env
docker compose up -d --build
```

启动后访问：

| 服务 | 地址 |
|:---|:---|
| Web 控制台 | `http://localhost:8080` |
| API 文档 | `http://localhost:8000/docs` |
| 健康检查 | `http://localhost:8000/healthz` |

默认管理员：`admin / admin123`

> ⚠️ 生产环境请务必修改 `.env` 中的 `JWT_SECRET`、`INITIAL_ADMIN_PASSWORD`、数据库连接串及 AI/情报 API 密钥。

### 本地开发

```bash
# 后端
cd backend
pip install -r requirements.txt
PYTHONPATH=backend:. uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload

# 前端
cd frontend
npm install
npm run dev
```

## 功能模块

| 模块 | 能力概述 |
|:---|:---|
| **运营总览** | 告警趋势、状态分布、平均处置耗时、Top 攻击源、实时统计看板 |
| **内容解析** | 原始日志粘贴 → 自动正则提取 → 资产命中检测 → 一键保存为告警工单 |
| **告警工作台** | Hash 搜索、认领/释放、状态流转、AI 研判、威胁情报查询、CSV 导出 |
| **AI 中心** | 提示词管理、多轮对话研判、执行链路可视化、STE 经验库、自动经验提取 |
| **资产中心** | 个体/网段资产维护、Excel 导入导出、资产指纹、负责人与区域管理 |
| **消息中心** | 工作流消息推送、未读提醒、按告警 Hash 快捷跳转 |
| **报告中心** | 新建/编辑/复制/导出报告，支持从模板和运营数据一键生成 |
| **规则中心** | 元规则 + 自定义正则规则，支持规则生成器和设备级适配 |
| **模板中心** | 消息模板、Excel 模板、CSV 模板，拖拽式字段拼接 |
| **IP 名单** | 白名单/黑名单、CIDR 范围匹配、批量导入导出 |
| **系统管理** | 用户/角色/项目/设备管理、审计日志、后台任务监控 |

## 角色权限

| 角色 | 职责 | 关键权限 |
|:---|:---|:---|
| `admin` | 系统管理员 | 全部权限，含用户管理、强制解锁、删除、全员配置 |
| `monitor` | 监测组 | 同步告警、内容解析、导入历史数据 |
| `analyst` | 研判组 | 认领告警、AI 研判、情报查询、转处置/闭环 |
| `disposer` | 处置组 | 认领处置、退回研判、封禁执行、闭环确认 |
| `viewer` | 只读人员 | 查看所有数据，无写入权限 |

## 技术栈

| 层级 | 技术 |
|:---|:---|
| 后端框架 | FastAPI + SQLAlchemy + Pydantic + Alembic |
| 前端框架 | React 18 + TypeScript + Ant Design + Vite |
| 数据库 | PostgreSQL 16（Docker 默认）/ SQLite（本地开发） |
| 缓存 | Redis 7 |
| AI 引擎 | LangGraph + OpenAI-Compatible API |
| Excel 处理 | openpyxl |
| 部署 | Docker Compose + Nginx 反向代理 |

## 内置演示数据

首次启动自动初始化：

- **演示用户**：`demo_analyst / demo123456`、`demo_viewer / demo123456`
- **演示项目**：攻防演练、日常运营
- **演示设备**：WAF、NDR、态势感知
- **演示资产**：门户系统、交易 API、数据库、办公终端、业务服务器、网段资产
- **演示规则**：通用五元组规则 + 设备级解析规则
- **演示模板**：研判通报、Excel 行、CSV 导出

> 演示数据中的公网 IP 使用 RFC 5737 文档保留地址段，不包含真实 IP。

## 更新日志

### v3.0 — AI-Guardian 重构版
- 全面重构为 AI-Guardian 架构，优化 AI Agent 执行链路
- 新增证据覆盖检查与结构化反思机制
- 系统设置升级：AI 连通性测试、模型列表获取、个人/全员配置隔离
- 规则引擎增强：支持 `match_all` 多命中提取
- 安全加固：修复 viewer 角色可绕过前端调用写接口的漏洞
- 报告中心：支持从内容解析和运营总览一键生成报告

### v2.1
- 新增报告中心模块
- 权限矩阵完善与安全修复

### v2.0.1
- Agent 链路重构，新增任务建模与定向补查
- 规则与部署修复

## License

[Apache License 2.0](./LICENSE)

---

<p align="center">
  <b>AI-Guardian</b> — 让安全运营从人力密集走向智能驱动<br/>
  <sub>Built with ❤️ by <a href="https://github.com/HankLEE-1">HankLee</a></sub>
</p>
