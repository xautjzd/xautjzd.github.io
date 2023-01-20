---
title: Go 二进制应用注入 version 信息
date: "2019-10-20T00:00:00Z"
tags: ["Go"]
---

当线上服务出现问题时，在排查问题前，我们需要确定线上服务部署版本、对应源码分支、最近一次提交记录、编译/打包时间、语言版本等，这些信息可写入配置文件，应用启动时读取。若采用 docker 部署时，也可将这些信息作为 LABEL 或应用镜像的 tag 中，方式多种多样，今天要介绍的是 go 应用下的另一个可选方式: 二进制编译时注入。

<!--more-->

## 如何使用？

在 `go build` 时动态传入:

```
go build -ldflags "-X importpath.name=value" xxx.go

```

ldflags 使用请参考:

> -X importpath.name=value  
	Set the value of the string variable in importpath named name to value.  
	This is only effective if the variable is declared in the source code either uninitialized  
	or initialized to a constant string expression. -X will not work if the initializer makes  
	a function call or refers to other variables.  
	Note that before Go 1.5 this option took two separate arguments.

更多编译相关参数，请参考: [https://golang.org/cmd/link/](https://golang.org/cmd/link/)

下面以一个具体实例说明:

```
package main

import (
    "fmt"
)

var (
    gitBranch string
    gitCommit string
    built     string
    goVersion string
)

func main() {
    fmt.Printf("gitBranch: %s, gitCommit: %s, built: %s, go version: %s\n", gitBranch, gitCommit, built, goVersion)
}

```

编译时，可通过 -X 动态传入版本等信息:

```
go build -ldflags "-X 'main.gitBranch=$(git rev-parse --abbrev-ref HEAD)' -X 'main.gitCommit=$(git rev-parse --short HEAD)' -X 'main.built=$(date +"%Y-%m-%d %T%z")' -X 'main.goVersion=$(go version)'" version.go
```

gitCommit 等变量放置在main.go里，故 `-X` 传参时指定的 importpath 为 main。若为其他，则改为对应 importpath

