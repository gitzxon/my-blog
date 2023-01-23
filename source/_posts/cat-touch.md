---
title: cat && touch
date: 2023-01-23 14:43:36
tags:
---

# cat && touch
cat 和 touch 是工作中用的比较多的命令，linux、macos 上基本都能用。本人总是非常神奇的会把这两兄弟的作用记混。

`cat` 命令用于将文件的内容显示在终端上。它可以将一个或多个文件的内容按顺序输出到标准输出。例如，在终端中输入 cat file.txt 将会显示 file.txt 文件的内容。
`touch` 命令用于更新文件的时间戳。如果文件不存在，它会创建一个新文件。例如，在终端中输入 touch file.txt 将会创建一个新文件名为 file.txt 的文件或更新 file.txt 文件的时间戳。

所以，输出内容的时候，使用 `cat`，创建文件的时候，使用 `touch`。