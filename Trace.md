# QEMU--Trace

## 1.  Trace event

 1. 在源码目录的每个文件夹中都可以在`trace-event`文件中声明一组静态的trace event

    ```shell
    (base) [root@node0 migration]# ls -la | grep trace-events
    -rw-r--r--.   1 1000 1000  19770 Apr 29 00:49 trace-events
    ```

    

 2. 所有包含`trace-event`文件的子文件夹都必须被列在源码根目录的`Makefile.objs`的`trace-event-subdirs`项中

    ```makefile
    trace-events-subdirs =
    trace-events-subdirs += accel/kvm
    trace-events-subdirs += backends
    trace-events-subdirs += monitor
    // ...
    trace-events-subdirs += migration
    // ...
    ```

    这样在编译时，被列出的子文件夹的`trace-event`文件就会被`tracetool`脚本处理，生成相关代码

	3. 在子文件夹中，关于`trace`的文件会被自动生成

    ```shell
    -rw-r--r--.   1 root root 268880 Aug 13 09:35 trace.h # 跟踪事件的宏定义内联函数等
    -rw-r--r--.   1 root root 101737 Aug 13 09:35 trace.c # 跟踪事件的一些状态声明
    ```

4. 位于子文件中的`.c`源码会包含`trace.h`文件，另外一些共享的`trace`会在顶层文件夹中的`trace-event`文件中定义，但顶层文件生成`trace-root.h`文件

   ```c
   // ...
   #include "qemu/thread.h"
   #include "trace.h"
   #include "exec/target_page.h"
   // ...
   ```

   ```shell
   -rw-r--r--.   1 root root   22440 Aug 13 09:35 trace-root.c
   -rw-r--r--.   1 root root   65014 Aug 13 09:35 trace-root.h
   ```

5. `Trace ecent`分类
   1. `Nop` 编译器会把trace events全部优化掉，这样可以做到没有性能的损失
   2. `Log`后端直接将trace events输出到标准错误`stderr`，这就相当于把`trace events`都转化为了`debug`的`printf`，这是最简单的后端，而且可以和原本的使用`DPRINTF()`的代码一起使用。
   3. `Simpletrace`后端支持一般的使用场景，并且就在QEMU的源码树中。它可能不像特定平台或者第三方追踪后端那样强大，但是它一定是可移植的。
   4. `Ftrace`后端将trace数据写到`ftrace marker`中。这相当于将`trace events`发送到`ftrace环状缓冲区`中，然后你可以拿`qemu`的`trace`数据和`kernel`的`trace`数据(尤其是应用`KVM`时的`kvm.ko`内核模块)作比照来看了。
   5. `Syslog`后端用`POSIX`的`syslog API`发送`trace events`，日志被以特定的`LOG_DAEMON`设备或`LOG_PID`选项打开(所以`events`会被打上生成它们的`QEMU`进程的`pid`标签)。所有的`events`会被日志记录在`LOG_INFO`级别。
   6. `System-stap`

6. QEMU monitor 过程
   1. 没有`trace-file`选项，直接在`logfile <filename>`指定会直接写到这个文件里面