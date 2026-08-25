# 契约 vs 代码 一致性审计报告 — trade-copilot

> **审计基线**
> - HEAD commit：`a27fda21a09b8aa37b9c07ee2efc714d33059780`
> - 提交日期：2026-08-24 03:08:04 +0800
> - 提交说明：`docs: PRD.md补入F7自选品种功能(C1)`
> - 工作区状态：**干净**（相对已跟踪文件）——仅 2 个未跟踪文件（`restThinking.txt`、`resume/`），均为用户暂存内容，不属于本次审计对象
> - 审计范围：只读比对，未修改任何代码或契约文件（唯一写操作是本报告文件本身）
> - 契约文档：`docs/*.md` 全部篇目，根目录 `PRD.md`/`README.md`/`AGENTS.md`，`docs/adr/0001`–`0004`，`.env.example`，`config/config.yaml`，`docs/permission-matrix.md`。排除项沿用历次：`docs/multi-agent-protocol.md`/`git-workflow.md`/`pr-review-agent.md`/`dispatch/**`、根目录会话身份文件、`.claude/**`
> - 代码范围：Go 非测试约 12,000+ 行、前端非测试 TS/TSX 约 4,400 行
> - 方法：7 个切片并行盲测取证（沿用验证有效的划分），全部不读取 `docs/audits/` 任何历史报告，独立判断当前代码/文档真实状态。本轮明确授权"深夜订阅 token 随便用"，切片方法论较往轮更彻底：全文通读而非节选、六项检查逐条实际执行、外部值优先用 WebFetch/WebSearch 核实而非停在"需人工核对"、并对上一轮报告的每个未复查条目做了针对性重新核对。全部 7 个切片证据已完整并入本报告；最后由本文档统一执行 Step 8 对账（对照 2026-08-23 报告的全部 26 项索引，及其自身"本次未复查"列表里溯及的更早期 2026-08-22 报告条目）。区间提交数：**9**（8个是上一轮会话对2026-08-23报告发现的定点修复，1个是上一轮审计报告本身）。

## 执行摘要：最该先看的 10 条

| # | 结论 | 影响 | 证据位置 |
|---|---|---|---|
| 1 | **本轮会话自己在上一轮新写的 PRD.md F7-TC3 验收用例是错的**——"删除自动加入的默认品种后重启服务→不会被重新加回来"这条断言，对它自己描述的具体场景（删除唯一一条自动 seed 的记录）而言是假的：删除后列表归零，`seedDefaultWatchlist` 判断条件只看 `len(symbols) > 0`，重启后**一定会被重新加回来**。这是新写的契约文字本身与新写的代码行为直接矛盾，不是历史遗留 | 这是本轮最高优先级问题：PRD 里一条明确的验收标准，字面执行会得到与文档承诺相反的结果——需要产品决策，二选一（改 PRD 措辞承认会被重新加回，或者改实现让"是否曾经播种过"变成持久化状态） | A1，详见下文 |
| 2 | **`docs/adr/0002` 的"从交易历史重建风控状态"承诺连续第 3 轮独立复核确认从未实现**——全仓库不存在任何按交易历史重建 `RiskState` 的代码路径，`risk_state` 表完全靠直接持久化读写维护 | 存活 3 轮以上的老问题，本轮上一次的 D1 修复曾主动把这条承诺排除在删除范围外（用户明确限定只删三块过时实现细节），但没有同步标注这条被保留下来的承诺本身依然是假的 | D2，详见下文 |
| 3 | **上一轮"删除 ADR-0002 三块过时内容"的修复本身留了一个新缺口**——修复时写的编者按语声称三块被删内容"具体设计见 ADR-0003 全文"，但其中"返回格式"（`{allowed, results, warnings}` 响应信封）这部分内容 ADR-0003 通篇不含，唯一记录处只有 Go 源码注释，编者按语的覆盖范围声明过于乐观 | 这是本轮会话上一次自己动手的修复第一次被审计出遗留问题，说明"删除过时内容"这类操作本身也需要下一轮验证 | D4（新发现），详见下文 |
| 4 | **CI 里 `golangci-lint-action@v6` 和仓库自己的 `.golangci.yml`（`version: "2"`）互相不兼容**——`golangci-lint-action@v6` 官方文档只声明支持 golangci-lint v1.14.0–v1.6x，v2 需要 action v7 起；而 `.golangci.yml` 第一行已经是 v2 专属 schema，`golangci-lint` 本体当前最新版是 v2.13.1。`coding-standards.md` "CI 必须通过 golangci-lint" 这条承诺在当前配置组合下随时可能因为版本不兼容而无法稳定兑现 | 这是本轮唯一一条经**联网核实**、且**随时可能真实炸掉 CI**的配置问题，性质上和上一轮的 A1（trivy-action 版本号错误）是同一类"从未真正验证过的 pinned 版本" | A4，详见下文 |
| 5 | **PRD.md F2-TC3 期望的连续亏损冷却文案/时长，与代码实际产出、`config.yaml` 实际种子值三方互不一致**——PRD 期望"连续亏损，需冷却 2 小时"，代码实际产出"连续亏损已达 %d 笔，强制休息至 %s"（绝对时刻），`config.yaml` 实际种子值是 24 小时（不是 2 小时） | 这条文案/时长上一轮的 A4 修复只改了代码格式和测试，没有回头检查 PRD 验收用例文字是否还对得上——一个照 PRD 理解产品行为的人会得到完全错误的期望 | A2，详见下文 |
| 6 | **PRD.md 两处断言"违反任何规则均被拦截"，被 `position-limit`/`daily-loss-limit` 的 warn 级结果直接证伪**——这两条规则在 80%~100% 区间只警告不拦截，`riskService.Check` 明确让 warn 级不影响 `Allowed` | 这是"所有/任何"这类普遍性主张里被证伪的典型案例——warn 级机制是刻意设计（给用户留余地），但 PRD 的绝对化措辞从未反映这个设计 | A3，详见下文 |
| 7 | **`docs/security-review.md` §六检查清单仍保留"审计日志定期清理（90天）"一项，与同文档已被上一轮 A15 改写的 §5.4 正文直接矛盾**——正文已经明确说没有任何自动清理，清单条目是修复正文时遗留的孤儿勾选项 | 存活 2 轮以上，这类"正文改了、清单没跟着改"的孤儿条目最容易被后来者当作待办清单直接执行，产生"文档说要做但其实早就决定不做"的混乱 | B9，详见下文 |
| 8 | **`docs/architecture.md` §四数据库设计完全没有 `klines_15m`/`kline_environment` 两张表的结构说明**——这是全项目唯一的行情物理存储表，本轮首次被明确坐实（此前两轮报告都因切片焦点不同未专门核对这一条） | 一个照着这份文档理解数据库设计的人，完全不会知道 K 线数据实际存在哪张表、结构是什么 | C5，详见下文 |
| 9 | **`docs/architecture.md` §八未说明：对 1h/4h/1d/1w 而言，历史聚合路径和 WS 原生推送路径是两套完全独立、代码零复用、数据质量把关标准不同的实现**——本轮首次用代码交叉核实坐实（历史/REST 路径有 `isPlausibleKline`+`isContinuousWithPrevClose` 两层过滤，WS 原生推送对非 15m 周期完全不落库不过滤，只靠前端兜底） | 存活 2 轮以上但此前一直停留在"疑似"阶段，本轮是首次有代码级证据链完整坐实这条派生来源问题，容易让人误以为两条路径数据质量一致 | C6，详见下文 |
| 10 | **`internal/handler/trade_handler.go` 触发本地下单限流（429）时用的错误码 `ErrRequestTimeout`(1002)，跟 `docs/error-codes.md` 给 1002 的定义"上游服务响应超时"语义完全不符**——1xxx 段没有专门的"本地限流拒绝"错误码，`ErrExchangeRateLimited`(3002) 又被文档明确限定为"OKX 返回 429"场景，不适用本地限流器 | 前端如果按错误码语义分支处理，会把"你自己点太快了"误判成"服务器网络超时"，引导用户做错误的重试行为 | A5，详见下文 |

**总体判断**：本轮的核心信号是——**上一轮会话密集修复的 10 个问题里 9 个经独立盲测复核确认修复扎实**（trivy-action 版本号、config.yaml 示例、trade_handler.go 拆分、CORS 白名单、私有 WS 频道文档、milestones.md 周报表述、环境变量表补全、`RecordingGenerating`/`RecordingFailed` 死代码删除，均确认 RESOLVED）；但同时暴露出一个值得警惕的新模式：**本轮会话自己刚写完的两处新内容（PRD F7-TC3、ADR-0002 的 D1 编者按语）都被证实存在缺陷**——这说明"契约审计→定点修复"这个循环本身的产出，也需要下一轮审计去验证，不能假设"刚改完的一定是对的"。此外，本轮首次系统性验证了两个此前一直停留在"未专门核对"状态的历史疑点（`klines_15m` 表结构缺失、历史聚合与 WS 原生推送两条独立管道未文档化），均被坐实为真实存在的 C 类缺口。

## 系统性模式

| 模式 | 涵盖条目 | 性质 |
|---|---|---|
| **P-E（新模式）：本轮修复的产出本身在下一轮就被发现有缺陷** | A1（PRD F7-TC3，本轮会话自己上一次写的验收用例）、D4（ADR-0002 D1 编者按语覆盖范围过度声称） | 系统性：这是三轮报告以来第一次出现"上一轮刚修好的东西，这一轮审计就发现新问题"——提示"审计发现→立刻定点修复→不做下一轮验证"这个流程本身有盲区，修复动作应该被当作新代码一样接受下一轮审计，而不是默认已经解决 |
| **P-A''（延续）：pinned 外部版本号从未真正验证过** | A4（golangci-lint-action v6 vs .golangci.yml v2 schema） | 系统性：跟上一轮的 A1（trivy-action 版本号）是同一类问题的第二个实例——`.golangci.yml` 本身升级到 v2 schema 时，没有人回头检查驱动它的 CI action 版本是否还兼容 |
| **P-B（延续）：文档只在被专门点名的那一行修，同一片区域的其它内容持续腐化** | B9（安全检查清单孤儿条目）、B2/C4（ADR-0001多处过时内容）、B8（security-review.md Dockerfile示例，与已修复的deployment.md同款示例走向分叉） | 系统性：贯穿全部四轮审计的最大模式。本轮新增一个值得注意的具体子案例——同一个 Dockerfile 描述在 `deployment.md` 里已经修复成三阶段真实版本，但 `security-review.md` 里的同类示例完全没有跟着更新，两份文档现在对"同一个真相"给出矛盾版本 |
| **P-C（延续）：新增能力实现完成和写进契约之间持续脱节** | C7（system/status 端点未入PRD，与已解决的 watchlist-未入PRD 是同一模式在不同功能点的复现）、C1（price-deviation-guard 规则未入PRD）、C2/C3（ADR-0001 的错误处理机制/交易所专属方法扩展模式未被记录） | 系统性：C1（watchlist未入PRD）刚在上一轮被解决，这一轮立刻在另一个功能点（system/status）发现同款问题，说明这不是某个功能点的偶然疏漏，而是"新功能上线时补文档"这个环节本身缺乏机制保障 |
| **老问题仍在（跨轮存活）** | B4→B2本轮（ADR-0001 Interval描述过期，存活3轮）、D3（architecture.md"待对接"矛盾标题，存活多轮）、C10（db.go行号引用过期，存活2轮且本轮进一步漂移） | 这些是本轮独立复核确认仍然成立、且从未被任何一轮会话触碰过的条目 |

## 覆盖归属

| 代码目录 | 负责切片 |
|---|---|
| `internal/engine/risk/**`, `service/{risk,trade}_service.go`, `handler/{risk,trade}_handler.go`, `repository/{risk,trade}_repo.go`, `domain/{risk,order}.go`, ADR-0002/0003 | 切片1 |
| `internal/exchange/**`, `domain/exchange.go`, ADR-0001 | 切片2 |
| `service/{market,analysis}_service.go`, `repository/kline_repo.go`, `handler/{market,analysis}_handler.go`, `domain/kline.go`, `web/src/components/{KlineChart,KlinePanel}/**`, `wire.go`/`market_wire.go` K线部分 | 切片3 |
| `service/{recorder,summary}_service.go`, `recorder_pipeline.go`, `obs/**`, `llm/**`, `handler/{recorder,summary}_handler.go`, `cron.go` | 切片4 |
| `service/watchlist_service.go`, `handler/{watchlist,system,ws}_handler.go`, `web/src/components/{SystemStatusBanner,Watchlist}/**`, `docs/permission-matrix.md` | 切片5 |
| `internal/app/*.go`全部, `handler/{router,middleware,common,order_validation}.go`, `docker-compose.yml`, `Dockerfile`, `nginx.conf`, `.github/workflows/*.yml`, `config/config.yaml`, `.env.example`, `.golangci.yml` | 切片6 |
| `api/swagger/**`, `web/src/lib/{api,http}.ts`, 其余前端组件（TradePage/Positions/RuleToggles/TradePanel/Summary）, 全部handler swagger注解 | 切片7 |
| `internal/testutil/**` | **无人认领**——纯测试基础设施 |
| `data/`, `whisper-models/`, `memory/`, `resume/`, `restThinking.txt` | **不在审计范围** |
| `docs/multi-agent-protocol.md`等协作工具文档, `.claude/**` | **超出范围**——历次指示排除 |

## 外部定义值清单（本轮联网核实汇总）

**OKX（v5 API / WebSocket）**——切片2/3再次核实：REST签名prehash/时间戳格式（毫秒ISO8601）、WS登录签名prehash/时间戳格式（秒级Unix，与REST不同）经WebFetch官方文档核实结构正确（一处网上摘要给出的"官方示例签名值"经本地Python复算证伪，判定为二手来源转录错误，不影响结构本身核实结论）；`intervalToBar`大小写映射、WS端点域名/路径、candle频道走business端点、心跳<30s约束、100x杠杆上限，均经WebSearch交叉验证一致；**新发现疑点**：`okxMaxLimit=100`同时用于`GetKlines`（实时）和`GetHistoryCandles`（历史），多个二手来源显示OKX现行`/market/candles`实时K线上限可能已是300（历史K线仍是100），代码可能因循旧值不必要地把实时请求砍到100——未能直接核实到官方原文确认，置信度中等，见下方"需要人工核对"表；1D/1W分桶边界（UTC+8参考时区，16:00 UTC日界/周日16:00 UTC周界）经OKX官方帮助页原文+第三方issue双重印证，与`okxDailyBucketOffsetMs`/`okxWeeklyBucketOffsetMs`常量完全吻合。

**GitHub Actions 第三方 action**：`aquasecurity/trivy-action@v0.28.0`（上轮修复，本轮切片6重新联网核实仍然有效，无需改动）；`golangci-lint-action@v6`——**本轮新增外部值，联网核实存在版本兼容风险**，见A4；`actions/checkout@v4`/`setup-go@v5`/`setup-node@v4`——均已落后当前最新主版本但仍可用，非阻断性问题；`swag@v1.8.1`——经Go module proxy核实该版本仍可正常`go install`。

**obs-websocket v5协议**：本轮切片4直接WebFetch官方`protocol.md`逐值核实opcode常量（`Hello=0/Identify=1/Identified=2/Reidentify=3/Event=5/Request=6/RequestResponse=7/RequestBatch=8/RequestBatchResponse=9`），代码完全一致，无偏差——此前两轮报告这条一直停留在"需要人工核对"状态，本轮首次坐实。

**whisper.cpp CLI**：Dockerfile固定`WHISPER_CPP_VERSION=v1.9.2`，本轮切片4直接WebFetch该确切tag下的`examples/cli/README.md`核实全部6个用到的CLI参数（含容易被怀疑拼写错误的`--suppress-nst`）均真实存在——此前两轮一直停留在"需要人工核对"状态，本轮首次坐实。

**Swagger/OpenAPI 2.0规范**：切片7脚本化全文扫描`swagger.json`，确认零处非法`type`声明，与上轮结论一致。

## 需要人工核对的外部值

| 值 | 出处 | 为什么可疑 | 推测的正确值 | 什么能定案 |
|---|---|---|---|---|
| `okxMaxLimit=100`是否应该给`GetKlines`（实时K线）单独提到300 | `okx/market.go:22-23` | 二手来源（含ccxt仓库issue）显示OKX `/market/candles`当前实时K线上限可能已经是300，只有`history-candles`历史K线仍是100；WebFetch未能从OKX官方大型单页文档提取到具体锚点文字，只能靠二手来源交叉印证 | 无法在本轮确认，中等置信度怀疑 | 人工直接打开OKX官方`/api/v5/market/candles`文档页确认当前`limit`参数上限，若确为300则`GetKlines`应该拆出独立常量而不是复用`okxMaxLimit` |
| `history-candles`端点精确限流阈值 | `okx/market.go:25-28`、`security-review.md:273-274` | 代码/契约均诚实标注"没有实测出精确阈值"；WebSearch只找到笼统的"public端点20请求/2秒"规则，未找到专门针对该端点的官方页面 | 无法确认是否被笼统规则覆盖 | 联系OKX API支持或做压测实测边界 |

（本轮已解决的历史疑点：`intervalToBar`16档映射、REST/WS签名preHash结构、OBS WebSocket opcode、whisper.cpp CLI参数——均已在本次"外部定义值清单"中标注为核实通过，不再需要人工核对。）

## 普遍性主张核对

| 契约声明 | 出口数 | 满足数 | 逃逸的是哪几个 |
|---|---|---|---|
| `PRD.md`："违反任何规则均被拦截并记录" | position-limit/daily-loss-limit/stop-loss-required/consecutive-loss-rest/price-deviation-guard 五条规则的全部结果路径 | 3/5（三条纯block规则符合） | position-limit、daily-loss-limit 的 warn 级结果不拦截，见A3 |
| `security-review.md`："所有API端点必须进行以下校验" | `POST /api/trade/order`、`POST /api/risk/validate`两个写端点复用同一套`validateOrderRequest` | 2/2一致（对称，历史bug已修复） | 无逃逸——正面结论 |
| `security-review.md`："所有写操作必须记录审计日志" | CreateOrder/Check/UpdateRule/RecordClosedTrade四个写路径 | 4/4 | 无逃逸——正面结论 |
| `docs/api-documentation.md`："代码即文档，永不漂移" | 8个写端点的swagger状态码声明 | 8/8（100%，逐端点重算） | 无逃逸——正面结论，但CI层面`golangci-lint`这条相关的"必须通过"承诺见A4风险 |
| `docs/architecture.md`：`WebSocket /ws/klines`"独立路径，只更新蜡烛本身" | 1h/4h/1d/1w四档周期的数据来源与质量把关是否与历史聚合一致 | 表述本身不算错，但未说明这个"独立"同时意味着数据来源、质量过滤标准也完全独立 | 见C6 |

## 派生来源核对

| 契约描述的派生 | 契约点名的输入 | 代码实际读取的来源 | 是否一致 |
|---|---|---|---|
| 1h/4h/1d/1w K线（REST/历史查询） | 从`klines_15m`本地SQL窗口函数聚合 | `kline_repo.go:getRangeAggregated`确认，聚合前经`isPlausibleKline`/`isContinuousWithPrevClose`过滤 | **一致**（切片3独立复核） |
| 1h/4h/1d/1w K线（WS推送） | architecture.md未明确点名输入，散文暗示"跟历史聚合是同一套数据、只是增量更新" | `okx.Client.SubscribeKlines`直连OKX原生`candle{Bar}`频道，逐条透传，完全不经过本地聚合/过滤 | **实为两条独立数据源**，与文档暗示的"同一套数据"存在认知落差，见C6 |
| `GetAccountEquity` | "直接用交易所自己算好的总权益作为唯一真实来源" | `account.go`调用`/api/v5/account/balance`取`totalEq`字段，独立发起请求 | **一致**（切片2独立复核） |
| `RiskState`（启动时） | ADR-0002："每次启动时从交易历史重建状态" | `risk_wire.go`只从`risk_rules`表加载规则配置，`risk_state`表直接持久化读写，无重建逻辑 | **不一致**，见D2（存活3轮） |
| `systemStatusResponse.OkxOrderStreamConnected` | "反映私有订单流连接状态，非K线公开连接" | `okx.Client.OrderStreamConnected()`读取私有频道`atomic.Bool`标志 | **一致**（切片5独立复核） |
| 周报（weekly summary） | "直接聚合原始素材，不经过每日总结二次汇总" | `TriggerWeekly`/`TriggerDaily`共用`generate()`，数据源统一是`analysis_records`表 | **一致**（切片4独立复核） |

## 全部发现索引

| 编号 | 一句话结论 | 完整条目在哪 |
|---|---|---|
| A1 | PRD F7-TC3断言与`seedDefaultWatchlist`实际行为直接矛盾（本轮最高优先级） | 下方A类，新发现 |
| A2 | PRD F2-TC3冷却文案/时长与代码产出/config.yaml种子值三方不一致 | 下方A类，新发现 |
| A3 | PRD"违反任何规则均被拦截"被warn级规则证伪 | 下方A类，新发现 |
| A4 | golangci-lint-action@v6与.golangci.yml v2 schema版本不兼容，CI随时可能失败 | 下方A类，新发现 |
| A5 | trade_handler.go限流用错误码1002(上游超时语义)，与本地限流场景不符 | 下方A类，新发现 |
| A6 | Watchlist/index.tsx 323行超过coding-standards.md的300行上限 | 下方A类，新发现 |
| A7 | （低置信，测试fixture）mocks/handlers.ts风控规则未找到mock错误码4004应为4201 | 下方A类，新发现 |
| B1 | security-review.md §3.2 OrderRequest示例缺4个现有字段+BASE-QUOTE-SWAP格式 | 下方B类，新发现 |
| B2 | ADR-0001接口签名(ctx参数)/方法清单/Interval描述均已过期 | 下方B类，=历史B4延续 |
| B3 | architecture.md引用runPeriodicGapScan的文件行号完全错误(应为market_wire.go) | 下方B类，新发现 |
| B4 | architecture.md引用EnsureHistory的行号漂移约5-6行 | 下方B类，新发现（轻微） |
| B5 | architecture.md数据库设计缺analysis_records的video_path/status列 | 下方B类，=历史(2026-08-22)B4延续，本轮首次坐实 |
| B6 | deployment.md ReadyHandler示例(OKX不可达→503)与实际(仍200)不符 | 下方B类，=历史B5延续 |
| B7 | coding-standards.md称.golangci.yml"待脚手架阶段创建"，实际已存在生效 | 下方B类，新发现 |
| B8 | security-review.md §4.3 Dockerfile示例仍是单阶段旧版，与已修复的deployment.md分叉 | 下方B类，新发现 |
| B9 | security-review.md检查清单"审计日志定期清理(90天)"与已改写的§5.4正文矛盾 | 下方B类，=历史B9延续 |
| C1 | price-deviation-guard规则(第5条风控规则)未入PRD功能清单 | 下方C类，新发现 |
| C2 | ADR-0001未提及APIError/ErrQuantityBelowMinimum错误处理机制 | 下方C类，新发现 |
| C3 | ADR-0001未提及"交易所专属方法可超出共享接口独立暴露"这一扩展模式 | 下方C类，新发现（低置信） |
| C4 | ws.go重连退避参数注释引用的"docs/adr/0004"实际不含此内容 | 下方C类，=历史C7延续 |
| C5 | architecture.md数据库设计节完全缺klines_15m/kline_environment表结构 | 下方C类，=历史(2026-08-22)C9延续，本轮首次坐实 |
| C6 | architecture.md未说明历史聚合与WS原生推送对1h/4h/1d/1w是两条独立管道 | 下方C类，=历史(2026-08-22)C10延续，本轮首次坐实 |
| C7 | GET /api/system/status端点+SystemStatusBanner前端组件未入PRD | 下方C类，新发现 |
| C8 | permission-matrix.md"WebSocket连接管理"行未区分外部客户端连接与本服务WS服务端连接 | 下方C类，新发现（低置信） |
| C9 | LLM_MODEL环境变量在docker-compose.yml里未被透传（Docker部署下设置无效） | 下方C类，新发现 |
| C10 | db.go/db_test.go注释引用security-review.md的行号(133/424)再次过期，实际141/448 | 下方C类，=历史C10延续，进一步漂移 |
| D1 | ADR-0002核心设计已被ADR-0003替换——上轮已部分删除过时内容，但遗留新问题 | 下方D类，=历史D1，部分处理 |
| D2 | ADR-0002"从交易历史重建状态"承诺从未实现 | 下方D类，=历史D2延续 |
| D3 | architecture.md §7/§8标题仍称"待对接现有录制工具"与正文自相矛盾 | 下方D类，=历史D3延续 |
| D4 | ADR-0002 D1编者按语过度声称ADR-0003覆盖"返回格式"，实际只有源码注释记录 | 下方D类，新发现（上轮修复的残留问题） |
| 测试质量附注 | WS重复IP拒绝状态码断言用t.Logf非t.Errorf，弱化了429语义的回归保护 | 下方"六项检查补充"一节 |
| 已解决/仍存在/未复查 | 2026-08-23报告全部26项+2026-08-22报告溯及条目的最新状态 | 下方「与上次审计的对照」 |

## A 类：代码偏离契约

### A1 — PRD F7-TC3断言与`seedDefaultWatchlist`实际行为直接矛盾 | 置信度：高（本轮最高优先级，新发现）
- **契约**：`PRD.md:227`——"`DELETE /api/watchlist/symbols/:id` 删除自动加入的默认品种后重启服务 → 不会被重新加回来（列表非空时不触发自动加入）"
- **代码**：`internal/app/wire.go:130-146`（`seedDefaultWatchlist`只判断`len(symbols) > 0`，无状态、无法区分"从未有过数据"与"用户删空"）；`internal/app/wire_test.go:580-587`（测试注释自认"但更常见的场景是列表清空后又重启——这种情况下确实会被重新seed，这是设计如此"）
- **判断**：TC3描述的具体场景是"删除**唯一一条**自动加入的默认品种"——这个操作执行后列表会变回空表，不是TC3断言的"非空"，下次启动`len(symbols)>0`判断为假，会**重新播种**，与TC3断言的结果直接相反。代码本身的设计（无状态判断）是一个经过深思熟虑的合理取舍（并有清晰注释说明"数据库层面无法区分两种情况"），问题完全出在PRD文字：TC3把这个已知设计取舍的其中一个具体子场景断言反了。需要产品决策：要么改PRD措辞如实描述"删除唯一一条默认品种后重启会被重新加回，这是预期行为"，要么改实现引入独立的"是否曾经播种过"持久化标志（如专门的flag字段/表）来真正实现TC3期望的语义。

### A2 — PRD F2-TC3冷却文案/时长与代码产出/config.yaml种子值三方不一致 | 置信度：高（新发现）
- **契约**：`PRD.md:188`（F2-TC3，期望"连续亏损，需冷却 2 小时"）
- **代码**：`internal/engine/risk/consecutive_loss.go:90-92`（实际产出格式"连续亏损已达 %d 笔，强制休息至 %s"，绝对截止时刻非相对时长）、`:34`（硬编码默认值2小时）；`config/config.yaml`（`cooldownHours: 24`，首次启动即写入DB成为权威值）
- **判断**：上一轮的A4修复（`a0af222`）改了代码文案格式和测试，但没有回头核对PRD.md的验收用例文字是否还对得上——这既不是文案格式对不上（能理解为改进），也不是单纯的数字漂移，是文案结构（相对时长vs绝对时刻）、默认时长（代码2h）、实际生效时长（config.yaml 24h）三方各自不同，PRD描述的是三者都不是的第四种状态。

### A3 — PRD"违反任何规则均被拦截"被warn级规则证伪 | 置信度：高（新发现）
- **契约**：`PRD.md:57`（"拦截模式：违反规则→阻止操作"）、`PRD.md:233`（"违反任何规则均被拦截并记录"）
- **代码**：`internal/service/risk_service.go:97-106`（"warn级别不影响allowed"）；`internal/engine/risk/position_limit.go:69-95`、`internal/engine/risk/daily_loss.go:87-97`（`warningRatioThreshold`逻辑，达到80%~100%区间只警告不拦截）
- **判断**：warn级机制是刻意设计——给用户在临近上限时留出自主决策空间，不是bug。但PRD两处使用了不加限定的绝对化措辞"任何规则"/"违反规则→阻止"，跟实际存在的两级（block/warn）机制矛盾。这是"契约主张全称命题、代码里有例外分支"的典型案例。

### A4 — golangci-lint-action@v6与.golangci.yml v2 schema版本不兼容 | 置信度：高（联网核实，新发现）
- **契约**：`docs/coding-standards.md:244`附近对应的"CI要求：必须通过golangci-lint"承诺；`.github/workflows/ci.yml:37-40`（`uses: golangci/golangci-lint-action@v6` + `version: latest`）、`docs/deployment.md:195-198`（同款配置的文档镜像）
- **代码**：`.golangci.yml:1`（`version: "2"`——v2专属schema）
- **判断**：经WebFetch核实，`golangci-lint-action@v6`官方README的兼容表只到golangci-lint v1.6x，v2需要action v7起才支持；`golangci-lint`本体当前最新版是v2.13.1（2026-08-20发布）。也就是说仓库自己的`.golangci.yml`已经按v2 schema编写，但驱动它运行的CI action版本理论上只认v1——这不是"文档滞后"，是两处配置互相矛盾，随时可能因为`golangci-lint-action@v6`解析v2配置失败或behavior不一致导致CI红灯，且CI通过与否目前并没有专门测试守护这个组合的兼容性。

### A5 — trade_handler.go限流错误码1002语义错配 | 置信度：中高（新发现）
- **契约**：`docs/error-codes.md:43`（"1002｜请求超时｜上游服务响应超时"）、`docs/error-codes.md:65`（"3002｜交易所限流｜OKX 返回 429 Too Many Requests"，明确限定交易所场景）
- **代码**：`internal/handler/trade_handler.go:82`（`writeError(w, http.StatusTooManyRequests, ErrRequestTimeout, "下单过于频繁，请稍后再试")`，触发本地下单限流时使用`ErrRequestTimeout`(1002)）
- **判断**：1002在文档里的定义是"等待上游响应超时"，语义上是"我方等待OKX响应超时"，而这里的场景是"客户端本地请求过于频繁被本地限流器拒绝"，是完全不同的两件事。1xxx段没有专门的"本地限流拒绝"错误码，`ErrExchangeRateLimited`(3002)又被文档明确限定给"OKX返回429"场景不适用。前端如果按`code`语义分支处理（`ValidationResult.code`字段的设计初衷正是给这种场景用的），会把"用户点太快"误判成"上游超时"，引导错误的重试/排查行为。

### A6 — Watchlist/index.tsx 323行超过300行上限 | 置信度：高（新发现）
- **契约**：`docs/coding-standards.md:303-304`（"✅ 组件文件 ≤ 300 行 / 超限 → 拆分子组件到同目录"）
- **代码**：`web/src/components/Watchlist/index.tsx`（`wc -l` = 323行）
- **判断**：跟上一轮报告A3（`trade_handler.go`超限）是同一类问题的前端实例——这条规范至今没有任何CI自动检查（无对应lint规则），单靠人工记忆遵守，正常迭代持续会有文件悄悄超限。

### A7 — （低置信，测试fixture）mocks/handlers.ts风控规则未找到mock错误码错配 | 置信度：中（测试fixture，主动降级，新发现）
- **契约**：`docs/error-codes.md:95`（"4201｜规则未找到｜规则 ID 不存在"）vs "4004｜订单被风控规则拦截｜`POST /api/trade/order`…"（完全不同语义的两个码）
- **代码**：`web/src/mocks/handlers.ts:372`（`PUT /api/risk/rules/:id`命中不存在id时mock返回`error.code: 4004`）；真实后端对应场景（`internal/handler/risk_handler.go:189-191`）返回的是4201
- **判断**：按skill方法论对测试fixture缺陷主动降级——没有任何测试断言这个具体`.code`值，属于未被覆盖到的死角，不影响当前测试通过与否，只会误导直接读mock数据做参考的开发者，不构成生产环境缺陷。

## B 类：契约落后

| # | 结论 | 置信度 | 契约↔代码证据 |
|---|---|---|---|
| B1 | `security-review.md`§3.2的`OrderRequest`示例结构体缺`Price`/`MarginMode`/`StopLossMode`/`TakeProfitMode`四个现已存在且强制校验的字段，"格式"维度只写`BASE-QUOTE`未提也合法的`BASE-QUOTE-SWAP`后缀 | 高 | `docs/security-review.md:187-204` ↔ `internal/domain/order.go:57-68`（完整10字段struct tag）、`internal/handler/order_validation.go:38`（`symbolFormatPattern`含`-SWAP`） |
| B2 | ADR-0001接口代码块与当前冻结接口有三处过期：(1)除`SubscribeKlines`外方法均无`context.Context`首参，实际全部方法都有；(2)ADR完全没有`ServerTime`/`GetAccountEquity`两个方法；(3)`Interval`被描述成"枚举：15m/1h/4h"三值封闭集合，实际是`type Interval = string`开放字符串别名，OKX侧支持16档 | 高 | `docs/adr/0001-exchange-abstraction-layer.md:21-40,56-57` ↔ `internal/exchange/interface.go:19-43`、`internal/domain/kline.go:8` |
| B3 | `architecture.md`§八"定时重扫"一节引用`runPeriodicGapScan`位于`internal/app/wire.go:130-151`，但该函数实际定义在`internal/app/market_wire.go:52-69`；`wire.go:130-151`这段行号实际对应的是完全不同的`seedDefaultWatchlist`函数体 | 中 | `docs/architecture.md:620` ↔ `internal/app/market_wire.go:52`（真实位置）、`internal/app/wire.go:130`（实际是`seedDefaultWatchlist`） |
| B4 | `architecture.md`§八"K线自愈回填机制"引用`MarketService.EnsureHistory`位于`market_service.go:389-453`，实际函数体为398-458，行号整体偏差约5-6行 | 低 | `docs/architecture.md:597` ↔ `internal/service/market_service.go:398-458` |
| B5 | `architecture.md`§四数据库设计的`analysis_records`建表SQL只列7个字段，完全没有`video_path`/`status`两列——这两列由`migrations/0002`补充，且是`domain.AnalysisRecord`/`RecorderService.Stop()`/前端`AnalysisRecord`实际在用的字段 | 高 | `docs/architecture.md:262-274` ↔ `migrations/0002_recorder_fields.up.sql:8-9`、`internal/domain/recorder.go:27-37`、`web/src/lib/api.ts:149-150` |
| B6 | `docs/deployment.md`§4.1的`ReadyHandler`示例（OKX不可达→503）与实际行为（仍返回200，`okx`字段标`unavailable`）不符——这是有意设计（DB是地基，OKX只影响部分能力），有`TestReadyHandler`子测试锁定，只是示例代码没跟上；示例调用`writeError`的第三参数传字符串，与真实签名（`code int`）不符，示例代码本身编译不过 | 高 | `docs/deployment.md:280-307` ↔ `internal/app/server.go:36-61`、`internal/app/server_test.go:79-103`、`internal/handler/common.go:50` |
| B7 | `docs/coding-standards.md`§2.7写"`.golangci.yml`配置待脚手架阶段创建"，但该文件已经存在于仓库根目录，是13行真实生效配置 | 高 | `docs/coding-standards.md:244` ↔ 仓库根目录`.golangci.yml` |
| B8 | `docs/security-review.md`§4.3「容器镜像」的Dockerfile示例是过时版本（`FROM golang:1.22-alpine AS builder`，单阶段，缺whisper-builder阶段/ffmpeg等运行时依赖），而`docs/deployment.md`§2.1的同类示例本轮已确认是跟真实`Dockerfile`同步的三阶段版本（`golang:1.25-alpine`）——两份文档现在对同一个Dockerfile给出两个互相矛盾的"真相版本" | 高 | `docs/security-review.md:307` ↔ 仓库根`Dockerfile:2`（三阶段）↔ `docs/deployment.md:54-105`（已同步） |
| B9 | `docs/security-review.md`§六"安全检查清单"仍保留`[ ] 审计日志定期清理（90 天）`一项，但§5.4（同一文档，已被上一轮A15改写）明确说明没有任何自动清理，日志留存完全交给尚未配置的Docker日志驱动`max-size`/`max-file`——这是A15修正5.4时遗留下来的孤儿条目 | 中高 | `docs/security-review.md:471`（检查清单，本轮实际行号有漂移，内容仍是这条） ↔ `docs/security-review.md:423-431`（§5.4已改写的真实情况） |

## C 类：契约未覆盖

| # | 结论 | 置信度 | 证据 |
|---|---|---|---|
| C1 | 第5条风控规则`price-deviation-guard`（价格偏离保护/防FOMO，`Locked()==true`）完全没有出现在PRD.md的F2功能清单和"六、交易规则详情"表格里——这两处都只列了4条规则 | 中高 | `PRD.md:50-58,129-137` ↔ `internal/app/risk_wire.go:35`（注册）、`internal/engine/risk/price_deviation.go`（实现+Locked）、`docs/adr/0003-rule-interface-abstraction.md:68`（ADR已记录，PRD未记录） |
| C2 | `exchange.ErrQuantityBelowMinimum`和`exchange.APIError`是接口实际依赖、被上层`errors.Is`/`errors.As`分类使用的核心错误处理机制，ADR-0001全篇未提及这两个类型或"上层如何区分交易所错误"这件事 | 中 | `internal/exchange/interface.go:13-15`、`internal/exchange/errors.go:1-25` ↔ ADR-0001全文grep"APIError"/"ErrQuantityBelowMinimum"均无命中 |
| C3 | `GetHistoryCandles`/`SubscribeOrderFills`/`OrderStreamConnected`是OKX具体类型对外暴露、但不进`exchange.Exchange`共享接口的"交易所特有方法"模式（代码注释明确说明是有意为之）。ADR-0001"风险缓解"一节只提到用`Extra map[string]any`兜底交易所差异，没提到这条实际采用的扩展路径 | 低 | `internal/exchange/okx/ws_private.go:268-270`、`internal/exchange/okx/market.go:126-133` ↔ `docs/adr/0001-exchange-abstraction-layer.md:84-86`（只提Extra字段） |
| C4 | `internal/exchange/okx/ws.go`重连退避参数注释引用"docs/adr/0004限制"作为依据，但`docs/adr/0004-multi-asset-symbol-abstraction.md`全文实际讲的是多资产符号抽象，通篇未提重连策略——引用失效，真正对得上数字的是`docs/architecture.md:134`/`docs/security-review.md:94` | 高 | `internal/exchange/okx/ws.go:16-20` ↔ `docs/adr/0004-multi-asset-symbol-abstraction.md`全文无重连相关内容 |
| C5 | `docs/architecture.md`§四《数据库设计》完整列出了trades/risk_rules/risk_state/watchlist/analysis_records/summaries六张表，但唯一的行情物理存储表`klines_15m`和辅助表`kline_environment`（`migrations/0005`定义）完全没有出现——既没建表SQL摘录也没进ER图，只在§八散文里顺带一句"写入klines_15m表" | 中 | `docs/architecture.md:199-360`（§四全节无klines_15m/kline_environment） ↔ `migrations/0005_create_klines_table.up.sql:1-24`、`internal/repository/kline_repo.go`全篇围绕这两张表 |
| C6 | 历史/REST聚合路径（`getRangeAggregated`，本地SQL窗口函数聚合，经`isPlausibleKline`/`isContinuousWithPrevClose`过滤）与WS原生推送路径（`SubscribeKlines`直连OKX原生`candle1H`/`candle4H`/`candle1D`/`candle1W`频道，逐条透传零后端过滤）对1h/4h/1d/1w是两套完全独立、代码零复用的实现——`architecture.md`§八只写"独立路径，只更新蜡烛本身"，容易被理解成"独立"仅指跟指标计算独立，未说明这个独立同时意味着数据来源和质量把关标准也完全不同 | 中 | `docs/architecture.md:566-582` ↔ `internal/repository/kline_repo.go:296-386`、`internal/exchange/okx/ws.go:189-202`+`ws_auth.go:34-42`、`internal/service/market_service.go:321-330`（仅`interval=="15m"`才落库过滤） |
| C7 | `GET /api/system/status`端点+前端`SystemStatusBanner`组件（OKX私有订单流断线提示）在PRD.md全篇未被提及——跟watchlist曾经"只写进architecture.md、PRD从未记录"（上轮C1，已解决）是同一类缺口在另一个功能点的复现 | 中 | `PRD.md`全文搜索`system/status`/`OrderStream`/`SystemStatusBanner`均无命中 ↔ `internal/handler/system_handler.go:34-50`、`web/src/components/SystemStatusBanner/index.tsx:1-55` |
| C8 | `docs/permission-matrix.md`"代码层权限矩阵"里"WebSocket连接管理"一行只标记external client层✅，handler/service/repository均❌，但`/ws/klines`（浏览器直连本服务的实时推送）的连接管理——升级/IP限流/心跳/超时——全部实现在handler层，并非"external client"专属；矩阵未区分"到OKX的WS客户端连接"和"浏览器到本服务的WS服务端连接"，字面读容易被误解为后者违规 | 低 | `docs/permission-matrix.md:30` ↔ `internal/handler/ws_handler.go:43-73`（`ipConnTracker`，纯handler层实现）对照`internal/exchange/okx/ws_private.go`（真正的external client层OKX WS客户端） |
| C9 | `LLM_MODEL`环境变量在`docker-compose.yml`的`environment:`段落里未被透传（只有`LLM_API_URL`/`LLM_API_KEY`两个LLM相关变量被传入容器）——Docker部署下用户在`.env`设置`LLM_MODEL`不会有任何效果，会静默retain默认值`deepseek-chat`；`architecture.md`环境变量表本身只写默认值，未像`DB_PATH`那样标注"仅本机直接运行生效" | 中 | `docker-compose.yml:37-71`（无`LLM_MODEL`） ↔ `internal/app/config.go:120`（确实读取该env var）、`docs/architecture.md`环境变量表（未标注限定说明） |
| C10 | `internal/app/db.go`及`db_test.go`的注释都写"`docs/security-review.md` 133/424行承诺SQLite文件chmod 600"，但当前文档里第133行、第424行内容均与chmod无关（文档在此前多次编辑中发生过行号漂移）；真正的chmod 600承诺现在位于第141行和第448行——不影响运行时行为（功能本身已有专门测试验证），纯粹是注释准确性问题，且比上一轮记录的行号（110/133/136/432）又进一步漂移了一次 | 低 | `internal/app/db.go:29-33`、`internal/app/db_test.go:100-102`（均写"133/424行"） ↔ `docs/security-review.md:130-145`（第133行实际内容）、`:418-431`（第424行实际内容）、`:141,448`（实际承诺所在行） |

## D 类：契约已失效

| # | 结论 | 置信度 | 证据 |
|---|---|---|---|
| D1 | `docs/adr/0002`核心设计已被ADR-0003替换——**上一轮已按用户明确指示删除了三块过时实现细节**（`RiskConfig`/`RiskEngine`结构体、校验流程、返回格式的代码块），替换为指向ADR-0003+真实代码文件的指针段落。本轮独立复核确认删除本身干净彻底，但发现修复动作遗留了两个新问题：(1) 编者按语对"返回格式"覆盖范围的声称过度乐观，见D4；(2) 同一份文档里未被本次修复触碰的"重建状态"承诺（"背景/风险缓解"章节，用户明确指示保留不动）依然是假的，见D2 | 高 | `docs/adr/0002-risk-engine-intercept-mode.md`（`e18a4bf`已修复主体） |
| D2 | ADR-0002"风险缓解"里"每次启动时从交易历史重建状态"这条具体承诺，从未实现也从未被新方案替代覆盖——本轮第3次独立复核确认现在是直接从`risk_state`表读取上次落盘值，`RiskState`注释明确写"本地目前没有'仓位平仓+已实现盈亏'这条数据管道" | 高 | `docs/adr/0002-risk-engine-intercept-mode.md:55`（"仍然成立"编者按语所声称范围内） ↔ `internal/app/risk_wire.go:29-60`（无重建逻辑）、`internal/service/risk_service.go:56-138`、全仓库grep"重建/rebuild"无匹配 |
| D3 | `docs/architecture.md`两处小节标题（§7"API设计"备注、§8"复盘流程"）仍写"待对接/与现有录制工具对接待后续更新"，是"调用外部录制工具"旧方案的遗留表述，与`PRD.md`八节已确认的原生实现方案（"2026-07-22确认...不是调用一个独立运行的外部工具"）矛盾，且与同一份文档紧接着的"每日总结触发机制2026-07-17确认"正文段落自相矛盾。本轮第3次独立复核确认仍是2处，代码侧`POST /api/recorder/stop`不接收任何请求体，K线快照由服务端自己现算，与"待对接"暗示的"外部工具POST数据进来"完全不符 | 高 | `docs/architecture.md:521,629-636` ↔ `PRD.md:151-166`、`docs/architecture.md:638-649`（同文档内已正确描述） ↔ `internal/handler/recorder_handler.go:77-88`、`internal/service/recorder_service.go:106-191` |
| D4 | ADR-0002的D1编者按语声称三块被删内容（结构体/校验流程/**返回格式**）"具体设计见ADR-0003全文"，但ADR-0003通篇不含`{allowed, results, warnings}`这个响应信封结构（`ValidationResult`）的任何描述——只在`internal/service/interface.go`的Go源码注释里存在。"结构体"（`Rule`/`RuleResult`）与"校验流程"（遍历+跳过禁用规则）两块确实在ADR-0003里能找到对应内容，但"返回格式"这一块的替代来源只有源码，没有任何ADR/契约文档承接，编者按语对覆盖范围的描述过于乐观 | 中 | `docs/adr/0002-risk-engine-intercept-mode.md:18-25`（D1按语） ↔ `docs/adr/0003-rule-interface-abstraction.md`全文（grep"allowed/warnings/ValidationResult"均无匹配）、`internal/service/interface.go:24-30`（实际定义处） |

## 六项检查补充

- **自证测试弱点**（非独立A/B/C/D分类）：`internal/handler/ws_handler_test.go:195-197`的`TestWSKlinesHandler_RejectDuplicateIP`对第二次连接被拒绝时的HTTP状态码只用`t.Logf`记录（非`t.Errorf`/`t.Fatalf`）——如果实现把状态码改错（比如误返回500而非429），这个测试仍然会通过（只要dial失败即可），实际没有断言`docs/security-review.md:275`承诺的"同一IP只能一个WS连接"这个具体行为细节的HTTP语义。机制本身（`ipConnTracker`）经切片5独立复核确认正确，问题仅在这一条测试断言强度不足。
- **签名测试自证式但结构核实正确**：OKX REST/WS签名相关测试（`client_test.go:184-193`、`ws_private_test.go:89-101`）名字暗示"已知向量验证"但实际只判非空/判不同，属自证式弱测试；本轮切片2用WebFetch官方文档独立核实了签名拼接结构（prehash组成、时间戳格式差异）本身正确，判定为"测试薄弱但实现正确"，不构成实现缺陷。

---

## 与上次审计的对照

> 上次审计：2026-08-23（`9903a0d`）· 本次基线：2026-08-24（`a27fda2`）· 区间提交数：**9**
> 2026-08-23报告本身已完成对2026-08-22报告全部52项的Step 8全量吸收，本轮直接对照它的完整26项索引；同时本轮对2026-08-23报告"本次未复查"表里溯及的更早期（2026-08-22）编号条目做了3项针对性重新核对（见下方"首次坐实的历史疑点"）。

### 已解决

| 上轮编号 | 结论 | 解决提交 | 本轮复核 |
|---|---|---|---|
| A1 | `aquasecurity/trivy-action@0.28.0`版本号不存在 | `fe95d1f` | 切片6重新联网核实`@v0.28.0`确认RESOLVED |
| A2 | architecture.md的config.yaml示例过期，UnmarshalStrict下会解析失败 | `fe95d1f` | 切片6逐字段核对（maxPositionRatio=0.33/maxStopLossPercent=10.0/maxDailyLossPercent=5.0/cooldownHours=24等）确认RESOLVED |
| A3 | trade_handler.go回归超过300行限制 | `cd380c3`（拆出order_validation.go） | 切片6行数表确认trade_handler.go=215行、order_validation.go=125行，均达标，RESOLVED |
| B1 | security-review.md CORS白名单缺第3个来源 | `7dd0404` | 切片6确认`allowedOrigins`三项与文档表格逐项一致，RESOLVED |
| B2 | ADR-0001/security-review.md把已完整实现的私有WS频道描述成未来才会做 | `da33179` | 切片6确认§1.4现在时态描述与`SubscribeOrderFills`/`OrderStreamConnected`真实实现完全吻合，RESOLVED |
| B3 | milestones.md仍称周报"汇总每日总结" | `cd96d37` | 切片4确认两处（190/291行）均已带"2026-08审计B3更正"说明，且无第三处遗留，RESOLVED |
| C1 | PRD.md Must-Have功能列表从未列出"自选品种" | `a27fda2`（新增F7章节） | 切片7确认F7章节+3条验收用例均已写入，机制层面RESOLVED——**但F7-TC3本身的文字内容有事实错误，见新报告A1，是"解决动作本身引入新缺陷"的案例，不影响C1本身"watchlist已入PRD"这个事实的解决状态** |
| C2 | architecture.md环境变量表缺11项真实生效变量 | `0d770b3`（6→19行） | 切片6完整18字段/19变量四层核对（Go读取/`.env.example`/docker-compose透传/architecture.md表格）确认表格本身已完整，RESOLVED——仅发现一处更窄的新缺口（docker-compose层LLM_MODEL透传遗漏，见新报告C9） |
| C3 | `RecordingGenerating`/`RecordingFailed`两个状态值全仓库从未被生产代码赋值 | `d099f93`（删除枚举值） | 切片4全仓库grep确认"删减彻底，无遗漏"，唯一保留处是测试占位符且带说明注释，RESOLVED |

### 仍存在

| 上轮编号 | 结论 | 已存活几次审计 | 本次编号/状态 |
|---|---|---|---|
| B4 | ADR-0001声称Interval只有3档，代码实际5个Timeframe/16个bar映射 | 3 | 本轮切片2独立复核**确认仍存在**，见新报告B2（补充了ctx参数、ServerTime/GetAccountEquity方法缺失两个新维度） |
| B5 | deployment.md ReadyHandler伪代码与实际不符 | 3 | 本轮切片6独立复核**确认仍存在**，见新报告B6（新增：示例代码本身`writeError`签名不对，编译不过） |
| B9 | security-review.md审计日志滚动删除检查清单条目与自己修正的5.4节矛盾 | 2 | 本轮切片1独立复核**确认仍存在**，见新报告B9 |
| C7 | WS重连退避参数注释引用的"docs/adr/0004"实际不含此内容 | 3 | 本轮切片2独立复核**确认仍存在**，见新报告C4 |
| D2 | ADR-0002"从交易历史重建状态"承诺从未实现 | 3 | 本轮切片1独立复核**第3次确认仍存在**，见新报告D2 |
| D3 | architecture.md两处"待对接现有录制工具"标题矛盾 | 3 | 本轮切片4独立复核**第3次确认仍存在**，仍是2处，见新报告D3 |
| D1 | ADR-0002核心设计已被ADR-0003替换但未标记superseded | 3 | **状态变化**：上轮会话已按用户指示删除三块过时内容，本轮复核确认删除本身干净，但**修复动作自己遗留了新问题**（编者按语覆盖范围声称过度乐观），见新报告D1/D4 |

### 首次坐实的历史疑点（源自2026-08-23报告"本次未复查"表溯及的2026-08-22编号）

| 溯及编号 | 结论 | 此前状态 | 本轮结果 |
|---|---|---|---|
| （2026-08-22）B4 | architecture.md数据库设计缺analysis_records的video_path/status列 | 连续2轮"未纳入本轮任何切片的具体核查项" | 本轮切片4**首次专门核对并坐实**，见新报告B5 |
| （2026-08-22）C9 | architecture.md数据库设计节完全缺klines_15m/kline_environment表结构 | 连续2轮"未专门核对数据库设计章节的表结构完整性" | 本轮切片3**首次专门核对并坐实**，见新报告C5 |
| （2026-08-22）C10 | architecture.md未说明历史聚合与WS原生推送对1h/4h/1d/1w是两条独立管道 | 连续2轮停留在"疑似但未坐实"状态 | 本轮切片3**首次用代码交叉核实完整坐实**，见新报告C6 |

### 本次未复查

| 上轮编号 | 结论 | 为什么没查 |
|---|---|---|
| A4 | （低置信）GET /api/trade/history的symbol查询参数无格式校验 | 未纳入本轮任何切片的具体核查项 |
| B6 | api-documentation.md的swag目录/导入路径/CI diff目标与实际不符 | 切片7本轮聚焦错误码/普遍性覆盖率/组件行数，未专门核对这条文档元信息claim |
| B7 | coding-standards.md称前端用ESLint+Prettier，实际用oxlint | 未纳入本轮任何切片的具体核查项 |
| B8 | deployment.md展示的Dockerfile.dev用npm ci，实际用npm install | 未纳入本轮任何切片的具体核查项 |
| C4 | RecordClosedTrade事件驱动机制未被任何契约文档描述 | 切片1本轮核对了审计日志action枚举等其它维度，未专门重新核对这条具体claim |
| C5（旧） | 市价单彻底下线这一产品决策未写入PRD | 未纳入本轮任何切片的具体核查项 |
| C6（旧） | security-review.md端点清单缺/api/risk/validate和/api/risk/status | 未纳入本轮任何切片的具体核查项 |
| C8（旧） | security-review.md 2.2节"环境变量仅用于非敏感配置"与1.2节Docker明文回退例外矛盾 | 未纳入本轮任何切片的具体核查项 |
| C9（旧） | WS单连接限制描述未记录其依赖的部署拓扑信任前提（须经nginx） | 切片5本轮核实了单连接限制机制本身（IP维度）与文档一致，但未专门核对"必须经nginx"这个部署拓扑前提是否被写明 |

---

## 汇总表

| 类别 | 条数 | 高置信度条数 | 占比 |
|---|---|---|---|
| A（代码偏离契约） | 7 | 5 | 71% |
| B（契约落后） | 9 | 6 | 67% |
| C（契约未覆盖） | 10 | 3 | 30% |
| D（契约已失效） | 4 | 3 | 75% |
| **本轮合计** | **30** | **17** | **57%** |
| 另：上轮26项中本轮独立复核确认 | 10项已解决 + 6项仍存在 + 1项状态变化(部分处理遗留新问题) + 9项本次未复查 | — | — |
| 另：更早期(2026-08-22)编号中本轮首次坐实 | 3项（均从"未复查"转为"确认仍存在"） | — | — |

**置信度降级集中区**：
- A7（mock错误码不匹配）——测试fixture，非生产缺陷，按方法论主动降级
- C3（ADR-0001扩展模式未记录）——低置信度，是否需要在ADR层面记录一种实现模式存在解释空间
- C8（permission-matrix.md WS行归属）——低置信度，需要产品/架构侧确认这行本意是否只指向external client层
- C10（行号引用过期）——低置信度，纯引用维护问题，不影响实质内容，但已连续2轮漂移，建议下次修文档时改成引用函数名而非行号

**未列为问题的重要正面结论**：
- 上一轮10项修复中9项本轮独立盲测复核确认扎实（trivy-action版本号、config.yaml示例、trade_handler.go拆分、CORS白名单、私有WS频道文档、milestones.md周报表述、环境变量表补全、`RecordingGenerating`/`RecordingFailed`死代码删除，均RESOLVED）；第10项（PRD watchlist章节，C1）机制上也已解决，只是新写的验收用例本身有缺陷（见A1）
- OKX 1D/1W分桶边界（16小时/3天16小时偏移量）本轮经WebFetch官方帮助页原文+第三方issue双重独立验证，代码注释"联网实测"的说法完全站得住脚
- obs-websocket v5协议opcode常量、whisper.cpp CLI参数（含容易被怀疑拼写错误的`--suppress-nst`）本轮首次分别对官方文档/精确锁定的v1.9.2 tag做了WebFetch直接核实，均无偏差——此前两轮报告这两条一直停留在"需要人工核对"状态
- 写操作端点swagger状态码声明100%覆盖（8/8），且每条覆盖都有对应handler测试实锤，非抽查结论
- `seedDefaultWatchlist`面对生产环境唯一调用路径（单进程顺序启动）没有真实并发风险，TOCTOU窗口有数据库唯一约束兜底，这部分健壮性设计合理，只是PRD文字表述有误（见A1）
- WS单连接限制（IP维度）从`extractClientIP`的X-Forwarded-For解析、`ipConnTracker`并发安全获取/释放、到端到端拒绝重复连接，代码实现与文档描述完全吻合，且有真实dial级别集成测试覆盖
- `.env.example`/`docker-compose.yml`/`architecture.md`三处环境变量描述本轮经18字段完整核对基本全部对齐，仅一处新的窄口径缺口（LLM_MODEL）
- 风控引擎审计日志（CreateOrder/Check/UpdateRule/RecordClosedTrade四个写路径）与`docs/security-review.md`5.1节action枚举完全对应，是本轮契约兑现质量最高的一块
