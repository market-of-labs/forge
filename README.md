# forge —— 私密市场的运行器

本仓库**不含任何 Go 源码**。它只做一件事：按事件把**执行体**取下来，跑一遍。

执行体（Go 源码、构建、二进制 Release）在私有的 **`market-of-labs/forge-core`**，
本仓库的三个 workflow 从它的 Release 里拉最新的二进制来执行。程序本身怎么跑、
有哪些动词、怎么本地调试 —— 见那个仓库的 README。

设计文档：`03-maintenance-action.md` 定义这套维护动作（下文凡写 `02 §x` / `03 §x`
都是指 `02-manifest-schema.md` 与它）。

---

## 为什么拆成两个仓库

`store` 必须私有（它装着全部数据与 Release），而 **GitHub 对私有仓库的 Actions
分钟数计费**。每日对账是这套流程里唯一耗时的大头 —— 遍历所有上游、可能下载若干 APK ——
所以它必须跑在一个**公有**仓库里。

但「逻辑要跑在公有仓库里」不等于「逻辑要公开」。于是拆成两半：公有这边只留 workflow
（触发面 + 加固 + 凭据隔离），私有那边放执行体。

| 仓库 | 可见性 | 装什么 | 能不能丢 |
|---|---|---|---|
| `market-of-labs/store` | 私有（部署期） | **数据**：`sources/` · `apps.json` · `store/index.json` · Releases | **不能**。它是唯一的事实 |
| **`market-of-labs/forge`（本仓库）** | 公有 | **只有 workflow**：3 个 YAML | 能。删了重建即可，什么都没存 |
| `market-of-labs/forge-core` | 私有 | **执行体**：Go 源码 + 构建 workflow + 二进制 Release | 不能（它是源码仓库） |
| `market-of-labs/companion` | 公有 | 伴侣应用源码。它的 Release 被 `store` 当成一个普通的 github 上游 | 不能（它是源码仓库） |

执行体是**编译后的二进制**：`strings` 与反汇编读不回设计意图，而源码留在私有仓库。
于是"跑"这个动作落在公开仓库的免费额度上，源码不必跟着公开。

---

## 三个 workflow

| workflow | 触发 | 干什么 |
|---|---|---|
| `on-dispatch.yml` | `repository_dispatch: store-event` · `workflow_dispatch` | 按事件分派：申请单 / `_incoming` 搬运 / 单应用收敛 / 全量对账 |
| `reconcile.yml` | `schedule`（每日 03:17 UTC）· `workflow_dispatch` | 全量对账（§4.4）。**幂等 = 漏跑自愈** |
| `rebuild-index.yml` | `workflow_dispatch`（**仅手动**） | 灾难恢复：从 Release 现状重建 `index.json` 与 `apps.json` |

`store` 侧的 `forward.yml` 把事件**原样转告**过来 —— 它只发一个信标（事件名、issue 号、
release tag、sha），内容由执行体自己用 API 读。所以外部字符串进不了执行环境。

三个 workflow 共用一条 `store-write` 并发队列，串行化消灭了"两个事件同时往同一个
Release 传 asset"这一整类竞态。

---

## 执行体从哪来

```bash
gh release download --repo market-of-labs/forge-core \
  --pattern forge-linux-amd64 --output "$RUNNER_TEMP/forge-bin" --clobber
```

**浮动取 latest，不钉 tag。** 仓库**只能**由 `--repo` 指定，**位置参数是 tag** ——
所以"不写位置参数"本身就是"取 latest"，这一点是刻意的。反过来，把仓库名写成位置参数
（`gh release download market-of-labs/forge-core`）是一处**踩过的坑**：gh 会把它当 tag，
然后去当前目录找 `.git` 猜仓库，而本仓库的 job 故意不 checkout，于是必然失败于
`failed to run git: fatal: not a git repository`（2026-09-11）。

`forge-core` 打完 tag，本仓库一个字都不用改，下一轮对账就换新版；要回滚就反过来做
（把旧版本重新发成 latest 的成本，是刻意接受的，换来"改代码不必动本仓库"）。

⚠️ **这条链的性质：`forge-core` 打 tag = 上生产。** 因为那个二进制是**带着能写
`store` 的 PAT 执行的**。所以那边一个 tag 只允许对应一个二进制 —— 重跑同一个 tag
会被它的 build workflow 拒绝（加固清单第 10 条）。

---

## 加固清单（03 §4.5）

这些不是"最佳实践"，每一条都对应一个具体的、会真的发生的坏结果。改这个仓库之前
值得先读一遍。

| # | 规矩 | 防的是什么 |
|---|---|---|
| 1 | `store` 侧死写 `on: release: types: [published]` | 裸 `on: release` = 全部 7 种动作，会在无关操作上发车 |
| 2 | 闸门：`tag_name == '_incoming'` 且 `prerelease == false` | `published` **对预发布同样触发** |
| 3 | 幂等：`_incoming` 无 asset → 退出；目标 asset 名已存在 → 跳过 | 重复 Publish、重复 dispatch、并发重跑 |
| 4 | 内部 Release 操作一律 `make_latest: false` | 每次 draft→published 都重打 `published_at`，`/releases/latest` 会抖 |
| 5 | 判操作者用 `github.triggering_actor`，不用 `release.author` | `release.author` 是原创建者，重发布时会误判 |
| 6 | **只用 unpublish（`draft: true`），绝不 delete** | 删 Release 会让 tag 消失；曾开过 Immutable Releases 则**永久烧毁该 tag**，而 tag = appId |
| 7 | 每个回写 commit 带 `[skip-dispatch]` | PAT 触发的 push **不被抑制**，会再次触发转发 → 死循环 |
| 8 | 禁 `set -x`、禁 `curl -v`、token 绝不拼进 URL | 公有仓库的 Actions 日志**任何登录用户都能读** |
| 9 | **每个碰 token 的 step 自己先发一次** `::add-mask::$TOKEN` | GitHub 自动脱敏的 token 模式表里只有 `ghp_/gho_/ghu_/ghs_/ghr_`，**不含 `github_pat_`** |
| 10 | 执行体只从 `forge-core` 的 Release 取；那边发布用 `overwrite_files: false`，tag 与二进制一一对应 | 浮动取 latest ⇒ 发布即上生产。若允许覆写 asset，cron 跑的代码会变、而 tag 没变，事后无迹可查。⚠️ 重跑已发过的 tag 是**跳过**（绿着过去），不是报错 |
| 11 | **本仓库的 workflow 只用第一方 action，一个第三方 action 都不许加** | 这三个 job 里握着能写三个仓库的 PAT（§6.2）。第三方 action = 在这个权限下多跑一段别人的代码；而它们要做的只是 `gh release download` + 执行我们的二进制，`gh` 是 runner 镜像自带的第一方 CLI，够用。**对照组**：`store/forward.yml` 用的是官方 `actions/github-script`，`forge-core/build.yml` 用第三方发布 action —— 那两个 job 都不握 PAT |

**规则 9 的落点在这套拆分之后变了。** 执行体自己在启动时也会发一句 `::add-mask::`，
但那只覆盖**它之后**的输出 —— 而 `Fetch executor` 那一步**先于**它运行、且已经握着
PAT。所以规矩从"二进制负责发"变成"**每个碰 token 的 step 自己发**"，`Fetch executor`
也不例外。`store/forward.yml` 早就是这个写法，本仓库现在跟上。

规则 7 的落点仍然在代码里（`job.Ctx.CommitBack` 是回写 store 的唯一出口，
无条件补上 `[skip-dispatch]`），只是那段代码现在住在 `forge-core`。

---

## 凭据

**一把 fine-grained PAT**，覆盖三个仓库：

| 仓库 | 需要 | 为什么 |
|---|---|---|
| `store` | Contents **R/W** + Issues **R/W** | 回写 `sources/`、`apps.json`、`index.json`；建/改 Release 与 asset；读申请、回评、关单 |
| `forge`（本仓库） | Contents **R/W** | `repository_dispatch` 要的是目标仓库的 Contents，**不是 Actions**。只用前者，所以取更小的集合 |
| `forge-core` | Contents **R** | 下载执行体 Release 的 asset |

存放：**每个仓库各存一份，secret 名统一叫 `GH_PAT`**（本仓库一份、`store` 一份，
**两份填同一把值**）。**90 天轮换**（fine-grained PAT 最长 1 年，不设满）。

> ⚠️ **secret 名统一了，环境变量名没有 —— 别顺手改后者。**
> 同一个值在 workflow 里以两个身份出现：`GH_TOKEN: ${{ secrets.GH_PAT }}`
> 是为了让 `gh release download` 认（`gh` 和 git 的 credential helper 只读 `GH_TOKEN`），
> `STORE_TOKEN: ${{ secrets.GH_PAT }}` 是为了让那个 Go 二进制认（`internal/job/env.go`
> 的常量就叫 `STORE_TOKEN`）。**统一成同一个 env 名会让其中一方静默读不到值。**

三个 workflow 的 `permissions` 都是 `{}` —— 本仓库没有源码要 checkout，`GITHUB_TOKEN`
一个权限都用不上：取执行体走 PAT，写 `store` 也走 PAT（`GITHUB_TOKEN` 跨不了仓库）。
这也是为什么本仓库**一次 checkout 都没有**：没有工作副本，PAT 就碰不到 `.git/config`。

---

## 上线前的部署检查

见 03 §7。与本仓库最相关的一条：**`store` 及其 org 的「Immutable releases」必须为 OFF**。
一旦开启，Release 的 asset 不能增删改、tag 不能删或移动 —— 追加上传（§3.1）、
`_incoming` 清场（§3.2）、unpublish 复用 tag 全部失效。

---

## 第一期已知的临时状态

`store/endpoints.json` 指向 GitHub 直链，所以**CF 隐藏网关这条红线暂时失守**：
真实的 `owner/repo` 会出现在 `apps.json` 里。这是**明确接受的**临时状态（D22），
部署期换一行地址模板即恢复，客户端不需要任何改动。
