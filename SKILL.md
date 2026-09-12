---
name: deepseek-harness-plugin-doctor
description: DeepSeek Harness（DSH）插件与升级体检医生：dsh 起不来、页面白屏/Failed to load plugins、插件加载崩溃、app/引擎/插件升级后兼容问题的一站式诊断与修复。内置插件台账（web profile 26 依赖三分类）、三大已知病速查（pnpm store 漂移、client bundle externals 漂移、插件缺失重装）、版本三层结构（桌面app/引擎/插件）、引擎升级后必检清单、浏览器实测验证法（健康检查会假阳性）。触发词：dsh起不来、dsh起不来了、插件修复、dsh插件、dsh体检、dsh检查、dsh升级、引擎升级、dsh doctor、plugin doctor、missed the module table、ERR_PNPM_UNEXPECTED_STORE、Failed to load plugins、插件台账、升级体检。
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
3. **浏览器实测（关键步骤）**：chrome-devtools 打开 `http://127.0.0.1:13080/?token=<token>` → 截图 + console
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

## 升级后必检清单（app / 引擎 / 插件任一升级后跑一遍）

- [ ] desktop.log 有 `Preinstall plugins installed successfully`（内部预设补装完成）
- [ ] 浏览器实测：截图正常 + console 零报错（不信健康检查）
- [ ] 手工垫片是否被插件升级覆盖（对照台账垫过的插件名单）
- [ ] 换机器/挪盘后 store 对齐（病1 检查法）
- [ ] 版本三层各自到哪了（`npm view ... next/latest` vs 本机）

## 红线与经验

- **杀 dsh 进程通常被拒（高权限）而且通常不需要**——client bundle 补丁即改即生效，刷新页面就行
- profile 的 `.npmrc` 必须钉 `store-dir`；全局挪 store 盘必炸 profile
- `@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app` 两条 `CORE_PLUGIN_PROFILE_ENTRY_MISSING` 是长期 WARN，不拦启动，别当病治
- `[pet-stream]` 2 秒重连刷屏是宠物插件常驻重连，与起不来无关
- 改任何文件前：备份到 `<profile>\.bak\`，方案带四字成语确认词

## 相关技能

- [deepseek-harness-settings-curator](https://github.com/huzhw/deepseek-harness-settings-curator)：DSH settings.yaml 模型配置梳理
- [agent-config-sync-check](https://github.com/huzhw/agent-config-sync-check)：四端同步守卫
