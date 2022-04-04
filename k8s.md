# RISC-V修包日志

## k8s修复心路历程

### 1. 跟据Unsupported arch添加缺少的riscv64参数

#### 1.1 定位问题

观测源码仓obs编译结果，发现报错

```
Unsupported host arch. Must be x86_64, 386, arm, arm64, s390x or ppc64le.
```

于是翻找源码并且尝试加入linux/riscv64的支持

在网络上搜索 "k8s 编译 Unsupported arch" 后，我得到如下搜索结果

http://liupeng0518.github.io/2019/05/15/k8s/deploy/%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91/

#### 1.2 加入环境变量

根据此篇教程进行源码的修补，报错更改为

```
go not found
```

问题十分简单，在安装golang时并没有加入环境变量。

```
$ vim /var/tmp/build-root/standard_riscv64-riscv64/etc/profile
```

加入go的PATH

```
export GOROOT=/usr/lib/golang
export GOPATH=$HOME/golang
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```
现在找得到go了

### 2. 修改错误的编译参数

#### 2.1 更改参数

之后又出现报错

```
go build runtime/cgo: invalid flag in go:cgo_ldflag: -Wl,-z,relro,-z,now
```

在 `kubernetes.spec` 中删除编译参数 `-Wl,-z,relro,-z,now` 后，编译器不再报错，开始正常编译

之后 `go compiler` 报错：`Undefined parseCPUInfo`

在源码中寻找这一段，发现是因为 `cpuinfo` 的函数没有连接到 `parseCPUInfoRISCV` 的文件

```
$ vim cpuinfo_riscv64.go
```

并引入函数

```
package procfs

var parseCPUInfo = parseCPUInfoRISCV
```

遂正常编译

#### 2.2 硬盘扩容

编译到一半报错硬盘空间不足，按照下面的教程实现了扩容qcow2

https://gitee.com/jinjuhan/open-euler-notes/blob/main/resize-qcow2.md

方便好用

#### 2.3 cgo 风波

obs 远端和本地同时报错：找不到 cgo

根据 https://github.com/golang/go/issues/36641

操作发现有 cgo 😓

尝试更改 `kubernetes.spec` 文件，添加

```
$ export CC="riscv64-linux-gnu-gcc"
```

无效

尝试直接修改 go env ，结果以失败告终

#### 2.4 开摆

好像cgo找不到并不是一个严重的问题，即使作为一个error，它依然允许程序继续运行下去，出现了新的错误信息

<details><pre>
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
</pre></details>

看来是某个内存操作影响了栈指针导致报错，具体情况明天再试试

根据 https://github.com/golang/go/issues/29362 在尝试添加 `GODEBUG=gcstoptheworld=1` 后，obs 自动构建出现 bug

<details><pre>
[ 4217s] # k8s.io/kubernetes/vendor/k8s.io/kubectl/pkg/cmd/clusterinfo
[ 4217s] fatal error: workbuf is empty
[ 4217s] 
[ 4217s] runtime stack:
[ 4217s] runtime.throw(0x8777fd, 0x10)
[ 4217s] 	/usr/lib/golang/src/runtime/panic.go:1116 +0x64
[ 4217s] runtime.(*workbuf).checknonempty(0x3fc50fd000)
[ 4217s] 	/usr/lib/golang/src/runtime/mgcwork.go:409 +0x50
[ 4217s] runtime.trygetfull(0x3fc50fe000)
[ 4217s] 	/usr/lib/golang/src/runtime/mgcwork.go:497 +0x4c
[ 4217s] runtime.(*gcWork).tryGet(0x3fc404ae98, 0x3fc404ae98)
[ 4217s] 	/usr/lib/golang/src/runtime/mgcwork.go:285 +0xa0
[ 4217s] runtime.gcDrain(0x3fc404ae98, 0x2)
[ 4217s] 	/usr/lib/golang/src/runtime/mgcmark.go:1130 +0x360
[ 4217s] runtime.gcBgMarkWorker.func2()
[ 4217s] 	/usr/lib/golang/src/runtime/mgc.go:1977 +0x154
[ 4217s] runtime.systemstack(0x50910)
[ 4217s] 	/usr/lib/golang/src/runtime/asm_riscv64.s:136 +0x7c
[ 4217s] runtime.mstart()
[ 4217s] 	/usr/lib/golang/src/runtime/proc.go:1116
[ 4217s] 
[ 4217s] goroutine 1 [runnable]:
[ 4217s] cmd/internal/obj.(*Link).LookupABIInit(0x3fc41751e0, 0x3fc50c9950, 0x48, 0x867601, 0x3fc4a30b58, 0x3fc4ee1ee0)
[ 4217s] 	/usr/lib/golang/src/cmd/internal/obj/sym.go:106 +0x194
[ 4217s] cmd/compile/internal/types.(*Sym).Linksym(0x3fc50f47e0, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/types/sym.go:92 +0xb4
[ 4217s] cmd/compile/internal/gc.(*importReader).symIdx(0x3fc50b8eb0, 0x3fc50f47e0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:702 +0x48
[ 4217s] cmd/compile/internal/gc.(*importReader).funcExt(0x3fc50b8eb0, 0x3fc50eb2c0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:671 +0x78
[ 4217s] cmd/compile/internal/gc.(*importReader).methExt(0x3fc50b8eb0, 0x3fc50f27c0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:693 +0x8c
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc50b8eb0, 0x3fc44081e0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:351 +0x70c
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc44081e0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc50b8e60, 0x3fc50b8e60)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x530, 0x3fc50e1020)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc50b8e10, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc50b8e10, 0x3fc50b8e10)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:522 +0xaa8
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x535, 0x3fc812af21)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc5064320, 0x3fc50e8d20)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc5064320, 0x3fc5064320)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:545 +0x674
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x5c8, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc50642d0, 0x33060000000be)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc50642d0, 0x3fc44170e0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:314 +0x2d4
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc44170e0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc5064280, 0x3fc5064280)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x1bc, 0x3fc812a7f7)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc5064230, 0x3fc5060c60)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc5064230, 0x3fc5064230)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:545 +0x674
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x1c1, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc50641e0, 0x26060000000be)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc50641e0, 0x3fc4416780)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:314 +0x2d4
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc4416780)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc5064190, 0x3fc5064190)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x1d9, 0x0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc5064140, 0x2)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc5064140, 0x3fc5064140)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:524 +0xae4
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x17c2b, 0x3fc8132e7d)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc50640f0, 0x3fc5060ba0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc50640f0, 0x3fc50640f0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:545 +0x674
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x17c77, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc50640a0, 0xb62060000000be)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc50640a0, 0x3fc44062d0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:314 +0x2d4
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc44062d0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc5064050, 0x3fc5064050)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x17e0b, 0x3fc812bf42)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc5064000, 0x3fc49147e0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc5064000, 0x3fc5064000)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:545 +0x674
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc448e040, 0x1b22b, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500ff90, 0xe75060000000be)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc500ff90, 0x3fc4406870)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:314 +0x2d4
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc4406870)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500ff40, 0x3fc500ff40)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x4886d, 0x3fc3e6f3cc)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500fef0, 0x3fc5060a20)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500fef0, 0x3fc500fef0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:545 +0x674
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x703e7, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500fea0, 0x1fb060000000b5)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc500fea0, 0x3fc46f44b0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:314 +0x2d4
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc46f44b0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500fe50, 0x3fc500fe50)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x343df, 0x3fc3e6753a)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500fe00, 0x3fc49147e0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500fe00, 0x3fc500fe00)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:545 +0x674
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x343ed, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500fdb0, 0x26a060000000b5)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc500fdb0, 0x3fc46f40f0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:314 +0x2d4
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc46f40f0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500fd60, 0x3fc500fd60)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x1e880, 0x3fc44dc9c0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500fd10, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500fd10, 0x3fc500fd10)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:522 +0xaa8
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x1e887, 0x3fc3e61300)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500fbd0, 0x3fc50608a0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).param(0x3fc500fbd0, 0x3fc5062500)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:630 +0xa8
[ 4217s] cmd/compile/internal/gc.(*importReader).paramList(0x3fc500fbd0, 0x40, 0x3feeea9108, 0x3fc50624c0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:621 +0x80
[ 4217s] cmd/compile/internal/gc.(*importReader).signature(0x3fc500fbd0, 0x3fc50624c0, 0x9)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:608 +0x30
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500fbd0, 0x3fc500fbd0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:583 +0x2e0
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x1e896, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500fb80, 0x28060000000bd)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc500fb80, 0x3fc44dbe00)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:314 +0x2d4
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc44dbe00)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500fb30, 0x3fc500fb30)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x116c4, 0x3fc5062440)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500f180, 0x0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).param(0x3fc500f180, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:630 +0xa8
[ 4217s] cmd/compile/internal/gc.(*importReader).paramList(0x3fc500f180, 0x3fc4f55ea8, 0x1, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:621 +0x80
[ 4217s] cmd/compile/internal/gc.(*importReader).signature(0x3fc500f180, 0x3fc50623c0, 0x3fc504f2c0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:609 +0x58
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc500f180, 0x3fc44db860)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:327 +0x3cc
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc44db860)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500f130, 0x3fc500f130)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:515 +0xa90
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x6be2, 0x3fc5004fc0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc500f0e0, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc500f0e0, 0x3fc500f0e0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:522 +0xaa8
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x6be9, 0x3fc3e51edd)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc4e89ef0, 0x3fc50406c0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).typ1(0x3fc4e89ef0, 0x3fc4e89ef0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:545 +0x674
[ 4217s] cmd/compile/internal/gc.(*iimporter).typAt(0x3fc40ed080, 0x6d9a, 0x1)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:494 +0x100
[ 4217s] cmd/compile/internal/gc.(*importReader).typ(0x3fc4e89ea0, 0x7906000000098)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:485 +0x40
[ 4217s] cmd/compile/internal/gc.(*importReader).doDecl(0x3fc4e89ea0, 0x3fc44da3c0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:314 +0x2d4
[ 4217s] cmd/compile/internal/gc.expandDecl(0x3fc44da3c0)
[ 4217s] 	/usr/lib/golang/src/cmd/compile/internal/gc/iimport.go:52 +0x90
[ 4236s] !!! [0330 09:07:57] Call tree:
[ 4236s] !!! [0330 09:07:57]  1: /home/abuild/rpmbuild/BUILD/kubernetes-1.20.2/src/k8s.io/kubernetes/hack/lib/golang.sh:717 kube::golang::build_some_binaries(...)
[ 4236s] !!! [0330 09:07:57]  2: /home/abuild/rpmbuild/BUILD/kubernetes-1.20.2/src/k8s.io/kubernetes/hack/lib/golang.sh:873 kube::golang::build_binaries_for_platform(...)
[ 4236s] !!! [0330 09:07:57]  3: hack/make-rules/build.sh:27 kube::golang::build_binaries(...)
[ 4237s] !!! [0330 09:07:58] Call tree:
[ 4237s] !!! [0330 09:07:58]  1: hack/make-rules/build.sh:27 kube::golang::build_binaries(...)
[ 4237s] !!! [0330 09:07:58] Call tree:
[ 4237s] !!! [0330 09:07:58]  1: hack/make-rules/build.sh:27 kube::golang::build_binaries(...)
[ 4237s] make: *** [Makefile:93: all] Error 1
[ 4237s] error: Bad exit status from /var/tmp/rpm-tmp.Cig2Xx (%build)
[ 4237s] 
[ 4237s] 
[ 4237s] RPM build errors:
[ 4237s]     bad date in %changelog: The Mar 23 2021 wangfengtu <wangfengtu@huawei.com> - 1.20.2-4
[ 4237s]     Bad exit status from /var/tmp/rpm-tmp.Cig2Xx (%build)
[ 4237s] 
[ 4237s] oe-RISCV-worker54-home failed "build kubernetes.spec" at Wed Mar 30 09:07:58 UTC 2022.
</pre></details>

### 3 更新 golang

找了一大圈子，感觉错误原因十分模糊，于是查看 go 的版本，发现为 1.15.5

因为是 go 编译时报错，怀疑可能是版本过低部分功能并不稳定，尝试将其更新到 1.18

```
# misc/cgo/testtls.test
/root/goroot/pkg/tool/linux_riscv64/link: running gcc failed: exit status 1
/usr/bin/ld: cannot find -latomic
collect2: error: ld returned 1 exit status

FAIL	misc/cgo/testtls [build failed]
2022/03/30 14:23:37 Failed: exit status 2
PASS
ok  	misc/cgo/nocgo	0.119s
PASS
ok  	misc/cgo/nocgo	0.178s
# misc/cgo/nocgo.test
/root/goroot/pkg/tool/linux_riscv64/link: running gcc failed: exit status 1
/usr/bin/ld: cannot find -latomic
collect2: error: ld returned 1 exit status

FAIL	misc/cgo/nocgo [build failed]
2022/03/30 14:23:51 Failed: exit status 2
skipped due to earlier error
go tool dist: FAILED
```

结果编译报错找不到 `atomic` 包

真的服了，然后我一方面通过 obs 在编译环境上安装，一方面在本地环境安装，防止出错

obs构建 `libatomic_ops` 把我的 golang 删了，我又不缺那点内存😄

在多次找不到 `libatomic` 后，我把 go env 中的 `CGO_ENABLE=1` 改成0

编译成功，成功安装了最新的go1.18

现在尝试使用go1.18对kubernetes进行编译

```
[ 5364s] Generating digest list: /usr/lib/rpm/brp-digest-list /home/abuild/rpmbuild/BUILDROOT/kubernetes-1.20.2-7.oe1.riscv64
[ 5365s] /usr/lib/rpm/brp-digest-list: line 32: gen_digest_lists: command not found
[ 5367s] Provides: kubernetes-kubelet = 1.20.2-7.oe1 kubernetes-kubelet(riscv-64) = 1.20.2-7.oe1
[ 5367s] Requires(rpmlib): rpmlib(CompressedFileNames) <= 3.0.4-1 rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1
[ 5367s] Requires: ld-linux-riscv64-lp64d.so.1()(64bit) ld-linux-riscv64-lp64d.so.1(GLIBC_2.27)(64bit) libc.so.6()(64bit) libc.so.6(GLIBC_2.27)(64bit) libdl.so.2()(64bit) libdl.so.2(GLIBC_2.27)(64bit) libpthread.so.0()(64bit) libpthread.so.0(GLIBC_2.27)(64bit) rtld(GNU_HASH)
[ 5367s] Processing files: kubernetes-help-1.20.2-7.oe1.riscv64
[ 5367s] Generating digest list: /usr/lib/rpm/brp-digest-list /home/abuild/rpmbuild/BUILDROOT/kubernetes-1.20.2-7.oe1.riscv64
[ 5367s] /usr/lib/rpm/brp-digest-list: line 32: gen_digest_lists: command not found
[ 5374s] Provides: kubernetes-help = 1.20.2-7.oe1 kubernetes-help(riscv-64) = 1.20.2-7.oe1
[ 5374s] Requires(rpmlib): rpmlib(CompressedFileNames) <= 3.0.4-1 rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1
[ 5374s] Checking for unpackaged file(s): /usr/lib/rpm/check-files /home/abuild/rpmbuild/BUILDROOT/kubernetes-1.20.2-7.oe1.riscv64
[ 5381s] Wrote: /home/abuild/rpmbuild/SRPMS/kubernetes-1.20.2-7.oe1.src.rpm
[ 5381s] Wrote: /home/abuild/rpmbuild/RPMS/riscv64/kubernetes-1.20.2-7.oe1.riscv64.rpm
[ 5382s] Wrote: /home/abuild/rpmbuild/RPMS/riscv64/kubernetes-help-1.20.2-7.oe1.riscv64.rpm
[ 5399s] Wrote: /home/abuild/rpmbuild/RPMS/riscv64/kubernetes-client-1.20.2-7.oe1.riscv64.rpm
[ 5399s] Wrote: /home/abuild/rpmbuild/RPMS/riscv64/kubernetes-kubeadm-1.20.2-7.oe1.riscv64.rpm
[ 5422s] Wrote: /home/abuild/rpmbuild/RPMS/riscv64/kubernetes-kubelet-1.20.2-7.oe1.riscv64.rpm
[ 5437s] Wrote: /home/abuild/rpmbuild/RPMS/riscv64/kubernetes-node-1.20.2-7.oe1.riscv64.rpm
[ 5488s] Wrote: /home/abuild/rpmbuild/RPMS/riscv64/kubernetes-master-1.20.2-7.oe1.riscv64.rpm
[ 5488s] Executing(%clean): /bin/bash -e /var/tmp/rpm-tmp.WtoruY
[ 5488s] + umask 022
[ 5488s] + cd /home/abuild/rpmbuild/BUILD
[ 5488s] + cd kubernetes-1.20.2
[ 5488s] + /usr/bin/rm -rf /home/abuild/rpmbuild/BUILDROOT/kubernetes-1.20.2-7.oe1.riscv64
[ 5489s] + RPM_EC=0
[ 5489s] ++ jobs -p
[ 5489s] + exit 0
[ 5491s] ... checking for files with abuild user/group
[ 5491s] 
[ 5491s] openEuler-RISCV-rare finished "build kubernetes.spec" at Fri Apr  1 06:05:10 UTC 2022.
[ 5491s] 
```

编译成功，由于mainline没有go1.18，故尝试使用1.17.3进行在线构建

好消息，1.17.3构建成功

现在开始进行PR的合并，先去conf仓库添加包的支持

fork包到本地仓库，加入自己的patch、spec和changelog

将其导入到obs中进行构建，等待结果

