# SuperSonic 自然语言查询链路说明

本文以本项目当前源码为准，说明一条自然语言查询是如何经过以下阶段，最终变成数据库可以执行的物理 SQL：

```text
自然语言
  -> 语义元素识别
  -> 语义查询解析
  -> SemanticParseInfo
  -> S2SQL
  -> OntologyQuery
  -> 物理 SQL
  -> JDBC 执行
```

本文中的示例问题是：

```text
查询最近 7 天各地区的销售额
```

文中的 Java 代码均来自当前项目对应类的关键实现，省略了 import、日志和与主流程无关的分支；行号用于帮助回到源码定位，不代表文档中的代码可以直接替换原文件。

需要先区分两个概念：

| 名称 | 含义 |
| --- | --- |
| 语义元素 | 语义 Schema 中定义的指标、维度、维度值、术语和数据集等对象 |
| S2SQL | 使用语义字段表达查询意图的中间 SQL |
| 物理 SQL | 已经替换为真实表、真实字段、真实函数和真实 JOIN 的 SQL |

S2SQL 不是数据库中的最终 SQL。它更像编译器的中间表示，物理 SQL 才会交给数据库执行。

---

## 1. 全链路概览

```text
ChatQueryController
        |
        v
ChatQueryServiceImpl
        |
        v
NL2SQLParser
        |
        +--> 规则候选解析
        |       |
        |       +--> QueryNLReq
        |       +--> SemanticSchema
        |       +--> SchemaMapper
        |       +--> SchemaElementMatch
        |       +--> RuleSemanticQuery
        |       +--> SemanticParseInfo
        |       +--> S2SQL
        |
        +--> LLM 解析（可选）
                |
                +--> Schema、历史上下文、Exemplar
                +--> LLMRequestService
                +--> LLMSqlParser
                +--> LLMResponseService
                +--> S2SQL
        |
        v
ChatWorkflowEngine
        |
        +--> MAPPING
        +--> PARSING
        +--> S2SQL_CORRECTING
        +--> TRANSLATING
        +--> PHYSICAL_SQL_CORRECTING
        |
        v
S2SemanticLayerService
        |
        v
DefaultSemanticTranslator
        |
        +--> SqlQueryParser
        +--> StructQueryParser
        +--> 维度/指标表达式解析器
        +--> OntologyQueryParser
        |
        v
SqlBuilder + Calcite
        |
        v
物理 SQL
        |
        v
JdbcExecutor
        |
        v
数据库
```

代码中的主要对象关系是：

```text
ChatQueryContext
  |- QueryNLReq       用户问题和请求参数
  |- SemanticSchema   当前数据集的语义 Schema
  |- SchemaMapInfo    语义元素匹配结果
  |- candidateQueries SemanticQuery 候选
  `- ParseResp        对外解析结果

SemanticQuery
  `- SemanticParseInfo
       |- metrics
       |- dimensions
       |- dimensionFilters
       |- dateInfo
       |- aggType
       `- sqlInfo
            |- parsedS2SQL
            |- correctedS2SQL
            `- querySQL
```

---

## 2. 请求入口：从 HTTP 到 NL2SQLParser

### 2.1 `ChatQueryController`

文件：

`chat/server/src/main/java/com/tencent/supersonic/chat/server/rest/ChatQueryController.java`

关键方法：

| 行号 | 方法 | 作用 |
| --- | --- | --- |
| 41 | `parse()` | 只解析自然语言，返回候选解析结果 |
| 48 | `execute()` | 执行已经选定的解析结果 |
| 63 | `query()` | 先解析，再执行 |

简化后的入口代码如下：

```java
public Object parse(@RequestBody ChatParseReq chatParseReq, ...) {
    return chatQueryService.parse(chatParseReq);
}

public Object query(@RequestBody ChatParseReq chatParseReq, ...) {
    ChatParseResp parseResp = chatQueryService.parse(chatParseReq);
    // 从 parseResp 选择一个 SemanticParseInfo 后执行
    return chatQueryService.execute(executeReq);
}
```

Controller 只负责 HTTP 层适配，不负责识别“销售额”或生成 SQL。

### 2.2 `ChatQueryServiceImpl`

文件：

`chat/server/src/main/java/com/tencent/supersonic/chat/server/service/impl/ChatQueryServiceImpl.java`

关键方法：

```text
parse()          110
execute()        141
parseAndExecute() 182
buildParseContext() 201
buildExecuteContext() 208
```

`parse()` 的职责是创建 `ParseContext`，然后遍历 Chat 层注册的 `ChatQueryParser`。其中负责自然语言转 SQL 的主要解析器就是 `NL2SQLParser`。

### 2.3 `QueryReqConverter`

文件：

`chat/server/src/main/java/com/tencent/supersonic/chat/server/util/QueryReqConverter.java`

关键方法：

```java
public static QueryNLReq buildQueryNLReq(ParseContext parseContext)
```

该方法把 Chat 请求转换成 Headless 层使用的 `QueryNLReq`，携带：

- 用户问题文本；
- 数据集 ID；
- 当前用户；
- 历史对话和上下文解析结果；
- 查询过滤器；
- 是否启用 LLM；
- `Text2SQLType`。

---

## 3. `NL2SQLParser`：规则和 LLM 的总调度

文件：

`chat/server/src/main/java/com/tencent/supersonic/chat/server/parser/NL2SQLParser.java`

关键方法：

```text
accept() 77
parse()  82
doParse() 158
```

### 3.1 先做规则候选解析

源码核心逻辑如下：

```java
public void parse(ParseContext parseContext) {
    if (Objects.isNull(parseContext.getRequest().getSelectedParse())) {
        QueryNLReq queryNLReq = QueryReqConverter.buildQueryNLReq(parseContext);
        queryNLReq.setText2SQLType(Text2SQLType.ONLY_RULE);

        if (parseContext.enableLLM()) {
            queryNLReq.setText2SQLType(Text2SQLType.NONE);
        }

        for (Long datasetId : queryNLReq.getDataSetIds()) {
            queryNLReq.setDataSetIds(Collections.singleton(datasetId));

            for (MapModeEnum mode :
                    Lists.newArrayList(MapModeEnum.STRICT, MapModeEnum.MODERATE)) {
                queryNLReq.setMapModeEnum(mode);
                doParse(queryNLReq, parseResp);
            }

            if (parseResp.getSelectedParses().isEmpty()) {
                queryNLReq.setMapModeEnum(MapModeEnum.LOOSE);
                doParse(queryNLReq, parseResp);
            }
        }
    }
}
```

实际代码位置：`NL2SQLParser.java:82-127`。

三个映射模式的含义：

| 模式 | 说明 |
| --- | --- |
| `STRICT` | 主要接受高置信度的精确匹配 |
| `MODERATE` | 接受一定程度的模糊匹配 |
| `LOOSE` | 使用更宽松的匹配，通常作为兜底 |

每个数据集会产生候选解析，系统按照 `SemanticParseInfo.sort()` 进行排序，再保留配置数量的候选结果。这样做的原因是一个词可能同时对应多个数据集、指标或维度，系统需要先保留候选，而不是过早做不可逆的唯一选择。

### 3.2 再根据候选进入 LLM

源码关键逻辑位于 `NL2SQLParser.java:130-154`：

```java
if (parseContext.needLLMParse() && !parseContext.needFeedback()) {
    QueryNLReq queryNLReq = QueryReqConverter.buildQueryNLReq(parseContext);
    queryNLReq.setText2SQLType(Text2SQLType.LLM_OR_RULE);

    SemanticParseInfo selectedParse = parseContext.getRequest().getSelectedParse();
    queryNLReq.setSelectedParseInfo(
            Objects.nonNull(selectedParse)
                    ? selectedParse
                    : parseContext.getResponse().getSelectedParses().get(0));

    rewriteMultiTurn(parseContext, queryNLReq);
    addDynamicExemplars(parseContext, queryNLReq);
    doParse(queryNLReq, parseContext.getResponse());
}
```

LLM 分支不是完全脱离语义 Schema 的自由生成。它使用：

```text
用户问题
+ 当前数据集 Schema
+ 候选指标和维度
+ 历史问题和历史 SQL
+ Text2SQL exemplar
+ 术语和模型信息
```

如果第一次生成失败，代码还会将 `MapModeEnum.ALL` 作为兜底重试模式。

---

## 4. 准备语义 Schema 和工作流上下文

### 4.1 `S2ChatLayerService`

文件：

`headless/server/src/main/java/com/tencent/supersonic/headless/server/facade/service/impl/S2ChatLayerService.java`

关键方法：

```text
map()                   46
parse()                 67
buildChatQueryContext() 95
```

核心代码：

```java
private ChatQueryContext buildChatQueryContext(QueryNLReq queryNLReq) {
    ChatQueryContext queryCtx = new ChatQueryContext(queryNLReq);
    SemanticSchema semanticSchema =
            schemaService.getSemanticSchema(queryNLReq.getDataSetIds());
    Map<Long, List<Long>> modelIdToDataSetIds =
            dataSetService.getModelIdToDataSetIds();

    queryCtx.setSemanticSchema(semanticSchema);
    queryCtx.setModelIdToDataSetIds(modelIdToDataSetIds);
    return queryCtx;
}
```

`SemanticSchema` 中保存了当前查询可能用到的：

- 数据集；
- 指标；
- 维度；
- 维度值；
- 指标表达式；
- 分区时间维度；
- 数据模型；
- 模型之间的 JOIN 关系；
- 查询配置。

解析接口的代码如下：

```java
public ParseResp parse(QueryNLReq queryNLReq) {
    ParseResp parseResp = new ParseResp(queryNLReq.getQueryText());
    ChatQueryContext queryCtx = buildChatQueryContext(queryNLReq);
    queryCtx.setParseResp(parseResp);

    if (queryCtx.getMapInfo().isEmpty()) {
        chatWorkflowEngine.start(ChatWorkflowState.MAPPING, queryCtx);
    } else {
        chatWorkflowEngine.start(ChatWorkflowState.PARSING, queryCtx);
    }
    return parseResp;
}
```

实际位置：`S2ChatLayerService.java:67-76`。

### 4.2 `ChatWorkflowEngine`

文件：

`headless/server/src/main/java/com/tencent/supersonic/headless/server/utils/ChatWorkflowEngine.java`

工作流状态：

```text
MAPPING
  -> PARSING
  -> S2SQL_CORRECTING
  -> TRANSLATING
  -> PHYSICAL_SQL_CORRECTING
  -> FINISHED
```

主循环源码：

```java
public void start(ChatWorkflowState initialState, ChatQueryContext queryCtx) {
    ParseResp parseResult = queryCtx.getParseResp();
    queryCtx.setChatWorkflowState(initialState);

    while (queryCtx.getChatWorkflowState() != ChatWorkflowState.FINISHED) {
        switch (queryCtx.getChatWorkflowState()) {
            case MAPPING:
                performMapping(queryCtx);
                queryCtx.setChatWorkflowState(
                        queryCtx.getMapInfo().isEmpty()
                                ? ChatWorkflowState.FINISHED
                                : ChatWorkflowState.PARSING);
                break;
            case PARSING:
                performParsing(queryCtx);
                queryCtx.setChatWorkflowState(
                        queryCtx.getCandidateQueries().isEmpty()
                                ? ChatWorkflowState.FINISHED
                                : ChatWorkflowState.S2SQL_CORRECTING);
                break;
            case S2SQL_CORRECTING:
                performCorrecting(queryCtx);
                queryCtx.setChatWorkflowState(ChatWorkflowState.TRANSLATING);
                break;
            case TRANSLATING:
                performTranslating(queryCtx, parseResult);
                queryCtx.setChatWorkflowState(
                        ChatWorkflowState.PHYSICAL_SQL_CORRECTING);
                break;
            case PHYSICAL_SQL_CORRECTING:
                performPhysicalSqlCorrecting(queryCtx);
                queryCtx.setChatWorkflowState(ChatWorkflowState.FINISHED);
                break;
        }
    }
}
```

实际完整逻辑见：`ChatWorkflowEngine.java:38-94`。

其中三个调度方法最重要：

```java
private void performMapping(ChatQueryContext queryCtx) {
    schemaMappers.forEach(mapper -> mapper.map(queryCtx));
}

private void performParsing(ChatQueryContext queryCtx) {
    semanticParsers.forEach(parser -> parser.parse(queryCtx));
}

private void performCorrecting(ChatQueryContext queryCtx) {
    for (SemanticQuery semanticQuery : queryCtx.getCandidateQueries()) {
        for (SemanticCorrector corrector : semanticCorrectors) {
            corrector.correct(queryCtx, semanticQuery.getParseInfo());
        }
    }
}
```

实际位置：`ChatWorkflowEngine.java:96-124`。

---

## 5. 语义元素识别：SchemaMapper

SchemaMapper 的注册顺序在：

`launchers/headless/src/main/resources/META-INF/spring.factories`

```properties
com.tencent.supersonic.headless.chat.mapper.SchemaMapper=\
    com.tencent.supersonic.headless.chat.mapper.EmbeddingMapper, \
    com.tencent.supersonic.headless.chat.mapper.KeywordMapper, \
    com.tencent.supersonic.headless.chat.mapper.QueryFilterMapper, \
    com.tencent.supersonic.headless.chat.mapper.PartitionTimeMapper,\
    com.tencent.supersonic.headless.chat.mapper.TermDescMapper
```

### 5.1 `BaseMapper`：统一映射框架

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/BaseMapper.java`

模板方法：

```java
public void map(ChatQueryContext chatQueryContext) {
    if (!accept(chatQueryContext)) {
        return;
    }

    try {
        doMap(chatQueryContext);
        MapFilter.filter(chatQueryContext);
    } catch (Exception e) {
        log.error("work error", e);
    }
}
```

实际位置：`BaseMapper.java:28-49`。

子类只需要实现 `doMap()`，公共逻辑负责：

1. 判断当前 Mapper 是否适用；
2. 执行具体匹配；
3. 过滤无效结果；
4. 将结果写入 `SchemaMapInfo`。

去重逻辑位于 `BaseMapper.java:57-80`：

```java
public void addToSchemaMap(SchemaMapInfo schemaMap, Long dataSetId,
        SchemaElementMatch newElementMatch) {
    List<SchemaElementMatch> matches = schemaMap
            .getDataSetElementMatches()
            .computeIfAbsent(dataSetId, k -> new ArrayList<>());

    AtomicBoolean shouldAddNew = new AtomicBoolean(true);
    matches.removeIf(existing -> {
        if (isEquals(existing, newElementMatch)) {
            if (newElementMatch.getSimilarity() > existing.getSimilarity()) {
                return true;
            }
            shouldAddNew.set(false);
        }
        return false;
    });

    if (shouldAddNew.get()) {
        matches.add(newElementMatch);
    }
}
```

同一个语义元素被多个策略命中时，只保留相似度更高的结果。对于 `VALUE`，还会比较具体的值文本。

### 5.2 `KeywordMapper`：词典和关键词匹配

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/KeywordMapper.java`

关键方法：

```text
doMap()                       34
convertMapResultToMapInfo()   53、129
doDimValueAliasLogic()        97
```

核心源码：

```java
public void doMap(ChatQueryContext chatQueryContext) {
    String queryText = chatQueryContext.getRequest().getQueryText();

    List<S2Term> terms = HanlpHelper.getTerms(
            queryText, chatQueryContext.getModelIdToDataSetIds());

    HanlpDictMatchStrategy hanlpMatchStrategy =
            ContextUtils.getBean(HanlpDictMatchStrategy.class);
    List<HanlpMapResult> hanlpMatchResults =
            getMatches(chatQueryContext, hanlpMatchStrategy);
    convertMapResultToMapInfo(hanlpMatchResults, chatQueryContext, terms);

    DatabaseMatchStrategy databaseMatchStrategy =
            ContextUtils.getBean(DatabaseMatchStrategy.class);
    List<DatabaseMapResult> databaseMatchResults =
            getMatches(chatQueryContext, databaseMatchStrategy);
    convertMapResultToMapInfo(chatQueryContext, databaseMatchResults);
}
```

实际位置：`KeywordMapper.java:34-50`。

它组合两种策略：

```text
HanLP 分词 + 词典匹配
数据库中的名称/别名匹配
```

将匹配结果转为 `SchemaElementMatch` 时，会根据词性编码解析：

```java
Long dataSetId = NatureHelper.getDataSetId(nature);
SchemaElementType elementType = NatureHelper.convertToElementType(nature);
Long elementID = NatureHelper.getElementID(nature);

SchemaElement element = getSchemaElement(
        dataSetId, elementType, elementID,
        chatQueryContext.getSemanticSchema());

SchemaElementMatch match = SchemaElementMatch.builder()
        .element(element)
        .word(hanlpMapResult.getName())
        .similarity(hanlpMapResult.getSimilarity())
        .detectWord(hanlpMapResult.getDetectWord())
        .build();

addToSchemaMap(chatQueryContext.getMapInfo(), dataSetId, match);
```

实际位置：`KeywordMapper.java:64-93`。

例如：

```text
“销售额” -> SchemaElementType.METRIC
“地区”   -> SchemaElementType.DIMENSION
“华东”   -> SchemaElementType.VALUE
```

维度值别名会在 `doDimValueAliasLogic()` 中替换为技术值。例如用户说“华东”，数据库中的技术值可能是 `east_region`。

### 5.3 `EmbeddingMapper`：向量匹配

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/EmbeddingMapper.java`

只有在 `LOOSE` 或 `LLM_OR_RULE` 模式下使用：

```java
public boolean accept(ChatQueryContext chatQueryContext) {
    boolean loose = MapModeEnum.LOOSE.equals(
            chatQueryContext.getRequest().getMapModeEnum());
    boolean llm = chatQueryContext.getRequest().getText2SQLType()
            == Text2SQLType.LLM_OR_RULE;
    return loose || llm;
}
```

实际位置：`EmbeddingMapper.java:29-33`。

通过 `EmbeddingMatchStrategy` 查询向量库，然后从 metadata 中取得：

```text
dataSetId
elementType
elementId
```

再构造 `SchemaElementMatch`。主要代码位于 `EmbeddingMapper.java:35-75`。

### 5.4 `QueryFilterMapper`：处理显式过滤条件

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/QueryFilterMapper.java`

如果前端已经明确传入维度和值，例如：

```json
{
  "dimensionId": 2000,
  "value": "华东"
}
```

该 Mapper 会直接创建一个 `VALUE` 类型的 `SchemaElementMatch`，不需要再次从自然语言猜测。

### 5.5 `PartitionTimeMapper`：补充分区时间维度

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/PartitionTimeMapper.java`

它把数据集的分区时间字段加入语义映射。例如：

```text
partition_date -> 日期维度
```

后面的 `TimeRangeParser` 才能把“最近 7 天”绑定到真实的日期字段。

### 5.6 `TermDescMapper`：从术语描述递归识别

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/TermDescMapper.java`

当用户说的是业务术语而不是 Schema 中的直接名称时，该 Mapper 会读取术语描述，再递归调用 SchemaMapper，尝试发现描述中的指标、维度和维度值。

---

## 6. `SchemaElementMatch` 和 `SemanticParseInfo`

### 6.1 `SchemaElementMatch`：识别结果

一个匹配结果可以抽象为：

```text
SchemaElementMatch
  |- detectWord  原句中命中的文本
  |- word        归一化后的值或名称
  |- element     被命中的 SchemaElement
  |- similarity  相似度
  |- frequency   词频
  `- llmMatched  是否由 LLM/向量匹配
```

它只说明“文本命中了哪个语义对象”，还没有决定这个对象是分组字段、聚合指标还是过滤值。

### 6.2 `SemanticParseInfo`：统一中间模型

文件：

`headless/api/src/main/java/com/tencent/supersonic/headless/api/pojo/SemanticParseInfo.java`

核心字段位于 `SemanticParseInfo.java:27-51`：

```java
private SchemaElement dataSet;
private Set<SchemaElement> metrics;
private Set<SchemaElement> dimensions;
private Set<QueryFilter> dimensionFilters;
private Set<QueryFilter> metricFilters;
private AggregateTypeEnum aggType;
private Set<Order> orders;
private long limit;
private double score;
private List<SchemaElementMatch> elementMatches;
private DateConf dateInfo;
private SqlInfo sqlInfo;
private QueryType queryType;
```

自然语言信息落位如下：

| 识别出的信息 | 进入的字段 |
| --- | --- |
| 数据集 | `dataSet` |
| 指标 | `metrics` |
| 维度 | `dimensions` |
| 维度值 | `dimensionFilters` |
| 指标过滤 | `metricFilters` |
| 时间范围 | `dateInfo` |
| 聚合类型 | `aggType` |
| 排序 | `orders` |
| 条数 | `limit` |
| 原始匹配结果 | `elementMatches` |
| 规则生成的 S2SQL | `sqlInfo.parsedS2SQL` |
| 修正后的 S2SQL | `sqlInfo.correctedS2SQL` |
| 翻译后的物理 SQL | `sqlInfo.querySQL` |

### 6.3 指标、维度和值如何落位

这段逻辑在：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/query/rule/RuleSemanticQuery.java:112-142`

```java
for (SchemaElementMatch schemaMatch : parseInfo.getElementMatches()) {
    SchemaElement element = schemaMatch.getElement();

    switch (element.getType()) {
        case VALUE:
            // 先按维度 ID 收集值，后面转换成过滤器
            dim2Values.computeIfAbsent(element.getId(), k -> new ArrayList<>())
                    .add(schemaMatch);
            break;
        case DIMENSION:
            parseInfo.getDimensions().add(element);
            break;
        case METRIC:
            parseInfo.getMetrics().add(element);
            break;
        default:
            break;
    }
}

addToFilters(dim2Values, parseInfo, semanticSchema,
        SchemaElementType.DIMENSION);
```

这里有一个容易误解的地方：

```text
DIMENSION -> dimensions
VALUE     -> dimensionFilters
```

维度值不会直接变成 `dimensions`。系统先用维度值对应的维度 ID 找到真实维度，再构造 `QueryFilter`。

单个值生成 `EQUALS`：

```java
dimensionFilter.setValue(schemaMatch.getWord());
dimensionFilter.setBizName(dimension.getBizName());
dimensionFilter.setOperator(FilterOperatorEnum.EQUALS);
parseInfo.getDimensionFilters().add(dimensionFilter);
```

多个值生成 `IN`：

```java
List<String> values = new ArrayList<>();
entry.getValue().forEach(i -> values.add(i.getWord()));
dimensionFilter.setValue(values);
dimensionFilter.setBizName(dimension.getBizName());
dimensionFilter.setOperator(FilterOperatorEnum.IN);
parseInfo.getDimensionFilters().add(dimensionFilter);
```

代码位置：`RuleSemanticQuery.java:144-175`。

---

## 7. 规则语义解析

### 7.1 `RuleSqlParser`

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/rule/RuleSqlParser.java`

关键源码：

```java
public void parse(ChatQueryContext chatQueryContext) {
    if (!chatQueryContext.getCandidateQueries().isEmpty()) {
        return;
    }

    SchemaMapInfo mapInfo = chatQueryContext.getMapInfo();
    List<SemanticQuery> candidateQueries = Lists.newArrayList();

    for (Long dataSetId : mapInfo.getMatchedDataSetInfos()) {
        List<SchemaElementMatch> elementMatches =
                mapInfo.getMatchedElements(dataSetId);
        List<RuleSemanticQuery> queries = RuleSemanticQuery.resolve(
                dataSetId, elementMatches, chatQueryContext);
        candidateQueries.addAll(queries);
    }

    chatQueryContext.setCandidateQueries(candidateQueries);

    auxiliaryParsers.forEach(p -> p.parse(chatQueryContext));

    candidateQueries.forEach(query -> query.buildS2Sql(
            chatQueryContext.getDataSetSchema(
                    query.getParseInfo().getDataSetId())));
}
```

实际位置：`RuleSqlParser.java:26-45`。

这段代码做了四件事：

1. 按数据集读取匹配结果；
2. 根据指标、维度和值的组合生成查询类型；
3. 执行时间和聚合辅助解析；
4. 将候选语义查询转换为 S2SQL。

### 7.2 `RuleSemanticQuery.resolve()`

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/query/rule/RuleSemanticQuery.java`

```java
public static List<RuleSemanticQuery> resolve(
        Long dataSetId,
        List<SchemaElementMatch> candidateElementMatches,
        ChatQueryContext chatQueryContext) {

    List<RuleSemanticQuery> matchedQueries = new ArrayList<>();

    for (RuleSemanticQuery semanticQuery : QueryManager.getRuleQueries()) {
        List<SchemaElementMatch> matches =
                semanticQuery.match(candidateElementMatches, chatQueryContext);

        if (!matches.isEmpty()) {
            RuleSemanticQuery query =
                    QueryManager.createRuleQuery(semanticQuery.getQueryMode());
            query.getParseInfo().getElementMatches().addAll(matches);
            query.fillParseInfo(chatQueryContext, dataSetId);
            matchedQueries.add(query);
        }
    }
    return matchedQueries;
}
```

实际位置：`RuleSemanticQuery.java:195-210`。

`QueryManager` 中注册的规则查询包括：

```text
MetricSemanticQuery
MetricModelQuery
MetricGroupByQuery
MetricFilterQuery
MetricTopNQuery
DetailSemanticQuery
DetailDimensionQuery
DetailValueQuery
```

它们通过 `QueryMatcher` 判断匹配结果是否符合某种查询模式。例如：

```text
指标查询       至少需要一个 METRIC
分组查询       需要 METRIC，可选 DIMENSION
TopN 查询      需要 METRIC 和 DIMENSION
明细查询       需要 DIMENSION 或 VALUE
```

### 7.3 `RuleSemanticQuery.buildS2Sql()`

```java
public void buildS2Sql(DataSetSchema dataSetSchema) {
    QueryStructReq queryStructReq = convertQueryStruct();
    convertBizNameToName(dataSetSchema, queryStructReq);
    QuerySqlReq querySQLReq = queryStructReq.convert();

    parseInfo.getSqlInfo().setParsedS2SQL(querySQLReq.getSql());
    parseInfo.getSqlInfo().setCorrectedS2SQL(querySQLReq.getSql());
}
```

实际位置：`RuleSemanticQuery.java:42-49`。

这个方法先把 `SemanticParseInfo` 转成结构化查询，再调用 `QueryStructReq.convert()` 生成 S2SQL。

---

## 8. 时间、聚合和查询类型解析

### 8.1 时间：`TimeRangeParser`

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/rule/TimeRangeParser.java`

关键方法：

```text
parse()          40
parseDateCN()    72
parseDateNumber() 93
parseRecent()    116
updateQueryContext() 58
```

支持的表达式包括：

```text
最近 7 天、近一个月、上周、今年、昨天、20260101、2026-01-01
```

处理结果是 `DateConf`：

```text
DateConf
  |- startDate
  |- endDate
  |- dateField
  |- dateMode
  `- detectWord
```

关键过程：

```java
DateConf dateConf = parseRecent(queryText);
if (dateConf == null) {
    dateConf = parseDateNumber(queryText);
}
if (dateConf == null) {
    dateConf = parseDateCN(queryText);
}
if (dateConf != null) {
    updateQueryContext(queryContext, dateConf);
}
```

然后从数据集拿分区维度：

```java
SchemaElement partitionDimension = dataSetSchema.getPartitionDimension();
dateConf.setDateField(partitionDimension.getName());
parseInfo.setDateInfo(dateConf);
```

因此“最近 7 天”并不会直接生成 SQL 字符串，而是先变成结构化的 `DateConf`，后面再由 SQL 构造器转换为日期过滤条件。

### 8.2 聚合：`AggregateTypeParser`

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/rule/AggregateTypeParser.java`

`resolveAggregateType()` 根据关键词识别：

```text
总和、SUM       -> SUM
平均、AVG       -> AVG
最大、MAX       -> MAX
最小、MIN       -> MIN
Top             -> TOPN
用户数、UV      -> DISTINCT 或 COUNT DISTINCT
明细            -> NONE
```

结果写入：

```java
parseInfo.setAggType(aggregateType);
```

如果用户没有明确说聚合方式，则使用指标的 `defaultAgg`。

### 8.3 查询类型：`QueryTypeParser`

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/QueryTypeParser.java`

通过 `SqlSelectFunctionHelper.hasAggregateFunction(s2SQL)` 判断：

```text
包含 SUM、AVG、COUNT 等聚合函数 -> QueryType.AGGREGATE
否则                         -> QueryType.DETAIL
```

---

## 9. 从语义结构生成 S2SQL

### 9.1 `QueryReqBuilder.buildStructReq()`

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/utils/QueryReqBuilder.java`

关键源码：

```java
public static QueryStructReq buildStructReq(SemanticParseInfo parseInfo) {
    QueryStructReq queryStructReq = new QueryStructReq();
    queryStructReq.setDataSetId(parseInfo.getDataSetId());
    queryStructReq.setDataSetName(parseInfo.getDataSet().getName());
    queryStructReq.setQueryType(parseInfo.getQueryType());
    queryStructReq.setDateInfo(parseInfo.getDateInfo());

    queryStructReq.setDimensionFilters(
            getFilters(parseInfo.getDimensionFilters()));
    queryStructReq.setMetricFilters(
            parseInfo.getMetricFilters().stream()
                    .map(f -> new Filter(f.getBizName(), f.getOperator(), f.getValue()))
                    .collect(Collectors.toList()));

    queryStructReq.setGroups(parseInfo.getDimensions().stream()
            .map(SchemaElement::getBizName)
            .collect(Collectors.toList()));
    queryStructReq.setLimit(parseInfo.getLimit());

    for (SchemaElement metric : parseInfo.getMetrics()) {
        queryStructReq.getAggregators().addAll(
                getAggregatorByMetric(parseInfo.getAggType(), metric));
        queryStructReq.setOrders(new ArrayList<>(
                getOrder(parseInfo.getOrders(), parseInfo.getAggType(), metric)));
    }
    return queryStructReq;
}
```

实际位置：`QueryReqBuilder.java:36-64`。

字段映射关系（这是结构化请求内部的第一步映射）：

```text
parseInfo.metrics           -> QueryStructReq.aggregators
parseInfo.dimensions        -> QueryStructReq.groups
parseInfo.dimensionFilters  -> QueryStructReq.dimensionFilters
parseInfo.metricFilters     -> QueryStructReq.metricFilters
parseInfo.dateInfo           -> QueryStructReq.dateInfo
parseInfo.orders             -> QueryStructReq.orders
parseInfo.limit              -> QueryStructReq.limit
```

这里的 `groups` 和 `aggregators` 先使用 `SchemaElement.bizName`，因为它们是语义层稳定的内部字段名。随后 `RuleSemanticQuery.buildS2Sql()` 会调用 `convertBizNameToName()`，把这些字段转换为数据集 Schema 中的语义名称，再由 `QueryStructReq.convert()` 生成 S2SQL。

聚合操作由 `getAggregatorByMetric()` 决定：

```java
private static List<Aggregator> getAggregatorByMetric(
        AggregateTypeEnum aggregateType, SchemaElement metric) {
    String agg = determineAggregator(aggregateType, metric);
    return Collections.singletonList(
            new Aggregator(metric.getBizName(), AggOperatorEnum.of(agg)));
}
```

如果 `aggType` 是 `NONE`，或者指标是 `COUNT_DISTINCT` 类型，通常使用指标自身的默认聚合。

### 9.2 `QueryStructReq.convert()`

文件：

`headless/api/src/main/java/com/tencent/supersonic/headless/api/pojo/request/QueryStructReq.java`

关键源码：

```java
public QuerySqlReq convert(boolean isBizName) {
    String sql = buildSql(this, isBizName);

    QuerySqlReq result = new QuerySqlReq();
    result.setSql(sql);
    result.setDataSetId(this.getDataSetId());
    result.setModelIds(this.getModelIdSet());
    result.getSqlInfo().setCorrectedS2SQL(sql);
    return result;
}
```

实际位置：`QueryStructReq.java:140-154`。

`buildSql()` 的顺序是：

```java
plainSelect.setSelectItems(buildSelectItems(queryStructReq));
plainSelect.setFromItem(new Table(queryStructReq.getTableName()));
plainSelect.setOrderByElements(buildOrderByElements(queryStructReq));
plainSelect.setGroupByElement(buildGroupByElement(queryStructReq));
plainSelect.setLimit(buildLimit(queryStructReq));
return addWhereClauses(select.toString(), queryStructReq, isBizName);
```

实际位置：`QueryStructReq.java:157-186`。

选择字段的顺序是：

```text
GROUP BY 字段
聚合指标
```

聚合字段由 `buildAggregatorSelectItem()` 生成，例如：

```text
Aggregator(column = sale_amount, func = SUM)
    -> SUM(sale_amount)
```

维度过滤和日期过滤在 `addWhereClauses()` 中追加：

```java
String whereClause = sqlFilterUtils.getWhereClause(
        queryStructReq.getDimensionFilters(), isBizName);

String dateWhereStr = dateModeUtils.getDateWhereStr(
        queryStructReq.getDateInfo());
```

数据集表名由 `getTableName()` 决定：

```java
if (dataSetId != null) {
    return Constants.TABLE_PREFIX + dataSetId;
}
```

`RuleSemanticQuery.buildS2Sql()` 会先把结构化请求中的 `bizName` 转回语义名称，然后生成 S2SQL。因此规则分支的 S2SQL 可能类似：

```sql
SELECT 地区, SUM(销售额)
FROM 销售数据集
WHERE 日期 BETWEEN '2026-08-01' AND '2026-08-07'
GROUP BY 地区
```

这里的 `地区`、`销售额` 和 `日期` 是语义名称，`销售数据集` 也只是语义层数据集表名，不是数据库中的真实物理表。后续 `SqlQueryParser` 会同时识别名称和 `bizName`，并把它们转换为统一的内部字段名。

---

## 10. LLM 分支如何生成 S2SQL

### 10.1 `LLMSqlParser`

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/llm/LLMSqlParser.java`

关键方法：

```text
parse()     26
tryParse()  48
```

核心流程：

```java
public void parse(ChatQueryContext queryCtx) {
    if (!queryCtx.getRequest().getText2SQLType().enableLLM()) {
        return;
    }

    LLMRequestService requestService =
            ContextUtils.getBean(LLMRequestService.class);
    Long dataSetId = requestService.getDataSetId(queryCtx);
    if (dataSetId == null) {
        return;
    }

    tryParse(queryCtx, dataSetId);
}
```

实际位置：`LLMSqlParser.java:26-45`。

`tryParse()` 会：

1. 调用 `LLMRequestService.getLlmReq()` 组织 Prompt；
2. 调用 `runText2SQL()`；
3. 对返回的 SQL 去重和校验；
4. 失败时重试；
5. 将每个有效 SQL 交给 `LLMResponseService.addParseInfo()`。

### 10.2 `LLMResponseService`

文件：

`headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/llm/LLMResponseService.java`

核心代码：

```java
public void addParseInfo(ChatQueryContext queryCtx,
        ParseResult parseResult, String s2SQL, Double weight) {
    LLMSemanticQuery semanticQuery =
            QueryManager.createLLMQuery(LLMSqlQuery.QUERY_MODE);
    SemanticParseInfo parseInfo = semanticQuery.getParseInfo();

    parseInfo.setDataSet(queryCtx.getSemanticSchema()
            .getDataSet(parseResult.getDataSetId()));
    parseInfo.setQueryConfig(queryCtx.getSemanticSchema()
            .getQueryConfig(parseResult.getDataSetId()));
    parseInfo.getElementMatches().addAll(
            queryCtx.getMapInfo().getMatchedElements(parseInfo.getDataSetId()));

    parseInfo.setQueryMode(semanticQuery.getQueryMode());
    parseInfo.getSqlInfo().setParsedS2SQL(s2SQL);
    parseInfo.getSqlInfo().setCorrectedS2SQL(s2SQL);

    queryCtx.getCandidateQueries().add(semanticQuery);
}
```

实际位置：`LLMResponseService.java:29-66`。

LLM 分支的特点是：LLM 直接返回 S2SQL，但结果仍然被包装为统一的 `SemanticQuery` 和 `SemanticParseInfo`，因此后续仍然走同一套语义层翻译流程。

返回结果会先通过：

```java
SqlValidHelper.equals(...)
SqlValidHelper.isValidSQL(...)
```

做去重和 SQL 有效性检查，相关代码位于 `LLMResponseService.java:68-87`。

---

## 11. S2SQL 转物理 SQL

### 11.1 `S2SemanticLayerService.translate()`

文件：

`headless/server/src/main/java/com/tencent/supersonic/headless/server/facade/service/impl/S2SemanticLayerService.java`

核心代码：

```java
public SemanticTranslateResp translate(
        SemanticQueryReq queryReq, User user) throws Exception {
    QueryStatement queryStatement = buildQueryStatement(queryReq, user);
    semanticTranslator.translate(queryStatement);

    return SemanticTranslateResp.builder()
            .querySQL(queryStatement.getSql())
            .isOk(queryStatement.isOk())
            .errMsg(queryStatement.getErrMsg())
            .build();
}
```

实际位置：`S2SemanticLayerService.java:91-98`。

对于 `QuerySqlReq`，`buildSqlQueryStatement()` 会：

```java
QueryStatement queryStatement = buildQueryStatement(querySqlReq);
queryStatement.setIsS2SQL(true);

SqlQuery sqlQuery = new SqlQuery();
sqlQuery.setSql(querySqlReq.getSql());
queryStatement.setSqlQuery(sqlQuery);
```

实际位置：`S2SemanticLayerService.java:324-343`。

同时将：

```text
SemanticSchema
Ontology
数据集 ID
数据库信息
```

放入 `QueryStatement`，供后续 Parser 使用。

### 11.2 `DefaultSemanticTranslator`

文件：

`headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/DefaultSemanticTranslator.java`

核心源码：

```java
public void translate(QueryStatement queryStatement) throws Exception {
    if (queryStatement.isTranslated()) {
        return;
    }

    for (QueryParser parser : ComponentFactory.getQueryParsers()) {
        if (parser.accept(queryStatement)) {
            parser.parse(queryStatement);
            if (!queryStatement.getStatus().equals(QueryState.SUCCESS)) {
                break;
            }
        }
    }

    mergeOntologyQuery(queryStatement);

    if (StringUtils.isNotBlank(
            queryStatement.getSqlQuery().getSimplifiedSql())) {
        queryStatement.setSql(
                queryStatement.getSqlQuery().getSimplifiedSql());
    }

    for (QueryOptimizer optimizer : ComponentFactory.getQueryOptimizers()) {
        if (optimizer.accept(queryStatement)) {
            optimizer.rewrite(queryStatement);
        }
    }
}
```

实际位置：`DefaultSemanticTranslator.java:27-56`。

Core Parser 的注册顺序位于：

`launchers/headless/src/main/resources/META-INF/spring.factories`

```properties
com.tencent.supersonic.headless.core.translator.parser.QueryParser=\
    ...SqlVariableParser,\
    ...StructQueryParser,\
    ...SqlQueryParser,\
    ...DefaultDimValueParser,\
    ...DimExpressionParser,\
    ...MetricExpressionParser,\
    ...MetricRatioParser,\
    ...OntologyQueryParser
```

### 11.3 `SqlQueryParser`：解析 S2SQL 并建立 OntologyQuery

文件：

`headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/parser/SqlQueryParser.java`

它只接受标记为 S2SQL 的 `QueryStatement`：

```java
public boolean accept(QueryStatement queryStatement) {
    return Objects.nonNull(queryStatement.getSqlQuery())
            && queryStatement.getIsS2SQL();
}
```

实际位置：`SqlQueryParser.java:39-42`。

`parse()` 的主要步骤：

```java
SqlQuery sqlQuery = queryStatement.getSqlQuery();
List<String> queryFields =
        SqlSelectHelper.getAllSelectFields(sqlQuery.getSql());

Ontology ontology = queryStatement.getOntology();
OntologyQuery ontologyQuery =
        buildOntologyQuery(ontology, queryFields);

if (!allFieldMatched(queryFieldsSet, ontologyMetricsDimensionsAndBizName)) {
    queryStatement.setStatus(QueryState.INVALID);
    return;
}

queryStatement.setOntologyQuery(ontologyQuery);
convertNameToBizName(queryStatement);
rewriteOrderBy(queryStatement);
sqlQuery.setTable(Constants.TABLE_PREFIX
        + queryStatement.getDataSetId());
```

实际位置：`SqlQueryParser.java:45-100`。

它解决四个关键问题：

1. SELECT 中的字段是否都是语义字段；
2. 每个指标和维度属于哪个数据模型；
3. 语义名称如何转换成 `bizName`；
4. 数据集表名如何统一成 `DATASET_<id>`。

字段替换代码：

```java
Map<String, String> fieldNameToBizNameMap =
        getNameToBizNameMap(queryStatement.getOntologyQuery());

String sql = queryStatement.getSqlQuery().getSql();
sql = SqlReplaceHelper.replaceFields(
        sql, fieldNameToBizNameMap, true);
sql = SqlReplaceHelper.replaceTable(
        sql, Constants.TABLE_PREFIX + queryStatement.getDataSetId());
queryStatement.getSqlQuery().setSql(sql);
```

实际位置：`SqlQueryParser.java:178-190`。

`buildOntologyQuery()` 会根据字段查找对应的指标模型和维度模型，优先让指标和维度落在同一模型；如果字段分散在不同模型，则保留多个模型，后续由 `SqlBuilder` 处理 JOIN。

### 11.4 `OntologyQueryParser`：生成底层 ontology SQL

文件：

`headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/parser/OntologyQueryParser.java`

```java
public void parse(QueryStatement queryStatement) throws Exception {
    Ontology ontology = queryStatement.getOntology();
    S2CalciteSchema semanticSchema = S2CalciteSchema.builder()
            .schemaKey("DATASET_" + queryStatement.getDataSetId())
            .ontology(ontology)
            .build();

    SqlBuilder sqlBuilder = new SqlBuilder(semanticSchema);
    String sql = sqlBuilder.buildOntologySql(queryStatement);
    queryStatement.getOntologyQuery().setSql(sql);
}
```

实际位置：`OntologyQueryParser.java:26-36`。

### 11.5 `SqlBuilder`：生成真实表和字段

文件：

`headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/parser/calcite/SqlBuilder.java`

关键方法：

```text
buildOntologySql() 48
probeRelatedModels() 82
buildGraph() 125
render() 157
buildJoin() 209
renderOne() 327
```

核心代码：

```java
public String buildOntologySql(QueryStatement queryStatement) throws Exception {
    OntologyQuery ontologyQuery = queryStatement.getOntologyQuery();
    Ontology ontology = queryStatement.getOntology();

    Set<ModelResp> dataModels = ontologyQuery.getModels();
    if (dataModels == null || dataModels.isEmpty()) {
        throw new Exception("data model not found");
    }

    TableView tableView;
    if (!CollectionUtils.isEmpty(ontology.getJoinRelations())
            && dataModels.size() > 1) {
        Set<ModelResp> models = probeRelatedModels(
                dataModels, queryStatement.getOntology());
        tableView = render(ontologyQuery, models, scope, schema);
    } else {
        tableView = render(ontologyQuery, dataModels, scope, schema);
    }

    SqlNode parserNode = tableView.build();
    return SemanticNode.getSql(parserNode, engineType);
}
```

实际位置：`SqlBuilder.java:48-80`。

`SqlBuilder` 最终会把语义层字段展开为：

```text
指标表达式 -> 真实字段 + 聚合函数
维度       -> 真实字段或 CASE 表达式
模型       -> 真实物理表
JoinRelation -> JOIN ... ON ...
```

如果查询涉及多个模型，它会先利用 `JoinRelation` 构建图，再查找相关模型之间的路径，最后由 `buildJoin()` 拼接 SQL。

### 11.6 `mergeOntologyQuery()`：合并外层 S2SQL 和内层物理 SQL

`DefaultSemanticTranslator.mergeOntologyQuery()` 位于 `DefaultSemanticTranslator.java:58-97`。

它拿到两部分 SQL：

```java
String ontologyOuterSql = sqlQuery.getSql();
String ontologyInnerTable = sqlQuery.getTable();
String ontologyInnerSql = ontologyQuery.getSql();
```

如果数据库支持 `WITH`，生成类似：

```sql
WITH DATASET_10001 AS (
    -- SqlBuilder 生成的真实物理 SQL
    SELECT
        t.region_name,
        SUM(t.sale_amount) AS sale_amount,
        t.partition_date
    FROM sales_detail t
    GROUP BY t.region_name, t.partition_date
)
-- SqlQueryParser 处理过的外层 S2SQL
SELECT region_name, SUM(sale_amount)
FROM DATASET_10001
WHERE partition_date BETWEEN '2026-08-01' AND '2026-08-07'
GROUP BY region_name
```

如果数据库不支持 `WITH`，则把内层 SQL 替换成外层 SQL 的子查询：

```sql
SELECT ...
FROM (
    SELECT ...
    FROM sales_detail
) DATASET_10001
WHERE ...
```

对应源码：

```java
if (sqlQuery.isSupportWith()) {
    finalSql = "with " +
            tables.stream()
                    .map(t -> String.format("%s as (%s)",
                            t.getLeft(), t.getRight()))
                    .collect(Collectors.joining(","))
            + "\n" + ontologyOuterSql;
} else {
    finalSql = StringUtils.replace(
            ontologyOuterSql,
            tb.getLeft(),
            "(" + tb.getRight() + ") " +
                    (sqlQuery.isWithAlias() ? "" : tb.getLeft()),
            -1);
}
```

---

## 12. 最终执行数据库查询

### 12.1 Chat 层 `SqlExecutor`

文件：

`chat/server/src/main/java/com/tencent/supersonic/chat/server/executor/SqlExecutor.java`

执行时优先使用已经翻译、修正过的物理 SQL：

```java
String finalSql = StringUtils.isNotBlank(
        parseInfo.getSqlInfo().getQuerySQL())
        ? parseInfo.getSqlInfo().getQuerySQL()
        : parseInfo.getSqlInfo().getCorrectedS2SQL();

QuerySqlReq sqlReq = QuerySqlReq.builder()
        .sql(finalSql)
        .build();
sqlReq.setSqlInfo(parseInfo.getSqlInfo());
sqlReq.setDataSetId(parseInfo.getDataSetId());

SemanticQueryResp queryResp = semanticLayer.queryByReq(
        sqlReq, executeContext.getRequest().getUser());
```

实际位置：`SqlExecutor.java:66-93`。

注意：这里的 `querySQL` 已经是物理 SQL；只有在没有物理 SQL 时才退回 `correctedS2SQL`。

### 12.2 `S2SemanticLayerService.queryByReq()`

文件：

`headless/server/src/main/java/com/tencent/supersonic/headless/server/facade/service/impl/S2SemanticLayerService.java`

执行流程源码：

```java
// 1. 查询缓存
Object query = queryCache.query(queryReq, cacheKey);
if (Objects.nonNull(query)) {
    return (SemanticQueryResp) query;
}

// 2. 翻译 SQL
QueryStatement queryStatement = buildQueryStatement(queryReq, user);
if (!queryStatement.isTranslated()) {
    semanticTranslator.translate(queryStatement);
}

// 3. 选择执行器
for (QueryExecutor queryExecutor : queryExecutors) {
    if (queryExecutor.accept(queryStatement)) {
        queryResp = queryExecutor.execute(queryStatement);
    }
}

// 4. 写入缓存
queryCache.put(cacheKey, queryResp);
```

实际位置：`S2SemanticLayerService.java:103-148`。

### 12.3 `JdbcExecutor`

文件：

`headless/core/src/main/java/com/tencent/supersonic/headless/core/executor/JdbcExecutor.java`

核心代码：

```java
public SemanticQueryResp execute(QueryStatement queryStatement) {
    SqlUtils sqlUtils = ContextUtils.getBean(SqlUtils.class);
    String sql = StringUtils.normalizeSpace(queryStatement.getSql());
    DatabaseResp database = queryStatement.getOntology().getDatabase();

    SemanticQueryResp result = new SemanticQueryResp();
    try {
        SqlUtils sqlUtil = sqlUtils.init(database);
        sqlUtil.queryInternal(queryStatement.getSql(), result);
        result.setSql(sql);
    } catch (Exception e) {
        result.setErrorMsg(e.getMessage());
    }
    return result;
}
```

实际位置：`JdbcExecutor.java:24-51`。

数据库连接和方言由 `DatabaseResp` 以及对应的数据库适配逻辑决定，例如 MySQL、PostgreSQL、ClickHouse 和 H2。

---

## 13. 一个完整示例

用户输入：

```text
查询最近 7 天各地区的销售额
```

假设语义 Schema 中定义了：

```text
指标：销售额
  name     = 销售额
  bizName  = sale_amount
  defaultAgg = SUM

维度：地区
  name     = 地区
  bizName  = region_name

分区维度：日期
  name     = 日期
  bizName  = partition_date
```

### 13.1 识别阶段

```text
“销售额” -> METRIC -> sale_amount
“地区”   -> DIMENSION -> region_name
“最近7天” -> DateConf -> partition_date + 起止日期
```

对应对象大致是：

```text
SchemaElementMatch("销售额", METRIC, sale_amount)
SchemaElementMatch("地区", DIMENSION, region_name)
DateConf(dateField = partition_date, startDate = ..., endDate = ...)
```

### 13.2 语义结构阶段

```text
SemanticParseInfo
  dataSet          = 销售数据集
  metrics          = [销售额]
  dimensions       = [地区]
  dimensionFilters = []
  dateInfo         = [partition_date, startDate, endDate]
  aggType          = SUM
  queryType        = AGGREGATE
```

### 13.3 结构化查询阶段

`QueryReqBuilder.buildStructReq()` 生成：

```text
groups = [region_name]
aggregators = [SUM(sale_amount)]
dateInfo = partition_date BETWEEN startDate AND endDate
queryType = AGGREGATE
```

这里展示的是 `QueryStructReq` 的内部结构。`RuleSemanticQuery.buildS2Sql()` 随后会把 `region_name`、`sale_amount` 和 `partition_date` 转回对应的 Schema 名称，再调用 `QueryStructReq.convert()` 生成 S2SQL。

### 13.4 S2SQL 阶段

```sql
SELECT 地区, SUM(销售额)
FROM 销售数据集
WHERE 日期 BETWEEN '2026-08-12' AND '2026-08-19'
GROUP BY 地区
```

### 13.5 Ontology 阶段

`SqlQueryParser` 找到并转换：

```text
地区       -> 地区维度 -> bizName region_name
销售额     -> 销售额指标 -> bizName sale_amount
销售数据集 -> 数据集 10001 -> DATASET_10001
```

`SqlBuilder` 根据模型和指标定义生成：

```sql
SELECT
    t.region_name,
    SUM(t.sale_amount) AS sale_amount,
    t.partition_date
FROM sales_detail t
GROUP BY
    t.region_name,
    t.partition_date
```

### 13.6 物理 SQL 阶段

`DefaultSemanticTranslator.mergeOntologyQuery()` 将两部分组合，得到类似：

```sql
WITH DATASET_10001 AS (
    SELECT
        t.region_name,
        SUM(t.sale_amount) AS sale_amount,
        t.partition_date
    FROM sales_detail t
    GROUP BY t.region_name, t.partition_date
)
SELECT region_name, SUM(sale_amount)
FROM DATASET_10001
WHERE partition_date BETWEEN '2026-08-12' AND '2026-08-19'
GROUP BY region_name
```

实际 SQL 可能因为数据库方言、指标表达式、默认过滤条件、模型 JOIN 和 SQL 优化器而不同。

### 13.7 执行阶段

```text
SqlExecutor
  -> S2SemanticLayerService.queryByReq()
  -> JdbcExecutor.execute()
  -> SqlUtils.queryInternal()
  -> 数据库
```

---

## 14. 规则分支和 LLM 分支的区别

| 对比项 | 规则分支 | LLM 分支 |
| --- | --- | --- |
| 语义识别 | 词典、数据库名称、向量匹配 | 使用 Schema 和上下文生成理解结果 |
| 查询模式 | `QueryMatcher` + 规则类 | LLM 直接生成 S2SQL |
| 中间对象 | `SemanticParseInfo` | 同样是 `SemanticParseInfo` |
| S2SQL 来源 | `QueryStructReq.convert()` | LLM 返回结果 |
| 后续物理 SQL | 统一经过 Semantic Layer | 统一经过 Semantic Layer |
| 可靠性 | 可解释、稳定、覆盖有限 | 表达能力强，但需要校验和重试 |
| 兜底方式 | STRICT -> MODERATE -> LOOSE | 多次重试，并可传入全部语义字段 |

关键结论：LLM 只替代了“如何理解问题和组织 S2SQL”的部分，不能绕过后面的语义 Schema、Ontology 和物理 SQL 翻译。

---

## 15. 关键文件索引

### 请求入口和 Chat 调度

```text
chat/server/src/main/java/com/tencent/supersonic/chat/server/rest/ChatQueryController.java
chat/server/src/main/java/com/tencent/supersonic/chat/server/service/impl/ChatQueryServiceImpl.java
chat/server/src/main/java/com/tencent/supersonic/chat/server/util/QueryReqConverter.java
chat/server/src/main/java/com/tencent/supersonic/chat/server/parser/NL2SQLParser.java
```

### 语义 Schema 和映射

```text
headless/api/src/main/java/com/tencent/supersonic/headless/api/pojo/SemanticSchema.java
headless/api/src/main/java/com/tencent/supersonic/headless/api/pojo/SemanticParseInfo.java
headless/api/src/main/java/com/tencent/supersonic/headless/api/pojo/SchemaElement.java
headless/api/src/main/java/com/tencent/supersonic/headless/api/pojo/SchemaElementMatch.java
headless/api/src/main/java/com/tencent/supersonic/headless/api/pojo/SchemaMapInfo.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/BaseMapper.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/KeywordMapper.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/EmbeddingMapper.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/QueryFilterMapper.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/PartitionTimeMapper.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/mapper/TermDescMapper.java
```

### 语义解析和 S2SQL

```text
headless/server/src/main/java/com/tencent/supersonic/headless/server/utils/ChatWorkflowEngine.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/rule/RuleSqlParser.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/query/rule/RuleSemanticQuery.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/rule/TimeRangeParser.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/rule/AggregateTypeParser.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/QueryTypeParser.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/utils/QueryReqBuilder.java
headless/api/src/main/java/com/tencent/supersonic/headless/api/pojo/request/QueryStructReq.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/llm/LLMSqlParser.java
headless/chat/src/main/java/com/tencent/supersonic/headless/chat/parser/llm/LLMResponseService.java
```

### 物理 SQL 翻译和执行

```text
headless/server/src/main/java/com/tencent/supersonic/headless/server/facade/service/impl/S2SemanticLayerService.java
headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/DefaultSemanticTranslator.java
headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/parser/SqlQueryParser.java
headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/parser/StructQueryParser.java
headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/parser/OntologyQueryParser.java
headless/core/src/main/java/com/tencent/supersonic/headless/core/translator/parser/calcite/SqlBuilder.java
headless/core/src/main/java/com/tencent/supersonic/headless/core/executor/JdbcExecutor.java
chat/server/src/main/java/com/tencent/supersonic/chat/server/executor/SqlExecutor.java
```

---

## 16. 一句话总结

这个项目不是简单的“自然语言交给 LLM 生成 SQL”，而是一个分层的语义查询编译流程：

```text
自然语言
  -> SchemaMapper 找到指标、维度和值
  -> SemanticParser 组合查询意图
  -> SemanticParseInfo 保存统一语义结构
  -> 规则或 LLM 生成 S2SQL
  -> SqlQueryParser 建立 OntologyQuery
  -> SqlBuilder 将语义对象展开为真实表和字段
  -> DefaultSemanticTranslator 合并 SQL
  -> JdbcExecutor 执行物理 SQL
```

最关键的两个中间对象是：

```text
SchemaElementMatch  负责回答：用户文字命中了哪个语义对象？
SemanticParseInfo   负责回答：这些对象在本次查询中分别扮演什么角色？
```
