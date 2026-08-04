---
title: GitLab 协作完整操作指南（从拉取到合并）
cssclasses:
  - archive-page
---

# GitLab 协作完整操作指南（从拉取到合并）

这份笔记用于梳理基于 GitLab 的团队协作流程，覆盖从同步远程主分支、创建个人开发分支、提交代码、处理冲突，到发起 Merge Request 并完成合并后的清理工作。

> [!summary] 核心主线
> 永远先同步主分支，再从最新主分支创建个人分支；开发完成后先更新个人分支，再推送并发起 MR，由 GitLab 完成代码评审和合并。

## 一、整体流程

> [!info] GitLab 协作流程总览
> | 阶段 | 本地操作 | GitLab 操作 | 目的 |
> |---|---|---|---|
> | 1 | `git checkout main`、`git pull` | 无 | 让本地主分支保持最新 |
> | 2 | `git checkout -b feature/xxx` | 无 | 创建个人开发分支 |
> | 3 | 修改代码、`git add`、`git commit` | 无 | 保存本地开发成果 |
> | 4 | `git fetch`、`git rebase origin/main` | 无 | 推送前同步最新主分支，提前解决冲突 |
> | 5 | `git push -u origin feature/xxx` | 无 | 把个人分支推到远程仓库 |
> | 6 | 无 | 创建 MR | 请求合并到 `main` |
> | 7 | 根据评审继续提交 | Review、CI、Approve | 修正问题并通过检查 |
> | 8 | 无 | Merge MR | 合并进入主分支 |
> | 9 | 删除本地和远程分支 | 可删除源分支 | 清理已完成分支 |

标准顺序是：

1. 切到主分支。
2. 拉取远程最新代码。
3. 基于最新主分支创建个人分支。
4. 在个人分支上开发。
5. 提交本地 commit。
6. 推送前再次同步主分支。
7. 解决可能出现的冲突。
8. 推送个人分支到 GitLab。
9. 在 GitLab 上发起 Merge Request。
10. 等待 CI、代码评审和审批。
11. 合并 MR。
12. 清理本地和远程分支。

## 二、前置准备

### 检查 Git 配置

首次参与项目协作前，先确认 Git 用户名和邮箱已经配置正确。

查看配置：

`git config --global user.name`

`git config --global user.email`

设置配置：

`git config --global user.name "你的名字"`

`git config --global user.email "你的邮箱"`

### 克隆项目

如果本地还没有仓库，先从 GitLab 克隆项目。

`git clone <项目URL>`

进入项目目录：

`cd <项目目录>`

### 确认远程仓库

查看远程地址：

`git remote -v`

正常情况下会看到 `origin` 指向 GitLab 仓库地址。

> [!note] 主分支名称
> 本文默认主分支是 `main`。如果你的项目使用 `master`，把所有命令里的 `main` 替换成 `master` 即可。

## 三、标准开发流程

### 第一步：同步主分支

开始开发前，不要直接在旧代码上开分支。先切回主分支并拉取最新代码。

`git checkout main`

`git pull origin main`

如果本地还没有 `main` 分支，可以先获取远程分支：

`git fetch origin`

再切换到远程主分支对应的本地分支：

`git checkout -b main origin/main`

### 第二步：创建个人开发分支

从最新 `main` 创建自己的开发分支。

`git checkout -b feature/login-page`

常见分支类型：

> [!info] 分支命名速查
> | 类型 | 命名示例 | 适用场景 |
> |---|---|---|
> | 功能开发 | `feature/login-page` | 新增页面、接口、模块 |
> | 问题修复 | `fix/login-error` | 修复普通 bug |
> | 紧急修复 | `hotfix/payment-crash` | 线上紧急问题 |
> | 重构 | `refactor/user-service` | 不改变功能的结构调整 |
> | 文档 | `docs/gitlab-guide` | 文档或注释更新 |
> | 测试 | `test/order-api` | 补充或调整测试 |

### 第三步：本地开发与提交

开发过程中可以随时查看文件变化。

`git status`

查看具体改动：

`git diff`

把需要提交的文件加入暂存区：

`git add <文件路径>`

如果确认所有修改都要提交：

`git add .`

提交代码：

`git commit -m "feat: add login page"`

> [!tip] 提交建议
> 一次 commit 尽量只表达一个完整意图。不要把“新增功能、格式化、调试日志、无关文件修改”全部混在一个提交里。

### 第四步：推送前同步最新主分支

本地开发期间，其他同事可能已经把代码合并进 `main`。推送前建议先同步一次远程主分支，提前发现冲突。

先获取远程更新：

`git fetch origin`

推荐用 rebase 把自己的提交接到最新 `main` 后面：

`git rebase origin/main`

如果过程中没有冲突，分支历史会更线性，MR 也更容易阅读。

> [!warning] 不要在共享分支随便 rebase
> `rebase` 会改写提交历史。个人开发分支通常可以使用；多人共同推送的共享分支要先和团队确认。

### 第五步：解决 rebase 冲突

如果出现冲突，Git 会提示哪些文件需要处理。

查看冲突文件：

`git status`

打开冲突文件后，会看到类似内容：

```text
<<<<<<< HEAD
主分支上的内容
=======
你当前分支的内容
>>>>>>> feature/login-page
```

处理方法：

1. 判断最终应该保留哪部分代码。
2. 删除 `<<<<<<<`、`=======`、`>>>>>>>` 这些冲突标记。
3. 保存文件。
4. 把解决后的文件加入暂存区。
5. 继续 rebase。

加入暂存区：

`git add <冲突文件>`

继续 rebase：

`git rebase --continue`

如果还有下一处冲突，重复上面的步骤。

如果想放弃本次 rebase：

`git rebase --abort`

> [!danger] 冲突标记不能提交
> 提交前一定检查文件里没有 `<<<<<<<`、`=======`、`>>>>>>>`。这些标记一旦进入主分支，通常会直接导致编译失败或逻辑异常。

### 第六步：推送个人分支

首次推送个人分支到远程：

`git push -u origin feature/login-page`

后续同一分支继续推送：

`git push`

如果你已经推送过分支，又在本地做了 rebase，普通 `git push` 可能会被拒绝。这时使用：

`git push --force-with-lease`

> [!warning] force-with-lease 使用边界
> `--force-with-lease` 比 `--force` 更安全，但仍然会改写远程分支历史。只建议用于自己的个人分支，不要用于 `main` 或团队共享分支。

### 第七步：创建 Merge Request

推送成功后，在 GitLab 项目页面创建 MR。

通常需要填写：

> [!info] MR 信息速查
> | 项目 | 建议填写内容 |
> |---|---|
> | Source branch | 你的个人分支，例如 `feature/login-page` |
> | Target branch | 主分支，例如 `main` |
> | Title | 简短说明本次改动，例如 `feat: add login page` |
> | Description | 写清楚改了什么、为什么改、如何验证 |
> | Assignee | 当前负责人，一般是自己 |
> | Reviewer | 请求评审的人 |
> | Labels | 可选，例如 `feature`、`bugfix`、`docs` |
> | Delete source branch | 合并后可勾选，自动删除远程个人分支 |
> | Squash commits | 小提交较多时可勾选，合并时压缩为一个提交 |

MR 描述可以按这个结构写：

```markdown
## 改动内容
- 新增登录页面
- 接入登录接口
- 补充错误提示

## 验证方式
- 本地运行通过
- 登录成功、失败、空输入场景已验证

## 影响范围
- 登录页面
- 用户认证流程
```

### 第八步：处理评审意见和 CI

MR 创建后，通常会经历：

1. GitLab CI 自动检查。
2. Reviewer 查看代码并留言。
3. 开发者根据意见修改代码。
4. 本地提交新的 commit。
5. 再次推送到同一个分支。
6. MR 自动更新。

修改后继续提交：

`git add <文件路径>`

`git commit -m "fix: handle empty password"`

`git push`

如果 CI 失败，先点进失败 Job 查看日志，修复后再推送。

### 第九步：合并 MR

当 MR 满足以下条件后，可以合并：

> [!success] 合并前检查
> | 检查项 | 状态 |
> |---|---|
> | CI 通过 | 必须通过 |
> | 代码评审通过 | 至少满足项目规则 |
> | 冲突已解决 | MR 页面不能显示 conflict |
> | 目标分支正确 | 确认是 `main` 或项目要求的分支 |
> | 无临时调试代码 | 删除 `console.log`、临时注释、测试数据 |
> | MR 描述完整 | 说明改动、验证和影响范围 |

合并方式由项目规则决定，常见有三种：

> [!example] 合并方式对比
> | 合并方式 | 特点 | 适合场景 |
> |---|---|---|
> | Merge commit | 保留分支合并节点 | 需要完整保留分支历史 |
> | Squash and merge | 多个提交压成一个 | 小提交较多，希望主分支清爽 |
> | Rebase and merge | 线性历史 | 团队要求主分支历史严格线性 |

## 四、冲突处理全攻略

### 冲突为什么出现

冲突通常发生在两个人改了同一个文件的同一段内容，Git 无法自动判断最终应该保留哪一版。

常见来源：

> [!warning] 冲突来源速查
> | 场景 | 原因 | 处理建议 |
> |---|---|---|
> | 同一行代码被多人修改 | Git 无法自动合并 | 读懂双方意图后手动合并 |
> | 文件被一方删除、一方修改 | 文件状态冲突 | 确认业务上是否还需要该文件 |
> | 大范围格式化 | 改动覆盖面积太大 | 避免和功能改动混在一个 MR |
> | 自动生成文件变化 | 工具生成结果不一致 | 重新生成或按项目规则取舍 |
> | 锁文件变化 | 依赖版本不同 | 确认依赖后重新安装生成 |

### 冲突处理流程

1. 先不要慌，执行 `git status` 看冲突文件列表。
2. 逐个打开冲突文件。
3. 理解 `HEAD` 和当前分支两边分别代表什么。
4. 手动整理成最终想要的内容。
5. 删除冲突标记。
6. 执行必要的测试或编译。
7. `git add` 标记冲突已解决。
8. 继续 `rebase` 或完成 `merge`。

### merge 冲突和 rebase 冲突的区别

> [!info] 冲突命令对照
> | 场景 | 解决后继续 | 放弃操作 |
> |---|---|---|
> | `git merge origin/main` | `git commit` | `git merge --abort` |
> | `git rebase origin/main` | `git rebase --continue` | `git rebase --abort` |
> | `git cherry-pick <commit>` | `git cherry-pick --continue` | `git cherry-pick --abort` |

## 五、分支命名与提交规范

### 分支命名

推荐格式：

`类型/简短说明`

示例：

`feature/user-profile`

`fix/order-price-error`

`docs/gitlab-workflow`

> [!tip] 命名建议
> 分支名用英文小写、短横线分隔，尽量表达业务含义。不要使用 `test`、`new`、`mybranch` 这类无法追踪目的的名字。

### Commit Message

推荐格式：

`type: subject`

常见类型：

> [!info] commit 类型速查
> | 类型 | 含义 | 示例 |
> |---|---|---|
> | `feat` | 新功能 | `feat: add login page` |
> | `fix` | 修复问题 | `fix: handle token expired` |
> | `docs` | 文档修改 | `docs: update gitlab guide` |
> | `style` | 代码格式调整 | `style: format user service` |
> | `refactor` | 重构 | `refactor: simplify order flow` |
> | `test` | 测试相关 | `test: add login api tests` |
> | `chore` | 构建、依赖、杂项 | `chore: update dependencies` |
>
> subject 使用英文时一般小写开头，不需要句号。

## 六、常见场景

### 临时保存当前工作

如果正在开发一半，需要切分支处理其他事情，可以使用 stash。

保存当前修改：

`git stash push -m "login page half done"`

查看 stash：

`git stash list`

恢复最近一次 stash：

`git stash pop`

只应用不删除 stash：

`git stash apply`

### 紧急修复 Hotfix

线上问题通常要求从最新主分支快速创建 hotfix 分支。

1. 切回主分支。
2. 拉取最新代码。
3. 创建 `hotfix/xxx` 分支。
4. 修复问题并提交。
5. 推送分支。
6. 创建 MR。
7. 优先评审、测试和合并。

对应命令：

`git checkout main`

`git pull origin main`

`git checkout -b hotfix/payment-crash`

`git add .`

`git commit -m "fix: handle payment crash"`

`git push -u origin hotfix/payment-crash`

### 多个提交合并成一个

如果个人分支上有很多零碎提交，可以在 MR 勾选 `Squash commits`，让 GitLab 合并时压缩成一个提交。

也可以在本地交互式整理提交，但这需要熟悉 Git 历史改写。团队新手优先使用 GitLab MR 页面里的 Squash 选项。

> [!warning] Squash 注意事项
> Squash 会改变最终进入主分支的提交形态。已经用于排查问题的重要提交，不一定适合压缩。

### 撤销还没提交的修改

只撤销某个文件的工作区修改：

`git restore <文件路径>`

撤销所有未提交修改：

`git restore .`

> [!danger] restore 会丢弃本地修改
> 执行前确认这些改动不再需要。对于不确定的改动，优先用 `git stash` 暂存起来。

### 修改最近一次提交

如果刚提交完发现漏了一个文件，可以追加到最近一次提交。

`git add <漏掉的文件>`

`git commit --amend`

如果这个提交已经推送到远程个人分支，需要再执行：

`git push --force-with-lease`

### 查看历史和定位问题

查看简洁提交历史：

`git log --oneline --graph --decorate --all`

查看某个文件是谁改的：

`git blame <文件路径>`

查看某个提交的内容：

`git show <commit-id>`

## 七、合并后清理工作

MR 合并后，本地也要同步主分支并清理旧分支。

切回主分支：

`git checkout main`

拉取最新主分支：

`git pull origin main`

删除本地个人分支：

`git branch -d feature/login-page`

如果分支已经合并但 Git 无法识别，可以强制删除本地分支：

`git branch -D feature/login-page`

删除远程个人分支：

`git push origin --delete feature/login-page`

清理远程已删除分支的本地引用：

`git fetch --prune`

> [!tip] GitLab 自动删除源分支
> 创建 MR 时可以勾选 `Delete source branch when merge request is accepted`。这样 MR 合并后，GitLab 会自动删除远程个人分支。

## 八、速查命令表

> [!info] 日常高频命令
> | 目的 | 命令 |
> |---|---|
> | 查看当前状态 | `git status` |
> | 查看远程地址 | `git remote -v` |
> | 切到主分支 | `git checkout main` |
> | 拉取主分支 | `git pull origin main` |
> | 创建并切换分支 | `git checkout -b feature/xxx` |
> | 查看修改 | `git diff` |
> | 添加指定文件 | `git add <文件路径>` |
> | 添加全部修改 | `git add .` |
> | 提交代码 | `git commit -m "feat: xxx"` |
> | 获取远程更新 | `git fetch origin` |
> | 基于最新主分支变基 | `git rebase origin/main` |
> | 首次推送分支 | `git push -u origin feature/xxx` |
> | 后续推送 | `git push` |
> | rebase 后推送 | `git push --force-with-lease` |
> | 删除本地分支 | `git branch -d feature/xxx` |
> | 删除远程分支 | `git push origin --delete feature/xxx` |

> [!info] 排错命令
> | 问题 | 命令 | 说明 |
> |---|---|---|
> | 不知道改了什么 | `git status` | 先看文件状态 |
> | 想看具体差异 | `git diff` | 查看未暂存改动 |
> | rebase 出错想放弃 | `git rebase --abort` | 回到 rebase 前状态 |
> | merge 出错想放弃 | `git merge --abort` | 回到 merge 前状态 |
> | 忘记当前分支 | `git branch --show-current` | 显示当前分支名 |
> | 远程分支列表太乱 | `git fetch --prune` | 清理已删除远程分支引用 |

## 九、常见问题

> [!faq] 为什么不能直接在 main 上开发？
> `main` 是团队共享主线，直接在上面开发容易把未完成代码、临时代码或错误提交带进主分支。个人分支可以让开发、评审和回滚都更清晰。

> [!faq] `git pull` 和 `git fetch` 有什么区别？
> `git fetch` 只把远程更新取回来，不自动合并；`git pull` 等于先 fetch 再 merge 或 rebase。协作中需要更可控时，常用 `git fetch` 加 `git rebase origin/main`。

> [!faq] MR 里显示有冲突怎么办？
> 回到本地个人分支，执行 `git fetch origin` 和 `git rebase origin/main`，按冲突处理流程解决后，再推送个人分支。MR 会自动更新。

> [!faq] CI 失败能不能合并？
> 通常不能。CI 失败说明构建、测试、检查或部署流程至少有一个环节没有通过，应先查看 Job 日志并修复。

> [!faq] 什么时候用 `--force-with-lease`？
> 只有当你在个人分支做过 rebase 或 amend，导致远程历史和本地历史不一致时才使用。不要在主分支或多人共享分支上使用。

## 最后记住

> [!quote] 一句话总结
> GitLab 协作的关键不是“把代码推上去”，而是让每次改动都从最新主分支出发，在个人分支完成、同步、验证、评审之后，再干净地合并回主分支。
