---
title: go get 报错410 Gone
date: 2021-05-26 11:08:48
tags: go
---

### 现象
> go版本`go1.16.3`使用命令`go get xxx@xxx`拉取私有库时发现会报错410Gone
```bash
go get xxx@xxx: xxx: verifying module: xxx: reading https://goproxy.io/sumdb/sum.golang.org/lookup/xxx: 410 Gone
	server response: not found:xxx: unrecognized import path "xxx": https fetch: Get "xxx?go-get=1": dial tcp xxx.xx.xx.xx:443: connect: connection refused
```
> 看起来走了proxy代理。目测是`proxy`配置有问题。但是检查发现`GONOPROXY`和`GONOPROXY`配置都是OK的。
```bash
//异常环境变量
export GOPROXY=https://goproxy.cn,direct
export GONOPROXY=xxx.xxx.xxx
export GOPRIVATE=xxx.xxx.xxx
```
> 该配置在另外一台电脑正常工作。

### 解决方案
> 翻了无数网友的经验贴发现是`GONOSUMDB`也需要设置。查看了工作正常的机器和新机器的设置，的确有差异。加上`export GONOSUMDB=xxx.xxx.xxx`后工作正常
```bash
//正常环境变量
export GOPROXY=https://goproxy.cn,direct
export GONOPROXY=xxx.xxx.xxx
export GOPRIVATE=xxx.xxx.xxx
export GONOSUMDB=xxx.xxx.xxx
```

### 结论
> - 环境变量 `GOSUMDB` 可以用来配置你使用哪个校验服务器和公钥来做依赖包的校验.
> - 如果你的代码仓库或者模块是私有的，那么它的校验值不应该出现在互联网的公有数据库里面，但是我们本地编译的时候默认所有的依赖下载都会去尝试做校验，这样不仅会校验失败，更会泄漏一些私有仓库的路径等信息，我们可以使用`GONOSUMDB`这个环境变量来设置不做校验的代码仓库，它可以设置多个匹配路径，用逗号相隔. 举个例子`GONOSUMDB=*.example.com,xxx.io/private`

### proxy的环境变量说明
- https://goproxy.io/zh/docs/introduction.html
