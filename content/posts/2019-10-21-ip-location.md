---
title: IP 归属地查询
date: "2019-10-21T00:00:00Z"
tags: ["网络", "tool"]
---

这里分享下几个不错的 ip 地址库:

## 1. [https://ip.cn](https://ip.cn)

### 查询指定 IP

```
curl https://ip.cn?ip=202.200.112.2
```

### 查询本机 IP

```
curl https://ip.cn
```

**优势**: 国内服务，速度较快, 且免费

**劣势**: 信息量太少, 没有经纬度信息

<!--more-->

## 2. [https://www.ipip.net](https://www.ipip.net)

### 查询指定 IP

命令行方式:

```
curl http://freeapi.ipip.net/202.200.112.2
```

网页方式:

```
https://www.ipip.net/ip.html
```

**优势**: 国内服务，速度快

**劣势**: 命令行免费版信息少(网页版 & 收费版信息较全)

## 3. [https://ip-api.com/](https://ip-api.com/)

### 查询指定 IP

命令行方式:

```
curl http://ip-api.com/json/202.200.112.2
```

API 使用请参考: [https://ip-api.com/docs](https://ip-api.com/docs)

也可在官网直接查询。

**优势**: 返回信息全，文档丰富

**劣势**: 国外服务，速度相对较慢(其实也挺快)
