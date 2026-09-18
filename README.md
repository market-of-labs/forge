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
| **`market-of-labs/forge`（本仓库）** | 公有 | **只有 workflow**：3 个 YAML + 2 个复合动作 | 能。删了重建即可，什么都没存 |
| `market-of-labs/forge-core` | 私有 | **执行体**：Go 源码 + 构建 workflow + 二进制 Release | 不能（它是源码仓库） |

⚠️ 这张表**从前有第四格** `market-of-labs/companion`（伴侣应用源码，D55 转私有）—— 它随 D58 的路线切换
**已删除**：客户端外部化成别人的 F-Droid 客户端之后，"应用源码放哪"这个问题不存在了（market-spec 03 §1）。

执行体是**编译后的二进制**：`strings` 与反汇编读不回设计意图，而源码留在私有仓库。
于是"跑"这个动作落在公开仓库的免费额度上，源码不必跟着公开。

---

## 三个 workflow

**一个功能一个文件。** 路由在**文件名与触发面**上，不在任何 `if:` 或 Go 里的 switch 上 ——
想知道"点了这个按钮会跑哪件事"，看文件名就够了。

| 界面显示 | workflow | 触发 | 干什么 |
|---|---|---|---|
| **`处理申请单`** | `source-change.yml` | `repository_dispatch: source-change` · `workflow_dispatch`（**手动按钮**，入参只有一个 issue 号） | 处理某一张申请单（§2.5） |
| **`搬运队列`** | `intake-incoming.yml` | `repository_dispatch: intake-incoming` · `workflow_dispatch`（**手动按钮**，无入参） | 搬 `_incoming` 队列 + 重建索引（§3.2） |
| **`全量对账`** | `reconcile.yml` | `schedule`（每日 **18:17 UTC = 北京 02:17**）· `repository_dispatch: reconcile` · `workflow_dispatch` | 全量对账（§4.4）。**幂等 = 漏跑自愈** |

⚠️ **第一列（`name:`）是给人看的，路由靠的是第二列那个文件名，两者互不影响** ——
`repository_dispatch: types:` 里的 `source-change` / `intake-incoming` / `reconcile` 是**两个仓之间的
线协议**，`store` 侧发出的词必须与这里逐字相同，改 `name:` 一个字都不会动到它。
`store` 侧的同名文件在界面上叫 **`申请单·转发` / `搬运队列·转发`**（名字里的"转发"就是
它俩的分工差别：那边只转告，这边才动手）。

⚠️ **`reconcile` 这个词目前没有人发**（2026-09-18）：`store` 侧那个只做转发的 `reconcile.yml` 已删除，
对账的三个触发面 —— 每日 cron、手动按钮、`repository_dispatch` —— **全部落在本文件上**，
所以"今天跑了几次对账"只在**一个仓库**里数得清。上面那一行里的 `repository_dispatch: reconcile`
是**有意留着**的监听面（一条已发布出去的线协议，撤掉是不可逆的：以后外部脚本或 bookmarklet 再发它
就是 204 静默丢弃），目前**无发送方**。本仓库的 `store` 侧入口文件也因此只剩两个：`source-change.yml`
（申请单）与 `intake-incoming.yml`（搬队列 + 它的手动按钮）。

从前这三个是一个 `on-dispatch.yml`：一个文件收下所有事件，再交给执行体的
`handle-dispatch` 按 payload 里的一个字符串分派。那条走法有三处代价 —— 入口的真假要靠读
四段式的 `||` 表达式才知道；"漏填 `verb`"就能静默跑错功能；两个 workflow 之间有十几行
逐字重复（已抽成 `.github/actions/` 下两个复合动作）。

⚠️ **`on-dispatch.yml` 仍在库里，但它是灰度的兼容壳、不是第四种设计** —— 拆分的顺序不能反：
`release` 事件跑的是 **tag 所指提交**上的 workflow 文件，而 `_incoming` 的 tag 引用要等
`cleanIncoming` 把旧引用删掉、下一次 Publish 重建之后才会指向新文件。在那之前 Publish 那条路
跑的仍是旧提交上那份 `forward.yml`，发出来的还是 `types: [store-event]`，所以这条监听必须活到
**确认引用刷新过**之后才删（连同 `handle-dispatch` 动词、`FDROID_*` secrets 一起，market-spec 03 §4.7 第 5 步）。

⚠️ **这条约束只对 `on-dispatch.yml` 成立。** `spike-fdroid-repo.yml` 曾经也在第 5 步那一格里，
**已于 2026-09-18 单独删掉** —— 它不写 `store`、不握任何 PAT、不进 `store-write` 队列，与灰度顺序
**零耦合**，删它不需要等任何东西（它留下的证据在 `market-spec/spike/FDROID-REPO-REFS.md` §3.1）。

从前的 `rebuild-index.yml`（手动跑的"账本灾难恢复"）也删掉了（D56）：账本是
派生数据，而它唯一的自动恢复途径本来就在日常路径上 —— 账本里没有 `upstreamTag` 的
最新版本会在下一轮对账里被重新镜像，重读上游 APK 时元数据就填回来了（`recordIndex`）。
真出了 git 事故，回滚 `sources/{appId}.json` 比下载更准、给得还更多。

`store` 侧的同名文件（两个 —— `source-change.yml` / `intake-incoming.yml`；对账没有对面的文件，
见上）把事件**原样转告**过来 —— 只发一个信标（issue 号、ref、sha），
内容由执行体自己用 API 读。所以外部字符串进不了执行环境。

`intake-incoming.yml` 的手动按钮是**搬运 `_incoming` 三条路之一**：日常那条是人上传完在
`store` 的 Release 页面上点 **Publish**（`release: published` → `store` 侧翻译成
`intake-incoming`，D57），第三条是上传 CI 自己发的信标。**这一条是兜底**：Publish 那条路
没反应时点它，效果一样 —— 往 Release 上传/改名/删 asset 不触发任何事件（03 §3.3），
所以发车必须挂在别的动作上，而那条链一旦哑掉就得有人能手动叫。

> **日常点是 `store` 那个，不是这里这个。** 两者等效（都是同一个 `event_type`），但人是在
> `store` 的 Release 页面上传的 APK，按钮就在同一页的 Actions 里 —— 不用切仓库。本仓库
> 这个留着，是因为改执行体的人常在这儿，能顺手看一眼 run 的日志。

三个 workflow 共用一条 `store-write` 并发队列，串行化消灭了"两个事件同时往同一个
Release 传 asset"这一整类竞态（也包括那盘 APK cache：两个写入口共用一盘，`actions/cache`
的"存"发生在 job 末尾，并发就会互相覆盖）。

---

## 执行体从哪来

三个入口文件在这一段上都只写两行：

```yaml
- name: 装工具链
  uses: $/.github/actions/setup-fdroid
- name: 取执行体
  uses: $/.github/actions/fetch-executor
  with: { token: ${{ secrets.GH_PAT }} }
```

**为什么抽成复合动作**：这段从前逐字出现在两个文件里（apt 清单、掩码、cache、取件、
`chmod`），而它埋着两处"必须一致、错了却是静默失败"的东西 —— `EXEC_ASSET` 这个名字
（改一处忘一处就是"下载成功但这个文件不存在"）与 cache 的 key 前缀。现在调用方只认
动作导出的 `$FORGE_EXEC` / `$APK_CACHE_DIR` 两个环境变量，常量只在一个文件里存在。
逐条理由（为什么 rolling key、为什么 cache 目录必须在 store 克隆之外）写在
`.github/actions/fetch-executor/action.yml` 里。

> ⚠️ **引用写 `$/` 而不是 `./`。** `./.github/actions/x` 是工作区相对路径，要求先
> `actions/checkout`；而这三个 job 跑在 `debian:trixie` 里、**那个镜像没有 git** ——
> 装 git 的恰好就是我们要复用的这个动作，闭环。`$/` 是 GitHub 的自仓库语法，解析到
> **本仓库、当前正在跑的那个 commit**，不需要 checkout。
>
> 顺带把"pin 一个 SHA 就什么都定住"这件事做对了：`$/` 跟着正在跑的 commit 走，
> 而 `./` 取的是工作区里那份，两者可以不一致。

**浮动取 latest，不钉 tag。** `forge-core` 打完 tag，本仓库一个字都不用改，下一轮对账
就换新版；要回滚就反过来做（把旧版本重新发成 latest 的成本，是刻意接受的，换来"改代码
不必动本仓库"）。**这条有个要紧的推论**：删掉执行体的一个动词之前，必须先让新二进制
发出去 —— 否则旧 workflow 下一轮就会用上刚发布的新二进制。

两处不能想当然：**`out-file-path` 只接受 `$GITHUB_WORKSPACE` 下的相对路径**，而且这个
action 按 **asset 原名**落盘、不改名 —— 所以二进制在 `$GITHUB_WORKSPACE/forge-linux-amd64`。
Release asset **不携带执行位**，所以动作里另有一个 `chmod +x` 的小步骤。

> ⚠️ **代价：这是本仓库唯一的第三方 action，而它跑的 job 握着能跨四个仓库的 PAT。**
> 换之前这一步是 `gh release download` —— 那是一个**命令**而不是 action：runner 镜像
> 自带、GitHub 自己维护，不进 `uses:`、没有要升级或 review 的依赖树。换过来之后，每次
> 它跑起来都有一份第三方代码在 PAT 的权限下执行；换来的只是"用现成的 action"这件事本身。
>
> 版本取**主版本标签 `@v1`**（与仓库里其它 action 一致）。代价是维护者对 `v1` 的任何
> 一次推送都会直接进这个 job —— 钉 `@v1.13` 或 commit SHA 能挡住这个。取舍与理由见
> 03 §4.5 第 11 条。跟新主版本交给 `.github/dependabot.yml`：它只**开 PR**、不合并、
> 不打 tag，而本仓库的 workflow 也不在 PR 上触发，所以既不改变线上跑的是什么，
> 也不花分钟数。
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
| 1 | `store` 侧**只放行队列那一种 `release`**（`on: release: [published]` + job 级 `if:` 筛 `tag_name == '_incoming'`，两条**同进同出**，见 03 §2.6） | 裸 `on: release` = 全部 7 种动作，会在无关操作上发车；而不加闸门的 `[published]` 会把 `store` 里**任何** Release 的发布都转告过来 —— 包括每个 app 自己的正式版本。⚠️ **症状已经不是红了**：从前 forge 对不认识的事件**硬错**，现在 `types:` 一命中就会真跑一轮搬运 —— 空队列会早退（03 §4.6），所以代价从"多一条红色告警"变成了**每一轮白跑一次完整的 `intake-incoming`**（下载 APK 凑 cache、`fdroid update`），静默地烧分钟数。**这比红更难发现** —— 闸门不能因为"不会再报错了"就删掉 |
| 2 | 搬运只从 `intake-incoming` 来（Publish 被 `store` 翻译成它 / 手动按钮 / 上传 CI 的信标），**本仓库侧没有闸门** | 旧闸门（`tag_name == '_incoming' && !prerelease`）的职责是筛掉广播流里"不是队列发布"的事件，现在筛在**广播流的入口**（`store`，规则 1），forge 连收都收不到 —— 没有"收到不认识的事件"这一整类故障。⚠️ 别在这边补一道同样的检查：同一件事有两处判断 = 两处能判错的地方 |
| 3 | 幂等：`_incoming` 无 asset → 退出；目标 asset 名已存在 → 跳过。⚠️ **出口仍要 PATCH `draft:true` + `tag_name`**（无条件动作，不是"搬成功后的事"），并在 unpublish 成功后**删掉 `refs/tags/_incoming`** | 重复 dispatch、并发重跑；把"人点的那次 Publish"收回去；以及**下次 Publish 解析到旧的 `intake-incoming.yml`** —— `release` 事件跑的是 tag 所指提交上那份文件，而引用一旦建立就不再移动（03 §3.2）。⚠️ PATCH 省掉 `tag_name` 会把队列名降级成 `untagged-<sha>`，之后谁也认不出它 |
| 4 | 内部 Release 操作一律 `make_latest: false` | 每次 draft→published 都重打 `published_at`，`/releases/latest` 会抖 |
| 5 | `intake-incoming` 有两个按钮入口，门槛都是**该仓库的 write**：本仓库的 `intake-incoming.yml`（**公有**仓，写权限名单必须短）与 `store` 的同名文件（名单 = 本来就够得着 `_incoming` 的那批人，见 03 §4.5 第 5 条）。⚠️ 下拉框没了：两边各自只干自己那一件事，**不可能选错** | 这两个按钮**直接引发一次对 `store` 的写入**。`workflow_dispatch` 要写权限才能点，于是**协作者名单就是这条链的授权面** —— 加人之前得知道这一点。⚠️ `store` 那个**不扩大**授权面：能点它的人本来就能往 `_incoming` 传文件 |
| 6 | **只用 unpublish（`draft: true`），绝不 delete** | 删 Release 会让 tag 消失；曾开过 Immutable Releases 则**永久烧毁该 tag**，而 tag = appId |
| 7 | 每个回写 commit 带 `[skip-dispatch]` | PAT 触发的 push **不被抑制**，会再次触发转发 → 死循环 |
| 8 | 禁 `set -x`、禁 `curl -v`、token 绝不拼进 URL | 公有仓库的 Actions 日志**任何登录用户都能读** |
| 9 | **每个碰 token 的 step 自己先发一次** `::add-mask::$TOKEN` —— 取件那一步现在是个 action、没有 `run:` 可写，所以那一句被拆成一个独立的 step 放在它**前面**（现在是复合动作 `fetch-executor` 的**第一步**） | GitHub 自动脱敏的 token 模式表里只有 `ghp_/gho_/ghu_/ghs_/ghr_`，**不含 `github_pat_`** |
| 10 | 执行体只从 `forge-core` 的 Release 取；那边发布用 `overwrite_files: false`，tag 与二进制一一对应 | 浮动取 latest ⇒ 发布即上生产。若允许覆写 asset，cron 跑的代码会变、而 tag 没变，事后无迹可查。⚠️ 重跑已发过的 tag 是**跳过**（绿着过去），不是报错 |
| 11 | **握 PAT 的 job 只许用官方 action 或 runner 自带的 CLI；第三方 action 只许出现在不握 PAT 的 job 里**。⚠️ **这条要穿透复合动作来算**：自己写的复合动作不算第三方代码，但它里面 `uses` 的每一个 action 仍然按这条逐条算 —— 包一层**不会**洗白。⚠️ 本仓库**有一处已知的、明确接受的例外**：取执行体那一步是第三方 action（`robinraju/release-downloader`，见上面「执行体从哪来」），而它跑的正是握 PAT 的 job —— 代价与取舍写在那里 | 判据不是"这个 action 好不好"，是"**这段代码是谁写的、它在什么权限下跑**"。三个 job 握着能跨四个仓库的 PAT（§6.2）⇒ 加一个第三方 action = 在这个权限下多跑一段别人的代码。复合动作是"一段被复用的 YAML"，不是"一段被信任的代码"：授权面看的是最终执行的那段代码是谁写的，判据一旦挂到"文件是不是我们写的"上就能被绕过。**两个参照物别记反**：`forge-core/build.yml` 用第三方发布 action，它**不握 PAT**（只有本仓库的 `GITHUB_TOKEN`）；`store/.github/actions/beacon` **握着 PAT**，用的是官方 `actions/github-script` —— 它合规的理由是"官方"，**不是"没 PAT"** |

**规则 9 的落点改过两次。** 执行体自己在启动时也会发一句 `::add-mask::`，但那只覆盖
**它之后**的输出 —— 而取件那一步**先于**它运行、且已经握着 PAT。所以规矩先是变成
"**每个碰 token 的 step 自己发**"（`store/.github/actions/beacon` 早就是这个写法）。
再后来取件从 `gh release download` 换成了 action，连 `run:` 都没有了 —— 于是那一句搬进
了一个独立的 `Mask token` step，位置在 action **之前**：`::add-mask::` 是 job 级生效的，
放在前面一样覆盖 action 自己打的东西。**最后它成了 `fetch-executor` 这个复合动作的第一步**
（`Mask the token and export the cache dir`）—— 位置的性质没变，只是现在由动作自己保证
"掩码一定在取件之前"，而不是靠调用方记得排对顺序。

规则 7 的落点仍然在代码里（`job.Ctx.CommitBack` 是回写 store 的唯一出口，
无条件补上 `[skip-dispatch]`），只是那段代码现在住在 `forge-core`。

---

## 凭据

**一把 fine-grained PAT**，覆盖四个仓库：

| 仓库 | 需要 | 为什么 |
|---|---|---|
| `store` | Contents **R/W** + Issues **R/W** | 回写 `sources/`（含账本）、`apps.json`；建/改 Release 与 asset；读申请、回评、关单 |
| `forge`（本仓库） | Contents **R/W** | `repository_dispatch` 要的是目标仓库的 Contents，**不是 Actions**。只用前者，所以取更小的集合 |
| `forge-core` | Contents **R** | 下载执行体 Release 的 asset |
| `companion` | Contents **R** | 把它的 Release 当**上游镜像**（列 Release、下 asset）。它是**私有仓**（D55）—— 匿名读不到，而读它的那个客户端本来就带这把 PAT，所以只需在这一行加个仓库、**零代码改动** |

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
