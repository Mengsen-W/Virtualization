# SystemTap 安装及使用

1. ## 安装过程（centos 7）

   1. `yum`安装必要组件
      
      1. `yum install systemtap systemtap-runtime kernel-devel systemtap-sdt-devel`
      
   2. 查看内核版本
      
      1. `uname -rm`
      
      2. ```shell
         (base) [root@node0 home]# rpm -qa|grep kernel
         kernel-debuginfo-common-x86_64-3.10.0-1127.19.1.el7.x86_64
         kernel-debug-devel-3.10.0-1160.6.1.el7.x86_64
         kernel-tools-3.10.0-1160.6.1.el7.x86_64
         kernel-devel-3.10.0-1160.6.1.el7.x86_64
         kernel-debuginfo-3.10.0-1127.19.1.el7.x86_64
         kernel-3.10.0-957.el7.x86_64
         kernel-3.10.0-1127.19.1.el7.x86_64
         kernel-tools-libs-3.10.0-1160.6.1.el7.x86_64
         kernel-headers-3.10.0-1160.6.1.el7.x86_64
         kernel-3.10.0-1160.6.1.el7.x86_64
         kernel-devel-3.10.0-1127.19.1.el7.x86_64
         (base) [root@node0 home]# uname -rm
         3.10.0-1160.6.1.el7.x86_64 x86_64
         ```
      
         真尼玛坑爹，这两个版本都不一样，我按照`uname -rm`下的debug包版本全是错误的
      
         ```shell
         (base) [root@node0 home]# rpm -qa|grep kernel
         kernel-tools-3.10.0-1160.6.1.el7.x86_64
         kernel-devel-3.10.0-1160.6.1.el7.x86_64
         kernel-tools-libs-3.10.0-1160.6.1.el7.x86_64
         kernel-headers-3.10.0-1160.6.1.el7.x86_64
         kernel-3.10.0-1160.6.1.el7.x86_64
         (base) [root@node0 home]# uname -a
         Linux node0 3.10.0-1160.6.1.el7.x86_64 #1 SMP Tue Nov 17 13:59:11 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
         ```
      
         修复好了，应该版本号是一致的
      
   3. `stap-prep` 检查缺少内容

      1. `kernel-debuginfo`下载 http://debuginfo.centos.org/7/x86_64/
      2. `rpm -ivh` 解压包
      3. 再次使用 `stap-perp` 检查
      4. `dtrace --help`

   4. 测试`stap`命令

      1. `stap -ve 'probe begin{printf("Hello, World\n"); exit();}'`

      2. `stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'`
      
      3. ```shell
         (base) [root@node0 systemtap]# stap -ve 'probe begin{printf("Hello, World\n"); exit();}'
         Pass 1: parsed user script and 476 library scripts using 272288virt/69516res/3484shr/66136data kb, in 670usr/30sys/701real ms.
         Pass 2: analyzed script: 1 probe, 1 function, 0 embeds, 0 globals using 273740virt/71360res/3820shr/67588data kb, in 0usr/10sys/10real ms.
         Pass 3: using cached /root/.systemtap/cache/7d/stap_7de225d50de7509efffebbf4fabc454d_1020.c
         Pass 4: using cached /root/.systemtap/cache/7d/stap_7de225d50de7509efffebbf4fabc454d_1020.ko
         Pass 5: starting run.
         Hello, World
         Pass 5: run completed in 10usr/30sys/384real ms.
         
         (base) [root@node0 systemtap]# stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'
         Pass 1: parsed user script and 525 library scripts using 411904virt/207084res/3484shr/205752data kb, in 2250usr/120sys/2365real ms.
         Pass 2: analyzed script: 1 probe, 1 function, 7 embeds, 0 globals using 591064virt/381920res/4816shr/384912data kb, in 1910usr/490sys/2406real ms.
         Pass 3: using cached /root/.systemtap/cache/7e/stap_7e37194c4cb6bf8ae90f12a4163ca73a_2799.c
         Pass 4: using cached /root/.systemtap/cache/7e/stap_7e37194c4cb6bf8ae90f12a4163ca73a_2799.ko
         Pass 5: starting run.
         read performed
         Pass 5: run completed in 10usr/70sys/391real ms.
         ```

2. ## qemu log类型

   1. log（默认） –发送给QEMU日志系统每个事件的printf格式化字符串，然后系统写入stderr
   
   2. syslog –通过syslog发送的每个事件的printf格式化字符串
   
   3. simple–写入文件或fifo管道的每个事件的二进制数据流
   
   4. ftrace –发送给内核ftrace工具的每个事件的printf格式化字符串
   
   5. dtrace –通过dtrace或systemtap动态启用的用户空间探针标记
   
   6. ust –通过LTT-ng动态启用用户空间探针标记
   
3. ## 使用过程

   1. 使用脚本生成`stp`

      ```shell
      cd qemu
      ./scripts/tracetool.py --backends=dtrace --format=stap --binary /home/qemu/qemu-binary --target-type system --target-name x86_64 --group=all ./trace-events-all > qemu.stp
      
      ```

      

