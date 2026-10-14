## 生成ssh密钥

1. `rm -rf ~/.ssh` 删除原有文件
2. `ssh-keygen -t ed25519 -C "[your_email@example.com](<mailto:your_email@example.com>)"` ，询问设置均采用默认
3. 查看密钥内容：`cat ~/.ssh/id_ed25519.pub`
4. 把整个内容复制到github设置里的ssh密钥中

## 仓库远程地址

```bash
git remote -v # 查看
git remote set-url origin <new_url> # 设置
```

## 上传

```bash
git add .
git reset HEAD . # 撤回 add
git commit -m "your_commit"
git push
```

## 分支

```bash
git branch # 查看所有本地分支
git switch <分支名> # 切换到<分支名>分支
git switch -c <分支名> # 创建并切换到<分支名>分支
git clone -b <分支名> <仓库地址> # 拉取指定分支
```

## 删除commit

删除最新的一次commit并保留修改：`git reset --soft HEAD~1`

不保留修改：`git reset --hard HEAD~1`

删除历史中某一次提交，并修改历史记录，代码不会保留：

1. `git log` 找出要删除的 commit 的**前一个** commit 的哈希值 `<commit _hash>`
2. 将 `commit_id` 替换为目标前一个版本的哈希值 `git rebase -i <commit_hash>`
3. 在弹出的编辑器中，找到想删除的那一行，将最前面的 `pick` 改为 `drop` ，保存并退出
4. `git push origin <branch_name> --force` 强制推送

如果是在团队协作的分支中，强制推送可能会覆盖其他人的代码。

不修改历史记录使用 `git revert <commit_hash>`

## 指定用户

`global` 表示全局指定，去掉为仅指定当前仓库

```bash
git config --global user.name "USERNAME"
git config --global user.email "mail@example.com"
git config user.name # 查看配置
git config user.email
```

## 提交规范

[约定式提交](https://www.conventionalcommits.org/zh-hans/v1.0.0/)

格式：`<类型>[可选 范围]: <描述>`

提交说明包含了下面的结构化元素，以向类库使用者表明其意图：

1. **fix:** _类型_ 为 `fix` 的提交表示在代码库中修复了一个 bug（这和语义化版本中的 [**`PATCH`**](https://semver.org/lang/zh-CN/#%E6%91%98%E8%A6%81) 相对应）。
2. **feat:** _类型_ 为 `feat` 的提交表示在代码库中新增了一个功能（这和语义化版本中的 [**`MINOR`**](https://semver.org/lang/zh-CN/#%E6%91%98%E8%A6%81) 相对应）。
3. **BREAKING CHANGE:** 在脚注中包含 `BREAKING CHANGE:` 或 <类型>(范围) 后面有一个 `!` 的提交，表示引入了破坏性 API 变更（这和语义化版本中的 [**`MAJOR`**](https://semver.org/lang/zh-CN/#%E6%91%98%E8%A6%81) 相对应）。 破坏性变更可以是任意 _类型_ 提交的一部分。
4. 除 `fix:` 和 `feat:` 之外，也可以使用其它提交 _类型_ ，例如 [**@commitlint/config-conventional**](https://github.com/conventional-changelog/commitlint/tree/master/%40commitlint/config-conventional)（基于 [**Angular 约定**](https://github.com/angular/angular/blob/22b96b9/CONTRIBUTING.md#-commit-message-guidelines)）中推荐的 `build:`、`chore:`、 `ci:`、`docs:`、`style:`、`refactor:`、`perf:`、`test:`，等等。
    - build: 用于修改项目构建系统，例如修改依赖库、外部接口或者升级 Node 版本等；
    - chore: 用于对非业务性代码进行修改，例如修改构建流程或者工具配置等；
    - ci: 用于修改持续集成流程，例如修改 Travis、Jenkins 等工作流配置；
    - docs: 用于修改文档，例如修改 README 文件、API 文档等；
    - style: 用于修改代码的样式，例如调整缩进、空格、空行等；
    - refactor: 用于重构代码，例如修改代码结构、变量名、函数名等但不修改功能逻辑；
    - perf: 用于优化性能，例如提升代码的性能、减少内存占用等；
    - test: 用于修改测试用例，例如添加、删除、修改代码的测试用例等。
5. 脚注中除了 `BREAKING CHANGE: <description>` ，其它条目应该采用类似 [**git trailer format**](https://git-scm.com/docs/git-interpret-trailers) 这样的惯例。

其它提交类型在约定式提交规范中并没有强制限制，并且在语义化版本中没有隐式影响（除非它们包含 BREAKING CHANGE）。 可以为提交类型添加一个围在圆括号内的范围，以为其提供额外的上下文信息。例如 `feat(parser): adds ability to parse arrays.`。

## fork仓库后同步commit更新

### merge

直接以该commit版本覆盖仓库

```bash
git merge <commit-hash>
git add .
git push
```

### 挑选指定commit版本加入代码

```bash
git remote add upstream <上游仓库URL>
git fetch upstream
git cherry-pick <commit-hash> # 添加commit修改的代码到自己fork的仓库，如果没冲突就添加成功
```

### 解决冲突

产生冲突后，在本地fork的仓库里打开提示冲突的文件

git发现上游（upstream）的那个 commit 修改了某一行，而本地分支的同一行也刚好被修改了，git 无法自动决定留哪一个，于是就在**你本地的那个文件**里写入了冲突标记例如 `<<<<<<< HEAD` 和 `>>>>>>>`

```bash
git add <修改文件名>
git cherry-pick --continue
```

#### 对fork版本中已删除的文件进行修改

```bash
git rm <file> # 选择删除文件
git add <file> # 选择添加文件
git cherry-pick --continue
```

## `.gitignore`

纯文本文件，告诉 git 哪些文件不要 push，例如不要 push `.vscode` 目录里的配置文件

```
### VisualStudioCode ###
.vscode/*
!.vscode/tasks.json
!.vscode/launch.json
!.vscode/extensions.json
!.vscode/*.code-snippets
```