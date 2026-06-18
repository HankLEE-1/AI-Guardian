# AI-Guardian 项目面试指南

> **AI-Guardian** — 基于 Agent 驱动的网络安全流量监控与分析平台
>
> 本文档重点围绕 **AI Agent 架构设计**，涵盖简历编写、核心原理、面试题与前置学习路线。

---

## 📋 目录

- [一、项目概述](#一项目概述)
- [二、Agent 架构全景（重点）](#二agent-架构全景重点)
- [三、简历编写指南](#三简历编写指南)
- [四、Agent 核心技术深度解析](#四agent-核心技术深度解析)
- [五、Agent 面试题（附参考答案）](#五agent-面试题附参考答案)
- [六、前置学习内容](#六前置学习内容)

---

## 一、项目概述

### 1.1 一句话定位

> AI-Guardian 是一个 **证据驱动的 AI Agent 安全运营平台**，核心创新在于：不让 LLM 凭空分析，而是先通过工具调用收集真实证据，再基于证据进行研判，从根本上防止 AI 幻觉。

### 1.2 技术栈

| 层级 | 技术 |
|------|------|
| **前端** | React 18 + TypeScript + Ant Design 5 + TanStack React Query |
| **后端** | FastAPI + SQLAlchemy + PostgreSQL + Redis |
| **AI 引擎** | LangChain Core + 自研 Agent + SSE 流式输出 |
| **部署** | Docker Compose + Nginx |

### 1.3 核心功能

```
┌─────────────────────────────────────────────────────────┐
│                   AI-Guardian 功能全景                    │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│ 告警工作台 │  AI 中心  │  资产中心  │  报告中心  │  系统管理    │
├──────────┼──────────┼──────────┼──────────┼─────────────┤
│ 告警生命周期│ Prompt管理│ 资产管理  │ 报告生成  │ 用户/角色   │
│ 认领/分配  │ 多轮对话  │ CIDR段   │ 模板引擎  │ 项目/设备   │
│ AI研判触发 │ STE经验库 │ Excel导入│ Markdown  │ 审计日志    │
│ 威胁情报   │ Agent工具 │ 批量操作  │ 分类标签  │ 多租户隔离  │
└──────────┴──────────┴──────────┴──────────┴─────────────┘
```

---

## 二、Agent 架构全景（重点）

### 2.1 为什么安全领域需要 Agent 而不是简单的 LLM 对话？

**传统 LLM 对话的问题：**

```
❌ 直接把日志丢给 LLM：
   用户: "51.222.47.156 这个 IP 的 SSH 暴力破解是不是误报？"
   LLM: "根据常见经验，SSH 暴力破解通常不是误报，建议封禁。"
   
   问题：
   1. LLM 没有查看这个 IP 的历史告警记录
   2. LLM 没有查询威胁情报（微步/绿盟评分）
   3. LLM 不知道目标资产的重要性
   4. LLM 不了解该 IP 之前的处置方式
   → 这就是"幻觉"：没有数据支撑的空洞结论
```

**AI-Guardian 的 Agent 方案：**

```
✅ 证据驱动的 Agent 分析：
   用户: "51.222.47.156 这个 IP 的 SSH 暴力破解是不是误报？"
   
   Agent 执行流程：
   Step 1: 调用 alert.search("51.222.47.156")
           → 发现该 IP 近 30 天有 12 次 SSH 暴力破解记录
   
   Step 2: 调用 ti.lookup_ip("51.222.47.156")
           → 微步在线评分: 85/100（高危），绿盟标记: 恶意IP
   
   Step 3: 调用 asset.get_by_ip("目标服务器IP")
           → 目标资产: 生产数据库服务器，重要性: 高
   
   Step 4: 调用 experience.search("SSH暴力破解")
           → 找到 3 条历史处置经验
   
   Step 5: 组装证据包，交给 LLM 分析
   
   LLM 输出:
   "根据证据分析：
    1. 该 IP (51.222.47.156) 在微步在线评分为 85/100，绿盟标记为恶意IP
    2. 近 30 天有 12 次 SSH 暴力破解记录，攻击频率稳定
    3. 目标为生产数据库服务器（重要性：高），一旦被攻破影响严重
    4. 历史经验显示类似攻击通常为自动化扫描，非误报
    
    结论：非误报，建议立即封禁该 IP 并通知运维团队检查服务器安全日志。
    置信度: 92%"
```

### 2.2 Agent 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AI Agent 引擎架构                             │
│                                                                     │
│  ┌─────────────┐                                                    │
│  │  用户输入     │  "分析这个告警" / "最近有什么攻击趋势"               │
│  └──────┬──────┘                                                    │
│         ↓                                                           │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  Layer 1: 任务分类器 (Task Classifier)                    │        │
│  │  ┌─────────────────────────────────────────────────┐    │        │
│  │  │ 14 种任务原型 (Task Archetypes)                   │    │        │
│  │  │                                                  │    │        │
│  │  │  • entity_lookup       (实体查询)                 │    │        │
│  │  │  • incident_investigation (事件调查)              │    │        │
│  │  │  • metric_trend        (趋势分析)                 │    │        │
│  │  │  • similar_case_retrieval (相似案例检索)           │    │        │
│  │  │  • threat_assessment   (威胁评估)                 │    │        │
│  │  │  • remediation_advice  (处置建议)                 │    │        │
│  │  │  • ... 共 14 种                                  │    │        │
│  │  └──────────────────────┬──────────────────────────┘    │        │
│  └─────────────────────────┼───────────────────────────────┘        │
│                            ↓                                        │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  Layer 2: 证据需求分析 (Evidence Requirement Analysis)   │        │
│  │  ┌─────────────────────────────────────────────────┐    │        │
│  │  │ 21 种证据角色 (Evidence Roles)                    │    │        │
│  │  │                                                  │    │        │
│  │  │  • subject_identity    (主体身份)                 │    │        │
│  │  │  • risk_signals        (风险信号)                 │    │        │
│  │  │  • external_intel      (外部情报)                 │    │        │
│  │  │  • historical_cases    (历史案例)                 │    │        │
│  │  │  • asset_context       (资产上下文)               │    │        │
│  │  │  • network_topology    (网络拓扑)                 │    │        │
│  │  │  • ... 共 21 种                                  │    │        │
│  │  └──────────────────────┬──────────────────────────┘    │        │
│  └─────────────────────────┼───────────────────────────────┘        │
│                            ↓                                        │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  Layer 3: 工具调用引擎 (Tool Calling Engine)              │        │
│  │  ┌─────────────────────────────────────────────────┐    │        │
│  │  │ 20+ 工具 (Tools)                                 │    │        │
│  │  │                                                  │    │        │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │    │        │
│  │  │  │alert.    │ │ti.       │ │asset.    │        │    │        │
│  │  │  │search    │ │lookup_ip │ │get_by_ip │        │    │        │
│  │  │  └──────────┘ └──────────┘ └──────────┘        │    │        │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │    │        │
│  │  │  │experience│ │intel.    │ │stats.    │        │    │        │
│  │  │  │.search   │ │ip_report │ │alert_trend│       │    │        │
│  │  │  └──────────┘ └──────────┘ └──────────┘        │    │        │
│  │  └──────────────────────┬──────────────────────────┘    │        │
│  └─────────────────────────┼───────────────────────────────┘        │
│                            ↓                                        │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  Layer 4: 证据包组装 (Evidence Pack Assembly)             │        │
│  │                                                          │        │
│  │  ┌─────────────────────────────────────────────────┐    │        │
│  │  │ {                                               │    │        │
│  │  │   "subject_identity": { "ip": "51.222.47.156",  │    │        │
│  │  │     "country": "CA", "asn": "OVH SAS" },       │    │        │
│  │  │   "risk_signals": { "threatbook_score": 85,     │    │        │
│  │  │     "nsfocus_mark": "malicious" },              │    │        │
│  │  │   "historical_cases": [ { "alert_id": 198,      │    │        │
│  │  │     "action": "blocked", "result": "有效" } ],  │    │        │
│  │  │   "asset_context": { "name": "生产数据库",       │    │        │
│  │  │     "criticality": "high" }                     │    │        │
│  │  │ }                                               │    │        │
│  │  └─────────────────────────────────────────────────┘    │        │
│  └─────────────────────────┬───────────────────────────────┘        │
│                            ↓                                        │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  Layer 5: LLM 分析引擎 (Analysis Engine)                  │        │
│  │                                                          │        │
│  │  Prompt = System Prompt + 证据包 + 经验注入 + 用户问题     │        │
│  │                                                          │        │
│  │  输出结构:                                                │        │
│  │  {                                                       │        │
│  │    "meta": { "task_type": "incident_investigation" },    │        │
│  │    "ste": {                                              │        │
│  │      "strategy": "SSH暴力破解攻击",                       │        │
│  │      "tactics": "字典攻击、凭据填充",                     │        │
│  │      "evidence": "高频登录失败、非常用IP"                 │        │
│  │    },                                                    │        │
│  │    "action": { "recommendation": "封禁IP",               │        │
│  │      "urgency": "high" },                               │        │
│  │    "quality": { "confidence": 0.92,                      │        │
│  │      "risk_level": "high", "review_needed": false }      │        │
│  │  }                                                       │        │
│  └─────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 工具调用链详解

#### 任务原型 → 证据角色 → 工具 映射关系

```python
# 核心设计：三层映射链
# Task Archetype → Evidence Role → Tool

TASK_EVIDENCE_MAP = {
    "incident_investigation": [
        "subject_identity",    # 主体身份 → alert.search, ti.lookup_ip
        "risk_signals",        # 风险信号 → ti.lookup_ip, intel.ip_report
        "historical_cases",    # 历史案例 → experience.search, alert.search
        "asset_context",       # 资产上下文 → asset.get_by_ip
    ],
    "threat_assessment": [
        "external_intel",      # 外部情报 → ti.lookup_ip, intel.ip_report
        "network_topology",    # 网络拓扑 → asset.get_by_ip, asset.search
        "attack_pattern",      # 攻击模式 → alert.search (按event_type)
    ],
    "metric_trend": [
        "time_series",         # 时间序列 → stats.alert_trend
        "distribution",        # 分布统计 → stats.top_attackers
    ],
    # ... 14 种任务原型
}

EVIDENCE_TOOL_MAP = {
    "subject_identity":  ["alert.search", "ti.lookup_ip"],
    "risk_signals":      ["ti.lookup_ip", "intel.ip_report"],
    "historical_cases":  ["experience.search", "alert.search"],
    "asset_context":     ["asset.get_by_ip"],
    "external_intel":    ["ti.lookup_ip", "intel.ip_report"],
    "time_series":       ["stats.alert_trend"],
    "distribution":      ["stats.top_attackers", "stats.alert_distribution"],
    # ... 21 种证据角色
}
```

#### 工具注册表完整列表

```python
# 20+ 工具，按功能分类

# ═══ 告警相关 ═══
alert.search       # 按条件搜索历史告警（IP/类型/时间范围）
alert.get_detail   # 获取单条告警详情（含解析字段、TI结果、AI分析）

# ═══ 威胁情报 ═══
ti.lookup_ip       # 查询单个 IP 的威胁情报（聚合微步/绿盟/奇安信/安恒）
intel.ip_report    # 获取 IP 综合报告（地理位置、ASN、历史攻击记录）

# ═══ 资产管理 ═══
asset.get_by_ip    # 根据 IP 查询资产信息（名称、重要性、区域、负责人）
asset.search       # 搜索资产（按名称/IP/标签/区域）

# ═══ 经验知识 ═══
experience.search  # 搜索历史处置经验（按事件类型/关键词/相似度）

# ═══ 统计分析 ═══
stats.alert_trend       # 告警趋势（按日/周/月统计）
stats.top_attackers     # Top 攻击源 IP 排行
stats.alert_distribution # 告警类型分布
stats.mttr_mttd         # 平均响应/处置时间

# ═══ IP 名单 ═══
ip_list.check      # 检查 IP 是否在黑白名单中
ip_list.add        # 添加 IP 到名单

# ═══ 规则解析 ═══
rule.match         # 测试日志文本与解析规则的匹配
```

### 2.4 Agent 工具调用的 Function Calling 定义

```python
# 每个工具都有标准的 Function Calling 定义
# LLM 通过这些定义知道如何调用工具

tools_schema = [
    {
        "type": "function",
        "function": {
            "name": "alert.search",
            "description": "搜索历史告警记录。可根据IP地址、事件类型、时间范围等条件搜索。",
            "parameters": {
                "type": "object",
                "properties": {
                    "source_ip": {
                        "type": "string",
                        "description": "源IP地址，如 '51.222.47.156'"
                    },
                    "event_type": {
                        "type": "string",
                        "description": "事件类型，如 'SSH暴力破解'、'恶意User-Agent扫描'"
                    },
                    "days": {
                        "type": "integer",
                        "description": "搜索最近N天的告警，默认30"
                    },
                    "limit": {
                        "type": "integer",
                        "description": "返回数量限制，默认10"
                    }
                },
                "required": ["source_ip"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "ti.lookup_ip",
            "description": "查询IP的威胁情报信息。聚合微步在线、绿盟NTI、奇安信、安恒四大平台数据。",
            "parameters": {
                "type": "object",
                "properties": {
                    "ip": {
                        "type": "string",
                        "description": "要查询的IP地址"
                    }
                },
                "required": ["ip"]
            }
        }
    },
    # ... 更多工具定义
]
```

### 2.5 STE 知识闭环系统

```
┌──────────────────────────────────────────────────────────────┐
│              STE (Strategy-Tactics-Evidence) 知识闭环         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 阶段一: 经验提取 (告警闭环时自动触发)                   │    │
│  │                                                      │    │
│  │  告警状态: analysis → disposal → disposed             │    │
│  │                          ↓                           │    │
│  │                   自动触发 STE 提取                    │    │
│  │                          ↓                           │    │
│  │  ┌─────────────────────────────────────────────┐     │    │
│  │  │ LLM 提取 STE:                               │     │    │
│  │  │                                             │     │    │
│  │  │ Strategy (策略): SSH暴力破解攻击             │     │    │
│  │  │   → 攻击者的总体目标和方法论                  │     │    │
│  │  │                                             │     │    │
│  │  │ Tactics (战术): 字典攻击、凭据填充           │     │    │
│  │  │   → 攻击者使用的具体技术手段                  │     │    │
│  │  │                                             │     │    │
│  │  │ Evidence (证据):                            │     │    │
│  │  │   → 高频登录失败 (>100次/小时)               │     │    │
│  │  │   → 非常用IP段 (海外VPS)                     │     │    │
│  │  │   → 非工作时间 (凌晨2-5点)                   │     │    │
│  │  │   → 使用默认用户名 (root/admin)              │     │    │
│  │  └─────────────────────────────────────────────┘     │    │
│  └──────────────────────────────────────────────────────┘    │
│                          ↓                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 阶段二: 经验审核 (管理员操作)                          │    │
│  │                                                      │    │
│  │  状态流转: draft → published (审核通过)               │    │
│  │           draft → rejected (审核拒绝)                 │    │
│  │                                                      │    │
│  │  审核内容: STE 提取是否准确、是否有价值                │    │
│  └──────────────────────────────────────────────────────┘    │
│                          ↓                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 阶段三: 经验注入 (下次研判时自动匹配)                  │    │
│  │                                                      │    │
│  │  新告警进入 → 系统自动搜索匹配的经验                   │    │
│  │            → 按事件类型 + IP段 + 攻击模式匹配          │    │
│  │            → 注入到 Prompt 中作为历史参考              │    │
│  │                                                      │    │
│  │  效果:                                               │    │
│  │  • 第1次遇到某类攻击: LLM 只能基于通用知识分析         │    │
│  │  • 第10次遇到同类攻击: LLM 可以参考9次历史经验         │    │
│  │  • 系统越用越聪明，研判准确率持续提升                   │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### 2.6 Prompt 管理系统

```python
# 6 种 Prompt 类型，覆盖不同场景

PROMPT_TYPES = {
    "alert_analysis": {
        "desc": "告警研判",
        "variables": ["{{alert_data}}", "{{evidence_pack}}", "{{experiences}}"],
        "usage": "Agent 分析告警时的主 Prompt"
    },
    "ste_extract": {
        "desc": "STE 经验提取",
        "variables": ["{{alert_data}}", "{{disposal_result}}"],
        "usage": "告警闭环后自动提取 STE"
    },
    "evidence_extract": {
        "desc": "证据字段提取",
        "variables": ["{{raw_log}}"],
        "usage": "从原始日志中提取结构化字段（严格: 禁止幻觉）"
    },
    "template_generate": {
        "desc": "AI 模板生成",
        "variables": ["{{requirement}}", "{{available_fields}}"],
        "usage": "根据需求自动生成报告模板"
    },
    "chat": {
        "desc": "通用对话",
        "variables": ["{{context}}", "{{tool_results}}"],
        "usage": "AI 助手多轮对话"
    },
    "regex_generate": {
        "desc": "正则表达式生成",
        "variables": ["{{sample_log}}", "{{target_fields}}"],
        "usage": "根据日志样本自动生成解析正则"
    }
}
```

### 2.7 多模型供应商抽象层

```python
# ai_gateway.py — 统一的 LLM 调用接口
# 支持 9 种模型供应商，一个接口调用所有

SUPPORTED_PROVIDERS = {
    "openai":      {"models": ["gpt-4o", "gpt-4o-mini", "gpt-3.5-turbo"]},
    "deepseek":    {"models": ["deepseek-chat", "deepseek-reasoner"]},
    "siliconflow": {"models": ["Qwen/Qwen2.5-72B-Instruct", "..."]},
    "qwen":        {"models": ["qwen-max", "qwen-plus", "qwen-turbo"]},
    "zhipu":       {"models": ["glm-4", "glm-4-flash"]},
    "moonshot":    {"models": ["moonshot-v1-8k", "moonshot-v1-32k"]},
    "anthropic":   {"models": ["claude-3.5-sonnet", "claude-3-haiku"]},
    "google":      {"models": ["gemini-pro", "gemini-flash"]},
    "ollama":      {"models": ["llama3", "qwen2", "..."]},  # 本地部署
}

# 统一接口
class AiGateway:
    def chat_completion(self, messages, model, temperature=0.7):
        """同步调用"""
        provider = self._get_provider(model)
        return provider.chat(messages, model, temperature)
    
    async def async_chat_stream(self, messages, model, temperature=0.7):
        """异步流式调用 (SSE)"""
        provider = self._get_provider(model)
        async for chunk in provider.stream(messages, model, temperature):
            yield chunk
```

---

## 三、简历编写指南

### 3.1 推荐写法（Agent 侧重点）

> **AI-Guardian — 基于证据驱动 Agent 的安全运营平台** ｜ 全栈开发
>
> *技术栈：React 18 + TypeScript / FastAPI + SQLAlchemy / LangChain + LLM / Docker*
>
> - 设计了 **14 种任务原型 × 21 种证据角色** 的 Agent 工具调用链，实现"先收集证据，再让 LLM 分析"的架构，从根源上防止 AI 幻觉
> - 实现了 **STE（Strategy-Tactics-Evidence）知识闭环**，从已处置告警中自动提取经验知识，注入后续研判 Prompt，形成"越用越聪明"的正反馈循环
> - 构建了 **20+ 工具的 Agent 工具注册表**，涵盖告警搜索、威胁情报、资产关联、经验检索、统计分析等能力
> - 设计了 **结构化 Prompt 管理系统**（6 种 Prompt 类型），支持工作区级别的自定义和版本管理
> - 实现了 **多模型供应商统一网关**（支持 OpenAI/DeepSeek/Qwen/GLM 等 9 种），通过 Function Calling 驱动工具调用
> - 开发了 **SSE 流式 Agent 对话**，支持实时工具调用过程展示和多轮上下文管理

### 3.2 不同岗位的 Agent 侧重点

#### AI 工程师 / Agent 开发

```
重点：
✅ 证据驱动架构设计（防幻觉）
✅ 14×21 任务-证据-工具三层映射
✅ Function Calling 工具定义与调用
✅ STE 知识闭环（自动学习）
✅ Prompt Engineering（6种类型）
✅ 多模型供应商抽象层
✅ SSE 流式输出 + 工具调用过程展示
```

#### 后端开发

```
重点：
✅ Agent 引擎的服务层设计
✅ 工具注册表的插件化架构
✅ 后台任务队列（AI研判异步化）
✅ 数据库模型设计（告警/经验/会话）
✅ 多租户数据隔离
```

#### 前端开发

```
重点：
✅ SSE 流式渲染（AI 对话实时展示）
✅ 工具调用过程的可视化
✅ Prompt 管理界面设计
✅ STE 经验库的展示与审核
✅ React Query 数据层设计
```

---

## 四、Agent 核心技术深度解析

### 4.1 工具调用的完整流程

```python
# 实际代码简化版 — ai_agent.py

async def run_agent(user_input: str, alert_id: int = None):
    """Agent 主入口"""
    
    # 1. 构建初始消息
    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        HumanMessage(content=user_input)
    ]
    
    # 2. 如果有关联告警，注入告警数据
    if alert_id:
        alert = get_alert(alert_id)
        messages.append(SystemMessage(
            content=f"当前告警数据:\n{json.dumps(alert, ensure_ascii=False)}"
        ))
    
    # 3. Agent 循环（最多 5 轮工具调用）
    for round in range(5):
        # 调用 LLM（带工具定义）
        response = await llm.ainvoke(
            messages,
            tools=TOOLS_SCHEMA  # 20+ 工具的 Function Calling 定义
        )
        
        # 检查是否有工具调用
        if not response.tool_calls:
            # 没有工具调用，返回最终回答
            return response.content
        
        # 4. 执行工具调用
        messages.append(response)  # 添加 assistant 消息
        for tool_call in response.tool_calls:
            tool_name = tool_call["name"]
            tool_args = tool_call["args"]
            
            # 执行工具
            result = await execute_tool(tool_name, tool_args)
            
            # 添加工具结果
            messages.append(ToolMessage(
                content=json.dumps(result, ensure_ascii=False),
                tool_call_id=tool_call["id"]
            ))
    
    # 5. 超过最大轮数，用当前上下文生成最终回答
    final_response = await llm.ainvoke(messages)
    return final_response.content
```

### 4.2 证据包的标准格式

```json
{
    "evidence_pack": {
        "subject_identity": {
            "ip": "51.222.47.156",
            "country": "CA",
            "asn": "16276 OVH SAS",
            "first_seen": "2024-01-15",
            "last_seen": "2024-06-17"
        },
        "risk_signals": {
            "threatbook": {
                "score": 85,
                "judgments": ["恶意IP", "僵尸网络C&C"],
                "confidence": "high"
            },
            "nsfocus": {
                "mark": "malicious",
                "categories": ["暴力破解", "SSH攻击"]
            },
            "qianxin": {
                "level": "high",
                "tags": ["Botnet", "Scanner"]
            }
        },
        "historical_cases": [
            {
                "alert_id": 198,
                "event_type": "SSH暴力破解",
                "created_at": "2024-06-15",
                "disposal_action": "block_ip",
                "result": "有效，攻击停止",
                "analyst_comment": "自动化扫描，非针对性攻击"
            }
        ],
        "asset_context": {
            "target_ip": "10.0.0.5",
            "asset_name": "生产数据库服务器",
            "criticality": "high",
            "area": "华东-上海",
            "owner": "运维团队"
        }
    }
}
```

### 4.3 Prompt 模板示例（告警研判）

```python
ALERT_ANALYSIS_PROMPT = """
你是一个资深的安全运营分析师。请基于以下证据分析告警。

## 告警信息
{{alert_data}}

## 证据包
{{evidence_pack}}

## 历史经验
{{experiences}}

## 分析要求
1. 判断是否为误报，并说明理由
2. 评估威胁等级（低/中/高/紧急）
3. 给出处置建议
4. 输出 STE（Strategy-Tactics-Evidence）

## 输出格式（严格JSON）
{
    "meta": {
        "task_type": "incident_investigation",
        "analysis_time": "ISO时间"
    },
    "conclusion": {
        "is_false_positive": false,
        "threat_level": "high",
        "confidence": 0.92,
        "reasoning": "详细分析过程..."
    },
    "ste": {
        "strategy": "攻击者的总体策略",
        "tactics": "具体技术手段",
        "evidence": "关键证据点"
    },
    "action": {
        "recommendation": "建议处置方式",
        "urgency": "high",
        "affected_assets": ["10.0.0.5"]
    },
    "quality": {
        "confidence": 0.92,
        "risk_level": "high",
        "review_needed": false,
        "data_completeness": "high"
    }
}
"""
```

### 4.4 为什么用 SSE 而不是 WebSocket？

```
AI 对话的通信特点：
├── 方向: 主要是 服务端 → 客户端（LLM 生成的文本流）
├── 频率: 不是高频双向通信
├── 场景: 用户发一条消息，服务端流式返回
└── 断线: 需要自动重连

SSE 的优势：
✅ 单向推送天然适合 LLM 流式输出
✅ 基于 HTTP，Nginx 代理配置简单
✅ 浏览器原生支持 EventSource API
✅ 自动重连机制
✅ 实现简单（sse-starlette 一行代码）

WebSocket 的优势（本项目不需要）：
❌ 需要协议升级，Nginx 需额外配置
❌ 双向通信能力在 AI 对话中用不到
❌ 实现复杂度高

# FastAPI 实现
from sse_starlette.sse import EventSourceResponse

@router.get("/ai/chat/stream")
async def chat_stream(message: str):
    async def generate():
        async for chunk in agent.async_chat_stream(message):
            yield {"data": chunk}
    
    return EventSourceResponse(generate())
```

---

## 五、Agent 面试题（附参考答案）

### 5.1 Agent 架构设计

#### Q1: 为什么选择"证据驱动"架构而不是直接让 LLM 分析？

**参考答案：**

> 核心问题是 **AI 幻觉**。如果直接把原始日志丢给 LLM，它会基于通用知识"编造"分析结果，而不是基于真实数据。
>
> 证据驱动的架构通过三层映射链解决这个问题：
>
> ```
> 任务原型(14种) → 证据角色(21种) → 工具(20+)
> ```
>
> 具体流程：
> 1. **任务分类**：先判断用户想做什么（如"事件调查"）
> 2. **证据需求**：确定需要哪些证据（如"主体身份"、"风险信号"）
> 3. **工具调用**：调用对应工具收集真实数据
> 4. **证据组装**：将工具返回的结构化数据组装成证据包
> 5. **LLM 分析**：基于证据包进行分析，而不是凭空猜测
>
> 这样做的好处：
> - LLM 的每个结论都有数据支撑
> - 可以追溯分析过程（哪些证据导致了什么结论）
> - 避免了"看起来很专业但其实是编的"问题

#### Q2: 14 种任务原型和 21 种证据角色是怎么设计出来的？

**参考答案：**

> 来自对安全运营实际场景的分析：
>
> **任务原型**（14种）来源于安全分析师的日常工作：
> - 看到一个 IP → `entity_lookup`（实体查询）
> - 调查一个事件 → `incident_investigation`（事件调查）
> - 看趋势数据 → `metric_trend`（趋势分析）
> - 找相似案例 → `similar_case_retrieval`（案例检索）
> - 评估威胁 → `threat_assessment`（威胁评估）
> - 给处置建议 → `remediation_advice`（处置建议）
>
> **证据角色**（21种）来源于分析师需要的信息类型：
> - 这个 IP 是谁？→ `subject_identity`（主体身份）
> - 有什么风险？→ `risk_signals`（风险信号）
> - 以前处理过吗？→ `historical_cases`（历史案例）
> - 目标资产重要吗？→ `asset_context`（资产上下文）
>
> 设计原则：
> 1. 任务原型覆盖 90% 的安全运营场景
> 2. 证据角色覆盖分析师需要的所有信息类型
> 3. 每个证据角色至少映射到一个工具

#### Q3: 工具调用失败了怎么办？Agent 会陷入死循环吗？

**参考答案：**

> 有三层保护机制：
>
> **1. 调用次数限制**
> ```python
> for round in range(5):  # 最多 5 轮工具调用
>     response = await llm.ainvoke(messages, tools=TOOLS)
>     if not response.tool_calls:
>         break  # 没有工具调用，结束循环
> ```
>
> **2. 工具执行异常捕获**
> ```python
> try:
>     result = await execute_tool(tool_name, tool_args)
> except Exception as e:
>     result = {"error": str(e)}  # 返回错误信息，让 LLM 自行修正
> ```
>
> **3. 参数校验**
> ```python
> # 工具调用前校验参数格式
> if not validate_args(tool_name, tool_args):
>     result = {"error": "参数格式错误，请检查必填字段"}
> ```
>
> **4. 兜底策略**
> 如果 Agent 无法完成任务（5轮都没结果），降级为普通 LLM 对话，直接基于上下文生成回答。

#### Q4: 多轮工具调用时，上下文窗口会爆吗？

**参考答案：**

> 这是一个实际问题。解决方案：
>
> **1. 工具结果压缩**
> ```python
> # 工具返回时只保留关键字段
> def compress_alert_result(alerts):
>     return [{
>         "id": a["id"],
>         "event_type": a["event_type"],
>         "source_ip": a["source_ip"],
>         "created_at": a["created_at"]
>     } for a in alerts[:5]]  # 最多返回5条
> ```
>
> **2. 历史消息裁剪**
> ```python
> # 超过阈值时，保留最近N轮对话
> if count_tokens(messages) > MAX_TOKENS * 0.7:
>     messages = messages[-10:]  # 只保留最近10条
> ```
>
> **3. 证据包预压缩**
> ```python
> # 组装证据包时进行摘要
> evidence_summary = summarize_evidence(raw_evidence)
> ```

### 5.2 Prompt Engineering

#### Q5: 你的 6 种 Prompt 类型有什么区别？为什么不共用一个？

**参考答案：**

> 不同场景对 Prompt 的要求完全不同：
>
> | Prompt 类型 | 特点 | 为什么不能共用 |
> |------------|------|---------------|
> | `alert_analysis` | 需要注入证据包和历史经验 | 需要大量上下文 |
> | `ste_extract` | 只需要告警数据和处置结果 | 输入简单，输出结构化 |
> | `evidence_extract` | **严格禁止幻觉**，只能提取日志中有的信息 | 需要特殊约束 |
> | `template_generate` | 需要可用字段列表 | 生成模板代码 |
> | `chat` | 通用对话，可调用工具 | 最灵活 |
> | `regex_generate` | 需要日志样本和目标字段 | 特定格式输出 |
>
> 特别是 `evidence_extract`，它的 Prompt 里有严格约束：
> ```
> 【重要】你只能提取日志中明确存在的信息。
> 如果某个字段在日志中找不到，必须输出空字符串，绝对不能编造。
> 这是安全分析场景，错误的信息可能导致误判。
> ```

#### Q6: Prompt 是怎么管理的？支持用户自定义吗？

**参考答案：**

> 使用 **结构化 Prompt 管理系统**：
>
> ```python
> class AiPrompt(Base):
>     workspace_id: int      # 所属工作区（多租户）
>     prompt_type: str       # 类型（alert_analysis/ste_extract/...）
>     name: str              # 名称
>     content: str           # Prompt 模板内容
>     is_default: bool       # 是否为默认
>     version: int           # 版本号
> ```
>
> 支持：
> - **工作区级别自定义**：每个团队可以有自己的 Prompt
> - **版本管理**：修改 Prompt 会递增版本号
> - **默认选择**：每个类型可以设置一个默认 Prompt
> - **变量替换**：Prompt 中的 `{{variable}}` 在运行时替换
>
> 这样非技术的安全分析师也可以调整 AI 的行为，而不需要改代码。

### 5.3 STE 知识系统

#### Q7: STE 提取的准确性怎么保证？LLM 提取的质量怎么验证？

**参考答案：**

> 三层质量保证：
>
> **1. Prompt 约束**
> ```python
> STE_EXTRACT_PROMPT = """
> 请从以下告警和处置结果中提取 STE。
> 要求：
> - Strategy: 1-2句话概括攻击者的总体策略
> - Tactics: 列出具体技术手段（不超过5个）
> - Evidence: 列出判断依据（必须来自告警数据，不能编造）
> """
> ```
>
> **2. 人工审核**
> 提取的 STE 初始状态为 `draft`，需要管理员审核后才变为 `published`
>
> **3. 结构化输出校验**
> ```python
> # 使用 Pydantic 校验输出格式
> class STEOutput(BaseModel):
>     strategy: str = Field(max_length=200)
>     tactics: List[str] = Field(max_items=5)
>     evidence: List[str] = Field(max_items=10)
> ```

#### Q8: 经验匹配是怎么做的？怎么找到"相似"的历史案例？

**参考答案：**

> 采用 **多维度匹配** 策略：
>
> ```python
> def search_experiences(alert):
>     candidates = []
>     
>     # 1. 事件类型精确匹配
>     by_type = db.query(AiExperience).filter(
>         AiExperience.event_type == alert.event_type,
>         AiExperience.status == "published"
>     ).all()
>     
>     # 2. IP 段匹配（同一 C 段）
>     ip_prefix = ".".join(alert.source_ip.split(".")[:3])
>     by_ip = db.query(AiExperience).filter(
>         AiExperience.source_ip.like(f"{ip_prefix}%")
>     ).all()
>     
>     # 3. 关键词匹配
>     by_keyword = db.query(AiExperience).filter(
>         AiExperience.strategy.contains(alert.event_type)
>     ).all()
>     
>     # 合并去重，按相关性排序
>     candidates = deduplicate(by_type + by_ip + by_keyword)
>     return candidates[:5]  # 最多返回5条
> ```

### 5.4 LLM 集成

#### Q9: 为什么支持 9 种模型供应商？怎么做的抽象？

**参考答案：**

> **为什么支持多种：**
> 1. **成本**：不同模型价格差异大（GPT-4o vs DeepSeek）
> 2. **能力**：复杂任务用大模型，简单任务用小模型
> 3. **合规**：某些场景需要私有化部署（Ollama）
> 4. **稳定性**：一个供应商挂了可以切换
>
> **怎么抽象：**
> ```python
> class LLMProvider(ABC):
>     @abstractmethod
>     async def chat(self, messages, model, temperature):
>         """同步调用"""
>         pass
>     
>     @abstractmethod
>     async def stream(self, messages, model, temperature):
>         """流式调用"""
>         pass
> 
> class OpenAIProvider(LLMProvider):
>     async def chat(self, messages, model, temperature):
>         return await openai.ChatCompletion.acreate(
>             model=model, messages=messages, temperature=temperature
>         )
> 
> class DeepSeekProvider(LLMProvider):
>     async def chat(self, messages, model, temperature):
>         # DeepSeek 兼容 OpenAI 接口
>         return await openai.ChatCompletion.acreate(
>             base_url="https://api.deepseek.com",
>             model=model, messages=messages
>         )
> ```
>
> 统一网关：
> ```python
> class AiGateway:
>     def __init__(self):
>         self.providers = {
>             "openai": OpenAIProvider(),
>             "deepseek": DeepSeekProvider(),
>             "qwen": QwenProvider(),
>             # ...
>         }
>     
>     def _get_provider(self, model: str) -> LLMProvider:
>         for name, provider in self.providers.items():
>             if model in provider.supported_models:
>                 return provider
>         raise ValueError(f"Unsupported model: {model}")
> ```

#### Q10: Function Calling 是怎么实现的？不同供应商的格式一样吗？

**参考答案：**

> 大部分供应商兼容 OpenAI 的 Function Calling 格式，但有差异：
>
> ```python
> # OpenAI / DeepSeek / Qwen — 格式相同
> tools = [{
>     "type": "function",
>     "function": {
>         "name": "alert.search",
>         "description": "...",
>         "parameters": { ... }
>     }
> }]
> 
> # Anthropic (Claude) — 格式不同
> tools = [{
>     "name": "alert.search",
>     "description": "...",
> "input_schema": { ... }
> }]
> 
> # Google (Gemini) — 格式也不同
> tools = [{
>     "function_declarations": [{
>         "name": "alert.search",
>         "description": "...",
>         "parameters": { ... }
>     }]
> }]
> ```
>
> 解决方案：**适配器模式**
> ```python
> class ToolAdapter:
>     @staticmethod
>     def to_openai_format(tools):
>         return [{"type": "function", "function": t} for t in tools]
>     
>     @staticmethod
>     def to_anthropic_format(tools):
>         return [{"name": t["name"], "input_schema": t["parameters"]} for t in tools]
> ```

### 5.5 工程实现

#### Q11: Agent 的工具执行是同步还是异步的？

**参考答案：**

> **LLM 调用是异步的**（因为要等待网络响应），**工具执行也是异步的**（需要查询数据库或调用外部 API）。
>
> ```python
> async def run_agent(user_input):
>     # 1. 异步调用 LLM
>     response = await llm.ainvoke(messages, tools=TOOLS)
>     
>     # 2. 异步执行工具（可以并发）
>     tasks = [
>         execute_tool(tc["name"], tc["args"]) 
>         for tc in response.tool_calls
>     ]
>     results = await asyncio.gather(*tasks)  # 并发执行
>     
>     # 3. 异步流式输出
>     async for chunk in llm.astream(messages):
>         yield chunk
> ```
>
> 使用 FastAPI 的异步能力，在等待 LLM 响应时不会阻塞其他请求。

#### Q12: SSE 流式输出时，工具调用过程怎么展示给用户？

**参考答案：**

> 通过 **自定义 SSE 事件类型** 展示不同阶段：
>
> ```python
> async def agent_stream(user_input):
>     # 阶段1: 任务分类
>     yield {"event": "status", "data": "正在分析任务类型..."}
>     
>     # 阶段2: 工具调用
>     for tool_call in response.tool_calls:
>         yield {
>             "event": "tool_call",
> "data": json.dumps({
>                 "tool": tool_call["name"],
>                 "args": tool_call["args"],
>                 "status": "calling"
>             })
>         }
>         result = await execute_tool(tool_call["name"], tool_call["args"])
>         yield {
>             "event": "tool_result",
>             "data": json.dumps({
>                 "tool": tool_call["name"],
>                 "result_summary": summarize(result)
>             })
>         }
>     
>     # 阶段3: LLM 分析
>     yield {"event": "status", "data": "正在生成分析报告..."}
>     async for chunk in llm.astream(messages):
>         yield {"event": "token", "data": chunk}
>     
>     # 阶段4: 完成
>     yield {"event": "done", "data": ""}
> ```
>
> 前端渲染：
> ```tsx
> eventSource.addEventListener('tool_call', (e) => {
>     const { tool, args } = JSON.parse(e.data);
>     setToolCalls(prev => [...prev, { tool, args, status: 'calling' }]);
> });
> 
> eventSource.addEventListener('token', (e) => {
>     setAnswer(prev => prev + e.data);
> });
> ```

#### Q13: Agent 的会话历史是怎么管理的？

**参考答案：**

> 使用 **数据库持久化 + 上下文窗口裁剪**：
>
> ```python
> class AiConversation(Base):
>     id: int
>     workspace_id: int
>     user_id: int
>     title: str
>     created_at: datetime
> 
> class AiMessage(Base):
>     id: int
>     conversation_id: int
>     role: str        # "user" / "assistant" / "tool"
>     content: str
>     tool_calls: JSON  # 工具调用记录
>     created_at: datetime
> ```
>
> 上下文管理策略：
> 1. 新消息追加到数据库
> 2. 构建 LLM 输入时，从数据库加载最近 N 条消息
> 3. 超过上下文窗口时，对历史消息进行摘要压缩
> 4. 工具调用的完整记录也保存，支持回溯

### 5.6 综合/开放类

#### Q14: 如果要让 Agent 支持更多工具（比如自动化处置），你会怎么设计？

**参考答案：**

> **1. 工具注册表扩展**
> ```python
> # 新增自动化处置工具
> tools["remediation.block_ip"] = {
>     "description": "自动封禁指定IP",
>     "parameters": {
>         "ip": {"type": "string"},
>         "duration": {"type": "string", "enum": ["1h", "24h", "permanent"]},
>         "reason": {"type": "string"}
>     },
>     "requires_approval": True  # 需要人工审批
> }
> ```
>
> **2. 安全机制**
> - 高风险操作（封禁IP、修改白名单）需要 **人工审批**
> - Agent 只能 **建议** 处置方案，不能直接执行
> - 所有自动化操作都有 **审计日志**
>
> **3. 权限控制**
> ```python
> TOOL_PERMISSIONS = {
>     "alert.search": ["analyst", "disposer", "admin"],
>     "remediation.block_ip": ["admin"],  # 只有管理员可以自动封禁
> }
> ```

#### Q15: 这个 Agent 架构有什么局限性？你会怎么改进？

**参考答案：**

> **当前局限性：**
>
> 1. **任务分类依赖 LLM**：如果 LLM 分类错误，后续整个链路都会走偏
>    - 改进：增加规则引擎兜底（如包含"IP"→自动分类为 `entity_lookup`）
>
> 2. **工具调用是串行的**：当前一轮只调用一个工具
>    - 改进：支持 **并行工具调用**（一次返回多个 tool_call）
>
> 3. **没有长期记忆**：每次对话都是独立的
>    - 改进：引入 **向量数据库**（如 Milvus），存储历史分析结果，支持语义检索
>
> 4. **STE 匹配是关键词匹配**：不够智能
>    - 改进：使用 **Embedding 向量相似度** 匹配经验
>
> 5. **没有自我反思能力**：Agent 不会检查自己的分析是否合理
>    - 改进：引入 **Reflection 机制**，让 Agent 在输出前自我审查

#### Q16: 对比 LangChain Agent 和你自研的 Agent，有什么异同？

**参考答案：**

> **相同点：**
> - 都基于 Function Calling 驱动工具调用
> - 都支持多轮工具调用循环
> - 都有工具注册机制
>
> **不同点：**
>
> | 维度 | LangChain Agent | AI-Guardian 自研 |
> |------|----------------|-----------------|
> | 复杂度 | 通用框架，功能丰富 | 轻量级，针对安全场景定制 |
> | 证据链 | 没有证据-任务映射 | 14×21 三层映射链 |
> | 知识系统 | 需要额外集成 | 内置 STE 闭环 |
> | Prompt 管理 | 硬编码或手动管理 | 数据库管理 + 版本控制 |
> | 多模型 | 需要配置多个 provider | 统一网关抽象 |
> | 适用场景 | 通用 AI 应用 | 安全运营垂直场景 |
>
> 自研的核心优势是 **垂直场景的深度优化**，比如证据驱动、STE 闭环、安全审计等，这些在通用框架中需要大量定制。

---

## 六、前置学习内容

### 6.1 AI Agent 开发（重点）

#### 必须掌握

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| OpenAI API 基础 | [OpenAI 文档](https://platform.openai.com/docs/) | ⭐⭐⭐⭐⭐ |
| Function Calling | [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling) | ⭐⭐⭐⭐⭐ |
| Prompt Engineering | [Prompt 工程指南](https://www.promptingguide.ai/zh) | ⭐⭐⭐⭐⭐ |
| LangChain 基础 | [LangChain 文档](https://python.langchain.com/docs/) | ⭐⭐⭐⭐ |
| Agent 设计模式 | [ReAct 论文](https://arxiv.org/abs/2210.03629) | ⭐⭐⭐⭐ |
| SSE 流式输出 | [sse-starlette](https://github.com/sysid/sse-starlette) | ⭐⭐⭐ |

#### 学习路线

```
Week 1: LLM API 基础
  ├── Day 1-2: OpenAI Chat Completion API 调用
  ├── Day 3-4: Prompt Engineering 技巧（Few-shot, CoT, ReAct）
  ├── Day 5-6: Function Calling / Tool Use
  └── Day 7: 练习 — 带工具的简单对话机器人

Week 2: Agent 架构
  ├── Day 1-2: LangChain Agent 入门
  ├── Day 3-4: 自定义工具开发
  ├── Day 5-6: Agent 循环（Observe-Think-Act）
  └── Day 7: 练习 — 实现一个简单的 ReAct Agent

Week 3: 进阶 Agent
  ├── Day 1-2: SSE 流式输出 + 工具调用过程展示
  ├── Day 3-4: 多轮对话 + 会话管理
  ├── Day 5-6: 结构化输出（JSON Schema）
  └── Day 7: 练习 — 实现带工具的流式 AI 对话

Week 4: 实战项目
  ├── Day 1-3: 实现一个简易版证据驱动 Agent
  ├── Day 4-5: 工具注册表设计
  ├── Day 6: Prompt 管理系统
  └── Day 7: 测试 + 优化
```

#### Function Calling 核心代码示例

```python
import openai

# 1. 定义工具
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如'北京'"
                    }
                },
                "required": ["city"]
            }
        }
    }
]

# 2. 调用 LLM
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=tools,
    tool_choice="auto"
)

# 3. 处理工具调用
message = response.choices[0].message
if message.tool_calls:
    for tool_call in message.tool_calls:
        # 解析参数
        args = json.loads(tool_call.function.arguments)
        city = args["city"]
        
        # 执行工具
        weather = get_weather(city)  # 你的业务逻辑
        
        # 4. 将结果返回给 LLM
        messages.append(message)
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(weather)
        })
    
    # 5. LLM 基于工具结果生成最终回答
    final_response = openai.chat.completions.create(
        model="gpt-4o",
        messages=messages
    )
    print(final_response.choices[0].message.content)
```

### 6.2 Python 后端基础

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| FastAPI 入门 | [FastAPI 官方文档](https://fastapi.tiangolo.com/zh/) | ⭐⭐⭐⭐⭐ |
| SQLAlchemy ORM | [SQLAlchemy 教程](https://docs.sqlalchemy.org/en/20/orm/quickstart.html) | ⭐⭐⭐⭐⭐ |
| Pydantic 数据校验 | [Pydantic 文档](https://docs.pydantic.dev/) | ⭐⭐⭐⭐ |
| 异步编程 (async/await) | [Python asyncio](https://docs.python.org/3/library/asyncio.html) | ⭐⭐⭐⭐ |
| JWT 认证 | [python-jose](https://python-jose.readthedocs.io/) | ⭐⭐⭐⭐ |

### 6.3 React 前端基础

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| TypeScript 基础 | [TypeScript 手册](https://www.typescriptlang.org/docs/handbook/) | ⭐⭐⭐⭐⭐ |
| React 18 + Hooks | [React 官方教程](https://react.dev/learn) | ⭐⭐⭐⭐⭐ |
| Ant Design | [Ant Design 文档](https://ant.design/docs/react/introduce-cn) | ⭐⭐⭐⭐ |
| React Query | [TanStack Query](https://tanstack.com/query/latest) | ⭐⭐⭐⭐ |
| EventSource (SSE) | [MDN EventSource](https://developer.mozilla.org/zh-CN/docs/Web/API/EventSource) | ⭐⭐⭐ |

### 6.4 DevOps 基础

| 知识点 | 学习资源 | 优先级 |
|--------|---------|--------|
| Docker 基础 | [Docker 入门](https://docs.docker.com/get-started/) | ⭐⭐⭐⭐⭐ |
| Docker Compose | [Compose 文档](https://docs.docker.com/compose/) | ⭐⭐⭐⭐ |
| Nginx 配置 | [Nginx 入门](https://nginx.org/en/docs/beginners_guide.html) | ⭐⭐⭐ |
| Git 版本控制 | [Git 教程](https://git-scm.com/book/zh/v2) | ⭐⭐⭐⭐⭐ |

### 6.5 推荐学习资源

#### 书籍
- 《Building LLM Apps》— LLM 应用开发实战
- 《LangChain 实战》— LangChain 中文教程
- 《Designing Data-Intensive Applications》— 系统设计

#### 在线课程
- [DeepLearning.AI - Building AI Agents](https://www.deeplearning.ai/) — 免费
- [LangChain 官方教程](https://python.langchain.com/docs/get_started/quickstart) — 免费
- [OpenAI Cookbook](https://cookbook.openai.com/) — 实战示例

#### 开源项目参考
- [LangChain](https://github.com/langchain-ai/langchain) — Agent 框架
- [AutoGPT](https://github.com/Significant-Gravitas/AutoGPT) — 自主 Agent
- [CrewAI](https://github.com/joaomdmoura/crewAI) — 多 Agent 协作

---

## 附录：Agent 核心概念速查

| 概念 | 解释 |
|------|------|
| **Agent** | 能够自主决策和执行任务的 AI 系统 |
| **Tool/Function** | Agent 可以调用的外部能力（如搜索、查询） |
| **Function Calling** | LLM 输出结构化的工具调用请求 |
| **ReAct** | Reasoning + Acting 的 Agent 设计模式 |
| **Evidence-Driven** | 先收集证据再分析，防止幻觉 |
| **STE** | Strategy-Tactics-Evidence 知识提取框架 |
| **SSE** | Server-Sent Events，服务端推送流式数据 |
| **Prompt Engineering** | 设计和优化 LLM 输入提示词的技术 |
| **Hallucination** | AI 幻觉，生成看似合理但不正确的内容 |
| **Multi-Turn** | 多轮对话，支持上下文连续交互 |

---

> 📝 **面试核心话术**：当面试官问到 Agent 相关问题时，重点强调三点：
> 1. **证据驱动** — 不让 LLM 凭空分析，先用工具收集真实数据
> 2. **知识闭环** — STE 系统让 Agent 越用越聪明
> 3. **工程落地** — 不只是 demo，而是完整的生产级系统（多租户、权限、审计）
>
> 祝面试顺利！🚀
