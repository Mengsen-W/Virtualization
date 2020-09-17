# This file for question of QEMU
1. 什么是Heterogeneous Memory Attribute Table，好像是一个专有名词
2. irqchip中断芯片的模拟，是使用QEMU还是使用KVM，参数split不清楚含义

3. TCG 微型代码生成器，转换guest二进制到host

4. SMP系统 对称多处理结构 NUMA 非一致存储访问结构 MPP 海量并行处理结构
   - SMP 所有cpu共享全部资源，操作系统管理着一个队列，每个处理器依次处理队列中的进程
   - NUMA 每个cpu模块由多个cpu物理实体组成，每个cpu模块具有单独的内存IO槽口等，节点之间可以通过互联模块进行连接和信息交互，因此每个cpu可以访问整个系统的内存，当然访问本地内
  存的速度要远高于访问远地内存
   - MPP 由多个SMP服务器通过一定的节点互联网络进行连接，协同工作，每个节点只访问自己本地
  资源，理论上扩展无限制，但是要通过复杂的调度机制平衡各节点的负载和并行处理过程
   - (https://www.cnblogs.com/yubo/archive/2010/04/23/1718810.html)

5. 一个示例
```shell
-machine hmat=on \
// machine 配置虚拟机属性
// hmat 默认关闭
-m 2G,slots=2,maxmem=4G \
// 设置guest启动RAM为2G
// 共设置两个内存插槽
// 最大内存设置为4G
// 采取热拔插技术，可以动态增减内存大小
-object memory-backend-ram,size=1G,id=m0 \
// 创建一个内存后端对象，该对象可用于备份guestRAM
// size 指定了存储区域的大小
// id 指定了一个唯一ID
-object memory-backend-ram,size=1G,id=m1 \
-numa node,nodeid=0,memdev=m0 \
// 添加NUMA节点
-numa node,nodeid=1,memdev=m1,initiator=0 \
// initiator指向使此节点拥有最佳性能的节点
-smp 2,sockets=2,maxcpus=2  \
-numa cpu,node-id=0,socket-id=0 \
// 给node0分配cpu0
-numa cpu,node-id=0,socket-id=1
```