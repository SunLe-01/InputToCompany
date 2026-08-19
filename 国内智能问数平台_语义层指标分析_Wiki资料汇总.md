# 国内智能问数平台语义层与指标分析 Wiki / 文档资料汇总

> 整理时间：2026-08-14  
> 主题：智能问数（ChatBI / NL2SQL / NL2Semantic2SQL）中的语义层、指标体系、指标口径、维度、业务知识与指标查询。

---

## 1. 文档说明

国内公开资料中，严格以 **Wiki** 形式公开“智能问数语义层 + 指标体系”的项目并不多。更常见的形式是：

1. 开源项目 GitHub Wiki；
2. 云厂商官方帮助文档；
3. BI 厂商指标中心文档；
4. 智能问数厂商关于 Semantic Layer、MQL、NL2Semantic2SQL 的技术文章；
5. 开源 ChatBI / Text2SQL 项目的 README 与知识库设计说明。

因此本文不局限于名字叫“Wiki”的网页，而是筛选**能直接用于设计企业智能问数指标 Wiki、语义层、指标字典和知识库的公开资料**。

---

# 2. 推荐资料总览

| 序号 | 平台 / 项目 | 文档类型 | 重点内容 | 与智能问数语义层关系 | 推荐度 |
|---|---|---|---|---|---|
| 1 | 腾讯音乐 SuperSonic | GitHub Wiki | Semantic Layer、指标、维度、Schema Mapper、逻辑 SQL | 非常直接 | ★★★★★ |
| 2 | 阿里云 Dataphin | 官方帮助文档 | 原子指标、派生指标、业务限定、统计粒度、统计周期 | 指标治理核心 | ★★★★★ |
| 3 | 阿里云 Quick BI 小Q问数 | 官方帮助文档 | 业务定义、数据解释、同义词、知识管理、问数配置 | AI 业务语义层 | ★★★★★ |
| 4 | 阿里云 DataWorks ChatBI | 官方帮助文档 | 问题模板、术语、业务逻辑、知识库 | NL2SQL 知识增强 | ★★★★☆ |
| 5 | 火山引擎 DataLeap | 官方帮助文档 | 指标平台、原子/衍生/复合指标、维度、指标字典 | 指标体系建设 | ★★★★★ |
| 6 | 华为云 DataArts Studio | 官方帮助文档 | 原子指标、衍生指标、业务指标、统计维度、限定 | 指标建模规范 | ★★★★★ |
| 7 | 帆软 FineBI / FineChatBI | 官方帮助文档 | 指标中心、模型、指标/维度、数据目录、语义层 | ChatBI 语义层 | ★★★★★ |
| 8 | Aloudata CAN / Agent | 官方技术文章 | NL2SQL vs NL2Semantic2SQL、MQL、指标语义层 | 技术路线非常直接 | ★★★★★ |
| 9 | 亿问 Data Agent | 官方技术词典 | MQL、指标语义层、企业世界语义层、指标口径 | 新型语义层理论 | ★★★★☆ |
| 10 | 灵犀 BI（LingxiBI） | 开源项目介绍 / GitHub | 知识库、Few-shot、精选指标库、列画像 | 知识增强型 Text2SQL | ★★★★☆ |
| 11 | DataEase SQLBot | GitHub / 官网 | RAG、术语库、SQL 示例、权限、Text2SQL | 可作为对照方案 | ★★★☆☆ |

---

# 3. 腾讯音乐 SuperSonic

## 3.1 文档

**SuperSonic GitHub Wiki**  
网页：<https://github.com/tencentmusic/supersonic/wiki>

GitHub 项目主页：  
<https://github.com/tencentmusic/supersonic>

## 3.2 为什么值得重点研究

SuperSonic 是国内公开项目中少数直接把 **Chat BI + Headless BI / Semantic Layer** 组合起来讲解的项目。

其 Wiki 对 Semantic Layer 的定位非常清楚：

- 将数据库的表、字段等技术对象翻译成业务可理解的 **指标、维度、标签**；
- 将关联关系、计算公式等技术口径统一管理；
- 让 LLM 不需要自己重新推断复杂 Join 和指标计算逻辑；
- 通过语义模型提高问数准确率和可治理性。

## 3.3 重点阅读章节

建议重点看 Wiki 中以下内容：

```text
总体思路
↓
系统设计
↓
Semantic Layer
↓
Schema Mapper
↓
Semantic Parser
↓
Semantic Corrector
↓
Rule-based Parser
```

其主链路可以概括为：

```text
用户自然语言问题
        ↓
Schema Mapper
指标 / 维度 / 字段 / 维值映射
        ↓
Semantic Parser
        ↓
生成逻辑 SQL
        ↓
Semantic Layer
统一指标、维度、Join、计算公式
        ↓
生成物理 SQL
        ↓
数据库 / OLAP 引擎
```

## 3.4 对企业指标 Wiki 的参考价值

适合参考：

- 指标名称；
- 维度；
- 标签；
- 指标别名；
- 计算公式；
- 数据模型关系；
- Schema 映射；
- 逻辑 SQL 与物理 SQL 分离；
- LLM 和 Semantic Layer 的职责边界。

**建议：研究智能问数“语义层总体架构”时优先阅读。**

---

# 4. 阿里云 Dataphin

## 4.1 核心指标定义文档

### 文档一：明确统计指标

网页：  
<https://help.aliyun.com/zh/dataphin/fullmanaged/getting-started/the-statistical-index>

Dataphin 将统计指标拆分为：

```text
业务过程
原子指标
业务限定
统计周期
统计粒度（维度）
派生指标
```

其核心关系可以概括为：

```text
派生指标
=
统计周期
+ 业务限定
+ 原子指标
+ 统计粒度
```

例如：

```text
原子指标：支付金额
统计周期：最近7天
业务限定：支付宝支付
统计粒度：买家

↓

最近7天买家支付宝支付金额
```

### 文档二：创建原子指标

网页：  
<https://help.aliyun.com/zh/dataphin/fullmanaged/user-guide/creating-atomic-metrics>

重点研究：

- 指标统计口径；
- 计算逻辑；
- 来源模型；
- 指标定义与实际数据开发之间的关系。

### 文档三：创建派生指标和衍生指标

网页：  
<https://help.aliyun.com/zh/dataphin/fullmanaged/user-guide/adding-derived-and-derived-metrics>

重点研究：

- 原子指标如何组成派生指标；
- 派生指标之间如何进一步组成衍生指标；
- 指标口径如何避免重复定义。

## 4.2 对智能问数的意义

智能问数真正难的通常不是模型不会写 SQL，而是：

```text
“销售额”究竟指什么？
“客户数”是否去重？
“订单数”是否包含取消订单？
“最近一个月”是自然月还是最近30天？
“部门”按当前组织还是历史组织？
```

Dataphin 的指标体系正适合解决这些问题。

## 4.3 对指标 Wiki 的参考价值

推荐将以下字段纳入企业指标 Wiki：

```text
指标名称
指标编码
指标类型
业务过程
原子指标
业务限定
统计周期
统计粒度
计算逻辑
来源模型
主题域
负责人
```

**建议：研究“指标字典、指标口径”时优先阅读。**

---

# 5. 阿里云 Quick BI 小Q问数

## 5.1 企业业务知识 / 知识管理

网页：  
<https://help.aliyun.com/zh/quick-bi/user-guide/manage-knowledge>

相关问数配置：  
<https://help.aliyun.com/zh/quick-bi/user-guide/prepare-data>

## 5.2 重点内容

Quick BI 的知识管理非常适合参考智能问数的 **业务语义知识层**。

核心字段包括：

```text
业务定义
数据解释
同义词
生效范围
专用知识
```

例如：

```text
业务定义：GMV

数据解释：
统计已支付且符合企业统一订单有效规则的成交金额。

同义词：
成交额
交易额
销售流水
总交易金额
```

用户输入：

```text
上个月成交额是多少？
```

语义层可以先把“成交额”映射到企业标准定义 GMV，再执行指标查询。

## 5.3 对指标 Wiki 的参考价值

除了传统指标定义，建议为每个指标额外维护：

```text
指标业务解释
同义词
缩写
常见用户问法
反例
适用范围
数据资源范围
注意事项
```

这类信息对 LLM 的价值通常比单纯给出字段名更大。

**建议：研究“指标 Wiki 如何让 AI 看懂”时重点阅读。**

---

# 6. 阿里云 DataWorks ChatBI

## 6.1 创建知识库提升准确率

网页：  
<https://help.aliyun.com/zh/dataworks/user-guide/chatbi-knowledge-base>

## 6.2 核心设计

DataWorks ChatBI 的知识库主要包含：

```text
问题模板
术语管理
业务逻辑
```

对应解决：

```text
高频固定分析问题
业务缩写 / 同义词映射
指标计算规则 / 过滤规则
```

例如：

```text
有效订单
=
订单金额 > 0
AND
订单状态 = 已支付
```

这类业务规则如果不进入语义层或知识库，LLM 即使生成语法正确 SQL，也可能得到业务错误结果。

## 6.3 参考价值

适合补充到指标 Wiki 的 AI 部分：

- 术语；
- 同义词；
- 问题模板；
- SQL / 查询示例；
- 业务过滤逻辑；
- Few-shot / Golden Query。

---

# 7. 火山引擎 DataLeap 指标平台

## 7.1 指标平台使用流程最佳实践

网页：  
<https://docs.volcengine.com/docs/6260/1337881?lang=zh>

指标 / 维度管理：  
<https://www.volcengine.com/docs/6260/1261858>

业务指标管理：  
<https://www.volcengine.com/docs/6260/1261841>

## 7.2 DataLeap 指标体系

官方流程包括：

```text
业务需求指标
      ↓
指标基础元素
      ↓
技术指标
├─ 原子指标
├─ 衍生指标
└─ 复合指标
      ↓
指标维度
      ↓
模型绑定
      ↓
业务指标
      ↓
指标字典
      ↓
指标专题 / 指标应用
```

这套流程非常适合企业构建自己的指标 Wiki，因为它不仅定义指标，还关注指标从：

```text
定义
→ 建模
→ 存储
→ 发布
→ 搜索
→ 应用
```

的全生命周期。

## 7.3 对指标 Wiki 的参考价值

可以借鉴：

```text
业务线
主题域
业务过程
度量
修饰词
时间周期
指标单位
数据类型
原子指标
衍生指标
复合指标
维度
模型绑定
指标负责人
指标字典
指标专题
```

**建议：如果要设计企业“指标中心 / 指标 Wiki 页面结构”，DataLeap 很值得参考。**

---

# 8. 华为云 DataArts Studio

## 8.1 新建衍生指标

网页：  
<https://support.huaweicloud.com/usermanual-dataartsstudio/dataartsstudio_01_0617.html>

## 8.2 新建原子指标

网页：  
<https://support.huaweicloud.com/intl/zh-cn/usermanual-dataartsstudio/dataartsstudio_01_0616.html>

## 8.3 业务指标

网页：  
<https://support.huaweicloud.com/usermanual-dataartsstudio/dataartsstudio_01_0633.html>

## 8.4 核心指标模型

DataArts Studio 将衍生指标定义为：

```text
衍生指标
=
原子指标
+ 统计维度
+ 时间限定
+ 通用限定
```

其中：

```text
原子指标
→ 明确统计口径和计算逻辑

统计维度
→ GROUP BY 分析视角

时间限定
→ 时间范围标准定义

通用限定
→ 业务过滤范围，类似 WHERE 条件
```

DataArts 还区分：

```text
业务指标
vs
技术指标
```

业务指标用于业务定义和业务口径，技术指标负责真正的数据计算实现。

## 8.5 对指标 Wiki 的参考价值

非常适合企业采用“双层定义”：

```text
业务层
指标名称
业务含义
统计口径
业务负责人
应用场景

        ↓ 映射

技术层
原子指标
计算公式
统计维度
限定条件
来源表
字段
刷新频率
```

**建议：如果希望 Wiki 同时服务业务人员与开发人员，DataArts 的设计非常有参考价值。**

---

# 9. 帆软 FineBI / FineChatBI 指标中心

## 9.1 指标中心简介

网页：  
<https://help.fanruan.com/finebi/edition-view-28533-0.html>

FineBI 7 更新说明：  
<https://help.fanruan.com/finebi/edition-view-29062-0.html>

## 9.2 核心价值

FineBI 指标中心明确将自己的能力定位为：

```text
统一数据模型
+
指标加工
+
指标管理
+
指标应用
```

并为 FineChatBI 提供语义层支撑。

核心组成包括：

```text
模型管理
指标管理
维度管理
指标集
数据目录
指标发布
版本控制
审批
血缘
```

整体链路可以理解为：

```text
事实数据 / 维度数据
        ↓
统一数据模型
        ↓
指标 + 维度
        ↓
指标集
        ↓
数据目录
        ↓
FineBI / FineChatBI
```

## 9.3 对指标 Wiki 的参考价值

特别值得参考以下治理字段：

- 指标状态；
- 指标版本；
- 发布状态；
- 指标申请；
- 指标审批；
- 数据血缘；
- 指标所属模型；
- 可分析维度；
- 指标集；
- 数据目录。

**建议：研究“指标从定义到发布再被 ChatBI 使用”的完整闭环时重点阅读。**

---

# 10. Aloudata CAN / Agent

## 10.1 NL2SQL vs NL2Semantic2SQL

网页：  
<https://aloudata.com/blogs/nl2sql-vs-nl2semantic2sql-technical-path>

## 10.2 NL2MQL2SQL 术语说明

网页：  
<https://aloudata.com/resources/glossary/nl2mql2sql>

## 10.3 NoETL 指标平台

网页：  
<https://aloudata.com/blogs/noetl-metrics-platform-data-governance>

## 10.4 核心技术路线

传统智能问数通常是：

```text
自然语言
↓
LLM
↓
SQL
↓
数据库
```

Aloudata 强调的路线则是：

```text
自然语言
↓
LLM 识别业务意图
↓
MQL / Semantic Query
↓
指标语义层
↓
Semantic Engine
↓
SQL
↓
数据库
```

这意味着大模型主要负责：

```text
理解“用户想查什么”
```

而不直接承担：

```text
猜测表关系
猜测指标公式
猜测复杂 Join
猜测业务口径
```

## 10.5 对指标 Wiki 的参考价值

如果企业计划采用 NL2Semantic2SQL，可以在 Wiki 中维护：

```text
Metric
Dimension
Measure
Filter
Time Range
Business Constraint
Metric Relation
Semantic Model
MQL 示例
数据血缘
权限
```

**建议：写智能问数方案中的“为什么必须建设语义层”章节时重点参考。**

---

# 11. 亿问 Data Agent

## 11.1 技术词典

网页：  
<https://yiwendata.com/resources/glossary>

## 11.2 核心概念

其公开技术词典涉及：

```text
MQL（指标查询语言）
指标平台
指标口径
指标配置层
企业世界语义层
对象 / 事件语义
时间语义
动态语义
```

其中指标口径不仅是 SQL 表达式，而包含：

```text
计算公式
统计维度
时间周期
数据范围
业务限定条件
```

它还把传统“指标语义”进一步扩展到企业对象、事件和业务关系。

## 11.3 参考价值

传统指标层解决：

```text
“这个数怎么算？”
```

企业世界语义层进一步试图解决：

```text
“客户是什么？”
“订单是什么事件？”
“客户和订单是什么关系？”
“企业业务过程如何运转？”
```

适合研究未来更复杂的数据分析 Agent，而不只是简单查指标。

---

# 12. 灵犀 BI（LingxiBI）

## 12.1 开源项目

GitHub：  
<https://github.com/bonfirer/ai-report>

开源介绍：  
<https://github.com/ruanyf/weekly/issues/10557>

## 12.2 与传统语义层的区别

灵犀 BI 当前公开方案更偏向：

```text
Text2SQL
+
知识库
+
Few-shot
+
精选指标库
+
列画像
+
用户反馈学习
```

它的知识沉淀主要包含：

| 知识层 | 作用 |
|---|---|
| 知识库 | 表关系、字段含义、业务规则 |
| Few-shot | 保存正确问题与查询示例 |
| 精选指标库 | 保存经过验证的命名 SQL 指标 |
| 列画像 | 保存枚举值、范围、样本等字段特征 |

其流程更接近：

```text
用户提问
↓
AI 生成 SQL
↓
执行 / 用户反馈
↓
提炼知识
↓
知识库 / Few-shot / 指标库
↓
下一次问数召回
```

## 12.3 参考价值

适合研究：

- Golden Query / Few-shot；
- 指标 SQL 模板；
- 用户反馈闭环；
- 表字段业务说明；
- 列值画像；
- 问数知识库自动积累。

需要注意：它与 SuperSonic、FineBI、Aloudata 这种“显式 Semantic Layer”路线并不完全相同，更接近**知识增强型 Text2SQL / ChatBI**。

---

# 13. DataEase SQLBot

## 13.1 GitHub

网页：  
<https://github.com/dataease/SQLBot>

官网：  
<https://sqlbot.org/>

## 13.2 核心设计

SQLBot 是国内较有代表性的开源智能问数系统，核心能力包括：

```text
LLM
RAG
Text2SQL
术语库
自定义 Prompt
SQL 示例
数据权限
工作空间隔离
```

它不像 SuperSonic 那样把 Headless BI Semantic Layer 作为核心架构，因此非常适合作为一个对照组。

可以对比两种路线：

```text
路线 A
NL → LLM → SQL
     ↑
RAG / 术语 / 示例

路线 B
NL → LLM → Semantic Query → Semantic Layer → SQL
```

这有助于分析什么时候“知识库 + Text2SQL”已经够用，什么时候必须建设正式的指标语义层。

---

# 14. 各平台最值得借鉴什么

| 平台 | 最值得学习的部分 |
|---|---|
| SuperSonic | Semantic Layer 与 LLM 的整体关系 |
| Dataphin | 原子指标、派生指标、统计口径设计 |
| Quick BI | 业务定义、同义词、数据解释如何提供给 AI |
| DataWorks ChatBI | 问题模板、术语、业务逻辑知识库 |
| DataLeap | 指标平台全生命周期、指标字典 |
| DataArts Studio | 业务指标与技术指标分层 |
| FineBI | 指标中心、模型、发布、版本、血缘、ChatBI 语义层 |
| Aloudata | NL2Semantic2SQL / NL2MQL2SQL 技术路线 |
| 亿问 | 从指标语义向企业世界语义扩展 |
| 灵犀 BI | Few-shot、精选指标库、用户反馈学习 |
| SQLBot | RAG + Text2SQL 的轻量问数路线 |

---

# 15. 推荐的企业智能问数“指标语义 Wiki”结构

综合以上产品，企业内部 Wiki 建议至少拆成以下七个部分。

## 15.1 指标基本信息

```text
指标中文名称
指标英文名称
指标编码
指标描述
指标类型
主题域
业务过程
负责人
状态
版本
```

## 15.2 业务语义

```text
业务定义
统计口径
业务规则
同义词
缩写
易混淆指标
适用范围
典型业务问题
```

## 15.3 指标计算语义

```text
Measure / 度量
Aggregation / 聚合方式
计算公式
业务限定
时间限定
统计周期
默认过滤条件
复合指标关系
```

## 15.4 维度语义

```text
可分析维度
默认统计粒度
维度层级
时间维度
组织维度
地域维度
产品维度
维值说明
```

## 15.5 数据模型映射

```text
数据源
数据库
Schema
事实表
维度表
字段
主键
Join 关系
逻辑模型
数据血缘
```

## 15.6 AI 问数知识

```text
指标同义词
用户常见问法
问题模板
Golden Query
Few-shot
正确 SQL 示例
错误问法 / 反例
维值映射
Semantic Query 示例
MQL 示例
```

## 15.7 治理信息

```text
业务负责人
技术负责人
指标 Owner
数据更新频率
质量规则
数据权限
行列权限
发布状态
审批状态
版本历史
修改记录
上下游血缘
```

---

# 16. 推荐的智能问数架构

综合这些厂商的设计，较成熟的企业级方案可以抽象为：

```text
                     用户自然语言问题
                            │
                            ▼
                     意图识别 / LLM
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
          业务术语       指标识别        维度识别
          同义词         Metric          Dimension
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                    Semantic Query
                       / MQL / IR
                            │
                            ▼
                     Semantic Layer
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
           指标定义       维度定义       数据模型
           Metric        Dimension      Join/Schema
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                       SQL Generator
                            │
                            ▼
                     权限 / 策略校验
                            │
                            ▼
                       数据库 / OLAP
                            │
                            ▼
                         查询结果
                            │
                            ▼
                    LLM 解读 / 可视化
```

这里最关键的是：

> **LLM 负责理解问题，Semantic Layer 负责确定业务口径，SQL Engine 负责执行。**

而不是让 LLM 同时猜业务含义、猜指标公式、猜表关系、猜权限，再顺便祈祷 SQL 正确。

---

# 17. 如果只阅读 6 份资料，推荐顺序

如果时间有限，建议按照以下顺序阅读：

### 第一份：SuperSonic Wiki

<https://github.com/tencentmusic/supersonic/wiki>

目的：理解 **Semantic Layer 为什么存在，以及它如何连接 LLM 与数据库**。

---

### 第二份：Dataphin 明确统计指标

<https://help.aliyun.com/zh/dataphin/fullmanaged/getting-started/the-statistical-index>

目的：理解 **指标到底应该怎么拆，指标口径应该怎样定义**。

---

### 第三份：Quick BI 企业业务知识

<https://help.aliyun.com/zh/quick-bi/user-guide/manage-knowledge>

目的：理解 **怎样把企业业务术语、同义词、指标解释提供给 AI**。

---

### 第四份：DataLeap 指标平台最佳实践

<https://docs.volcengine.com/docs/6260/1337881?lang=zh>

目的：理解 **指标从定义、建模、字典到应用的完整生命周期**。

---

### 第五份：FineBI 指标中心

<https://help.fanruan.com/finebi/edition-view-28533-0.html>

目的：理解 **指标中心如何真正成为 ChatBI 的语义层**。

---

### 第六份：Aloudata NL2SQL vs NL2Semantic2SQL

<https://aloudata.com/blogs/nl2sql-vs-nl2semantic2sql-technical-path>

目的：理解 **传统 NL2SQL 与语义层智能问数的架构区别**。

---

# 18. 最终建议

如果目标是为企业内部智能问数项目建设“指标口径 Wiki + Semantic Layer”，不建议只模仿一个产品。

更合理的组合是：

```text
Dataphin
负责参考：指标定义规范

        +

DataLeap / DataArts
负责参考：指标治理、维度、模型与生命周期

        +

Quick BI / DataWorks ChatBI
负责参考：业务知识、同义词、问数模板

        +

SuperSonic / FineBI
负责参考：Semantic Layer 与 ChatBI 架构

        +

Aloudata
负责参考：NL2Semantic2SQL / MQL 技术路线
```

最终企业内部可以形成：

```text
企业指标 Wiki
      ↓
业务术语 + 指标口径 + 维度 + 数据模型 + 权限 + Few-shot
      ↓
Semantic Layer
      ↓
Semantic Query / MQL / Query IR
      ↓
SQL
      ↓
企业数据库
```

这套结构比单纯维护“指标名称 + SQL + 备注”的 Excel 更适合智能问数，因为它同时满足：

1. **人能看懂**；
2. **AI 能理解**；
3. **系统能执行**；
4. **指标能治理**；
5. **结果能追溯**。

---

# 19. 网页链接汇总

1. SuperSonic Wiki  
   <https://github.com/tencentmusic/supersonic/wiki>

2. SuperSonic GitHub  
   <https://github.com/tencentmusic/supersonic>

3. Dataphin：明确统计指标  
   <https://help.aliyun.com/zh/dataphin/fullmanaged/getting-started/the-statistical-index>

4. Dataphin：创建原子指标  
   <https://help.aliyun.com/zh/dataphin/fullmanaged/user-guide/creating-atomic-metrics>

5. Dataphin：创建派生指标和衍生指标  
   <https://help.aliyun.com/zh/dataphin/fullmanaged/user-guide/adding-derived-and-derived-metrics>

6. Quick BI：企业业务知识  
   <https://help.aliyun.com/zh/quick-bi/user-guide/manage-knowledge>

7. Quick BI：数据准备 / 问数配置  
   <https://help.aliyun.com/zh/quick-bi/user-guide/prepare-data>

8. DataWorks ChatBI：创建知识库提升准确率  
   <https://help.aliyun.com/zh/dataworks/user-guide/chatbi-knowledge-base>

9. DataLeap：指标平台使用流程最佳实践  
   <https://docs.volcengine.com/docs/6260/1337881?lang=zh>

10. DataLeap：指标 / 维度管理  
    <https://www.volcengine.com/docs/6260/1261858>

11. 华为 DataArts Studio：新建衍生指标  
    <https://support.huaweicloud.com/usermanual-dataartsstudio/dataartsstudio_01_0617.html>

12. 华为 DataArts Studio：业务指标  
    <https://support.huaweicloud.com/usermanual-dataartsstudio/dataartsstudio_01_0633.html>

13. FineBI：指标中心简介  
    <https://help.fanruan.com/finebi/edition-view-28533-0.html>

14. FineBI 7：指标中心 / FineChatBI 语义层  
    <https://help.fanruan.com/finebi/edition-view-29062-0.html>

15. Aloudata：NL2SQL vs NL2Semantic2SQL  
    <https://aloudata.com/blogs/nl2sql-vs-nl2semantic2sql-technical-path>

16. Aloudata：NL2MQL2SQL  
    <https://aloudata.com/resources/glossary/nl2mql2sql>

17. Aloudata：NoETL 指标平台  
    <https://aloudata.com/blogs/noetl-metrics-platform-data-governance>

18. 亿问 Data Agent：技术词典  
    <https://yiwendata.com/resources/glossary>

19. 灵犀 BI GitHub  
    <https://github.com/bonfirer/ai-report>

20. 灵犀 BI 开源介绍  
    <https://github.com/ruanyf/weekly/issues/10557>

21. DataEase SQLBot GitHub  
    <https://github.com/dataease/SQLBot>

22. SQLBot 官网  
    <https://sqlbot.org/>

---

## 20. 一句话总结

> 国内智能问数语义层最值得研究的公开资料组合是：**SuperSonic 看架构、Dataphin 看指标口径、Quick BI 看 AI 业务知识、DataLeap/DataArts 看指标治理、FineBI 看语义层落地、Aloudata 看 NL2Semantic2SQL 技术路线。**
