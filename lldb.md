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


```log
lldb             Processing command: r
lldb             HandleCommand, cmd_obj : 'process launch'
lldb             HandleCommand, (revised) command_string: 'process launch -c/bin/zsh --'
lldb             HandleCommand, wants_raw_input:'False'
lldb             HandleCommand, command line after removing command name(s): '-c/bin/zsh --'
lldb             Target::Launch() called for /root/main
lldb             Target::Launch the process instance doesn't currently exist.
lldb             have platform=true, platform_sp->IsHost()=true, default_to_use_pty=true
lldb             at least one of stdin/stdout/stderr was not set, evaluating default handling
lldb             target stdin='(empty)', target stdout='(empty)', stderr='(empty)'
lldb             Generating a pty to use for stdin/out/err
lldb             0x5a6b10 Listener::Listener('lldb.Target.Launch.hijack')
lldb             Target::Launch asking the platform to debug the process
lldb             target 0x00000000005B20A0
lldb             having target create process with gdb-remote plugin
lldb             0x00000000006241F0 Broadcaster::Broadcaster("lldb.process")
lldb             0x00000000006242A8 Broadcaster::Broadcaster("lldb.process.internal_state_broadcaster")
lldb             0x00000000006242D8 Broadcaster::Broadcaster("lldb.process.internal_state_control_broadcaster")
lldb             0x5a69f0 Listener::Listener('lldb.process.internal_state_listener')
lldb             0x0000000000624728 Broadcaster::Broadcaster("process.stdio")
lldb             0x0000000000624728 Communication::Communication (name = process.stdio)
lldb             0x43f530 Listener::StartListeningForEvents (broadcaster = 0x6241f0, mask = 0x0000002d) acquired_mask = 0x0000002d for lldb.Debugger
lldb             0x6241b8 Process::Process()
lldb             0x43f530 Listener::StartListeningForEvents (broadcaster = 0x6241f0, mask = 0x0000003f) acquired_mask = 0x0000003f for lldb.Debugger
lldb             0x5a69f0 Listener::StartListeningForEvents (broadcaster = 0x6242a8, mask = 0x00000003) acquired_mask = 0x00000003 for lldb.process.internal_state_listener
lldb             0x5a69f0 Listener::StartListeningForEvents (broadcaster = 0x6242d8, mask = 0x00000007) acquired_mask = 0x00000007 for lldb.process.internal_state_listener
lldb             0x0000000000624C88 Broadcaster::Broadcaster("gdb-remote.client")
lldb             0x0000000000624C88 Communication::Communication (name = gdb-remote.client)
lldb             0x0000000000625220 Broadcaster::Broadcaster("lldb.process.gdb-remote.async-broadcaster")
lldb             0x582f10 Listener::Listener('lldb.process.gdb-remote.async-listener')
lldb             0x582f10 Listener::StartListeningForEvents (broadcaster = 0x625220, mask = 0x00000003) acquired_mask = 0x00000003 for lldb.process.gdb-remote.async-listener
lldb             0x582f10 Listener::StartListeningForEvents (broadcaster = 0x624c88, mask = 0x00000004) acquired_mask = 0x00000004 for lldb.process.gdb-remote.async-listener
lldb             successfully created process
lldb             0x00000000005B2E88 Broadcaster("lldb.process")::HijackBroadcaster (listener("lldb.Target.Launch.hijack")=0x00000000005A6B10)
lldb             launching process with the following file actions:
lldb             file action: open fd 0 with '/dev/pts/2', OFLAGS = 0x100
lldb             file action: open fd 1 with '/dev/pts/2', OFLAGS = 0x141
lldb             file action: open fd 2 with '/dev/pts/2', OFLAGS = 0x141
lldb             0x6414e0 Listener::Listener('LaunchEventHijack')
lldb             0x00000000005B2E88 Broadcaster("lldb.process")::HijackBroadcaster (listener("LaunchEventHijack")=0x00000000006414E0)
lldb             0x0000000000624728 Communication::Disconnect ()
lldb             Process::SetPublicState (state = launching, restarted = 0)
lldb             shlib dir -> /root/git/llvm-project/build/lib/
lldb             HostInfo::ComputePathRelativeToLibrary() attempting to derive the path /bin relative to liblldb install path: /root/git/llvm-project/build/lib
lldb             HostInfo::ComputePathRelativeToLibrary() derived the path as: /root/git/llvm-project/build/bin
lldb             support exe dir -> /root/git/llvm-project/build/bin/
ait4(pid=3367)>  thread created
lldb             started monitoring child process.
ait4(pid=3367)>  pid = 3367
ait4(pid=3367)>  ::waitpid(3367, &status, 0)...
lldb             0x6438c0 ConnectionFileDescriptor::ConnectionFileDescriptor (fd = 6, owns_fd = 1)
lldb             0x6438c0 ConnectionFileDescriptor::CloseCommandPipe()
lldb             0x6438c0 ConnectionFileDescriptor::OpenCommandPipe() - success readfd=8 writefd=9
lldb             0x0000000000624C88 Communication::Disconnect ()
```
<details><pre>
```log
b-remote.async>  thread created
b-remote.async>  this = 0x0000000000582F10, timeout = <infinite> for lldb.process.gdb-remote.async-listener
lldb             0x0000000000624C88 Communication::Write (src = 0x0000003FFF90EF33, src_len = 1) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x3fff90ef33, src_len = 1)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x3fff90ef33, src_len = 1) => 1 (error = (null))
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000640EB0, src_len = 19) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x640eb0, src_len = 19)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x640eb0, src_len = 19) => 19 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90C960, dst_len = 8192, timeout = 6000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 6000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90c960, dst_len = 8192) => 1, error = (null)
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90CBC0, dst_len = 8192, timeout = 6000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 6000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90cbc0, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000003FFF90C853, src_len = 1) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x3fff90c853, src_len = 1)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x3fff90c853, src_len = 1) => 1 (error = (null))
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000646520, src_len = 86) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x646520, src_len = 86)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x646520, src_len = 86) => 86 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90C810, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90c810, dst_len = 8192) => 250, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000645880, src_len = 26) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x645880, src_len = 26)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x645880, src_len = 26) => 26 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90CC60, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90cc60, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000645880, src_len = 27) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x645880, src_len = 27)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x645880, src_len = 27) => 27 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90CC60, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90cc60, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000003FFF90E988, src_len = 13) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x3fff90e988, src_len = 13)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x3fff90e988, src_len = 13) => 13 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90C8A0, dst_len = 8192, timeout = 10000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 10000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90c8a0, dst_len = 8192) => 272, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000003FFF90ED28, src_len = 10) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x3fff90ed28, src_len = 10)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x3fff90ed28, src_len = 10) => 10 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90CC40, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90cc40, dst_len = 8192) => 17, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000641AC0, src_len = 27) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x641ac0, src_len = 27)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x641ac0, src_len = 27) => 27 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90CC60, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90cc60, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000640EB0, src_len = 23) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x640eb0, src_len = 23)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x640eb0, src_len = 23) => 23 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90CC60, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90cc60, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000644040, src_len = 34) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x644040, src_len = 34)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x644040, src_len = 34) => 34 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D0E0, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d0e0, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000644840, src_len = 35) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x644840, src_len = 35)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x644840, src_len = 35) => 35 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D0E0, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d0e0, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000644840, src_len = 35) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x644840, src_len = 35)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x644840, src_len = 35) => 35 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D0E0, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d0e0, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000640EB0, src_len = 21) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x640eb0, src_len = 21)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x640eb0, src_len = 21) => 21 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D180, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d180, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000640EB0, src_len = 23) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x640eb0, src_len = 23)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x640eb0, src_len = 23) => 23 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D180, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d180, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000640EB0, src_len = 23) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x640eb0, src_len = 23)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x640eb0, src_len = 23) => 23 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D120, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d120, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000646020, src_len = 26) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x646020, src_len = 26)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x646020, src_len = 26) => 26 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006460D0, src_len = 33) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6460d0, src_len = 33)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6460d0, src_len = 33) => 33 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000642450, src_len = 29) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x642450, src_len = 29)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x642450, src_len = 29) => 29 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000645D90, src_len = 42) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x645d90, src_len = 42)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x645d90, src_len = 42) => 42 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006460D0, src_len = 31) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6460d0, src_len = 31)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6460d0, src_len = 31) => 31 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000642450, src_len = 27) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x642450, src_len = 27)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x642450, src_len = 27) => 27 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000645D50, src_len = 52) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x645d50, src_len = 52)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x645d50, src_len = 52) => 52 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000645D50, src_len = 53) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x645d50, src_len = 53)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x645d50, src_len = 53) => 53 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000642450, src_len = 24) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x642450, src_len = 24)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x642450, src_len = 24) => 24 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000645D50, src_len = 48) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x645d50, src_len = 48)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x645d50, src_len = 48) => 48 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000646520, src_len = 81) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x646520, src_len = 81)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x646520, src_len = 81) => 81 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000642450, src_len = 27) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x642450, src_len = 27)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x642450, src_len = 27) => 27 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000647840, src_len = 3063) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x647840, src_len = 3063)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x647840, src_len = 3063) => 3063 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006460D0, src_len = 35) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6460d0, src_len = 35)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6460d0, src_len = 35) => 35 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000645D50, src_len = 48) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x645d50, src_len = 48)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x645d50, src_len = 48) => 48 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006460D0, src_len = 31) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6460d0, src_len = 31)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6460d0, src_len = 31) => 31 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000641AF0, src_len = 43) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x641af0, src_len = 43)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x641af0, src_len = 43) => 43 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006460D0, src_len = 37) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6460d0, src_len = 37)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6460d0, src_len = 37) => 37 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000642450, src_len = 24) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x642450, src_len = 24)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x642450, src_len = 24) => 24 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006460D0, src_len = 31) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6460d0, src_len = 31)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6460d0, src_len = 31) => 31 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006460D0, src_len = 36) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6460d0, src_len = 36)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6460d0, src_len = 36) => 36 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006460D0, src_len = 39) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6460d0, src_len = 39)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6460d0, src_len = 39) => 39 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006436F0, src_len = 62) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6436f0, src_len = 62)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6436f0, src_len = 62) => 62 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x00000000006439C0, src_len = 58) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x6439c0, src_len = 58)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x6439c0, src_len = 58) => 58 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D070, dst_len = 8192, timeout = 5000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 5000000 us
```
</pre></details>

```log
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90d070, dst_len = 8192) => 6, error = (null)
lldb             0x0000000000624C88 Communication::Write (src = 0x0000000000642450, src_len = 29) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x642450, src_len = 29)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x642450, src_len = 29) => 29 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90D020, dst_len = 8192, timeout = 10000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 10000000 us
lldb             0x0000000000624C88 Communication::Write (src = 0x0000003FFF90CE38, src_len = 6) connection = 0x00000000006438C0
lldb             0x6438c0 ConnectionFileDescriptor::Write (src = 0x3fff90ce38, src_len = 6)
lldb             0x6438c0 ConnectionFileDescriptor::Write(fd = 6, src = 0x3fff90ce38, src_len = 6) => 6 (error = (null))
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90ADE0, dst_len = 8192, timeout = 10000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 10000000 us
lldb             this = 0x0000000000624C88, dst = 0x0000003FFF90ADE0, dst_len = 8192, timeout = 10000000 us, connection = 0x00000000006438C0
lldb             this = 0x00000000006438C0, timeout = 10000000 us
lldb             0x6438c0 ConnectionFileDescriptor::Read()  fd = 6, dst = 0x3fff90ade0, dst_len = 8192) => 0, error = Connection reset by peer
lldb             0x0000000000624C88 Communication::Disconnect ()
lldb             0x6438c0 ConnectionFileDescriptor::Disconnect ()
lldb             0x0000000000624C88 Communication::Disconnect ()
lldb             0x6438c0 ConnectionFileDescriptor::Disconnect ()
lldb             0x6438c0 ConnectionFileDescriptor::Disconnect(): Nothing to disconnect
ait4(pid=3367)>  ::waitpid(3367, &status, 0) => pid = 3367, status = 0x86
lldb             0x0000000000624C88 Communication::Disconnect ()
lldb             0x6438c0 ConnectionFileDescriptor::Disconnect ()
lldb             0x6438c0 ConnectionFileDescriptor::Disconnect(): Nothing to disconnect
lldb             0x00000000005B2E88 Broadcaster("lldb.process")::RestoreBroadcaster (about to pop listener("LaunchEventHijack")=0x00000000006414E0)
lldb             0x6414e0 Listener::Clear('LaunchEventHijack')
lldb             0x6414e0 Listener::~Listener('LaunchEventHijack')
lldb             'A' packet returned an error: -1
lldb             HandleCommand, command did not succeed
```
```log
error: 'A' packet returned an error: -1
```
```log
(lldb) lldb             this = 0x0000000000582C48, timeout = <infinite>
ait4(pid=3367)>  Process::SetExitStatus (status=-1 (0xffffffff), description="debugserver died with signal SIGABRT")
ait4(pid=3367)>  Process::SetPrivateState (exited)
ait4(pid=3367)>  Process::SetPrivateState (exited) stop_id = 1
ait4(pid=3367)>  0x632368 Broadcaster("lldb.process.internal_state_broadcaster")::BroadcastEvent (event_sp = {0x3fa8001210 Event: broadcaster = 0x6242a8 (lldb.process.internal_state_broadcaster), type = 0x00000001, data = { process = 0x6241b8 (pid = 0), state = exited}}, unique =0) hijack = (nil)
ait4(pid=3367)>  0x5a69f0 Listener('lldb.process.internal_state_listener')::AddEvent (event_sp = {0x3fa8001210})
ait4(pid=3367)>  0x0000000000624C88 Communication::Disconnect ()
ait4(pid=3367)>  0x6438c0 ConnectionFileDescriptor::Disconnect ()
ait4(pid=3367)>  0x6438c0 ConnectionFileDescriptor::Disconnect(): Nothing to disconnect
ait4(pid=3367)>  pid = 3367 thread exiting...
```
