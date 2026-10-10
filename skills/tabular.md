# tabular — 表格资产与原始数据查询

检索命中表格摘要后，使用表格接口查看结构、查询真实行数据或还原原始 CSV 网格。

## 何时必须使用

当 `knowledge-search` 返回 `chunk_type == "table_summary"` 或 `"table_column"`，且用户询问具体行、单元格、筛选、排序、聚合、列值或表头布局时，必须使用本流程。分块 `content` 只是 AI 描述，不能作为原始数值依据。

## 接口

| 接口 | 用途 |
|---|---|
| `POST /knowledge-bases/:id/tabular-query` | 将自然语言问题转换为后端校验的受限查询计划并执行 |
| `GET /knowledge-bases/:id/tabular-assets` | 列出知识库下的表格资产 |
| `GET /tabular-assets/:asset_id` | 获取资产详情和子表清单 |
| `GET /tabular-tables/:table_id/schema` | 获取列白名单、类型和统计信息 |
| `POST /tabular-tables/:table_id/rows` | 使用受控 DSL 查询行数据 |
| `POST /tabular-tables/:table_id/source-preview` | 预览原始网格、合并单元格和不规则布局 |

## 工作流

用户已指定知识库但未明确表或列时，可先使用下文自然语言查询；表、列和过滤条件已确定时，继续使用以下 schema → rows 流程。两种方式都只使用实际数据，不发送 SQL。

1. 优先从命中项 `chunk_metadata.table_id` 或 `asset_id` 取得标识。
2. `table_column` 没有足够元数据时，可通过 `parent_chunk_id` 追回对应 `table_summary`。
3. 只有 `asset_id` 时，调 `GET /tabular-assets/:asset_id` 获取子表清单。
4. 调 `GET /tabular-tables/:table_id/schema`。查询参数中的列名必须取自 `columns[].canonical_name`；展示时使用 `display_name`。
5. 用受控 DSL 调 rows 接口；不得发送原始 SQL，不得使用 schema 白名单之外的列。
6. 需要查看原始布局、合并单元格或不规则报表时使用 source-preview。

## 自然语言查询

```bash
dk_api POST "knowledge-bases/<kb_id>/tabular-query" \
  '{"query":"华东区销售额最高的产品是什么？"}'
```

只传非空 `query`，知识库 ID 必须已确认；需要该库查看权限。后端使用知识库配置的模型规划并校验，不接受调用方 SQL、模型凭据或存储路径。

读取 `response.data.status`：

- `completed`：读取 `plan` 和 `results[]`，按真实结果回答；说明所用表、字段、过滤和统计口径，不把计划本身当结果。
- `clarification`：按 `clarification` 追问最少信息，收到补充后更新问题；不能把候选计划当已执行。
- `no_table`：该范围没有可查询表，说明未获得表格数据，不改查其他知识库。
- 未知状态或结果缺失：说明响应不足，不猜数值、不循环重试。

`plan` 可能包含 `table_id`、`operation`、`column`、`columns`、`aggregate`、`filters`、`group_by`、`sort_by`、`sort_direction`、`limit`；`results` 使用后端检索结果结构。只解释实际返回的字段，保留可复核的表标识；区分后端计算结果、原始行和 AI 摘要。

## Schema 与行查询

```bash
dk_api GET "tabular-tables/<table_id>/schema"
```

关注 `data.columns[].canonical_name`、`display_name`、`logical_type` 和可用统计信息。

```bash
dk_api POST "tabular-tables/<table_id>/rows" \
  '{
    "columns":["region","sales_amount"],
    "filters":[{"column":"region","op":"eq","value":"华东"}],
    "sort":[{"column":"sales_amount","direction":"desc"}],
    "page_size":50
  }'
```

请求支持的核心字段：`columns`、`filters`、`sort`、`cursor`、`page_size`、`version`。具体运算符和类型以 schema 或后端错误说明为准，不猜测未声明的 DSL 语法。

- 首次请求不传 `cursor`；有下一页时使用响应游标继续。
- 如果命中元数据含 `table_version`，可作为 `version` 传入以固定数据快照；版本冲突时重新读取 schema/资产详情，不静默混用不同版本。
- 聚合能力未在 schema 或接口定义中明确提供时，先取得所需行，再在可控数据量内本地计算，并向用户说明口径。

## 原始网格预览

```bash
dk_api POST "tabular-tables/<table_id>/source-preview" \
  '{"row_offset":0,"page_size":50}'
```

使用 `row_offset` 和 `page_size` 分页。该接口适合需要还原 CSV/报表原貌、合并单元格或不规则列数的场景；普通筛选查询优先用 rows。

## 输出与证据

- 展示列名使用 `display_name`，同时保留必要的 `table_id` 和数据版本以便复核。
- 明确区分“原始行数据”“本地计算结果”和“AI 生成的表摘要”。
- 排序、筛选或汇总时说明所用列、过滤条件和数据范围。
- 数据跨页时继续分页直至满足用户问题或达到合理上限；不能把第一页说成全表。

