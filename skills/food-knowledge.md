# food-knowledge — 外部资料搜索与证据读取

专业来源通过后端 REST 复用独立主 MCP，使用带 `scenario` 的搜索，再按需读取真实摘要、标准条款或政策正文。公众号文章知识库、食品伙伴网和受控网页按下方说明使用各自已有入口。所有请求沿用 `SKILL.md` 的固定基础地址、鉴权、渠道和版本头。

## 业务分类与来源说明

业务分类用于理解用户所说的来源范围，不等于接口的 `scenario` 或 MCP 功能分组。下表沿用产品的分类；同一来源可以服务多个主题，但不因此扩大用户限定范围或重复查询。

| 业务分类 | 来源范围 | 使用说明 |
|---|---|---|
| 学术文献 | PubMed（`pubmed`）、PubChem（`pubchem`）、IJFST（`ijfst`） | 论文与摘要选 PubMed、IJFST；化合物身份、结构及属性选 PubChem。化合物记录不是论文。 |
| 原料 / 成分 / 标准 | GRAS、全国标准信息公共服务平台、CFSA、食品伙伴网、中检院 NIFDC、CNCA | 区分原料监管、标准原文、结构化使用规则、标准物质公告和认证认可法规；来源 ID 见下表，食品伙伴网沿用独立入口。 |
| 公告 / 抽检 | 国家市场监管总局、山东省市场监督管理局、国家卫健委、食药监不合格食品信息 | 按全国或地方范围及公告主题选择；不合格食品限定总局抽检信息，批次和检测项目仅以实际返回内容为准。 |
| 海外法规官网 | 美国 FDA、NIH-ODS、USP、香港、新西兰、澳新 FSANZ、加拿大 CFIA、国际标准 Codex、新加坡 HSA、韩国 MFDS、日本 | 先确定国家或地区，再按法规、标签、添加剂、营养等主题选择下表中的来源；国际标准与各地法规分别说明。 |
| 植提化合物 | COCONUT（`coconut`）、NPASS（`npass`）、BATMAN-TCM（`batman`）、SymMap（`symmap`） | COCONUT 查天然产物身份与结构；NPASS 查实验活性与物种来源；BATMAN-TCM、SymMap 查成分、靶点等关联，区分预测与实验依据。 |
| 靶点 / 蛋白 / 基因 | ChEMBL（`chembl`）、BindingDB（`bindingdb`）、STITCH（`stitch`）、STRING（`string_db`）、Open Targets（`opentargets`）、HPO（`hpo`）、CTD（`ctd`） | 分别用于实验活性、结合数据、化合物与蛋白关联、蛋白网络、靶点疾病证据、表型术语、化学物质与基因疾病关联；关联不等于因果或疗效。 |
| 通路与关联 | Reactome（`reactome`） | 查通路、参与分子和生物过程，核对物种及实体标识。 |
| TCM / 天然产物专用集成知识图谱 | HerbComb（`herbcomb`）、TCM-ID（`tcmid`）、TCMKD（`tcmkd`）、TCMSSD（`tcmssd`） | 查药材、方剂、成分及关联实体；按来源实际支持的实体类型查询，保留原始证据出处。 |
| 公众号文章知识库 | 存放外部公众号文章、当前用户有权限访问的知识库 | 在相关知识库中检索已有文章，按需读取正文并提供来源；已入库不改变其外部来源属性。按下方知识库流程执行。 |

### 来源名称与检索范围

以下专业来源 ID 可用于证据接口的 `source_ids`，但不表示当前部署必然启用，也不保证支持该机构全部业务。前端卡片名称或别名不能直接作为来源 ID。表中多个 ID 是候选范围，按问题选择；没有列出专业 ID 的官网按 `external-search.md` 的受控网页流程执行，不构造 ID。

| 来源名称（前端别名） | 专业来源 ID 或已有入口 | 使用范围 |
|---|---|---|
| GRAS（`fda-gras`） | `us_fda_gras` | GRAS 通知及详情；数据库收录不等于批准。 |
| 全国标准信息公共服务平台（`national-standards`） | `cn_samr_national_standards` | 已接入的国家标准目录与详情，不承诺覆盖所有行业、企业标准。 |
| CFSA（`cfsa`） | `cn_cfsa_food_standards` | 食品安全国家标准、正文及勘误。 |
| 食品伙伴网（`foodpartner`） | `external-search.md` 的 `/foodpartner/search`、`/foodpartner/fetch` | 结构化专业规则；其来源 ID 必须取自该接口返回的候选，不使用本表的 MCP ID。 |
| 中检院（`nifdc`） | `cn_nifdc_reference_material_notices` | 已接入的标准物质相关公告，不代表全部检验方法或标准品库存。 |
| 国家认监委（`cnca`） | `cn_cnca_regulations` | 认证认可相关法规。 |
| 市场监管总局（`samr`）、食药监不合格食品信息（`samr-unqualified-food`） | `cn_samr_food_safety` | 前者包含已接入的抽检公告与检验方法；后者仅取不合格食品抽检相关内容，不因共用来源扩大主题。 |
| 山东省市场监管局（`shandong-amr`） | `cn_shandong_samr_food_safety` | 山东食品抽检与相关公告。 |
| 国家卫健委（`nhc`） | `cn_nhc_food_policy` | 已接入的食品政策公告、解读及附件，不代表全部卫生健康信息。 |
| 美国 FDA（`us-fda`） | `us_fda_gras`、`us_fda_scogs`、`us_fda_color_additives`、`us_fda_substances_added_to_food`、`us_fda_food_contact_notifications`、`us_fda_bam`、`openfda` | 按 GRAS、SCOGS 安全评估、色素、食品添加物、食品接触材料、检验方法、召回或不良事件选择。 |
| 美国 NIH-ODS（`nih-ods`） | `dsld`、`nih_ods_dri` | 分别查询补充剂标签、营养素参考上限；其他官网资料按批准范围检索。 |
| 美国 USP（`usp`） | 受控官网 `usp.org` | 公开的药典与原料质量标准资料，不承诺取得付费全文。 |
| 香港（`hong-kong`） | `hk_elegislation`、`hk_drug_office`、`hk_cfs` | 分别查询法例、药品注册资料、食品安全资料。 |
| 新西兰（`new-zealand`） | `nz_legislation`、`nz_medsafe`、`nz_food_composition`；MPI 网页按批准范围使用 `mpi.govt.nz` | 分别查询法例、召回信息、食品成分和 MPI 官网资料，不推断产品准入。 |
| 澳新 FSANZ（`fsanz`） | `fsanz_standards` | 澳新食品标准目录；涉及关联法规正文时保持用户批准范围，不能自动扩大到其他来源。 |
| 加拿大 CFIA（`cfia`） | `ca_cfia_food_labelling`、`ca_cfia_incorporated_reference`、`ca_cfia_food_import_requirements` | 分别查询标签指导、引用文件、食品进口要求。 |
| 国际标准 Codex（`codex`） | 受控官网 `fao.org`、`wto.org` | 区分 Codex 食品标准与 WTO 的 TBT/SPS 资料，不把两者当作同一种标准。 |
| 新加坡 HSA（`singapore-hsa`） | `sg_hsa_vns` | 已接入的健康产品通知资料，不等于注册批准，也不自动包含 SFA。 |
| 韩国 MFDS（`korea-mfds`） | 受控官网 `mfds.go.kr` | 公开的食品及保健功能食品相关资料。 |
| 日本（`japan`） | `jp_caa_ffc`、`jp_caa_foshu`、`jp_mhlw_food_additives`、`jp_maff_jas` | 分别查询功能性标示食品、特定保健用食品、添加剂使用标准、JAS 标准。 |

其他已接入专业来源包括临床试验注册 `clinicaltrials`、`euctr`，欧盟法规 `eurlex`，澳大利亚法例 `au_legislation`，日本 e-Gov 法令 `japan_egov`，新加坡食品安全 `sg_sfa`。它们未作为独立卡片列入上述分类；用户明确需要相应资料时可在获准范围内选择，不自动补入“学术文献”“新加坡 HSA”等范围。

选择专业来源后，按资料类型设置 `scenario`，并用 `source_ids` 保留选定范围。例如，学术文献中的论文使用 `literature` 与 `pubmed` / `ijfst`，化合物资料使用 `general` 与 `pubchem`。业务分类不是同时查询全部成员的指令；无法确定必要范围时再澄清。API、网页解析等实际取数方式仍由服务端决定，不能从前端展示字段推断，也不因来源失败自行换入口。

### 公众号文章知识库

这是存放外部公众号文章的知识库。用户明确要求检索公众号资料时：

1. 按 `knowledge-bases.md` 使用用户已明确提供的知识库 ID，或通过 `GET /knowledge-bases` 的真实名称、描述确认目标库；无法确定或存在多个有歧义的候选时向用户确认，不把全部可见知识库都视为公众号知识库。
2. 按 `search.md` 调用 `POST /knowledge-search`，通过 `knowledge_base_ids` 明确限定已确认的知识库。找不到相关库、无权限或没有结果时说明情况，不省略范围回退到全库，也不改用无来源限制的综合检索。
3. 需要上下文时，按 `knowledge-bases.md` 使用命中的真实文档或分块 ID 深读。沿用知识库引用规则，保留标题、分块 ID 和原文依据；公众号名称或文章原始链接仅在实际返回时提供，不编造。

已入库不改变外部来源属性；该流程只查询库内已有文章，不表示全网公众号搜索，不构造新的 MCP 来源 ID，也不假定知识库接口支持前端公众号专用的分组或账号参数。

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
