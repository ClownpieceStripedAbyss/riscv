# RISC-V修包日志

## rubygem-puma修复心路历程

### 1. 分析obs构建错误信息

```
[  550s] /usr/lib/systemd/systemd-sysctl: error while loading shared libraries: libkmod.so.2: cannot open shared object file: No such file or directory
[ 1304s] /usr/bin/systemctl: error while loading shared libraries: libjson-c.so.5: cannot open shared object file: No such file or directory
```

### 2. 一些社会工程学（

这里发现同时少了两个包，分别是 `kmod` 和 `json-c` ，不过群友恰好在讨论找不到json-c的事情，据说是因为[ pjconf ](https://build.openeuler.org/project/prjconf/openEuler:Mainline:RISC-V)中去除了对 libjson-c 的依赖

```
Ignore: libjson-c.so.5()(64bit)
Ignore: libjson-c.so.5(JSONC_0.14)(64bit)
```

因此，本着最小改动的原则，我仅仅在 `BuildRequirements` 里添加了 `json-c`

### 3. 在线构建

在自己的obs仓库中加入 `json-c` 和 `kmod` ，使用obs在线进行自动构建

### 4.个人项目构建完成

修复十分顺利，仅仅在 `rubygem-puma.spec` 中加入 `json-c` 就修复成功了
