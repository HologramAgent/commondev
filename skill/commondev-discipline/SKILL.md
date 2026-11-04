---
name: commondev-discipline
description: commondev 主管式多仓库开发流水线的**总纲与使用者纪律**。当你要为某个工程开一条分支、把工程派给一个下属 agent、或拉起执行后端 agent 干活时必读。核心动作：为每个工程对话新建驻留式子 agent 并把该工程的一对通道划给它（否则所有工程都挤在根 agent 上）。另含**红牌**与主管行为红线。★ git 平台操作（Gitea/GitHub/GitLab 的认证头、API、PR、分支保护）见技能 commondev-git；★ hmap 插件开发与部署纪律见技能 commondev-hmap。
version: 3.0.0
author: HomeAgent
---

## ★ 本插件默认不自启：先拉起再用

commondev 的 44 个工具占约 726 tokens，是本组最大的。因此它**默认不自启**
（`auto_start: false`），不在初始工具清单里。

需要开工程线/派活时：

```
plugin_start  name=commondev
```

**用完必须关掉**（44 个工具常驻会持续吃上下文）：

```
plugin_stop  name=commondev
```

### 什么时候可以关

安全关闭的判据是"**没有在跑的工程线**"：

1. 先查有没有活着的 sidecar：`commondev_agent_state`（看 `agent_alive`）
2. 有在跑的线，先停：`commondev_agent_down`（暂停）或
   `commondev_agent_interrupt`（只停当前这一轮，保留会话）
3. 确认都不在跑，再 `plugin_stop`

★ **不要在有 sidecar 在跑时直接 disable** —— 进程会变成孤儿。
插件 `Stop()` 里有幂等全清（sidecar / 执行后端进程一个不留），但那是兜底，
不是正常路径。

### 关闭不影响已做的活

台账（`ledger.json`）、工作目录、分支状态、凭据全部**持久化在磁盘**。
下次 `plugin_start` 后 `commondev_project_list` / `commondev_branch_status`
能看到之前的一切，`commondev_agent_up` 能接着用原来的 session 续投。


本技能是**总纲**。三部分按场景拆成了独立技能，避免一次吞掉上万 token：

| 你要做的事 | 读哪个技能 |
|---|---|
| 开工程、派活、收活、判定完成（使用者纪律） | **本技能**（往下读） |
| 开 PR、审批、分支保护、平台凭据（Gitea/GitHub/GitLab） | `skill_info("commondev-git")` |
| 写/装/调 hmap 插件，插件排查与安全纪律 | `skill_info("commondev-hmap")` |

- **第一部分：使用者纪律** —— 怎么开工程、怎么派活。
- **第二部分：★ git 平台操作** —— **本文是权威**。开 PR、审批、分支保护怎么做。
- **第三部分：开发与部署纪律** —— 怎么不踩坑。

> ## ★ 架构边界（用户拍板 2026-10-15）
>
> | 层 | 负责 |
> |---|---|
> | **插件 commondev** | 项目台账、工作目录、执行后端会话与通道、判据与变异复核、冲突预检、凭据保存与安全执行 |
> | **本文（skill）** | **各 git 托管平台的认证头、API 端点、PR 与分支保护流程** |
>
> **插件里没有平台逻辑。** 操作 Gitea/GitHub/GitLab 等平台一律：
> 读本文 → 用 `enc_shell` + `@{凭据名}` 执行 `curl`/`gh`/`tea`。
>
> ⇒ **新增一个托管平台只需改本文，不需要改插件、不需要重编译、不需要重启服务。**
>
> 原本这些在插件里（`platformSpec` / `commondev_account_*` / `commondev_pr_*` /
> `commondev_protect_branch` / `commondev_repo_search`），代价是：
> 每接一个平台都要改 Go 代码 → 重编译 → 重打包 → 重启，
> 且**未列出的平台直接报错**（GitLab/Gitee/Bitbucket 全不支持）——
> 而 agent 用 curl 本来什么平台都能打。这 10 个工具已移除。

> 看不懂某个词时的权威文档（按顺序读）：
> 1. `docs/zh/commondev-design.md` —— 目标机制、台账 schema、判据与变异、安全（§12 是架构变更）。
> 2. `docs/zh/commondev-agent-runtime-design.md` —— 运行时与内核接入（续篇）。
> 3. `docs/zh/kernel-secret-firewall-design.md` —— 凭据模块（enc_* 的底座）。
> 4. `docs/zh/resident-subagent-design.md` —— 驻留子设计的原始文档。
>
> **本文与文档冲突时，以文档为准，并回来修正本文。**

---

## 使用者纪律

> **★ 设计总纲 ［已定］**：插件对执行后端 agent 做的事，本质就是
> **「把给执行后端发消息包装一下、把执行后端返回包装一下」**，深度融合内核的 IO 抽象与中断系统
> —— 即 **执行后端 agent = 一个 Device**（默认后端是 pi，也可能接 zcode / dsh 等）。**不要自己造调度器 / 会话路由 / 中断栈**，
> 全部接到内核已有的那套上（`InjectInputTo` / `RegisterOutputChannel` / `InjectInterrupt*Opts`）。

### 21. 角色与单位

| 角色 | 是谁 | 干什么 |
|---|---|---|
| **人** | 老板 | 只与主管对接 |
| **主管** | 你（长期驻留的 HomeAgent 主 agent） | 拆任务、**定义判据**、复审、决定合入 |
| **驻留子 agent** | 每条工程对话的项目分管（resident 子） | 只与执行后端 agent 打交道、向上（父 agent）汇报；不越级直接给执行层下结论 |
| **执行后端 agent** | 按工程拉起的一个执行后端实例（由 `agent_cli` 决定，默认 pi，也可选 zcode / dsh 等） | 在某棵工作树里干活 |
| **commondev 插件** | 外部独立插件 | 仓库/凭据/分支/PR/测试，全部机械化操作 |

> **★ 四层不可混用**：人 ↔ 主管 ↔ **驻留子 agent（项目分管）** ↔ **执行后端 agent**。
> 主管只对接项目分管，**不直接给执行层投任务**；项目分管只与执行后端打交道、
> 有结论向父 agent 汇报。
>
> **⚠ 别把 `agentcli` 当成执行后端**：`agentcli` 插件管的是**交互式终端会话**
> （创建 / 读 / 写终端），跟执行后端 agent **没有任何关系** —— 它既不拉起执行后端，
> 也不参与工程投递。名字像而已，别混为一谈。
>
> **⚠ 执行层 ≠ pi**：后端由每条线自己的 `agent_cli` 决定（默认 pi，也可选 zcode / dsh 等），
> **多条工程线可能各用不同后端**。凡指「执行层」一律写 **执行后端 agent** /
> **执行后端会话**；只有 pi 专有的技术标识（`pi.prompt`、`PI_CODING_AGENT_DIR`、
> sidecar 契约里的 pi、配置键名、`commondev_pi_*` 工具名）才保留 pi 的字面。

**为何能用内核机制**（而不是自己造）：

| 内核概念 | 执行后端侧对应 | 落点（内核已有） |
|---|---|---|
| **给执行后端发消息** | `pi.prompt` / `steer` / `follow_up` | `InjectInputTo(source, "commondev/<id>", …)` |
| **执行后端返回** | sidecar 回调 / `tool_execution_*` / `agent_settled` | `InjectTextTo(source, "commondev/<id>", …)`（按 inputch Owner 路由） |
| **中断执行后端** | `steer`（插到当前轮之后） | `InjectInterruptTextOpts(..., Priority: L3)` |
| **音声/图像产物** | 执行后端可产出的媒体 | `OutputCapabilities()` 声明 |

⇒ **插件只做两件额外事**：① 把执行后端接成 Device；② 台账记 `session_id` 并断言未变。
调度/中断/路由/记忆策略**全是内核已经在做的事**。

### 22. 「工程对话」是一等对象

它是四元组，缺一不可：

```
工程对话 (id = br-<project>-<slug>)
├─ workdir      <插件数据目录>/program/<hash>-<name>
│                ★ hash = H(项目+分支) 是**唯一性来源**
│                  ⇒ **同一仓库的不同分支必然是不同目录**（否则两 agent 共用工作树、改动互相污染）
│                ★ name = 分支名派生，**仅做可读性**，不承担唯一性
│                  （派生后可能撞：`feat/a` 与 `feat.a` 都成 `feat_a`，所以必须带 hash）
│                独立 clone（非 worktree、非 --reference）；合入后**整目录删除**
├─ session_id   执行后端会话（★ 续投的唯一依据）
├─ runtime      local（默认，内嵌） | ssh（远端同样建目录拉执行后端 agent）
├─ host         runtime=ssh 时的目标主机（**branch_open 时给**，见 §24.1）
├─ agent_cli    执行后端（默认 pi；其他 CLI 需自备 sidecar，见 §24.2）
└─ channels     插件为该工程注册的一对通道
                ├─ inputch  commondev/<id>
                └─ outputch commondev_<id>
```

**一个仓库可并行开发**：每分支一份独立 clone，互不污染、可同时推进。
但并行有前提——必须是**互不相干的模块**；有依赖的任务请先拆边界。

> **为何不能用 worktree / `--reference`**（实测）：worktree 删后有
> `.git/worktrees/` 残留（需 `prune`）；`--reference` 有 alternates 外部依赖，
> mirror 被清即损坏。**本地 clone 对象为硬链接**（实测 inode 相同）⇒ 省磁盘、
> 无外部依赖、刪掉即彻底。代价：失去 worktree「同分支不可双挂」的保护
> ⇒ 台账层必须补**唯一性闸**（同名未结分支响亮拒绝）。

### 23. ★ 必做：为每个工程对话新建驻留子，并把通道划给它

**★ 顺序很关键**：通道是**拉起执行后端 agent 并拿到 session 之后**才注册的，所以你的动作是：

```
建工程 ──► commondev_agent_up（拉起执行后端 agent 并拿到 session_id）
            └─► 插件**此刻才**注册通道 inputch commondev/<id> + outputch commondev_<id>
                  └─► ★ 你建驻留子并把这对通道划给它
```

**严格三步，不许换顺序、不许合并**：

| 步 | 动作 | 谁做 | 不做会怎样 |
|---|---|---|---|
| ① | `commondev_branch_open` → `commondev_agent_up` 拿到 `session_id` | 主管 | 没有会话，后续投递没有落点 |
| ② | 插件注册该工程的一对通道（inputch `commondev/<id>` + outputch `commondev_<id>`） | 插件 | 通道不存在，`output_send__…` 指向空壳 |
| ③ | 建驻留子（`dev-<br-id>`）并把这对通道**划入** | 主管（**内核工具，插件代劳不了**） | 输入全涌向根 agent，等于没拆 |

**为何必须先有 session**：通道是“与某个具体执行后端会话对话的入口”，没 session 就是空壳。
顺序错了（先建子）会拿到一个指向不存在通道的 `output_send__…`。

**插件只能把通道登记出来；「划入（assign）」只有内核能做。** 所以这一步**必须由你（agent）
自己调内核工具完成**，插件无法代劳。

```
resident_agents create
    id              = dev-<br-id>
    input_chs       = commondev/<br-id>       ← 划入该 inputch（可用斜杠）
    allowed_outputs = commondev_<br-id>       ← 授权该 outputch（★ 下划线，见下）
    capacity        = 64
    task_prompt     = "你是 <project> 分支 <branch> 的项目分管，在 <workdir> 推进 <work>。
                       下任务给执行后端用 output_send__commondev_<br-id>(payload=…)，
                       执行后端的输出会回投到你的 commondev/<br-id>。
                       向上级（父 agent）汇报：有结论/卡住/需要决策时用 notify_parent；
                       不越级：结论只报父 agent，不直接对人拍板、
                       也不让执行后端越过你联系上级；执行后端的产出一律由你收口。"
```

**★ 汇报方向是单向的**：驻留子 → 父 agent（`notify_parent`）→ 人。驻留子**不**向执行后端
要结论、**不**越级直接对执行层拍板；反过来，主管也**不**绕过分管给执行后端投任务
（那正是本节要防的错）—— 投递一律走该工程的 `output_send__commondev_<br-id>`。

**★ 下任务不需要新工具**：给执行后端发消息就是子调**内核原生的输出门**
`output_send__commondev_<br-id>(payload="…")` ⇒ 打到插件的通道 handler ⇒ 插件
查台账拿到 `session_id` 后 `sidecar.prompt()`。**插件不提供 `agent_send` 之类的工具**。

**⚠️ 通道命名（很容易踩）**：

| | 能用 `/` 吗 | 为什么 |
|---|---|---|
| **inputch** | ✅ | 内核自己就用 `child/<id>` |
| **outputch** | ❌ **绝对不行** | 它会拼成 `output_send__<name>`，上游要求函数名 `^[a-zA-Z0-9_-]{1,64}$`；违反的后果不是“这个工具不可用”，而是**整条请求 400、agent 整体失语** |

⇒ 所以 inputch 叫 `commondev/<br-id>`、outputch 叫 `commondev_<br-id>`。
`<br-id>` 本身也只能用 `[A-Za-z0-9_-]`。

**为什么必须这么做（不是可选的优化）：**

1. inputch 是**最基本的输入路由单位**，路由按 `Owner` 在**进内核之前**完成。
   划给子之后，**该工程的输入只流向那个子**，根 agent 看不到它 —— 这正是
   多工程并行能做到"互不串台"的原因。
2. **不划会的代价**（如果你偷懒）：所有工程对话的输入都涌向**根 agent**，
   根成了瓶颈、上下文被 N 条工程的细节淹没，多工程并行**等于没做**。
   现场教训（`resident-subagent-design.md`）：子挂着 `inputch=[timer]` 而 timer
   的输入打在父身上，**子的轮次永远是 0** —— 登记了不等于划了。

**落地判据**（怎么知道做对了）：

- 投一条输入到 `commondev/<br-id>`，应只被 `dev-<br-id>` 处理；
- 根 agent 的轮次**不**因这条输入增加；
- `input_channels` 工具（内核自带，单工具多视图）用 `view=by_agent` 能看到划分总览。

### 24. 形态选择、自包含运行时与远端环境

**默认本地内嵌**（`agent_cli=pi` 时是**进程内 pi 实例**，换后端则为该后端的 sidecar）；
通过配置 `runtime=ssh` 切到**远程**，
远端**同样创建目录并在该目录拉起执行后端 agent**（默认 pi）。

**★ 自包含运行时**：插件**自带独立的 node 与独立的 pi**，都装在**插件工作目录**，
**pi 的插件也装到这个目录** —— 不靠宿主机的 node/pi，也不污染用户的 `~/.pi`。
**独立 node/pi 随包分发**（不首次下载、不依赖用户装）。

```
<插件数据目录>/runtime/
  node/  pi/
  agentdir/          ← 每个内层 pi 必须传 PI_CODING_AGENT_DIR 指这里
    settings.json    ← pi 自己的配置（偏好 / 默认 model / 全局包声明）
    models.json      ← pi 自己的配置（★ LLM 源写这里）
    auth.json        ← pi 自己的配置（key / OAuth，0600）
    mcp.json         ← pi 自己的配置（MCP 服务器）
    skills/ prompts/ themes/ extensions/
```

#### 24.1 远端（ssh）怎么开

两个轴**正交**，开线时一起给：

```
commondev_branch_open(project="p", branch="feat/x",
                      runtime="ssh", ssh_host="build@10.0.0.9")
```

- `runtime=ssh` 而**没给 `ssh_host`** → 直接拒绝（不会拖到拉起时才报）
- **未知 runtime**（比如 `k8s`）→ 响亮拒绝，**不静默降级为 local**
  （静默降级会让"我以为跑在远端"一路错下去）
- 远端前置条件（**先配好，不要指望交互输密码**）：
  - 从本机到目标机的**免密**（`BatchMode=yes`，卡在密码提示会超时）
  - 远端有 node（不在 PATH 里就配 `ssh_node_path`，例如 `/opt/node/bin/node`）
  - **必配两个根目录**（缺任一，远端线开不起来）：
    - `ssh_workdir_root` — 远端放**工程工作目录**的根
    - `ssh_runtime_root` — 远端放 **sidecar 脚本**的目录
- **★ 工作目录在远端准备**：本地 `clone --mirror` 出来的目录远端看不到，
  所以远端线走的是**远端建目录 → 远端共享 mirror（`<root>/.mirrors/`）→
  远端 clone → 远端切分支**。同一远端上的多条分支线共享一份 mirror。
  台账里的 `workdir` 会被改写成**远端路径**（不是本地路径）。
  远端拉取要求**远端自己能认证该仓库** —— 本插件**不把凭据推到远端**
  （凭据不进工作目录、不进 remote URL）。拉不动就响亮报错。
- 每次拉起会 `scp` 把 sidecar 推过去（幂等）——**不推就会跑上一次的旧脚本**，
  症状是"改了没生效"，极难归因。推送失败**响亮报错**，不静默用旧脚本。
- 用 `commondev_agent_state` 看 `ssh_host` 确认打到哪台机。

#### 24.2 接其他 CLI（zcode 等）——**不改插件核心**

执行后端是**声明式**的：放一个目录就多一个后端。

```
<插件数据目录>/runtime/<cli>/
    driver.json     ← 描述：脚本名 / 参数 / 环境变量 / LLM 源策略
    sidecar.mjs     ← 实现 sidecar 契约（ping / prompt / state / ...）
```

`driver.json` 关键字段：

| 字段 | 含义 |
|---|---|
| `script` | 脚本文件名（默认 `sidecar.mjs`） |
| `node` | `auto` = 用插件自带的 node |
| `args` | 参数模板，占位符 `{workdir} {sessionId} {sessionName} {dataDir} {pluginDir}` |
| `env` | 环境变量模板（同样支持占位符） |
| `llmConfig` | `pi` = 插件替它写 pi 的配置；**`none` = 它自管凭据，插件不碰** |
| `sessionDirSuffix` | 会话目录后缀 |

**纪律**：

- **未知 `agent_cli` 响亮报错，绝不回退到 pi** —— 回退会用错后端跑任务而无提示。
- `driver.json` **损坏也报错**（不猜默认），否则是"改了没生效"的经典来源。
- 内置 pi **不需要** `driver.json`（有内建描述），老部署不受影响。
- 现成例子看 `examples/custom-cli/zcode/`（**源码核实过**的 ZCode 契约）。

### 25. ★ 两种插件安装粒度（临时 vs 全局）

| 粒度 | 作用域 | 落到实处 | 用什么 |
|---|---|---|---|
| **临时** | 一个工程对话 | `<workdir>/.pi/settings.json` | `pi install -l` |
| **全局** | 插件自带 pi 的**所有**工程 | `<agentdir>/settings.json` | `pi install` + `PI_CODING_AGENT_DIR` |

**支持的源**（实测）：`npm:@scope/pkg@1.0.0`（带版本即钉住）/ `git:github.com/u/r@v1` /
`https://github.com/u/r`（当 git 源）/ `ssh://…` / 本地路径（**不复制**）。

**三条纪律**：

1. **必须传 `PI_CODING_AGENT_DIR`**，否则「全局」安装会写进**用户的** `~/.pi/agent/settings.json`
   —— 那是对宿主的越界。
2. **⚠ 安装是复制，不是引用**：`pi install ./pkg` 会把包**复制**到目标；改源目录
   **不影响已装的那份**（我曾因此得出「插件不生效」的错误结论）。要更新必须重装。
3. **安装是显式一步**，不藏在 `agent_up` 里；来源（npm 包名 / 仓库 URL / 版本钉点）要可见。

### 26. ★ LLM 源与 pi 配置：直接写 pi 的配置文件

⛔ **不要自建第二套配置存储**（本稿前版曾提 `config_commondev`，**已废弃**）。
**pi 本身就有完整的配置系统**，插件只是它的**受控写入入口**：

```
<agentdir>/settings.json   ← 偏好 / 默认 model / 「全局」包声明
<agentdir>/models.json     ← 兼容端点与模型（★ LLM 源写这里）
<agentdir>/auth.json       ← API key / OAuth（0600）
<agentdir>/mcp.json        ← MCP 服务器
```

内核把 LLM 源存在 **`config` 表**（不是 `config_<plugin>`），键前缀 `core.llm.`：

```
core.llm.provider                 = llmsproxy     ← ★ 默认源就是这一项
core.llm.sources.<n>.base_url / .api_key / .model / .adapter / .context_window …
```

**导入时字段一对一对上**：`base_url`→`baseUrl`、`api_key`→`apiKey`、
`adapter=openai`→`api="openai-completions"`、`model`→`models[].id`、
`context_window`→`contextWindow`。

**工具面**（每个工具对应一处 pi 配置）：

```
commondev_pi_source_import   把内核的源批量导入内置 pi
commondev_pi_source_add      添加源（写 models.json / auth.json）
commondev_pi_source_list     列出（默认**不返回** api_key）
commondev_pi_source_default  指定默认源（写 settings.json）
commondev_pi_config_get/set  读写其他 pi 配置
commondev_pi_mcp_add/remove/list
```

**四条实现纪律**：

1. **写的就是 pi 的配置文件**，不造中间层；格式与 pi 自己写出的保持一致
   （否则 pi 重写文件时会把自定义字段抹掉）。
2. **改前备份、改后回读校验**：json 写坏会让内层 pi 直接起不来；失败则回滚 + **响亮报错**。
3. **key 不进日志/不进返回值**。
4. **不改用户全局**：所有写入带 `PI_CODING_AGENT_DIR=<agentdir>` 或显式 scope，
   绡不碰 `~/.pi/agent/`。

**两条红线**：

1. **key 不能明文落盘到无保护位置**：`models.json`/`auth.json` 含明文 key ⇒ 目录 **0700**、
   文件 **0600**。内核那套是明文存 `config.db`（**644 全局可读**）—— **别把这套惯例带进来**。
2. **不得静默回退到宿主凭据**：导入失败/源不可用就**报错**，不要偷偷用 `~/.pi/agent/auth.json`
   跑起来（那是宿主的凭据，用了就是越界）。

### 27. 远端环境缺失：问用户，不要自己装（硬纪律）

```
ssh 不通          → 报错（配置问题，不是缺环境）
无执行后端 / 无 node → ★ 注入 L3「远端 <host> 未装 X，是否安装？」→ **你（agent）去问用户**
目录不可写        → 报错（权限问题，需人处理）
```

- **SDK 没有“向用户提问”的 API**（插件是子进程，不能直接弹 UI）。现有机制是插件把
  决策消息注入给 agent（L1–L3），**由 agent 问用户**（先例：`cli` 的 `/stop` 用 L3）。
- **不得擅自安装**：不 `curl | sh` 到别人的机器上。要先说清「装什么、装哪里、要什么权限」。
- **不冒充成功**：环境缺失时 `agent_up` **响亮失败** + 留一条待决策项，
  **不静默降级到本地跑**（那会让人以为活派到了远端）。
- **问完要能续**：安装动作本身是**可见的一步**（可查、可重跑），不是 `agent_up` 的隐式副作用。

### 28. 投递纪律（**这是最容易犯的错**）

**每个工程的投递必须落到同一个 `session_id`。**（与后端是 pi 还是 zcode 无关，规则一样） 实测反例（别重犯）：

| 错法 | 后果 |
|---|---|
| 漏传 `--session-id` | 后端 CLI（默认 `pi`）**静默开新会话** —— 你以为在催上一次，其实是把活派给了陌生人 |
| 两个进程**并发**同 `--session-id` | 会话文件被写坏：`user A`/`user B` 相邻，`assistant B` 在前、`assistant A` 落在最后 ⇒ B 那轮的模型上下文里**根本没有 A 的用户消息**，树链交错 |

⇒ 投递前先拿台账里的 `session_id`，投完**断言它没变**；一个 session **绝不并发投递**。
这两条插件侧已做成结构性保证（单一投递队列 + 断言），你做主管时**不要绕过插件**直接
去敲后端 CLI（默认就是 `pi -p`）。

**★ 另外两条实测硬经验（别再踩）：**

3. **SDK 路径必须显式指定会话 id**。用 `createAgentSession({cwd})` 而不传
   `sessionManager` 时，pi **自己生成 UUID** 作会话 id ⇒ 与台账名字不一致 ⇒
   断言**永远失败**（实测：期望 `br-cdtest-feat_alpha`，实际 `01a13959-…`）。
   正确：`SessionManager.create(cwd, sessionDir, {id})` 再传给 `createAgentSession`。
4. **工具返 `ok` ≠ 消息到了**。内核对成功回执刻意**极简**（防 output loop），
   所以 `ok` 只说 handler 成功了。必须去会话文件里看到 `user: <payload>` 才算数。
   （这条能防住“以为投了、其实没投”这类假成功。）

### 29. 三层审查的分工（别把自己变成贵的高级码农）

| 层 | 审什么 | 不审什么 |
|---|---|---|
| agent 自审 | 代码正确性、风格、实现细节 | — |
| **你（主管）复审** | **这次的改动，判据够不够**：引入了哪些新失败方式？判据覆盖了吗？变异咬得住吗？ | 逐行读 diff |
| 插件复核 | 机械执行「正向 + 变异」 | 语义判断 |

### 30. ★ 不要把「测试通过」当作「改对了」

插件只能跑**你预先写下的判据**。判据没覆盖那条 bug，插件照样打勾 ——
自动化**不会**发现假绿，**只会让假绿跑得更快、更可信**。

因此复核**必须两步**：

| 阶段 | 做什么 | 不通过的后果 |
|---|---|---|
| **正向** | 跑判据 → 期望通过 | 实现有 bug，驳回 |
| **变异** ★ | 自动改坏被测代码 → 期望判据**报红** | **判据太松，驳回** |

**没有变异阶段，就是在给「测试通过」这个信号不断充值信用，直到它骗一次大的。**
本仓实证：一条只查前缀的判据在变异测试下仍全绿，是变异把它咬出来的。

配套纪律：判据写「某标识符是否出现在函数体」时**必须先剔除注释再匹配**
（曾因注释里含 `outputParent` 之类的词导致假绿）。

### 31. 工程的目录与生命周期（一个仓库并行开发）

```
branch_open
  └─ <hash>-<name>/ （由 mirror **本地 clone** 派生）
       ★ hash = H(项目+分支)：**唯一性来源**（同仓库不同分支必不同目录）
       ★ name = 分支名派生：仅可读性
       └─ 干活 → 提交 → push → PR
            └─ 合入（或废弃）
                 └─ ★ **整个目录删除**（不保留、不归档）
```

**要点**：

- **合入后直接删目录** —— 用户明确要求；不删则下次同名分支会与残留目录混淆。
- **一个仓库可并行**：每分支一份独立 clone。
- **★ 同仓库不同分支必然不同目录**：`<hash>` 含分支（`H(项目+分支)`），hash 是**唯一性来源**；
  `name` 只做可读性。实测：`feat/alpha` vs `feat/beta` → `b3426c01-feat_alpha` vs `0566c1f5-feat_beta`。
- **⚠ 慢操作必须异步**：内核把**每次工具调用**硬限制在 **60 秒**（`toolcall.go:62`，
  硬编码无配置项）。`git clone --mirror` 同步做 ⇒ 工具报超时、**但 mirror 其实已建好**
  —— 调用方只看到失败、状态却是“做了一半”。正确做法：工具只登记 + 发起后台作业，
  用状态工具追进度（本插件：`commondev_job_status`）。
- **为什么不是 worktree / `--reference`**（实测）：worktree 删后有 `.git/worktrees/`
  残留（需 `prune`）；`--reference` 有 alternates 外部依赖，mirror 被清即损坏。
  **本地 clone 对象为硬链接** ⇒ 省磁盘、无外部依赖、删掉即彻底。
- **⚠️ 代价：唯一性闸必须补上**。worktree 天然禁止「同分支双挂」，独立 clone **没有**这条
  保护（两份 clone 都能在 `feat/x` 上各自 push，直到被拒才知道）。
  台账必须对「同名且未结（open/review）的分支」建唯一索引，重复 `branch_open`
  **响亮拒绝**，不静默复用。
- **并行有前提**：并行的必须是**互不相干的模块**。有依赖的任务（A 改接口、B 用接口）
  两棵树互不可见，问题会集中到 merge 时爆发 ⇒ **先拆边界再谈并行**。
- 与内核「子不得比父活得久、无残留」同一条纪律（见第 20 节）。

---

## 红牌（做了就是错）

1. **不划通道就当派了活** —— 登记 ≠ 划入；不划则所有输入涌向根 agent。
2. **绕过插件直接敲后端 CLI（默认 `pi -p`）** —— 漏 `--session-id` 会静默开新会话。
3. **并发投同一个 session** —— 会写坏会话（实测）。
4. **没有变异阶段的复核** —— 等于给假绿充值。
5. **凭据写进 `config.db` 或 remote URL**。
6. **改了内置插件源码只重启服务** —— 必须重编 homed。
7. **手改生成的文档**（`docs/api/*.md` 首行写着「请勿手改」）—— 改 `sdk/*.go` 注释再重新生成。
8. **为了让插件生效去改 homed 的源码** —— 先确认它是不是内置插件。

---

## ★★ 主管行为红线（2026-10-20 跑飞教训 · 踩了就视为失职）

> **现场事实**：一条工程线派给执行后端后，主管（我）没建驻留子、没划通道，
> 自己开了十几个终端轮着 `cmd_run` 查进度，还给自己定了 timer 定时轮询；
> 用户连喊几次都没被听见，最后靠**重启内核两次**才恢复。
> 结论：这不是"勤快"，是把主管当成了跑腿，把"盯着"当成了工作。

### 主管只做四件事

**拆任务 → 定判据（含变异规则）→ 建驻留子并把通道划进去 → 收工核结论。**

除此之外的一切执行动作，都是越界。**管家让园丁去修草坪，不该自己站在旁边数草。**

### 红线清单（任一条踩了就是错，不许"下不为例"）

1. **不许自己跑验证。** `go build` / `go test` / 逐轮 `git diff` / 翻工作区源码 —— 
   全部归执行后端与驻留子。主管只在**两个时刻**碰事实：
   ① 派活前的一次性摸底（必须收敛，不许边干边摸）；
   ② 收工时的核对（交付项逐条对账 + 判据的反向变异是否真咬红），**核的是结论，不是重做一遍**。
2. **不许开终端盯进度。** 进度用 `commondev_agent_state` / `commondev_branch_status`，
   一次拿到就够。`terminal_create` 只留给真正需要交互的程序（vim/ssh/top），用完即关；
   查日志、查状态优先单条非交互的 `cmd_run`。
3. **不许给自己定 timer 轮询。** timer 轮询 = 把"盯着"当工作，还持续吃上下文、
   挤占中断通道，最后把对话淹掉。进度由**驻留子的 `notify_parent`** 驱动，
   不靠父 agent 自转。唯一例外：父要连续几小时离开，留**一个**兜底 timer，回来立刻撤。
4. **不许绕过驻留子直接给执行后端投任务。** 投递只走 `output_send__commondev_<br-id>`。
5. **不许把"一条命令能查清"的事拆成十条交互式命令。** 一轮拖成粘住，
   中断消息随之堆积，最后表现为"喊不应"。

### 一句话自查

派完之后主管的正确状态是**空转等待**。只问一句：
**"这件事做完了，我除了知道结论，还改变了什么？"**
答案是"我把活干了一半" —— 那就是抢了子的事。

### 收工前过一遍

- 这条线的驻留子存在吗？通道真的划进去了吗（`resident_agents list` + `input_channels view=by_agent`）？
- 这一轮我跑过 build / test / diff 吗？跑过就是越界。
- `timer_list` 是空的吗？（除非是那唯一一个"父要离开"的兜底）
- 执行后端交回后，我是在**核结论**还是在**重做**？只该核。

## ★ 用 plugin_stop / plugin_start，不要用 plugin_disable / plugin_enable

四个工具看着像同义词，语义差别很大：

| 工具 | 作用 | 跨重启 |
|---|---|---|
| `plugin_stop` | **运行态**关闭，现在停掉 | **不影响**，重启照常加载 |
| `plugin_start` | **运行态**拉起，现在开回来 | 不影响 |
| `plugin_disable` | **持久**禁用 | **会记下来**，开机也不自启 |
| `plugin_enable` | 解除持久禁用 | 会记下来 |

**释放上下文用 `plugin_stop`。** 用 `plugin_disable` 会顺手把"开机自启"
永久关掉 —— 用户下次开机发现插件没了，得再手动开回来。

（本插件声明 `auto_start: false`，所以它本来就不自启；但 `plugin_disable`
仍会写进持久禁用清单，让 `plugin_start` 拒绝拉起，只能再用 `plugin_enable` 解。）
