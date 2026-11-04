---
name: commondev-hmap
description: hmap 插件的开发与部署纪律：开工前要搞清的三件事、五个必踩的坑、注册工具的正确形状、外部插件的能力边界、安装与验证、"装了却不生效"的排查、安全纪律、孤儿清理。写插件或排查插件问题时读本文；开工程/派活见 commondev-discipline。
version: 3.0.0
author: HomeAgent
---

## hmap 插件开发与部署纪律

> 面向「要改 commondev」或「要写另一个插件」的场景。**所有条目都在本机实测过**，
> 不是从文档抄的。

### 32. 先搞清三件事，别猜

```bash
hmapdev version        # 工具链版本 + SDK 模块 + 构建用 Go
hmapdev sdk current    # 当前活跃 SDK 版本（★ 动手前必看）
hmapdev sdk list       # 本机已装的所有 SDK 版本
```

- 核心仓 `/home/program/TrueAgent`（Go module `gitcode.com/JianFeeeee/HomeAgent`）
- **SDK 仓** `third_party/homeagent-sdk`（独立 git 仓、独立版本号）

### 33. 开工流程

```bash
hmapdev init <name>              # Go 脚手架（--lua / --type remotedevice 可选）
cd <name> && vim plugin.go       # 注册工具
hmapdev debug .                  # 不想装就先跑一遍
hmapdev build                    # 产出 dist/<name>_bundle.hmap
```

### 34. ★ 五个必踩的坑（按踩中概率排序）

| # | 坑 | 事实 |
|---|---|---|
| 1 | **SDK 版本** | 插件编译时**校验 SDK 能力**，用旧接口的工程直接构建失败，报错形如「当前 SDK 缺少 sdk.InjectOptions.Priority」，并给出两条出路：`hmapdev sdk install <version>` 或 `hmapdev sdk install --from <本地源码>`。**核心仓有新能力但 SDK 未跟上时用 `--from`** |
| 2 | **`hmapdev build`/`debug` 不支持 `--help`** | 不是打印帮助，而是去读当前目录的 `plg.json` ⇒ 你会看到 `error: read plg.json: no such file or directory`。**这不是故障**。要看 flag 用 `hmapdev --help` |
| 3 | **产物是 ZIP，不是二进制** | `dist/<name>_bundle.hmap` 是 ZIP（魔数 `PK`），用 `unzip` 解，内含 `plugin.bin.linux.amd64` 等多平台二进制。**生产要的是解包后的单个二进制** `/home/newqqagent/plugins/<name>/plugin.bin`（不是 zip） |
| 4 | **`plugin.json` 是安装标志** | 生产目录里没有它，插件不会被识别。`plg.json` 是**构建**标志，两者不同 |
| 5 | **`plg.json` 的 `sdk` 字段必须完整 x.y.z** | 写 `"1.4"` 会被**直接拒绝**（SDK 版本纪律：跟随内核中版本、patch 位恒为 `.0`，一条内核线只有一个 SDK 版本） |

### 35. 注册工具的正确形状

```go
s.RegisterTool(tp+"mytool", sdk.ToolDef{
    Name:        tp + "mytool",     // ← Name 必填，放结构体**首位**
    Description: "……",
    Parameters:  map[string]interface{}{"type": "object", "properties": map[string]interface{}{}},
    Handler:     h.mytool,
    NoMemory:    true,               // 声明字段一律放**末尾**
}, s)
```

**并发声明**（同一批多个 tool_call 时）：

- `ParallelSafe: true` —— 可并发。**三个条件都满足才可声明**：handler 线程安全 /
  不与同批工具争抢同一资源（SQLite 写、同一设备、同一输出通道）/ 执行顺序无关。
- `Serial: true` —— 必须串行，**优先级高于 `ParallelSafe`**。
- **默认（都不写）= 整批串行**，是保守设计不是遗漏。

⇒ commondev 的**写操作**（凭据增删、台账写、`workdir_prepare`、`pr_open/merge`、
`agent_send`）**一律 `Serial: true`**；**只读**（list/status）才 `ParallelSafe`。

### 36. ★ 外部插件的能力边界（选外部形态就必须接受）

**公开 SDK 里声明的符号，不等于外部插件拿得到。** 外部插件跑在独立进程里，能力由桥接注入：

| API | 外部插件的实际 | 该用什么 |
|---|---|---|
| `Events()` | **恒为 nil**（桥不注入 subscriber） | 桥接运行时按你的声明完成订阅 |
| `UnregisterOutputChannel` | **静默无效**（只注入 registrar，不注入 unregistrar） | 注销需重载插件 |
| `PriorityL4` | **被夹到 L3** | 用 L1–L3 |
| `SocialAPI` 写操作 | 只有 6 个**只读**方法 | 读用 `GetPerson` 等 |
| `PluginMgr()` | **只有 3 个方法**（`ReloadOne`/`ListLoadedPlugins`/`IsPluginDisabled`） | 就这 3 个 |
| `resident_agents` 等内核工具 | **调不到**（proc 桥无对应方法） | 由 agent 侧调（见第 3 节） |

> **外部形态的 L4 天花板**：commondev 作为外部插件**永远拿不到 L4**。

> **⚠️ 不要用 `ToolAPI` 去调内核内置工具（如 `resident_agents`）**：`ToolAPI` 是
> **单例**，per-agent 数据靠「最后注入者」，而创建驻留子也会注入 ⇒ 你可能打到
> **某个驻留子的表上**。这是内核已自陈的局限，别踩。

**L4 归属表**（这块最容易搞错）：

| agent | 它的"内核" | 谁能产生 L4 |
|---|---|---|
| 根 agent | 内核自身 + **内置**插件 | `raiseKernelInterrupt`、WebUI 终止按钮等 |
| 驻留子 | **父 agent** | 父（父→子消息固定 L4） |
| **外部插件（如 commondev）** | — | **永远不能**（被夹到 L3） |

⇒ 外部形态下，「立即取消整条分支」只能靠 **L3 打断主管 + 主管下令**。

### 37. 安装与验证

```bash
# ① 从 bundle 取出本机平台二进制
unzip -o dist/<name>_bundle.hmap 'plugin.bin.linux.amd64'
# ② 放进插件目录（<name> 必须与 plg.json 的 name 一致）
mkdir -p /home/newqqagent/plugins/<name>
cp plugin.json /home/newqqagent/plugins/<name>/
cp plugin.bin.linux.amd64 /home/newqqagent/plugins/<name>/plugin.bin
chmod +x /home/newqqagent/plugins/<name>/plugin.bin
# ③ 重启并确认工具**真的注册了**
systemctl restart homeagent.service
journalctl -u homeagent.service --since "-2 min" | grep "registering tool: <你的工具名>"
```

**内置 vs 独立**（目录存在 ≠ 有 `plugin.bin`）：

```bash
for p in webui cmd seq qq; do
  printf "%-8s " $p
  ls /home/newqqagent/plugins/$p/plugin.bin >/dev/null 2>&1 \
    && echo "独立（要单独构建部署）" || echo "内置（随 homed 部署）"
done
```

★ **改了内置插件的源码 ⇒ 必须重新构建并部署 homed**，重启服务不生效（`//go:embed` 编进二进制）。
★ 判断线上跑的是哪次构建，看 `commit` 字段（**不要用 `strings`**）：

```bash
K=$(sqlite3 /home/newqqagent/config.db "select value from config_webui where key='api_key';")
curl -s -H "X-API-Key: $K" http://127.0.0.1:8080/api/v1/status | grep -oE '"commit":"[^"]*"'
```

⚠ 必须带 `X-API-Key`：无认证时返回 **200 + 登录页 HTML**，只看状态码会误判。

### 38. 排查「插件装了却不生效」

按顺序查，**别跳步**：

```bash
ls /home/newqqagent/plugins/<name>/plugin.bin          # 1) 内置还是独立
journalctl -u homeagent.service --since "-5 min" | grep "registering tool:" | grep -i <name>   # 2) 注册了吗
journalctl -u homeagent.service --since "-5 min" | grep -E "loaded: <name>"                    # 3) 加载了吗
journalctl -u homeagent.service --since "-5 min" | grep -iE "<name>.*(error|panic|failed)"     # 4) 启动报错
```

### 39. ★ 安全纪律（commondev 拿的是**能 push 用户所有仓库**的权限）

> **★ 架构决定（用户拍板 2026-10-14）：敏感信息拦截下沉为内核安全模块。**
>
> 理由：**LLM 调用本身不可信**，而插件在**信任边界之外** ——
> 插件注册的 stage 拦截是“自愿的”，不是强制的，也算不上边界。
>
> ★ **它不是单纯拦截，而是一个闭环**（用户纠正：“只是简单的拦截的话，
> 绝大部分插件会失去原本的功能”）：
>
> ```
> ① 拦截      LLM 出向前扫到明文 → 换成 @{别名}
> ② 加密暫存  明文 AES-GCM **落盘**存内核（不落明文）
> ③ 别名      ★ **哈希键**：sec_ + HMAC-SHA256(masterKey, plain)[:8]
>               ⇒ **同一明文自动得同一个别名**（同变量合并，天然性质）
>               ⇒ 必须 HMAC 而非裸哈希（裸哈希可字典枚举）；**不加 salt**（否则失去合并）
> ④ 回填      插件调用前把 @{别名} 换回明文 ← 插件无感知
> ```
>
> ★ **变量不自动淘汰**（跨轮/跨子 agent 随时要用）；agent 可**主动**调用
> 安全模块提供的**提交/删除**工具。泄漏风险靠主动删控制，不靠超时。
>
> ★ **因为内核零 IO（一切外部交互都从插件走），只需两根咽喉**：
> · 入向 toolcall：`StageHost.ExecuteTool(name, args)`（`core/stages.go:140`）
> · 入向 outputch：设备 `Execute`（`core/output.go:76`）
> · 出向 LLM：provider 层的 `Chat(req)`（已核：构造 `CompletionRequest` 有 **7 处**，
>   task 层拦不全）
>
> ★ **变量系统是全局唯一的**（用户拍板）：主 agent / 子 agent / 驻留子 agent
> **同一套**别名表。已核依据：驻留子共享父的 `ProviderManager`（`resident.go:185`），
> 而它**全进程只建一份**（`bootstrap.go:249`）⇒ 安全模块天然是进程级单例，
> **不需要跨 agent 同步**。
> ⚠ **变量共享 ≠ 授权共享**：能不能调某工具/通道仍看该 agent 自己的 `AllowedOutputs`
> （与 `KernelSource` 一道是 per-agent 的）。两者**正交**。
>
> ★ **“插件无感知”是硬要求**：替换发生在路由到插件的**之前**；
> 插件代码一行不改、不可分辨、无法绕过 ⇒ 存量插件自动获得该能力，
> 且无需给 SDK 加参数（不破接口兼容）。
>
> **对本插件的直接影响（大幅简化）**：插件可以**把密钥明文直接投给 LLM**，
> 由内核安全模块全权负责；`enc_*` 工具族与加密 shell 应**删减**，
> 不再自带加密库/别名表（前稿那套作废）。
>
> 详见 `docs/zh/kernel-secret-firewall-design.md`（含硬约束 S1–S9）。
>
> **两条不可让步的约束**：
> · **只能替代引用，永远不可取回明文**（用户明确）—— 读接口与用接口是两件事；
> · **模式库宁漏勿误伤** —— 误报会搞坏正常任务。

**凭据：按名字引用 + 变量填充**（用户明确要求的能力）

```
enc_store(alias="gitea-token", value="${GITEA_TOKEN}")   ← ★ 引用，不是明文
enc_list                                                  ← 只列名字，**绝不返回值**
```

之后一切操作只带**别名**：`@{gitea-token}` —— 参数里永远不出现 secret。
详见 **第二部分 §13**（那里是权威）。

**为何要 `${ENV_VAR}` 变量填充**：（①）明文不落盘；（②）dev/prod 换环境变量即可；
（③）轮换凭据不需改插件数据。

#### ★★ 判据与复核：**插件 GREEN ≠ 改对了**

**这是本流水线最重要的一条纪律。** 插件只能跑你**预先写下**的判据；
判据没覆盖那条 bug，插件照样打勾。自动化**不会**发现假绿，
只会让假绿跑得更快、更可信。

**真实数据**（我用最小例子复刻过）：

```go
func Max(a,b int) int { if a>b { return a }; return b }
```
- 判据只测 `Max(5,3) == 5`（漏了 `b>a`）
- 变异 `return b` → `return a`（**真 bug**）⇒ **判据仍全绿**
- 补一条 `Max(3,5) == 5` ⇒ 同一变异**立刻报红**

⇒ 所以**复核必须两步**，你定义判据时**必须同时给变异规则**：

```
commondev_test_define(
  project="MyProj", name="clamp 上下界", kind="unit",
  script="go test -count=1 ./...",
  expect="上下界都被限制",
  mutations=[{ id="M1", file="calc.go",
               old="	if v > hi {\n		return hi\n	}\n", new="",
               note="删掉上界分支（真 bug）——松判据看不见它" }]
)
commondev_review_submit(branch_id="br-...")
commondev_review_result()          ← 取结论
```

**结论三档，注意第二档不是"通过"**：

| verdict | 含义 | 你该做什么 |
|---|---|---|
| `pass` | 正向全过 **且**全部变异被咬住 | 可以进下一步 |
| **`pass_with_gaps`** | 正向过，但**有变异没被咬住**或**判据没变异规则** | **不能当通过** —— 去补判据/变异，再复核 |
| `fail` | 有判据没过 | 打回执行后端 agent |

★ **`pass_with_gaps` 就是"你的判据有盲区"**。把它当绿色勾，
就是在给自己攒一次大的意外。

**定义变异规则的三个要点**：

1. **`old` 必须在文件里恰好出现一次** ——
   出现 0 次 ⇒ 变异无效（却会被误报成"判据太松"）；
   出现多次 ⇒ 可能改错地方（却会被误报成"咬住了"）。
   多出现时给出更长的上下文。
2. **变异要"真坏"** —— 改的是逻辑，不是格式。改不坏就验不出判据的松紧。
3. **`script` 里带 `-count=1`** —— Go 默认缓存测试结果；
   命中缓存时测试**根本没跑**，"变异后报红"就无从谈起。

#### 双凭据与分支保护：**已移到第二部分**（技能纪律 §15–§16）

原来这里用 `commondev_account_add(role=...)` + `commondev_protect_branch` 两个
插件工具实现。**这些工具已移除**（平台知识不再进插件）。

现在这样做（命令见 **第二部分 §15**）：

1. **在平台上建两个用户**：一个普通用户（执行侧，只能 push `feat/*`）、
   一个有管理权限的（主管侧，能审批与合并）。各自签发 token。
2. **两个 token 都存进凭据库**：`enc_store(alias="gitea-exec", ...)` /
   `enc_store(alias="gitea-sup", ...)`。
3. **给主干上锁**（第二部分 §15 的 curl 命令，四个开关都要）。
4. 审批与合并用主管侧别名（§16）。

★ **执行侧必须用普通用户 token** —— 用 admin token 会**直接绕过分支保护**，
机制失效且无任何报错。这是最容易搞错的一点。

★ **拦你的是平台，不是插件** —— 所以改插件代码也绕不过去。
这才是「机制」与「流程约定」的区别。

**两条不可逆的红线**：

1. **变量解析不到值 ⇒ 响亮报错**，绝不静默当空串 —— 后者会退化成匿名访问，
   表现成 **401** 而非“凭据没配”，极难排查。
2. **secret 绝不进返回值/日志**。

**平台形状差异**（不能一个认证头走天下）：

| 平台 | API 前缀 | 认证头 |
|---|---|---|
| Gitea | `<base>/api/v1` | `Authorization: token <T>` ← **不是 Bearer** |
| GitHub | `https://api.github.com` | `Authorization: Bearer <T>` |
| GitCode | `https://api.gitcode.com/api/v5` | `Authorization: Bearer <T>` |

未知平台**必须报错**（不猜形状）。

**其他：**

1. **凭据不入 `config.db`**：现有惯例是明文存 `config_<plugin>` 表，而 `config.db` 是
   **644 全局可读**。commondev 的凭据性质完全不同 ⇒ 放独立文件
   `<DataDir>/credentials.json`，**0600，启动时校验权限，不对就拒绝启动**
   （本仓纪律：响亮失败，不静默降级）。
2. **优先 SSH key 而非 PAT**；每平台独立、最小授权，**不要一个 token 通吃**。
3. **token 不进 remote URL**：拼接仅用于**单次命令**（`pushURLWithToken`），
   clone 出来的 remote 不带凭据 —— 否则删掉临时目录也删不掉凭据。
4. **不要硬编码 token**：走 `Settings()` 或宿主注入。
5. **拉起执行后端（默认 pi）时隔离端口**：内层 pi 会尝试监听 A2A/ACP 端口并打 `EADDRINUSE`，
   显式设 `PI_CODING_AGENT_SESSION_DIR` / `PI_A2A_PORT` / `PI_ACP_PORT`。

### 40. ★ 孤儿清理（插件有自己的硬约束）

内核对自己的驻留子有硬约束：**子不得比父活得久 / 无孤儿 goroutine / 无残留 temp**
（`resident-subagent-design.md` §10，写成判据）。

**插件对自己拉起的进程必须同样硬**：`Stop()` 必须是**幂等的全清** ——
执行后端进程（默认 pi）、sidecar、ssh 会话、workdir（含凭据残留检查）一个不留。
写成判据：`TestStopKillsAllRuntimes`（变异：删掉某个清理 → 必须报红）。

### 41. 参考资料（**别凭记忆写 API**）

| 内容 | 路径 |
|---|---|
| SDK 源码 | `third_party/homeagent-sdk/sdk/` |
| **可运行的示例插件** | `third_party/homeagent-sdk/example/`（a2a / acp / qq / plugindev…） |
| 能力边界 | `third_party/homeagent-sdk/docs/guide/capability-boundary.md` |
| 并发声明 | `third_party/homeagent-sdk/docs/guide/parallel-tool-declaration.md` |
| 部署手册 | `docs/zh/deploy-runbook.md` |
| sidecar 参考实现 | `internal/plugins/clawhubadapter/sidecar.go` |

★ **写插件前先翻 `example/`**：那里的代码是**编译通过**的，比文档可靠。
**文档与示例冲突时以示例为准。** 注意 `example/` 是**外部插件**，
能被它们编译通过的写法，外部就一定可用。

---
