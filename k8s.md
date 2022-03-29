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

好像cgo找不到并不是一个严重的问题，即使作为一个error，它依然允许程序继续运行下去，出现了新的错误信息

```
+++ [0329 14:25:17] Building go targets for linux/riscv64:
[ 4595s]     cmd/gendocs
[ 4595s]     cmd/genkubedocs
[ 4595s]     cmd/genman
[ 4595s]     cmd/genyaml
[ 4910s] # k8s.io/kubernetes/cmd/genkubedocs
[ 4910s] runtime: pointer 0x3f1c143ff0 to unused region of span span.base()=0x3f1c142000 span.limit=0x3f1c143fe0 span.state=1
[ 4910s] runtime: found in object at *(0x3f375c6000+0x2d0)
[ 4910s] object=0x3f375c6000 s.base()=0x3f375c6000 s.limit=0x3f375e5400 s.spanclass=0 s.elemsize=131072 s.state=mSpanInUse
[ 4910s]  *(object+0) = 0x3f8980a9c4
[ 4910s]  *(object+8) = 0x42
[ 4910s]  *(object+16) = 0x4200900010001
[ 4910s]  *(object+24) = 0xffffffff
[ 4910s]  *(object+32) = 0xa93f60
[ 4910s]  *(object+40) = 0x20
[ 4910s]  *(object+48) = 0x0
[ 4910s]  *(object+56) = 0x176b2a
[ 4910s]  *(object+64) = 0x0
[ 4910s]  *(object+72) = 0x3f9014caf0
[ 4910s]  *(object+80) = 0x3f1c143f60
[ 4910s]  *(object+88) = 0x20
[ 4910s]  *(object+96) = 0x7ebc0a0
[ 4910s]  *(object+104) = 0x3f03988de0
[ 4910s]  *(object+112) = 0x2
[ 4910s]  *(object+120) = 0x2
[ 4910s]  *(object+128) = 0x3f8980aa06
[ 4910s]  *(object+136) = 0x47
[ 4910s]  *(object+144) = 0x4200900010001
[ 4910s]  *(object+152) = 0xffffffff
[ 4910s]  *(object+160) = 0xa93f80
[ 4910s]  *(object+168) = 0x28
[ 4910s]  *(object+176) = 0x0
[ 4910s]  *(object+184) = 0x176b2b
[ 4910s]  *(object+192) = 0x0
[ 4910s]  *(object+200) = 0x3f9014caf0
[ 4910s]  *(object+208) = 0x3f1c143f80
[ 4910s]  *(object+216) = 0x28
[ 4910s]  *(object+224) = 0x7ebc080
[ 4910s]  *(object+232) = 0x3f03988e20
[ 4910s]  *(object+240) = 0x2
[ 4910s]  *(object+248) = 0x2
[ 4910s]  *(object+256) = 0x3f8980aa4d
[ 4910s]  *(object+264) = 0x42
[ 4910s]  *(object+272) = 0x4200900010001
[ 4910s]  *(object+280) = 0xffffffff
[ 4910s]  *(object+288) = 0xa93fa8
[ 4910s]  *(object+296) = 0x18
[ 4910s]  *(object+304) = 0x0
[ 4910s]  *(object+312) = 0x176b2c
[ 4910s]  *(object+320) = 0x0
[ 4910s]  *(object+328) = 0x3f9014caf0
[ 4910s]  *(object+336) = 0x3f1c143fa8
[ 4910s]  *(object+344) = 0x18
[ 4910s]  *(object+352) = 0x7ebc058
[ 4910s]  *(object+360) = 0x3f03988e60
[ 4910s]  *(object+368) = 0x2
[ 4910s]  *(object+376) = 0x2
[ 4910s]  *(object+384) = 0x3f8980aa8f
[ 4910s]  *(object+392) = 0x4a
[ 4910s]  *(object+400) = 0x4200900010001
[ 4910s]  *(object+408) = 0xffffffff
[ 4910s]  *(object+416) = 0xa93fc0
[ 4910s]  *(object+424) = 0x14
[ 4910s]  *(object+432) = 0x0
[ 4910s]  *(object+440) = 0x176b2d
[ 4910s]  *(object+448) = 0x0
[ 4910s]  *(object+456) = 0x3f9014caf0
[ 4910s]  *(object+464) = 0x3f1c143fc0
[ 4910s]  *(object+472) = 0x14
[ 4910s]  *(object+480) = 0x7ebc040
[ 4910s]  *(object+488) = 0x3f03988ea0
[ 4910s]  *(object+496) = 0x2
[ 4910s]  *(object+504) = 0x2
[ 4910s]  *(object+512) = 0x3f8980aad9
[ 4910s]  *(object+520) = 0x4c
[ 4910s]  *(object+528) = 0x4200900010001
[ 4910s]  *(object+536) = 0xffffffff
[ 4910s]  *(object+544) = 0xa93fd8
[ 4910s]  *(object+552) = 0x18
[ 4910s]  *(object+560) = 0x0
[ 4910s]  *(object+568) = 0x176b2e
[ 4910s]  *(object+576) = 0x0
[ 4910s]  *(object+584) = 0x3f9014caf0
[ 4910s]  *(object+592) = 0x3f1c143fd8
[ 4910s]  *(object+600) = 0x18
[ 4910s]  *(object+608) = 0x7ebc028
[ 4910s]  *(object+616) = 0x3f03988ee0
[ 4910s]  *(object+624) = 0x2
[ 4910s]  *(object+632) = 0x2
[ 4910s]  *(object+640) = 0x3f8980ab25
[ 4910s]  *(object+648) = 0x49
[ 4910s]  *(object+656) = 0x4200900010001
[ 4910s]  *(object+664) = 0xffffffff
[ 4910s]  *(object+672) = 0xa93ff0
[ 4910s]  *(object+680) = 0x18
[ 4910s]  *(object+688) = 0x0
[ 4910s]  *(object+696) = 0x176b2f
[ 4910s]  *(object+704) = 0x0
[ 4910s]  *(object+712) = 0x3f9014caf0
[ 4910s]  *(object+720) = 0x3f1c143ff0 <==
[ 4910s]  *(object+728) = 0x18
[ 4910s]  *(object+736) = 0x7ebc010
[ 4910s]  *(object+744) = 0x3f03988f20
[ 4910s]  *(object+752) = 0x2
[ 4910s]  *(object+760) = 0x2
[ 4910s]  *(object+768) = 0x3f8980ab6e
[ 4910s]  *(object+776) = 0x42
[ 4910s]  *(object+784) = 0x4200900010001
[ 4910s]  *(object+792) = 0xffffffff
[ 4910s]  *(object+800) = 0xa94008
[ 4910s]  *(object+808) = 0x18
[ 4910s]  *(object+816) = 0x0
[ 4910s]  *(object+824) = 0x176b30
[ 4910s]  *(object+832) = 0x0
[ 4910s]  *(object+840) = 0x3f9014caf0
[ 4910s]  *(object+848) = 0x3f1c144008
[ 4910s]  *(object+856) = 0x18
[ 4910s]  *(object+864) = 0x7ebbff8
[ 4910s]  *(object+872) = 0x3f03988f60
[ 4910s]  *(object+880) = 0x2
[ 4910s]  *(object+888) = 0x2
[ 4910s]  *(object+896) = 0x3f8980abb0
[ 4910s]  *(object+904) = 0x41
[ 4910s]  *(object+912) = 0x4200900010001
[ 4910s]  *(object+920) = 0xffffffff
[ 4910s]  *(object+928) = 0xa94020
[ 4910s]  *(object+936) = 0x6c
[ 4910s]  *(object+944) = 0x0
[ 4910s]  *(object+952) = 0x176b31
[ 4910s]  *(object+960) = 0x0
[ 4910s]  *(object+968) = 0x3f9014caf0
[ 4910s]  *(object+976) = 0x3f1c144020
[ 4910s]  *(object+984) = 0x6c
[ 4910s]  *(object+992) = 0x7ebbfe0
[ 4910s]  *(object+1000) = 0x3f03988fa0
[ 4910s]  *(object+1008) = 0x5
[ 4910s]  *(object+1016) = 0x5
[ 4910s]  ...
[ 4910s] fatal error: found bad pointer in Go heap (incorrect use of unsafe or cgo?)
[ 4910s]
[ 4910s] runtime stack:
[ 4910s] runtime.throw(0x3049b9, 0x3e)
[ 4910s]        /usr/lib/golang/src/runtime/panic.go:1116 +0x64 fp=0x3f9008fe40 sp=0x3f9008fe18 pc=0x4aa1c
[ 4910s] runtime.badPointer(0x3f23f8db38, 0x3f1c143ff0, 0x3f375c6000, 0x2d0)
[ 4910s]        /usr/lib/golang/src/runtime/mbitmap.go:380 +0x288 fp=0x3f9008fe80 sp=0x3f9008fe40 pc=0x259e0
[ 4910s] runtime.findObject(0x3f1c143ff0, 0x3f375c6000, 0x2d0, 0x3fba82f1f8, 0x3f90052698, 0x19)
[ 4910s]        /usr/lib/golang/src/runtime/mbitmap.go:416 +0xb8 fp=0x3f9008feb0 sp=0x3f9008fe80 pc=0x25aa0
[ 4910s] runtime.scanobject(0x3f375c6000, 0x3f90052698)
[ 4910s]        /usr/lib/golang/src/runtime/mgcmark.go:1385 +0x2f8 fp=0x3f9008ff40 sp=0x3f9008feb0 pc=0x32758
[ 4910s] runtime.gcDrain(0x3f90052698, 0x7)
[ 4910s]        /usr/lib/golang/src/runtime/mgcmark.go:1143 +0x250 fp=0x3f9008ffa0 sp=0x3f9008ff40 pc=0x31df0
[ 4910s] runtime.gcBgMarkWorker.func2()
[ 4910s]        /usr/lib/golang/src/runtime/mgc.go:1981 +0x1b0 fp=0x3f9008ffd8 sp=0x3f9008ffa0 pc=0x78018
[ 4910s] runtime.systemstack(0x50128)
[ 4910s]        /usr/lib/golang/src/runtime/asm_riscv64.s:136 +0x7c fp=0x3f9008ffe0 sp=0x3f9008ffd8 pc=0x7de4c
[ 4910s] runtime.mstart()
[ 4910s]        /usr/lib/golang/src/runtime/proc.go:1116 fp=0x3f9008ffe0 sp=0x3f9008ffe0 pc=0x50128

```

看来是某个内存操作影响了栈指针导致报错，具体情况明天再试试
