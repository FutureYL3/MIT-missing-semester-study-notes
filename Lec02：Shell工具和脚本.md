---
layout: default
title: Shell 工具和脚本
---

# Lec02：Shell 工具和脚本

本 Lec 的主要内容：

- 了解如何编写 shell 脚本
- shell 的变量、字符串、函数、运算符、流程控制语句、通配符
- 两个特性：命令替换和进程替换
- 如何在 shell 中调用其他语言编写的脚本
- 一系列常用的 shell 工具（命令）

## Shell 脚本

大多数 shell 都有自己的一套脚本语言，包括变量、控制流和自己的语法。

shell脚本与其他脚本语言不同之处在于，shell 脚本针对 shell 所从事的相关工作进行了优化。因此，创建命令流程（pipelines）、将结果保存到文件、从标准输入中读取输入，这些都是 shell 脚本中的原生操作，这让它比通用的脚本语言更易用。本节中，我们会专注于 bash 脚本，因为它最流行，应用更为广泛。

### 变量赋值

在bash中为变量赋值的语法是 `foo=bar`，访问变量中存储的数值，其语法为 `$foo`。

> 注意⚠️：`foo = bar` （使用空格隔开）是不能正确工作的，因为解释器会调用程序`foo` 并将 `=` 和 `bar`作为参数。 总的来说，==在shell脚本中使用空格会起到分割参数的作用==，有时候可能会造成混淆，请务必多加检查。

### 字符串

Bash中的字符串通过 `'` 和 `"` 分隔符来定义，但是它们的含义并不相同。

以 `'` 定义的字符串为==原义字符串，其中的变量不会被转义==，而 `"` 定义的字符串==会将变量值进行替换==。

```shell
foo=bar
echo "$foo"
# 打印 bar
echo '$foo'
# 打印 $foo
```

### 函数

和其他大多数的编程语言一样，`bash` 也支持 `if`, `case`, `while` 和 `for` 这些控制流关键字。同样地， `bash` 也支持函数，它可以接受参数并基于参数进行操作。

下面这个函数是一个例子，它会创建一个文件夹并使用 `cd` 进入该文件夹。

```shell
mcd () {
    mkdir -p "$1"
    cd "$1"
}
```

> 这里 `$1` 是脚本的第一个参数。与其他脚本语言不同的是，bash使用了很多特殊的变量来表示参数、错误代码和相关变量。
>
> 下面列举了其中一些变量，更完整的列表可以参考 [这里](https://www.tldp.org/LDP/abs/html/special-chars.html)。
>
> | 参数名      | 含义                                                         |
> | ----------- | ------------------------------------------------------------ |
> | `$0`        | 脚本的名称                                                   |
> | `$1` ~ `$9` | 脚本的第 1 到第 9 个参数                                     |
> | `$@`        | 脚本的所有参数                                               |
> | `$#`        | 脚本参数的个数                                               |
> | `$?`        | 前一个命令的返回值                                           |
> | `$$`        | 当前脚本的进程识别码                                         |
> | `!!`        | 完整的上一条命令，包括参数。常见应用：当你因为权限不足执行命令失败时，可以使用 `sudo !!`再尝试一次。 |
> | `$_`        | 上一条命令的最后一个参数。如果你正在使用的是交互式 shell，你可以通过按下 `Esc` 之后键入 . 来获取这个值。 |
>
> 命令通常使用 `STDOUT` 来返回输出值，使用 `STDERR` 来返回错误及错误码，便于脚本以更加友好的方式报告错误。
>
> ==返回码或退出状态是脚本/命令之间交流执行状态的方式==。返回值 0 表示正常执行，其他所有非 0 的返回值都表示有错误发生。

### 逻辑运算符

退出码可以搭配 `&&`（与操作符）和 `||`（或操作符）使用，用来进行条件判断，决定是否执行其他程序。它们都属于短路[运算符](https://en.wikipedia.org/wiki/Short-circuit_evaluation)（short-circuiting）。同一行的多个命令可以用 ` ; ` 分隔。程序 `true` 的返回码永远是`0`，`false` 的返回码永远是 `1`。让我们看几个例子

```shell
false || echo "Oops, fail"
# Oops, fail

true || echo "Will not be printed"
#

true && echo "Things went well"
# Things went well

false && echo "Will not be printed"
#

false ; echo "This will always run"
# This will always run
```

> 小技巧：`||` 运算符常用于设置第一个程序非正常执行时要执行的操作；`&&` 运算符常用于设置第一个程序正常执行时要执行的操作。

### 命令替换

当通过 `$( CMD )` 这样的方式来执行 `CMD` 这个命令时，它的输出结果会替换掉 `$( CMD )` 。例如，如果执行 `for file in $(ls)` ，shell 首先将调用 `ls` ，然后遍历得到的这些返回值。

这个操作常用于需要使用某个命令的输出来做一些操作的情况。

### 进程替换

 `<( CMD )` 会执行 `CMD` 并将结果输出到一个临时文件中，并将 `<( CMD )` 替换成临时文件名。这在我们希望输出值通过文件而不是 `STDIN` 传递时很有用。例如， `diff <(ls foo) <(ls bar)` 会显示文件夹 `foo` 和 `bar` 中文件的区别。

### 一个综合前面知识的例子

```shell
#!/bin/sh # 告诉shell这段代码要用系统位于/bin/sh的程序来执行

echo "Starting program at $(date)" # 命令替换，date会被替换成日期和时间

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    # 如果模式没有找到，则grep退出状态为 1
    # 我们将标准输出流和标准错误流重定向到Null，因为我们并不关心这些信息
    # 解释：
    # 	grep foobar "$file"：这部分命令在变量 $file 指定的文件中搜索字符串 foobar。
	#	> /dev/null：将标准输出（stdout）重定向到 /dev/null，即丢弃输出内容。
	#	2> /dev/null：将标准错误（stderr）重定向到 /dev/null，即丢弃错误信息。
	# 目的：
    # 这条命令的目的是执行 grep 搜索而不在终端显示任何输出或错误信息。
    if [[ $? -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
# 遍历我们提供的参数，使用grep 搜索字符串 foobar，如果没有找到，则将其作为注释追加到文件中。
```

> - `[]` 是 `test` 命令的简写形式，用于条件测试，是 `if` 和 `while` 语句中最常见的用法。
>
> - 你可以直接使用 `test` 命令来代替 `[]`。
>
> - 在 `Bash` 和 `Zsh` 中，`[[ ]]` 提供了更强大的条件测试功能，处理字符串更加安全，并且支持更多的操作符。
>
> - 在脚本中，使用 `[]`、`test` 或 `[[ ]]` 取决于你的具体需求和 `Shell` 环境。对于简单的条件测试，`[]` 足够；对于更复杂的条件测试，特别是在 `Bash` 和 `Zsh` 中，建议使用 `[[ ]]`。
> - 在 `Shell` 脚本中，使用 `[]` 时，==前后的空格是必须的==，因为它们是 `test` 命令的简写形式，参数需要用空格分隔。同样，在使用 `[[ ]]` 进行条件测试时，==空格也是必须的==。这是确保条件表达式被正确解析和执行的必要语法要求。
> - 在条件语句中，我们比较 `$?` 是否等于0。具体的比较操作，可以查看 `test` 命令的手册。

### 通配

当执行脚本时，我们经常需要提供形式类似的参数。bash 使我们可以轻松的实现这一操作，它可以基于文件扩展名展开表达式。这一技术被称为 shell 的 *通配*（*globbing*）。

| 通配符 | 说明                     | 举例                                                    |
| ------ | ------------------------ | ------------------------------------------------------- |
| `?`    | 匹配一个字符             | `foo?`<br />匹配 `foo1`, `foox`, `foo(`....             |
| `*`    | 匹配任意个字符           | `foo*`<br />匹配 `foo123`, `foo28y`...                  |
| `{}`   | 生成字符串               | 见下面灰色字体                                          |
| `**`   | 匹配目录和子目录中的文件 | `ls **`<br />这将列出当前目录及其所有子目录中的所有文件 |

> ```shell
> convert image.{png,jpg}
> # 会展开为
> convert image.png image.jpg
> 
> cp /path/to/project/{foo,bar,baz}.sh /newpath
> # 会展开为
> cp /path/to/project/foo.sh /path/to/project/bar.sh /path/to/project/baz.sh /newpath
> 
> # 也可以结合通配使用
> mv *{.py,.sh} folder
> # 会移动所有 *.py 和 *.sh 文件
> 
> mkdir foo bar
> 
> # 下面命令会创建foo/a, foo/b, ... foo/h, bar/a, bar/b, ... bar/h这些文件
> touch {foo,bar}/{a..h}
> touch foo/x bar/y
> # 比较文件夹 foo 和 bar 中包含文件的不同
> diff <(ls foo) <(ls bar)
> # 输出
> # < x
> # ---
> # > y
> ```

### 在 shell 中调用其他语言编写的脚本

脚本并不一定只有用 bash 写才能在终端里调用。比如说，这是一段 Python 脚本，作用是将输入的参数倒序输出：

```python
#!/usr/bin/python3
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```

内核知道去用 `python` 解释器而不是 `shell` 命令来运行这段脚本，是因为脚本的开头第一行的 [`shebang`](https://en.wikipedia.org/wiki/Shebang_(Unix))。

在 `shebang` 行中使用 [`env`](https://man7.org/linux/man-pages/man1/env.1.html) 命令是一种好的实践，它会利用环境变量中的程序来解析该脚本，这样就提高了你的脚本的可移植性。`env` 会利用我们第一节讲座中介绍过的 `PATH` 环境变量来进行定位。 例如，使用了 `env` 的``shebang` 看上去是这样的`#!/usr/bin/env python3`。

### shell 函数和 shell 脚本

`shell` 函数和脚本有如下一些不同点：

- ==语言==：函数只能与 `shell` 使用相同的语言，脚本可以使用任意语言。因此在脚本中包含 `shebang` 是很重要的。
- ==加载时刻==：函数仅在定义时被加载，脚本会在每次被执行时加载。这让函数的加载比脚本略快一些，但每次修改函数定义（`source 函数文件名`），都要重新加载一次。
- ==执行环境==：函数会在当前的 `shell` 环境中执行，脚本会在单独的进程中执行。因此，**函数可以对环境变量进行更改，比如改变当前工作目录，脚本则不行。脚本需要使用 [`export`](https://man7.org/linux/man-pages/man1/export.1p.html) 将环境变量导出，并将值传递给环境变量。**
- ==两者关联==：与其他程序语言一样，函数可以提高代码模块性、代码复用性并创建清晰性的结构。`shell` 脚本中往往也会包含它们自己的函数定义。

## Shell 工具

### 查看命令如何使用

可用工具：

- `命令名 --help`
- 在交互式的、基于字符处理的终端窗口中，一般也可以通过 `:help` 命令或键入 `?` 来获取帮助。
- `man 命令名`
- `tdlr 命令名`（第三方，需要安装）

### 查找文件

可用工具：

- `find`：递归地搜索符合条件的文件，如：

  ```shell
  # 查找所有名称为src的文件夹
  find . -name src -type d
  # 查找所有文件夹路径中包含test的python文件
  find . -path '*/test/*.py' -type f
  # 查找前一天修改的所有文件
  find . -mtime -1
  # 查找所有大小在500k至10M的tar.gz文件
  find . -size +500k -size -10M -name '*.tar.gz'
  ```

  除了列出所寻找的文件之外，`find` 还能对所有查找到的文件进行操作。这能极大地简化一些单调的任务。

  ```shell
  # 删除全部扩展名为.tmp 的文件
  find . -name '*.tmp' -exec rm {} \;
  # 查找全部的 PNG 文件并将其转换为 JPG
  find . -name '*.png' -exec convert {} {}.jpg \;
  ```

  > 解释：
  >
  > `find .`：从当前目录（.）开始递归搜索。
  >
  > `-name '*.tmp'`：查找所有扩展名为 `.tmp` 的文件。`*.tmp` 使用了通配符 `*`，匹配所有以 `.tmp` 结尾的文件名。
  >
  > `-exec rm {} \;`：
  >
  > - `-exec`：对于每个找到的文件执行指定的命令。
  >
  > - `rm {}`：删除找到的文件。`{}` 是 `find` 命令的一个占位符，代表当前找到的文件。
  >
  > - `\;`：表示 `-exec` 命令的结束。
  >
  > 
  >
  > `find .`：从当前目录（.）开始递归搜索。
  >
  > `-name '*.png'`：查找所有扩展名为 `.png` 的文件。`*.png` 使用了通配符 `*`，匹配所有以 `.png` 结尾的文件名。
  >
  > `-exec convert {} {}.jpg \;`：
  >
  > - `-exec`：对于每个找到的文件执行指定的命令。
  >
  > - `convert {}`：使用 convert 命令（通常是 ImageMagick 软件包中的工具）进行文件格式转换。
  >
  > - `{}.jpg`：将转换后的文件命名为原文件名加上 `.jpg` 扩展名。`{}` 是 `find` 命令的一个占位符，代表当前找到的文件。
  >
  > - `\;`：表示 `-exec` 命令的结束。

- `fd`

  - 尽管 `find` 用途广泛，它的语法却比较难以记忆。例如，为了查找满足模式 `PATTERN` 的文件，你需要执行 `find -name '*PATTERN*'` (如果你希望模式匹配时是不区分大小写，可以使用 `-iname` 选项）

    你当然可以使用 `alias` 设置别名来简化上述操作，但 `shell` 的哲学之一便是寻找（更好用的）替代方案。 记住，`shell` 最好的特性就是你只是在==调用程序==，因此你只要找到合适的替代程序即可（甚至自己编写）。

  - [`fd`](https://github.com/sharkdp/fd) 就是一个更简单、更快速、更友好的程序，它可以用来作为 `find` 的替代品。它有很多不错的默认设置，例如输出着色、默认支持正则匹配、支持 unicode 并且它的语法更符合直觉。以模式`PATTERN` 搜索的语法是 `fd PATTERN`。

- `locate`

  - 大多数人都认为 `find` 和 `fd` 已经很好用了，但是有的人可能想知道，我们是不是可以有更高效的方法，例如不要每次都搜索文件而是通过编译索引或建立数据库的方式来实现更加快速地搜索。
  - 这就要靠 [`locate`](https://man7.org/linux/man-pages/man1/locate.1.html) 了。 `locate` 使用一个由 [`updatedb`](https://man7.org/linux/man-pages/man1/updatedb.1.html) 负责更新的数据库，在大多数系统中 `updatedb` 都会通过 [`cron`](https://man7.org/linux/man-pages/man8/cron.8.html) 每日更新。这便需要我们在速度和时效性之间作出权衡。而且，`find` 和类似的工具可以通过别的属性比如文件大小、修改时间或是权限来查找文件，`locate` 则只能通过文件名。 [这里](https://unix.stackexchange.com/questions/60205/locate-vs-find-usage-pros-and-cons-of-each-other)有一个更详细的对比。

### 查找代码

查找文件是很有用的技能，但是很多时候你的目标其实是查看文件的内容。一个最常见的场景是你希望查找具有某种模式的全部文件，并找它们的位置。

为了实现这一点，很多类UNIX的系统都提供了 [`grep`](https://man7.org/linux/man-pages/man1/grep.1.html) 命令，它是用于对输入文本进行匹配的通用工具。它是一个非常重要的 `shell` 工具，我们会在后续的数据清理课程中深入的探讨它。

`grep` 有很多选项，这也使它成为一个非常全能的工具。其中经常使用的有：

- `-C` ：获取查找结果的上下文（Context）；

- `-v` 将对结果进行反选（Invert），也就是输出不匹配的结果。

  举例来说， `grep -C 5` 会输出匹配结果前后五行。当需要搜索大量文件的时候，使用 `-R` 会递归地进入子目录并搜索所有的文本文件。

但是，我们有很多办法可以对 `grep -R` 进行改进，例如使其忽略 `.git` 文件夹，使用多 CPU 等等。

因此也出现了很多它的替代品，包括 [ack](https://beyondgrep.com/), [ag](https://github.com/ggreer/the_silver_searcher) 和 [rg](https://github.com/BurntSushi/ripgrep)。它们都特别好用，但是功能也都差不多，比较常用的是 ripgrep (`rg`) ，因为它速度快，而且用法非常符合直觉。例子如下：

```shell
# 查找所有使用了 requests 库的文件
rg -t py 'import requests'
# 查找所有没有写 shebang 的文件（包含隐藏文件）
rg -u --files-without-match "^#!"
# 查找所有的foo字符串，并打印其之后的5行
rg foo -A 5
# 打印匹配的统计信息（匹配的行和文件的数量）
rg --stats PATTERN
```

与 `find`/`fd` 一样，==重要的是你要知道有些问题使用合适的工具就会迎刃而解==，而具体选择哪个工具则不是那么重要。

### 查找 shell 命令

目前为止，我们已经学习了如何查找文件和代码，但随着你使用shell的时间越来越久，你可能想要找到之前输入过的某条命令。首先，按向上的方向键会显示你使用过的上一条命令，继续按上键则会遍历整个历史记录。

`history` 命令允许你以程序员的方式来访问 shell 中输入的历史命令。这个命令会==在标准输出中打印 shell 中的历史命令==。如果我们要搜索历史记录，则可以利用管道将输出结果传递给 `grep` 进行模式搜索。 `history | grep find` 会打印包含 find 子串的命令。

对于大多数的shell来说，你可以使用 `Ctrl+R` 对命令历史记录进行回溯搜索。敲 `Ctrl+R` 后你可以输入子串来进行匹配，查找历史命令行。

反复按下就会在所有搜索结果中循环。在 [zsh](https://github.com/zsh-users/zsh-history-substring-search) 中，使用方向键上或下也可以完成这项工作。

`Ctrl+R` 可以配合 [`fzf`](https://github.com/junegunn/fzf/wiki/Configuring-shell-key-bindings#ctrl-r) 使用。`fzf` 是一个通用的模糊查找工具，它可以和很多命令一起使用。这里我们可以对历史命令进行模糊查找并将结果以赏心悦目的格式输出。

另外一个和历史命令相关的技巧我喜欢称之为**基于历史的自动补全**。 这一特性最初是由 [fish](https://fishshell.com/) shell 创建的，它可以根据你最近使用过的开头相同的命令，动态地对当前的shell命令进行补全。这一功能在 [zsh](https://github.com/zsh-users/zsh-autosuggestions) 中也可以使用，它可以极大的提高用户体验。

### 文件夹导航

之前对所有操作我们都默认一个前提，即你已经位于想要执行命令的目录下，但是如何才能高效地在目录间随意切换呢？有很多简便的方法可以做到，比如设置 alias，使用 [ln -s](https://man7.org/linux/man-pages/man1/ln.1.html) 创建符号连接等。而开发者们已经想到了很多更为精妙的解决方案。

由于本课程的目的是尽可能对你的日常习惯进行优化。因此，我们可以使用 [`fasd`](https://github.com/clvv/fasd) 和 [autojump](https://github.com/wting/autojump) 这两个工具来查找最常用或最近使用的文件和目录。

Fasd 基于 [*frecency* ](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/Places/Frecency_algorithm)对文件和文件排序，也就是说它会同时针对频率（*frequency*）和时效（*recency*）进行排序。默认情况下，`fasd` 使用命令 `z` 帮助我们快速切换到最常访问的目录。例如， 如果你经常访问 `/home/user/files/cool_project` 目录，那么可以直接使用 `z cool` 跳转到该目录。对于 autojump，则使用`j cool`代替即可。

还有一些更复杂的工具可以用来概览目录结构，例如 [`tree`](https://linux.die.net/man/1/tree), [`broot`](https://github.com/Canop/broot) 或更加完整的文件管理器，例如 [`nnn`](https://github.com/jarun/nnn) 或 [`ranger`](https://github.com/ranger/ranger)。
