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
| `market-of-labs/store` | 私有（部署期） | **数据**：`sources/`（含 `versions` 账本）· `apps.json` · Releases | **不能**。它是唯一的事实 |
| **`market-of-labs/forge`（本仓库）** | 公有 | **只有 workflow**：3 个 YAML | 能。删了重建即可，什么都没存 |
| `market-of-labs/forge-core` | 私有 | **执行体**：Go 源码 + 构建 workflow + 二进制 Release | 不能（它是源码仓库） |
| `market-of-labs/companion` | 公有 | 伴侣应用源码。它的 Release 被 `store` 当成一个普通的 github 上游 | 不能（它是源码仓库） |

执行体是**编译后的二进制**：`strings` 与反汇编读不回设计意图，而源码留在私有仓库。
于是"跑"这个动作落在公开仓库的免费额度上，源码不必跟着公开。

---

## 三个 workflow

| workflow | 触发 | 干什么 |
|---|---|---|
| `on-dispatch.yml` | `repository_dispatch: store-event` · `workflow_dispatch`（**手动按钮**，`verb` 二选一） | 按事件分派：申请单 / `_incoming` 搬运 / 单应用收敛 / 全量对账 |
| `reconcile.yml` | `schedule`（每日 **18:17 UTC = 北京 02:17**）· `workflow_dispatch` | 全量对账（§4.4）。**幂等 = 漏跑自愈** |
| `rebuild-index.yml` | `workflow_dispatch`（**仅手动**） | 灾难恢复：从 Release 现状重建各来源的 `versions` 账本与 `apps.json` |

`store` 侧的 `forward.yml` 把事件**原样转告**过来 —— 它只发一个信标（事件名、issue 号、
sha），内容由执行体自己用 API 读。所以外部字符串进不了执行环境。

`on-dispatch.yml` 的手动按钮是**搬运 `_incoming` 唯一的两种叫法之一**（另一种是上传 CI
自己发的信标）：队列常驻 draft，没有"发布即搬运"那条自动路 —— 往 Release 上传/改名/删
asset 不触发任何事件（03 §3.3），传完文件不会有谁来替你发车。所以按钮的 `verb` 缺省值
是 `intake-incoming`（另一个选项 `reconcile` = 全量对账）。`verb` 用 `choice` 类型而不是
自由文本：取值由 GitHub 服务端**先校验**，进 job 时已经是一个封闭集合的成员。

三个 workflow 共用一条 `store-write` 并发队列，串行化消灭了"两个事件同时往同一个
Release 传 asset"这一整类竞态。

---

## 执行体从哪来

```yaml
- name: Fetch executor
  uses: robinraju/release-downloader@v1      # 第三方 action，见下面的代价
  with:
    repository: market-of-labs/forge-core
    latest: true                             # 不钉 tag ⇒ 浮动取 latest
    fileName: forge-linux-amd64
    out-file-path: .
    token: ${{ secrets.GH_PAT }}
```

**浮动取 latest，不钉 tag。** `forge-core` 打完 tag，本仓库一个字都不用改，下一轮对账
就换新版；要回滚就反过来做（把旧版本重新发成 latest 的成本，是刻意接受的，换来"改代码
不必动本仓库"）。

两处不能想当然：**`out-file-path` 只接受 `$GITHUB_WORKSPACE` 下的相对路径**，而且这个
action 按 **asset 原名**落盘、不改名 —— 所以二进制在 `$GITHUB_WORKSPACE/forge-linux-amd64`，
后面每一步都得用这个路径（不再是 `$RUNNER_TEMP/forge-bin`）。Release asset **不携带执行位**，
所以另有一个 `chmod +x` 的小步骤。

> ⚠️ **代价：这是本仓库唯一的第三方 action，而它跑的 job 握着能写三个仓库的 PAT。**
> 换之前这一步是 `gh release download` —— 那是一个**命令**而不是 action：runner 镜像
> 自带、GitHub 自己维护，不进 `uses:`、没有要升级或 review 的依赖树。换过来之后，每次
> 它跑起来都有一份第三方代码在 PAT 的权限下执行；换来的只是"用现成的 action"这件事本身。
>
> 版本取**主版本标签 `@v1`**（与仓库里其它 action 一致）。代价是维护者对 `v1` 的任何
> 一次推送都会直接进这个 job —— 钉 `@v1.13` 或 commit SHA 能挡住这个。取舍与理由见
> 03 §4.5 第 11 条。跟新主版本交给 `.github/dependabot.yml`（本仓库从没配过，这是它
> 第一个依赖）：它只**开 PR**、不合并、不打 tag，而本仓库的 workflow 也不在 PR 上
> 触发，所以既不改变线上跑的是什么，也不花分钟数。
>
> 它**自己不发 `::add-mask::`**（`src/` 里没有任何 `setSecret`），所以规则 9 那一步被
> 拆成了一个独立的 step 放在它**前面** —— 不能因为"action 没有 `run:` 可写"就不发。

⚠️ **这条链的性质：`forge-core` 打 tag = 上生产。** 因为那个二进制是**带着能写
`store` 的 PAT 执行的**。所以那边一个 tag 只允许对应一个二进制 —— 重跑同一个 tag
会被它的 build workflow 拒绝（加固清单第 10 条）。

---

## 加固清单（03 §4.5）

这些不是"最佳实践"，每一条都对应一个具体的、会真的发生的坏结果。改这个仓库之前
值得先读一遍。

| # | 规矩 | 防的是什么 |
|---|---|---|
| 1 | `store` 侧**根本不监听 `release`** | 裸 `on: release` = 全部 7 种动作，会在无关操作上发车；而写死 `[published]` 只服务"发布即发车"那一种设计 —— 队列常驻 draft 之后那条设计不存在了 |
| 2 | 搬运只从 `intake-incoming` 来（手动按钮 / 上传 CI 的信标），**没有闸门** | 旧闸门（`tag_name == '_incoming' && !prerelease`）的唯一职责是筛掉广播流里"不是队列发布"的事件。来源变成**显式指名**之后，广播流与闸门一起消失 —— 少一个能判错的地方 |
| 3 | 幂等：`_incoming` 无 asset → 退出；目标 asset 名已存在 → 跳过。⚠️ **出口仍要 PATCH `draft:true`**（无条件动作，不是"搬成功后的事"） | 重复 dispatch、并发重跑；以及**队列万一卡在 published** —— 那之后上传什么都不再触发，且网页上按不动 Publish，只能手工 Convert to draft |
| 4 | 内部 Release 操作一律 `make_latest: false` | 每次 draft→published 都重打 `published_at`，`/releases/latest` 会抖 |
| 5 | `intake-incoming` 的门槛 = 对 `forge` 的 **Contents: write**；本仓库是**公有**的，所以那份写权限名单必须短 | 这个按钮**直接引发一次对 `store` 的写入**。公有仓库的 `workflow_dispatch` 要写权限才能点，于是**协作者名单就是这条链的授权面** —— 加人之前得知道这一点 |
| 6 | **只用 unpublish（`draft: true`），绝不 delete** | 删 Release 会让 tag 消失；曾开过 Immutable Releases 则**永久烧毁该 tag**，而 tag = appId |
| 7 | 每个回写 commit 带 `[skip-dispatch]` | PAT 触发的 push **不被抑制**，会再次触发转发 → 死循环 |
| 8 | 禁 `set -x`、禁 `curl -v`、token 绝不拼进 URL | 公有仓库的 Actions 日志**任何登录用户都能读** |
| 9 | **每个碰 token 的 step 自己先发一次** `::add-mask::$TOKEN` —— 取件那一步现在是个 action、没有 `run:` 可写，所以那一句被拆成一个独立的 step 放在它**前面** | GitHub 自动脱敏的 token 模式表里只有 `ghp_/gho_/ghu_/ghs_/ghr_`，**不含 `github_pat_`** |
| 10 | 执行体只从 `forge-core` 的 Release 取；那边发布用 `overwrite_files: false`，tag 与二进制一一对应 | 浮动取 latest ⇒ 发布即上生产。若允许覆写 asset，cron 跑的代码会变、而 tag 没变，事后无迹可查。⚠️ 重跑已发过的 tag 是**跳过**（绿着过去），不是报错 |
| 11 | **握 PAT 的 job 只许用官方 action 或 runner 自带的 CLI；第三方 action 只许出现在不握 PAT 的 job 里**。⚠️ 本仓库**有一处已知的、明确接受的例外**：取执行体那一步是第三方 action（`robinraju/release-downloader`，见上面「执行体从哪来」），而它跑的正是握 PAT 的 job —— 代价与取舍写在那里 | 判据不是"这个 action 好不好"，是"**这段代码是谁写的、它在什么权限下跑**"。三个 job 握着能写三个仓库的 PAT（§6.2）⇒ 加一个第三方 action = 在这个权限下多跑一段别人的代码。**两个参照物别记反**：`forge-core/build.yml` 用第三方发布 action，它**不握 PAT**（只有本仓库的 `GITHUB_TOKEN`）；`store/forward.yml` **握着 PAT**，用的是官方 `actions/github-script` —— 它合规的理由是"官方"，**不是"没 PAT"** |

**规则 9 的落点改过两次。** 执行体自己在启动时也会发一句 `::add-mask::`，但那只覆盖
**它之后**的输出 —— 而取件那一步**先于**它运行、且已经握着 PAT。所以规矩先是变成
"**每个碰 token 的 step 自己发**"（`store/forward.yml` 早就是这个写法）。再后来取件从
`gh release download` 换成了 action，`Fetch executor` 连 `run:` 都没有了 —— 于是那一句
搬进了一个**独立的 `Mask token` step**，位置在 action **之前**：`::add-mask::` 是 job 级
生效的，放在前面一样覆盖 action 自己打的东西。

规则 7 的落点仍然在代码里（`job.Ctx.CommitBack` 是回写 store 的唯一出口，
无条件补上 `[skip-dispatch]`），只是那段代码现在住在 `forge-core`。

---

## 凭据

**一把 fine-grained PAT**，覆盖三个仓库：

| 仓库 | 需要 | 为什么 |
|---|---|---|
| `store` | Contents **R/W** + Issues **R/W** | 回写 `sources/`（含账本）、`apps.json`；建/改 Release 与 asset；读申请、回评、关单 |
| `forge`（本仓库） | Contents **R/W** | `repository_dispatch` 要的是目标仓库的 Contents，**不是 Actions**。只用前者，所以取更小的集合 |
| `forge-core` | Contents **R** | 下载执行体 Release 的 asset |

存放：**两个仓库都要解析得到 `GH_PAT`**（**仓库级或组织级都行**；现状是 `store` 用
仓库级、本仓库走组织级），**值填同一把**。**90 天轮换**（fine-grained PAT 最长 1 年，不设满）。

> ⚠️ **secret 名统一了，"读它的那个名字"没有 —— 别顺手改后者。**
> 同一个值在 workflow 里以两个身份出现：`token: ${{ secrets.GH_PAT }}` 是**取件那个
> action 的 input 名**，`STORE_TOKEN: ${{ secrets.GH_PAT }}` 是为了让那个 Go 二进制认
> （`internal/job/env.go` 的常量就叫 `STORE_TOKEN`）。**把后者改成别的名字会让二进制
> 静默读不到值。**
>
> 历史上这里还有第三个名字：`GH_TOKEN`。那是 `gh release download` 要的（`gh` 与 git
> 的 credential helper 只读这个名字）—— 取件换成 action 之后它就没有了。

三个 workflow 的 `permissions` 都是 `{}` —— 本仓库没有源码要 checkout，`GITHUB_TOKEN`
一个权限都用不上：取执行体走 PAT，写 `store` 也走 PAT（`GITHUB_TOKEN` 跨不了仓库）。
本仓库**一次 checkout 都没有**：没有源码要取，也就没有工作副本里的 `.git/config` 可以
把 PAT 漏进去 —— ⚠️ 但这**不等于**"PAT 碰不到本仓库"，取件那个 action 是拿着它的（规则 11）。

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
