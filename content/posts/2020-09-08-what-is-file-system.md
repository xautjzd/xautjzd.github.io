---
title: 什么是文件系统？
date: "2020-09-08T00:00:00Z"
tags: ["基础技术", "OS"]
---

- 文件系统是什么？
- 文件系统组成
- 如何写一个简单的文件系统

## 文件系统是什么？

- 软件
- 记录文件元信息(文件大小，所属用户，包含块等)


## 文件系统组成

### inode

inode 记录着文件的元信息，包含以下部分:

- 文件大小
- 组件文件的块(block)
- 属主(owner)
- 访问权限
- 访问时间
- 创建时间
- 修改时间

### inode table

为了管理 inode 信息，需要有个数据数据结构存储 inode 信息, 称之为 inode table, 会占一部分磁盘空间，若 inode 大小为 256 bytes, 一个 4k 大小的 block 可存储 16 个 inodes。

### allocation list

为了管理 block 是否已分配，需要有个数据结构存储，通常可以用[位图(bitmap)](https://en.wikipedia.org/wiki/Bitap_algorithm) 算法进行存储, 用 1bit 表示该 block 是否已使用，一个 4k 大小的 block 可存储 32k 个 block 使用情况。

### superblock

superblock 用于存储文件系统元信息，包括 inodes 数量、文件系统中数据块(data block)数量、inode table 起始块位置、用于标识文件系统类型的魔数(magic number)等。操作系统挂载文件系统时，会首先读取 superblock 信息。

## 参考资料

1. [File System Implementation](http://pages.cs.wisc.edu/~remzi/OSTEP/file-implementation.pdf)
2. [Writing your own file system is easier than you think](https://medium.com/datadriveninvestor/writing-your-own-file-system-is-not-as-hard-as-you-may-think-aa4af09329cd)
3. [Understanding Linux filesystems: ext4 and beyond](https://opensource.com/article/18/4/ext4-filesystem)
