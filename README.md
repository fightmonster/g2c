# g2c

> Gerrit-to-Claw CLI — Gerrit review 自动化命令行工具,支持人与 AI Agent 两种使用方式。

[![npm version](https://img.shields.io/badge/npm-1.0.0-blue.svg)](https://www.npmjs.com/package/g2c)
[![node](https://img.shields.io/badge/node-%E2%89%A520-green.svg)](https://nodejs.org)
[![license](https://img.shields.io/badge/license-MIT-brightgreen.svg)](LICENSE)

`g2c` 是 `gerrit2claw` 的命令行入口,把 Gerrit REST API 与本地 Git 仓库操作封装成 50+ 条结构化命令。设计目标:

- **可被 AI Agent 直接消费**:`--json` 信封、`--help` 全命令覆盖、`g2c help` 内置完整中文参考手册
- **写操作有保护协议**:`batch` / `conflict-continue` 默认 dry-run,必须显式 `--yes` 才落 Gerrit
- **可恢复的本地冲突链**:`conflict-start → conflict-read → conflict-resolve → conflict-continue` 五步走

同系列工具:[j2c (jira2claw-cli)](#相关项目)。

---

## 安装

### 方式 A:从 npm 全局安装(发布后)

```bash
npm install -g g2c
g2c --version     # 应输出 1.0.0
```

### 方式 B:从本地源码安装(开发或未发布)

```bash
git clone <this-repo>
cd gerrit-cli
npm install
npm run build
npm install -g .
```

### 方式 C:不安装,直接跑

```bash
git clone <this-repo>
cd gerrit-cli
npm install
npm run dev -- auth status
```

---

## 快速上手

### 1. 登录 Gerrit

```bash
g2c auth login \
  --url http://localhost:8080 \
  --ssh-url ssh://luojun@localhost:29418 \
  --username luojun \
  --password '<http-password>' \
  --repo /Users/luojun/workspace/<your-repo> \
  --branch master
```

凭据写入 `~/.gerrit2claw/config.json`(目录 0700,文件 0600)。环境变量(`GERRIT_HTTP_URL` / `GERRIT_USERNAME` / `GERRIT_PASSWORD`)会覆盖文件配置。

### 2. 看一眼状态

```bash
g2c                # 默认输出:连接状态、服务器、当前用户、配置路径、内置手册路径
g2c auth status    # 验证连通性 + 当前账号
g2c me             # 名下 change 数量汇总(Open/Merged/Abandoned)
```

### 3. 跑一次查询

```bash
g2c change list --query 'status:open owner:self' --limit 5 --json
g2c change list --query 'status:open owner:self' --all --page-size 100 --json
g2c change info 7 --json
```

### 4. 评审 / 打分

```bash
g2c review score 3 --label Code-Review=+2 --json
g2c review submit 3 --json
```

### 5. 批量(默认 dry-run,需 --yes)

```bash
g2c batch score --query 'status:open owner:self' --label Code-Review=+1 --limit 5 --json
g2c batch submit --query 'status:open owner:self' --limit 5 --yes --json
```

### 6. 本地冲突处理链(可选)

```bash
g2c repo conflict-start 5 --repo /path/to/repo
g2c repo conflict-read cli-conflict.txt --repo /path/to/repo
g2c repo conflict-resolve cli-conflict.txt \
  --content-file /tmp/resolved.txt \
  --repo /path/to/repo
g2c repo conflict-continue --repo /path/to/repo --yes
```

---

## AI Agent 专属使用指南

本 CLI 的核心目标之一是**让集成 AI Agent(Claude / Cursor / Aider / Cline / 自研 agent)零摩擦地消费 Gerrit 数据**。下面是约定。

### 字段命名(全文一致)

Agent 在解析 JSON 时只用关心这几个字段名:

| Agent 读到的字段 | 含义 | 例子 |
|---|---|---|
| `URL-number` | Gerrit 整数 ID,URL 末尾那个数字 | `7` |
| `Git-Change-Id` | commit message 里的 SHA,跟着提交走 | `I0000000000000000000000000000000000a10001` |
| `Gerrit-REST-id` | `项目~编号`,REST API 用 | `g2c-e2e-alpha~7` |
| `currentRevision` | 一次 push 的 patch set SHA | `2f6cb182cf01360d8e7371991c03c1090381c7f1` |

`change list` / `change info` / `me --status` 默认只输出**友好字段**(`URL-number` / `subject` / `owner` / `repo` / `branch` / `updated` / `CR` / `Git-Change-Id` 等);要看 `labels` / `submitRequirements` 等原始结构,加 `--full`。

### 拿帮助(三种姿势,选合适的)

```bash
# 1) 速查命令签名(commander 简版)
g2c <subcommand> --help

# 2) 拿完整中文参考手册(随包发布,离线可读)
g2c help                       # 25 KB markdown
g2c help change                # 拿手册第 3 节(change 命令组)
g2c help 3                     # 同上,按编号

# 3) 套上 JSON 信封,Agent 友好
g2c --json help --sections     # 列出全部章节
g2c --json help 3              # 拿第 3 节 JSON
g2c --json help --path         # 拿本机手册绝对路径
```

### 默认输出 JSON 信封

每条命令 `--json` 输出固定结构:

```json
{
  "ok": true,
  "data": <T>,
  "warnings": [],
  "meta": { "command": "change.list", "durationMs": 21 }
}
```

**Agent 必读**:

1. 先判 `ok`,为 `false` 时读 `error.code` 决定下一步(常见码见手册第 9 节:`WORKTREE_DIRTY` / `PROTECTED_ACTION` / `AUTH_FAILED` / `NOT_FOUND` / `CONFLICT` / `INVALID_QUERY` / `MISSING_ARG`)
2. `warnings` 数组非空也要读,通常是"账号找不到但仍继续"等非致命告警
3. `meta.command` 是命令路径(点号分隔),可用于日志/审计
4. 大量数据用 `--all --page-size 100` 自动翻页,配合 `--limit` 做硬上限

### 写操作保护协议(Agent 必须遵守)

凡是要写 Gerrit 或 push 到 `refs/for/*` 的命令,统一规则:

| 命令 | 保护机制 |
|---|---|
| `batch score` / `batch submit` | 默认 dry-run,必须 `--yes` 才执行 |
| `repo conflict-continue` | 必须 `--yes` 才 push |
| `repo cherry-pick` | 走 Gerrit REST,无需 `--yes`,但 JSON 标 `mode: "gerrit-rest"` |
| 其它单条写(`abandon` / `restore` / `rebase` / `revert` / `move` / `wip` / `ready` / `topic` / `hashtags` / `reviewer` / `attention` / `review *`) | 按调用即生效,**Agent 调用前必须先 dry-run 检查**(比如 `change info <n> --full --json` 看 `submitRequirements`) |

**Agent 推荐调用模式**:

```bash
# 1) 干跑计划
g2c --json batch submit --query 'status:open owner:self label:Code-Review=+2' --limit 5

# 2) 看到 planned / canSubmit 全 true 才加 --yes
g2c --json batch submit --query 'status:open owner:self label:Code-Review=+2' --limit 5 --yes

# 3) 单条写前查
g2c --json change info 7 --full
# 读 data.submitRequirements,确保都 SATISFIED / NOT_APPLICABLE 再 --json review submit 7
```

### 输出格式

| 触发 | 格式 | 适用 |
|---|---|---|
| `--json` / `--format json` | JSON 信封 | Agent / 程序 |
| `--format table`(默认) | 表格 | 人眼 |
| `--format csv` | CSV | Excel / DataFrame |
| `--output <path>` | 配合 `--format` 落盘 | 导出 |

### Agent 安装包内定位文档

`npm install -g g2c` 后,手册落盘在:

```
/usr/local/lib/node_modules/g2c/docs/HELP.md
```

Agent 三种拿路径:

```bash
g2c help --path                                # 最稳,直接走 CLI
node -e "console.log(require.resolve('g2c/package.json'))"  # Node 解析
ls "$(npm root -g)/g2c/docs/"                  # 拼路径
```

---

## 完整命令索引

```
g2c
├── auth               login / status
├── config             get / set
├── user               search / lookup / perms
├── me                 汇总 / 列出名下 change
├── list-repo          仓库列表(支持 --prefix/--match/--regex/--description/--branches)
├── list-branch        分支列表
├── change             读 + 状态变更 + 协作元数据(共 25+ 子命令)
│   ├── list / info / revisions / files / file-content / diff / patch
│   ├── comments / messages / submitted-together / related
│   ├── abandon / restore / rebase / revert / move / wip / ready
│   ├── topic / hashtags / reviewer / attention
├── review             message / comment / score / submit
├── batch              score / submit(默认 dry-run)
├── repo               status / conflict-check / cherry-pick / conflict-* / abort
└── server             version / capabilities / config
```

详细参数、示例、错误码、保护协议见 `docs/HELP.md`(`g2c help`)。

---

## 配置

### 配置源优先级

```
命令行全局选项  >  环境变量  >  ~/.gerrit2claw/config.json  >  .env / .g2crc(只读兼容)
```

### 必需字段(写到 config.json)

| 字段 | 说明 |
|---|---|
| `GERRIT_HTTP_URL` | Gerrit HTTP 入口,如 `http://localhost:8080` |
| `GERRIT_USERNAME` | Gerrit 用户名 |
| `GERRIT_PASSWORD` | Gerrit HTTP 密码 |

### 可选字段(可省)

| 字段 | 说明 |
|---|---|
| `GERRIT_SSH_URL` | SSH 入口,`g2c` 不直接用,记录备查 |
| `G2C_DEFAULT_REPO` | 本地 Git 仓库默认路径,`repo` 子命令用 |
| `G2C_DEFAULT_BRANCH` | 默认目标分支,实际场景几乎都用 change 自带 branch |

> `auth login` 和 `config set` **只写**用户级 `~/.gerrit2claw/config.json`,不会污染项目仓库。

### 环境变量(覆盖文件配置)

```bash
export GERRIT_HTTP_URL=http://localhost:8080
export GERRIT_USERNAME=luojun
export GERRIT_PASSWORD='<http-password>'
export G2C_DEFAULT_REPO=/Users/luojun/workspace/<repo>
export G2C_DEFAULT_BRANCH=master
```

---

## 贡献 / 开发指南

### 本地开发

```bash
git clone <this-repo>
cd gerrit-cli
npm install
npm run dev -- auth status      # tsx 跑 src,改完即生效
```

### 测试 / 检查

```bash
npm run typecheck    # tsc --noEmit
npm run build        # tsc → dist/
npm test             # vitest,24 个测试
npm audit            # 依赖安全审计
```

测试涵盖 `config` / `diff` / `gerrit client` / `git conflict marker` / `ui` / `program integration` 六个文件。

### 打包 tgz(本地验证发布物)

```bash
npm pack
# → g2c-1.0.0.tgz
tar tzf g2c-1.0.0.tgz | head
# 应包含: package/dist/* + package/docs/HELP.md + package/package.json + package/README.md
# 不应包含: src/、testfiles/、node_modules/、.git/
```

`package.json` 的 `files` 字段是白名单,新增要发布的文件请加进去;`src/` 等私有文件**不会**被打包,已验证。

### 发布到 npm

```bash
npm login
npm version patch    # 0.1.0 → 0.1.1
npm publish
```

发布后,集成 AI Agent 可以通过以下方式定位本工具:

```bash
node -e "console.log(require.resolve('g2c/package.json'))"
# → /usr/local/lib/node_modules/g2c/package.json
```

### 提交约定

- 一个 PR 一件事
- `npm run typecheck && npm test` 必须过
- 新增命令需更新 `docs/HELP.md`
- 写操作 / 批量命令必须遵守 `--yes` / dry-run 保护协议

---

## 路线图 / 已知缺口

完整未完成项见 `Gerrit2Claw-cli_TODO.md`。当前 1.0 主要缺口:

- `--verbose` 仅声明,未消费请求路径/耗时/重试日志
- `review comment --range <startLine,startChar,endLine,endChar>` 未实现(只支持 `--line`)
- `change diff --base / --parent / --whitespace` 未实现
- 表格输出未处理全角字符宽度(中文/emoji 对齐)
- CLI 集成测试覆盖度有限(commander 注册 / 参数解析路径待扩)

---

## 许可证

MIT — 见 [LICENSE](LICENSE)。

---

## 相关项目

`g2c` 是 "2claw" CLI 家族成员,设计风格与配套工具保持一致:

- **j2c (jira2claw-cli)** — Jira 自动化(兄弟项目)

如果你在维护类似的 "X-to-claw" CLI,欢迎把 `g2c` 当模板:**docs 进 npm 包 / 默认 home 带 AI 提示 / `g2c help` 内置手册 / 写操作 `--yes` 保护** 是这个家族的公约。
