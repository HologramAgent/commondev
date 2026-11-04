---
name: commondev-git
description: git 托管平台（Gitea/GitHub/GitLab 等）的操作权威：认证头与 API 前缀的四种形状、保存并复用凭据、开/查/合 PR、分支保护的机制化设置、审批的三个实测 API 坑。**插件里没有平台逻辑**——一律用 enc_shell + @{凭据名} 调 curl/gh/tea。接到 commondev 主管流程时先读 commondev-discipline。
version: 3.0.0
author: HomeAgent
---

## git 平台操作（本文是权威，不要凭记忆写 API）

### 11. 所有平台都是同一个形状

「带认证头发一个 REST 请求」。用 `enc_shell` + `curl` 即可：

```
enc_shell(command="curl -s -H \"<认证头>\" <API前缀><路径>")
```

**差异只有两个维度：API 前缀 与 认证头。** 其余是 REST 常规。

### 12. 认证头与 API 前缀（逐平台，★ 四种形状互不相同）

| 平台 | API 前缀 | 认证头 |
|---|---|---|
| **Gitea** | `<base>/api/v1` | `Authorization: token <T>` ← **不是 Bearer** |
| **GitHub** | `https://api.github.com` | `Authorization: Bearer <T>` |
| **GitLab** | `<base>/api/v4` | `PRIVATE-TOKEN: <T>` ← **不是 Authorization** |
| **GitCode** | `https://api.gitcode.com/api/v5` | `Authorization: Bearer <T>` |
| **Gitee** | `https://gitee.com/api/v5` | `access_token=<T>`（**查询参数**，不是头） |

⚠ 头名、前缀、甚至「用头还是用查询参数」都不同。**未知平台先查官方文档，不要猜形状。**

### 13. 先保存凭据，再用它

```
enc_store(alias="gitea-token", value="${GITEA_TOKEN}")     # 值优先走 ${ENV} 或 @file:
enc_shell(command="curl -s -H \"Authorization: token @{gitea-token}\" http://192.168.2.106:3000/api/v1/user")
```

⚠ **优先用 `enc_store`，不要用 `commondev_account_add`** —— 后者已不存在
（它内建了平台语义：platform/role/base_url，那正是被移出的东西）。

**先验凭据再干活**（早失败，别推了一半才发现 token 错）：

```
enc_shell(command="curl -s -o /dev/null -w '%{http_code}' -H \"Authorization: token @{gitea-token}\" http://192.168.2.106:3000/api/v1/user")
# 期望 200；401/403 = 凭据无效或权限不足
```

### 14. 常见操作（以 Gitea 为例，其他平台换前缀与头）

```bash
# 搜仓库
curl -s -H "Authorization: token @{t}" "http://host/api/v1/repos/search?q=myname&limit=20"

# 看仓库 / 分支
curl -s -H "Authorization: token @{t}" "http://host/api/v1/repos/jianf/MyProj"
curl -s -H "Authorization: token @{t}" "http://host/api/v1/repos/jianf/MyProj/branches"

# 开 PR（head=你的分支，base=目标分支）
curl -s -X POST -H "Authorization: token @{t}" -H 'Content-Type: application/json'   -d '{"title":"...","head":"feat/x","base":"main"}'   "http://host/api/v1/repos/jianf/MyProj/pulls"

# 查 PR 状态
curl -s -H "Authorization: token @{t}" "http://host/api/v1/repos/jianf/MyProj/pulls/7"

# 合 PR
curl -s -X POST -H "Authorization: token @{t}" -H 'Content-Type: application/json'   -d '{"Do":"merge"}' "http://host/api/v1/repos/jianf/MyProj/pulls/7/merge"
```

**push 时凭据不进 remote URL**：

```bash
# ✅ 一次性注入（仅本次命令）；clone 出来的 remote 保持干净
enc_shell(command="cd <repo> && git -c http.extraheader=\"Authorization: token @{t}\" push origin feat/x")
```

⚠ 若改用 `https://token@host/...` 形式，token 会**持久化进 `.git/config`** ——
删掉临时目录也删不掉凭据（`git remote -v` 就能读出来）。设计 §7.2 明令禁止。

### 15. Gitea 分支保护：让「必须复审」成为**机制**而非礼貌

**光有流程约定不算机制** —— 执行后端 agent 可以直接推主干（`git push` 到 `main` 本身不经过任何评审）。必须让**平台**拦。

```bash
curl -s -X POST -H "Authorization: token @{t}" -H 'Content-Type: application/json'   -d '{"enable_push":false,"required_approvals":1,"block_admin_merge_override":true,"dismiss_stale_approvals":true}'   "http://host/api/v1/repos/jianf/MyProj/branch_protections?name=main"
```

四个开关的意义（缺一个机制就有洞）：

| 开关 | 作用 | 缺了会怎样 |
|---|---|---|
| `enable_push: false` | 禁止直接推送 | 执行后端 agent 可直接推主干（`git push main` 不看 PR 状态），绕过评审 |
| `required_approvals: ≥1` | 至少一个审批 | 自己就能合 |
| `block_admin_merge_override: true` | admin 也不能绕过 | 用 admin token 的人绕过一切 |
| `dismiss_stale_approvals: true` | 新提交作废旧审批 | 先批准再改代码，批准白拿 |

**真机实测（Gitea 1.24.2）**：

| 动作 | 普通用户 token | admin token |
|---|---|---|
| push `main` | ❌ `pre-receive hook declined` | ⚠ 可能成功（见下）|
| push `feat/*` | ✅ | ✅ |
| 自合 PR | ❌ `Does not have enough approvals` | ⚠ 取决于开关 |
| 审批 → 合 PR | — | ✅ |

⇒ **推论：执行侧必须用普通用户 token。** 用 admin token 会**静默绕过分支保护**
（看起来正常，实际机制失效且无报错）—— 这是最容易搞错的一点。

⚠ 分支保护已存在时用 **PATCH**（POST 会 403 `already exist`）。

### 16. 审批：三个必踩的 API 坑（实测）

1. **`event` 必须是 `APPROVED`，不是 `APPROVE`。**
   传 `APPROVE` 时请求**返回 200 且不报错**，但 review 状态落成 `PENDING` ——
   于是 `required_approvals` 永远不满足，合并时报 `Does not have enough approvals`。
   表象（权限不足）与真因（枚举值少个 `D`）毫无关联，极难自查。
2. **必须带 `commit_id`** —— 不带时同样落成 `PENDING`（与 `event` 是两个独立条件）。
3. **`required_approvals` 也约束 admin** —— 别指望用 admin 跳过审批。

```bash
# ① 先取 head sha（审批必须绑定**具体提交**，不是"移动的目标"）
enc_shell(command="curl -s -H \"Authorization: token @{t}\" http://host/api/v1/repos/jianf/MyProj/pulls/7 | grep -o '\"sha\":\"[a-f0-9]*\"' | head -1")
# ② 审批（event 必须 APPROVED，且带 commit_id）
enc_shell(command="curl -s -X POST -H \"Authorization: token @{t}\" -H 'Content-Type: application/json' -d '{\"body\":\"通过\",\"event\":\"APPROVED\",\"commit_id\":\"<sha>\"}' http://host/api/v1/repos/jianf/MyProj/pulls/7/reviews")
```

**失败时保留平台原话**：`Does not have enough approvals` 是区分「机制生效」
与「配置错了」的唯一线索，别把它翻译掉或吞掉。

---
