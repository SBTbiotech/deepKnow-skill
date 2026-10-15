# food-knowledge — 外部资料搜索与证据读取

通过后端 REST 复用独立主 MCP 的已接入来源。普通问题使用带 `scenario` 的搜索，再按需读取真实摘要、标准条款或政策正文。所有请求沿用 `SKILL.md` 的固定基础地址、鉴权、渠道和版本头。

## 搜索证据

调用 `POST /food-knowledge/search`：

```bash
dk_api POST "food-knowledge/search" \
  '{"keyword":"槲皮素","scenario":"literature","query_translations":{"en":"quercetin"},"limit":5}'

dk_api POST "food-knowledge/search" \
  '{"keyword":"<用户指定的食品安全标准编号或名称>","scenario":"standard","source_ids":["cn_cfsa_food_standards"],"limit":5}'

dk_api POST "food-knowledge/search" \
  '{"keyword":"食品政策","scenario":"policy","include_domains":["nhc.gov.cn"],"date_from":"<YYYY-MM-DD>","date_to":"<YYYY-MM-DD>","limit":5}'
```

| 参数 | 规则 |
|---|---|
| `keyword` | 原主题或检索词，去除首尾空白后非空，最多 200 个字符；保留用户实体与限定信息 |
| `scenario` | `general`：其他专业资料；`literature`：文献；`standard`：标准；`policy`：政策 |
| `query_translations` | 可选的 `en`、`zh` 检索词；英文来源需要时提供准确译名，保留原词，不翻译标准号、PMID、结构串等标识 |
| `source_ids` | 可选的来源范围，使用文档明确的 ID 或真实返回的 `source_id`，不凭工具名猜 ID |
| `include_domains` | 可选的纯域名范围，仅使用现有批准域名；同时指定来源时不能扩大该来源范围 |
| `limit` | 每来源数量，默认 5，1–10；各来源还受其真实工具上限约束 |
| `date_from` / `date_to` | 可选的 `YYYY-MM-DD` 发布日期范围，起始不晚于结束；按来源支持能力处理 |

已确认的代表来源：文献 `pubmed`、`ijfst`；食品安全国家标准 `cn_cfsa_food_standards`；其他中国国家标准 `cn_samr_national_standards`；卫健委食品政策 `cn_nhc_food_policy`。源代码存在不代表部署已启用；缺少工具、凭证或权限时不要改查网页冒充该源成功。

证据搜索与读取的 JSON 请求体总上限为 16 KiB；各字段上限同时受此预算约束。来源和网站较多时先缩小用户关注的范围。

中文未限定国家的食品安全标准优先 CFSA。全国标准目录不收录食品安全国家标准，不用它替代 CFSA。用户未说明标准版本或关注事项时，先展示实际候选；不能凭年份自行断言现行有效或选取有歧义的版本。

“最近”未给日期时默认近 30 个自然日（含当天），由当前用户时区计算并传入明确范围。发布日期、生效日和抓取时间分开；未知发布日期的结果不能当作已确认的近期政策。受控网页搜索没有完整日期过滤保证，需要正文时继续核验。

## 响应和证据

读取 `response.data`，新证据流程必须包含 `mode: "evidence"` 和 `schema_version: "1.0"`。缺少任一标记时停止并说明后端或 MCP 尚未支持证据检索，不把旧卡片当作摘要或正文。

核心字段：`status`、`items[]`、`source_status[]`、`warnings[]`、`limitations[]`。只解释实际返回的字段，保留接口顺序。

- `complete`：本次已执行范围正常完成，不代表全库或全部网页。
- `partial`：存在来源失败、截断或覆盖限制，保留已有可靠证据并说明缺口。
- `unavailable`：没有可靠可用分支，不把 HTTP 200 当成完成。
- `clarification`：需要补充检索词、记录类型或版本信息，先向用户澄清。
- 来源级 `empty` 表示正常零结果，和失败、输入不足、需选择版本不同；不能自行换源或无限重试。

每项可能包含 `source_id`、`record_id`、`title`、`summary`、`summary_kind`、`summary_truncated`、`url`、`links[]`、`retrieval_method`、`metadata` 和 `read`。来源元数据、日期、版本和状态缺失时明确未知，不补造。

`links[]` 使用 `kind`、`label`、`url`。`record`、`fulltext`、`attachment` 分别表示记录页、全文或附件；`source_home` 仅表示数据库或官网入口。没有具体链接时可以展示真实返回的首页，但不能用首页支持某篇文献或条款结论，也不能把首页当正文读取地址。有真实记录 ID 时仍可按 API 读取对应摘要。

## 按需读取

调用 `POST /food-knowledge/read`，定位参数只取自该项实际返回的 `read`：

```bash
dk_api POST "food-knowledge/read" \
  '{"source_id":"pubmed","record_id":"<搜索返回的真实 PMID>"}'

dk_api POST "food-knowledge/read" \
  '{"source_id":"cn_cfsa_food_standards","record_id":"<搜索返回的真实记录 ID>","content_offset":0,"content_limit":20000}'
```

读取使用 `source_id`、`record_id`，以及实际 `read` 中提供的网站范围。仅在对应来源支持时使用 `content_offset`、`content_limit` 或 `cursor`；正文默认每次 20,000 字符，最多 50,000。不要发送工具名、请求头、下游凭据或自行构造读取网址。

读取结果仍验证证据协议，使用 `items[].content`、`content_kind`、`record`、`metadata`、`pagination` 和限制。`content_kind` 区分摘要、Markdown 正文和结构化记录。正文分页有 `has_more` 时按实际 `next_offset` 或 `next_cursor` 继续，到取得所需依据或合理读取上限即停止；不能把第一页说成全文。

三个常见流程：

1. **文献**：搜索后选取准备展示的少量结果。用户要求摘要时，`summary_kind` 为摘要摘录、标明截断或完整度未知的结果应读取详情，再生成摘要。PubMed 搜索提供摘要摘录，详情提供完整结构化摘要，均不等于论文全文。中文概括忠实于已读摘要，未返回摘要时明确说明。
2. **标准**：确认标准和版本，读取官方详情及已解析附件，从实际内容摘取相关条款。引用完整标准号、实际条款号、原文短句和出处；页码不存在时不编造。PDF 解析失败或截断时报告证据不足，不能用标题或搜索描述代替原文。
3. **政策**：限定用户官网，展示真实发布日期和链接，必要时读取正文。用户要求最近发布而列表日期未知或未核验时，读取准备展示的候选以核对发布日期；只把核实在时间范围内的记录列为近期政策，其余明确标注待核实。优先已有专用详情工具，其他批准官网复用受控 scrape。公告、解读和附件分别说明，已发布不代表已生效。

FoodPartner 的来源或版本选择继续按 `external-search.md` 的独立流程；其完整结构化规则不是任意标准 PDF 全文。不要在统一检索中替它选择版本。

## 兼容资料卡片查询

不传 `scenario` 的旧 `{ "keyword": "..." }` 请求保持原资料卡片行为，可聚合已配置专业来源、食品伙伴网、受控官网和指定库内可访问、解析完成且启用的公众号文章。公众号分支不是全网公众号搜索。

旧响应为 `items`、`total`、`tool_summary` 和降级摘要；只证明找到资料，不证明已取得摘要、正文或具体数值。旧模式不支持来源过滤或详情参数，不用它满足要求原文证据的问题。

## 错误和限制

新流程仍由服务端选择 API、专用网页或批准网站；Skill 不自行并发追加另一入口。参数无效时修正，权限失败或来源停用时停止，超时和解析失败按返回状态说明；重试沿用公共规则且最多一次，不重放可能计费而执行结果不明的网页请求。API 空结果不是换源指令。

检索与读取均有预算；PDF 冷解析可能超过读取上限。超时不能说成无匹配记录或完整正文。所有引用仅使用真实返回的内容、标识和链接，不暴露凭据、内部请求信息或堆栈。
