# RISC-V修包日志

## rubygem-puma修复心路历程

### 1. 分析obs构建错误信息

```
[  550s] /usr/lib/systemd/systemd-sysctl: error while loading shared libraries: libkmod.so.2: cannot open shared object file: No such file or directory
[ 1304s] /usr/bin/systemctl: error while loading shared libraries: libjson-c.so.5: cannot open shared object file: No such file or directory
```

这里发现同时少了两个包，分别是kmod和json-c，不过群友恰好在讨论找不到json-c的事情，据说是因为[pjconf](https://build.openeuler.org/project/prjconf/openEuler:Mainline:RISC-V)中去除了对libjson-c的依赖

```
Ignore: libjson-c.so.5()(64bit)
Ignore: libjson-c.so.5(JSONC_0.14)(64bit)
```

