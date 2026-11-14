# Git 管理规范

Git 用于配合知识库整理，负责整理前备份和整理后归档。

## 整理前备份

```
git add -A
git commit -m "整理前备份_YYYYMMDD"
git push
```

## 整理后归档

```
git add -A
git commit -m "整理后归档_YYYYMMDD"
git push
```

## 执行要求

- 仅在 Inbox 整理流程中调用。
- 整理前后各执行一次 Git 三件套。
- Commit 日期使用 `YYYYMMDD`。
- Commit 信息使用中文。
- 无变更时不创建空提交。
- 推送失败时停止后续操作，并向用户说明。

Git 只负责版本备份和归档，不参与笔记分类、命名和内容分析。
