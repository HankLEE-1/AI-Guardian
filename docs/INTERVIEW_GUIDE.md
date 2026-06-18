# AI-Guardian 项目面试指南

> **AI-Guardian** — 基于 Agent 驱动的网络安全流量监控与分析平台
>
> 本文档包含：简历编写建议、项目深度解析、常见面试题与参考答案、前置学习路线。

---

## 📋 目录

- [一、项目概述](#一项目概述)
- [二、系统架构](#二系统架构)
- [三、简历编写指南](#三简历编写指南)
- [四、核心技术深度解析](#四核心技术深度解析)
- [五、项目面试题（附参考答案）](#五项目面试题附参考答案)
- [六、前置学习内容](#六前置学习内容)

---

## 一、项目概述

### 1.1 项目背景

在大型网络安全攻防演练（护网行动）中，安全运营团队面临以下痛点：

| 痛点 | 传统方式 | AI-Guardian 方案 |
|------|----------|-----------------|
| 告警量巨大（日均数千条） | 人工逐一分析 | AI Agent 自动研判 |
| 告警格式不统一 | 正则手动匹配 | 语义化日志解析引擎 |
| 缺乏历史经验沉淀 | 依赖老员工记忆 | STE 知识自动提取与注入 |
| 多人协作混乱 | 微信/电话沟通 | 工作流状态机 + 消息中心 |
| 威胁情报分散 | 多平台手动查询 | 四大威胁情报平台聚合 |
| 报告编写耗时 | Word 手动整理 | 模板引擎 + AI 自动生成 |

### 1.2 核心功能

```
┌─────────────────────────────────────────────────────────────┐
│                    AI-Guardian 功能全景                       │
├──────────┬──────────┬──────────┬──────────┬─────────────────┤
│ 告警工作台 │  AI 中心  │  资产中心  │  报告中心  │   系统管理      │
├──────────┼──────────┼──────────┼──────────┼─────────────────┤
│ 告警生命周期│ Prompt管理│ 资产管理  │ 报告生成  │  用户/角色管理   │
│ 认领/分配  │ 多轮对话  │ CIDR段   │ 模板引擎  │  项目/设备管理   │
│ AI研判触发 │ STE经验库 │ Excel导入│ Markdown  │  操作审计日志    │
│ 威胁情报   │ Agent工具 │ 批量操作  │ 分类标签  │  权限控制(RBAC)  │
│ 资产关联   │ 正则生成  │ IP名单   │ 导出复用  │  多租户隔离      │
└──────────┴──────────┴──────────┴──────────┴─────────────────┘
```

### 1.3 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| **前端** | React 18 + TypeScript 5.7 | 类型安全的组件开发 |
| **UI 框架** | Ant Design 5 | 企业级 UI 组件库 |
| **构建工具** | Vite 6 | 极速 HMR 开发体验 |
| **数据获取** | TanStack React Query v5 | 声明式数据获取 + 缓存 |
| **图表** | Recharts | React 图表库 |
| **后端框架** | FastAPI | 高性能异步 Python Web 框架 |
| **ORM** | SQLAlchemy | Python ORM 事实标准 |
| **数据库** | PostgreSQL 16 | 生产级关系型数据库 |
| **缓存** | Redis 7 | 高速缓存 + 消息队列 |
| **AI 框架** | LangChain Core + 自研 Agent | LLM 工具调用 + 流式输出 |
| **部署** | Docker Compose | 一键容器化部署 |
| **反向代理** | Nginx | 静态文件 + API 代理 |

---

## 二、系统架构

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                         用户浏览器                                 │
│              React 18 + TypeScript + Ant Design                   │
│    ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐   │
│    │Dashboard│Alerts│ AI   │Assets│Reports│ Settings  │   │
│    └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └────┬─────┘   │
│       └────────┴────────┴────────┴────────┴──────────┘          │
│                          ↓ HTTP/REST + SSE                       │
└──────────────────────────────────┬───────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │          Nginx              │
                    │   (静态文件 + API 反向代理)     │
                    └──────────────┬──────────────┘
                                   │
┌──────────────────────────────────┴───────────────────────────────┐
│                    FastAPI 后端 (Python)                          │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ API 路由  │→│ 服务层    │→│ 数据模型  │  │  AI Agent 引擎   │  │
│  │ /alerts  │  │ workflow │  │ SQLAlchemy│  │  LangChain Core  │  │
│  │ /ai      │  │ alert    │  │ entities  │  │  Tool Registry   │  │
│  │ /assets  │  │ asset    │  │           │  │  Prompt Manager  │  │
│  │ /reports │  │ parser   │  │           │  │  SSE Streaming   │  │
│  └─────────┘  └──────────┘  └──────────┘  └──────────────────┘  │
│                              │          │                        │
│  ┌───────────────────────────┴──────────┴───────────────────┐   │
│  │              外部集成层                                     │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │   │
│  │  │ 微步在线  │ │ 绿盟NTI │ │ 奇安信   │ │ 安恒    │        │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘        │   │
│  │  ┌─────────────────────────────────────────────────┐     │   │
│  │  │ LLM Provider (OpenAI/DeepSeek/Qwen/Zhipu/...)  │     │   │
│  │  └─────────────────────────────────────────────────┘     │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────┬──────────────────┬───────────────────────────┘
                   │                  │
          ┌────────┴────────┐  ┌──────┴──────┐
          │  PostgreSQL 16  │  │   Redis 7   │
          │  (持久化存储)     │  │  (缓存/队列) │
          └─────────────────┘  └─────────────┘
```

### 2.2 后端分层架构

```
backend/app/
├── api/                    # 🌐 API 路由层 (FastAPI Router)
│   ├── alerts.py           #   告警 CRUD + 工作流操作
│   ├── ai.py               #   AI 研判 + SSE 流式对话
│   ├── assets.py           #   资产管理
│   ├── auth.py             #   JWT 认证
│   ├── reports.py          #   报告生成
│   ├── messages.py         #   消息中心
│   ├── settings.py         #   系统配置
│   ├── rules.py            #   解析规则
│   ├── templates.py        #   模板管理
│   ├── imports.py          #   数据导入
│   ├── ops.py              #   运维接口
│   ├── admin.py            #   管理后台
│   └── deps.py             #   依赖注入 (当前用户、权限校验)
│
├── services/               # ⚙️ 业务逻辑层
│   ├── workflow_service.py #   告警状态机 (分析→处置→闭环)
│   ├── alert_service.py    #   告警创建、去重、字段规范化
│   ├── ai_service.py       #   AI 研判入口 + 经验注入
│   ├── ai_agent.py         #   Agent 引擎 (工具调用、证据收集)
│   ├── ai_gateway.py       #   LLM 网关抽象 (多供应商)
│   ├── parser_service.py   #   日志解析引擎
│   ├── asset_service.py    #   资产匹配与关联
│   ├── report_service.py   #   报告生成
│   ├── template_service.py #   模板渲染 ({{变量}}替换)
│   ├── message_service.py  #   消息通知
│   ├── task_service.py     #   后台任务管理
│   ├── ip_list_service.py  #   IP 黑白名单
│   ├── stats_service.py    #   统计数据
│   └── audit_service.py    #   操作审计
│
├── services/ai_tools/      # 🔧 AI 工具注册表
│   └── registry.py         #   20+ 工具: alert.search, ti.lookup_ip, ...
│
├── models/                 # 📦 数据模型层
│   ├── entities.py         #   SQLAlchemy 实体定义
│   ├── database.py         #   数据库连接
│   └── bootstrap.py        #   初始化数据 + 系统设置
│
├── schemas/                # 📋 数据校验层 (Pydantic)
│   └── common.py           #   请求/响应 Schema
│
├── core/                   # 🏗️ 基础设施层
│   ├── security.py         #   JWT 生成/验证、密码哈希
│   ├── settings.py         #   环境变量配置
│   └── utils.py            #   工具函数
│
└── workers/                # 🔄 后台任务
    └── worker.py           #   轮询式任务执行器
```

### 2.3 告警生命周期状态机

```
                    ┌──────────────────────────────────────┐
                    │           告警生命周期                  │
                    └──────────────────────────────────────┘

    ┌─────────┐    认领/分配    ┌──────────┐    流转     ┌──────────┐
    │  监测中   │──────────────→│   研判中   │──────────→│   处置中   │
    │ (新建)    │               │ (分析师)   │           │ (处置员)   │
    └─────────┘               └─────┬─────┘           └─────┬─────┘
                                    │                       │
                                    │ 流转                   │ 流转
                                    ▼                       ▼
                              ┌──────────┐           ┌──────────┐
                              │  误报     │           │  已处置   │
                              │ (关闭)    │           │ (闭环)    │
                              └──────────┘           └──────────┘
                                    │                       │
                                    │                       │
                                    ▼                       ▼
                              ┌──────────┐           ┌──────────────┐
                              │  忽略     │           │ 自动生成      │
                              │ (关闭)    │           │ STE 经验记录  │
                              └──────────┘           └──────────────┘
```

**关键操作：**
- **认领（Claim）**：分析师/处置员认领告警，锁定处理权
- **释放（Release）**：放弃认领，告警回到队列
- **强制释放**：管理员可强制解锁被占用的告警
- **流转（Transition）**：状态变更 + 触发副作用（IP封禁、白名单、经验生成）

### 2.4 AI Agent 架构

```
┌──────────────────────────────────────────────────────────────┐
│                    AI Agent 引擎架构                          │
│                                                              │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────┐  │
│  │ 用户输入     │───→│ 任务分类器    │───→│ 证据需求分析    │  │
│  │ (告警/问题)  │    │ (14种任务原型) │    │ (21种证据角色)  │  │
│  └─────────────┘    └──────────────┘    └───────┬────────┘  │
│                                                  │           │
│                          ┌───────────────────────┼──────┐    │
│                          │        工具调用链       │      │    │
│                          │  ┌─────┐ ┌─────┐ ┌───┴──┐  │    │
│                          │  │搜索  │ │查询  │ │资产   │  │    │
│                          │  │告警  │ │TI   │ │关联   │  │    │
│                          │  └──┬──┘ └──┬──┘ └───┬──┘  │    │
│                          │     └───────┴────────┘     │    │
│                          │            ↓               │    │
│                          │    ┌──────────────┐        │    │
│                          │    │  证据包组装    │        │    │
│                          │    │ (结构化数据)   │        │    │
│                          │    └──────┬───────┘        │    │
│                          └───────────┼────────────────┘    │
│                                      ↓                      │
│                           ┌─────────────────┐               │
│                           │   LLM 分析       │               │
│                           │ (Prompt + 证据)   │               │
│                           └────────┬────────┘               │
│                                    ↓                        │
│                           ┌─────────────────┐               │
│                           │  结构化输出       │               │
│                           │ meta/ste/action  │               │
│                           │ quality/confidence│              │
│                           └─────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

**核心设计思想：先收集证据，再让 LLM 分析 — 防止幻觉**

### 2.5 STE 知识闭环

```
┌──────────────────────────────────────────────────────────┐
│                STE (Strategy-Tactics-Evidence) 知识闭环    │
│                                                          │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐        │
│   │  告警     │────→│  处置     │────→│  闭环     │        │
│   │  (新建)   │     │  (分析)   │     │  (关闭)   │        │
│   └──────────┘     └──────────┘     └────┬─────┘        │
│                                          │               │
│                                          ▼               │
│                                  ┌──────────────┐        │
│                                  │ STE 自动提取   │        │
│                                  │ Strategy 策略  │        │
│                                  │ Tactics 战术   │        │
│                                  │ Evidence 证据  │        │
│                                  └──────┬───────┘        │
│                                         │                │
│                                         ▼                │
│                                  ┌──────────────┐        │
│                                  │ 经验库 (Draft) │        │
│                                  │  审核后发布     │        │
│                                  └──────┬───────┘        │
│                                         │                │
│          ┌──────────────────────────────┘                │
│          ▼                                               │
│   ┌──────────────┐     ┌──────────────┐                  │
│   │ 下次同类告警   │────→│ 经验注入       │                  │
│   │ (自动匹配)    │     │ (Prompt增强)   │                  │
│   └──────────────┘     └──────────────┘                  │
│                                                          │
│   效果：系统越用越聪明，每次处置都在积累知识                    │
└──────────────────────────────────────────────────────────┘
```

---

## 三、简历编写指南

### 3.1 项目经历模板

#### 推荐写法（STAR 法则）

> **AI-Guardian — 基于 Agent 驱动的网络安全运营平台** ｜ 全栈开发
>
> *技术栈：React 18 + TypeScript / FastAPI + SQLAlchemy / PostgreSQL + Redis / LangChain + LLM*
>
> - **S（场景）**：针对护网行动中安全运营团队面临的告警量大、研判效率低、经验难沉淀等痛点
> - **T（任务）**：独立设计并开发了 AI 驱动的安全运营协作平台
> - **A（行动）**：
>   - 设计了 **14 种任务原型 × 21 种证据角色** 的 Agent 工具调用链，实现先收集证据再 LLM 分析的架构，有效防止 AI 幻觉
>   - 实现了 **STE（Strategy-Tactics-Evidence）知识闭环**，从已处置告警中自动提取经验，注入后续研判 Prompt，形成"越用越聪明"的正反馈循环
>   - 开发了 **语义化日志解析引擎**，支持正则/固定/内置三种规则模式，结合资产关联和威胁情报实现上下文富化
>   - 构建了 **角色驱动的告警工作流状态机**（监测→研判→处置→闭环），支持认领/分配/强制释放，配合乐观锁解决并发冲突
>   - 集成了 **四大国内威胁情报平台**（微步在线、绿盟NTI、奇安信、安恒），统一接口聚合 IP 信誉查询
>   - 前端采用 React 18 + TypeScript + Ant Design，后端 FastAPI + SQLAlchemy，Docker Compose 一键部署
> - **R（结果）**：告警研判效率提升 3-5 倍，报告生成时间从 2 小时缩短至 10 分钟

### 3.2 不同岗位的侧重点

#### 后端开发工程师

重点突出：
- FastAPI 异步架构设计
- SQLAlchemy ORM 模型设计（多租户、关联查询）
- 状态机设计模式
- 乐观锁并发控制
- 后台任务队列
- JWT 认证 + RBAC 权限

```
核心亮点：
✅ 分层架构设计（API → Service → Model）
✅ 多租户数据隔离（workspace_id）
✅ 5 种角色的 RBAC 权限控制
✅ 乐观锁 + 审计日志
✅ 结构化 Prompt 管理系统
```

#### 前端开发工程师

重点突出：
- React 18 + TypeScript 类型安全
- TanStack React Query 数据层设计
- 复杂表单/表格交互
- SSE 流式渲染
- 组件抽象设计

```
核心亮点：
✅ 12 个功能页面的完整 SPA
✅ React Query 声明式数据获取 + 缓存策略
✅ SSE 流式 AI 对话渲染
✅ Ant Design 深度定制
✅ 响应式布局 + 侧边栏导航
```

#### AI/算法工程师

重点突出：
- Agent 架构设计（工具调用链）
- Prompt Engineering（6 种 Prompt 类型）
- STE 知识提取与注入
- 结构化输出设计
- 多供应商 LLM 网关

```
核心亮点：
✅ 证据驱动的 Agent 架构（防幻觉）
✅ STE 知识闭环（自动学习）
✅ 14×21 任务-证据矩阵
✅ SSE 流式输出 + 工具调用
✅ 多模型供应商抽象层
```

#### 安全工程师

重点突出：
- 威胁情报集成
- 日志解析引擎
- 告警去重与关联
- IP 黑白名单管理
- 安全审计

```
核心亮点：
✅ 四大 TI 平台聚合（微步/绿盟/奇安信/安恒）
✅ 语义化日志解析（正则/固定/内置规则）
✅ SHA-256 告警去重
✅ 资产关联 + 上下文富化
✅ IP 自动封禁 + 白名单联动
```

### 3.3 常见简历措辞优化

| ❌ 普通写法 | ✅ 优化写法 |
|------------|-----------|
| 做了一个安全告警系统 | 设计了基于 Agent 驱动的安全运营平台，实现告警自动研判与经验闭环 |
| 用了 React 和 FastAPI | 前端 React 18 + TypeScript + Ant Design，后端 FastAPI + SQLAlchemy，Docker Compose 容器化部署 |
| 接入了 AI 功能 | 构建了证据驱动的 AI Agent 架构，14 种任务原型 × 21 种证据角色的工具调用链，先收集证据再 LLM 分析，有效防止幻觉 |
| 做了告警管理 | 设计了角色驱动的告警工作流状态机（监测→研判→处置→闭环），支持认领/分配/强制释放，配合乐观锁解决并发冲突 |
| 加了威胁情报 | 集成四大国内 TI 平台（微步在线/绿盟NTI/奇安信/安恒），统一接口聚合 IP 信誉查询与地理位置信息 |

---

## 四、核心技术深度解析

### 4.1 AI Agent 架构详解

#### 为什么选择"证据驱动"而非"直接问答"？

```
❌ 传统方式（直接问答）：
   用户: "这个告警是不是误报？"
   LLM: "根据经验，SSH暴力破解通常不是误报。"  ← 幻觉！没有看实际数据

✅ AI-Guardian（证据驱动）：
   1. 系统先调用工具收集证据：
      - alert.search("51.222.47.156") → 找到历史告警
      - ti.lookup_ip("51.222.47.156") → 查询威胁情报
      - asset.get_by_ip("51.222.47.156") → 查资产信息
   2. 组装证据包
   3. LLM 基于证据分析：
      "该 IP 在威胁情报中标记为恶意（微步评分 85/100），
       近 30 天有 12 次 SSH 暴力破解记录，
       目标资产为生产服务器（重要性：高），
       建议封禁并通知运维团队。"  ← 有理有据
```

#### 工具注册表（20+ 工具）

```python
# 核心工具分类
tools = {
    # 告警相关
    "alert.search":       "按条件搜索历史告警",
    "alert.get_detail":   "获取告警详情",
    
    # 威胁情报
    "ti.lookup_ip":       "查询 IP 威胁情报",
    "intel.ip_report":    "获取 IP 综合报告",
    
    # 资产管理
    "asset.get_by_ip":    "根据 IP 查资产",
    "asset.search":       "搜索资产",
    
    # 经验知识
    "experience.search":  "搜索历史经验",
    
    # 统计分析
    "stats.alert_trend":  "告警趋势统计",
    "stats.top_attackers":"Top 攻击源",
    # ... 更多
}
```

### 4.2 工作流状态机详解

#### 乐观锁防止并发冲突

```python
# 核心逻辑
def transition_alert(db, user, alert, target_status, updated_at=None):
    # 乐观锁检查：防止两人同时操作同一告警
    if updated_at and alert.updated_at:
        if alert.updated_at.replace(microsecond=0) > updated_at.replace(microsecond=0):
            raise HTTPException(409, "告警已被他人修改，请刷新页面后再试")
    
    # 状态流转校验
    if (alert.status, target_status) not in VALID_TRANSITIONS:
        raise HTTPException(400, f"不允许从 {alert.status} 流转到 {target_status}")
    
    # 执行流转 + 副作用
    alert.status = target_status
    # ... IP 封禁、白名单、经验生成等副作用
```

#### 状态流转规则

```
analysis  → disposal       (研判完成，需要处置)
analysis  → false_positive (判定为误报)
analysis  → ignored        (忽略)
disposal  → disposed       (处置完成)
```

### 4.3 日志解析引擎

```
原始日志 → 正则匹配 → 字段提取 → 资产关联 → 威胁情报注入 → 模板渲染
   │          │          │          │            │             │
   │     ParseRule    parsed_    src_asset    ti_result    formatted_
   │     (优先级)      fields     dst_asset                 chat/excel
   │
   └─ "Jun 17 09:01 sshd[12345]: Failed password for root from 51.222.47.156"
```

**三种规则模式：**
| 模式 | 说明 | 示例 |
|------|------|------|
| `regex` | 正则表达式 | `Failed password for (?P<user>\S+) from (?P<src_ip>\S+)` |
| `fixed` | 固定字段 | 固定分隔符切分 |
| `builtin` | 系统内置 | 41+ 预定义安全字段 |

### 4.4 多租户设计

```python
# 所有查询都带 workspace_id 过滤
class Alert(Base):
    workspace_id: int  # 租户隔离字段
    # ...

# 查询示例
alerts = db.query(Alert).filter(
    Alert.workspace_id == user.workspace_id,  # 租户隔离
    Alert.status == status
).all()
```

---

## 五、项目面试题（附参考答案）

### 5.1 架构设计类

#### Q1: 为什么选择前后端分离架构？

**参考答案：**

> 前后端分离有三个核心优势：
>
> 1. **开发效率**：前端用 React + TypeScript 独立开发，后端用 FastAPI 独立开发，可以并行工作
> 2. **部署灵活性**：前端打包成静态文件由 Nginx 服务，后端独立容器化，可以独立扩缩容
> 3. **技术栈解耦**：前端可以选择最适合 UI 开发的 React 生态，后端选择最适合 AI 集成的 Python 生态
>
> 在本项目中，前端需要复杂的告警工作台交互（拖拽、实时状态更新），后端需要 LangChain、SQLAlchemy 等 Python 生态，分离架构是最佳选择。

#### Q2: 为什么用 FastAPI 而不是 Django/Flask？

**参考答案：**

> 选择 FastAPI 基于三个考量：
>
> 1. **异步支持**：FastAPI 原生支持 async/await，在调用外部 API（威胁情报、LLM）时可以非阻塞等待，提升并发性能
> 2. **自动文档**：FastAPI 基于 Pydantic 自动生成 OpenAPI 文档，前端开发时可以直接参考 Swagger UI
> 3. **类型安全**：Pydantic Schema 与 Python 类型注解结合，在编译期就能发现类型错误
>
> Django 太重（ORM、Admin、模板引擎我们都不需要），Flask 太轻（没有自动文档、没有类型校验）。FastAPI 恰好在中间。

#### Q3: 数据库为什么选择 PostgreSQL 而不是 MySQL？

**参考答案：**

> 1. **JSON 支持**：告警的 `parsed_fields`、`ti_result`、`ai_result` 都是 JSON 字段，PostgreSQL 的 JSONB 类型支持索引和查询
> 2. **全文搜索**：PostgreSQL 内置全文搜索，可以用于告警文本检索
> 3. **并发性能**：PostgreSQL 的 MVCC 机制在高并发写入场景下性能更好
> 4. **扩展性**：如果未来需要地理空间查询（IP 地理位置），PostgreSQL 有 PostGIS 扩展

### 5.2 AI/Agent 类

#### Q4: 你提到"防止 AI 幻觉"，具体是怎么做的？

**参考答案：**

> 核心思想是 **"先收集证据，再让 LLM 分析"**，而不是直接把原始日志丢给 LLM。
>
> 具体实现：
>
> 1. **任务分类**：将用户输入分类为 14 种任务原型（如 `entity_lookup`、`incident_investigation`）
> 2. **证据需求分析**：每种任务原型对应 21 种证据角色（如 `subject_identity`、`risk_signals`）
> 3. **工具调用**：根据证据角色自动选择工具（如 `ti.lookup_ip`、`asset.get_by_ip`）
> 4. **证据包组装**：将工具返回的结构化数据组装成证据包
> 5. **Prompt 注入**：将证据包注入到 Prompt 中，让 LLM 基于真实数据分析
>
> ```python
> # 伪代码
> def analyze_alert(alert):
>     # 1. 分类任务
>     task_type = classify_task(alert)  # → "incident_investigation"
>     
>     # 2. 确定需要哪些证据
>     evidence_roles = TASK_EVIDENCE_MAP[task_type]
>     # → ["subject_identity", "risk_signals", "historical_cases"]
>     
>     # 3. 调用工具收集证据
>     evidence = {}
>     for role in evidence_roles:
>         tool = ROLE_TOOL_MAP[role]
>         evidence[role] = tool.execute(alert)
>     
>     # 4. 组装 Prompt
>     prompt = build_prompt(alert, evidence)
>     
>     # 5. LLM 分析（有证据支撑）
>     return llm.analyze(prompt)
> ```
>
> 这样 LLM 的分析结果都是基于真实数据的，大幅降低了幻觉风险。

#### Q5: STE 知识闭环是怎么实现的？

**参考答案：**

> STE（Strategy-Tactics-Evidence）知识闭环分三个阶段：
>
> **阶段一：经验提取**
> 当告警流转到终态（已处置/误报/忽略）时，自动触发 STE 提取：
> ```python
> # 从已处置告警中提取经验
> ste = llm.extract_ste(closed_alert)
> # 输出：
> # {
> #   "strategy": "SSH暴力破解攻击",
> #   "tactics": "字典攻击、凭据填充",
> #   "evidence": "高频登录失败、非常用IP、非工作时间"
> # }
> ```
>
> **阶段二：经验审核**
> 提取的经验初始状态为 `draft`，管理员审核后变为 `published`
>
> **阶段三：经验注入**
> 当同类告警再次出现时，系统自动搜索匹配的历史经验，注入到 Prompt 中：
> ```python
> # 研判新告警时
> similar_experiences = experience.search(alert.event_type, alert.source_ip)
> prompt = build_prompt(alert, evidence, experiences=similar_experiences)
> # LLM 分析时可以参考历史经验
> ```
>
> **效果**：系统越用越聪明，每次处置都在积累知识。

#### Q6: 为什么用 SSE 流式输出而不是 WebSocket？

**参考答案：**

> 1. **单向通信**：AI 对话本质上是服务端向客户端的单向推送，SSE 天然适合
> 2. **HTTP 兼容**：SSE 基于 HTTP，不需要额外的协议升级，Nginx 代理配置简单
> 3. **自动重连**：浏览器原生支持 SSE 的自动重连机制
> 4. **实现简单**：FastAPI 的 `sse-starlette` 库一行代码就能实现流式输出
>
> WebSocket 适合双向实时通信（如聊天室、协同编辑），但 AI 对话场景下 SSE 更轻量。

### 5.3 工作流/状态机类

#### Q7: 告警工作流的状态机是怎么设计的？

**参考答案：**

> ```
> 状态定义：
> - analysis    (研判中)  → 分析师角色
> - disposal    (处置中)  → 处置员角色
> - disposed    (已处置)  → 终态
> - false_positive (误报) → 终态
> - ignored     (忽略)    → 终态
> 
> 流转规则：
> analysis → disposal      (研判完成，需要处置)
> analysis → false_positive (判定为误报)
> analysis → ignored       (忽略)
> disposal → disposed      (处置完成)
> ```
>
> 关键设计点：
> 1. **角色隔离**：分析师只能操作 `analysis` 状态的告警，处置员只能操作 `disposal` 状态
> 2. **认领机制**：防止多人同时操作同一告警
> 3. **乐观锁**：通过 `updated_at` 时间戳检测并发冲突
> 4. **副作用触发**：状态流转时自动触发 IP 封禁、白名单、经验生成等操作

#### Q8: 乐观锁是怎么实现的？为什么不用悲观锁？

**参考答案：**

> **乐观锁实现：**
> ```python
> # 前端提交时带上上次读取的 updated_at
> PATCH /alerts/123
> {
>     "parsed_fields": {...},
>     "updated_at": "2024-06-17T10:00:00"  # 上次读取的时间
> }
> 
> # 后端校验
> if alert.updated_at > payload.updated_at:
>     raise 409 Conflict "告警已被他人修改"
> ```
>
> **为什么不用悲观锁：**
> 1. **并发冲突概率低**：安全告警通常不会被多人同时编辑
> 2. **用户体验**：悲观锁会锁定记录，其他用户无法查看
> 3. **实现复杂度**：悲观锁需要锁超时、死锁检测等机制
> 4. **Web 场景**：HTTP 是无状态的，维护长连接锁成本高

### 5.4 安全/性能类

#### Q9: 多租户数据隔离是怎么做的？

**参考答案：**

> 采用 **共享数据库 + workspace_id 字段** 的方案：
>
> ```python
> # 所有实体都有 workspace_id
> class Alert(Base):
>     workspace_id: int
>     # ...
> 
> # 所有查询都带 workspace_id 过滤
> def list_alerts(db, user):
>     return db.query(Alert).filter(
>         Alert.workspace_id == user.workspace_id
>     ).all()
> ```
>
> **为什么不用独立数据库/Schema：**
> 1. **成本**：每个租户一个数据库会大幅增加运维成本
> 2. **复杂度**：跨租户查询（如系统管理员）需要动态切换连接
> 3. **规模**：本项目是企业内部工具，租户数量有限，共享数据库足够
>
> **安全保证：**
> - API 层统一通过 `current_user` 依赖注入获取用户信息
> - 所有查询都自动带上 `workspace_id` 过滤
> - 前端无法指定 `workspace_id`，由后端从 JWT 中解析

#### Q10: 告警去重是怎么做的？

**参考答案：**

> 使用 **SHA-256 哈希** 实现告警去重：
>
> ```python
> def normalize_alert_fields(alert):
>     # 提取关键字段
>     key_fields = {
>         "source_ip": alert.source_ip,
>         "destination_ip": alert.destination_ip,
>         "event_type": alert.event_type,
>         # 排除时间戳、ID 等变化字段
>     }
>     # 生成哈希
>     alert.alert_hash = hashlib.sha256(
>         json.dumps(key_fields, sort_keys=True).encode()
>     ).hexdigest()
> 
> def find_duplicate_alert(db, user, parsed_fields, device_id):
>     # 先算哈希
>     hash_value = compute_hash(parsed_fields)
>     # 查重
>     return db.query(Alert).filter(
>         Alert.alert_hash == hash_value,
>         Alert.workspace_id == user.workspace_id
>     ).first()
> ```
>
> **设计要点：**
> 1. 只对业务关键字段计算哈希，排除时间戳等变化字段
> 2. 同一 workspace 内去重，不同 workspace 独立
> 3. 去重时返回已有告警的信息，前端提示用户

### 5.5 前端/工程化类

#### Q11: 为什么用 TanStack React Query 而不是 Redux？

**参考答案：**

> 1. **服务端状态 vs 客户端状态**：本项目 90% 的状态都是服务端数据（告警列表、资产列表），React Query 专门管理服务端状态
> 2. **缓存管理**：React Query 自动管理缓存、过期、后台刷新，不需要手动写 reducer
> 3. **乐观更新**：mutation 后自动 invalidate 相关 query，触发重新获取
> 4. **代码量**：相比 Redux 的 action/reducer/dispatch，React Query 的代码量减少 60%+
>
> ```tsx
> // React Query 方式（简洁）
> const { data: alerts } = useQuery({
>     queryKey: ['alerts', status],
>     queryFn: () => api.get(`/alerts?status=${status}`)
> });
> 
> // Redux 方式（繁琐）
> dispatch(fetchAlertsRequest());
> // reducer: FETCH_ALERTS_REQUEST → FETCH_ALERTS_SUCCESS
> // 中间件: fetchAlertsMiddleware
> // selector: selectAlerts
> ```

#### Q12: 前端路由为什么没用 React Router？

**参考答案：**

> 采用 **URL 参数路由** 而非 React Router：
>
> ```tsx
> // 通过 URL 参数切换页面
> // http://localhost:8180/?page=dashboard
> // http://localhost:8180/?page=alerts&alert_hash=abc123
> 
> const page = new URLSearchParams(window.location.search).get('page');
> 
> switch(page) {
>     case 'dashboard': return <DashboardPage />;
>     case 'alerts': return <AlertWorkbench />;
> // ...
> }
> ```
>
> **原因：**
> 1. **简单性**：项目只有 12 个页面，不需要嵌套路由、动态路由等复杂功能
> 2. **部署兼容**：Nginx 只需要配置一个 `try_files` 就能处理所有路由
> 3. **状态保持**：URL 参数天然支持页面状态的序列化（如告警 hash）

### 5.6 综合/开放类

#### Q13: 如果要支持 10 万级告警量，你会怎么优化？

**参考答案：**

> **数据库层：**
> 1. 告警表按时间分区（PostgreSQL 原生分区表）
> 2. `alert_hash`、`source_ip`、`status` 建立复合索引
> 3. 历史告警归档到冷存储
>
> **缓存层：**
> 1. Redis 缓存热点告警（最近 24 小时）
> 2. 资产信息缓存（变化频率低）
> 3. 威胁情报缓存（TTL 1 小时）
>
> **架构层：**
> 1. 引入消息队列（RabbitMQ/Kafka）解耦告警创建和处理
> 2. AI 研判异步化，后台 Worker 池处理
> 3. 前端分页 + 虚拟滚动（只渲染可见区域）
>
> **AI 层：**
> 1. 批量研判：一次请求处理多条同类告警
> 2. 模型缓存：相同类型的告警复用分析结果
> 3. 轻量模型：简单任务用小模型，复杂任务用大模型

#### Q14: 这个项目你遇到的最大挑战是什么？

**参考答案（示例）：**

> **挑战：AI Agent 的工具调用可靠性**
>
> 问题：LLM 在调用工具时经常出现参数格式错误、调用不存在的工具、无限循环调用等问题
>
> 解决方案：
> 1. **严格的工具描述**：为每个工具提供详细的参数说明和示例
> 2. **参数校验**：工具调用前先校验参数格式
> 3. **调用次数限制**：设置最大调用次数，防止无限循环
> 4. **错误恢复**：工具调用失败时返回错误信息，让 LLM 自行修正
> 5. **兜底策略**：如果 Agent 无法完成任务，降级为普通 LLM 对话
>
> 结果：工具调用成功率从 70% 提升到 95%+

#### Q15: 如果让你重新设计，你会做什么不同的决定？

**参考答案（示例）：**

> 1. **事件溯源**：告警的状态变更应该用事件溯源模式，记录完整的状态变更历史，而不是只保留当前状态
> 2. **GraphQL**：前端有大量嵌套查询（告警+资产+TI），GraphQL 可以减少请求次数
> 3. **微服务拆分**：AI 服务、告警服务、报表服务可以拆分为独立微服务，独立扩缩容
> 4. **WebSocket**：多人协作场景下，WebSocket 可以实现实时状态同步（如"XX 正在编辑此告警"）

---

## 六、前置学习内容

### 6.1 Python 后端基础

#### 必须掌握

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| Python 基础语法 | [Python 官方教程](https://docs.python.org/3/tutorial/) | ⭐⭐⭐⭐⭐ |
| FastAPI 入门 | [FastAPI 官方文档](https://fastapi.tiangolo.com/zh/) | ⭐⭐⭐⭐⭐ |
| SQLAlchemy ORM | [SQLAlchemy 教程](https://docs.sqlalchemy.org/en/20/orm/quickstart.html) | ⭐⭐⭐⭐⭐ |
| Pydantic 数据校验 | [Pydantic 文档](https://docs.pydantic.dev/) | ⭐⭐⭐⭐ |
| JWT 认证 | [python-jose 文档](https://python-jose.readthedocs.io/) | ⭐⭐⭐⭐ |
| 异步编程 (async/await) | [Python 异步编程](https://docs.python.org/3/library/asyncio.html) | ⭐⭐⭐⭐ |

#### 学习路线

```
Week 1: Python 基础 + FastAPI 入门
  ├── Day 1-2: Python 语法、类型注解、装饰器
  ├── Day 3-4: FastAPI 路由、请求体、响应模型
  ├── Day 5-6: FastAPI 依赖注入、中间件
  └── Day 7: 练习 - 写一个简单的 CRUD API

Week 2: 数据库 + 认证
  ├── Day 1-3: SQLAlchemy ORM、模型定义、关联查询
  ├── Day 4-5: Pydantic Schema、数据校验
  ├── Day 6: JWT 认证、密码哈希
  └── Day 7: 练习 - 带认证的 Todo API

Week 3: 进阶特性
  ├── Day 1-2: 异步编程、SSE 流式输出
  ├── Day 3-4: 中间件、异常处理、日志
  ├── Day 5-6: Docker 容器化
  └── Day 7: 练习 - 部署到 Docker
```

### 6.2 React 前端基础

#### 必须掌握

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| TypeScript 基础 | [TypeScript 手册](https://www.typescriptlang.org/docs/handbook/) | ⭐⭐⭐⭐⭐ |
| React 18 基础 | [React 官方教程](https://react.dev/learn) | ⭐⭐⭐⭐⭐ |
| React Hooks | [Hooks 文档](https://react.dev/reference/react/hooks) | ⭐⭐⭐⭐⭐ |
| Ant Design | [Ant Design 文档](https://ant.design/docs/react/introduce-cn) | ⭐⭐⭐⭐ |
| React Query | [TanStack Query](https://tanstack.com/query/latest) | ⭐⭐⭐⭐ |
| Axios HTTP 客户端 | [Axios 文档](https://axios-http.com/docs/intro) | ⭐⭐⭐ |

#### 学习路线

```
Week 1: TypeScript + React 基础
  ├── Day 1-2: TypeScript 类型、接口、泛型
  ├── Day 3-4: React 组件、Props、State
  ├── Day 5-6: React Hooks (useState, useEffect, useRef)
  └── Day 7: 练习 - 计数器 + Todo List

Week 2: 进阶 React
  ├── Day 1-2: 自定义 Hooks、Context API
  ├── Day 3-4: React Router 或 URL 参数路由
  ├── Day 5-6: Ant Design 组件库
  └── Day 7: 练习 - 带导航的 Dashboard

Week 3: 数据层 + 工程化
  ├── Day 1-3: React Query 数据获取、缓存、Mutation
  ├── Day 4-5: Axios 拦截器、错误处理
  ├── Day 6: Vite 构建工具
  └── Day 7: 练习 - 完整的 CRUD 页面
```

### 6.3 AI/LLM 开发基础

#### 必须掌握

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| OpenAI API | [OpenAI 文档](https://platform.openai.com/docs/) | ⭐⭐⭐⭐⭐ |
| Prompt Engineering | [Prompt 工程指南](https://www.promptingguide.ai/zh) | ⭐⭐⭐⭐⭐ |
| LangChain 基础 | [LangChain 文档](https://python.langchain.com/docs/) | ⭐⭐⭐⭐ |
| Function Calling | [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling) | ⭐⭐⭐⭐⭐ |
| SSE 流式输出 | [sse-starlette](https://github.com/sysid/sse-starlette) | ⭐⭐⭐ |

#### 学习路线

```
Week 1: LLM API 基础
  ├── Day 1-2: OpenAI API 调用、Chat Completion
  ├── Day 3-4: Prompt Engineering 技巧
  ├── Day 5-6: Function Calling / Tool Use
  └── Day 7: 练习 - 简单的 AI 对话机器人

Week 2: Agent 开发
  ├── Day 1-3: LangChain 基础概念
  ├── Day 4-5: Agent 设计模式 (ReAct, Tool Calling)
  ├── Day 6: SSE 流式输出
  └── Day 7: 练习 - 带工具的 AI Agent

Week 3: 进阶
  ├── Day 1-2: 结构化输出 (JSON Schema)
  ├── Day 3-4: Prompt 模板管理
  ├── Day 5-6: 多模型供应商抽象
  └── Day 7: 练习 - 完整的 AI 应用
```

### 6.4 安全/网络基础

#### 必须掌握

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| HTTP 协议基础 | [MDN HTTP 文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP) | ⭐⭐⭐⭐⭐ |
| RESTful API 设计 | [REST API 设计指南](https://restfulapi.net/) | ⭐⭐⭐⭐⭐ |
| JWT 认证原理 | [JWT.io](https://jwt.io/introduction) | ⭐⭐⭐⭐ |
| OWASP Top 10 | [OWASP Top 10](https://owasp.org/www-project-top-ten/) | ⭐⭐⭐ |
| 网络安全基础 | [网络安全入门](https://www.freebuf.com/) | ⭐⭐⭐ |

### 6.5 DevOps 基础

#### 必须掌握

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| Docker 基础 | [Docker 入门教程](https://docs.docker.com/get-started/) | ⭐⭐⭐⭐⭐ |
| Docker Compose | [Compose 文档](https://docs.docker.com/compose/) | ⭐⭐⭐⭐ |
| Nginx 配置 | [Nginx 入门](https://nginx.org/en/docs/beginners_guide.html) | ⭐⭐⭐ |
| Git 版本控制 | [Git 教程](https://git-scm.com/book/zh/v2) | ⭐⭐⭐⭐⭐ |

### 6.6 推荐学习资源

#### 书籍
- 《Fluent Python》— Python 进阶必读
- 《Designing Data-Intensive Applications》— 系统设计圣经
- 《Clean Architecture》— 架构设计原则

#### 在线课程
- [FastAPI 官方教程](https://fastapi.tiangolo.com/zh/tutorial/) — 免费
- [React 官方教程](https://react.dev/learn) — 免费
- [LangChain 官方教程](https://python.langchain.com/docs/get_started/quickstart) — 免费

#### 实战项目
1. **初级**：Todo List（FastAPI + React）
2. **中级**：Blog 系统（带认证、评论、标签）
3. **高级**：AI-Guardian（完整项目）

---

## 附录：项目文件速查表

```
AI-Guardian/
├── backend/
│   ├── app/
│   │   ├── api/           → API 路由（12 个模块）
│   │   ├── services/      → 业务逻辑（15 个模块）
│   │   ├── models/        → 数据模型（4 个实体文件）
│   │   ├── schemas/       → Pydantic Schema
│   │   ├── core/          → 基础设施（安全、配置、工具）
│   │   └── workers/       → 后台任务
│   ├── alembic/           → 数据库迁移
│   └── requirements.txt   → Python 依赖
├── frontend/
│   ├── src/
│   │   ├── pages/         → 12 个页面组件
│   │   ├── components/    → 公共组件
│   │   └── api/           → API 客户端 + 类型定义
│   └── package.json       → Node.js 依赖
├── core/                  → 解析器核心逻辑
├── docker-compose.yml     → Docker 编排
├── Dockerfile.backend     → 后端镜像
├── Dockerfile.frontend    → 前端镜像
└── .env.example           → 环境变量模板
```

---

> 📝 **最后提醒**：面试时不要背诵答案，要结合自己的理解用自己的话表达。重点展示你对技术选型的思考过程，而不仅仅是结果。
>
> 祝面试顺利！🚀
