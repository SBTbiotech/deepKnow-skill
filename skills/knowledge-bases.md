# knowledge-bases — 知识库与内容浏览

列出知识库、分类和知识条目，或读取单个知识条目的完整状态。

## 接口

### `GET /knowledge-bases`

列出当前 API Key 可见的知识库。

核心回包字段：`data[].id`、`name`、`description`、`type`（`document` 或 `faq`）、`embedding_model_id`、`knowledge_count`、`chunk_count`、`is_processing`、`created_at`。

### `GET /knowledge-bases/:id`

查看单个知识库详情。只有用户已给出 ID，或已通过列表唯一确定 ID 后才调用。

### `GET /knowledge-bases/:id/tags`

列出知识库下的树形分类。

| 参数 | 说明 |
|---|---|
| `page` | 页码 |
| `page_size` | 每页数量 |
| `keyword` | 可选的分类名称关键词 |

常用字段：`id`、`name`、`parent_id`、`knowledge_count`、`chunk_count`。父分类可包含子分类；检索时传父分类 ID，后端会展开其子树。

### `GET /knowledge-bases/:id/knowledge`

分页浏览知识库内容。

| 参数 | 说明 |
|---|---|
| `page` | 页码，从 1 开始 |
| `page_size` | 每页数量 |
| `tag_id` | 可选，按单个分类筛选 |

响应为 `data[]`，并包含 `total`、`page`、`page_size`。展示列表时优先给出标题、类型、解析状态和创建时间。

### `GET /knowledge/:id`

获取知识条目详情，也用于检查导入进度。

核心字段：

| 字段 | 说明 |
|---|---|
| `id` | 知识条目 ID |
| `title` / `description` | 标题 / 自动生成摘要 |
| `type` | `file`、`url` 或 `manual` |
| `parse_status` | `pending` → `processing` → `completed`，或 `failed` |
| `enable_status` | `enabled` 或 `disabled`；通常解析成功后自动启用 |
| `file_name` / `file_type` / `file_size` | 文件元数据 |
| `source` | URL 来源 |
| `created_at` / `processed_at` | 创建 / 完成时间 |
| `error_message` | 解析失败原因 |

## 工作流

1. 用户用名称指代知识库时，先调 `GET /knowledge-bases`，按 `name` 和 `description` 匹配。
2. 唯一匹配时继续；多个合理候选时用编号列表让用户选择。
3. 浏览条目时使用分页接口，不把单页数量说成知识库总数；总数使用响应 `total`。
4. 查看导入状态时调 `GET /knowledge/:id`。`failed` 时展示 `error_message`，不要自动重新解析。

## 输出

- 知识库列表：名称、描述、类型、知识条目数、分块数、是否正在处理。
- 知识列表：标题、来源类型、解析状态、启用状态、创建时间。
- 时间字段按用户所在时区转换为可读日期时间；保留时区信息，避免歧义。

