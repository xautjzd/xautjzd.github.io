---
title: Emacs 常用基本操作
date: "2022-08-04T00:00:00Z"
tags: ["emacs", "tool"]
---

 Emacs 作为 Programmer 熟知的两大 Editor 之一流行了很多年了，据说学习曲线陡峭，一直处于劝退状态。但作为喜欢折腾各种工具的自己来说，内心一直有跃跃欲试的想法，终于最近两个月得闲，开始研究学习上了。通过学习[官网Tutorials](https://www.gnu.org/software/emacs/manual/html_node/emacs/index.html) 熟悉其基本操作 & 配置，同时 Google 了解 Emacs 使用频次最高的一些插件，在使用过程中不断优化配置，打造成自己趁手的工具。目前，我主要有三种使用场景: 1. 文本编辑 2. 代码阅读 3. 写业余代码。以下是我在学习过程中整理的常用基本操作，分享出来一方面是当作学习笔记输出，以便后续遗忘时快速查找，另一方面也希望能带给他人一些帮助。

目前的常用操作有：

- 单文件操作 
- 光标移动
- 复制(copy) & 粘贴(paste)
- 插入(insert) & 删除(delete)
- 撤销(undo) & 重做(redo)
- 窗口操作
- 阅读代码常用操作
- 查看帮助文档

在使用 Emacs 时，有几个关键概念:

- Buffer
- Window
- Frame
- Mode

请自行查阅文档了解。

> **注：**`C-h` 表示按住 Ctrl 键的同时按 h 键；`M-x` 表示按住 Alt/Meta 键的同时按 x 键; `S-p f` 表示按住 Windows 键，再依次按 p f 键。

<!--more-->

## 1. 文件操作

- 启动 Emacs: `emacs` 或 `emacs <file to open>`
- Emacs 内新建/打开文件: `C-x C-f`
- 文件保存: `C-x C-s`
- 退出 minibuffer: `C-g`
- 退出 Emacs: `C-x C-c`

## 2. 光标移动

- 上移: `C-p`
- 下移: `C-n`
- 左移: `C-b` 左移一个字符, `M-b` 左移一个单词
- 右移: `C-f` 右移一个字符，`M-b` 右移一个单词
- 移至行首: `C-a`
- 移至行尾: `C-e`
- 移至句首: `M-a`
- 移至句尾: `M-e`
- 移至文件首: `M-<`
- 移至文件尾: `M->`
- 向上滚动(内容下移): `C-v`
- 向下滚动(内容上移): `M-v`
- 括号间移动: `C-M-f`, `C-M-b`, `C-M-n`, `C-M-p`

## 3. 复制粘贴

- 复制: `M-w`
- 剪贴: `C-w`
- 粘贴：`C-y`
- 区域选中: `C-SPC`
- 全选当前缓冲区: `C-x h`

## 4. 插入删除

- 在当前行下一行插入: `C-e C-j`/`C-e C-m`
- 删除当前行: `C-S-backspace`/ `C-a C-k`

## 5. 撤销(undo)/重做(redo)

- 撤销: `C-/`
- 重做: `C-g C-/`

## 6. 窗口操作

- 切换窗口: `C-x o`
- 关闭当前窗口: `C-x 0`
- 关闭其他所有窗口: `C-x 1`
- 垂直切分窗口: `C-x 3`
- 水平切分窗口: `C-x 2`

## 7. 程序模式

- 跳转定义： `M-.`
- 返回定义：`M-,`
- 查看调用：`M-?`
- 移至下一个方法: `C-M-e`
- 移至上一个方法: `C-M-a`
- 注释/取消注释当前行: `C-x C-;`
- 多行注释：选中范围后，`M-;`
- 项目内按文件查找(已安装 projectile & ivy 插件包的前提下)：`S-p f`。
- 项目内按内容查找: `S-p s r` 使用 ripgrep 搜索, `S-p s s` 使用 ag 搜索, `S-p s g` 使用 grep 搜索 。
- 错误信息实时显示: `S-l g e` (依赖 flymake, lsp-mode, 调用方法为: `lsp-treemacs-errors-list`)

## 8. 帮助文档

- 查看帮助文档操作: `C-h ?`
- 查看互动教程: `C-h t`
- 查看快捷键对应命令: `C-h k`
- 查看命令: `C-h w`

## 其他

- 运行命令: `M-x`
- 运行上一个表达式: `C-x C-e`
- 多行编辑: `C-x r t` (先选中范围)
- 设置快捷键: `(global-set-key "\C-x2" 'split-window-quietly)`

以上是默认 Emacs 操作，若习惯了 Vim 的操作方式，切换至 Emacs 时可采用 [evil 插件](https://github.com/emacs-evil/evil)。最后在附上我个人的 Emacs 配置: [https://github.com/xautjzd/emacs.d](https://github.com/xautjzd/emacs.d)
