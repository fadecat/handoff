# handoff

换机交接物料仓库。与代码仓库 [`fadecat/web`](https://github.com/fadecat/web)
（分支 `codex/bond-lifecycle-vnext`）配套使用。

> 为什么要单独一个仓库：`web` 是业务代码库，不适合塞 2.5GB 的说明书 PDF。
> 但这些 PDF 是 310 次 LLM 抽取的原始输入，重爬成本高，所以单独存放。

---

## 目录

| 路径 | 大小 | 说明 |
|---|---|---|
| `datasets/` | 2.5 GB | 说明书原始物料（**本仓库的主体**） |
| `repo.bundle` | 6 MB | `web` 仓库完整快照，不依赖 GitHub 的克隆兜底 |
| `memory/` | 136 KB | 项目记忆副本（正本在 `web` 的 `docs/research/memory-archive/`） |
| `MIGRATION-CHECKLIST.md` | 9 KB | 换机迁移清单（推送 / 密钥 / 落地验证 / 未完工作） |

### `datasets/` 细分

| 子目录 | 大小 | 内容 |
|---|---|---|
| `prospectus-lean/` | 2.4 GB | **314 份募集说明书 PDF** + 310 份抽取 JSON |
| `prospectus-batch/` | 24 MB | LLM proposal 内容寻址缓存 + sqlite |
| `prospectus-out/` | 15 MB | 同上 |
| 根下 5 个 JSON | 2.2 MB | `webapi_no_cookie` / `webapi_with_member_cookie`、下修审计 v2/v3/unknown |

单文件最大 21.7 MB，未超 GitHub 100 MB 限制。

---

## 新机器怎么用

```bash
# 1. 代码仓库
git clone -b codex/bond-lifecycle-vnext https://github.com/fadecat/web.git web

# 2. 大件物料（本仓库）
git clone https://github.com/fadecat/handoff.git handoff

# 3. 放回工作区（脚本默认从这里读）
cp -r handoff/datasets web/.workbuddy/datasets

# 4. 恢复记忆，让新环境自动读到
mkdir -p web/.workbuddy/memory
cp handoff/memory/*.md web/.workbuddy/memory/
```

代码仓库推不上去时的兜底（完全不经过 GitHub）：

```bash
git clone handoff/repo.bundle web
cd web && git checkout bond-lifecycle-vnext
```

---

## 别忘了

- `.env` **不在这个仓库里**（含 tushare / deepseek / 集思录密钥），需手动带到新机器。
  键名清单见 `MIGRATION-CHECKLIST.md`。
- `web` 仓库里已入库的关键资产：309 份说明书 **LLM 抽取结果**（7.4 MB，最贵资产）、
  会员真值快照、全部脚本与文档四件套。**本仓库只是原始 PDF，丢了还能按链接重下**
  （`prospectus_link_scan_v2.json` 在 `web` 里）。
- P0 时效样本：锦浪转02（123259）股东大会 2026-10-09、易瑞转债（123220）2026-10-14
  —— 唯一能外部验证底价预估的活样本，过期不候。
