# 迁移参数优化选项

## 1. announce 相关

### 	1. announce-initial: int

​	发送第一个声明的初始延迟，单位是毫秒，默认是50毫秒

### 	2. announce-max: int

​	通知中数据包的最大延迟，单位是毫秒，默认是550毫秒

### 	3. announce-rounds: int

​	迁移之后发送自描述包的数量，默认是5个

### 	4. announce-step: int

​	在通知中随后的数据包之间的延迟增加值，单位是毫秒，默认是100毫秒

------



## 2. tls相关

### 	1. tls-creds: string

​	`ls-creds `是传输对象凭证的ID，该对象为在迁移数据通道上建立TLS连接提供凭据。传出端必须与传入端符合，传入端凭据也必须与传出端符合。空字符串意味着QEMU将使用纯文本模式进行迁移，不适用凭证ID

### 	2. tls-hostname: string

​	用于验证主机名的凭证

### 	3. tls-authz: string

​	动态创建的证书，有点类似私钥？

------



## 3. 压缩相关

### 	1. max-bandwidth: int

​	最大的迁移速度，bytes per second，源码设置的是`qemu_file_set_rate_limit`

### 	2. compress-level: int

​	压缩等级由1（压缩速度最快）到9（压缩率最高），在传输带宽不是限制因素的时候选择压缩等级1对于整体最快

### 	3. compress-thread: int

​	参与压缩的线程数，默认是8

### 	4. compress-wait-thread: boolean

​	在所有压缩线程都忙的时候，是否需要压缩再发送。默认是`true`，也就是等待到有线程空闲，再压缩再发送。如果为`false`则直接发送而不压缩

### 	5. decompress-thread: int

解压缩线程数，默认是2，这里源码注释解压的速度至少是压缩速度的4倍，所以这里默认是2

### 	6. block-incremental: boolean

​	作用于启动块迁移时需要迁移的存储数量。当为false时，将整个存储迁移到目标位置的存储中；当为true时，只迁移活动的部分，并且目标必须已经能够访问源上使用的相同的后备链。源码位置在`init_blk_migration()`默认是true

### 	7. multifd-channels: int

​	迁移数据传输的并行数，类似于socket数量，默认是2，在1-255之间

### 	8. xbzrle-cache-size: int

​	XBZRLE迁移使用的缓存大小。它需要是目标页面大小的倍数和2的幂 ，默认是(64 * 1024 * 1024)

### 	9. max-postcopy-bandwidth: int

​	postcopy后台传输带宽限制，默认值为0(无限制)。单位是字节每秒。页面请求仍然可以超过这个限制。

### 	10. multifd-compression: MultiFDCompression

​	使用哪种压缩算法，默认不设置

### 	11. multifd-zlib-level: int

​	设置实时迁移中使用的压缩级别，压缩级别是0到9之间的整数，其中0表示没有压缩，1表示最佳压缩速度，9表示最佳压缩比，这会消耗更多CPU。默认为1。

### 	12. multifd-zstd-level: int

​	设置实时迁移中使用的压缩级别，压缩级别是0到20之间的整数，其中0表示没有压缩，1表示最佳压缩速度，20表示将消耗更多CPU的最佳压缩比。默认为1。

------



## 4. cpu节流阀相关

### 	1. throttle-trigger-threshold: int

​	这是节流阀选型，由节流阀可以确定传输字节数`bytes_xfer_period * threshold / 100;(ram.c)`这个值默认是50

​	检查脏字节和上次中所传输的大约字节量之间的比率是否达到阈值。如果这个事情发生两次就下一步行动。如果之前没有节流阀就启动节流阀，如果已经启动节流阀了就增加阈值。

### 	2. cpu-throttle-initial: int

​	当迁移自动收敛被激活时（第一次需要节流阀时），客户cpu节流的初始时间百分比默认是20

### 	3. cpu-throttle-increment: int

​	当自动收敛检测到迁移没有进展时（后面几次需要节流阀时），节流阀百分比增加，默认是20

### 	4. cpu-throttle-tailslow: boolean

​	在节流调节的末阶段，虚拟机对CPU百分比速度的百分比非常敏感，而`cpu-throttle-increment`通常在末段过高。如果该参数为真，将计算客户机使用的理想CPU百分比，这可能恰好使脏率匹配脏率到达阈值。然后，我们将在`cpu-throttle-increment`指定的和由理想CPU百分比生成的之间选择一个更小的节流增量。因此，它与传统节流方式是兼容的，同时后部节流增量不会过大。简单来说，这个开关会使节流变的更小。默认值为false。**从5.1开始**

### 	5. max-cpu-throttle: int

​	最大CPU节流阀百分比。 默认是99。

------



## 5. 停机时间

### 	1. downtime-limit: int

​	设置迁移的最大可容忍停机时间。最大停机时间默认为 2000ms

### 	2. x-checkpoint-delay: int

​	两次`colo checkpoint`的间隔时间 默认是 200 * 100 ms