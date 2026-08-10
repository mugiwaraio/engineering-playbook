# Git 分支开发规范：main 只读镜像模式

## 核心原则

- 本地 `main` 仅用于同步 `origin/main`，不直接开发、不提交业务代码。
- 所有功能开发和 Bug 修复必须从最新 `main` 创建独立分支。
- 所有正式代码通过 GitHub Pull Request 合并到 `main`。
- PR 合并后同步本地 `main`，再开始下一项任务。

分支模型：

    origin/main
        │
        ▼
    local main
        │
        ├── feature/*
        └── fix/*


## 1. 开始新任务

先同步最新 `main`：

    git switch main
    git fetch --prune origin
    git pull --ff-only origin main

然后创建开发分支：

    git switch -c feature/new-feature

Bug 修复：

    git switch -c fix/bug-name

推荐命名：

    feature/lark-alert-controls
    feature/payer-permission-check
    fix/bedrock-timeout
    fix/login-race-condition


## 2. 开发并 Push

所有开发都在 Feature/Fix 分支进行：

    git add .
    git commit -m "feat: add new feature"
    git push -u origin feature/new-feature

后续提交：

    git add .
    git commit -m "fix: adjust behavior"
    git push

禁止：

- 直接在 `main` 开发。
- 直接向 `main` Push。
- 在本地 `main` 创建业务 Commit。


## 3. 通过 GitHub PR 合并

标准流程：

    Feature/Fix Branch
            │
            ▼
          Push
            │
            ▼
        GitHub PR
            │
            ▼
        CI / Review
            │
            ▼
          Merge
            │
            ▼
        origin/main

`main` 的正式变更统一通过 Pull Request 进入。


## 4. PR 合并后同步本地 main

GitHub PR 合并后：

    git switch main
    git fetch --prune origin
    git pull --ff-only origin main

检查状态：

    git status

正常情况下：

    On branch main
    Your branch is up to date with 'origin/main'.


## 5. 验证本地与远程一致

检查 Commit：

    git rev-parse main
    git rev-parse origin/main

两个 SHA 一致表示指向同一 Commit。

也可以检查内容差异：

    git diff main origin/main

没有输出表示内容一致。


## 6. 清理已合并分支

PR 合并后，可以删除本地开发分支：

    git branch -d feature/new-feature

远程分支已删除时，清理失效引用：

    git fetch --prune origin

优先使用 `-d`，不要默认使用 `-D`。


## 7. main 同步安全规则

日常同步统一使用：

    git switch main
    git fetch --prune origin
    git pull --ff-only origin main

使用 `--ff-only`，避免本地 `main` 意外产生 Merge Commit。

不要将以下命令作为日常同步方式：

    git reset --hard origin/main

该命令可能丢弃本地修改。

如果 `git pull --ff-only` 失败，说明本地 `main` 与 `origin/main` 已发生分叉。

此时不要自动 Merge、Rebase 或 Reset，应先检查：

    git status
    git log --oneline --graph --decorate --all -20
    git diff origin/main...main

确认分叉原因后再处理。


## 8. 标准开发循环

    # 1. 同步 main
    git switch main
    git fetch --prune origin
    git pull --ff-only origin main

    # 2. 创建分支
    git switch -c feature/xxx

    # 3. 开发并提交
    git add .
    git commit -m "feat: xxx"
    git push -u origin feature/xxx

    # 4. GitHub
    Feature Branch
        ↓
    Pull Request
        ↓
    CI / Review
        ↓
    Merge
        ↓
    origin/main

    # 5. PR 合并后同步 main
    git switch main
    git fetch --prune origin
    git pull --ff-only origin main

    # 6. 删除已完成分支
    git branch -d feature/xxx


## 9. 最终约定

    main      = origin/main 的本地只读镜像
    feature/* = 功能开发
    fix/*     = Bug 修复

统一遵循：

- `main` 只同步，不开发。
- 开发使用独立分支。
- 合并统一通过 PR。
- PR 合并后同步本地 `main`。
- 新任务始终基于最新 `main`。
- 使用 `--ff-only` 防止意外 Merge Commit。
- 不使用破坏性 Git 命令作为日常同步手段。
