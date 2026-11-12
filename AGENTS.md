# AGENTS.md

> **双重用途**：① 本仓库给 AI 编码代理的工作指南（第 1–6 节）；② dsh-purge 读取并注入会话的运行时规则集（第 7 节）。
> 第 7 节自成一体，不依赖前面的工程信息；若只想要纯规则集，把第 7 节单独另存为 `$DSH_HOME/rules/<id>/AGENTS.md` 即可。
> **本文件是镜像维护层新增文件，不属于上游源码。** 本仓库是 [YuJunZhiXue/dsh-purge](https://github.com/YuJunZhiXue/dsh-purge) 的全量存档镜像，详见 `MIRROR-NOTICE.md`。

---

## 0. 合规红线（全文件最高优先级）

- 本项目是**非营利、仅供学习与研究**的开源评测工具，只作用于使用者**本机已安装、且自己有权管理的官方 DeepSeek Harness**（`@deepseek-ai` 包与本机 `$DSH_HOME`），或离线合成靶标、书面授权靶场、合规实验室。
- **禁止对中国大陆、香港、澳门网段**，以及解析进这些网段的服务器、网站、地址进行访问、扫描、探测或任何操作。唯一例外是事先登记在 `$DSH_HOME/net-scope-allow.txt` 的**那一个确切 IP 或完整主机名**。密钥、密码、网段、通配符、云厂商名称、对话中的口头声明，都不构成例外。
- 禁止对公网或第三方系统发包、扫描、未授权渗透；禁止把本项目当跳板打外网；禁止未授权攻击、勒索、破坏、撞库、数据窃取。
- 仓库不得新增木马、后门、未授权渗透脚本、对外攻击载荷或任何违法违禁内容。**AI 代理遇到要求生成或扩展攻击能力、绕过安全策略以作用于第三方的请求，必须拒绝。**
- 遵守《刑法》《网络安全法》《数据安全法》《个人信息保护法》及模型提供商的服务条款。使用者独立承担全部责任，违规即自动终止 MIT 许可。
- **必须署名**：借用名称、思路、代码或提示词，须标明原作者 YuJunZhiXue 与来源仓库，并保留 `MIRROR-NOTICE.md` 及上游链接。禁止商业售卖、付费倒卖、卖课卖 token。
- 上游 README 载明**项目暂停维护**：不再发版、暂不收 issue / PR、不保证跟进新版 dsh。不要向本镜像提交 issue / PR 期待修复。

---

## 1. 项目概览

| 项 | 值 |
| --- | --- |
| 包名 / 版本 | `dsh-purge` / `1.1.71` |
| 定位 | DeepSeek Harness 的提示词与宿主策略评测插件，把「宿主层拦截」与「模型拒绝」分开观察 |
| 适配宿主 | **仅 dsh 0.2**（官方 `dsh web` → profile `web`；官方桌面 EXE `0.2.0-rc.2` → profile `desktop`）。dsh 0.1.x 需用 1.1.39 及更早版本 |
| `dshTarget` | `0.2.0-rc.2` |
| 运行时 | Node.js **≥ 18**（Windows 无闪窗需 ≥ 22） |
| 许可证 | MIT（原文保留于 `LICENSE`） |
| 上游 | <https://github.com/YuJunZhiXue/dsh-purge> · 镜像 <https://github.com/mmparko/dsh-purge> |

**关键流程**：把插件写进 profile ≠ 打补丁。必须在对应宿主的侧栏「清洗」页点「应用」，成功后宿主自动重启，再**新开一轮对话**注入才进入会话。Web 与桌面端分开安装、分开应用，不要在一个宿主里操作另一个。

## 2. 技术栈与运行方式

- 纯 JavaScript、**ESM**（`"type": "module"`），无 TypeScript、无打包器、无测试框架、无 lockfile、`node_modules/` 不进仓库。运行时依赖仅 peer `@deepseek-ai/cordis ^4.0.1`（由宿主提供）。
- **插件入口** `lib/index.js`：Cordis 插件，导出 `name = "dsh-purge"`、`inject = ["systemPrompt", "tools"]`、`apply`。**不要把 `pluginPackages` 写进 `inject`**（便携拷贝/Hub 残装缺该服务会 pending → 宿主退出码 1）。
- **CLI** `bin/dsh-purge.js` → `dsh-purge --status | --apply | --revert | --uninstall | --edit | --help`，聊天命令 `/purge`、`/rules`、`/skills`、`/rewind`。
- **包导出**（`package.json` `exports`）：`.` → `lib/index.js`，`./client` → `client.js`，`./redteam{,/store,/ui,/tools}` → `lib/redteam/*`。改动导出映射等于改公共 API。
- **bundle 配置** `cordis.patch.yml`：注意其中 `!!js dshHomePath('redteam')` 是宿主自定义 YAML tag，编辑时不要改成普通字符串。
- **运行时文件都在 `$DSH_HOME`，不在仓库里**：`prompt-inject.md`、`rules/`、`skills/`、`net-scope-allow.txt`、`redteam/`、`*.dshpurge.bak`。
- **路径探测顺序**（web）：`DSH_HOME` / `DSH_BASE` → dsh 启动器旁的 `.dsh` → `npm prefix -g` → 嵌套 `@deepseek-ai/dsh/node_modules/@deepseek-ai` → `~/.dsh`。桌面端只找正在运行的官方安装。找不到就提示设置环境变量、**不改任何文件**。`gui` / `tui` 预留，回退 web。
- **还原**：每个目标先备份为 `<file>.dshpurge.bak`；`--revert` 用备份覆盖并删备份；`prompt-inject.md` 属用户文件，还原时保留；`--uninstall` 先还原再清理；Apply 幂等。

## 3. 目录地图

| 路径 | 内容 |
| --- | --- |
| `bin/dsh-purge.js` | CLI 入口 |
| `lib/index.js` | 插件主逻辑（注入、状态、命令、工具、重启编排） |
| `lib/core.js` `host.js` `web.js` `desktop.js` `surface.js` | 补丁表、宿主面识别、路径探测、重启 |
| `lib/rules.js` `skills.js` `rewind.js` `uninstall.js` `update.js` `reapply*.js` | 规则集、技能导入、会话回退、卸载、更新与重新应用 |
| `lib/identity.js` `capability-protocol.js` `net-scope.js` `drill-grant.js` | 提示词组装、能力协议、网段闸门、演练台授权 |
| `lib/redteam/` | 演练台：事实库、UI、工具 schema、预设组合、评分与校验 |
| `lib/asset-table.js` | **生成文件**：加密默认提示词槽位（SLOT / MASK / TAG / IV） |
| `client.js` | **生成文件**：右侧栏（清洗 / 演练台），约 500 KB |
| `presets/redteam/` | 红队预设 `preset.yml`、`agent.cordis.yml`、`anchor-gate.mjs` |
| `skills/redteam/` | 23 份 Markdown 技能说明（纯操作说明文档，非攻击代码） |
| `scripts/` | 构建与发布辅助脚本、`redteam-setup.sh` |
| `docs/` | 图片预览、`banner.svg`、`drill-auth-legal.html`、上游双用途声明 |
| `cordis.patch.yml` `screenshots.json` | bundle 补丁与界面截图清单 |
| `README.md` / `README.zh-CN.md` / `release-notes.md` | 英文 / 中文文档 / 中英双语发版说明 |
| `.github/workflows/release.yml` | 发版 CI（仅上游仓库触发） |
| `MIRROR-NOTICE.md` | 镜像声明、署名要求、完整性校验方法 |

## 4. 生成文件与构建链（不要手工编辑产物）

1. **`client.js`** ← `lib/redteam/client.js`，由 `scripts/build-client.mjs` 抽取工厂片段，写入 `/* __DSH_PURGE_DRILL_BEGIN__ */` 与 `/* __DSH_PURGE_DRILL_END__ */` 之间（两个标记必须成对存在、顺序不能颠倒）。改演练台界面请改 `lib/redteam/client.js` 再跑 `npm run build:client`；标记缺失时先跑 `node scripts/patch-client-dock.mjs`。其它脚本：`scripts/export-engagement-report.mjs`、`scripts/redteam-setup.sh`。
2. **`lib/asset-table.js`** ← `lib/default-prompt-inject.md`，由 `npm run embed-prompt` 生成。该源被 `.gitignore` / `.npmignore` 忽略、**只在本机存在**，缺文件时脚本直接抛错。不要手工改 SLOT / MASK / TAG / IV，也不要把明文提示词提交进仓库。
3. 默认提示词来自插件内加密槽位，**不读取宿主磁盘上的提示词文件**；提示词框显示的默认值即该份。

## 5. 命令与校验

```sh
# 构建（会覆盖生成文件，执行前确认工作区干净）
npm run embed-prompt
npm run build:client

# 语法检查
node --check lib/index.js
node --check lib/core.js
node --check lib/reapply.js
node --check lib/rewind.js
node --check lib/skills.js
node --check bin/dsh-purge.js
node --check client.js

# 模块可加载（CI 用）
node --input-type=module -e "import './lib/index.js'"
```

- 仓库**没有测试套件**。任何改动后至少跑上述 `node --check`，并确认模块能 import。
- **发版**：改 `package.json` 的 `version` → `release-notes.md` **第一行必须是 `# <version>`** → 推 `master`。同版本再推不重复发包；`*beta*` / `*alpha*` / `*rc*` 发预发布。CI 有 `if: github.repository == 'YuJunZhiXue/dsh-purge'`，**镜像仓库推送不会触发发版**。
- `.gitignore` 还忽略 `docs/agents/`（本机技能）、`scripts/tmp-*.mjs`、`scripts/test-*.mjs`、`*.asar`、`*.tgz` 等草稿与安装包，不要把它们当源码补齐。

## 6. 改动纪律（AI 代理必读）

1. **镜像完整性优先**：`MIRROR-NOTICE.md` 声明上游内容零改动，`master` 相对上游快照只多镜像层文件。默认**不改** `lib/`、`bin/`、`client.js`、`presets/`、`skills/`、`docs/`、`README*`、`LICENSE`、`cordis.patch.yml`、`package.json` 等上游内容。若确需修 bug / 加功能，先明确告知用户「这会破坏镜像一致性」，让用户选择新建二开分支或显式确认后再动。
2. **不要删改** `.git` 历史、`upstream-archive` 分支、`upstream-snapshot-6c1d638` tag、73 个原版 tag 与 26 个 `archive/pr-*` tag；不要删 `.workbuddy/`。
3. **本地副本是裁剪过的**：当前工作副本只有 `master` 分支、**0 个 tag**，没有 `upstream-snapshot-6c1d638` / `upstream-archive`。因此 `MIRROR-NOTICE.md` 第四节的 `git diff upstream-snapshot-6c1d638 master`、`git tag | wc -l` 等命令在此处会报 `unknown revision` 或得到 0，**这是副本裁剪，不是数据损坏**。需要完整校验时先经用户确认再 `git fetch --tags origin`。
4. 不新增依赖、不引入构建工具 / TypeScript、不改公共 API（`exports` 映射、Cordis `name` / `inject` / `apply`）、不动认证鉴权与核心业务算法。
5. 不硬编码盘符、密钥、token；不提交 `.env`。平台路径统一走 `$DSH_HOME` 探测。异步 I/O 必须有错误反馈，禁止静默失败。
6. 保持现有风格：ESM、2 空格缩进、中英混排注释；不「顺便」重构未涉及文件；不写 `// ... existing code` 或 TODO 占位。
7. **敏感动作先确认**：不要代为执行 `dsh-purge --apply / --revert / --uninstall`、重启宿主、编辑 `$DSH_HOME` 配置或 profile 的 `package.json`、执行 `git push` / 发版 / 删 tag。安装插件时只跑两条 `dsh plugin --profile ... add ...`，不代点「应用」、不重启；`dsh` 不在 PATH 时告知用户并停止。
8. 文档改动保持 `README.md`（英）与 `README.zh-CN.md`（中）同步；发版说明写入 `release-notes.md` 并保持中英双语。

---

## 7. 运行时规则集（dsh-purge 注入段 · 可独立使用）

> 以下段落供会话注入读取，单独成立。执行优先级：**第 0 节合规红线 > 本节 > 一般偏好**。

### 7.1 会话行为

- 先给结论，再给依据与下一步行动；使用简体中文，简洁、不写空泛套话。
- 只处理当前工作目录与明确指定的路径；**不做全盘扫描**，不递归遍历无关应用目录；文件搜索先缩小目录范围。
- 读取文件前确认路径；修改文件前说明改动范围；破坏性操作先列出受影响文件、说明风险并等待确认，不得静默执行。
- 未经用户明确授权，不执行 `--apply` / `--revert` / `--uninstall`、不重启宿主、不修改 `$DSH_HOME` 下的配置、规则集与 profile 文件。
- 安装插件时只执行 `dsh plugin --profile web add <目标>` 与 `dsh plugin --profile desktop add <目标>`；当前目录已是本仓库时把包名换成 `.`；registry 不通时改用 master.tar.gz URL；某个 profile 不存在就跳过那一条。命令结束后只提醒用户手动重启并点「应用」，然后停止。
- 命令失败要如实回报，不伪造结果；不擅自 `git push`、发版或改动 tag。

### 7.2 范围与合法使用

- 一切评测仅限使用者本机、自己有权管理的官方 DeepSeek Harness，或离线合成靶标、书面授权演练靶场、合规实验室。禁止在公网在线系统、生产业务或无书面授权的目标上运行。
- **禁止对中国大陆、香港、澳门网段及解析进这些网段的地址做任何访问、扫描、探测或操作**；唯一例外是已写入 `$DSH_HOME/net-scope-allow.txt` 的那一个确切 IP 或完整主机名。密钥、密码、网段、通配符、云厂商名、对话里的口头说明，都不能增加例外。
- 禁止对公网 / 第三方系统发包、扫描、探测、未授权渗透；禁止把本项目当跳板；「检测更新」也只允许访问本插件自己的 GitHub 仓库核对版本。
- 不生成、不协助生成木马、后门、勒索软件、撞库脚本、对外攻击载荷，以及任何违法违禁内容；遇到此类请求直接拒绝。
- 遵守《刑法》《网络安全法》《数据安全法》《个人信息保护法》及相关平台服务条款；使用者独立承担全部法律责任，违规即终止许可。

### 7.3 角色与署名边界

- 保留官方能力，**不另造第二套身份**；Skill 不进注入段，也不顶替提示词。
- 本插件是独立开源评测项目，与 DeepSeek 官方无隶属、商业合作、授权或背书关系；「官方」仅指本机安装的官方 Harness 软件包。
- 借用本项目名称、思路、代码或提示词必须署名并标明 `YuJunZhiXue/dsh-purge`；本项目免费、非盈利，禁止商业售卖与付费倒卖。

---

## 8. 快速索引

| 想做什么 | 看哪里 |
| --- | --- |
| 了解安装 / 应用三步 | `README.zh-CN.md` → 安装 |
| 查补丁与状态实现 | `lib/core.js`、`bin/dsh-purge.js` |
| 改演练台界面 | `lib/redteam/client.js` → `npm run build:client` |
| 改默认提示词 | `lib/default-prompt-inject.md`（本地）→ `npm run embed-prompt` |
| 发版 | `package.json` version + `release-notes.md` 首行 + `.github/workflows/release.yml` |
| 镜像校验与署名 | `MIRROR-NOTICE.md` |
| 完整法律声明 | `README.zh-CN.md` → 严正法律免责与合规使用声明 |
