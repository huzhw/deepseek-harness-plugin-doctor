# Junction 说明

本仓库（`F:\idea-workspase-skills\deepseek-harness-plugin-doctor`）是唯一数据源，四个工具端通过 Windows Junction 链接消费，**四端看到的永远是同一份文件**。

## 四端指向关系

| 端 | Junction（链接） | 目标（真身） |
|---|---|---|
| Claude Code | `C:\Users\Administrator\.claude\skills\deepseek-harness-plugin-doctor` | `F:\idea-workspase-skills\deepseek-harness-plugin-doctor` |
| DSH | `C:\Users\Administrator\.dsh\skills\deepseek-harness-plugin-doctor` | 同上 |
| Codex | `C:\Users\Administrator\.codex\skills\deepseek-harness-plugin-doctor` | 同上 |
| ZCode | `C:\Users\Administrator\.zcode\skills\deepseek-harness-plugin-doctor` | 同上 |
| 全局路径（junction，Qoder） | `C:\Users\Administrator\.qoder\skills\deepseek-harness-plugin-doctor` |

## 双向同步说明

- **改文件只在 F 盘真身改**（或从任一端进入后沿 Junction 落到 F 盘，效果等同）
- 四端没有"各自副本"，不存在内容分叉；分叉只可能发生在 Junction 断链后某端落了实体文件
- 定期体检：`pwsh -NoProfile -File "F:\idea-workspase-skills\agent-config-sync-check\scripts\sync-check.ps1"`（修复加 `-Fix`）

## 检查 Junction 是否完好

```bat
dir "C:\Users\Administrator\.claude\skills" | findstr deepseek-harness-plugin-doctor
dir "C:\Users\Administrator\.dsh\skills" | findstr deepseek-harness-plugin-doctor
dir "C:\Users\Administrator\.codex\skills" | findstr deepseek-harness-plugin-doctor
dir "C:\Users\Administrator\.zcode\skills" | findstr deepseek-harness-plugin-doctor
cmd /c dir "C:\Users\Administrator\.qoder\skills" | findstr deepseek-harness-plugin-doctor
```

看到 `<JUNCTION>` 字样即完好；看到 `<DIR>` 说明变成了实体目录（断链后被落了文件），需人工处理。

## 回滚（删链接）警告

```bat
rd "C:\Users\Administrator\.claude\skills\deepseek-harness-plugin-doctor"
rd "C:\Users\Administrator\.dsh\skills\deepseek-harness-plugin-doctor"
rd "C:\Users\Administrator\.codex\skills\deepseek-harness-plugin-doctor"
rd "C:\Users\Administrator\.zcode\skills\deepseek-harness-plugin-doctor"
rd "C:\Users\Administrator\.qoder\skills\deepseek-harness-plugin-doctor"
```

**`rd` 后面绝对不能加 `/s`**：不加 `/s` 只删 Junction 链接本身，F 盘真身安然无恙；加 `/s` 会顺着链接把 F 盘真身内容一并删掉。
