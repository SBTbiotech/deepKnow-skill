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

### 文件夹树：`GET /knowledge-base-folders/tree`

返回 `data.folders[]` 和 `data.root_knowledge_bases[]`。文件夹节点包含 `id`、`name`、`parent_id`、`children[]`、`knowledge_bases[]`、`display_group`、`role`。

```bash
dk_api GET "knowledge-base-folders/tree"
```

按名称和父子路径定位真实文件夹 ID；重名时展示完整路径让用户选择。文件夹 ID 用于 `knowledge_base_folder_ids`，不是分类 `tag_ids`；权限由后端校验，不根据名称或树中的角色自行扩大范围。

### 分块深读

```bash
# 文档 ID 用于列出该文档的文本分块
dk_api GET "chunks/<knowledge_id>?page=1&page_size=20"
# 分块 ID 用于读取单个分块，包括已知父分块
dk_api GET "chunks/by-id/<chunk_id>"
```

- 列表接口 `GET /chunks/:knowledge_id` 返回 `data[]`、`total`、`page`、`page_size`；页码从 1 开始，默认每页 10 条，最多 100 条。
- 当前列表仅返回 `text` 类型分块，不代表包含图片、表格摘要和所有其他类型的完整文档。按本次 `total` 分页，不能把第一页称为全文。
- 单块接口 `GET /chunks/by-id/:id` 返回 `data` 对象。保留实际 `id`、`knowledge_id`、`content`、`chunk_index`、`chunk_type`、`parent_chunk_id` 等字段（以响应为准）。
- 从检索命中取得文档和分块标识，不猜 ID；返回后核对归属。无权限或不存在时停止，不换凭据绕过。
- 深读引用保留文档标题、真实分块 ID 和 `content` 中支持结论的原句；列表若没有标题，沿用已确认的文档标题或读取知识条目详情。
- 按用户问题读取所需上下文，超过合理读取量时说明已读范围，不无限翻页。

### 浏览与状态检查

1. 用户用名称指代知识库时，先调 `GET /knowledge-bases`，按 `name` 和 `description` 匹配。
2. 唯一匹配时继续；多个合理候选时用编号列表让用户选择。
3. 浏览条目时使用分页接口，不把单页数量说成知识库总数；总数使用响应 `total`。
4. 查看导入状态时调 `GET /knowledge/:id`。`failed` 时展示 `error_message`，不要自动重新解析。

## 输出

- 知识库列表：名称、描述、类型、知识条目数、分块数、是否正在处理。
- 知识列表：标题、来源类型、解析状态、启用状态、创建时间。
- 时间字段按用户所在时区转换为可读日期时间；保留时区信息，避免歧义。

