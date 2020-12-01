# SystemTap 安装及使用

1. ## 安装过程（centos 7）

   1. `yum`安装必要组件
      
      1. `yum install systemtap systemtap-runtime kernel-devel`
      
   2. 查看内核版本
      
      1. `uname -rm`
      
   3. `stap-prep` 检查缺少内容

      1. `kernel-debuginfo`下载 http://debuginfo.centos.org/7/x86_64/
      2. `rpm -ivh` 解压包
      3. 再次使用 `stap-perp` 检查

   4. 测试`stap`命令

      1. `stap -ve 'probe begin{printf("Hello, World\n"); exit();}'`

      2. ```shell
         (base) [root@node0 systemtap]# stap -ve 'probe begin{printf("Hello, World\n"); exit();}'
         Pass 1: parsed user script and 476 library scripts using 272288virt/69516res/3484shr/66136data kb, in 670usr/30sys/701real ms.
         Pass 2: analyzed script: 1 probe, 1 function, 0 embeds, 0 globals using 273740virt/71360res/3820shr/67588data kb, in 0usr/10sys/10real ms.
         Pass 3: using cached /root/.systemtap/cache/7d/stap_7de225d50de7509efffebbf4fabc454d_1020.c
         Pass 4: using cached /root/.systemtap/cache/7d/stap_7de225d50de7509efffebbf4fabc454d_1020.ko
         Pass 5: starting run.
         Hello, World
         Pass 5: run completed in 10usr/30sys/384real ms.
         ```

      3. 

2. ## 使用过程