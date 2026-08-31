# g2c 命令参考手册

> 适用版本:`g2c 1.0.6`(Node ≥ 20)
> 文档定位:命令参考。命令快查与英文示例见根目录 `README.md`,未完成项见 `Gerrit2Claw-cli_TODO.md`。
> 任何时候不确定参数,直接执行 `g2c <command> --help`(已实现,commander 自带)。
>
> **AI / 自动化读取入口**:本文件随 npm 包(`files: ["dist", "docs"]`)一起发布,运行时通过 `g2c help` 命令输出到 stdout,无需读源码或文档仓库:
>
> ```bash
> g2c help                    # 完整手册
> g2c help --sections         # 列出全部章节
> g2c help 3                  # 按编号取一节
> g2c help change             # 按关键词取一节
> g2c help --path             # 本机手册绝对路径
> g2c help --json             # 套上 JSON 信封,方便 Agent 解析
> ```

---

## 0. 约定

### 0.1 字段命名

| 文档里写作 | JSON / Gerrit 字段 | 含义 |
|---|---|---|
| **URL-number** | `URL-number` | Gerrit 分配的单调递增整数 ID,URL 末尾那个数字(如 `+/5`) |
| **Change-Id** | `Git-Change-Id` | commit message 里 `Change-Id: I...` 的 SHA 串,跟着提交走 |
| **REST-id** | `Gerrit-REST-id` | `项目~编号` 复合 ID,供 REST API 使用 |
| **revision** | `currentRevision` | 当前 revision SHA；change 已合并时也是 Gerrit UI 的 `Merged As` |
| **parent commit ID** | `parentCommitIds[]` | 当前 revision 的父提交 SHA；merge commit 可能有多个 parent |
| **merged commit ID** | `mergedCommitId` | 已合并 change 的当前提交 SHA，即 Gerrit UI 的 `Merged As` |
| **clone URL** | `cloneUrls.<scheme>` | Gerrit 为当前 revision 公布的仓库 URL，不由 CLI 猜测 |

`change list` / `change info` / `me --status` 默认只输出友好字段(URL-number / 状态 / 标题 / owner / reviewers / repo / branch / updated / CR);想看 `labels` / `submitRequirements` 等原始 normalized 字段,加 `--full`。

### 0.2 全局选项

```text
-V, --version                输出版本号
--json                       输出机器可读 JSON
--format <format>            table | json | csv(默认 table)
--output <path>              把格式化结果写文件(--format 配套)
--url <url>                  覆盖 GERRIT_HTTP_URL
--username <name>            覆盖 GERRIT_USERNAME
--password <password>        覆盖 GERRIT_PASSWORD(慎用,会进 shell 历史)
--repo <path>                覆盖本地 Git 仓库
--color / --no-color         强制开关颜色
--verbose                    启用诊断日志到 stderr(仅声明,未消费,见 TODO 5.1)
-h, --help                   显示帮助
```

### 0.3 写操作保护协议

`g2c` 凡是会写 Gerrit 或 push 到 `refs/for/*` 的命令,统一遵守:

- `batch score` / `batch submit`:默认 dry-run,必须传 `--yes` 才真的写。
- `repo conflict-continue`:必须传 `--yes` 才 push。
- `repo cherry-pick`:直接走 Gerrit REST,无需 `--yes`,但会在 JSON 中标注 `mode: "gerrit-rest"`。
- 单条写命令(`change abandon` / `change restore` / `change revert` / `change move` / `change wip` / `change ready` / `change topic` / `change hashtags` / `change reviewer` / `change attention` / `review message` / `review comment` / `review score` / `review submit`):按调用即生效,不需 `--yes`。

`--dry-run` 与 `--yes` 互斥;不传 `--yes` 等价于 dry-run。

### 0.4 配置来源优先级

```
命令行全局选项  >  环境变量  >  ~/.gerrit2claw/config.json  >  .env / .g2crc(只读兼容)
```

`setup`、`auth login` 与 `config set` 只写用户级 `~/.gerrit2claw/config.json`,不会写项目仓库(决策见 TODO 5.2)。默认本地仓库和默认分支配置已废弃;本地仓库操作请显式传 `--repo`。

---

## 1. auth / config / user

### 1.0 `g2c update [--check]`

从 GitHub Release API 查询最新稳定版，并只使用发布资产 `g2c.tgz` 更新。默认在发现新版本时执行全局 npm 安装，并自动沿用当前 `g2c` 所在的 npm prefix，避免系统中存在多个全局 prefix 时更新到错误位置；`--check` 只返回版本比较结果，不修改本机安装。

```bash
g2c update
g2c update --check --json
```

### 1.1 `g2c setup`

首次使用的一行配置入口。必须验证 Gerrit 连通性,验证失败不会写入配置。先在 Gerrit Web UI 的 `Settings -> HTTP Credentials` 生成 HTTP Password。未提供 `--ssh-url` 时,根据 HTTP URL 和用户名自动推导 `ssh://<username>@<http-host>:29418`。

| 选项 | 说明 |
|---|---|
| `--url <url>` | Gerrit HTTP URL |
| `--ssh-url <url>` | Gerrit SSH URL,可省略 |
| `--username <name>` | Gerrit 用户名 |
| `--password <password>` | Gerrit HTTP 密码 |

```bash
g2c setup --url <gerrit-http-url> --username <gerrit-username> --password '<Gerrit HTTP Password>'
```

### 1.2 `g2c auth login`

把 Gerrit 凭据写入 `~/.gerrit2claw/config.json`(目录 0700,文件 0600)。必须 ping 一次 `/a/accounts/self` 验证连通性,不能跳过。

| 选项 | 说明 |
|---|---|
| `--url <url>` | Gerrit HTTP URL,必填 |
| `--ssh-url <url>` | Gerrit SSH URL |
| `--username <name>` | Gerrit 用户名 |
| `--password <password>` | Gerrit HTTP 密码(也可走环境变量) |

```bash
g2c auth login \
  --url <gerrit-http-url> \
  --username <gerrit-username> \
  --password '<Gerrit HTTP Password>'
```

### 1.3 `g2c auth status`

验证当前凭据并打印当前账号信息。退出码 0 表示通。

```bash
g2c auth status --json
```

### 1.4 `g2c config get [key]`

打印生效后的配置。不带 `key` 输出全集,带 `key` 取单值。

```bash
g2c config get
g2c config get GERRIT_HTTP_URL --json
```

### 1.5 `g2c config set <key> <value>`

写一条配置到 `~/.gerrit2claw/config.json`。`key` 大小写不敏感(内部统一存大写)。

```bash
g2c config set GERRIT_HTTP_URL <gerrit-http-url>
```

### 1.6 `g2c user search <query>` / `g2c user lookup <query>`

按用户名 / 全名 / display name / 邮箱查账号,带 `--limit`(默认 20)。`search` 与 `lookup` 在 Gerrit 接口上等价,保留两套命令名只为兼容习惯。

```bash
g2c user search <account-query> --limit 5 --json
g2c user lookup <account-query> --json
```

### 1.7 `g2c user perms [--change <change>]`

显示当前用户在指定 change(可省)上的关键权限,辅助判断"我能不能 submit / abandon"。

```bash
g2c user perms --change 7 --json
```

---

## 2. 查询视图

### 2.1 `g2c me`

显示当前用户最近 7 天名下 change 数量(Open / Merged / Abandoned)。统计直接查询 Gerrit change 索引(`owner:self after:<7天前>`),不再遍历仓库/分支,因此不会被父路径或不可访问项目打断。

| 选项 | 说明 |
|---|---|
| `--repo-limit <n>` | 兼容旧参数;当前 `me` 不再扫描仓库 |
| `--branch-limit <n>` | 兼容旧参数;当前 `me` 不再扫描分支 |
| `--count-limit <n>` | 兼容旧参数;当前 `me` 直接统计最近 change |
| `--status <status>` | `open` / `merged` / `abandoned` 之一,改为列出最近 7 天对应 change |
| `--limit <n>` | `--status` 列表的最大行数(默认 100) |
| `--full` | 输出完整 normalized 结构,而不是友好字段 |

```bash
g2c me --json                       # 最近 7 天汇总计数
g2c me --status open --json         # 最近 7 天我名下 open change
g2c me --status merged --limit 20   # 最近 7 天我名下 merged,最多 20 条
```

### 2.2 `g2c list-repo [options]`（兼容命令）

列出当前用户可见的仓库。新脚本优先使用 `g2c repo list`，本命令继续兼容。

| 选项 | 说明 |
|---|---|
| `--limit <n>` | 上限(默认 100) |
| `--skip <n>` | 跳过前 N 条 |
| `--prefix <prefix>` | 仅前缀匹配(走 `?prefix=`) |
| `--match <text>` | 包含匹配(走 `?match=`) |
| `--regex <regex>` | 正则匹配(走 `?r=`) |
| `--description` | 顺带拉 description |
| `--branches <branch...>` | 仅保留含这些分支的仓库,顺带返回分支 HEAD |

`--prefix` / `--match` / `--regex` 三选一,后端走不同接口。

```bash
g2c list-repo --json
g2c list-repo --prefix g2c- --description --branches master --json
```

### 2.3 `g2c list-branch [options] [repo]`（兼容命令）

列某仓库的分支。新脚本优先使用 `g2c repo branches <project>`。

| 选项 | 说明 |
|---|---|
| `--project <name>` | 仓库名(等价于位置参数) |
| `--limit <n>` | 上限(默认 100) |

```bash
g2c list-branch gerrit-test-repo --json
g2c list-branch --project g2c-e2e-alpha --limit 20 --json
```

---

## 3. change

`change` 是最大的一组命令,分三类:读、状态变更、协作元数据。

### 3.1 读

#### 3.1.1 `g2c change list [options]`

跑 Gerrit 查询字符串。`<change>` 占位在很多子命令里是 URL-number 或 Change-Id 都可以。

| 选项 | 说明 |
|---|---|
| `--query <query>` | Gerrit query 字符串,例如 `status:open owner:self` |
| `--limit <n>` | 最大返回条数 |
| `--skip <n>` | 跳过前 N 条(分页起点) |
| `--page-size <n>` | `--all` 时的每页大小(默认 100) |
| `--all` | 自动翻页,直到 Gerrit 报空 |
| `--full` | 输出完整 normalized,而不是友好字段 |

```bash
g2c change list --query 'status:open owner:self' --limit 5 --json
g2c change list --query 'status:open owner:self' --all --page-size 100 --json
g2c change list --query 'status:open project:gerrit-test-repo' --json
g2c change list --query 'owner:self today' --json     # today 按本机本地时区展开
```

`--all` 是 Agent 导出 1000+ 行的关键;配合 `--limit` 做硬上限保护。
查询里的 `today` / `after:YYYY-MM-DD` / `before:YYYY-MM-DD` 会先按当前系统时区换算成 Gerrit 时间,不需要手动计算 UTC 偏移。

#### 3.1.2 `g2c change info <change> [--full]`

单条 change 的友好元数据;`--full` 出完整 normalized。友好输出包含 `branch`、`currentRevision`、`mergedCommitId`、`parentCommitIds`、`cloneUrls` 和 `fetchRef`。`mergedAs`、`parents` 作为兼容别名保留；`mergedCommitId` 仅在状态为 `MERGED` 时有值。

```bash
g2c change info 7 --json
g2c change info I18ae4ecff7c6c867958e8d749b9fd2817aece3e2 --full --json
```

#### 3.1.3 `g2c change parent <change> [--revision <rev>]`

获取指定 revision 的所有父提交。规范字段为 `parentCommitIds[]`；兼容字段 `parents[]` 额外包含 subject。普通提交通常有一个 parent，merge commit 可以有多个。

```bash
g2c change parent 12593 --json
g2c change parent 12593 --revision current --json
```

#### 3.1.4 `g2c change merged-as <change>`

按 Gerrit Web UI 的规则返回 `mergedCommitId`：当 change 状态为 `MERGED` 时取 `current_revision`，未合并时返回 `null`。同时保留 `mergedAs` 兼容别名。

```bash
g2c change merged-as 12593 --json
```

#### 3.1.5 `g2c change clone-url <change> [--scheme <scheme>]`

返回 Gerrit 在 `RevisionInfo.fetch` 中实际公布的 clone/download URL、fetch ref 和下载命令。生成的 `cloneCommand` 必定包含 Change 目标分支，例如 `git clone -b 'master' '<url>'`。使用 `--scheme ssh` 或 `--scheme http` 选定一种协议；不传时返回全部协议的 `cloneCommands`。

```bash
g2c change clone-url 12593 --json
g2c change clone-url 12593 --scheme ssh --json
```

#### 3.1.6 `g2c change revisions <change>`

列出该 change 的所有 patch set(每个 revision 的 SHA、commit 作者、uploader、时间)。

#### 3.1.7 `g2c change files [--revision <rev>] <change>`

列出本 revision 变更的文件(含 status、insertions / deletions、size delta)。

```bash
g2c change files 2 --json
g2c change files 2 --revision 7b685d30627f6e51e52c47a08ceb477855269d84 --json
```

#### 3.1.8 `g2c change export <change> [options]`

不 clone Git，通过 REST 导出 metadata、完整 patch、所有真实变更文件和逐文件 patch。适合 `frameworks/base` 等超大仓库。Gerrit 虚拟文件（如 `/COMMIT_MSG`）记录到 `skippedFiles`，不会导致导出失败。

```bash
g2c change export 12603 --output-dir ./change-12603 --json
g2c change export 12603 --no-files --output-dir ./patch-only --json
g2c change export 12603 --no-file-patches --concurrency 8 --output-dir ./files-only --json
```

#### 3.1.9 `g2c change file-content [options] <change>`

读或下载一个文件的当前内容。

| 选项 | 说明 |
|---|---|
| `--file <path>` | 文件路径 |
| `--revision <rev>` | revision(默认 `current`) |
| `--decode` | 返回解码后的明文 |
| `--output <path>` | 写到本地,默认用源文件 basename |

```bash
g2c change file-content 7 --file alpha-master-change.txt --decode --json
g2c change file-content 7 --file alpha-master-change.txt \
  --output /tmp/alpha-master-change.txt --json
```

#### 3.1.10 `g2c change diff [options] <change>`

清洗后的 diff。**当前未实现 `--base / --parent / --whitespace`**(见 TODO 5.4),只能看相对 parent 的 diff。

| 选项 | 说明 |
|---|---|
| `--file <path>` | 单文件 diff |
| `--revision <rev>` | revision(默认 `current`) |
| `--context <n>` | 上下文行数 |
| `--max-bytes <n>` | 截断字节数 |

```bash
g2c change diff 2 --file cli-open-change.txt --json
```

#### 3.1.11 `g2c change patch [options] <change>`

Gerrit 格式化 patch(默认 base64,`--decode` 拿明文;`--zip` 直接拿 zip 归档)。

| 选项 | 说明 |
|---|---|
| `--revision <rev>` | revision(默认 `current`) |
| `--file <path>` | 仅一个文件的 patch |
| `--decode` | 返回明文 base64-decoded |
| `--zip` | 走 zip 归档接口 |
| `--output <path>` | 默认按 Gerrit Web 命名:`<revision7>.diff.base64` 或 `<revision7>.diff.zip` |

```bash
g2c change patch 7 --decode --json
g2c change patch 7 --zip --output /tmp/2f6cb18.diff.zip --json
g2c change patch 7 --output /tmp/2f6cb18.diff.base64 --json
```

#### 3.1.12 `g2c change comments <change>`

返回该 change 已发布的 review 评审(包含 inline comments,按 file / line 组织)。

#### 3.1.13 `g2c change messages <change>`

返回 change message(评审历史、状态变更说明、机器人消息等)。

#### 3.1.14 `g2c change submitted-together [options] <change>`

Gerrit 拓扑排序会一起被 submit 的 changes。`--option` 透传 Gerrit 参数(如 `non_conflicting`)。

```bash
g2c change submitted-together 7 --json
g2c change submitted-together 7 --option non_conflicting --json
```

#### 3.1.15 `g2c change related [--revision <rev>] <change>`

返回相关 changes(同 topic、cherry-pick 关系等)。

### 3.2 状态变更

下列命令直接生效,不带 `--yes` 保护;但都是单目标,Agent 触发时一般已经过查询。

| 命令 | 说明 | 关键参数 |
|---|---|---|
| `change abandon <change>` | 放弃 open change | `--msg <message>` |
| `change restore <change>` | 恢复 abandoned change | `--msg <message>` |
| `change rebase <change>` | Gerrit 端 rebase | `--base <revision>` |
| `change revert <change>` | 为 merged change 创建 revert change | `--msg <message>` |
| `change move <change>` | 改目标分支 | `--branch <branch>` `--msg <message>` |
| `change wip <change>` | 标 WIP | `--msg <message>` |
| `change ready <change>` | 标 ready for review | `--msg <message>` |

```bash
g2c change abandon 7 --msg 'not needed' --json
g2c change rebase 7 --json
g2c change move 7 --branch release/1.0 --msg 'move to release branch' --json
g2c change wip 7 --msg 'work in progress' --json
g2c change ready 7 --msg 'ready for review' --json
```

> `change move --branch` 与全局 `--branch` 同名,本命令的 `--branch` 优先;全局 `--branch` 在这里被忽略。

### 3.3 协作元数据

#### 3.3.1 `change topic`

```bash
g2c change topic get 7 --json
g2c change topic set 7 g2c-smoke --json
g2c change topic delete 7 --json
```

#### 3.3.2 `change hashtags`

`get` 返回数组;`set` 只能 add / remove,不接受"整组替换"语义(避免误删别人的 tag)。

```bash
g2c change hashtags get 7 --json
g2c change hashtags set 7 --add g2c-smoke --json
g2c change hashtags set 7 --remove g2c-smoke --json
```

#### 3.3.3 `change reviewer`

| 子命令 | 说明 |
|---|---|
| `reviewer list <change>` | 列出 reviewer / CC |
| `reviewer add <change>` | 加 reviewer 或 CC,`--state REVIEWER\|CC`,`--confirmed` 忽略账号找不到的告警 |
| `reviewer delete <change>` | 移 reviewer,`--account <id>` |
| `reviewer suggest <change>` | 推荐 reviewer,`--query` 是 Gerrit suggest 字符串,`--exclude-groups` 排除组 |

```bash
g2c change reviewer list 2 --json
g2c change reviewer add 7 --reviewer <account> --state REVIEWER --json
g2c change reviewer suggest 2 --query <account-query> --limit 5 --json
```

#### 3.3.4 `change attention`

| 子命令 | 说明 |
|---|---|
| `attention show <change>` | 当前 attention set |
| `attention add <change>` | 加一个用户到 attention set,`--reason` 必填 |
| `attention remove <change>` | 移除,`--reason` 必填 |

```bash
g2c change attention show 2 --json
g2c change attention add 7 --account 1000002 --reason 'please review' --json
```

---

## 4. review

向 Gerrit 提交评审动作。

> `review message` / `review comment` 对 **任意状态**(NEW / MERGED / ABANDONED)的 change 都可发,因为 Gerrit 的 `/review` 端点本身允许在已合并/已废弃的 change 上追加消息和 inline 评论(常用于事后说明、reply)。
> `review score` / `review submit` 仅对 open(NEW)change 有效 —— Gerrit 在非 open change 上投票或提交会被服务端拒绝,g2c 会提前抛 `CHANGE_NOT_OPEN`。

### 4.1 `g2c review message [options] <change>`

发一条评审消息(无 label,无 inline)。任意状态 change 都可发。

```bash
g2c review message 2 --msg 'looks good overall' --json
```

### 4.2 `g2c review comment [options] <change>`

发单条 inline 评审。任意状态 change 都可发。**`--range` 未实现**(TODO 5.3),当前只支持 `--line`。

| 选项 | 说明 |
|---|---|
| `--file <path>` | 文件路径 |
| `--line <n>` | 行号 |
| `--msg <message>` | 评审文本 |
| `--revision <rev>` | revision(默认 `current`) |
| `--side <side>` | `REVISION` 或 `PARENT`(默认 `REVISION`) |

```bash
g2c review comment 2 --file cli-open-change.txt --line 1 \
  --msg 'review comment' --json
```

### 4.3 `g2c review score [options] <change>`

给一个或多个 label 打分。`--label` 可重复,格式 `Code-Review=+2`、`Verified=+1`。

```bash
g2c review score 3 --label Code-Review=+2 --json
g2c review score 3 --label Code-Review=+2 --label Verified=+1 --json
```

### 4.4 `g2c review submit <change>`

提交 change。前提是该 change 满足 submit requirements(可先 `change info --full` 看 `submitRequirements`)。

```bash
g2c review submit 3 --json
```

---

## 5. batch(批量)

所有 `batch` 子命令默认 dry-run,只返回计划;**`--yes` 才真的写 Gerrit**。

### 5.1 `g2c batch score [options]`

| 选项 | 说明 |
|---|---|
| `--query <query>` | Gerrit query 字符串 |
| `--label <name=value>` | 评分,可重复 |
| `--limit <n>` | 最大 change 数(默认 20) |
| `--dry-run` | 强制 dry-run(默认就是 dry-run) |
| `--yes` | 真的写 Gerrit |

```bash
g2c batch score --query 'status:open owner:self' --label Code-Review=+1 \
  --limit 5 --json
g2c batch score --query 'status:open owner:self' --label Code-Review=+1 \
  --limit 5 --yes --json
```

JSON 包含 `planned` / `executed` / `skipped` / `failed`,以及每个 change 的独立结果。

### 5.2 `g2c batch submit [options]`

| 选项 | 说明 |
|---|---|
| `--query <query>` | Gerrit query 字符串 |
| `--limit <n>` | 最大 change 数(默认 20) |
| `--dry-run` | 强制 dry-run(默认) |
| `--yes` | 真的 submit |
| `--continue-on-error` | 部分失败时继续(默认 true) |

```bash
g2c batch submit --query 'status:open owner:self' --limit 5 --json
g2c batch submit --query 'status:open owner:self' --limit 5 --yes --json
```

dry-run 时还会顺手返回每个 change 的 `canSubmit` 判断,方便预检。

---

## 6. repo（Gerrit 仓库查询与兼容的本地操作）

`repo list/info/branches/branch` 只查询 Gerrit，不修改本地目录；返回的 `commands` 供用户、AI Agent 或脚本交给原生 `git` 执行。

### 6.1 `g2c repo list [options]`

列出当前账号可见仓库，选项与兼容命令 `list-repo` 相同：`--limit`、`--skip`、`--prefix`、`--match`、`--regex`、`--description`、`--branches`。

```bash
g2c repo list --match system_ext --json
```

### 6.2 `g2c repo info <project>`

查询仓库元数据，并根据已验证的 Gerrit 配置生成 SSH/HTTP clone URL 和原生 `git clone` 命令模板。输出以 `cloneUrlSource: configured-server` 明确标记其来源；Change 的 URL 则继续由 `change clone-url` 读取 Gerrit revision fetch 数据。

```bash
g2c repo info platform/vendor/xos/apps/system_ext --json
```

### 6.3 `g2c repo branches <project> [--limit <n>]`

列出仓库的分支名称、完整 ref 和 HEAD revision。

```bash
g2c repo branches platform/vendor/xos/apps/system_ext --limit 200 --json
```

### 6.4 `g2c repo branch <project> <branch>`

精确定位一个分支，返回 HEAD revision、clone URL，以及 `clone/fetch/checkout` 原生 Git 命令；命令本身不执行 Git。

```bash
g2c repo branch platform/vendor/xos/apps/system_ext MTK_Phoenix_A16_mssi_DEV --json
```

### 6.5 `g2c repo clone <change> [options]`

查询 Change 的 project、branch 和 clone URL，生成原生 `git clone` 命令。该命令自身不执行 Git、不创建目录；JSON 中明确返回 `executed: false`。

```bash
g2c repo clone 12603 --output-dir ./aiagent --json
g2c repo clone 12603 --depth 1 --filter blob:none --output-dir ./aiagent --json
```

### 6.6 `g2c repo download <change> [options]`

查询 Change 的 clone URL、`fetchRef` 和 `currentRevision`，生成 `init / remote add / fetch / switch` 原生 Git 命令。该命令自身不下载、不创建目录；JSON 中明确返回 `executed: false`。

```bash
g2c repo download 12603 --output-dir ./aiagent-change-12603 --json
```

对于超大仓库，如果只需评审文件，优先 `change export`。需要 Git 工作树时，先用 `repo download` 或 `repo clone` 生成命令，再由用户、AI Agent 或脚本明确调用原生 Git。

### 6.7 `g2c repo status [--repo <path>]`

`git status` 的结构化版,JSON 返回当前 branch、HEAD、clean / dirty、untracked、冲突状态。

```bash
g2c repo status --json
g2c repo status --repo /path/to/repo --json
```

### 6.4 `g2c repo conflict-check [--revision <rev>] <change>`

问 Gerrit:这个 change 当前能否 merge(走 `SubmitTypeInfo` / `mergeable`)。**只读**,不修改本地。

```bash
g2c repo conflict-check 5 --json
```

### 6.5 `g2c repo cherry-pick [options] <change>`

走 **Gerrit REST** 创建 cherry-pick change。返回 `mode: "gerrit-rest"`,带源 change、目标分支、revision 和新 change 的 normalized 信息。

| 选项 | 说明 |
|---|---|
| `--target <branch>` | 目标分支,必填 |
| `--revision <rev>` | revision(默认 `current`) |
| `--msg <message>` | cherry-pick 提交说明 |
| `--notify <notify>` | `NONE` / `OWNER` / `OWNER_REVIEWERS` / `ALL` |
| `--keep-reviewers` | 复制原 reviewer 到新 change |
| `--allow-conflicts` | 允许 Gerrit 创一个有冲突的 change |
| `--repo <path>` | 为兼容 PRD 接受,实际不消费 |

```bash
g2c repo cherry-pick 7 --target release/1.0 --json
```

> 本地 fetch + cherry-pick + push 的增强版**未实现**(TODO 3.3);`conflict-start` 提供本地链路。

### 6.6 冲突链(`repo conflict-*`)

完整流程:`conflict-start → conflict-read → conflict-resolve → conflict-continue`,任何阶段可 `repo abort` 中止。

#### 6.6.1 `g2c repo conflict-start [options] <change>`

在本地 fetch 目标分支和当前 revision,基于目标分支创建 `g2c-conflict-<change>` 本地工作分支,尝试 cherry-pick。

- dirty worktree → 返回 `WORKTREE_DIRTY`。
- 无冲突 → 返回 clean cherry-pick 结果。
- 冲突 → 返回冲突文件列表、Git 状态、下一步建议。

| 选项 | 说明 |
|---|---|
| `--repo <path>` | 本地 Git 仓库 |
| `--revision <rev>` | revision(默认 current) |

```bash
g2c repo conflict-start 5 \
  --repo /path/to/repo --json
```

#### 6.6.2 `g2c repo conflict-read [options] <file>`

读冲突文件并解析 conflict hunks(ours / theirs / marker 行号 / 文件路径)。CLI 走 `src/git.ts` 的 conflict marker parser。

| 选项 | 说明 |
|---|---|
| `--repo <path>` | 本地 Git 仓库 |

```bash
g2c repo conflict-read cli-conflict.txt \
  --repo /path/to/repo --json
```

#### 6.6.3 `g2c repo conflict-resolve [options] <file>`

用 `--content-file` 覆盖冲突文件,执行 `git add`。**不直接写远端**;所有冲突解决后必须显式 `conflict-continue --yes`。

| 选项 | 说明 |
|---|---|
| `--content-file <path>` | 解决后的内容文件 |
| `--repo <path>` | 本地 Git 仓库 |

```bash
g2c repo conflict-resolve cli-conflict.txt \
  --content-file /tmp/resolved-cli-conflict.txt \
  --repo /path/to/repo --json
```

#### 6.6.4 `g2c repo conflict-continue [options]`

确认无未解决冲突 → `git cherry-pick --continue` 或 `git rebase --continue` → push 到 `origin HEAD:refs/for/<change.branch>`。**必须 `--yes`**,否则返回 `PROTECTED_ACTION`。

| 选项 | 说明 |
|---|---|
| `--repo <path>` | 本地 Git 仓库 |
| `--yes` | 真的 push |

```bash
g2c repo conflict-continue \
  --repo /path/to/repo --yes --json
```

#### 6.6.5 `g2c repo abort [options]`

中止进行中的 cherry-pick / rebase。仅在检测到对应 Git 状态时执行;无状态时返回结构化错误或 no-op。

| 选项 | 说明 |
|---|---|
| `--repo <path>` | 本地 Git 仓库 |

```bash
g2c repo abort \
  --repo /path/to/repo --json
```

---

## 7. server

| 命令 | 说明 |
|---|---|
| `g2c server version` | Gerrit 服务端版本 |
| `g2c server capabilities` | 服务端能力清单 |
| `g2c server config` | 服务端 info / config |

```bash
g2c server version --json
g2c server capabilities --json
g2c server config --json
```

---

## 8. 输出格式

### 8.1 三种格式

| 格式 | 触发方式 | 适用场景 |
|---|---|---|
| `table` | 默认 / `--format table` | 人眼 |
| `json` | `--json` / `--format json` | Agent / 程序消费 |
| `csv` | `--format csv` | Excel / DataFrame |

`--json` 走的是结构化 `{ok, data, warnings, meta}` 信封,Agent 处理时不要只看 `data`,**先判 `ok`**,再读 `warnings`。

### 8.2 写文件

```bash
g2c --format csv --output /tmp/open-changes.csv \
  change list --query 'status:open owner:self' --all --page-size 100

g2c --format csv --output /tmp/change-7.csv change info 7
```

`--output` 配合 `--format table` 也能落盘;`--output` 配合 `--json` 不推荐(信封会被写入,Agent 取数时反而麻烦)。

### 8.3 友好字段 vs `--full`

- `change list` / `change info` / `me --status` 默认字段:`URL-number`、`changeType`、`subject`、`owner`、`ownerEmail`、`reviewers`、`repo`、`branch`、`updated`、`cr`、`Git-Change-Id`、`Gerrit-REST-id`、`currentRevision`、`insertions`、`deletions`、`totalCommentCount`、`unresolvedCommentCount`。
- 完整 normalized(包括 `status`、`labels`、`submitRequirements` 等):加 `--full`,对 `json` / `csv` / `table` 三种格式都生效。

### 8.4 Agent 推荐调用模式

```bash
# 1) 先做汇总,确定范围
g2c --json me --status open

# 2) 大结果集必须 --all + --page-size,避免漏
g2c --json change list --query 'status:open owner:self' --all --page-size 100

# 3) 写操作先 dry-run,再 --yes
g2c --json batch submit --query 'status:open owner:self label:Code-Review=+2' --limit 5
g2c --json batch submit --query 'status:open owner:self label:Code-Review=+2' --limit 5 --yes

# 4) 读单条一律用 --json
g2c --json change info 7
g2c --json change diff 7 --file <file>

# 5) 出错先看 meta 和 warnings
g2c --json change info 7 | jq '.meta, .warnings'
```

---

## 9. 常见错误码

CLI 错误统一进 JSON 信封的 `meta.error` 或顶层非 0 退出。常见类型:

| code | 触发 |
|---|---|
| `WORKTREE_DIRTY` | `conflict-start` 时本地有未提交改动 |
| `PROTECTED_ACTION` | `conflict-continue` 没传 `--yes` |
| `AUTH_FAILED` | 用户名 / 密码错 / 账号无权限 |
| `NOT_FOUND` | change / 仓库 / 分支 / revision 不存在 |
| `CONFLICT` | `conflict-start` 解析到未解决的冲突 |
| `INVALID_QUERY` | Gerrit query 语法错 |
| `MISSING_ARG` | 必填参数缺失 |

---

## 10. 完整命令索引

```
g2c
├── auth
│   ├── login            配置 Gerrit 凭据
│   └── status           验证连通性 + 当前账号
├── config
│   ├── get [key]        读生效配置
│   └── set <k> <v>      写单条配置
├── user
│   ├── search <query>   查账号
│   ├── lookup <query>   查账号(同 search)
│   └── perms [--change] 查当前用户权限
├── me [--status ...]    最近 7 天汇总 / 列表
├── list-repo            仓库列表
├── list-branch [repo]   分支列表
├── change
│   ├── list             查询
│   ├── info             单条元数据
│   ├── parent / merged-as / clone-url
│   ├── revisions        patch set 列表
│   ├── files            文件列表
│   ├── export           REST 导出变更文件和 patch
│   ├── file-content     读 / 下载文件
│   ├── diff             diff
│   ├── patch            patch(.base64 / .zip)
│   ├── comments         评审
│   ├── messages         change message
│   ├── submitted-together
│   ├── related
│   ├── abandon / restore / rebase / revert
│   ├── move / wip / ready
│   ├── topic (get/set/delete)
│   ├── hashtags (get/set)
│   ├── reviewer (list/add/delete/suggest)
│   └── attention (show/add/remove)
├── review
│   ├── message          发评审消息
│   ├── comment          inline 评审
│   ├── score            打 label
│   └── submit           提交
├── batch
│   ├── score            批量打分(默认 dry-run)
│   └── submit           批量 submit(默认 dry-run)
├── repo
│   ├── clone            shallow/full clone 目标分支
│   ├── download         只 fetch Change ref
│   ├── status           本地 git 状态
│   ├── conflict-check   问 Gerrit 能否 merge
│   ├── cherry-pick      REST 版
│   ├── conflict-start   本地 fetch + cherry-pick
│   ├── conflict-read    读冲突文件
│   ├── conflict-resolve 解决冲突
│   ├── conflict-continue push 到 refs/for/<branch>(需 --yes)
│   └── abort            中止 cherry-pick / rebase
└── server
    ├── version
    ├── capabilities
    └── config
```

---

## 11. 还没实现的(透明告知)

- `--verbose` 字段已声明,但**未消费**(TODO 5.1)— 不会输出请求路径 / 耗时 / 重试。
- `review comment --range <startLine,startChar,endLine,endChar>`(TODO 5.3)— 只支持 `--line`。
- `change diff --base / --parent / --whitespace`(TODO 5.4)。
- 表格宽度**未处理全角字符**(TODO 6.2),中文 / emoji 会偏移。
- 表格输出**未覆盖 commander 注册 / 参数解析 / 错误路径**的 CLI 集成测试(TODO 6.4)。

如要开发任意写操作或批量命令,请严格遵守第 0.3 节保护协议。
