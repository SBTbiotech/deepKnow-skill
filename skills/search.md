# search — 知识检索与可核验引用

使用 `POST /knowledge-search` 进行默认范围、单知识库或跨知识库检索。该接口与快速问答、MCP 共用 Search → Rerank → Merge → FilterTopK 链路。

## 请求参数

| 参数 | 说明 |
|---|---|
| `query` | 必填，用户的检索问题 |
| `knowledge_base_ids` | 可选；未指定知识库或文件夹范围时，由后端按名称/描述路由，未命中时回退到可见知识库 |
| `knowledge_base_folder_ids` | 可选的文件夹 ID 数组；后端动态展开可访问知识库，与显式知识库 ID 取并集并去重 |
| `tag_ids` | 可选的分类 ID 数组，硬过滤到这些分类；父分类自动展开子孙 |
| `enable_rerank` | 是否启用 rerank |
| `rerank_top_k` | 最终最多返回的结果数；不传时使用租户配置 |
| `vector_threshold` | 可选，向量通道过滤阈值 |
| `keyword_threshold` | 可选，关键词通道过滤阈值 |
| `rerank_threshold` | 可选，rerank 通道过滤阈值 |

```bash
# 推荐默认检索
dk_api POST "knowledge-search" \
  '{"query":"马来西亚 保健品 进口 法规"}'

# 显式单库检索
dk_api POST "knowledge-search" \
  '{"query":"部署流程","knowledge_base_ids":["<kb_id>"],"enable_rerank":true,"rerank_top_k":5}'

# 跨库并按分类硬过滤
dk_api POST "knowledge-search" \
  '{"query":"跨境电商申报流程","knowledge_base_ids":["<kb_id_1>","<kb_id_2>"],"tag_ids":["<tag_id>"]}'
```

## 范围选择

1. 用户明确指定知识库时，先用 `GET /knowledge-bases` 把名称解析为 ID。
2. 问题包含明确国家、地区、产品、品牌或标准名时，先匹配知识库 `name` 和 `description`；一个或少数几个库明显相关时优先限定范围。
3. 没有明确范围时不传 `knowledge_base_ids` 和 `knowledge_base_folder_ids`，使用后端默认路由。
4. 正常问答不要调用旧的 `GET /knowledge-bases/:id/hybrid-search`；它只用于历史兼容和检索诊断。

## 文件夹范围

用户指定文件夹时，先按 `knowledge-bases.md` 调用 `GET /knowledge-base-folders/tree`，结合名称和路径唯一确定 ID：

```bash
dk_api POST "knowledge-search" \
  '{"query":"稳定性研究","knowledge_base_folder_ids":["<folder_id>"]}'
```

文件夹展开范围与显式 `knowledge_base_ids` 是并集，不是交集。用户要求“只查此文件夹”时不要额外加入其他知识库；`tag_ids` 继续对结果硬过滤。空文件夹范围不应回退全库；文件夹不存在或不可用时说明错误，不删除范围参数重试。

## 分类过滤

问题中有明确品类、区域、部门、专题或法规名称时，可以先调用 `GET /knowledge-bases/:id/tags?page_size=100`，选择 1–3 个具体分类 ID 传入 `tag_ids`。

- `tag_ids` 是硬过滤；只返回属于这些分类或其展开子树的文档。
- `knowledge_count == 0` 的分类不要传。
- 传入不可见或不存在的分类会返回空结果，不会退化成全库检索。
- 名称过短或过泛时不要仅凭一个词匹配；结合父子关系和 `knowledge_count` 判断。
- 用户问题没有分类线索时不要强行添加过滤。

## 核心响应字段

`data[]` 每项包含：

| 字段 | 说明 |
|---|---|
| `rank` | 后端最终证据顺序，从 1 开始 |
| `id` | 分块 ID，即引用使用的 `chunk_id` |
| `content` | 分块原文；事实结论和 quote 的唯一文本依据 |
| `score` / `score_type` | 当前排序通道的诊断分及其语义 |
| `knowledge_base_id` | 知识库 ID |
| `knowledge_id` | 文档 ID，不能代替 chunk ID |
| `knowledge_title` / `knowledge_filename` | 文档标题 / 文件名 |
| `chunk_index` / `chunk_type` | 分块位置 / 类型 |
| `parent_chunk_id` | 父分块 ID |
| `chunk_metadata` | chunk 级元数据；表格分块可能含 `asset_id`、`table_id`、`table_version`、`kind` |
| `image_info` / `match_type` / `metadata` | 图片、匹配和文档级元数据 |

常见 `chunk_type`：`text`、`summary`、`image_ocr`、`image_caption`、`table_summary`、`table_column`、`faq`、`wiki_page`。

## 证据与引用

先按响应顺序确定性构造证据列表，再生成答案：

```text
rank            = item.rank
chunk_id        = item.id
knowledge_id    = item.knowledge_id
knowledge_title = item.knowledge_title
quote           = item.content 中逐字支持结论的最短原句
```

- 保持数组顺序并以 `rank` 为准，不按 `score` 重排。不同 `score_type` 的值不可直接比较，也不保证统一在 0–1。
- 每个事实结论必须能在对应 `content` 找到逐字依据。标题、摘要元数据、生成问题、`matched_content` 或同篇共现不能单独作为证据。
- 引用至少保留 `knowledge_title + chunk_id + quote`。序号引用必须能映射回这些字段，不能只输出孤立的 `[N]`。
- 同一分块提及多个成分、人物或事件，不代表它们具有相同作用或因果关系。原文未明示时写“证据不足”。
- 没有足够结果时说明未检索到可靠依据，并建议调整范围或关键词；不要补写常识当作库内答案。

## 命中后深读

用户要求上下文或命中内容不足时，按 `knowledge-bases.md` 使用 `knowledge_id` 分页读取文本分块，或使用分块 ID／`parent_chunk_id` 读取单块。新增结论必须引用实际深读原文，不由同篇标题或相邻位置推断；文档 ID 不能代替引用的分块 ID。

## 表格命中

`table_summary` 和 `table_column` 的 `content` 是 AI 生成的描述，不是原始行数据。问题涉及具体单元格、行、筛选、排序、汇总或表头样式时，读取 `tabular.md` 并通过表格接口取真实数据。

