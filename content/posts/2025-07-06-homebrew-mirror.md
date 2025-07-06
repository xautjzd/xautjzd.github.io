---
title: Homebrew 国内下载慢的解决之法及下载过程剖析
date: "2025-07-06T10:00:00+08:00"
tags: ["tool", "macos"]
---

对懂点技术的人来说， [HomeBrew](https://brew.sh/) 几乎成为了 MacOS 上的必装软件，主打一个安装方便，在终端一个命令即可安装配置好。MacOS 下使用的软件大致分为两类： 一类是如 [fd](https://github.com/sharkdp/fd)、[neovim](https://neovim.io/) 在终端下使用的软件，另一类是如 Google Chrome 的 GUI 软件，通常可在 Apple Store 下载。 终端安软件安装方式: `brew install <formula>`，GUI 软件安装方式: `brew install --cask <cask>`，其中的 `<formula>` 和 `<cask>` 均为你想安装的软件名称， 如果不确定具体名称，可通过: `brew search <keyword>`搜索。 在国内网络环境下，在 MacOS 上安装软件时，通常会卡在如下所示的流程：

![](/images/brew-downloading.png)

等了好久也没见动静，这是因为默认下载源是从 GitHub 数据中心下载，国内对 GitHub 有限制。

## 解决办法：配置国内镜像源

目前国内主流的 Homebrew 镜像有：清华大学、阿里云、中科大、腾讯云等。以清华源和阿里云为例，介绍配置方法。

### 1. 配置清华源

#### 替换 Homebrew 源
```bash
#替换 brew.git
cd "$(brew --repo)"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git

# 替换 homebrew-core.git
cd "$(brew --repo homebrew/core)"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git

# 替换 homebrew-cask.git
cd "$(brew --repo homebrew/cask)"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask.git

# 配置 bottle 镜像(formula 二进制)
# 推荐写入 ~/.zshrc 或 ~/.bash_profile
export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles

# 立即生效
source ~/.zshrc  # 或 source ~/.bash_profile

# 更新 brew
brew update
```

### 2. 配置阿里云源

```bash
# 替换 brew.git
cd "$(brew --repo)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git

# 替换 homebrew-core.git
cd "$(brew --repo homebrew/core)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git

# 替换 homebrew-cask.git
cd "$(brew --repo homebrew/cask)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-cask.git

# 配置 bottle 镜像
export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles
source ~/.zprofile  # 或 source ~/.bash_profile
brew update
```

其他镜像如中科大、腾讯云等配置方法类似，详见各镜像站帮助文档。

### 恢复官方源

如需恢复官方源：
```bash
git -C "$(brew --repo)" remote set-url origin https://github.com/Homebrew/brew.git
git -C "$(brew --repo homebrew/core)" remote set-url origin https://github.com/Homebrew/homebrew-core.git
git -C "$(brew --repo homebrew/cask)" remote set-url origin https://github.com/Homebrew/homebrew-cask.git
unset HOMEBREW_BOTTLE_DOMAIN
brew update
```

## Homebrew 软件下载过程剖析

Homebrew 的软件包分为两大类：
- **Formula**：命令行工具、库等（如 wget、git、python）
- **Cask**：图形界面应用（如 Google Chrome、Visual Studio Code）

### Formula 的下载流程
1. 用户执行 `brew install <formula>`。
2. 自 Homebrew 4.0 起，Homebrew 优先通过 API（如 https://formulae.brew.sh/api/formula.jws.json）获取 formula 信息，Homebrew 默认会直接请求 API 获取最新的 formula 元数据（包括版本、依赖、bottle 地址等），在 API 不可用或用户设置了  HOMEBREW_NO_INSTALL_FROM_API=1 时，会回退到查询本地 formula 脚本索引（Ruby 脚本，描述如何下载/编译软件），Apple Silicon 芯片的脚本位置通常位于： `/opt/homebrew/Library/Taps/homebrew/homebrew-core/Formula` 的目录下。
3. 优先尝试下载预编译的二进制包（bottle），URL 由 `HOMEBREW_BOTTLE_DOMAIN` 控制。
4. 若无 bottle 或下载失败，则源码编译安装，源码 URL 由 formula 文件指定。
5. 安装后自动链接到 `/usr/local/bin` 或 `/opt/homebrew/bin`。

#### 关键点
- 配置国内 bottle 镜像后，绝大多数常用软件可直接高速下载二进制包
- 极少数软件或自定义参数会触发源码编译，速度取决于源码仓库和本地编译性能

### Cask 的下载流程
1. 用户执行 `brew install --cask <cask>`
2. Homebrew 查询本地 cask 索引（Ruby 脚本，描述 App 的下载和安装方式）
3. 直接从 App 官方网站或 GitHub 下载 dmg/pkg/zip 等安装包
4. 下载完成后自动挂载/解压并安装到 `/Applications`

#### 关键点
- Cask 下载链接通常直连 App 官方站点，**不受 bottle 镜像影响**
- 国内访问部分 App 官网（如 Google Chrome、VSCode）可能依然较慢
- 可通过科学上网或手动下载后 `brew install --cask <cask> --no-quarantine --appdir=/Applications <本地文件>` 安装

## 总结与建议

- Formula 类软件优先配置国内 bottle 镜像，极大提升下载速度
- Cask 类软件如遇下载慢，可考虑手动下载安装包
- 遇到 `brew update`、`brew install` 卡顿，优先检查源配置和网络环境
- 推荐定期关注各大镜像站公告，及时调整配置

## 参考链接

- [清华大学开源软件镜像站 Homebrew 帮助](https://mirror.tuna.tsinghua.edu.cn/help/homebrew/)
- [阿里云 Homebrew 镜像](https://developer.aliyun.com/mirror/homebrew)
- [Homebrew 官方文档](https://docs.brew.sh/) 