# 使用 QMP 命令和结构体记录 `checkpoint` 过程相关信息

## 设置 `QMP` `json`对象

1. 在`./qapi/migration.json`中增加结构体对象和命令

   ```json
   ##
   # @checkpoint_recorder:
   #
   # Used to get checkpoint average time, total time, total number,
   # amount of data transmitted
   #
   # @avg_time: average time of every checkpoint
   #
   # @tot_time: total time of all checkpoint
   #
   # @init_time: init process time
   #
   # @tot_num: total number of checkpoint times
   #
   # @dev_time: device migrate average time
   #
   # @ram_time: ram migrate average time
   #
   # @stat_time: VM state average time
   #
   # @dev_size: device migrate bytes size
   #
   # @ram_size: ram migrate bytes size
   #
   # @info: MigrationInfo
   #
   # Since: 5.0.0
   ##
   { 'struct': 'checkpoint_recorder',
      'data': { 'avg_time': 'int64', 'tot_time': 'int64', 'init_time': 'int64',
               'tot_num': 'int64', 'dev_time': 'int64', 'stat_time': 'int64',
               'ram_time': 'int64', 'dev_size': 'int64', 'ram_size': 'int64',
               'info': 'MigrationInfo' } }
   { 'struct': 'checkpoint_recorder',
       'data': { 'tot_time': 'int64', 'tot_num': 'int64', 'init_time': 'int64',
               'wait_time': 'int64', 'avg_time': 'int64',
               'vmstate_time': 'int64', 'stat_time': 'int64',
               'ram_time': 'int64', 'dev_time': 'int64',
               'dev_size': 'int64', 'ram_size': 'int64',
               'info': 'MigrationInfo' } }
   
   ```

   ```json
   ##
   # @info_checkpoint:
   #
   # Return information about checkpoint_recorder
   #
   # Returns: @checkpoint_recorder
   #
   # Since: 5.0.0
   #
   # Example:
   #
   # -> { "execute": "info_checkpoint" }
   #	 { "execute": "reset_checkpoint" }
   # <- { "return": {
   #         "tot_time": "123456",
   #		  "tot_num": "123456",
   #         "init_time": "123456",
   #    	  "wait_time": "123456",
   #         "avg_time": "123456",
   #         "dev_time": "123456",
   #         "ram_time": "123456",
   #		  "vmstate_time": 123456,
   #         "dev_size": "123456789",
   #         "ram_size": "123456789"
   #         "info": "MigrationInfo"
   #      }
   #    }
   ##
   { 'command': 'info_checkpoint', 'returns': 'checkpoint_recorder' }
   ```

2. **<u>注意这里复用一部分`MigrationInfo`的代码，用来记录`COLO`过程迁移内存大小（后续可能会继续调整），而且目前应该先不会使用这部分代码，计划先把`get_clock()`记录时间调通过，再修改这部分。</u>**

## 修改`migration.c`源码

1. 检查实际结构体声明和函数声明

   ```c
   checkpoint_recorder *qmp_info_checkpoint(Error **errp);
   
   //../qapi/qapi-types-migration.h
   struct checkpoint_recorder {
       int64_t tot_time; 		// 全生命周期的总时间
       int64_t tot_num;  		// 总次数
       int64_t init_time; 		// 初始化变量时间
       int64_t wait_time; 		// 主机每次checkpoint等待时间
       int64_t avg_time; 		// 每次进入的平均时间
       int64_t dev_time; 		// 传输设备时间
       int64_t stat_time; 		// 改变状态时间
       int64_t ram_time; 		// 传输ram时间
       int64_t vmstate_time; 	// 改变状态时间
       int64_t dev_size; 		// 传输设备内存大小
       int64_t ram_size; 		// 传输ram大小
       MigrationInfo *info; 	// Migration信息
   };
   
   void qapi_free_checkpoint_recorder(checkpoint_recorder *obj);
   void qapi_free_MigrationInfo(MigrationInfo *obj);
   ```

2. 注意这里使用`MigrationInfo`结构体的指针

3. 声明全局指针

   ```c
   // ./migration.c
   
   checkpoint_recorder *g_cr;
   ```

4. 选择初始化位置，我选择了在`migration_iteration_finish()`这个函数中初始化主节点`global checkpoint_recorder`，在`process_incoming_migration_co()`初始化从节点`global chekcpoint_recorder`

5. **释放是可以不用管**，**构造要单独构造**。

   ```c
   // 这里单独构造
   info->xbzrle_cache = g_malloc0(sizeof(*info->xbzrle_cache));
   
   MigrationInfo *qmp_query_migrate(Error **errp)
   {
      	MigrationInfo *info = g_malloc0(sizeof(*info));
   
      	fill_destination_migration_info(info);
     	fill_source_migration_info(info);
   
   	// 这里没有析构
      	return info;
   }
   ```

   这里我一开始想是先析构再构造，后来我觉得还是直接将结构体内部变量置`0`。因为我考虑对于全局变量，内存分区在全局变量区，是不是最好全生命周期只指向一个堆区内存比较好，频繁的构造析构也许可能会产生一些堆区碎片，而且最重要的是万一某变量拿到堆区指针，后面我构造又析构后，某变量可能会段错误。
   
   ```c
   /**
    * @Brief: init global checkpoint recorder point
    * @Author: mengsen
    * @Data: 2020-10-22 15:09:36
   **/
    static void checkpoint_recorder_init() {
        if(g_cr != NULL) {
            memset(g_cr->info, 0, sizeof(MigrationInfo));
            memset(g_cr, 0, 8 * sizeof(int64_t));
          } else {
            g_cr = g_malloc0(sizeof(checkpoint_recorder));
        }
        return;
   }
   ```
   
   我这里仍然释放结构体
   
```c
   /**
 * @Brief: destroy global checkpoint_recorder
    * @Param: [void]
    * @Return [void]
    * @Author: mengsen
    * @Date: 2020-10-23 10:56:11
   **/
   static void checkpoint_recorder_destroy(void) {
       if(g_cr != NULL) {
           qapi_free_MigrationInfo(g_cr->info);
       }
       return;
   }
```

   这里复用了一部分`MigrationInfo`的代码

   ```c
   /**
    * @Brief: fill global checkpoint_recorder to temporary checkpoint_recorder,
    * and that temporary used lazy loading
    * @Param: cr [checkpoint_recorder *] temporary checkpoint_recorder pointer
    * @Return: [void]
    * @Author: mengsen
    * @Date: 2020-10-23 10:40:22
   **/
   static void fill_checkpoint_recorder(checkpoint_recorder *cr){
       cr->tot_time = g_cr->tot_time;
      	cr->tot_num = g_cr->tot_num;
       cr->init_time = g_cr->init_time;
       
       cr->wait_time = g_cr->wait_time / cr->tot_num;
       cr->avg_time = cr->tot_time / cr->tot_num;
       cr->dev_time = g_cr->dev_time / cr->tot_num;
       cr->ram_time = g_cr->ram_time / cr->tot_num;
       cr->vmstate_time = g_cr->vmstate_time / cr->tot_num;
       cr->dev_size = g_cr->dev_size / cr->tot_num;
       cr->ram_size = g_cr->ram_size / cr->tot_num;
   
       fill_destination_migration_info(cr->info);
       fill_source_migration_info(cr->info);
       
       return;
   }
   
   ```

   `qmp`接口部分，这样每次除了`checkpoint_recorder`的内存需要我管理，其他的都不需要我管理了。

   ```c
   /**
    * @Brief: qmp interface
    * @Param: errp [Error **] error handle
    * @Return: [checkpoint_recorder *]
    * @Author: mengsen
    * @Date: 2020-10-22 15:26:43
   **/
   checkpoint_recorder *qmp_info_checkpoint(Error **errp) {
       checkpoint_recorder *cr = g_malloc0(sizeof(checkpoint_recorder));
      	cr->info = g_malloc0(sizeof(MigrationInfo));
     	fill_checkpoint_recorder(cr);
       return cr;
   }
   
   ```

   、、

## 修改`colo.c`源码

1. 声明全局变量

   ```c
   extern checkpoint *g_cr;
   ```

2. 一些添加位置

   1. `init_time`

      - 主节点添加在`migrate_start_colo_process()`函数开始，结束在进入`colo`的`while`循环之前
      - 从节点添加在`colo_process_incoming_thread()`函数开始处，结束在进入`colo`的`while`循环之前

   2. `tot_num`

      - 添加在每个`while`的尾部部，意思是进行了一次`checkpoint`

   3. `tot_time`

      - 在放在`while`的首尾分别记录时间点，然后在每次`checkpoint`结束后，这个时间点在最后，统计时间点的差值并累加

   4. `wait_time`

      - 统计主节点每次`sem_wait`的时间
      - 从节点在`colo_wait_handle_message()`测试接受数据是否是阻塞接受的

   5. `avg_time`

      - 采用懒加载每次调用`qmp`接口时，在`fill_checkpoint`计算统计这个量

   6. `dev_time`

      - 主节点在`qemu_save_device_state()`函数上下加点
      - 从节点在`qemu_load_device_state`函数上下加点

   7. `ram_time`

      - 主节点在`qemu_savevm_live_state*()`函数上下加点
      - 从节点在`cpu_synchronize_all_pre_loadvm()`和`qemu_loadvm_state_main()`上下加点

   8. `vmstate_time`

      - 主节点在`qemu_put_buffer()`函数上下加点
      - 从节点在`colo_receive_message()`函数上下加点

   9. `ram_size`

      - 主节点在统计`bioc->usage`
      - 从节点统计`total_size`，这个长度由主节点发送而来

   10. 测试结果

       1. 4核4G 编译内核

       ```
       {"return": {"tot_time": 9662176973, "wait_time": 13666197977, "init_time": 55992172, "stat_time": 0, "dev_size": 0, "dev_time": 937410, "vmstate_time": 20038, "ram_size": 23552, "avg_time": 878379724, "tot_num": 11, "ram_time": 209590288, "info": {"expected-downtime": 892, "status": "colo", "setup-time": 42, "total-time": 1191391, "ram": {"total": 2165121024, "postcopy-requests": 0, "dirty-sync-count": 70, "multifd-bytes": 0, "pages-per-second": 397, "page-size": 4096, "remaining": 0, "mbps": 13.370343, "transferred": 15187177267, "duplicate": 672480, "dirty-pages-rate": 495, "skipped": 0, "normal-bytes": 15151529984, "normal": 3699104}}}}
       ```

       ```
       {"return": {"tot_time": 13715646658, "wait_time": 8342171814, "init_time": 129270482, "stat_time": 0, "dev_size": 0, "dev_time": 931910, "vmstate_time": 23924, "ram_size": 23552, "avg_time": 721876139, "tot_num": 19, "ram_time": 108490503, "info": {"expected-downtime": 2760, "status": "colo", "setup-time": 38, "total-time": 175604, "ram": {"total": 2165121024, "postcopy-requests": 0, "dirty-sync-count": 22, "multifd-bytes": 0, "pages-per-second": 621, "page-size": 4096, "remaining": 0, "mbps": 20.513435, "transferred": 2328786212, "duplicate": 562646, "dirty-pages-rate": 1464, "skipped": 0, "normal-bytes": 2319192064, "normal": 566209}}}}
       ```
       
       