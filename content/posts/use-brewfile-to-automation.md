---
title: "MacOS 环境批量安装软件"
date: 2023-01-19T17:22:37+08:00
tags: ["macos", "tool"]
---

最近工作使用的电脑新换到 MacBook Pro M1 13寸，平时工作使用的各种工具 & 软件都需要重新安装 & 配置，耗时耗力而且没有太多成就感，所以一直在思索如何将其自动化掉，避免每次换电脑都要做重复的劳动。自己平时安装各种软件比较依赖 [HomeBrew](https://brew.sh/)，印象中它有提供 Brewfile 可以批量安装软件，只是自己一直没有使用起来。刚好趁这次机会拿来练练手，一来总结经验以备下次复用，二来分享出来希望可以帮助到他人。下面分享下我是如何使用 Brewfile 批量安装工作必备软件的。

## 1. 前提

使用 Brewfile 进行批量安装前，需要安装：

- [git](https://git-scm.com/)
- [HomeBrew](https://brew.sh)

另外若在 Brewfile 中指定 mas 从 AppleStore 安装软件，请保证 AppleID 已登录。

## 2. 列出所需软件清单，形成 Brewfile

在批量安装前，列出自己平时工作所需的软件，在任意目录下创建名为 `Brewfile` 的文件，文件内容及格式可以参考我目前使用的 [Brewfile](https://github.com/xautjzd/dotvim/blob/master/Brewfile) 来声明:

```
# taps
tap "homebrew/bundle"
tap "homebrew/cask"
tap "homebrew/core"

# packages
brew 'vim'
brew 'git'

brew 'zsh'
brew 'tmux'
brew 'ripgrep'
brew 'bat'
brew 'helix'
brew 'jq'
brew 'mas'

# tools
cask 'emacs'
cask 'google-chrome'
cask 'notion'
cask 'alfred'
cask 'warp'
cask 'intellij-idea-ce'
cask 'sequel-pro'
cask 'clashx'
cask 'tunnelblick'

# language
# cask 'java'
brew 'go'
brew 'cmake'
brew 'maven'
brew 'yarn'

# install apps from apple store: 1. mas search <keyword>, find id 2. add one line below 
mas "MindNode", id: 1289197285
mas "CopyClip", id: 595191960
mas "QQ音乐", id: 595615424
```

文件内容主要分为四部分:

- tap: 声明 brew 下载软件源，详细介绍请通过: `brew help tap` 查看。
- brew: 声明安装终端使用工具。
- cask: 声明安装 MacOS 本地应用软件。
- mas: 声明将从 AppleStore 下载的应用。

其中 `#` 开头是注释，这里提一下 mas。[mas](https://github.com/mas-cli/mas) 是 Apple Store 的命令行工具，可以利用 mas 在终端下安装 Apple Store 中的软件，Brewfile 中 mas 部分需指定软件名称及 id，若我们已知软件名称，可通过 `mas search <appName>` 搜索，找到待安装软件 id, 填入 Brewfile。

## 2. 利用 Brewfile 批量安装软件

在终端下切换到 Brewfile 所在目录，执行: `brew bundle` 或 `brew bundle install` 进行批量安装/更新，更多信息可查阅 bundle 帮助文档: `brew help bundle`。 

## 3. 如何清理从 Brewfile 中移除的已安装软件？

在 Brewfile 所在目录执行: `brew bundle cleanup` 卸载未列在当前 Brewfile 中的已安装软件(仅通过 brew 安装, 不包含 mas 部分)。
