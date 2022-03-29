# RISC-V修包日志
## k8s修复心路历程
观测源码仓编译结果，发现Unsupported arch，于是翻找源码并且尝试加入linux/riscv64的支持，后报错依旧。

在网络上搜索k8s编译"Unsupported arch"后，得到如下搜索结果

http://liupeng0518.github.io/2019/05/15/k8s/deploy/%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91/

根据此篇教程进行源码的修补，报错更改为go not found

问题十分简单，在安装golang时并没有加入环境变量。
手动在
`/var/tmp/build-root/standard_riscv64-riscv64/etc/profile`

文件中加入go的PATH

```
export GOROOT=/usr/lib/golang
export GOPATH=$HOME/golang
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```
现在找得到go了



之后又出现报错
`go build runtime/cgo: invalid flag in go:cgo_ldflag: -Wl,-z,relro,-z,now`
尝试在spec中删除编译参数-Wl,-z,relro,-z,now
编译器不再报错，开始正常编译

之后go compiler报错：`Undefined parseCPUInfo`
在源码中寻找这一段，发现是因为cpuinfo的函数没有连接到parseCPUInfoRISCV的文件，故新建文件
`$ vim cpuinfo_riscv64.go`
并引入函数

```
package procfs

var parseCPUInfo = parseCPUInfoRISCV
```

遂正常编译



编译到一半报错硬盘空间不足，按照下面的教程实现了扩容qcow2
https://gitee.com/jinjuhan/open-euler-notes/blob/main/resize-qcow2.md
方便好用



obs远端和本地同时报错：找不到cgo

根据
https://github.com/golang/go/issues/36641

操作发现有cgo😓

尝试更改spec文件，添加
`export CC="riscv64-linux-gnu-gcc"`

无效

尝试直接修改go env
