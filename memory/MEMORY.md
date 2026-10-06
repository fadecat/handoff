# MEMORY.md — 项目长期笔记（bond-lifecycle-vnext / C:/Users/hc/Desktop/web）

## 项目目标
对集思录全部 319 只存量可转债，解析募集说明书得到结构化条款，计算
**「下修计数」**与**「股东大会前下修底价」**，替代集思录三个会员付费字段
`adjust_count` / `readjust_dt` / `lt_bps`。

## 已定稿口径（2026-10-06 会员真值校准，勿再改）
- 区间 = 最近 `window` 个交易日 ∩ `[readjust_dt, 最新交易日]`，锚点当天计入（`>=`）
- 计数 = 区间内满足比较式的天数，**不封顶**（不是 `min(raw, required_days)`）
- 窗口 = `closes[-W:]`，**不偏移**（不是 `closes[-(W-1):]`）
- 「低于」→ `<`；「不高于」→ `<=`
- `adjust_remain_days = -1` ⟺ 计数 = 0
- 底价 = `max(vwap20, vwap1, 每股净资产, 面值)`，基准日 = 股东大会前 1 个交易日

## 成绩（对 319 只会员真值快照）
`adjust_count` 192/198 (97%)　条款四件套 198/198 (100%)
`readjust_dt` 229/231 + 2 兜底 = 100%　`lt_bps` 289/309 (93.5%)
（`lt_bps` 的 20 例不符经查为集思录标错，非我方错）

**免费/付费边界已澄清**：`adjust_remain_days` 是免费字段；`readjust_dt` 可由
U 日志（不下修公告，220 只/1182 条）+ D 日志（92 只/139 条）+ `drdays` 100% 重建。
会员到期不影响任何结论，真值快照已入库 `data/research/jisilu_adjust_member_20261006.json`。

## 目录约定（写新脚本必须遵守）
```
scripts/research/   25 个研究探针
data/research/      关键数据 7.8MB（入库）
.workbuddy/datasets/  2.5GB 说明书 PDF + 317 份中间抽取（.gitignore 忽略，勿入库）
```
脚本内路径：
- `HERE` / `DATA` = `Path(__file__).resolve().parents[2] / "data" / "research"`
- `WORK`         = `... / ".workbuddy" / "datasets"`（prospectus-lean / -batch / -out）
- `SCRIPT_DIR`   = `Path(__file__).resolve().parent` ← **只用于 `sys.path.insert`**

⚠ 踩过的坑：`sys.path.insert` 用 `HERE` 会指到数据目录，导致
`from parse_adjust_tc_v2 import parse` 失败。已有 5 个文件因此改过。

⚠ `.gitignore:77` 忽略整个 `.workbuddy/`。研究成果**必须落在 scripts/ 或 data/ 下**才有版本管理。

⚠⚠ **别为甩掉大文件把小资产一起甩了**（2026-10-06 踩过）：
`.workbuddy/datasets/prospectus-lean/` 里 PDF 占 2.4GB、**310 份 LLM 抽取 JSON 只占 7.4MB**，
却因同在被忽略目录，导致 310 次 DeepSeek 调用的结果一度零版本管理。
已用 `scripts/research/archive_prospectus_extract.py` 剥离到
`data/research/prospectus_extract/`（含绝对路径清洗，309/310 份原本硬编码 `C:\Users\hc\...`）。
**检测绝对路径的坑**：文件里是 JSON 转义的双反斜杠，正则 `[A-Za-z]:[\\/]Users` 匹配不到，
要写 `[\\/]+` 或直接搜 `Users` 子串。

## 端到端验证结论（2026-10-06，分支 verify/free-only-count）
**模拟会员到期（会员快照改名 .EXPIRED + 删派生产物）从零跑通**：
匿名重抓 319×53 → 重建锚点 231/231 → 计数 319/319 全有值；
真值对账 **196/198 / 231/231 / 289/309**，与会员在场时**完全相同**。
重抓快照与入库版本**逐只零差异**（仅 fetched_at 时间戳变）。
→ 不依赖会员，计数完全跑得出来。

⚠ 发现的脆弱点：U 日志（需登录但**不要会员**）是 `count_engine_final` 的**硬依赖**，
文件不在就崩。但它只用于 `n_unadj_announcements` 诊断字段，**不参与计数**。
修复（缺失时降级）在分支 `7d71fd9`，**尚未合回主分支**。

## 集思录接口（2026-10-06 二次更正 —— 推翻前一版，以此为准）
- `GET https://www.jisilu.cn/webapi/cb/adjust/` —— **必须带末尾斜杠**，不带一律 404
  （旧笔记写成"该接口不存在"，是被 404 误导了）。
- ⚠ **该接口分免费层 / 会员层，靠 cookie 区分**。此前记的"不需登录、带不带 cookie
  完全一致"是**错的** —— 那次是把两次**匿名**请求互相比（当时的 session cookie
  已失效/非会员），所以看不出差异。2026-10-06 18:07 重新登录后实测：
  **53 个字段里 10 个随 cookie 变化**：
  | 字段 | 无 cookie | 会员 cookie |
  |---|---|---|
  | `lt_bps` | 0/319 | **319/319**（不得低于 164 / 或可低于 155） |
  | `adjust_count` | 0/319 | **208/319**（198 可解析 + 10「存续期内不下修」） |
  | `adjust_event` (+5 个子字段) | 0 | **4 只**（进行中的下修事件） |
  | `adjust_orders` / `owned` | 0 | 个性化值 |
  - 坑点：无 cookie 时会员字段**不报错、静默返回 null**，极易误判成"接口没有这字段"。
  - 抓会员态：先跑 `jisilu_login_probe.py` 刷新 `data/state/jisilu_session.json`，
    再用 `probe_webapi_member_diff.py` 确认 `lt_bps` 非 null。
- **免费层（不带 cookie 也全给）**：`readjust_dt`(231) `adjust_tc` `threshold_value`
  `adjust_remain_days` `drdays` `adjust_count_days` `convert_dt/begin/cd_tip`
  `sc_notes` `convert_price_tips` `adj_scnt/adj_cnt` `icons/redeem_icon/bond_nm_tip`
  → **`readjust_dt` 确认为免费字段**，与是否会员无关。
- `POST /data/cbnew/adjust_list/` 同样：无 cookie 时 `adjust_count`/`lt_bps`/`readjust_dt`
  字面值为 `'member'` 占位符。两个接口的会员字段值**已交叉核对 198/198 全等**，
  但 webapi 是 JSON 直读，比解析 HTML 干净。
- 脚本：`scripts/research/fetch_adjust_webapi.py`；对照探针 `probe_webapi_member_diff.py`
  （需 `requests` + `cryptography`，用 anaconda python 跑）。
- **会员态快照已冻结入库**：`data/research/jisilu_adjust_webapi_member_20261006.json`
  （495KB，含 lt_bps 319 + adjust_count 208 + 4 只下修事件）。会员 10-13 前后到期，
  过期后无法再取 —— 后续一切真值核对都基于这份。

## 计数残差根因（P1，2026-10-06 定位，未落地）
**4 只真值 0 的残差 = 全部是刚发行、未满一个窗口的新券**：
莱特 118077 / 隆新 123288 / 满坤 123287 / 奥士 127117，
共同点：`price == 100.000`（尚未有效交易）、`readjust_dt = -`、发行日 2026-09-23~10-12。
→ **根因：我们的窗口回溯了正股的历史，但转债自己还没活够一个 window。**
→ 修复口径：窗口起点应加上 **max(readjust_dt, 转债上市日)**；
  **上市后累计交易日 < window 时不得进入计数态（count = 0，remain = -1）**。
另 3 只同组新券（瑞鹄02 127116 / 国科 123286 / 常青 113711）我们本来就算出 0，不受影响。
剩下的宜化 127114（10 vs 9）、震裕02 123282（4 vs 7）是另一类（off-by-one / 复权口径），未解。

⚠ **修复口径已于 2026-10-06 晚复核修正 —— 上面第 61 行的第二句不要照做**：
- **「未到转股期 → 不计数」是错的**。52 只未到转股期里 **15 只 remain ≥ 0**（在计数）；
  宜化 / 震裕自己就是"未到转股期却在计数"（remain 6 / 8）。判别信号不是 `convert_dt`。
- **「上市后累计交易日 < window → 不进计数态」也是错的**。震裕只有约 18 个存续交易日
  却被计了 7 天，说明集思录**允许用不足一个窗口的天数计数**。加这条会把震裕推得更远。
- **正确的、也是唯一需要的改动**：窗口起点 = `max(readjust_dt, 转债上市日)`。
  未上市时窗口为空 → count 自然为 0。不要额外加"不足窗口即封零"。
- 判别"未上市"的可用信号：**`price == 100`**（全 319 只里恰好 7 只，就是上面那 7 只）。
  ⚠ 该接口 53 个字段里**没有上市日/发行日**，只有 `convert_dt` / `maturity_dt`，
  所以要么用 `price == 100` 作代理，要么另找数据源补上市日。
- 预期收益：仅修这 4 只 → **196/198**。宜化 / 震裕不动。

## 会员到期后的自给自足评估（2026-10-06 实测，重要）
**前提更正**：`/webapi/cb/adjust/` **不是**纯免费接口，会员字段要靠 cookie 才吐
（见上节）。以下结论不变，但"会员到期后能否自算"要看**输入**是否免费，而非输出。

实测四个数据源（均不带 cookie）：
| 数据源 | 结果 | 结论 |
|---|---|---|
| `GET /webapi/cb/adjust/` | 200，352KB/319行 | 需登录**即可拿全部，会员则多 10 个字段** |
| `POST /data/cbnew/adjust_list/` | 200，319行 | 同上，`adjust_count`/`lt_bps`/`readjust_dt` 无 cookie 时为 `'member'` |
| `get_adjustlog?adj_type=D` | 200，有数据 | ✅ 免费，不需登录 |
| `get_adjustlog?adj_type=U` | **402** | ⚠️ 需**登录**（但登录 ≠ 会员，到期后账号仍能登录） |

**证明方式（不要只靠嘴说）**：`scripts/research/pipeline_free_only.py`
硬约束 —— 计算阶段只读匿名快照 `jisilu_adjust_webapi_20261006.json`
（内含卫生检查：lt_bps/adjust_count 有值即判"快照被污染"并退出）、
tushare 行情缓存、我们自己的说明书抽取产物；
**完全不用** U 日志（需登录）和会员快照（只在对账环节读）。产物
`data/research/selfcheck_free_only.json`。2026-10-06 实测输出：

| 字段 | 自算覆盖率 | 依赖 |
|---|---|---|
| `adjust_count` | 196/198 (99%) | 条款+阈值+正股行情+免费 readjust_dt，**全免费** |
| `readjust_dt` | 231/231 (100%) | webapi 免费字段，**不需登录** |
| `lt_bps` | 307/319 自算 | 说明书原文抽取，与集思录无关 |

`lt_bps` 剩下 12 只 = 10 只未抽取 + 2 只抽取失败（鼎通 118072 / 阳谷 123211），
是**沿用已入库历史真值的冻结值**；新债或重抽必须补跑说明书解析。
20 例与真值冲突中 18 例我方有原文净资产句（判定集思录标错），仅 2 例是我方抽失败。

⚠ 关键佐证：`jisilu_adjustlog_20261001.json` 里 `lt_bps`/`adjust_count`/`readjust_dt`
字面值就是 `'member'` —— 说明**我们整条管线一直跑在纯免费数据上**，
会员只用于最终验证，从未进入计算链路。

## 募集说明书：一次解析，不必重复（2026-10-06 定论）
- `batch_extract_prospectus.py` 有 `already_done()` + `--force**：已成功则跳过，可断点续跑**。
  重跑只补缺口，**不会重复烧 LLM**。
- 产物 309 份已入库 `data/research/prospectus_extract/`（298 SUCCESS + 11 SALVAGED，7.4MB）。
- **下修条款 = 发行时敲定的静态条款**，存续期内不变；`lt_bps`（能否破净）也是静态判定。
- **唯一时变项「每股净资产」不来自说明书**：底价里用 `sprice / pb`（webapi 免费字段）
  反推，或取下修日志 `net_asset` → **底价计算零 LLM 调用**。
- 需重跑 LLM 的仅 3 种：新债上市（增量）、补抽 10 只缺口、条款被股东大会修订（罕见）。
- ⚠ `not_extracted_count=22` 在 309 份里**全部一致**，是 lean profile 有意裁剪，
  **不是失败信号**，别据此判失败。

## 换机复现（2026-10-06 裸克隆实测）
`git clone -b bond-lifecycle-vnext` → 用零第三方库的 python 3.13 跑：
计数 **319/319**、adjust_count **196/198**、readjust_dt **231/231**、lt_bps **289/309**，
**与原机完全一致**。仓库已跟踪 25.2MB / 1085 文件。

⚠ **计数链路不读说明书产物**：`count_engine_final.py:80` 用 `parse(adjust_tc)`，
条款来自集思录**免费字段**。说明书 309 份 LLM 产物服务 **lt_bps + 条款溯源**，不服务计数。
→ LLM 物料是否持久化，影响重抽与 lt_bps，**不影响计数能否跑出来**。

换机需自备（均未入库）：
1. `.env` —— ⚠ **`.env.example` 缺 `TUSHARE_KEY` 与 `deepseek_key`**（后者要求 `.env` 里
   字面 `deepseek_key=`，见 `announcement_llm.load_deepseek_key`）。计数不需要，补行情/重抽需要。
2. `.workbuddy/datasets/` 2.5GB PDF —— 有意排除；链接清单 `prospectus_link_scan_v2.json` 已入库，可重下。
3. `data/state/jisilu_session.json` —— 会员快照已冻结入库，验证不受影响。
4. Python ≥3.10；**计数引擎纯标准库**，无需 pip install。

⚠ `count_engine_final.py` 自打印的「可比对 50 / 一致 36」是**遗留粗查**（对 Oct-01 旧日志），
不是真值对账；真值看 `verify_against_member.py`。

## 换机交接（2026-10-06 晚，旧机弃用）
- ⚠️ 上游分支名是 **`origin/codex/bond-lifecycle-vnext`**（带 codex/ 前缀），本地叫
  `bond-lifecycle-vnext`。**本机无 GitHub 凭据**（无 gh、SSH key 未授权）→ 推送只能用户自己来：
  `git push origin HEAD:codex/bond-lifecycle-vnext`（密码用 PAT，不是登录密码）。
- 记忆已归档进仓库 `docs/research/memory-archive/`；**新机器要拷回 `.workbuddy/memory/`**。
- 交接包（仓库外）：`C:/Users/hc/Desktop/web-handoff/`（含 2.5GB datasets + MIGRATION-CHECKLIST.md）。
- `.env` 需手动带走；`.env.example` **缺 `tushare_api_key` / `deepseek_key`**（待批准补键名）。

## 环境坑
- 本机托管 python **无 requests / httpx**；只有 anaconda 的 `requests` + `cryptography`
  → 抓集思录用 `C:/Users/hc/anaconda3/python.exe`
- tushare 走代理 `https://pcd.mobcvb.cn/tushare/pro/daily`，header `X-API-Key`，key 在 `.env`
- 东财 push2his 约 30 次后限流
- 项目需 Python ≥3.10；装包/建环境**给命令让用户自己跑**（用户用 conda）

## 文档三件套（2026-10-06，docs/superpowers/specs/）
- `2026-10-01-xiu-pipeline-design.md` —— 设计**口径**（算法怎么写、为什么）
- `2026-10-06-xiu-pipeline-runbook.md` —— **怎么跑**（环境/命令/数据字典/坑速查）
- `2026-10-06-xiu-pipeline-handoff.md` —— **交接与完结**（成绩/gap/优先级/下一步）

## 复现缺口已补（2026-10-06 晚）
`quote_cache.json` 和 `readjust_dt_rebuilt.json` 原先只有读、没有生成脚本
（一次性内联命令产物），管线不可复现。已补：
- `fetch_quote_cache.py`（tushare 代理，默认只补缺失；**区间末端取快照日，
  不要改成"到今天"**，否则窗口位移、历史对账全变）
- `build_readjust_dt.py`（webapi 免费 readjust_dt + drdays，231/231，**不需登录**）

## 已完成（2026-10-06 晚，已落地）
- **4 只新券残差已修复** → `adjust_count` **196/198 (99%)**。
  改法：窗口起点 `max(readjust_dt, 上市日)`，未上市 → 空窗口 → 0；
  判别用 `price == 100.000`（319 只里恰好 7 只）。已重跑，无 collateral damage。
- 免费接口 `GET /webapi/cb/adjust/` 全量 53 字段已入库
  （`data/research/jisilu_adjust_webapi_20261006.json`，319×53）；
  **会员态**快照 `jisilu_adjust_webapi_member_20261006.json` 亦已入库（含 lt_bps 319、
  adjust_count 208、4 只进行中下修事件）。

### 进行中的下修事件（adjust_event_*，P0 活样本，会员字段）
| 债 | 会议日 | status | 结果 |
|---|---|---|---|
| 123259 锦浪转02 | **2026-10-09** | R（待决议） | 新转股价待揭晓 |
| 123220 易瑞转债 | **2026-10-14** | R（待决议） | 新转股价待揭晓 |
| 123109 昌红转债 | 2026-09-28 | S（已完成） | 新转股价 **20.930**，2026-09-29 生效 |
| 127034 绿茵转债 | 2026-09-29 | S（已完成） | 新转股价 **9.580**，2026-09-30 生效 |
→ 锦浪/易瑞是**唯一能外部验证底价预估**的样本，过期不候；
→ 昌红/绿茵是**已完成**的两例，可拿来反向验证底价公式（此前只有 127 条下修日志）。

## 未完成（gap）
- **2 只计数残差**：127114 宜化（我 10 / 真 9）、123282 震裕转02（我 4 / 真 7）
  —— 疑似 off-by-one 或复权口径，非新券问题
- 10 只 `lt_bps` 未抽取：113049 / 118050 / 118056 / 118064 / 123114 / 127072 /
  127105 / 127109 / 132024 / 132026（末两只 EB 可排除）
- 2 只抽取失败：118072 鼎通、123211 阳谷
- ~~3 只正股行情取不到~~ **已关闭**：300973 / 300622 / 002120 原缓存存成空列表，
  `fetch_quote_cache.py` 已补齐（各 120 条）
- ~~`readjust_dt` 覆盖 229~~ **已关闭**：现 231/231，并纠正 118016
  （旧值 2025-03-12 是错的，真值实为 `'-'`）
- **尚未写任何 backend/ 业务代码**，落地前需跟用户确认（设计文档工程约束第 1 条）

## 用户协作偏好（本项目）
- 提交前会问"当前进度可以 commit 了吗"——倾向**先更新过时的设计文档再提交**，
  不愿把错误口径固化进仓库
- 倾向"脚本 + 关键数据入库，大体积缓存留本地"
