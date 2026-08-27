# import — 知识导入与管理

上传文件、导入 URL、写入或编辑 Markdown，并跟踪解析状态。

## 核心约束

1. 所有请求必须遵循 `SKILL.md` 的鉴权、版本上报和错误处理规范。
2. 用户用名称指定知识库时，先调 `GET /knowledge-bases` 获取唯一的 `knowledge_base_id`；有多个候选时让用户选择。
3. 文件上传使用 `multipart/form-data`；URL、Markdown 和编辑请求使用 JSON。不要混用两种格式。
4. 普通网页、公众号文章和飞书文档统一走 URL 导入；不要预先抓取并改写成 Markdown，除非用户明确要求。
5. 自动写入 Markdown 必须显式传 `status: "publish"`；只有用户明确要求草稿时才传 `status: "draft"`。
6. 编辑、重新解析或删除已有条目前，先解析到唯一的知识条目 ID；删除必须取得用户明确确认。
7. 导入成功后有界轮询解析状态。失败时展示 `error_message` 并停止，不自动重新解析。

## 接口

### `POST /knowledge-bases/:knowledge_base_id/knowledge/file` — 上传文件

**请求参数：**

路径参数：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `knowledge_base_id` | string | 是 | 目标知识库 ID |

`multipart/form-data` 字段：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `file` | file | 是 | 本地文件；使用 `curl -F 'file=@<path>'` 上传 |
| `enable_multimodel` | boolean | 否 | 是否启用多模态解析 |

**回包：**

| 字段 | 说明 |
|---|---|
| `success` | 请求是否成功 |
| `data.id` | 新建知识条目 ID，即后续轮询使用的 `knowledge_id` |

支持类型包括 `pdf`、`docx`、`xlsx`、`pptx`、`txt`、`md`、`csv`、`html` 等，最终以服务端校验为准。

```bash
# 复用 SKILL.md 公共调用设置中的变量；版本值只从其 frontmatter 读取。
curl -sS -X POST "$DEEPKNOW_API_BASE/knowledge-bases/<kb_id>/knowledge/file" \
  -H "X-API-Key: $DEEPKNOW_API_KEY" \
  -H "X-DeepKnow-Channel: skill" \
  -H "X-DeepKnow-Skill-Version: $DEEPKNOW_SKILL_VERSION" \
  -F 'file=@document.pdf' \
  -F 'enable_multimodel=true'
```

不要手动设置 `Content-Type`；由 `curl -F` 生成带 boundary 的 multipart 请求头。

### `POST /knowledge-bases/:knowledge_base_id/knowledge/url` — 导入 URL

**请求参数：**

| 参数 | 位置 | 类型 | 必填 | 说明 |
|---|---|---|---|---|
| `knowledge_base_id` | path | string | 是 | 目标知识库 ID |
| `url` | JSON body | string | 是 | 网页、公众号文章或飞书资源 URL |
| `enable_multimodel` | JSON body | boolean | 否 | 是否启用多模态解析 |

**回包：**

| 字段 | 说明 |
|---|---|
| `success` | 请求是否成功 |
| `data.id` | 新建知识条目 ID，即后续轮询使用的 `knowledge_id` |

```bash
dk_api POST "knowledge-bases/<kb_id>/knowledge/url" \
  '{"url":"https://example.com/article","enable_multimodel":true}'
```

#### 飞书 URL

域名包含 `feishu.cn` 时仍调用本接口。后端负责识别 `/wiki/`、`/docx/`、`/docs/`、`/sheets/`、`/base/`、`/file/`、`/minutes/` 等资源路径，并校验当前用户的飞书 OAuth 和文档访问权限。

如果路径不受支持或授权失败，向用户返回后端错误；不要自动退回普通网页抓取。

### `POST /knowledge-bases/:knowledge_base_id/knowledge/manual` — 写入 Markdown

**请求参数：**

| 参数 | 位置 | 类型 | 必填 | 说明 |
|---|---|---|---|---|
| `knowledge_base_id` | path | string | 是 | 目标知识库 ID |
| `title` | JSON body | string | 是 | 知识标题 |
| `content` | JSON body | string | 是 | Markdown 内容 |
| `status` | JSON body | string | 是 | `publish` 或 `draft`；自动写入时传 `publish` |
| `tag_id` | JSON body | string | 否 | 目标分类 ID |

**回包：**

| 字段 | 说明 |
|---|---|
| `success` | 请求是否成功 |
| `data.id` | 新建知识条目 ID，即后续轮询使用的 `knowledge_id` |

```bash
dk_api POST "knowledge-bases/<kb_id>/knowledge/manual" \
  '{"title":"会议纪要","content":"# Q1 复盘\n\n关键要点...","status":"publish"}'
```

省略 `status` 时后端会按草稿处理，因此不得依赖默认值。

### `PUT /knowledge/manual/:id` — 编辑 Markdown 知识

**请求参数：**

| 参数 | 位置 | 类型 | 必填 | 说明 |
|---|---|---|---|---|
| `id` | path | string | 是 | 已有 Markdown 知识条目 ID |
| `title` | JSON body | string | 是 | 更新后的标题 |
| `content` | JSON body | string | 是 | 更新后的 Markdown 内容 |
| `status` | JSON body | string | 是 | `publish` 或 `draft` |

**回包：**

| 字段 | 说明 |
|---|---|
| `success` | 更新是否成功 |

执行前展示目标标题和 ID，确认不是同名的其他条目。

### `GET /knowledge/:id` — 查询解析状态

**请求参数：**

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | 知识条目 ID |

**回包：**

| 字段 | 说明 |
|---|---|
| `data.id` | 知识条目 ID |
| `data.title` | 标题 |
| `data.parse_status` | `pending`、`processing`、`completed` 或 `failed` |
| `data.enable_status` | `enabled` 或 `disabled`；通常解析完成后自动启用 |
| `data.error_message` | 解析失败原因 |
| `data.processed_at` | 解析完成时间 |

### `POST /knowledge/:id/reparse` — 重新解析

**请求参数：**

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | 需要重新解析的知识条目 ID |

**回包：**

| 字段 | 说明 |
|---|---|
| `success` | 是否成功提交重新解析任务 |

只在已经查看失败原因且用户同意后执行，不自动重试。提交成功后重新调用 `GET /knowledge/:id` 获取实际解析状态。

### `DELETE /knowledge/:id` — 删除知识条目

**请求参数：**

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | 需要删除的知识条目 ID |

**回包：**

| 字段 | 说明 |
|---|---|
| `success` | 是否删除成功 |

删除前必须展示目标标题和 ID，并取得用户明确确认。不得根据模糊名称直接删除。

## 工作流

1. **确定知识库**：用户提供 ID 时直接使用；提供名称时先调 `GET /knowledge-bases` 解析 ID。
2. **选择导入方式**：本地文件走 file；网页、公众号和飞书链接走 url；用户提供或要求直接写入的 Markdown 走 manual。
3. **创建知识条目**：按对应接口提交请求，从 `data.id` 记录 `knowledge_id`。
4. **轮询解析状态**：调 `GET /knowledge/:id`；`pending` 或 `processing` 时建议按 2、4、8、15 秒逐步放慢，之后不高于每 15 秒一次，并设置总等待上限。
5. **结束条件**：`completed` 且 `enable_status == "enabled"` 时报告完成；`failed` 时展示 `error_message` 并停止；超过等待上限时报告当前状态和 `knowledge_id`。
6. **已有条目管理**：编辑、重新解析或删除前，先通过列表或详情接口确认唯一 ID；按各接口的确认要求执行。

## 输出格式

- 创建成功后展示知识标题、目标知识库、`knowledge_id` 和当前解析状态。
- 解析完成时明确说明知识已启用；仍在处理时说明当前状态，不宣称已经可检索。
- 解析失败时展示后端 `error_message`，并说明可在用户确认后重新解析。
- 编辑或删除完成后展示目标标题和 ID；不要输出 API Key、完整请求头或内部堆栈。
- 达到轮询上限时提供 `knowledge_id`，方便用户稍后继续查询。

## 错误处理

- `400`：检查必填字段、JSON 格式、`status` 枚举和 URL 格式。
- `401`：确认 `DEEPKNOW_API_KEY` 是否有效。
- `403`：确认知识库权限；飞书导入还需确认 OAuth 和文档访问权限。
- `404`：确认知识库或知识条目 ID 是否存在。
- `413`：减小文件体积或拆分文档。
- `5xx`：可安全重试一次；仍失败则停止并向用户说明。
- 解析任务失败：以 `GET /knowledge/:id` 返回的 `error_message` 为准，不自动调用 reparse。
