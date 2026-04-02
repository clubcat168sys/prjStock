# CLAUDE.md — prjStock Workspace

此為 workspace 容器，實際開發在子模組 `prjTWStock/`。

## 結構

```
prjStock/
└── prjTWStock/    ← 台股分析引擎，獨立 git repo
    ├── CLAUDE.md  ← 完整開發 context 在這
    ├── .claude/   ← 權限設定、指令
    ├── claude/    ← 專案記憶與知識
    ├── src/       ← Python 原始碼（twstock）
    └── template/  ← job / SQL 模板
```

## 使用方式

在 `prjTWStock/` 目錄下啟動 Claude Code，才能載入正確的 context。
