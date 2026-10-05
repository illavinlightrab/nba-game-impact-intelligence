# 架构与实现范围

## 对原项目的判断

保留 FastAPI + SQLAlchemy + PostgreSQL。已有 NBA 球队、球员、赛程、球员统计、新闻事件和 Alembic 迁移可以复用。新增 React + Vite 前端放在 `frontend/`，后端全部位于 `backend/`。

原有 `app/agents/forecaster.py` 是球员得分和出场时间预测，不是球队胜率模型。新增 `app/intelligence/` 处理比赛级概率，不把球员预测值直接当作球队获胜概率。

## 与分享框架的对应关系

| 分享框架 | 当前实现 | 状态 |
| --- | --- | --- |
| Data Agent | NBA 目录、赛程、历史常规赛赛果导入 | 已实现 |
| Injury Agent | 官方 PDF/ESPN 伤病采集、状态核验、长期跟踪 | 已实现；来源失败显式显示 |
| News Agent | RSS 原文摘要 → Structured Outputs → 证据校验 → 人工审核 | 已实现；自动收费提取默认关闭 |
| Feature Agent | Elo、主场；出场分钟、缺阵/不确定/限时暴露差 | 已实现；数据缺失显式标记 |
| Prediction Agent | Elo + 条件启用的伤病残差 Logistic | 训练流程已实现；真实伤病模型待样本，均未校准 |
| Market Agent | 手动报价、公开 CLOB 订单簿、持久化快照 | 已实现；人工确认映射 |
| Risk/Audit Agent | 时效、样本、价差、深度、成本、金额和敞口检查 | 模拟流程已实现 |
| Orchestrator | `intelligence/service.py::run_game`，事务内保存预测与两队方向的决策 | 已实现；未引入 LangGraph |
| Backtest | 时间顺序更新 Elo；概率质量指标与分组检验 | 已实现，时间代理有局限 |
| Paper Trading | 模拟入场、去重、金额限制、按 NBA 比分模拟结算 | 已实现 |
| Scheduler | 后端生命周期内每 15 分钟采集，自动保存未来 48 小时赛前快照 | 已实现；不自动同步赛程或市场，不发送通知 |
| Live execution | 无真实下单 | 未实现 |

首版采用显式 Python 流程，避免在流程尚未稳定时额外引入 LangGraph、队列和多个服务。以上 Agent 是职责分层，不代表已经存在七个独立自治进程。

## 概率引擎

```text
历史已结束常规赛 / 季后赛
    ↓ 赛果可用时间 <= 当前预测时点
按时间更新球队 Elo
    ↓ 跨赛季向 1500 回归
主客队 Elo 差 + 主场参数
    ↓ logistic 变换
主队概率 P / 客队概率 1-P
    ↓ 盘口比较与风控
持久化预测、快照引用和每个方向的决策
```

`P(home) = 1 / (1 + 10 ** (-(home_elo - away_elo + home_advantage) / 400))`。

默认 `K=20`、主场参数 `65`、跨赛季保留比例 `0.75`。这些是基线参数，未经数据拟合；季前赛、全明星、附加赛和未知类型不进入首版训练与预测。

概率因素按相对 50% 的贡献展示：实力差、主场及已启用伤病模型的变化相加等于最终概率减 50%。近 10 场胜率和休息天数仍为描述性信息。伤病只在验证模型和完整本场数据同时满足时调整概率。

## 数据时序

- 预测只支持未来 `scheduled` 常规赛 / 季后赛。
- 回测按比赛开赛时间逐场预测，赛果在开赛后 6 小时才更新 Elo；同一时刻比赛的赛果不提前进入彼此预测。
- 只使用非空、非负、无平局的最终比分。
- 新闻的原始发布时间和入库时间都必须不晚于当前预测时点。新 Elo 模型不使用新闻，所以其历史回测不会隐含未来新闻。
- 历史 `LeagueGameLog` 只有日期，没有真实开赛时间。新建历史比赛以 NBA 日期的次日 00:00（纽约时区，含夏令时）保守估计，并标记 `tipoff_time_estimated`。赛程同步会用正式 UTC 时间覆盖估计值。
- **不声称严格复现真实历史数据可用性。** 缺少比赛结束时间、数据修订历史与首次抓取时间；正式策略评估前需要补齐这些数据。

## 新闻上下文与长期事件

- 使用 `app/news/context.py` 统一筛选比赛与原有球员 Forecaster 上下文，复用现有 `news_events`，无需新增数据库迁移。
- 普通新闻默认窗口为 168 小时，通过 `NEWS_RECENT_HOURS` 配置。最多展示 30 条普通新闻；长期事件在单独处理后展示，不被该上限挤掉。
- 长期类型：`injury`、`long_term_injury`、`season_ending_injury`、`ruled_out`、`suspension`、`trade`、`role_change`、`minutes_restriction`、`coach_change`。类型来自结构化录入，不通过标题猜测严重程度。
- 解除事件：`return_from_injury` / `injury_resolved` / `available` 解除伤病；`return_from_suspension` / `suspension_ended` 解除停赛；`minutes_restriction_lifted` 解除限时；`role_change_ended`、`coach_change_ended`、`trade_cancelled` 分别解除对应状态。长期类型的 `player_status=resolved/ended` 也表示解除，伤病类型的 `available` 表示解除伤病。复出不会自动解除上场时间限制。
- 同一球员、同一事件类别、同一比赛范围的最新记录替换旧状态。教练变化按球队关联；缺少球员 ID 的其他长期事件不会相互覆盖或自动解除，必须复核并补齐关联。
- 用原有 `POST /api/v1/news/events` 追加后续状态记录，保留原始新闻。解除时使用同一 `nba_player_id` 与比赛范围；对应新记录的发布时间和入库时间均不能晚于预测时点。
- 指定 `nba_game_id` 的事件只用于该比赛。跨比赛持续生效的事件应以球员/球队关联录入，不绑定单一比赛。即使后续事件未携带旧球队关联，也查询同一球员的更新，避免已复出的旧伤病或被后续交易替换的旧交易记录继续展示。
- 超过 `NEWS_REVIEW_AFTER_HOURS=168` 未更新的长期事件显示“待复核”。未记录解除不等于确认仍有影响；量化中的伤病状态另以 24 小时新鲜度检查。来源失败不生成健康断言。
- 预测是保存的快照；旧预测不会被静默修改，重新运行 Agent 后使用新规则。搜索接口的时间范围仅用于抓取新报道，与数据库中长期事件的保留规则分开。
- 当前 108 项内存后端测试通过；新增审核、量化、训练回退与网页搜索已通过隔离验证。测试不写入现有 PostgreSQL 数据。

## 自动采集、出场量化与训练

- `news/sources.py`：固定来源地址与官方 PDF 白名单；上限 8MB/60 页；按 PDF 坐标恢复表格阅读顺序。目录 403/404、空报告、排版变化或未提交均保留为不可用状态。
- `news/archive.py`：项目内 SQLite/WAL，保存原始文档版本、事件、不可回溯的审核时间、赛前特征和模型元数据；无需 PostgreSQL 迁移。事实按来源与内容去重，不以再次读取刷新原始发布时间。
- `news/extraction.py`：Responses Pydantic Structured Outputs；模型仅提取事件、状态、分钟数字和原文引用。代码核验引用、球员姓名、日期与当前球队唯一映射，所有新闻提取事件都待人工审核。
- `news/collector.py`：SQLite 租约防止多进程重复采集，每轮最多 3 篇收费提取；失败来源互不阻断。API 生命周期启动可取消后台任务，QA fixture 不启动真实采集。
- `intelligence/injury.py`：赛前同队近 180 天内最近 10 场出场数据，至少 5 场；伤病、停赛和分钟限制分别管理。旧状态、冲突、报告移除或缺少限制数字均需复核。不把得分暴露等同因果净损失，不假设替补价值。
- `capture_upcoming`：采集后在本地自动保存未来 48 小时最多 30 场的特征，不向 PostgreSQL 写入预测。`run_game` 同样记录快照；训练每场只使用一次、严格早于开赛的合格快照。当前状态不能用于历史倒填。
- 残差模型：Elo logit 作为固定 offset，缺阵/不确定/限时分钟差除以 240，得分差除以 120，固定 L2=0.05。默认至少 200 场，时间顺序 80/20 留出与 6 小时边界剔除；验证 Brier、Log Loss 均改善后保存模型，不在验证集调参。
- 数据覆盖：双方最近官方报告已提交，无相关未匹配伤病，球员统计完整且状态无需复核，Elo 历史充分。未提交与空白并不视为健康；G League 明确分类为阵容事件。超出训练特征范围时回退 Elo。
- `GET/PUT /api/v1/evidence/...` 提供状态、来源原文、自动任务设置；POST `collect/extract/reports/events/{id}/review/train` 分别采集、提取、导入、审核与训练。收费提取默认关闭。
- 真实模型尚无充足样本，默认只展示分钟量化而不调整胜率。当前没有独立校准器、替补补偿、阵容协同或教练/交易事件概率系数。

## 市场与模拟风控

- 查询地址固定为 `https://clob.polymarket.com/book`，只读取公开订单簿；Token ID 限定为数字。
- 校验返回 Token 一致、盘口为有限数值、价格位于 `(0,1)`、非交叉且包含买卖两侧。
- 不假设返回数组的排序，取最大 bid 和最小 ask。
- `liquidity` 在本实现中指**最优卖价档位的美元深度**，不是整个平台定义的流动性。
- 买入模型方向的有效价格 = `ask + paper_cost_buffer`；概率差 = 模型概率 − 有效价格。
- 默认要求双方历史各至少 10 场、报价与预测不超过 15 分钟、价差不超过 0.05、最优卖价深度至少 $1,000、成本后概率差至少 0.05。
- 手动报价仅做比较，不能生成允许开模拟仓位的信号。
- 入场再次检查当前报价、当前预测、比赛状态、单笔 $100、总敞口 $500 及方向去重。
- 使用 PostgreSQL 事务咨询锁串行化模拟敞口检查；数据库唯一约束阻止同比赛同方向重复记录。SQLite 测试适配器不验证这一锁的语义。
- 结算按 NBA 最终比分计算研究损益，具有幂等性；取消比赛、合约作废和交易所不同裁定规则需要未来单独实现。

## 新增 API

所有路径前缀为 `/api/v1/intelligence`。

| 方法 | 路径 | 功能 |
| --- | --- | --- |
| GET | `/config` | 不含密钥的配置状态 |
| GET | `/overview` | 数据表计数 |
| GET | `/games?season=2026-27` | 未来比赛、最近预测和报价 |
| POST | `/games/{uuid}/run` | 生成比赛预测及两队方向的风控决策 |
| POST | `/games/{uuid}/markets` | 保存手动报价 / 读取 CLOB 盘口 |
| POST | `/sync/results?season=2025-26` | 导入官方常规赛赛果 |
| GET | `/backtest?season=2025-26` | 指定赛季评估，先前赛季作为预热 |
| GET | `/paper` | 模拟仓位和敞口 |
| POST | `/paper` | 通过信号记录模拟仓位 |
| POST | `/paper/settle` | 模拟结算已结束比赛 |

原有 `/nba`、`/news`、`/prediction`、`/agents/forecaster`、`/ai`、`/health` API 保留。新接口采用可读的 400 / 422 / 502 / 503 错误，数据库不可用时前端明确显示错误，不填入虚假结果。

新增数据库表：`game_predictions`、`game_signals`、`paper_positions`。复用 `games`、`teams`、`news_events`、`market_snapshots`。迁移 `81a5d2904c01` 只增加表与时间估计字段，不删除原有表或数据；离线生成 SQL 不代表已经在 PostgreSQL 上应用。

## 球队与球员分析

前端导航为比赛预测、模型分析、新闻与伤病、数据与设置。已删除 `ResearchViews.jsx` 中的回测/模拟页面，数据页面独立为 `DataView.jsx`；比赛详情不再提供模拟金额或入场操作。保留原有后端回测/模拟 API 和数据库记录。

- `GET /api/v1/analysis/catalog`：球队、当前球员目录、模型名称和配置状态；不返回密钥。
- `POST /api/v1/analysis/chat`：校验球队/球员、赛季/比赛类型与最多 20 条、总计 40000 字的用户/助手消息，禁止客户端 system/developer 角色。
- `analysis/intent.py` 在读取统计前解析问题。英文全名/已知别名和显式赛季采用本地匹配；未知称呼、否定提及等使用 Responses 结构化识别，再按原文提及和本地规范姓名核验。问题中明确的范围覆盖客户端默认值；无法唯一匹配时返回澄清，不默默读取默认球队。
- `analysis/context.py` 只读本地阵容、最多 200 场可用比赛、球员统计与最多 40 条相关新闻，生成证据编号及时间/样本边界。统计按历史球队分组，空值保留并附字段样本数；效率字段为逐场均值，不是累计加权值。
- `analysis/service.py` 复用现有 OpenAI client，以 Responses 流式接口生成中文报告；`store=False`，上限 3500 输出 Token。本地模式 150 秒总超时；`web_search=true` 时使用原生 `web_search`、`tool_choice=required`、实时访问、最多 3 次工具调用、240 秒总超时。不更新球队概率，不写入统计或伤病状态。
- `analysis/web_sources.py` 读取 API 的 `url_citation` 注释与 `web_search_call.action.sources`，区分已引用来源和检索参考 URL；校验 URL，将各文本片段的引用位置归一为 `[W1]` 等编号，不将模型自行生成的网址当作证据。
- SSE 事件为 `meta`、`delta`、`done`；网页模式另发 `web_status` 和 `web_sources`（引用来源、检索时间、规范化正文）。失败返回 `error`，搜索请求未实际执行时不能完成；部分内容不能被标为完整报告。断开或取消会关闭上游流。
- `AnalysisChat.jsx` 提供默认开启的网页搜索开关，处理中文分片、多轮追问、取消、范围变化、来源详情与 Markdown 导出。模型内容以 React 文本节点渲染；只有 API 网页引用成为可点击链接，拒绝非 HTTP(S) 协议及带认证信息的 URL，不执行模型生成的 HTML。
- 对话区高度随视口调整，生成时只滚动内部消息；输入区通过 sticky 吸附在窗口底部，回答结束后使用 `preventScroll` 恢复输入焦点，避免长报告把追问入口推出视野。
- 识别结果通过 `meta.routing` 返回；前端自动更新球队、球员、赛季与比赛类型，并保留对话和生成过程。手动改范围才重置会话。会话在当前网页内存保存，导航切换保留，网页刷新清空；最多最近 8 条已完成消息、每条 3500 字发给模型。所选数据和近期对话会发送至已配置的 OpenAI API。
- 新功能无需数据库迁移；单元及浏览器验证使用隔离 SQLite 和合成模型输出，不写入现有 PostgreSQL。网页搜索回答分支另外用一次公开 NBA 问题实测成功，取得官方页面引用；未读取或外传数据库资料。明确姓名、中文常见别名与赛季识别已用本地真实目录验证；复杂问题识别模型分支仍仅隔离测试。

流式接口参考 [OpenAI Responses streaming 文档](https://developers.openai.com/api/docs/guides/streaming-responses)。
网页搜索参考 [OpenAI Web search 文档](https://developers.openai.com/api/docs/guides/tools-web-search)。

## 浏览器验证

Browser 插件未提供，IAB 创建标签页曾超时，因此使用本机 Chrome + Playwright 进行隔离验证。所有测试脚本及截图路径限定在项目内。

在独立终端启动测试 API：

```bash
NBA_QA_FIXTURE=1 PYTHONPATH=backend backend/.venv/bin/python -B -m uvicorn tests.browser_fixture:app --host 127.0.0.1 --port 8001
```

另开终端从项目根目录启动专用测试前端（3001 代理至 8001）：

```bash
node frontend/node_modules/vite/bin/vite.js --config frontend/vite.qa.config.js
```

然后从项目根目录执行。`node` 不在 PATH 时参照 README 使用已有 Node 的绝对路径：

```bash
# 使用已有 Playwright；没有安装时，可在项目内安装测试依赖。
# pnpm add -D playwright
node frontend/tests/stream.mjs
node frontend/tests/smoke.mjs
# 或 PLAYWRIGHT_MODULE=/absolute/path/to/playwright/index.mjs node frontend/tests/smoke.mjs
```

**测试 API 是合成数据环境，不能当作实际后端运行。** 默认 `app.main:app` 不导入该测试模块，也不会出现测试球队或测试盘口。浏览器脚本首先调用只存在于测试环境的重置接口，否则停止。
测试脚本默认使用 8001/3001，可通过 `QA_API_URL` / `QA_FRONTEND_URL` 覆盖；不要指向正式服务。

## 后续开发顺序

1. 在隔离 PostgreSQL 验证迁移、事务回滚和并发敞口限制；验证官方数据真实连通性。
2. 保存赛果首次可用时间和原始响应，建立可复现的数据版本。
3. 积累真实赛前伤病样本；引入球队高级统计、替补补偿和阵容协同特征。
4. 用按时间分离的训练、校准、评估区间训练 Logistic / XGBoost；不可在同一回测窗口拟合校准器。
5. 验证胜者合约映射，采集历史盘口，再评估含真实执行成本的模拟策略。
6. 扩展已有新闻调度至赛程与市场数据，并加入经用户授权的外部通知。

