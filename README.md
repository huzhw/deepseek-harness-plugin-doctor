# deepseek-harness-plugin-doctor — DSH 插件与升级体检医生

> **适用平台：DeepSeek Harness（DSH）** —— 覆盖 DSH 消费端（`~/.dsh/skills`），同时兼容挂载到 Claude Code / Codex / ZCode 技能目录。

dsh 起不来、插件加载崩溃、升级后犯病的一站式诊断修复：三大已知病速查（store 漂移 / externals 漂移 / 插件缺失）、插件台账、引擎升级后必检清单、浏览器实测验收。

## 相关技能

- [agent-config-sync-check](https://github.com/huzhw/agent-config-sync-check)：四端同步守卫：链接/硬链接/README 同步检查与修复
- [git-commit](https://github.com/huzhw/git-commit-skill)：Git 提交规范
- [daily-record-gitlab-md](https://github.com/huzhw/daily-record-gitlab-md-skill)：日报记录
- [daily-merge-gitlab-excel](https://github.com/huzhw/daily-merge-gitlab-excel-skill)：日报合并
- [code-check](https://github.com/huzhw/code-check-skill)：增量代码隐患检查
- [deepseek-harness-settings-curator](https://github.com/huzhw/deepseek-harness-settings-curator)：DSH 模型配置梳理
- [reread-rules](https://github.com/huzhw/reread-rules-skill)：重载 CLAUDE.md / AGENTS.md 规则
- [coding-rules](https://github.com/huzhw/coding-rules)：编码规则库（独立仓库，非 skill）
- [service-manager](https://github.com/huzhw/service-manager)：服务管理器（关联仓库，非 skill）
- [daily-report-panel](https://github.com/huzhw/daily-report-panel)：日报管家（关联仓库，非 skill，自动合并/导出/发件）

---

## 解决了什么问题

DSH 的故障面大部分落在「插件 + 升级」交叉带上，而且同一天的更新波能连踩两坑（2026-09-12 实录）：

- 引擎自动升级后，pnpm store 位置对不上，插件重装失败，**整个 dsh 起不来**
- 插件作者日更的新版本打包配置漂移，前端 bundle 引用了运行时没有的 Node 内置模块，**页面卡死在 Failed to load plugins**
- 健康检查显示 74/74 通过，**看着健康实际 UI 早就崩了**——假阳性骗人
- 修过一次的坑（profile 忘钉 store-dir、垫片被升级覆盖）隔几天换个 profile 复发

这个技能把诊断证据链、三大病速查、垫片配方、验收标准固化下来，照方抓药不再临场翻车。

## 使用

触发词（任选）：`dsh起不来`、`插件修复`、`dsh体检`、`dsh检查`、`dsh升级`、`引擎升级`、`dsh doctor`、`plugin doctor`、`missed the module table`、`ERR_PNPM_UNEXPECTED_STORE`、`插件台账`。

核心口诀：**健康检查会说谎，浏览器不会**——验收只认浏览器截图 + console。

## 文件结构

```
deepseek-harness-plugin-doctor/
├── SKILL.md                 ← 技能定义（三层版本/诊断流程/三大病速查/升级清单）
├── README.md                ← 本文档
├── JUNCTION说明.md          ← Junction 挂接说明
├── .gitignore
└── references/
    └── 台账.md              ← 插件清单 + 修复案例 + 待办观察
```

## 安装

四端已在消费端目录建好 Junction（本仓库为唯一数据源）：

```bash
C:\Users\Administrator\.claude\skills\deepseek-harness-plugin-doctor
C:\Users\Administrator\.dsh\skills\deepseek-harness-plugin-doctor
C:\Users\Administrator\.codex\skills\deepseek-harness-plugin-doctor
C:\Users\Administrator\.zcode\skills\deepseek-harness-plugin-doctor
```

## 许可

MIT
