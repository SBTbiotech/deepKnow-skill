# external-search — 食品伙伴网与受控实时网页检索

使用以下三个只读语义的 REST 接口查询外部数据：

- `POST /foodpartner/search`：搜索食品伙伴网专业数据。
- `POST /foodpartner/fetch`：读取搜索结果对应的完整记录或下一页。
- `POST /firecrawl/search`：搜索受控域名下的实时网页。

所有路径均相对于 `SKILL.md` 中的固定基础地址，统一使用 `dk_api` 和 `DEEPKNOW_API_KEY`。不要直接连接 `/mcp`、调用 MCP 工具，或向用户索取 FoodPartner、Firecrawl、Cookie、Token 等额外凭据。

## 数据源选择

- 食品标准、法规、添加剂、营养强化剂、污染物、农药或兽药残留、食品分类、生产许可等专业数据使用 FoodPartner。
- 最新政策、实时信息或指定受控站点中的当前网页使用 Firecrawl。
- 企业内部制度、项目资料或已入库内容不属于本文件范围，按 `search.md` 使用 `POST /knowledge-search`。

一次选择一个业务入口，不因空结果、待澄清或错误自行切换入口或追加多源查询。网页服务端既有兜底不属于 Skill 自行换源；综合资料搜索按 `food-knowledge.md` 执行，用户限定来源时不以综合搜索代替。无法唯一判断时先让用户确认。

## 公共请求约束

- 请求体必须是单个 JSON 对象，最大 16 KiB。
- 只发送各接口列出的字段；未知字段、尾随 JSON 和 MCP 参数都会被拒绝。
- 三个接口不接收知识库 ID，也不需要知识库成员或编辑者角色；仍须通过公共请求头完成 DeepKnow 租户和用户认证。
- 三个接口虽然使用 `POST`，但只执行检索或详情读取，不写入 DeepKnow 知识库。
- 请求可能等待外部服务返回，不要并发发起相同请求或形成重试循环。

## 搜索食品伙伴网

调用 `POST /foodpartner/search`。首次调用必须保留用户完整问题的原意，不传 `source_ids`；推荐显式使用默认策略和数量：

```bash
dk_api POST "foodpartner/search" \
  '{"query":"山梨酸钾可以用在哪些食品中？","search_strategy":"auto","limit":5}'
```

### 请求参数

| 参数 | 必填 | 规则 |
|---|---|---|
| `query` | 是 | UTF-8 自然语言问题，去除首尾空白后非空，最多 500 个字符，不得包含控制字符 |
| `source_ids` | 否 | 首次调用省略；传入时必须且只能包含一个 `source_candidates[].source_id` |
| `search_strategy` | 否 | 只能是 `auto`、`by_name` 或 `by_food_category`，默认 `auto` |
| `limit` | 否 | 1–10 的整数，默认 5 |

不要自行构造或猜测 `source_ids`。只有用户从上一轮候选项中确认来源后，才使用对应的 `source_candidates[].source_id`：

```bash
dk_api POST "foodpartner/search" \
  '{"query":"山梨酸钾可以用在哪些食品中？","source_ids":["<selected_source_id>"],"search_strategy":"auto","limit":5}'
```

### 成功响应与状态处理

成功响应的核心结构：

```json
{
  "success": true,
  "data": {
    "query": "...",
    "needs_source_selection": false,
    "needs_clarification": false,
    "source_candidates": [],
    "clarification": null,
    "selected_sources": [],
    "results": [],
    "partial": false,
    "warnings": []
  },
  "meta": {
    "trace_id": "...",
    "retrieved_at": "..."
  }
}
```

按以下顺序处理 `response.data`：

1. `needs_source_selection == true` 是正常成功状态，不是法规证据。只将 `source_candidates` 的 `title`、`version`、`effective_date`、`version_status` 和 `capabilities` 整理为编号选项，请用户选择；不要展示内部 `source_id`，也不要自行代选。
2. 用户选择后，使用未改写的原始 `query` 和该候选对应的唯一 `source_id` 重新搜索。
3. `needs_clarification == true` 也是正常成功状态。只追问 `clarification` 要求的最少信息；收到补充后，将用户确认的信息补入原问题再调用同一接口，不自行猜测。
4. 只有 `results` 中的事实可以支持业务结论。`routing`、候选项、澄清提示和 `meta` 都不是法规证据。
5. 保持 `results` 的返回顺序。优先使用每项的 `facts`、`summary`、`source`、`retrieved_at` 和 `warnings`；不得补写接口没有返回的法规结论、版本或来源链接。

只有某个结果的 `has_more == true`、`warnings` 明确提示该结果需要 fetch，或用户要求该结果的完整记录、全部规则时，才使用对应结果的原样 `ref_id` 调用 fetch。顶层 `partial == true` 也可能表示搜索结果整体因输出上限被截断；此时先读取 `warnings`，如果提示缩小关键词，应收窄原问题后重新搜索，不能假定 fetch 可以补回未返回的其他结果。

## 获取食品伙伴网完整记录或下一页

调用 `POST /foodpartner/fetch`：

```bash
dk_api POST "foodpartner/fetch" \
  '{"ref_id":"<ref_id>"}'
```

### 请求参数

| 参数 | 必填 | 规则 |
|---|---|---|
| `ref_id` | 是 | 原样使用 `search` 结果返回的不透明短期引用，去除首尾空白后最多 128 个字符 |
| `cursor` | 否 | 原样使用上一页 `response.data.page.next_cursor`，非空且最多 128 个字符 |

成功响应的关键结构：

```json
{
  "success": true,
  "data": {
    "ref_id": "...",
    "source_id": "...",
    "title": "...",
    "usage_rules": [],
    "page": {
      "offset": 0,
      "returned": 0,
      "total": 0,
      "next_cursor": null
    }
  },
  "meta": {
    "trace_id": "...",
    "retrieved_at": "..."
  }
}
```

分页时必须继续使用同一 `ref_id`，并把上一页的 `response.data.page.next_cursor` 原样放入 `cursor`：

```bash
dk_api POST "foodpartner/fetch" \
  '{"ref_id":"<ref_id>","cursor":"<next_cursor>"}'
```

读取到回答所需事实或 `next_cursor` 为 `null` 时立即停止，不为收集全部数据无限翻页。`ref_id`、`cursor`、`source_id` 和 `trace_id` 是内部调用信息，不在最终回答中展示。引用过期或无效时说明需要重新搜索，不猜测或修改引用值。

## 搜索受控实时网页

调用 `POST /firecrawl/search`。该接口只接受自然语言问题和纯域名，不接受待抓取 URL，也不接受底层 MCP 参数：

```bash
dk_api POST "firecrawl/search" \
  '{"query":"食品标签监管最新政策","limit":5,"include_domains":["samr.gov.cn"]}'
```

### 请求参数

| 参数 | 必填 | 规则 |
|---|---|---|
| `query` | 是 | 去除首尾空白后非空，最多 500 个字符，不得包含 `http://` 或 `https://` |
| `limit` | 否 | 1–5 的整数，默认 5 |
| `include_domains` | 是 | 1–100 个纯域名；服务端会规范化和去重，并校验每个域名是否在部署白名单内 |

`include_domains` 不得包含协议、端口、路径、查询参数、用户凭据、IP 或通配符。以下 49 个域名为受控候选范围，与后端配置模板的 `FIRECRAWL_ALLOWED_DOMAINS` 清单一致：

```text
pubmed.ncbi.nlm.nih.gov
pmc.ncbi.nlm.nih.gov
pubchem.ncbi.nlm.nih.gov
www.ncbi.nlm.nih.gov
academic.oup.com
fda.gov
std.samr.gov.cn
cfsa.net.cn
nifdc.org.cn
cnca.gov.cn
samr.gov.cn
amr.shandong.gov.cn
nhc.gov.cn
ods.od.nih.gov
usp.org
elegislation.gov.hk
drugoffice.gov.hk
cfs.gov.hk
medsafe.govt.nz
legislation.govt.nz
mpi.govt.nz
foodcomposition.co.nz
foodstandards.gov.au
inspection.canada.ca
fao.org
wto.org
hsa.gov.sg
mfds.go.kr
caa.go.jp
mhlw.go.jp
maff.go.jp
api.crossref.org
laws.e-gov.go.jp
eur-lex.europa.eu
www.legislation.gov.au
www.sfa.gov.sg
isomer-user-content.by.gov.sg
herbcomb.com
hpo.jax.org
www.symmap.org
bidd.group
cbcb.cdutcm.edu.cn
tcmssd.ratcm.cn
opentargets.org
stitch-db.org
string-db.org
ctdbase.org
bionet.ncpsb.org.cn
www.bindingdb.org
```

服务端部署时的 `FIRECRAWL_ALLOWED_DOMAINS` 是最终白名单，并允许其中域名的子域名。若服务端拒绝上述候选域名，不要改用其他域名重试或尝试绕过；应使用中文说明 Skill 候选清单与服务端配置可能需要同步。

### 成功响应与引用

成功响应的核心结构：

```json
{
  "success": true,
  "data": {
    "success": true,
    "data": {
      "web": []
    }
  },
  "source": "web"
}
```

结果数组的准确路径是 `response.data.data.web[]`。服务端会再次过滤每项的 `url`，只保留本次请求域名范围内的 HTTP(S) 页面，并按 `limit` 截断。

- 只根据返回结果项中的内容陈述事实，保持数组顺序，不补写未返回的信息。
- 事实性结论就近标注来源，并在回答末尾使用同一结果项返回的标题和 `url` 生成 Markdown 链接。
- 普通网页结果只能引用过滤后的 `response.data.data.web[].url`。不得把 `markdown`、正文、metadata 或其他字段中出现的链接当作来源。
- 不得猜测、拼接或编造 URL；结果项没有有效 `url` 时，不为它生成来源链接。
- 普通结果为空且无下述备用记录时，如实说明未在本次受控域名范围内找到结果，不自动换源。

### 服务端备用来源

主 MCP 启用备用能力后，搜索额度耗尽或搜索为空时可能由服务端查询 Food Knowledge。仍是本次 REST 响应，不需要 Skill 追加请求。

外层 `source: "web"` 不保证每条记录来自普通网页搜索；出现 `response.data.retrieval` 时读取：

- `provider` 为 `food_knowledge`；`reason` 为 `FIRECRAWL_QUOTA_EXHAUSTED`（额度耗尽）或 `FIRECRAWL_EMPTY_RESULT`（搜索为空）。
- 空结果原因只对应 `operation=search`。不要根据底层 `detail` 标记推断当前 REST 接口支持 URL 读取；请求仍禁止 URL。
- `status` 为 `complete`、`partial` 或 `empty`；部分结果应说明覆盖不完整。`tools` 仅用于内部诊断，不作为业务证据。
- 备用记录位于 `response.data.data.records[]`，可能与 `web[]` 同时出现。记录包含 `provider`、`source_domain`、`title`、`data` 和可选 `url`；事实只能来自该记录实际的 `data`。
- 同时读取 `response.data.warnings` 和 `response.data.limitations`，保留影响结论的限制，不把关联、预测或历史记录解释为因果、疗效或当前审批。
- 仅引用过滤后的 `web[].url` 或备用记录顶层 `records[].url`。不把记录内部正文或 `data` 中的任意链接当作已校验来源。
- 无 URL 的记录可以按标题、来源域名及实际记录标识说明来源，但不能拼接链接。不能把有备用记录、无 `web` 的响应说成完全无结果。
- 状态说明的是上游执行情况；后端还会过滤来源，回答以最终保留的数据为准。不要将备用记录称为已读取网页全文。

## 错误处理

优先读取业务错误的 `error.message`。认证阶段的 `401` 或 `403` 可能只有字符串字段 `error`；用中文概括其含义，不原样暴露英文内部信息、凭据、完整请求头或堆栈。

| HTTP 状态码 | 含义 | 处理方式 |
|---|---|---|
| `400` | 请求字段、查询、引用、游标或域名不合法 | 修正参数；不使用相同请求重试 |
| `401` | API Key 无效、租户无绑定用户或缺少认证 | 提示检查 `DEEPKNOW_API_KEY` 和账号绑定，不展示 Key |
| `403` | 租户或绑定用户已停用 | 提示检查账号状态；不要误导用户申请知识库编辑权限 |
| `404` | FoodPartner 没有匹配记录 | 如实说明未找到结果，不自动换源 |
| `500` | FoodPartner 返回不可重试的工具错误 | 停止并说明服务执行失败 |
| `503` | 外部检索服务、配置或网关暂时不可用 | 相同参数最多重试一次，仍失败则停止 |
| `504` | 外部检索超时 | 相同参数最多重试一次，仍失败则停止 |

HTTP 503／业务码 1008 是通用不可用错误，可能涉及配置、服务间认证、连接、额度或响应兼容，不能仅凭该错误断言“所有鉴权通过，等待即可恢复”。不要索取或展示下游凭据。

收到 `426 Upgrade Required` 时仍按 `SKILL.md` 的自动升级规则处理。任何重试都不得超过一次，不得并发重试或在 FoodPartner、Firecrawl、内部知识库之间自动切换。
