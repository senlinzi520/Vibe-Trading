# Fork 二开工作流

> 本项目基于上游开源仓库做二次开发，核心诉求是：**既能随时拿到上游更新，又不被上游改动冲掉自己的代码。**
>
> | 角色 | 仓库 | 说明 |
> |---|---|---|
> | 上游（upstream） | `https://github.com/HKUDS/Vibe-Trading` | 只读，原作者持续更新 |
> | 自己的 Fork（origin） | `https://github.com/senlinzi520/Vibe-Trading` | 可写，二开成果推这里 |
> | 本地工作区 | `d:/WWW/Vibe-Trading` | Windows + Git Bash |

---

## 一、双 Remote 结构

本地配置了**两个远程仓库**，各司其职：

| Remote 名 | 指向 | 权限 | 用途 |
|---|---|---|---|
| `origin` | 自己的 Fork | **可读写** | 推送自己的二开成果 |
| `upstream` | 原作者仓库 | **只读** | 拉取上游更新 |

### 验证当前配置

```bash
git remote -v
```

**期望输出**（`origin` 和 `upstream` 各只出现一次）：

```
origin    git@github.com:senlinzi520/Vibe-Trading.git (fetch)
origin    git@github.com:senlinzi520/Vibe-Trading.git (push)
upstream  https://github.com/HKUDS/Vibe-Trading.git (fetch)
upstream  https://github.com/HKUDS/Vibe-Trading.git (push)
```

### `.git/config` 的标准内容

```ini
[core]
    repositoryformatversion = 0
    filemode = false
    bare = false
    logallrefupdates = true
    symlinks = false
    ignorecase = true
[remote "origin"]
    url = git@github.com:senlinzi520/Vibe-Trading.git
    fetch = +refs/heads/*:refs/remotes/origin/*
[remote "upstream"]
    url = https://github.com/HKUDS/Vibe-Trading.git
    fetch = +refs/heads/*:refs/remotes/upstream/*
[branch "main"]
    remote = upstream
    merge = refs/heads/main
    vscode-merge-base = upstream/main
```

> **注意**：`origin` 走 SSH（`git@...`），因为 HTTPS 方式在 Windows 上频繁遇到凭据问题。SSH key 已配置完成，用 `ssh -T git@github.com` 可验证（看到 `Hi senlinzi520!` 即成功）。

---

## 二、分支模型

只有两个分支，职责严格分离：

| 分支 | 角色 | 规则 |
|---|---|---|
| `main` | **上游镜像** | 只做一件事：跟随上游。**永远不在这里提交自己的代码** |
| `dev` | **二开分支** | 所有自己的开发、提交、推送都在这里进行 |

之所以这样设计：只要 `main` 保持纯净，上游更新时它就能**永远快进（fast-forward）**，不会产生冲突；然后把 `main` 合并进 `dev`，冲突只发生在 `dev` 上，可控且可回退。

> ⚠️ 一旦你往 `main` 提交了自己的代码，`git merge --ff-only` 会立刻失败，整套流程就失去了"自动跟随上游"的能力。

---

## 三、日常同步流程

每次想拿上游最新代码时，执行以下命令。

```bash
# ① 拉上游最新（含版本标签）
git fetch upstream --tags

# ② 让 main 镜像跟上上游，并同步到你自己的 Fork
git switch main
git merge --ff-only upstream/main
git push origin main

# ③ 回到二开分支，把上游更新合进来
git switch dev
git rebase main
```

**第 ④ 条（条件执行）**：如果你在第 ③ 步用了 `rebase`，且 `dev` 之前已经推送到远程，那么 rebase 会改写提交历史，需要强制更新远程分支：

```bash
git push --force-with-lease origin dev
```

> `--force-with-lease` 比 `--force` 安全：如果远程 `dev` 在你这期间被别人改过，它会**拒绝执行**，避免覆盖别人的工作。
> **铁律：只对 `dev` 用 force，永远不要对 `main` 用。**

### 各步骤说明

| 步骤 | 作用 | 关键点 |
|---|---|---|
| `git fetch upstream --tags` | 把上游的分支和标签下载到本地，**不动你的工作区** | 一定要带 `--tags`，否则上游打的版本标签拿不到 |
| `git merge --ff-only upstream/main` | 把 `main` 快进到上游最新 | `--ff-only` 保证只做快进，一旦失败说明 `main` 被污染过 |
| `git push origin main` | 把镜像状态同步给你的 Fork | 方便在 GitHub 上对比，非必需但推荐 |
| `git rebase main` | 把你的二开提交"搬"到上游最新代码之上 | 历史线性、干净，上游更新永远排在你的提交下面 |

### `rebase` 还是 `merge`？

| 场景 | 推荐 | 原因 |
|---|---|---|
| 你一个人二开 | `git rebase main` | 提交历史线性，不会堆一堆合并提交 |
| 有别人协作你的 `dev` | `git merge main` | 不改写已有提交，避免重写队友的历史 |

选了 `merge` 的话，第 ④ 条就不需要 `--force-with-lease`，正常 `git push origin dev` 即可。

---

## 四、冲突处理流程

同步上游时出现冲突是正常的（你和上游改了同一处代码）。处理步骤：

### 1. 定位冲突

```bash
git status
```

冲突文件会显示为 `both modified`。

### 2. 手动解决

打开这些文件，搜索冲突标记：

```
<<<<<<< HEAD
你自己的代码
=======
上游的代码
>>>>>>> upstream/main
```

决定保留哪边（或两边融合），然后**删掉这三行标记**。

### 3. 标记已解决并继续

```bash
git add <冲突文件>
```

- 如果你在 **rebase** 流程中：

```bash
git rebase --continue
```

- 如果你在 **merge** 流程中：

```bash
git commit
```

### 4. 想放弃整次操作（会完全回退，可放心使用）

```bash
git rebase --abort
```

或

```bash
git merge --abort
```

### 降低冲突概率的实践

- **尽量新增文件，而不是修改上游文件**（定制代码放到新模块/新目录）
- 必须改上游文件时，改动**小块、集中**，不要顺手重排格式或调整缩进
- 把"对上游文件的修改"做成**几个用途明确的独立 commit**，冲突时容易定位取舍

---

## 五、踩坑记录：`.git/config` 重复段

> 这是本项目实际踩过的一个大坑，导致 git 行为完全错乱、折腾很久才定位。记录下来避免重犯。

### 症状

在 Windows + Git Bash + VS Code 环境下反复执行 `git remote rename` / `add` / `set-url` 之后，出现了下面这些**互相矛盾**的现象：

| 命令 | 异常表现 |
|---|---|
| `git remote set-url origin <url>` | 报 `error: No such remote 'origin'`（说没有 origin） |
| `git remote -v` | 却正常列出 `origin`（而且可能**打印两遍**） |
| `git remote remove origin` | 删不干净，残留的段会"复活" |
| `git remote add origin <url>` | 明明已存在，**却不报 `already exists`**，静默追加新段 |
| `git push origin` | 用的是旧的 URL，一直推送失败 |

**症状特征一句话**：`git remote -v` 能列出某个 remote，但 `git remote set-url` 却说它不存在。

### 根因

`.git/config` 里出现了**同名的重复段**：

```ini
[remote "origin"]
    url = https://github.com/senlinzi520/Vibe-Trading      ← 第 1 段
    fetch = +refs/heads/*:refs/remotes/origin/*
[remote "origin"]
    url = git@github.com:senlinzi520/Vibe-Trading.git      ← 第 2 段（重复）
    fetch = +refs/heads/*:refs/remotes/origin/*
```

一个名字对应多份定义，导致 git 各子命令行为不一致：有的"后者覆盖前者"、有的遍历全部、有的匹配失败。

**雪上加霜**：`git remote remove origin` 在删除 remote 时，还会连带删掉 `[branch "main"]` 里引用该 remote 的 `remote` / `merge` 两行，使分支失去跟踪目标。

### 排查方法

**第一步：看 `git remote -v` 是否有重复行**

```bash
git remote -v
```

**第二步：直接看 config 原文**

```bash
cat .git/config
```

**第三步：查不可见字符（关键一步）**

```bash
cat -A .git/config
```

`-A` 会把所有不可见字符显示出来：

| 显示 | 含义 |
|---|---|
| `$` | 行尾（正常） |
| `$` 前面出现 `^M` | CRLF 换行（Windows 常见） |
| `^I` | 制表符（缩进，正常） |
| `M-oM-;M-?` | UTF-8 BOM（**异常**，可能干扰解析） |
| 名字中间有异常字符 | 零宽字符/尾随空格（**异常**） |

如果 `cat -A` 输出很干净（行尾都是 `$`、无 BOM、无零宽字符），那问题就是**纯粹的段重复**。

**第四步：列全部配置项，检查有无重复 key**

```bash
git config --local --list
```

同一项出现两次即为异常，例如：

```
remote.origin.url=https://github.com/senlinzi520/Vibe-Trading
remote.origin.url=git@github.com:senlinzi520/Vibe-Trading.git    ← 重复
```

### 解决方案：手工重写 `.git/config`

**此状态下 git 子命令已完全不可信，不要再用 `git remote remove/add/set-url` 去绕——每绕一次就多一份垃圾。唯一可靠的办法是直接改文件。**

```bash
# 1. 备份
cp .git/config .git/config.bak

# 2. 用编辑器打开 d:\WWW\Vibe-Trading\.git\config
#    全选删除，粘贴第一章里的「.git/config 的标准内容」，保存
```

**保存注意**：

- 编码必须是 **UTF-8（无 BOM）**
- 换行符保持 **LF**（不要改成 CRLF）
- 文件末尾留一个换行符

**改完验证**：

```bash
git remote -v
git config --local --list
git fetch upstream --tags
git push origin main
```

四项全通即修复完成。

想回退备份：

```bash
cp .git/config.bak .git/config
```

### 预防措施

1. **每次动完 remote，都执行一次 `git remote -v`** —— 发现某个名字出现两次，立即清理
2. **不要在 VS Code 的 Git 源码管理面板里操作 remote** —— IDE 会自己改写 config，是重复段的高概率来源
3. **改 config 尽量用命令行或纯文本编辑器**，改完别立刻点 IDE 的 Git 按钮

---

## 六、速查表

### 每日/每周同步上游

```bash
git fetch upstream --tags
git switch main
git merge --ff-only upstream/main
git push origin main
git switch dev
git rebase main
```

### 只想看上游改了哪些文件，不做合并

```bash
git fetch upstream
git diff --stat main upstream/main          # 看统计
git diff --name-status main upstream/main   # 看文件清单
```

### 只吸收上游的某几个提交 / 某个文件

```bash
git cherry-pick <commit-hash>                        # 取某个 commit
git checkout upstream/main -- path/to/file.py        # 取某个文件的上游版本
```

### 三条铁律

| # | 铁律 |
|---|---|
| 1 | **`main` 上永远不提交自己的代码** —— 它只是上游的镜子 |
| 2 | **`git push` 必须显式写 `git push origin main`** —— 因为 `branch.main` 的跟踪目标是 `upstream`（只读、无权限），不带参数的 `git push` 会去推上游然后失败 |
| 3 | **改完 remote 一定用 `git remote -v` 复查** —— 重复段这个坑就是这么来的 |

### 环境信息

| 项 | 值 |
|---|---|
| 操作系统 | Windows |
| Shell | Git Bash (MINGW64) / PowerShell |
| 远程认证 | SSH（`git@github.com:senlinzi520/Vibe-Trading.git`） |
| 上游 | `https://github.com/HKUDS/Vibe-Trading.git` |
