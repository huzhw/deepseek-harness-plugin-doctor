---
name: deepseek-harness-plugin-doctor
description: DeepSeek Harness（DSH）插件与升级体检医生：dsh 起不来、页面白屏/Failed to load plugins、插件加载崩溃、app/引擎/插件升级后兼容问题的一站式诊断与修复。内置插件台账、三大已知病速查（pnpm store 漂移、client bundle externals 漂移、插件缺失重装）、版本三层结构（桌面app/引擎/插件）、引擎升级后必检清单、浏览器实测验证法（健康检查会假阳性）。另含：插件全量盘点与升级代办（npm dist-tags + git commit 比对法）、插件安全审计（出站声明法，安全≠成熟分开报）、市场插件成熟度分组与替代品调研（awesome-dsh-plugin 注册表 star 榜 + 量纲陷阱）、插件精简卸载（取证四件套）。触发词：dsh起不来、dsh起不来了、插件修复、dsh插件、dsh体检、dsh检查、dsh升级、引擎升级、dsh doctor、plugin doctor、missed the module table、ERR_PNPM_UNEXPECTED_STORE、Failed to load plugins、插件台账、升级体检、插件盘点、插件精简、卸载插件、插件替代、这插件安全吗、插件star。
author: 胡志伟
motto: "健康检查会说谎，浏览器不会——插件的事，眼见为实。"
---

# deepseek-harness-plugin-doctor —— DSH 插件与升级体检医生

## 这是什么

DeepSeek Harness（DSH）的插件与升级体检医生。dsh 起不来、页面白屏/报错、插件加载失败、app/引擎/插件升级后犯病，按本技能流程走：先分清三层版本，再按证据链定位，对照三大已知病速查修复，最后浏览器实测验收。

插件清单与历史修复案例：[references/台账.md](references/台账.md)

## 版本三层结构（排查前先分清是哪层的病）

| 层 | 版本示例 | 版本查询口径 | 谁负责升级 |
|---|---|---|---|
| 桌面 app（Tauri 壳） | 0.12.0 | app 自身更新通道 | 官方推送 |
| dsh 引擎（核心） | 0.1.5-rc.2 | npm `@deepseek-ai/dsh-web-app` 的 **next 标签**（其 latest 是 0.0.1-rc.1 占位假标签，别认） | 桌面 app 自动下载（github deepseek-harness-pkg releases） |
| 插件（各 npm 包） | 各自版本 | `npm view <pkg> dist-tags`（认 latest，另有 legacy/beta 通道） | dshmarket / app 刷新 |

## 诊断流程（按序执行，别跳步）

1. **后端活着吗**：`netstat -ano | findstr 13080`，有 LISTENING + pid 即活着
2. **HTTP 通吗**：`xh GET http://127.0.0.1:13080/`，返回 HTML 即通
3. **浏览器实测（关键步骤）**：用 dsh-builtin-browser 插件的 `browser_*` 工具打开 `http://127.0.0.1:13080/?token=<token>` → 截图 + console（**2026-09-12 起 DSH 已弃用 chrome-devtools MCP**；在 CC/Codex/ZCode 端做此体检才用它们的 chrome-devtools MCP）
   - token 从 `%APPDATA%\io.github.hairyf.deepseek-harness-desktop\logs\dsh-web.log` 最后一行拿（每次启动轮换）
   - **健康检查 74/74 通过 ≠ UI 正常**（假阳性实锤过），一切以浏览器截图和 console 为准
4. **日志定位**（都在 `%APPDATA%\io.github.hairyf.deepseek-harness-desktop\logs\`）：
   - `desktop.log` 后台总日志（插件补装、CORE_PLUGIN_*、启动流程）
   - `desktop.frontdesk.log` 前台（health check、startup failed 堆栈）
   - `dsh-web.log` dsh 进程 stdout（MCP 加载、web token）

## 三大已知病速查

### 病1：pnpm store 漂移

- **症状**：启动失败 `INTERNAL_PLUGIN_INSTALL_FAILED` + `ERR_PNPM_UNEXPECTED_STORE Unexpected store location`，desktop.log 指明犯病的 profile 目录
- **根因**：全局 `~\.npmrc` 的 `store-dir` 挪过位置，而 profile 的 `node_modules\.modules.yaml` 记录的还是旧 store 路径，pnpm 一比对就炸
- **修法**：给犯病 profile 的 `.npmrc` 钉死项目级 store（web/safe 都要钉）：
  ```
  store-dir=C:\Users\Administrator\AppData\Local\pnpm\store
  @deepseek-ai:registry=https://mirrors.cloud.tencent.com/npm/
  ```
- **验证**：`pnpm -C <profile> store path` 与 `.modules.yaml` 的 `storeDir` 一致；`pnpm -C <profile> install --frozen-lockfile` 秒过零下载

### 病2：client bundle externals 漂移（插件前端包引用了运行时没有的模块）

- **症状**：后端活着、HTTP 正常，浏览器渲染 "Failed to load plugins"，console：`failed to import loader entry xxxxxx (<插件名>): client-modules: require("process") missed the module table — not a platform seed word...`
- **根因**：插件新版打包时把 Node 内置模块（process/buffer 等）设为外部依赖，dsh client-modules 运行时模块表里没有这个种子。**先比对核心运行时新旧版本是否一致**（`@deepseek-ai/dsh-client-modules/lib/client.js`，旧核心留在 `%APPDATA%\io.github.hairyf.deepseek-harness-desktop\dependencies\`），一致则锅在插件构建
- **修法四步**（以 web profile 为例）：
  1. 列全：`grep -o 'require("[a-z_]*")' <profile>\node_modules\<插件>\lib\client.js | sort | uniq -c`
  2. 看用法：`grep -n -B2 -A12 'require("xxx")'`——有 `typeof` 守卫的垫空壳即可，只读 env 的垫 `{env:{}}`
  3. 备份到 `<profile>\.bak\<插件>-<版本>-client.js.bak`，再 Edit replace_all 打垫片
  4. 浏览器直接刷新即生效——**服务端每次页面加载现读磁盘**（loader entry 哈希会变），不用重启进程
- **已知垫片配方**（yaml@2.9.0 外置场景，dsh-permission-rules 0.7.1 实战验证）：
  - `require("process")` → `(globalThis.process ?? {env:{},emitWarning:function(){}})`
  - `var node_buffer = require("buffer");` → `var node_buffer = { Buffer: globalThis.Buffer };`（yaml 的 binary 标签有 `typeof Buffer === "function"` 守卫，自动走 atob/btoa 浏览器分支，零损失）
- **注意**：插件升级会覆盖垫片。复发时先看新版是否自带修复；没有就重打垫片，或回滚上一版（peer 依赖声明兼容就回得去，用 `npm pack <pkg>@<旧版>` 拉下来核对 `require("process")` 计数为 0 再定）

### 病3：插件缺失 / 重装失败

- **症状**：desktop.log `INTERNAL_PLUGIN_NEEDS_REINSTALL: <名字>（dep_ok=false, link_ok=false, expected=link:...）`
- **修法**：内部预设（dsh-tauri\*）app 启动自动补装（源头 `D:\tools\Deepseek Harness Desktop\resources\node_modules`），病1修好后它就能自己装好；市场插件走 dshmarket 重装；成功标志查 desktop.log `Preinstall plugins installed successfully`

## 插件全量盘点与升级代办（用户说"盘点插件 / 帮我升级"时走这节）

1. **盘点**：读 `profiles\web\package.json` dependencies → 逐条比对 `node_modules\<pkg>\package.json` 实装版本 vs `npm view <pkg> dist-tags`（加 `--registry=https://mirrors.cloud.tencent.com/npm/`）；git 源走第 3 条
2. **引擎**：`npm view @deepseek-ai/dsh-web-app dist-tags` 认 **next**；再拉 `versions` 全列确认没有更高版（latest 是占位假标签）
3. **git 源插件**：`git ls-remote --tags --refs <url>` 可直连 GitHub（shell 走 policy proxy 放行）。**版本判定以 lockfile 锁的 commit 对比远端 HEAD 为准，package.json 的 version 字段会骗人**（先例：win-terminal-inspector 作者打 tag 不 bump version，"1.0.0 vs v1.0.1"是假差异）
4. **app**：官方推送通道；GitHub `releases.atom` 直连解析异常、ghproxy 对 atom 403（对 raw 文件通）——查不了远端就明说，别编
5. **升级姿势**：`pnpm -C <profile> update <pkg>` 后必跑 `install --frozen-lockfile` 验证一致性 + 对改动插件跑病2扫描（内置模块 `require(` 计数）；输出 `Packages: -N` 是 prune 不是损坏，frozen 会按 lockfile 重建
6. **PS5.1 坑**：读 package.json 版本用正则 `"version"\s*:\s*"([^"]+)"`，别用 ConvertFrom-Json——含中文描述/特殊字符必炸，且失败时变量残留上轮值造成串行错版

## 插件安全审计法（用户问"这插件安全吗"时走这节）

1. **出站声明**：package.json 的 network permissions 计数 + dependencies/peerDependencies 构成
2. **判定口径**：0 出站声明 + 依赖全本地库（yaml/zod/chokidar 等）或 @deepseek-ai 官方件 = 可信
3. **安全性（审计通过）≠ 成熟度（版本线+补丁史），两个维度分开报，别混**（教训：permission-rules 是权限门禁核心，但 0.x + 手工垫片史 = 不成熟，分组成熟度时必须归观察块）
4. **实证记录有价值**：dsh-defend 2026-09-13 实证拦截过一次工具结果 secret 外泄（generic-assignment 规则），审计时引实证

## 升级后必检清单（app / 引擎 / 插件任一升级后跑一遍）

- [ ] desktop.log 有 `Preinstall plugins installed successfully`（内部预设补装完成）
- [ ] 浏览器实测：截图正常 + console 零报错（不信健康检查）
- [ ] 手工垫片是否被插件升级覆盖（对照台账垫过的插件名单）
- [ ] 换机器/挪盘后 store 对齐（病1 检查法）
- [ ] 版本三层各自到哪了（`npm view ... next/latest` vs 本机）

## 市场插件调研（用途/配置/成熟度分组/替代品/star，"列一下插件都是干啥的 / 有没有更好替代"时走这节）

1. **用途**：读各插件 package.json 的 `description` / `dshhub.summary`；PS 直读中文可能乱码——用正则抓或看 npm stderr 里的原始 JSON
2. **配置归属**（~/.dsh 根文件实查）：`rules.yaml`=permission-rules（**热重载改完即生效**，`/rules list` 查生效规则）、`pet.json`=dsh-tauri-pet、`storages\cost-meter\`=费用账本、`settings.yaml`=LLM 供应商（settings-curator 管）、`dream-skin.json`+`skin-center-active.json`=内置皮肤中心（非插件）；通用规律：**web-ui 插件→「设置→插件」面板，host-only→纯后台或专属文件**
3. **成熟度分块判据**：✅稳定=1.0+ 正式版本线 + 无手改史；🧪观察=0.x / git 源 / 有垫片史 / 依赖上游存亡。**观察≠不安全≠别用**，是"升级必看 changelog"；DSH 生态整体年轻（引擎才 0.1.5-rc），0.x 是常态
4. **替代品调研标准流程**：
   - `curl.exe -sS --max-time 280 -o $env:TEMP\awesome-plugins.json https://awesome-dsh-plugin.com/plugins.json`（约 3.2MB，90s 不够下完）
   - **必须用 node -e 解析**（PS5.1 ConvertFrom-Json 对大 JSON/中文字符必炸）
   - 按 category（security/usage/ui/memory/vision/docs/dev/session…）列 top 星竞品，与本机插件逐个对比
   - **star 量纲陷阱**：套件子包的星数是整个仓库的（dsh-web ★7131、archify ★53k、hindsight ★23k 都是主项目数，别当插件本体星）；**细分第一星低≠不成熟**（defend ★7 是注入拦截唯一，无同类竞品）
   - **同名仿品警示**：dsh-cost-meter 有 ★0/★2 仿品、better-sidebar 有 -N23 仿品——安装/核对认准 owner
5. **官方口径**：deepseek-ai **没有官方市场，也没指定任何市场**（官方只有 `dsh plugin add` CLI + Settings→Plugins 官方面板）；dshmarket=社区事实标准（app 出厂内置 + awesome-dsh-plugin 注册表唯一 Recommended，两者互为数据源）

## 插件精简/卸载（用户说"去掉不需要的"时走这节）

**取证四件套，全查完再定卸载清单**（凭印象必翻车）：
1. **配置实锤**：grep settings.yaml 是否真在用（例：dsh-opencode-session 靠 33 处 OpenCode 配置保命，看着像闲置实则在岗）
2. **官方能力覆盖**：desktop.log 是否已自带同能力（例：win-terminal-inspector vs 官方日志 "provides the official Windows process inspector"）
3. **活动痕迹**：dsh-web.log 有无该插件动静
4. **依赖方**：awesome-dsh-plugin 注册表收录情况 + 有没有其他插件依赖它（dsh-better-sidebar 被第三方注册页签，不能动）

**执行**：备份 package.json+pnpm-lock.yaml 到 `.bak\`（rN 序号递增）→ `pnpm -C <profile> remove <pkg>` → 验证（目录消失、deps 计数 28→27 这类、`install --frozen-lockfile` 秒过）→ **汇报必须附回滚命令** → 台账同步记账（另出确认词），防下次体检账实不符

## 红线与经验

- **杀 dsh 进程通常被拒（高权限）而且通常不需要**——client bundle 补丁即改即生效，刷新页面就行
- profile 的 `.npmrc` 必须钉 `store-dir`；全局挪 store 盘必炸 profile
- `@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app` 两条 `CORE_PLUGIN_PROFILE_ENTRY_MISSING` 是长期 WARN，不拦启动，别当病治
- `[pet-stream]` 2 秒重连刷屏是宠物插件常驻重连，与起不来无关
- 改任何文件前：备份到 `<profile>\.bak\`，方案带四字成语确认词
- **读含 token/bearer 的日志先脱敏**：`-replace '(token[=:])\S+','$1***'`——否则 dsh-defend 按规则拦（generic-assignment），直接 tail 必被拒
- **chrome-devtools MCP `Target closed` 终局结论（2026-09-12，DSH 已退役该 MCP 改用 dsh-builtin-browser 插件）**：根因三层 = 引擎 0.12.2 stdio 裸 spawn `npx` ENOENT + 四端共用 Edge profile 单例互顶 + **引擎子进程环境里 puppeteer 带管道句柄的浏览器孵化无声失败**（普通 spawn 正常、事件日志无崩溃、profile 目录零写入；"重启让 MCP 重拉"旧假设已证伪）。CC/Codex/ZCode 端仍用 chrome-devtools MCP（各占独立 `User Data MCP-<端名>` 目录）；排障证据链存 `~\.dsh\.bak\`（cdm-edge-probe.log 探针 / cdm-env-safe.txt 安全环境子集）；没有浏览器实测条件时，用"本会话即 13080 页面"活体证据 + 病2 bundle 主动巡检（全插件 `require("内置模块")` 计数=0）替代
- **grep/glob 递归扫 `~\.dsh` 会撞 `profiles\node_modules` 断链 junction**（react/immer/clsx 等旧结构遗留）报错一片——走精确文件路径，别递归扫 profile 根
- **台账/skill 改前备份序号惯例**：`.bak\<文件名>.bak-YYYYMMDD-rN`（r1、r2 递增），回滚全靠它；profile 三件套（package.json/pnpm-lock.yaml/cordis.patch.yml）动前必备

## 相关技能

- [deepseek-harness-settings-curator](https://github.com/huzhw/deepseek-harness-settings-curator)：DSH settings.yaml 模型配置梳理
- [agent-config-sync-check](https://github.com/huzhw/agent-config-sync-check)：四端同步守卫
