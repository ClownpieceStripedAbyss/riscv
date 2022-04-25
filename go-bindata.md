# RISC-V修包日志

## go-bindata修复心路历程

### 1. 分析obs构建错误信息

```
[ 2012s] + go build -o bin/go-bindata github.com/go-bindata/go-bindata/go-bindata
[ 2013s] no required module provides package github.com/go-bindata/go-bindata/go-bindata: go.mod file not found in current directory or any parent directory; see 'go help modules'
[ 2013s] error: Bad exit status from /var/tmp/rpm-tmp.cj1QAl (%build)
[ 2013s] 
[ 2013s] 
[ 2013s] RPM build errors:
[ 2013s]     Bad exit status from /var/tmp/rpm-tmp.cj1QAl (%build)
[ 2013s] 
[ 2013s] oe-RISCV-worker29 failed "build go-bindata.spec" at Sat Apr  9 20:30:54 UTC 2022.
[ 2013s] 
```


### 2. 查找解决方案
针对 `go.mod file not found` 我在 Stackoverflow 上搜索到了如下[结果](https://stackoverflow.com/questions/66894200/error-message-go-go-mod-file-not-found-in-current-directory-or-any-parent-dire)

在设置 GO111MODULE=off 后，个人构建成功。

### 3. 查询并[讨论](https://gitee.com/openeuler-risc-v/go-bindata/pulls/1) go module 作用

#### **Emmmer**
go module 是一个官方的包管理器，它以代码仓库的链接，作为一个 go 项目依赖的地址，如

```
	github.com/kisielk/errcheck v1.2.0
	golang.org/x/lint v0.0.0-20191125180803-fdd1cda4f05f
```

就代表该项目编译时，下载这两个仓库作为依赖。

在编译 go-bindata 时，由于设置了 `GO111MODULE=on` ，同时源码包中没有文件 go.mod ，所以 go 会报错 `go.mod not found.`

关闭 go module ，会禁止 go 从 go.mod 中查找依赖，在项目不缺少依赖的情况下，就不会报错，最后编译成功。

综上所述 go module 的关闭只影响了编译时，应该不会影响运行时。

#### **kaokz**

[这个文章](https://maelvls.dev/go111module-everywhere/)说，对go1.16以上，如果要使用GOPATH解决包依赖，必须关闭GO111MODULE:

As of Go 1.16, the default behavior is GO111MODULE=on, meaning that if you want to keep using the old GOPATH way, you will have to force Go not to use the Go Modules feature:

export GO111MODULE=off

### 4. 最后的收尾工作

将地址改为 oerv中间仓 并且构建成功后，将obs提交到上游，观察其第二次构建成功后，整个构建流程结束。