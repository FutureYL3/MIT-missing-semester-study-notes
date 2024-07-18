---
layout: default
title: 命令行环境
---

# Lec05：命令行环境

本 Lec 主要学习：

- 如何同时执行多个不同的进程并追踪它们的状态
- 如何停止或暂停某个进程
- 如何使进程在后台运行
- 定义别名
- 如何使用 SSH 操作远端机器

## 任务控制

某些情况下我们需要中断正在执行的任务，比如当一个命令需要执行很长时间才能完成时（假设我们在使用 `find` 搜索一个非常大的目录结构）。大多数情况下，我们可以使用 `Ctrl-C` 来停止命令的执行。但是**它的工作原理是什么呢？为什么有的时候会无法结束进程？**

### 结束进程

你的 shell 会使用 *UNIX 提供的信号机制* 执行==进程间通信==。当一个进程接收到信号时，它会

1. 停止执行、处理该信号
2. 基于信号传递的信息来改变其执行。

就这一点而言，信号是一种*软件中断*。

在上面提到的例子中，当我们输入 `Ctrl-C` 时，shell 会发送一个`SIGINT` （Signal Interrupt）信号到进程。

程序可以捕捉到这些信号，并自定义处理方式，比如下面这个 Python 程序，它捕获了信号 `SIGINT` 并忽略它，这导致你按 `Ctrl-C` 时，这个程序不会停止。为了停止这些程序，需要使用 `SIGINT` 以外的终端信号，比如 `SIGQUIT`，可以通过 `Ctrl-\` 发送该信号。

```python
#!/usr/bin/env python
import signal, time

def handler(signum, time):
    print("\nI got a SIGINT, but I am not stopping")

signal.signal(signal.SIGINT, handler)
i = 0
while True:
    time.sleep(.1)
    print("\r{}".format(i), end="")
    i += 1
```

当向这个程序发送两次 `SIGINT`，然后再发送一次 `SIGQUIT`，以下是结果：

```shell
$ python sigint.py
24^C
I got a SIGINT, but I am not stopping
26^C
I got a SIGINT, but I am not stopping
30^\[1]    39913 quit       python sigint.pyƒ
```

尽管 `SIGINT` 和 `SIGQUIT` 都常常用来发出和终止程序相关的请求。`SIGTERM` 则是一个更加通用的、也更加优雅地退出信号。为了发出这个信号我们需要使用 [`kill`](https://www.man7.org/linux/man-pages/man1/kill.1.html) 命令, 它的语法是： `kill -TERM <PID>`。

> `PID` 是进程号，可以通过 
>
> 1. `ps aux | grep <pattern>` 查看符合指定模式的进程的进程号
> 2. `pgrep <pattern>` 查看符合模式的进程的进程号，与上面类似
>    - `-l`：显示进程 ID 和进程名称
>    - `-u user`：仅显示指定用户的进程
>    - `-f`：匹配整个命令行，而不仅仅是进程名
>    - `-x`：要求完全匹配
>    - `-v`：反转匹配

### 暂停和后台执行进程

信号可以让进程做其他的事情，而不仅仅是终止它们。例如，`SIGSTOP` 会让进程暂停。在终端中，键入 `Ctrl-Z` 会让 shell 发送 `SIGTSTP` 信号，`SIGTSTP`是 Terminal Stop 的缩写（即 `terminal` 版本的 SIGSTOP）。

> `SIGSTOP` 和 `SIGTSTP` 的比较：
>
> `SIGSTOP`：
>
> - 强制暂停进程
> - 无法被捕获、阻塞或忽略
> - 用于系统级别的进程控制
>
> `SIGTSTP`：
>
> - 用户请求暂停进程
> - 可以被捕获或忽略
> - 用于交互式终端中的进程控制

我们可以使用 [`fg`](https://www.man7.org/linux/man-pages/man1/fg.1p.html) 或 [`bg`](http://man7.org/linux/man-pages/man1/bg.1p.html) 命令恢复暂停的工作。它们分别表示**在前台继续或在后台继续**。

[`jobs`](http://man7.org/linux/man-pages/man1/jobs.1p.html) 命令会列出==当前终端会话中尚未完成==的全部任务。你可以使用 pid 引用这些任务（也可以用 [`pgrep`](https://www.man7.org/linux/man-pages/man1/pgrep.1.html) 找出 pid）。更加符合直觉的操作是你可以使用百分号 + 任务编号（`jobs` 会打印任务编号在第一列）来选取该任务。如果要选择最近的一个任务，可以使用 `$!` 这一特殊参数。

还有一件事情需要掌握，那就是命令中的 `&` 后缀可以让命令在**直接在后台运行**，这使得你可以直接在 shell 中继续做其他操作，**不过它此时还是会使用 shell 的标准输出（即它的输出内容还是会打印到终端中），这一点有时会比较恼人（这种情况可以使用 shell 重定向处理）。**

**让已经在运行的进程转到后台运行**，你可以先 `Ctrl-Z` ，然后再 `bg`。**注意，后台的进程仍然是你的终端进程的子进程，一旦你关闭终端（会发送另外一个信号 `SIGHUP`），这些后台的进程也会终止**。为了防止这种情况发生，你可以使用 [`nohup`](https://www.man7.org/linux/man-pages/man1/nohup.1.html) (一个用来忽略 `SIGHUP` 的封装) 来运行程序。针对已经运行的程序，可以使用`disown` 。除此之外，你可以使用**终端多路复用器**来实现，下一章节我们会进行详细地探讨。

下面这个简单的会话中展示了这些概念的应用：

```shell
$ sleep 1000
^Z
[1]  + 18653 suspended  sleep 1000

$ nohup sleep 2000 &
[2] 18745
appending output to nohup.out

$ jobs
[1]  + suspended  sleep 1000
[2]  - running    nohup sleep 2000

$ bg %1
[1]  - 18653 continued  sleep 1000

$ jobs
[1]  - running    sleep 1000
[2]  + running    nohup sleep 2000

$ kill -STOP %1
[1]  + 18653 suspended (signal)  sleep 1000

$ jobs
[1]  + suspended (signal)  sleep 1000
[2]  - running    nohup sleep 2000

$ kill -SIGHUP %1
[1]  + 18653 hangup     sleep 1000

$ jobs
[2]  + running    nohup sleep 2000

$ kill -SIGHUP %2

$ jobs
[2]  + running    nohup sleep 2000

$ kill %2
[2]  + 18745 terminated  nohup sleep 2000

$ jobs
```

`SIGKILL` 是一个特殊的信号，它**不能被进程捕获并且它会马上结束该进程**。不过这样做会有一些**副作用**，例如留下**孤儿进程**。

你可以在 [这里](https://en.wikipedia.org/wiki/Signal_(IPC)) 或输入 [`man signal`](https://www.man7.org/linux/man-pages/man7/signal.7.html) 或使用 `kill -l` 来获取更多关于信号的信息。

## 终端多路复用

当你在使用命令行时，你通常会希望**同时执行多个任务**。举例来说，你可以想要**同时运行你的编辑器，并在终端的另外一侧执行程序**。尽管再打开一个新的终端窗口也能达到目的，使用==终端多路复用器==则是一种更好的办法。

像 [`tmux`](https://www.man7.org/linux/man-pages/man1/tmux.1.html) 这类的终端多路复用器可以允许我们**基于面板和标签**分割出多个终端窗口，这样你便可以同时与**多个 shell 会话**进行交互。

不仅如此，终端多路复用使我们可以**分离当前终端会话并在将来重新连接**（在 detach 的时候不会终止 tmux 会话内的进程，并可以在将来重新连接至会话，会话的状态会被保留）。

这让你**操作远端设备时**的工作流大大改善，避免了 `nohup` 和其他类似技巧的使用。

现在最流行的终端多路器是 [`tmux`](https://www.man7.org/linux/man-pages/man1/tmux.1.html)。`tmux` 是一个高度可定制的工具，你可以使用相关快捷键创建多个标签页并在它们间导航。

`tmux` 中对象的继承结构如下：

- **会话** - 每个会话都是一个独立的工作区，其中包含一个或多个窗口
  - `tmux` 开始一个新的会话
  - `tmux new -s NAME` 开始一个新的会话并指定其名称
  - `tmux ls` 列出当前所有会话
  - `<prefix> d` 从当前会话分离
  - `tmux a` 重新连接最后一个会话。可以通过 `-t NAME` 来指定具体的会话
- **窗口** - 相当于编辑器或是浏览器中的 *标签页*，从视觉上将一个会话分隔为多个部分
  - `<prefix> c` 创建一个新的窗口，使用 `<C-d>` 关闭
  - `<prefix> N` 跳转到第 N 个窗口，注意每个窗口都是有编号的
  - `<prefix> p` 切换到前一个窗口
  - `<prefix> n` 切换到后一个窗口
  - `<prefix> ,` 重命名当前窗口
  - `<prefix> w` 列出当前所有窗口
- **面板** - 像 vim 中的分屏一样，面板使我们可以在一个屏幕里显示多个 shell
  - `<prefix> "` 水平分割，已改为 `<prefix> -`
  - `<prefix> %` 垂直分割，已改为 `<prefix> ｜`
  - `<prefix> <arrow-keys>` 切换到指定方面的面板，使用方向键
  - `<prefix> z` 切换当前面板的缩放
  - `<prefix> [` 开始往回卷动屏幕。你可以按下空格键来开始选择，回车键复制选中的部分
  - `<prefix> <space>` 在不同的面板排布间切换

> 扩展阅读： [这里](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/) 是一份 `tmux` 快速入门教程， [而这一篇](http://linuxcommand.org/lc3_adv_termmux.php) 文章则更加详细，它包含了 `screen` 命令。你也许想要掌握 [`screen`](https://www.man7.org/linux/man-pages/man1/screen.1.html) 命令，因为在大多数 UNIX 系统中都默认安装有该程序。

## 别名

输入一长串包含许多选项的命令会非常麻烦。因此，大多数 shell 都支持设置别名。**shell 的别名相当于一个长命令的缩写，shell 会自动将其替换成原本的命令**。例如，bash 中的别名语法如下：

```bash
alias alias_name="command_to_alias arg1 arg2"
```

> 注意， `=` 两边是没有空格的，因为 [`alias`](https://www.man7.org/linux/man-pages/man1/alias.1p.html) 是一个 shell 命令，它只接受一个参数。

别名有许多很方便的特性：

- 常用命令的缩写
- 打错命令下的纠正
- 重新定义一些命令的默认行为
- 别名可以嵌套（别名内可以包含其他别名）
- 忽略或禁用别名
- 获取别名的定义（使用 `alias NAME`）

```bash
# 创建常用命令的缩写
alias ll="ls -lh"

# 能够少输入很多
alias gs="git status"
alias gc="git commit"
alias v="vim"

# 手误打错命令也没关系
alias sl=ls

# 重新定义一些命令行的默认行为
alias mv="mv -i"           # -i prompts before overwrite
alias mkdir="mkdir -p"     # -p make parent dirs as needed
alias df="df -h"           # -h prints human readable format

# 别名可以组合使用
alias la="ls -A"
alias lla="la -l"

# 在忽略某个别名
\ls
# 或者禁用别名
unalias la

# 获取别名的定义
alias ll
# 会打印 ll='ls -lh'
```

值得注意的是，在默认情况下 shell 并不会保存别名。为了让别名持续生效，你需要将配置放进 shell 的启动文件里，像是 `.bashrc` 或 `.zshrc`，下一节我们就会讲到。

## 配置文件（Dotfiles）

很多**程序的配置**都是通过纯文本格式的被称作 *点文件* 的配置文件来完成的（之所以称为点文件，是因为它们的文件名以 `.` 开头，例如 `~/.vimrc`。也正因为此，它们**默认是隐藏文件**，`ls` 并不会显示它们）。

shell 的配置也是通过这类文件完成的。在启动时，你的 shell 程序会读取很多文件以加载其配置项。根据 shell 本身的不同，你从登录开始还是以交互的方式完成这一过程可能会有很大的不同。关于这一话题，[这里](https://blog.flowblok.id.au/2013-02/shell-startup-scripts.html) 有非常好的资源。

对于 `bash` 来说，在大多数系统下，你可以通过编辑 `.bashrc` 或 `.bash_profile` 来进行配置。在文件中你可以添加**需要在启动时执行的命令**，例如上文我们讲到过的**别名**，或者是你的**环境变量**。

> 实际上，很多程序都要求你在 shell 的配置文件中包含一行类似 `export PATH="$PATH:/path/to/program/bin"` 的命令，这样才能确保这些程序能够被 shell 找到。

以下是一些可以通过点文件配置的常用工具：

- `bash` - `~/.bashrc` 和 `~/.bash_profile`
- `git` - `~/.gitconfig`
- `vim` - `~/.vimrc` 和 `~/.vim` 目录
- `ssh` - `~/.ssh/config`
- `tmux` - `~/.tmux.conf`

我们应该如何管理这些配置文件呢？它们应该**在它们的文件夹**下，并使用**版本控制系统**进行管理，然后通过脚本将其 **符号链接** 到需要的地方。这么做有如下好处：

- **安装简单**: 如果你登录了一台新的设备，在这台设备上应用你的配置只需要几分钟的时间
- **可移植性**: 你的工具在任何地方都以相同的配置工作
- **同步**: 在一处更新配置文件，可以同步到其他所有地方
- **变更追踪**: 你可能要在整个程序员生涯中持续维护这些配置文件，而对于长期项目而言，版本历史是非常重要的

配置文件中需要放些什么？你可以通过在线文档和[帮助手册](https://en.wikipedia.org/wiki/Man_page)了解所使用工具的设置项。另一个方法是在网上搜索有关特定程序的文章，作者们在文章中会分享他们的配置。还有一种方法就是直接浏览其他人的配置文件：你可以在这里找到无数的[dotfiles 仓库](https://github.com/search?o=desc&q=dotfiles&s=stars&type=Repositories) —— 其中最受欢迎的那些可以在[这里](https://github.com/mathiasbynens/dotfiles)找到（我们建议你不要直接复制别人的配置）。[这里](https://dotfiles.github.io/) 也有一些非常有用的资源。

本课程的老师们也在 GitHub 上开源了他们的配置文件： [Anish](https://github.com/anishathalye/dotfiles), [Jon](https://github.com/jonhoo/configs), [Jose](https://github.com/jjgo/dotfiles).

### 可移植性

配置文件的**一个常见的痛点是它可能并不能在多种设备上生效**。例如，如果你在不同设备上使用的操作系统或者 shell 是不同的，则配置文件是无法生效的。或者，有时你仅希望特定的配置只在某些设备上生效。

有一些技巧可以轻松达成这些目的。使用 if 语句，则你可以借助它针对不同的设备编写不同的配置。例如，你的 shell 可以这样做：

```bash
if [[ "$(uname)" == "Linux" ]]; then {do_something}; fi

# 使用和 shell 相关的配置时先检查当前 shell 类型
if [[ "$SHELL" == "zsh" ]]; then {do_something}; fi

# 你也可以针对特定的设备进行配置
if [[ "$(hostname)" == "myServer" ]]; then {do_something}; fi
```

如果配置文件支持 include 功能，你也可以多加利用。例如：`~/.gitconfig` 可以这样编写：

```
[include]
    path = ~/.gitconfig_local
```

然后我们可以**在日常使用的设备上创建配置文件** `~/.gitconfig_local` 来**包含与该设备相关的特定配置**。你甚至应该创建一个单独的代码仓库来管理这些与设备相关的配置。

如果你希望**在不同的程序之间共享某些配置**，该方法也适用。例如，如果你想要在 `bash` 和 `zsh` 中同时启用一些别名，你可以把它们写在 `.aliases` 里，然后在这两个 shell 里应用：

```bash
# Test if ~/.aliases exists and source it
if [ -f ~/.aliases ]; then
    source ~/.aliases
fi
```

## 远端设备

对于程序员来说，在他们的日常工作中使用远程服务器已经非常普遍了。如果你需要使用远程服务器来部署后端软件或你需要一些计算能力强大的服务器，你就会用到安全 shell（SSH）。和其他工具一样，SSH 也是可以高度定制的，也值得我们花时间学习它。

### 连接远端设备

通过如下命令，你可以使用 `ssh` 连接到其他服务器：

```bash
ssh foo@bar.mit.edu
```

这里我们尝试

- 以用户名 `foo` 
- 登录服务器 `bar.mit.edu`。服务器可以通过 URL 指定（例如`bar.mit.edu`），也可以使用 IP 指定（例如`foobar@192.168.1.42`）。

后面我们会介绍如何修改 ssh 配置文件使我们可以用类似 `ssh bar` 这样的命令来登录服务器。

### 执行命令

`ssh` 的一个经常被忽视的特性是**它可以直接远程执行命令**。 

`ssh foobar@server ls` 可以直接以 foobar 用户的身份执行 `ls` 命令。 

想要配合管道来使用也可以， `ssh foobar@server ls | grep PATTERN` 会在本地查询远端 `ls` 的输出而 `ls | ssh foobar@server grep PATTERN` 会在远端对本地 `ls` 输出的结果进行查询。

### SSH 密钥

基于密钥的验证机制使用了密码学中的**公钥**，我们只需要**向服务器证明客户端持有对应的私钥，而不需要公开其私钥**。这样你就可以**避免每次登录都输入密码**的麻烦了。不过，私钥（通常是 `~/.ssh/id_rsa` 或者 `~/.ssh/id_ed25519`），其等效于你的密码，所以一定要好好保存它。

#### 生成密钥

使用 [`ssh-keygen`](http://man7.org/linux/man-pages/man1/ssh-keygen.1.html) 命令可以生成一对密钥（公钥和私钥）：

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

> 命令解释：
>
> - `ssh-keygen`：用于生成、管理和转换认证密钥的命令行工具。
>
> - `-o`：使用 OpenSSH 的新格式输出私钥文件。该格式比旧的 PEM 格式更安全，支持更强的加密。从 OpenSSH 6.5 开始默认开启。
>
> - `-a 100`：指定 KDF (Key Derivation Function) 的迭代次数。KDF 将密码转换为密钥，通过增加计算复杂度来提高私钥的安全性。`-a 100` 表示使用 100 次迭代。更多的迭代次数会增加加密强度，但也会使生成密钥的过程变得更慢。
>
> - `-t ed25519`：指定生成的密钥类型。ed25519 是一种用于数字签名的椭圆曲线算法，相比于 RSA，更安全且密钥长度更短。
> - `-f ~/.ssh/id_ed25519`：指定输出文件的路径和名称。生成的私钥将保存到 `~/.ssh/id_ed25519`，对应的公钥将保存到 `~/.ssh/id_ed25519.pub`。

你可以为密钥设置密码，防止有人持有你的私钥并使用它访问你的服务器。你可以使用 [`ssh-agent`](https://www.man7.org/linux/man-pages/man1/ssh-agent.1.html) 或 [`gpg-agent`](https://linux.die.net/man/1/gpg-agent) ，这样就不需要每次都输入该密码了。

如果你曾经配置过使用 SSH 密钥推送到 GitHub，那么可能你已经完成了[这里](https://help.github.com/articles/connecting-to-github-with-ssh/) 介绍的这些步骤，并且已经有了一个可用的密钥对。要检查你是否持有密码并验证它，你可以运行 `ssh-keygen -y -f /path/to/key`.

#### 基于密钥的认证机制

在服务器端，`ssh` 会查询 `.ssh/authorized_keys` 来确认哪些用户可以被允许登录。你可以通过下面的命令将一个**公钥**拷贝到这里：

```bash
cat .ssh/id_ed25519.pub | ssh foobar@remote 'cat >> ~/.ssh/authorized_keys'
```

如果支持 `ssh-copy-id` 的话，可以使用下面这种更简单的解决方案：

```bash
ssh-copy-id -i .ssh/id_ed25519.pub foobar@remote
```

### 通过 SSH 复制文件

使用 ssh 复制文件有很多方法：

- `ssh + tee`：最简单的方法是执行 `ssh` 命令，然后通过这样的方法利用标准输入实现： `cat localfile | ssh remote_server tee serverfile`。回忆一下，[`tee`](https://www.man7.org/linux/man-pages/man1/tee.1.html) 命令会将标准输出写入到一个文件；
- [`scp`](https://www.man7.org/linux/man-pages/man1/scp.1.html) ：当**需要拷贝大量的文件或目录时**，使用 `scp` 命令则更加方便，它可以方便地遍历相关路径。语法如下：`scp path/to/local_file remote_host:path/to/remote_file`；
- [`rsync`](https://www.man7.org/linux/man-pages/man1/rsync.1.html) 对 `scp` 进行了改进，它可以**检测本地和远端的文件以防止重复拷贝**。它还可以提供一些诸如符号连接、权限管理等精心打磨的功能。甚至还可以基于 `--partial` 标记实现**断点续传**。`rsync` 的语法和`scp`类似。

### 端口转发

很多情况下我们都会遇到软件需要监听特定设备的端口。如果是在你的本机，可以使用 `localhost:PORT` 或 `127.0.0.1:PORT`。但是如果需要监听远程服务器的端口该如何操作呢？这种情况下 *远端的端口并不会直接通过网络暴露给你*。

此时就需要进行 *端口转发*。**端口转发有两种，一种是本地端口转发和远程端口转发**（参见下图，该图片引用自这篇[StackOverflow 文章](https://unix.stackexchange.com/questions/115897/whats-ssh-port-forwarding-and-whats-the-difference-between-ssh-local-and-remot)）中的图片。

**本地端口转发**![Local Port Forwarding](https://i.stack.imgur.com/a28N8.png%C2%A0)

**远程端口转发**![Remote Port Forwarding](https://i.stack.imgur.com/4iK3b.png%C2%A0)

常见的情景是使用 *本地端口转发*，即**远端设备上的服务监听一个端口，而你希望在本地设备上的一个端口建立连接并转发到远程端口上**。

例如，我们在远端服务器上运行 Jupyter notebook 并监听 `8888` 端口。 然后，建立从本地端口 `9999` 的转发，使用 `ssh -L 9999:localhost:8888 foobar@remote_server` 。这样只需要访问本地的 `localhost:9999` 即可。

> Tips：
>
> 在使用 ssh 命令时，`-N` 和 `-f` 是两个常用的选项，通常用于创建端口转发或隧道，而不执行远程命令。以下是对这两个选项的详细解释：
>
> `-N` 选项告诉 ssh 只进行端口转发或隧道，而不执行远程命令。这在你只需要创建一个 SSH 隧道或端口转发，而不需要在远程服务器上运行任何命令时非常有用。
>
> `-f` 选项告诉 ssh 在执行命令之前先进入后台。这在你希望 SSH 会话在后台运行时非常有用，通常与 `-N` 选项一起使用来创建后台运行的端口转发。
>
> 当这两个选项结合使用时，**SSH 会在后台运行并只进行端口转发或隧道，而不执行任何命令**。

### SSH 配置

我们已经介绍了很多参数。为它们创建一个别名是个好想法，我们可以这样做：

```bash
alias my_server="ssh -i ~/.id_ed25519 --port 2222 -L 9999:localhost:8888 foobar@remote_server
```

不过，更好的方法是使用 `~/.ssh/config`：

```
Host vm
    User foobar
    HostName 172.16.174.141
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    LocalForward 9999 localhost:8888

# 在配置文件中也可以使用通配符
Host *.mit.edu
    User foobaz
```

这么做的好处是，使用 `~/.ssh/config` 文件来创建别名，类似 `scp`、`rsync`和`mosh`的这些命令都可以读取这个配置并将设置转换为对应的命令行选项。

> 注意，`~/.ssh/config` 文件也可以被当作配置文件，而且一般情况下也是可以被导入其他配置文件的。不过，如果你将其公开到互联网上，那么其他人都将会看到你的服务器地址、用户名、开放端口等等。这些信息可能会帮助到那些企图攻击你系统的黑客，所以请务必三思。

服务器侧的配置通常放在 `/etc/ssh/sshd_config`。你可以在这里配置免密认证、修改 ssh 端口、开启 X11 转发等等。 你也可以为每个用户单独指定配置。

### 杂项

连接远程服务器的一个常见痛点是遇到由关机、休眠或网络环境变化导致的掉线。如果连接的延迟很高也很让人讨厌。[Mosh](https://mosh.org/)（即 mobile shell ）对 ssh 进行了改进，它允许**连接漫游、间歇连接及智能本地回显**。

有时将一个远端文件夹挂载到本地会比较方便， [sshfs](https://github.com/libfuse/sshfs) 可以**将远端服务器上的一个文件夹挂载到本地**，然后你就可以**使用本地的编辑器**了。

## Shell & 框架

在 shell 工具和脚本那节课中我们已经介绍了 `bash` shell，因为它是目前**最通用**的 shell，大多数的系统都将其作为默认 shell。但是，它并不是唯一的选项。

例如，`zsh` shell 是 `bash` 的超集并提供了一些方便的功能：

- 智能替换，`**`
- 行内替换 / 通配符扩展
- 拼写纠错
- 更好的 tab 补全和选择
- 路径展开 （`cd /u/lo/b` 会被展开为 `/usr/local/bin`）

**框架** 也可以改进你的 shell。比较流行的通用框架包括 [prezto](https://github.com/sorin-ionescu/prezto) 或 [oh-my-zsh](https://ohmyz.sh/)。还有一些更精简的框架，它们往往专注于某一个特定功能，例如 [zsh 语法高亮](https://github.com/zsh-users/zsh-syntax-highlighting) 或 [zsh 历史子串查询](https://github.com/zsh-users/zsh-history-substring-search)。 像 [fish](https://fishshell.com/) 这样的 shell 包含了很多用户友好的功能，其中一些特性包括：

- 向右对齐
- 命令语法高亮
- 历史子串查询
- 基于手册页面的选项补全
- 更智能的自动补全
- 提示符主题

需要注意的是，使用这些框架**可能会降低你 shell 的性能**，尤其是如果这些框架的代码没有优化或者代码过多。你随时可以测试其性能或禁用某些不常用的功能来实现速度与功能的平衡。

## 终端模拟器

和自定义 shell 一样，花点时间选择适合你的 **终端模拟器** 并进行设置是很有必要的。有许多终端模拟器可供你选择（这里有一些关于它们之间[比较](https://anarc.at/blog/2018-04-12-terminal-emulators-1/)的信息）

你会花上很多时间在使用终端上，因此研究一下终端的设置是很有必要的，你可以从下面这些方面来配置你的终端：

- 字体选择
- 彩色主题
- 快捷键
- 标签页/面板支持
- 回退配置
- 性能（像 [Alacritty](https://github.com/jwilm/alacritty) 或者 [kitty](https://sw.kovidgoyal.net/kitty/) 这种比较新的终端，它们支持GPU加速）。
