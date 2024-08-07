---
layout: default
title: 版本控制（Git）
---

# Lec06：版本控制（Git）

本 Lec 的主要内容：

- 什么是版本控制系统？
- Git 的数据模型是什么样的？
- Git 的命令行接口有哪些？
- Git 的一些高级操作
- 学习 Git 的一些资源推荐

## 版本控制系统（VCSs）

### 什么是版本控制系统？

版本控制系统 (VCSs) 是一类用于**追踪源代码（或其他文件、文件夹）改动的工具**。顾名思义，这些工具可以帮助我们**管理代码的修改历史**；不仅如此，它还可以**让协作编码变得更方便**。VCS通过一系列的==快照==将某个文件夹及其内容保存了起来，每个快照都包含了文件或文件夹的完整状态。同时它还维护了快照==创建者的信息以及每个快照的相关信息==等等。

为什么说版本控制系统非常有用？即使你只是一个人进行编程工作，它也可以帮你**创建项目的快照**、**记录每个改动的目的**、**基于多分支并行开发**等等。和别人**协作开发**时，它更是一个无价之宝：你可以**看到别人对代码进行的修改**，同时**解决由于并行开发引起的冲突**。

现代的版本控制系统可以帮助你轻松地（甚至自动地）回答以下问题：

- 当前模块是谁编写的？
- 这个文件的这一行是什么时候被编辑的？是谁作出的修改？修改原因是什么呢？
- 最近的1000个版本中，何时 / 为什么导致了单元测试失败？

尽管版本控制系统有很多， 其事实上的标准则是 **Git** 。

> 因为 Git 接口的抽象泄漏（leaky abstraction）问题，通过自顶向下的方式（从命令行接口开始）学习 Git 可能会让人感到非常困惑。很多时候你只能死记硬背一些命令行，然后像使用魔法一样使用它们，一旦出现问题，你就不知道该怎么继续往下了。
>
> 尽管 Git 的接口有些丑陋，但是它的底层设计和思想却是非常优雅的。丑陋的接口只能靠死记硬背，而优雅的底层设计则非常容易被人理解。因此，我们将通过一种自底向上的方式向你介绍 Git。我们会从数据模型开始，最后再学习它的接口。一旦你搞懂了 Git 的数据模型，再学习其接口并理解这些接口是如何操作数据模型的就非常容易了。

## Git 的数据模型

进行版本控制的方法很多。Git 拥有一个经过精心设计的模型，这使其能够支持版本控制所需的所有特性，例如维护历史记录、支持分支和促进协作。

### 快照

Git 将==顶级目录==中的文件和文件夹作为集合，并通过一系列快照来管理其历史记录。

在 Git 的术语里，**文件被称作 Blob 对象（数据对象），也就是一组数据。目录则被称之为 “树”，它将==名字与 Blob 对象或树对象进行映射==（使得==目录中可以包含其他目录==）。快照则是被追踪的==最顶层的树==。**例如，一个树看起来可能是这样的：

```
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.txt (blob, contents = "hello world")
|
+- baz.txt (blob, contents = "git is wonderful")
```

这个顶层的树包含了两个元素，一个名为 “foo” 的树（它本身包含了一个 blob 对象 “bar.txt”），以及一个 blob 对象 “baz.txt”。

### 历史记录建模：关联快照

在 Git 中，历史记录是一个由快照组成的==有向无环图==。这代表 Git 中的每个快照都有一系列的 **“父辈”**，也就是其之前的一系列快照。==注意，快照具有多个 “父辈” 而非一个，因为某个快照可能由多个父辈而来==。例如，经过合并后的两条分支。

在 Git 中，这些快照被称为 “提交”（commit）。通过可视化的方式来表示这些历史提交记录时，看起来差不多是这样的：

```
o <-- o <-- o <-- o
            ^  
             \
              --- o <-- o
```

**箭头指向了当前提交的父辈**。在第三次提交之后，历史记录分岔成了两条独立的分支。这可能因为**此时需要同时开发两个不同的特性，它们之间是相互独立的**。开发完成后，这些分支可能会被合并，并且创建一个新的提交，这个新的提交会==同时包含==这些特性。新的提交会创建一个新的历史记录，看上去像这样（最新的合并提交用粗体标记）：

```
o <-- o <-- o <-- o <---- o
            ^            /
             \          v
              --- o <-- o
```

**Git 中的提交是不可改变的**。但这并不代表错误不能被修改，只不过这种 “修改” 实际上是创建了一个全新的提交记录。而引用（参见下文）则被更新为指向这些新的提交。

### Git 的数据模型及其伪码表示

以下是 Git 数据模型的伪码表示：

```
// 文件就是一组数据
type blob = array<byte>

// 目录就是包含目录或文件的映射表
type tree = map<string, tree | blob>

// 每个提交都包含一个或多个父辈，元数据（描述作者，提交信息等）和顶层树（即快照）
type commit = struct {
    parent: array<commit>
    author: string
    message: string
    snapshot: tree
}
```

这是一种简洁的历史模型。

### Git 数据模型中的对象的数据寻址

Git 中的对象可以是 blob、tree 或 commit：

```
type object = blob | tree | commit
```

Git 在储存数据时，所有的对象都会基于它们的 [SHA-1 哈希](https：//en.wikipedia.org/wiki/SHA-1) 进行寻址。

```
objects = map<string, object>

def store(object):
    id = sha1(object)
    objects[id] = object

def load(id):
    return objects[id]
```

Blobs、tree 和 commit 都一样，==它们都是对象==。当它们引用其他对象时，它们并没有真正的在硬盘上保存这些对象，而是==仅仅保存了它们的哈希值作为引用==。

例如，[上面](https：//missing-semester-cn.github.io/2020/version-control/#snapshots)例子中的树（可以通过 `git cat-file -p 698281bc680d1995c5f4caaf3359721a5a58d48d` 来进行可视化），看上去是这样的：

```
100644 blob 4448adbf7ecd394f42ae135bbeed9676e894af85    baz.txt
040000 tree c68d233a33c5c06e0340e4c224f0afca87c8ce87    foo
```

树本身会包含一些指向其他内容的指针，例如 `baz.txt` （blob）和 `foo` （tree）。如果我们用 `git cat-file -p 4448adbf7ecd394f42ae135bbeed9676e894af85`，即通过哈希值查看 baz.txt 的内容，会得到以下信息：

```
git is wonderful
```

### 引用

现在，所有的快照都可以通过它们的 SHA-1 哈希值来标记了。但这也太不方便了，谁也记不住一串 40 位的十六进制字符。

针对这一问题，Git 的解决方法是给这些哈希值**赋予人类可读的名字，也就是引用（references）**。==引用是指向提交的指针==。与对象不同的是，它是 *可变* 的（引用可以被更新，指向新的提交）。例如，`master` 引用通常会指向主分支的最新一次提交。

引用的数据模型如下：

```
// 前一个string是人类可读名字，后一个string是哈希值
references = map<string, string>

def update_reference(name, id):
    references[name] = id

def read_reference(name):
    return references[name]

def load_reference(name_or_id):
    if name_or_id in references:
        return load(references[name_or_id])
    else:
        return load(name_or_id)
        
// 单个引用是从人类可读名字到哈希值的映射
```

这样，Git 就可以使用诸如 “master” 这样人类可读的名称来表示历史记录中某个特定的提交，而不需要在使用一长串十六进制字符了。

有一个细节需要我们注意， 通常情况下，我们会想要知道 “我们当前所在位置”，并将其标记下来。这样当我们创建新的快照的时候，我们就可以知道它的相对位置（如何设置它的 “父辈” ）。**在 Git 中，我们当前的位置有一个特殊的索引，它就是 “HEAD”。**

### Git 的数据仓库

最后，我们可以粗略地给出 **Git 仓库的定义了：`对象` 和 `引用`。**

**在硬盘上，Git 仅存储对象和引用**：因为其数据模型仅包含这些东西。**所有的 `git` 命令都对应着对提交树的操作，例如增加对象，增加或删除引用。**

当你输入某个指令时，请思考一下这条命令是如何对底层的图数据结构进行操作的。另一方面，如果你希望修改提交树，例如 “丢弃未提交的修改“ 和 ”将 ‘master’ 引用指向提交 `5d83f9e` 时“，有什么命令可以完成该操作（针对这个具体问题，你可以使用 `git checkout master; git reset --hard 5d83f9e`）

### Git 暂存区

Git 中还包括一个和数据模型完全不相关的概念，但它确是创建提交的接口的一部分。

就上面介绍的快照系统来说，你也许会期望它的实现里包括一个 “创建快照” 的命令，该命令能够基于当前工作目录的当前状态创建一个全新的快照。有些版本控制系统确实是这样工作的，但 Git 不是。我们希望简洁的快照，而且每次从当前状态创建快照可能效果并不理想。例如，考虑如下场景，你开发了两个独立的特性，然后你希望创建两个独立的提交，其中第一个提交仅包含第一个特性，而第二个提交仅包含第二个特性。或者，假设你在调试代码时添加了很多打印语句，然后你仅仅希望提交和修复 bug 相关的代码而丢弃所有的打印语句。

Git 处理**这些场景**的方法是使用一种叫做 “暂存区（staging area）”的机制，它允许你指定下次快照中要包括那些改动。

## Git 的命令行接口

为了避免重复信息，我们将不会详细解释以下命令行。强烈推荐阅读 [Pro Git 中文版](https：//git-scm.com/book/zh/v2)或可以观看本讲座的视频来学习。

### 基础

- `git help <command>`： 获取 git 命令的帮助信息

- `git init`： 创建一个新的 git 仓库，其数据会存放在一个名为 `.git` 的目录下

- `git status`： 显示当前的仓库状态

- `git add <filename>`： 添加文件到暂存区

- `git commit`： 创建一个新的提交
  - 如何编写 [良好的提交信息](https：//tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)!
  - 为何要 [编写良好的提交信息](https：//chris.beams.io/posts/git-commit/)
  
- `git log`： 显示历史日志

- `git log --all --graph --decorate`： 可视化历史记录（有向无环图）

- `git diff <filename>`： 显示与暂存区文件的差异

- `git diff <revision> <filename>`： 显示某个文件两个版本之间的差异

- `git checkout <revision>`： 更新 HEAD 和目前的分支

### 分支和合并

- `git branch`：显示分支

- `git branch <name>`：创建分支

- `git checkout -b <name>`：创建分支并切换到该分支

  - 相当于 `git branch <name>; git checkout <name>`

- `git merge <revision>`：合并到当前分支

- `git mergetool`：使用工具来处理合并冲突

- `git rebase`：将一系列补丁变基（rebase）为新的基线

### 远端操作

- `git remote`：列出远端
- `git remote add <name> <url>`：添加一个远端
- `git push <remote> <local branch>:<remote branch>`：将对象传送至远端并更新远端引用
- `git branch --set-upstream-to=<remote>/<remote branch>`：创建本地和远端分支的关联关系
- `git fetch`：从远端获取对象/索引
- `git pull`：相当于 `git fetch; git merge`
- `git clone`：从远端下载仓库

### 撤销

- `git commit --amend`：编辑提交的内容或信息
- `git reset HEAD <file>`：恢复暂存的文件
- `git checkout -- <file>`：丢弃修改
- `git restore`：git 2.32 版本后取代 git reset 进行许多撤销操作

### Git 高级操作

- `git config`：Git 是一个 [高度可定制的](https://git-scm.com/docs/git-config) 工具
- `git clone --depth=1`：浅克隆（shallow clone），不包括完整的版本历史信息，只克隆最新的快照
- `git add -p`：交互式暂存
- `git rebase -i`：交互式变基
- `git blame`：查看最后修改某行的人
- `git stash`：暂时移除工作目录下的修改内容
- `git bisect`：通过二分查找搜索历史记录
- `.gitignore`：[指定](https://git-scm.com/docs/gitignore) Git 不要追踪的文件

## 杂项

- **图形用户界面**: Git 的 [图形用户界面客户端](https://git-scm.com/downloads/guis) 有很多，但是我们自己并不使用这些图形用户界面的客户端，我们选择使用命令行接口
- **Shell 集成**: 将 Git 状态集成到：你的 shell 中会非常方便。([zsh](https://github.com/olivierverdier/zsh-git-prompt), [bash](https://github.com/magicmonty/bash-git-prompt))。[Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh) 这样的框架中一般以及集成了这一功能
- **编辑器集成**: 和上面一条类似，将 Git 集成到编辑器中好处多多。[fugitive.vim](https://github.com/tpope/vim-fugitive) 是 Vim 中集成 GIt 的常用插件
- **工作流**: 我们已经讲解了数据模型与一些基础命令，但还没讨论到进行大型项目时的一些惯例 ( 有[很多](https://nvie.com/posts/a-successful-git-branching-model/) [不同的](https://www.endoflineblog.com/gitflow-considered-harmful) [处理方法](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow))
- **GitHub**: Git 并不等同于 GitHub。 在 GitHub 中：你需要使用一个被称作 [拉取请求（pull request）](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests)的方法来向其他项目贡献代码
- **其他 Git 提供商**: GitHub 并不是唯一的。还有像 [GitLab](https://about.gitlab.com/) 和 [BitBucket](https://bitbucket.org/) 这样的平台。

## 资源

- [Pro Git](https://git-scm.com/book/en/v2)，**强烈推荐**！学习前五章的内容可以教会：你流畅使用 Git 的绝大多数技巧，因为：你已经理解了 Git 的数据模型。后面的章节提供了很多有趣的高级主题。（[Pro Git 中文版](https://git-scm.com/book/zh/v2)）；
- [Oh Shit, Git!?!](https://ohshitgit.com/)，简短的介绍了如何从 Git 错误中恢复；
- [Git for Computer Scientists](https://eagain.net/articles/git-for-computer-scientists/)，简短的介绍了 Git 的数据模型，与本文相比包含较少量的伪代码以及大量的精美图片；
- [Git from the Bottom Up](https://jwiegley.github.io/git-from-the-bottom-up/) 详细的介绍了 Git 的实现细节，而不仅仅局限于数据模型。好奇的同学可以看看；
- [How to explain git in simple words](https://smusamashah.github.io/blog/2017/10/14/explain-git-in-simple-words)；
- [Learn Git Branching](https://learngitbranching.js.org/) 通过基于浏览器的游戏来学习 Git ；