# RISC-V 修包日志

## lldb 修复心路历程

### 1. 分析 obs 构建错误信息

初步判断是没有加入编译参数 -latomic

#### 在。spec文件里加入编译参数 -latomic
```
%ifarch riscv64
LDFLAGS="%{__global_ldflags} -latomic -lpthread -ldl"
%else
LDFLAGS="%{__global_ldflags} -lpthread -ldl"
%endif
```
加入时需要注意针对某个架构进行最小修改

### 2. 开始针对obs编译报错，修改和添加riscv支持

我首先进行编译，发现lldb-server部分报错，遂尝试在cmake中关闭这部分的功能，然后lldb成功出包，使用时发现无法调试，因为无法连接到lldb-server

不得已，恢复 lldb-server 的构建，然后在报错中寻找undefied error，并且仿照ARM64的支持，参考riscv寄存器的分布添加了riscv整数的支持，暂未添加浮点数和其他寄存器。

### 3. 尝试修改代码，完整适配（直接向上游提交）

在修改到Clion不报错后，尝试编译，这次，编译成功且lldb能尝试启动lldb-server，但是无法完整地处理信息，lldb报错