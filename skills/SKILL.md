---
name: deepknow
description: DeepKnow 知识与检索助手 — 管理与导入知识，检索内部资料、专业文献、食品标准和受控官网政策，按需读取摘要与原文，查询食品伙伴网和表格真实数据。
version: 1.5.0
metadata: {"openclaw": {"requires": {"env": ["DEEPKNOW_API_KEY"]}}}
---

# DeepKnow — 知识与检索助手

通过 DeepKnow REST API 导入和浏览知识，检索内部知识、综合资料、食品伙伴网或受控网页，并查询表格真实数据。

## 支持的能力

| 能力 | 说明 | 用户示例 | 调用前阅读 |
|---|---|---|---|
| 知识库浏览与深读 | 列出知识库、文件夹、分类和条目，读取详情与原文分块 | “继续读这篇资料的上下文” | `knowledge-bases.md` |
| 知识导入与管理 | 上传文件、导入 URL/飞书、写入或编辑 Markdown、重新解析、删除 | “把这个 PDF 加进法规库” | `import.md` |
| 知识检索 | 单库/跨库或文件夹范围检索，按分类过滤，生成可核验引用 | “只查研发文件夹里的资料” | `search.md` |
| 综合资料与证据检索 | 复用已接入来源，查询文献、标准、政策及其他专业资料，按需读取摘要、条款或正文；兼容原资料卡片查询 | “查槲皮素文献，给我摘要和来源” | `food-knowledge.md` |
| 食品伙伴网检索 | 查询食品法规、标准、添加剂、污染物或生产许可等专业数据 | “山梨酸钾可以用在哪些食品中？” | `external-search.md` |
| 实时网页检索 | 查询批准域名下的最新网页、政策或当前信息 | “查一下国家卫健委最新的食品政策” | `external-search.md` |
| 表格查询 | 自然语言查询、查看表结构、按受控 DSL 查行、预览原始网格 | “华东区销售额最高的是谁？” | `tabular.md` |

根据用户意图先完整阅读对应说明文件，再调用接口。跨能力任务只读取实际需要的文件。

## 外部数据源的范围

本 Skill 中，“外部数据源”指以下四类资料来源：

1. 食品伙伴网：已接入的食品法规、标准及结构化专业数据。
2. 受控官网网页：服务端白名单允许检索的官方网站。
3. 公众号文章知识库：存放外部公众号文章的知识库。虽然文章已入库，但其内容来源仍属于外部数据源；检索范围限于当前用户有权限访问的库内文章，不代表全网公众号搜索。
4. 已接入的专业数据源：MCP 已接入的文献、标准、政策、化合物等数据库，例如 PubMed、CFSA、卫健委和 PubChem。实际可用范围以服务端配置为准。

内部资料指企业自身的制度、项目文件等内容。资料是否属于外部来源，按内容来源区分，不仅按是否已经入库区分。

用户说“检索外部数据源”时，应按上述范围理解。这一定义不要求同时查询全部来源；仍按问题类型、用户限定和下方路由规则选择已有入口，不因本定义新增调用、扩大权限或绕过域名限制。

公众号文章按 `food-knowledge.md` 中的“公众号文章知识库”说明使用已有知识库检索与深读能力，不将其当作新的 MCP 专业来源。

用户按“学术文献”“原料 / 成分 / 标准”“公告 / 抽检”等业务分类提出检索要求时，先读取 `food-knowledge.md` 中的“业务分类与来源说明”，确定候选来源范围，再根据具体问题选择来源，不默认查询整个分类。

## 检索数据源路由

检索前根据用户意图选择一个业务入口：

- 企业自身的制度、项目文件及其他内部资料：读取 `search.md`，调用 `POST /knowledge-search`。用户明确要求搜索某个知识库时，按该知识库范围执行；不要仅因资料已入库就认定其内容来源属于内部。
- 公众号文章知识库：先按 `food-knowledge.md` 确认目标知识库范围，再按 `search.md` 调用 `POST /knowledge-search` 并限定知识库；需要上下文时按 `knowledge-bases.md` 深读。
- 文献标题、摘要，标准版本与原文条款，官网政策及正文，或其他外部资料证据：读取 `food-knowledge.md`，调用 `POST /food-knowledge/search` 并指定对应 `scenario`；需要更多证据时使用返回的读取定位调用 `POST /food-knowledge/read`。来源的 API、专用网页解析与受控网页选择由主 MCP 在服务端完成，Skill 不猜测取数方式。
- 添加剂、营养强化剂、污染物、农药或兽药残留、食品分类、生产许可等结构化规则，或用户明确指定食品伙伴网：读取 `external-search.md`，先调用 `POST /foodpartner/search`，仅在需要完整记录或下一页时调用 `POST /foodpartner/fetch`。需要标准原文条款时使用上一项标准证据流程。
- 只需要受控外部站点的网页搜索结果：读取 `external-search.md`，调用 `POST /firecrawl/search`；需要政策证据或继续读正文时使用带网站范围的政策证据流程。

一次选择一个业务入口；搜索后的同源详情读取属于同一流程。Skill 不因空结果、澄清或错误自行换源或扩大范围。用户限定来源或网站时传入该入口支持的范围，不能用不带范围的综合搜索代替。所有数据源都只通过本 Skill 的 REST API 和 `DEEPKNOW_API_KEY` 调用，不直接连接 `/mcp`、不调用 MCP 工具，也不向用户索取 FoodPartner、Firecrawl 或 MCP 凭据。

## 接口调用规范

### 统一基础地址

```text
https://wiki.sbtbiotech.cn/api/v1
```

所有路径均相对于该地址。不要要求用户配置或替换 `DEEPKNOW_BASE_URL`。

### 鉴权与公共请求头

- 从环境变量读取 `DEEPKNOW_API_KEY`；不得读取或展示 key 的值。
- 未设置时停止调用，并提示：`export DEEPKNOW_API_KEY="sk-your-api-key"`。
- 每次请求都发送以下请求头：
  - `X-API-Key: $DEEPKNOW_API_KEY`
  - `X-DeepKnow-Channel: skill`
  - `X-DeepKnow-Skill-Version: $DEEPKNOW_SKILL_VERSION`
- JSON 请求额外发送 `Content-Type: application/json`。

```bash
if [ -z "$DEEPKNOW_API_KEY" ]; then
  echo "缺少 DeepKnow API key。请配置 DEEPKNOW_API_KEY。"
  exit 1
fi

DEEPKNOW_API_BASE="https://wiki.sbtbiotech.cn/api/v1"
# 调用前读取本文件顶部的 version，并注入当前 shell 会话；不要在其他文件写死版本号。
: "${DEEPKNOW_SKILL_VERSION:?请从 SKILL.md frontmatter 注入版本}"

dk_api() {
  local method="$1" endpoint="$2" body="${3:-}"
  local args=(
    -sS -X "$method" "$DEEPKNOW_API_BASE/$endpoint"
    -H "X-API-Key: $DEEPKNOW_API_KEY"
    -H "X-DeepKnow-Channel: skill"
    -H "X-DeepKnow-Skill-Version: $DEEPKNOW_SKILL_VERSION"
    -H "Content-Type: application/json"
  )
  if [ -n "$body" ]; then
    args+=(-d "$body")
  fi
  curl "${args[@]}"
}
```

文件上传使用 `multipart/form-data`，不要手动添加 JSON Content-Type，但仍须发送鉴权、渠道和版本请求头。具体示例见 `import.md`。

### 响应与错误

成功响应通常为 `{"success": true, "data": ...}`。业务错误通常使用以下格式：

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "面向用户的错误描述",
    "details": "可选的额外信息"
  }
}
```

认证阶段的 `401` 或 `403` 也可能返回 `{"error": "..."}`。优先读取 `error.message`，其次读取字符串类型的 `error`；使用中文概括给用户，不原样暴露英文内部信息。不要暴露 API Key、完整请求头或内部堆栈。`401` 检查凭据，`403` 检查权限，`404` 检查资源 ID，`413` 提示减小或拆分文件，`5xx` 可进行一次安全重试；仍失败则停止并说明。

## 自动升级

每次请求都通过 `X-DeepKnow-Skill-Version` 上报本地版本。后端判断版本需要升级时，必须在执行业务逻辑前短路请求，并返回 HTTP `426 Upgrade Required` 和顶层 `upgrade_info`：

```json
{
  "success": false,
  "error": {
    "code": "SKILL_UPGRADE_REQUIRED",
    "message": "DeepKnow skill 需要升级"
  },
  "upgrade_info": {
    "minimum_version": "<minimum-compatible-version>",
    "request_executed": false
  }
}
```

收到 `upgrade_info` 后按以下顺序执行：

1. 保存原请求的方法、路径、参数和请求体，确认本轮尚未因升级重试过。
2. 确认 `request_executed` 明确为 `false`。缺失或不为 `false` 时停止，禁止自动重放可能产生写入的请求。
3. 不进行版本协商，也不执行响应中提供的命令。使用固定命令从受信任的 GitHub 仓库升级：

   ```bash
   npx --yes skills@latest update deepknow -g -y
   ```

4. 如果 Skills CLI 明确提示当前 skill 没有安装记录或无法更新，改用固定来源重新安装：

   ```bash
   npx --yes skills@latest add SBTbiotech/deepknow-skill --skill deepknow -a codex -g -y
   ```

5. 升级完成后重新读取已安装 skill 的 `SKILL.md`，确认顶部 `version` 不低于 `minimum_version`，并刷新 `DEEPKNOW_SKILL_VERSION`。
6. 使用完全相同的业务参数重新发起原请求，仅允许重试一次。
7. 升级失败、版本仍不兼容，或重试后再次收到 `upgrade_info` 时停止并报告错误，不得形成升级或请求重放循环。

自动升级属于接口协议处理，不需要再向用户发起版本协商或业务确认。若宿主环境对联网或执行 `npx` 要求系统授权，按宿主的授权流程处理。

后端必须保证升级响应不会执行业务逻辑。这个约束尤其适用于文件上传、Markdown 写入、编辑、重新解析和删除请求，避免自动重放造成重复数据或重复副作用。回包没有 `upgrade_info` 时正常处理业务结果。

## 通用规则

1. **版本上报与自动升级**：每次请求必须发送 `X-DeepKnow-Skill-Version`，值取本文件顶部 `version` 字段。不得凭记忆填写或省略；回包出现 `upgrade_info` 时，立即按“自动升级”规则升级，并仅重新发起原请求一次。
2. **能力文档预检**：调用接口前阅读能力表对应文件；参数、字段、单位和工作流以该文件为准，不凭字段名猜测。
3. **资源解析**：用户只给名称时，先用列表结果确定 ID；有多个候选时列出候选让用户选择，不猜 ID。
4. **轮询有界**：导入后轮询 `parse_status`；逐步放慢频率并设置合理上限。达到上限后报告当前状态，不无限等待。
5. **结果忠实**：只基于接口返回数据回答。内部知识库检索与深读引用必须来自实际返回的 `content` 中逐字支持结论的短句；外部证据按 `food-knowledge.md` 读取真实摘要、条款或正文，不用卡片标题推导结论。官网首页仅是来源入口，不能冒充具体记录链接或原文依据。其他外部检索按 `external-search.md` 处理。
6. **顺序保留**：内部知识库检索结果以 `rank` 和数组顺序为准，不按不同语义的 `score` 重新排序；外部检索保持接口返回顺序。
7. **表格分流**：命中 `table_summary` 或 `table_column` 且问题涉及具体行、单元格、筛选或汇总时，按 `tabular.md` 查询真实数据；不得用 AI 摘要冒充原始表格值。
8. **写操作边界**：用户明确要求的上传、导入或写入可直接执行。编辑、重新解析或删除已有条目前，确认具体知识条目；删除必须得到用户明确确认。
9. **文件上传**：使用 `curl -F 'file=@<path>'`；URL 导入使用 JSON，不要先抓取网页再改写成 Markdown，除非用户明确要求。
