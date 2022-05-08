# RISC-V 修包日志

## pytorch 修复心路历程

### 1. 分析 obs 构建错误信息

```
[ 2505s] [ 22%] Performing download step (git clone) for 'psimd'
[ 2528s] Cloning into 'psimd'...
[ 2539s] fatal: unable to access 'https://github.com/Maratyszcza/psimd.git/': Could not resolve host: github.com
[ 2539s] -- Had to git clone more than once:
[ 2539s]           3 times.
[ 2539s] CMake Error at /home/abuild/rpmbuild/BUILD/pytorch-1.6.0/build/confu-deps/psimd-download/psimd-prefix/tmp/psimd-gitclone.cmake:31 (message):
[ 2539s]   Failed to clone repository: 'https://github.com/Maratyszcza/psimd.git'
```

在群里问询，得知 obs 构建环境是断网的，但是我仔细观察项目中的 `updateSource.sh` 脚本，发现其已经为仓库 clone 了所有的 submodules 。

### 2. 尝试添加 submodules

带着这个疑问，我尝试 clone 仓库，并且运行 `updateSource.sh`

运行到一半我又意识到，既然都 update 了，为什么不贯彻到底呢

于是我去查询了 pytorch 最新的版本，并将源码包更新到了1.11.0

### 3. 准备第一次编译和修复

更新后，我们开启假设性原则，假设这个项目是全新的，从头开始。

因为担心老版本的patch文件与新版本的目录结构产生冲突，于是禁用之。

在这种情况下开始，编译报错如下：

```python
ModuleNotFoundError: No module named 'typing_extensions'
```

于是在 `BuildRequires` 中加入 `python-typing-extensions`

### 4. 为 `breakpad` 添加 `riscv64` 支持

#### 4.1 搭建修复环境

在编译环境搭建完整后，我对源码修修补补，多次尝试编译运行，得到了许多错误信息

```
third_party/breakpad/src/common/linux/memory_mapped_file.cc:74:7: error: ‘sys_fstat64’ was not declared in this scope
third_party/breakpad/src/common/linux/memory_mapped_file.cc:76:5: error: ‘sys_close’ was not declared in this scope
third_party/breakpad/src/common/linux/memory_mapped_file.cc:91:16: error: ‘sys_mmap’ was not declared in this scope
```

经我研究，这些没有定义的函数，都是系统中自带的系统调用，但是都带有多余的sys_前缀，并且都引入了 `linux_syscall_support.h` 这个头文件。

打开这个4k多行的文件，其中密密麻麻地定义了各种架构的 cpu 在 linux 上的系统调用，在这种情况下，用普通的文本修改器进行 patch 已经十分的不现实。

于是我开始着手下载 CLion ，准备使用 openEuler-riscv 作为远程工具链，在 IDE 的加持下处理这么复杂的问题。

#### 4.2 配置 CLion

设置 Toolchains

File -> Settings -> Build, Execution, Deployment -> Toolchains

选择 **+** 号添加 Remote Host ，填写 openEuler 的 ssh 信息，设置一下工具的绝对路径即可。

设置 CMake

File -> Settings -> Build, Execution, Deployment -> CMake

将 Toolchain 设置为刚才新增的 Remote Host ，然后就可以 Build 了

#### 4.3 设置 openEuler 为 user-mode 

由于项目过于庞大，所以使用 qemu-user 加速编译，为此，我写了一个脚本

```bash
#!/usr/bin/env bash
set -e

sudo /usr/sbin/modprobe nbd max_part=8
sudo qemu-nbd --connect=/dev/nbd0 YOUR_QCOW2_IMG_PATH
sudo /usr/sbin/fdisk -l /dev/nbd0
mkdir -p mnt
sudo mount /dev/nbd0p1 mnt
sudo mount -t proc /proc mnt/proc
sudo mount --rbind /sys mnt/sys
sudo mount --rbind /dev mnt/dev
sudo chroot mnt /bin/zsh
# you are in user-mode from here
# ...
# Note: you should close all files from qcow2 before you exit
# after exit
sudo umount -f -l mnt/dev
sudo umount -f -l mnt/sys
sudo umount -f -l mnt/proc
sudo umount -f -l mnt
sudo qemu-nbd --disconnect /dev/nbd0
sudo /usr/sbin/rmmod nbd
```

启动 qumu-user 后，需要手动开启 sshd 服务，手动配置 dns 服务器

#### 4.4 参考 aarch64 代码实现，为 breakpad 添加 riscv 支持

它们的部分内容比较近似，其代码也可直接复用

由于操作系统中 user 权限的应用并没有权限访问 pc 寄存器，并且 linux 并没有 auipc 的接口，所以部分寄存器的 dump 只能不了了之，等待上游更新。

### 5. 追踪后续上游项目进展

https://github.com/pytorch/pytorch/pull/75394

不！谷歌！你没有心！！！