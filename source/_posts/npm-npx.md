---
title: npm && npx 是什么关系？
date: 2023-01-23 16:28:02
tags:
---

# npm
npm 是 NodeJs 的包管理工具。现代语言基本都有一个包管理工具。比如：
* Python: pip
* Ruby: gem
* Java: maven, gradle
* C#: nuget
* PHP: composer
* Go: go mod, dep
* Rust: cargo
* Scala: sbt, ivy
* Swift: cocoapods, carthage

# npx
npx 是 npm 的一个包运行工具，它可以帮助你在本地运行一个 npm 包中的命令行工具。它会在运行命令时自动安装该工具，并在运行完后自动删除，使得不用全局安装就能使用。

比如你可以使用 npx create-react-app my-app 在本地运行 create-react-app 包中的命令，来创建一个新的 React 应用而不用全局安装 create-react-app。

# 结论
简单来说，npm 是用来管理安装和卸载包，npx 是用来运行这些安装好的包中的命令。



