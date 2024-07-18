---
layout: default
title: 数据整理
---

# Lec04：数据整理

> 本节的笔记记录主要以记录用于数据整理的工具为主

本 Lec 的主要内容：

- 一系列数据整理工具的使用：`grep sed sort uniq paste awk wc bc st R gnuplot xargs ffmpeg less`

## 用 grep 来提取含有某模式的数据

- `ssh myserver journalctl | grep sshd`

- `ssh myserver 'journalctl | grep sshd | grep "Disconnected from"' | less`

  > `less` 创建了一个文件分页器，使我们可以通过翻页的方式浏览较长的文本。

## sed 

`sed` 是一个基于文本编辑器 `ed` 构建的 ”流编辑器” 。在 `sed` 中，你基本上是利用一些简短的命令来修改文件，而不是直接操作文件的内容（尽管你也可以选择这样做）。相关的命令行非常多，但是最常用的是 `s`，即 *替换* 命令。

```shell
ssh myserver journalctl
 | grep sshd
 | grep "Disconnected from"
 | sed 's/.*Disconnected from //'
```

上面这段命令中，我们使用了一段简单的 *正则表达式*。正则表达式是一种非常强大的工具，可以让我们基于某种模式来对字符串进行匹配。`s` 命令的语法如下：`s/REGEX/SUBSTITUTION/`, 其中 `REGEX` 部分是我们需要使用的正则表达式，而 `SUBSTITUTION` 是用于替换匹配结果的文本。

> 正则表达式语法：
>
> | 符号        | 含义                                               |
> | ----------- | -------------------------------------------------- |
> | `abc...`    | 字母                                               |
> | `123...`    | 数字                                               |
> | `\d`        | 任意数字                                           |
> | `\D`        | 任意非数字                                         |
> | `.`         | 任意符号                                           |
> | `\.`        | 句号                                               |
> | `[abc]`     | a、b 或 c                                          |
> | `[^abc]`    | 不是 a、b 和 c                                     |
> | `[a-z]`     | a 到 z                                             |
> | `[0-9]`     | 0 到 9                                             |
> | `\w`        | 任意字母                                           |
> | `\W`        | 任意非字母                                         |
> | `{m}`       | 重复 m 次                                          |
> | `{m,n}`     | 重复 m 到 n 次                                     |
> | `*`         | 重复 0 到多次                                      |
> | `+`         | 重复 1 到多次                                      |
> | `?`         | 可能存在；加在在 `*` 或 `+` 后面使其变成非贪婪模式 |
> | `\s`        | 空格                                               |
> | `\S`        | 非空格                                             |
> | `^...$`     | 匹配行首和行尾                                     |
> | `(...)`     | 捕获组                                             |
> | `(a(bc))`   | 嵌套捕获组                                         |
> | `(.*)`      | 捕获所有                                           |
> | `(abc|def)` | 捕获 abc 或 def                                    |
>
> 建议在使用 `sed` 时都是加上 `-E` 选项
>
> 注意⚠️：`sed` 默认不支持将 `?` 作为非贪婪模式标志，要启用这个用法,使用 `perl`：`perl -pe 's/.*?Disconnected from //'`
>
> tips：正则表达式在线调试工具 [regex debugger](https://regex101.com/r/qqbZqh/2)

> `sed` 的捕获组：
>
> 被圆括号内的正则表达式匹配到的文本，都会被存入一系列以编号区分的捕获组中。捕获组的内容可以在替换字符串时使用（有些正则表达式的引擎甚至支持替换表达式本身），例如 `\1`、 `\2`、`\3`等等。
>
> 捕获组序号按括号**由左向右被解析的顺序**来标号。

## sort

`sort` 用于对输入进行排序

`sort` 默认会对输入流按**字母序**排序

加上选项 `-r` 会将排序结果**颠倒**

加上选项 `-n` 则会按照**数字顺序**排序

`sort -nk1,1` 会对输入流按照**第一列的数字顺序**排序，`sort -nk1,2` 会对输入流按照**第一列和第二列的数字顺序**排序，当且仅当第一列数字相同时才比较第二列数字的大小。

## uniq

`uniq` 用于处理有重复数据的输入

`uniq -c` 会把连续出现的行折叠为一行并使用出现次数作为前缀（第一列）。

## paste（MacOS）

`paste` 用于将多行内容合并为一行

`paste -sd ',' input.txt`：`-s` 表示合并行，`-d` 表示选择分隔符，后面接具体的分隔符，输入可以是文件也可以是流

## awk - 另外一种编辑器

`awk` 其实是一种编程语言，只不过它碰巧非常善于处理文本。关于 `awk` 可以介绍的内容太多了，限于篇幅，这里我们仅介绍一些基础知识。

`awk` 程序接受一个模式串（可选），以及一个代码块，指定当模式匹配时应该做何种操作。

在代码块中，`$0` 表示整行的内容，`$1` 到 `$n` 为一行中的 n 个区域，区域的分割基于 `awk` 的域分隔符（默认是空格，可以通过`-F`来修改）。

```shell
ssh myserver journalctl
 | grep sshd
 | grep "Disconnected from"
 | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'
 | sort | uniq -c
 | sort -nk1,1 | tail -n10
 | awk '{print $2}' | paste -sd ','
```

在这个例子中，我们的代码意思是：对于每一行文本，打印其第二个部分，也就是用户名。

不过，既然 `awk` 是一种编程语言，那么则可以这样：

```shell
BEGIN { rows = 0 }
$1 == 1 && $2 ~ /^c[^ ]*e$/ { rows += $1 }
END { print rows }
```

`BEGIN` 也是一种模式，它会匹配输入的开头（ `END` 则匹配结尾）。然后，对每一行第一个部分进行累加，最后将结果输出。事实上，我们完全可以抛弃 `grep` 和 `sed` ，因为 `awk` 就可以[解决所有问题](https://backreference.org/2010/02/10/idiomatic-awk)。至于怎么做，就留给读者们做课后练习吧。

## wc

`wc` 用于对输入进行数量统计

`wc -l` 统计输入行数

## bc

`bc` 主要用于对输入数据作数学运算

```shell
 | paste -sd '+' | bc -l
```

这段代码将每行的数字加起来

下面这种更加复杂的表达式也可以：

```shell
echo "2*($(data | paste -sd '+'))" | bc -l
```

## st

你可以通过多种方式获取统计数据。如果已经安装了R语言，[`st`](https://github.com/nferraz/st) 是个不错的选择

## R

R 也是一种编程语言，它非常适合被用来进行数据分析和[绘制图表](https://ggplot2.tidyverse.org/)。这里我们不会讲的特别详细， 你只需要知道`summary` 可以打印某个向量的统计结果。我们将输入的一系列数据存放在一个向量后，利用 R 语言就可以得到我们想要的统计数据。

```shell
ssh myserver journalctl
 | grep sshd
 | grep "Disconnected from"
 | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'
 | sort | uniq -c
 | awk '{print $1}' | R --slave -e 'x <- scan(file="stdin", quiet=TRUE); summary(x)'
```

## gnuplot

如果你希望绘制一些简单的图表， `gnuplot` 可以帮助到你：

```shell
ssh myserver journalctl
 | grep sshd
 | grep "Disconnected from"
 | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'
 | sort | uniq -c
 | sort -nk1,1 | tail -n10
 | gnuplot -p -e 'set boxwidth 0.5; plot "-" using 1:xtic(2) with boxes'
```

## xargs

`xargs` 用于将其**输入**转化为某个程序的若干个**参数**

例子：

```shell
rustup toolchain list | grep nightly | grep -vE "nightly-x86" | sed 's/-x86.*//' | xargs rustup toolchain uninstall
```

## 不只是文本数据

虽然到目前为止我们的讨论都是基于文本数据，但对于二进制文件其实同样有用。例如我们可以用 `ffmpeg` 从相机中捕获一张图片，将其转换成灰度图后通过 SSH 将压缩后的文件发送到远端服务器，并在那里解压、存档并显示。

```shell
ffmpeg -loglevel panic -i /dev/video0 -frames 1 -f image2 -
 | convert - -colorspace gray -
 | gzip
 | ssh mymachine 'gzip -d | tee copy.jpg | env DISPLAY=:0 feh -'
```

