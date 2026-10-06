# 换机迁移清单（2026-10-06）

**本文件位置：仓库内 `handoff/`**（与代码一起版本管理，换机 clone 下来就在）
分支：`bond-lifecycle-vnext` @ `d484564`　远端：`https://github.com/fadecat/web.git`

**大件物料已迁出到独立仓库 [`fadecat/handoff`](https://github.com/fadecat/handoff)**
（2.5GB 说明书 PDF 不适合塞进代码库），两个仓库配套使用：

```
web/handoff/                        ← 本仓库（轻量）
├── MIGRATION-CHECKLIST.md   9KB    本文件
├── repo.bundle              6MB    web 仓库完整快照（不依赖 GitHub 的兜底）
└── memory/                  136KB  项目记忆副本（正本 docs/research/memory-archive/）

fadecat/handoff                     ← 独立仓库（大件）
├── datasets/                2.5GB  314 份说明书 PDF + LLM 缓存
├── repo.bundle              6MB
├── memory/                  136KB
└── MIGRATION-CHECKLIST.md
```

---

## 📦 两个仓库都要推

```bash
# ① 代码仓库（web）
cd C:/Users/hc/Desktop/web
git push origin HEAD:codex/bond-lifecycle-vnext      # 密码用 PAT，不是登录密码

# ② 大件仓库（handoff）
cd C:/Users/hc/Desktop/handoff
git push -u origin main
```

⚠️ ② 要传 **2.5GB**，耗时长且可能被代理中断（本机代理 `127.0.0.1:7890` 间歇性
报 `CONNECT tunnel failed 502`）。传不动就改用移动硬盘直接拷 `datasets/`，
或到新机器按 `web` 仓库里的 `prospectus_link_scan_v2.json` 重新下载。

---

## ⚠️ 第 0 步：推送（不做这一步，研究成果全部丢失）

本地有 **12 个提交从未推送到远端**，包含：309 份说明书 LLM 抽取、会员真值快照、
全部下修计数脚本与文档。**旧机一旦弃用，这些就没了。**

我这台机器上**没能完成推送**：

- 第一次推送被超时中断；第二次报 `could not read Username for 'https://github.com'`
  （未装 `gh`，`~/.ssh/id_rsa` 在 GitHub 未授权：`Permission denied (publickey)`）；
  之后重试均报 `CONNECT tunnel failed, response 502`（代理挡住，无法访问 GitHub）。
- 本地 `origin/codex/bond-lifecycle-vnext` 这个跟踪 ref 显示到了 `56c720b`，
  但因为网络不通，**我无法用 `git ls-remote` 向 GitHub 核实**。所以这个状态**不可信**。

### 现状（21:57 更新）

`git reflog` 明确记录过 **3 次成功推送**（`4fe244a` → `56c720b` → `ec374fc`），
说明**主体研究成果已在 GitHub 上**。但之后凭据失效，最后一个提交 **未推送**：

```
b7e35aa  docs(research): 补齐排查手册与推进阶段文档，修 .env.example 缺键
```

⚠️ `git ls-remote` 也因无凭据无法核实（仓库是私有的）。**弃机前请亲自确认**：

```bash
cd C:/Users/hc/Desktop/web
git push origin HEAD:codex/bond-lifecycle-vnext
git ls-remote origin codex/bond-lifecycle-vnext   # 应看到 b7e35aa...
```

若提示输入用户名密码：用户名 `fadecat`，密码是 GitHub **PAT**（不是登录密码），
去 GitHub → Settings → Developer settings → Personal access tokens，勾选 `repo` 生成。

推送后验证（应输出 `0`）：

```bash
git rev-list --count --branches HEAD --not --remotes
```

### 🆘 兜底：repo.bundle（不依赖 GitHub）

本包里 `repo.bundle`（6.0 MB）是**完整仓库快照，含全部历史与所有分支**
（`git bundle create --all`，已 `git bundle verify` 通过，HEAD = `522c565`）。
**即使 GitHub 那边一个提交都没推上去，靠它也能 100% 恢复。**

```bash
git clone <新仓库>/handoff/repo.bundle web
cd web
git checkout bond-lifecycle-vnext      # 或 git checkout 522c565
git remote set-url origin https://github.com/fadecat/web.git   # 恢复远端
```

这条路完全不经过 GitHub。**建议无论如何都把 bundle 带走。**

### 网络提示

本机 git 走本地代理 `http://127.0.0.1:7890`（`git config --local` 里配的），
间歇性报 `CONNECT tunnel failed, response 502` —— 是代理抽风，不是仓库问题。
新机器若也需要访问 GitHub，记得配同样的代理。

---

## 第 1 步：密钥（不要进 git，手动带走）

`.env` 在旧机 `C:/Users/hc/Desktop/web/.env`，**未入库（正确）**。新机器需要重建，
里面共 12 个键：

| 键名 | 用途 | 缺失后果 |
|---|---|---|
| `tushare_api_key` | 正股行情（唯一外部单点） | 补行情失败；已入库的 quote_cache 够现有 319 只用 |
| `deepseek_key` | 说明书 LLM 抽取 | 不能重抽说明书；不影响计数 |
| `JISILU_USERNAME` / `JISILU_PASSWORD` | 集思录登录（抓 U 日志） | 只影响诊断字段，计数不依赖 |
| `DATABASE_URL` `ENV` `LOG_LEVEL` `HOST` `PORT` `SCHEDULER_ENABLED` | 后端运行 | 研究脚本不用 |

⚠️ `.env.example` 里**缺 `tushare_api_key` 和 `deepseek_key`** 两个键（已确认），
新机器照抄 example 会漏。**建议把这两个键名补进 `.env.example`（值留空）。**

---

## 第 2 步：大件物料（2.5GB → 独立仓库 `fadecat/handoff`）

已迁到独立仓库，本地在 `C:/Users/hc/Desktop/handoff/datasets/`，含：

- `prospectus-lean/` 2.4GB —— 314 份募集说明书 PDF + 310 份抽取 JSON
- `prospectus-batch/` 24MB —— LLM proposal 内容寻址缓存 + sqlite
- `prospectus-out/` 15MB —— 同上
- 5 个探针 JSON（webapi 无 cookie / 会员 cookie、下修审计 v2/v3/unknown）

三种拿法，任选：

```bash
# A. clone 大件仓库（最省事，但要拖 2.5GB）
git clone https://github.com/fadecat/handoff.git handoff

# B. 移动硬盘直接拷（推荐，不受 GitHub 大小限制影响）
#    拷 C:/Users/hc/Desktop/handoff/datasets

# C. 都不做 —— web 仓库里有 prospectus_link_scan_v2.json，新机器按需重下 PDF
```

放回工作区（脚本默认从 `.workbuddy/datasets` 读）：

```bash
cp -r handoff/datasets web/.workbuddy/datasets
```

⚠️ 大件**不是**最贵资产：真正的成本是 310 次 LLM 调用的**抽取结果**（7.4MB），
那个已在 `web` 仓库里。PDF 丢了只是要重下 + 重跑，不会丢结论。

---

## 第 3 步：项目记忆（已双份保险）

`.workbuddy/memory/` 被 `.gitignore:77` 忽略，本会丢失。已做两手：

1. **入库**：`docs/research/memory-archive/`（含 4 篇日志 + 项目 MEMORY.md + 用户级 MEMORY.md）
2. **本包**：`handoff/memory/`

新机器落地后，把它们拷回 `<新仓库>/.workbuddy/memory/` 才能被自动读取：

```bash
cp docs/research/memory-archive/*.md .workbuddy/memory/
# 再把 user-level-MEMORY.md 放回 <用户目录>/.workbuddy/<user-id>-personal/MEMORY.md
```

---

## 第 4 步：新机器落地验证

```bash
git clone -b codex/bond-lifecycle-vnext https://github.com/fadecat/web.git web
cd web
cp -r handoff/datasets .workbuddy/datasets    # 可选
cp docs/research/memory-archive/*.md .workbuddy/memory/ # 可选
# 建 .env：至少 tushare_api_key、deepseek_key

python scripts/research/count_engine_final.py
python scripts/research/verify_against_member.py
```

**预期输出（裸克隆实测，零第三方依赖、Python ≥3.10 即可）**：

| 项 | 预期 |
|---|---|
| 计数产出 | 319 / 319 |
| `adjust_count` 对真值 | 196 / 198 (99%) |
| `readjust_dt` | 231 / 231 (100%) |
| `lt_bps` | 289 / 309 (94%) |

⚠️ 别被 `count_engine_final.py` 自己打印的「可比对 50 | 一致 36」误导 ——
那是拿 Oct-01 旧日志的遗留粗查，**不是**真值对账。真值看 `verify_against_member.py`。

---

## 第 4.5 步：文档四件套（新机器先读这两篇）

仓库 `docs/superpowers/specs/` 下：

| 想做什么 | 文档 |
|---|---|
| **口径**：算法为什么这么写 | `2026-10-01-xiu-pipeline-design.md` |
| **使用**：怎么跑、每步干什么、数据字典 | `2026-10-06-xiu-pipeline-runbook.md` |
| **排查**：算错了 / 报错了怎么定位 | `2026-10-06-xiu-pipeline-troubleshooting.md` |
| **推进阶段**：做到哪了、还剩什么、下一步 | `2026-10-06-xiu-pipeline-status.md` |
| **交接**：成绩、gap、换机迁移 | `2026-10-06-xiu-pipeline-handoff.md` |

刚开工建议先读 **status**（知道做到哪了）+ **troubleshooting**（遇到问题直接查），
要跑管线再看 **runbook**，要改算法才看 **design**。

---

## 第 5 步：接上未完成的工作

- 计数残差 2 只：宜化 127114（我 10 / 真 9）、震裕转02 123282（我 4 / 真 7）
- `lt_bps` 12 只沿用冻结真值：10 只未抽取（113049 / 118050 / 118056 / 118064 /
  123114 / 127072 / 127105 / 127109 + EB 132024、132026）+ 2 只抽取无净资产句
  （118072 鼎通、123211 阳谷）
- P0 活样本：锦浪转02 123259（股东大会 2026-10-09）、易瑞转债 123220（2026-10-14）
  —— 唯一能外部验证底价预估的样本，**过期不候**
- 尚未写任何 `backend/` 业务代码，落地前需确认

详细口径见 `docs/superpowers/specs/` 三件套（design / runbook / handoff）。
