# Migration and COLO

``` ASCII
 Primary Node                                                              Secondary Node
+------------+  +-----------------------+       +------------------------+  +------------+
|            |  |       HeartBeat       +<----->+       HeartBeat        |  |            |
| Primary VM |  +-----------+-----------+       +-----------+------------+  |Secondary VM|
|            |              |                               |               |            |
|            |  +-----------|-----------+       +-----------|------------+  |            |
|            |  |QEMU   +---v----+      |       |QEMU  +----v---+        |  |            |
|            |  |       |Failover|      |       |      |Failover|        |  |            |
|            |  |       +--------+      |       |      +--------+        |  |            |
|            |  |   +---------------+   |       |   +---------------+    |  |            |
|            |  |   | VM Checkpoint +-------------->+ VM Checkpoint |    |  |            |
|            |  |   +---------------+   |       |   +---------------+    |  |            |
|Requests<--------------------------\ /-----------------\ /--------------------->Requests|
|            |  |                   ^ ^ |       |       | |              |  |            |
|Responses+---------------------\ /-|-|------------\ /-------------------------+Responses|
|            |  |               | | | | |       |  | |  | |              |  |            |
|            |  | +-----------+ | | | | |       |  | |  | | +----------+ |  |            |
|            |  | | COLO disk | | | | | |       |  | |  | | | COLO disk| |  |            |
|            |  | |   Manager +---------------------------->| Manager  | |  |            |
|            |  | ++----------+ v v | | |       |  | v  v | +---------++ |  |            |
|            |  |  |+-----------+-+-+-++|       | ++-+--+-+---------+ |  |  |            |
|            |  |  ||   COLO Proxy     ||       | |   COLO Proxy    | |  |  |            |
|            |  |  || (compare packet  ||       | |(adjust sequence | |  |  |            |
|            |  |  ||and mirror packet)||       | |    and ACK)     | |  |  |            |
|            |  |  |+------------+---^-+|       | +-----------------+ |  |  |            |
+------------+  +-----------------------+       +------------------------+  +------------+
+------------+     |             |   |                                |     +------------+
| VM Monitor |     |             |   |                                |     | VM Monitor |
+------------+     |             |   |                                |     +------------+
+---------------------------------------+       +----------------------------------------+
|   Kernel         |             |   |  |       |   Kernel            |                  |
+---------------------------------------+       +----------------------------------------+
                   |             |   |                                |
    +--------------v+  +---------v---+--+       +------------------+ +v-------------+
    |   Storage     |  |External Network|       | External Network | |   Storage    |
    +---------------+  +----------------+       +------------------+ +--------------+
```

## 1. 初始化

`qemu_init()`
1.`migration_object_init()`
- 初始化迁移传入对象，无论是否使用。

2.`blk_mig_init()`
- 初始化块设备迁移，包括块设备状态的初始化，块设备迁移列表初始化，和锁的初始化

3.`ram_mig_init()`
-`XBZRLE` 算法锁的初始化

4.`dirty_bitmap_mig_init()`
- 脏内存页面标记初始化

## 2. Migrate调用栈

**`hmp_migrate()`** 迁移部分

- `qmp_migrate()` ./migration/migration.c
- `migrate_get_current()` ./migration/migration.c，确定 migrate 对象状态，并返回此对象
- ​	`migrate_prepare()` ./migration/migration.c，做迁移前的准备工作，返回`true` 证明准备完成继续迁移，返回`false` 直接`return``
- `socket_start_outgoing_migration()` ./migration/socket.c，用`TCP` 通讯，这里解析了socket参数，并进行函数调用

  - `socket_start_outgoing_migration_internal()` ./migration/socket.c，这是内部函数，主要创建了`qio_channel` 对象，并进行连接
    - `qio_channel_socket_connect_async()`
    - `exec_start_outgoing_migration()` ./migration/exec.c，用`shell` 通讯，最终会用`qio_channel` 维护通讯
    - `migration_channel_connect()` ./migration/channel.c
    - `fd_start_outgoing_migration()` ./migration/fd.c，用`fd` 通讯，最终会用`qio_channel` 维护通讯
    - `migration_channel_connect()` ./migration/channel.c
    - `error_set() migrate_set_state() block_cleanup_parameters()` 出错处理

**`migration_channel_connect()`** IO 部分，./migration/channel.c 绑定`QIOChannel`和`MigrationState`

- `migrate_fd_connect()` ./migration/migration.c 设置一些参数，状态和通知函数，针对`PreCopy` 和`PostCopy` 操作有不同
- `qemu_thread_create()` ./migration/migration.c 创建新线程处理迁移的IO操作

**` migration_thread()`** 迁移部分专用的线程 ./migration/migration.c

- `qemu_clock_get_ms()` ./include/qemu/timer.h 拿到开始时间
- `rcu_register_thread()` ./util/rcu.c 注册这个线程到链表
- `object_ref()` .qom/object.c，增加这个迁移对象的引用次数
- `update_iteration_initial_status()` ./migration/migration.c，更新传入的`MigrationState` 状态，防止计算错误
- `qemu_savevm_state_header()` ./migration/savevm.c，储存`QEMUFile` 头状态，挂载一些头信息
- `qemu_savevm_start_setup()` ./migration/savevm.c，存储`QEMUFile` 头状态和项目状态
- `migrate_set_state()` ./migration/migration.c，设置迁移状态
- `migrate_is_active()` ./migration/migration.c，判断迁移对象是否还在存活，是迁移过程while循环的一部分
- `migration_iteration_run()` ./migration/migration.c，主要迭代工作的控制函数
  - `qemu_savevm_state_pending()` ./migration/savevm.c，循环遍历每一个发送条目，调用不同回调发送
    - `save_live_pending()`./include/migration/register.h，实际回调发送函数
  - `migration_completion()` ./migration/migration.c，剩余数据已经足够一次发送完毕
    - `migration_maybe_pause()` ./migration/migration.c，申请停止源虚拟机
    - `qemu_savevm_state_complete_percopy()` ./migration/savevm.c 调用回调和错误处理
      - `sace_live_complete_percopy()` ./include/migration/register.h，实际回调函数
- `migration_iteration_finish()` ./migration/migration.c，做一些清理工作
-  `object_unref(OBJECT(s))` .qom/object.c，减少这个迁移对象的引用次数
- `rcu_unregister_thread()`./util/rcu.c，解除注册这个线程到链表

## 3. 函数细节

` migrate_prepare()` ./migration/migration.c

* 传入过程中会设置`bool resume` ，在设置过`True` 的情况下会导致特殊情况
* 这里好像是说`Postcopy recovery` 不能很好的和`release-ram` 一起工作，因为这可能会导致网络的发送缓冲区页面丢失，但是幸运的事`release-ram` 被设计源主机和目的主机在同一物理服务器上工作，所以应该是没问题的。
* 当调用`migrate_release_ram()` 返回`True` 时，此函数返回`True` ，否则返回`False`
* 传入过程中会设置`bool resume` ，在设置过`False` 的情况下\
* 会检查`migrate` 对象状态，并且初始化此对象，初始化`ram_counters` 返回`True`
`migrate_fd_connect()` ./migration/migration.c

`migrate_thread()` ./migration/migration.c

`migration_iteration_run()` ./migration/migration.c 迭代的关键逻辑
