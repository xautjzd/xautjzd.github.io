---
title: 域名是如何解析的？
date: "2022-08-18T00:00:00Z"
tags: ["基础技术", "网络"]
---
## 基本概念

我们生活在互联网时代，每天的工作生活几乎都在与互联网打交道，比如使用 Google 查找工作学习所需资料，使用 Youtube、抖音、网易云音乐等进行娱乐活动，通过 Twitter、微博、微信等获取新闻资讯及社交，使用京东、天猫、拼多多等购物，使用微信支付、支付宝支付进行付款等，这些都离不开网络。那么作为构成网络基础设施核心之一的域名(Domain)你知道是如何工作的吗？下面我们简单展开介绍下。

所有的信息都存储在服务器上，可通过 IP 地址访问获取，但通常 IP 地址难以记住，于是乎产生了域名概念。起初网络仅有少数服务器构成，域名到 IP 的映射信息可存储在单个文件(eg: /etc/hosts)，随着网络的蓬勃发展，单文件难以容纳所有的域名IP 映射关系，同时映射关系变更时，难以及时同步至各服务器，于是 [DNS(Domain Name System) 协议](https://en.wikipedia.org/wiki/Domain_Name_System)诞生，映射关系通过 DNS 服务器集中存储，用户根据所需向 DNS 服务器查询。那么 DNS 服务器是如何来存储域名映射关系的呢？这个映射关系是谁来负责写入的呢？下面来揭晓。

<!--more-->

## 域名组成

在介绍域名分配存储前，先简要介绍下域名的组成。比如我们经常访问的 Google，其域名是: `www.google.com`, 其实完整的域名应该是: `www.google.com.`，最后面应该有一个 `.`，这个点称为根域名(root domain)，全球一共只有 13 个根域名(每个根域名其实都对应一个集群), 分别为: [a-m].root-servers.net，其 IP 地址固定，每个 DNS 服务器都有一份根域名的配置，内容参考下面引用；`com` 称为顶级域名(Top Level Domain, aka: TLD)，顶级域名由 [ICANN(the Internet Corporation for Assigned Names and Numbers)](https://www.icann.org/resources/pages/welcome-2012-02-25-en) 负责分配，顶级域名分类请参考: [https://developer.mozilla.org/en-US/docs/Glossary/TLD](https://developer.mozilla.org/en-US/docs/Glossary/TLD), 全量顶级域名列表请参考: [https://www.icann.org/resources/pages/tlds-2012-02-25-en](https://www.icann.org/resources/pages/tlds-2012-02-25-en)；`google` 称为二级域名(Second Level Domain， aka: 2LD)，通常为企业或组织向域名服务提供商(Domain Registrars)申请；`www` 称为三级域名(Third Level Domain)，表示提供具体的某项服务，三级域名由二级域名所有者自己负责管理及解析。由此可以看出域名其实一个倒序结构，从左至右范围由小到大。

```
a.root-servers.net. IN  A   198.41.0.4
a.root-servers.net. IN  AAAA    2001:503:BA3E::2:30

b.root-servers.net. IN  A   192.228.79.201

c.root-servers.net. IN  A   192.33.4.12

d.root-servers.net. IN  A   199.7.91.13
d.root-servers.net. IN  AAAA    2001:500:2d::d

e.root-servers.net. IN  A   192.203.230.10

f.root-servers.net. IN  A   192.5.5.241
f.root-servers.net. IN  AAAA    2001:500:2f::f

g.root-servers.net. IN  A   192.112.36.4

h.root-servers.net. IN  A   128.63.2.53
h.root-servers.net. IN  AAAA    2001:500:1::803f:235

i.root-servers.net. IN  A   192.36.148.17
i.root-servers.net. IN  AAAA    2001:7fe::53

j.root-servers.net. IN  A   192.58.128.30
j.root-servers.net. IN  AAAA    2001:503:C27::2:30

k.root-servers.net. IN  A   193.0.14.129
k.root-servers.net. IN  AAAA    2001:7fd::1

l.root-servers.net. IN  A   199.7.83.42
l.root-servers.net. IN  AAAA    2001:500:3::42

m.root-servers.net. IN  A   202.12.27.33
m.root-servers.net. IN  AAAA    2001:dc3::35
```

## 域名存储

域名服务器主要分为两类: 1. 本地域名服务器(Local DNS Server) 2. 权威域名服务器(Authority DNS Server)。其中本地域名服务器通常由网络运行商提供，通过 DHCP 获取，用户无感知，主要负责解析用户发起的域名解析请求，解析完成后缓存解析结果，缓存一定时间，在缓存有效期内再次域名解析时可直接返回上次的解析结果；权威域名服务器负责解析自己所拥有的域名及子域名，以 Google 为例，google.com 这个域名为 Google 所有，所有的 *.google.com 域名请求都会发送至 google 自己搭建的域名服务器(如没有能力维护自己的域名服务器，可以使用第三方的域名服务，如: [DNSPod](https://www.dnspod.cn))。目前大部分域名服务器都是基于 [Bind](https://www.isc.org/bind/) 搭建，有兴趣可以深入了解。

## 域名解析

域名解析过程如下图所示:

![](/images/dns-resolve.png)

具体为:

1. 用户在浏览器向本地域名服务器(MacOS/Linux 系统可通过 /etc/resolv.conf 查看服务器地址)发起域名解析请求，如 www.google.com。
2. 本地域名服务器收到解析请求后，向根域名服务器发送顶级域名解析请求。
3. 根域名服务器返回解析结果至本地域名服务器。
4. 本地域名服务器向顶级域名服务器发送二级域名解析请求。
5. 顶级域名服务器返回解析结果至本地域名服务器。
6. 本地域名服务器向二级域名服务器发送三级域名解析请求。
7. 二级域名服务器返回解析结果至本地域名服务器。
8. 本地域名服务器缓存解析结果至本地，返回解析结果给用户。

## 域名记录

常用的域名记录类型有:

- A: 根据域名查询对应 IP v4 地址。
- AAAA: 根据域名查询对应 IP v6 地址。
- CNAME: 域名别名记录，通常 CDN 时会使用，CDN 服务商提供一个加速域名，将自己的域名 cname 到 CDN 的加速域名上。
- NS: nameserver 记录，即权威域名服务器。
- MX: Mail eXchange，指定邮件服务器地址。
- PTR: 根据 IP 反向查询对应域名记录。
- SOA: Start Of Authority, 记录 zone 的一些属性信息，如 TTL，刷新间隔等。
- TXT: 通常用于 [SPF](http://www.open-spf.org/)。

## 域名解析工具

- dig
- nslookup

下面使用 dig 演示下常用使用场景。

### 查询 A 记录

`$dig www.google.com`

结果如下:

```
; <<>> DiG 9.10.6 <<>> www.google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45838
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;www.google.com.                        IN      A

;; ANSWER SECTION:
www.google.com.         147     IN      A       172.217.163.36

;; Query time: 9 msec
;; SERVER: 30.30.30.30#53(30.30.30.30)
;; WHEN: Fri Aug 19 17:05:59 CST 2022
;; MSG SIZE  rcvd: 59
```

### 迭代方式查询 A 记录

`$dig +trace www.github.com`

结果如下:

```
; <<>> DiG 9.10.6 <<>> +trace www.github.com
;; global options: +cmd
.			445010	IN	NS	h.root-servers.net.
.			445010	IN	NS	i.root-servers.net.
.			445010	IN	NS	j.root-servers.net.
.			445010	IN	NS	l.root-servers.net.
.			445010	IN	NS	k.root-servers.net.
.			445010	IN	NS	m.root-servers.net.
.			445010	IN	NS	a.root-servers.net.
.			445010	IN	NS	b.root-servers.net.
.			445010	IN	NS	c.root-servers.net.
.			445010	IN	NS	d.root-servers.net.
.			445010	IN	NS	e.root-servers.net.
.			445010	IN	NS	f.root-servers.net.
.			445010	IN	NS	g.root-servers.net.
;; Received 544 bytes from 30.30.30.30#53(30.30.30.30) in 33 ms

com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			86400	IN	DS	30909 8 2 E2D3C916F6DEEAC73294E8268FB5885044A833FC5459588F4A9184CF C41A5766
com.			86400	IN	RRSIG	DS 8 1 86400 20220901050000 20220819040000 20826 . lKRP5U+aBRp7KAVK2NVEP8D2aH4+LerGeT7mYWeQx/zXc/TLgBN3XpdT 14Gb4EXuiyS97P1XSqfqmizPFTAkrL1KcnFwmqg7nuHgtW2xY1Dw5agc z1dkR1SyORdZ0ClONfifrYn2f4zozTpHAk9/8+H6bsuCN5qidz5AefnK lU5yUR47/+u/E7jzY/CCkfkSj7PCdWH02hf5pqBjka+bc6tQvSRBlG/j HEUc8FSiEHVeu8jjZb87nf+I6Uq5pZLDLtR1KMR/Ewtt35YXwRl7x9gT flB3aKTerOyElvKMmjF2deZInF8So//ibl7PzdLbYDtX2fjDpJT4oA08 BEaW+w==
;; Received 1174 bytes from 192.203.230.10#53(e.root-servers.net) in 156 ms

github.com.		172800	IN	NS	ns-520.awsdns-01.net.
github.com.		172800	IN	NS	ns-421.awsdns-52.com.
github.com.		172800	IN	NS	ns-1707.awsdns-21.co.uk.
github.com.		172800	IN	NS	ns-1283.awsdns-32.org.
github.com.		172800	IN	NS	dns1.p08.nsone.net.
github.com.		172800	IN	NS	dns2.p08.nsone.net.
github.com.		172800	IN	NS	dns3.p08.nsone.net.
github.com.		172800	IN	NS	dns4.p08.nsone.net.
CK0POJMG874LJREF7EFN8430QVIT8BSM.com. 86400 IN NSEC3 1 1 0 - CK0Q2D6NI4I7EQH8NA30NS61O48UL8G5  NS SOA RRSIG DNSKEY NSEC3PARAM
CK0POJMG874LJREF7EFN8430QVIT8BSM.com. 86400 IN RRSIG NSEC3 8 2 86400 20220823042413 20220816031413 32298 com. Enbw7kmsz48csnt/+2IImg9UXUMiORjdqjCbvReZvY+xnAS1HG8z6usT EK2Niky82goEJvdt9m7PPy50IR1c2UgLtDxP7RUjpjj5BsEJCxpZkrVI ZamykLOmdtYV1N9uw5HrLVi/tdFoFWWCXxx9Riqo6pQ/vou4o1QIjw8W eoeiv4WDVfVSFFZfroDth2pSZcE+on4uDcFdxUTXUNJ6fA==
4KB4DFS71LEP8G8P8VT4CCUSQNL4CNCS.com. 86400 IN NSEC3 1 1 0 - 4KB4PTQQ5CTA7POCTGM7RUFC8B1RKTEU  NS DS RRSIG
4KB4DFS71LEP8G8P8VT4CCUSQNL4CNCS.com. 86400 IN RRSIG NSEC3 8 2 86400 20220824050324 20220817035324 32298 com. VuGioMRkwcuadRj+JJI51fVSvEWu/v15K9nbhaDsxhu1Z9Yf4Yq0HgjD lLiaNOGUHKPFB0uUzVqd+JjdQeyWMvUufCKwAp27ydUpvbFYEpM2MAox S2XrWRf6xYYsNnSpIWZWMXOQNQybVPnFyORBJKpyKkRpH8ZCHNIuEfnV Xw1L8D52M4dHiMbIiTN0c5UTIj8qmJQCYD2FT4nu4Wd4Aw==
;; Received 831 bytes from 192.12.94.30#53(e.gtld-servers.net) in 234 ms

www.github.com.		3600	IN	CNAME	github.com.
github.com.		60	IN	A	20.205.243.166
github.com.		900	IN	NS	dns1.p08.nsone.net.
github.com.		900	IN	NS	dns2.p08.nsone.net.
github.com.		900	IN	NS	dns3.p08.nsone.net.
github.com.		900	IN	NS	dns4.p08.nsone.net.
github.com.		900	IN	NS	ns-1283.awsdns-32.org.
github.com.		900	IN	NS	ns-1707.awsdns-21.co.uk.
github.com.		900	IN	NS	ns-421.awsdns-52.com.
github.com.		900	IN	NS	ns-520.awsdns-01.net.
;; Received 296 bytes from 205.251.194.8#53(ns-520.awsdns-01.net) in 209 ms

```

### 指定域名服务器查询

`dig @8.8.8.8 www.bytedance.com`

结果如下:

```

; <<>> DiG 9.10.6 <<>> @8.8.8.8 www.bytedance.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 23328
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;www.bytedance.com.		IN	A

;; ANSWER SECTION:
www.bytedance.com.	600	IN	CNAME	www.bytedance.com.edgesuite.net.
www.bytedance.com.edgesuite.net. 20054 IN CNAME	a1958.r.akamai.net.
a1958.r.akamai.net.	20	IN	A	23.211.136.80
a1958.r.akamai.net.	20	IN	A	23.211.136.98

;; Query time: 111 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Aug 19 17:15:59 CST 2022
;; MSG SIZE  rcvd: 152

```

### 查询所有记录

`$dig google.com any`

结果如下:

```

; <<>> DiG 9.10.6 <<>> google.com any
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55402
;; flags: qr rd ra; QUERY: 1, ANSWER: 26, AUTHORITY: 0, ADDITIONAL: 9

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;google.com.			IN	ANY

;; ANSWER SECTION:
google.com.		43	IN	A	172.253.125.138
google.com.		43	IN	A	172.253.125.102
google.com.		43	IN	A	172.253.125.139
google.com.		43	IN	A	172.253.125.101
google.com.		43	IN	A	172.253.125.100
google.com.		43	IN	A	172.253.125.113
google.com.		21581	IN	NS	ns1.google.com.
google.com.		21581	IN	NS	ns3.google.com.
google.com.		21581	IN	NS	ns2.google.com.
google.com.		21581	IN	NS	ns4.google.com.
google.com.		41	IN	SOA	ns1.google.com. dns-admin.google.com. 468416168 900 900 1800 60
google.com.		281	IN	MX	10 smtp.google.com.
google.com.		3581	IN	TXT	"docusign=1b0a6754-49b1-4db5-8540-d2c12664b289"
google.com.		3581	IN	TXT	"MS=E4A68B9AB2BB9670BCE15412F62916164C0B20BB"
google.com.		3581	IN	TXT	"apple-domain-verification=30afIBcvSuDV2PLX"
google.com.		3581	IN	TXT	"v=spf1 include:_spf.google.com ~all"
google.com.		3581	IN	TXT	"webexdomainverification.8YX6G=6e6922db-e3e6-4a36-904e-a805c28087fa"
google.com.		3581	IN	TXT	"google-site-verification=wD8N7i1JTNTkezJ49swvWW48f8_9xveREV4oB-0Hf5o"
google.com.		3581	IN	TXT	"atlassian-domain-verification=5YjTmWmjI92ewqkx2oXmBaD60Td9zWon9r6eakvHX6B77zzkFQto8PQ9QsKnbf4I"
google.com.		3581	IN	TXT	"globalsign-smime-dv=CDYX+XFHUw2wml6/Gb8+59BsH31KzUr6c1l2BPvqKX8="
google.com.		3581	IN	TXT	"docusign=05958488-4752-4ef2-95eb-aa7ba8a3bd0e"
google.com.		3581	IN	TXT	"google-site-verification=TV9-DBe4R80X4v0M4U_bd_J9cpOJM0nikft0jAgjmsQ"
google.com.		3581	IN	TXT	"facebook-domain-verification=22rm551cu4k0ab0bxsw536tlds4h95"
google.com.		281	IN	AAAA	2404:6800:4005:81c::200e
google.com.		21581	IN	TYPE65	\# 13 00010000010006026832026833
google.com.		21581	IN	CAA	0 issue "pki.goog"

;; ADDITIONAL SECTION:
ns1.google.com.		75020	IN	A	216.239.32.10
ns1.google.com.		67499	IN	AAAA	2001:4860:4802:32::a
ns3.google.com.		75020	IN	A	216.239.36.10
ns3.google.com.		67499	IN	AAAA	2001:4860:4802:36::a
ns2.google.com.		75020	IN	A	216.239.34.10
ns2.google.com.		67499	IN	AAAA	2001:4860:4802:34::a
ns4.google.com.		75020	IN	A	216.239.38.10
ns4.google.com.		67499	IN	AAAA	2001:4860:4802:38::a

;; Query time: 90 msec
;; SERVER: 30.30.30.30#53(30.30.30.30)
;; WHEN: Fri Aug 19 17:21:09 CST 2022
;; MSG SIZE  rcvd: 1302

```

### 查询 PTR 记录

`$dig -x 8.8.8.8`

结果如下:

```

; <<>> DiG 9.10.6 <<>> -x 8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12829
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;8.8.8.8.in-addr.arpa.		IN	PTR

;; ANSWER SECTION:
8.8.8.8.in-addr.arpa.	63145	IN	PTR	dns.google.

;; Query time: 66 msec
;; SERVER: 30.30.30.30#53(30.30.30.30)
;; WHEN: Fri Aug 19 17:22:55 CST 2022
;; MSG SIZE  rcvd: 73

```

## 参考资料

1. [RFC 1035](https://www.ietf.org/rfc/rfc1035.txt)
2. [DNS 的工作方式](https://www.cloudflare.com/zh-cn/learning/dns/what-is-dns/)
