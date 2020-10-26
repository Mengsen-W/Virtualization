# 使用 QMP 命令和结构体记录 `checkpoint` 过程相关信息

1. ## 设置 `QMP` `json`对象

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
      # <- { "return": {
      #         "avg_time": "123456",
      #         "tot_time": "123456",
      #         "init_time": "123456",
      #         "dev_time": "123456",
      #         "stat_time": "123456",
      #         "ram_time": "123456
      #         "dev_size": "123456789",
      #         "ram_size": "123456789"
      #         "info": "MigrationInfo"
      #      }
      #    }
      ##
      { 'command': 'info_checkpoint', 'returns': 'checkpoint_recorder' }
      ```

   2. **<u>注意这里复用一部分`MigrationInfo`的代码，用来记录`COLO`过程迁移内存大小（后续可能会继续调整），而且目前应该先不会使用这部分代码，计划先把`get_clock()`记录时间调通过，再修改这部分。</u>**

2. ## 修改`migration.c`源码

   1. 检查实际结构体声明和函数声明

      ```c
      checkpoint_recorder *qmp_info_checkpoint(Error **errp);
      
      //../qapi/qapi-types-migration.h
      struct checkpoint_recorder {
          int64_t avg_time; // 每次进入的平均时间
          int64_t tot_time; // 全生命周期的总时间
          int64_t init_time; // 初始化变量时间
          int64_t tot_num;  // 总次数
          int64_t dev_time; // 传输设备时间
          int64_t stat_time; // 改变状态时间
          int64_t ram_time; // 传输ram时间
          int64_t dev_size; // 传输设备内存大小
          int64_t ram_size; // 传输ram大小
          MigrationInfo *info; // 信息
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
        // not sure that free checkpoint_recorder at same this free MigrationInfo
        // so free MigrationInfo first and than free checkpoint_recorder
        if(g_cr != NULL) {
          qapi_free_MigrationInfo(g_cr->info);
          qapi_free_checkpoint_recorder(g_cr);
        }
      
        g_cr = g_malloc0(sizeof(checkpoint_recorder));
        g_cr->info = g_malloc0(sizeof(MigrationInfo));
        g_cr->init_time = get_clock();
      
        return;
   }
      ```

      我这里仍然先释放内部结构体在释放本结构体
      
      ```c
      /**
       * @Brief: init global checkpoint recorder point
       * @Author: mengsen
       * @Data: 2020-10-22 15:09:36
       **/
      static void init_checkpoint_recorder() {
       // not sure that free checkpoint_recorder at same this free MigrationInfo
       // so free MigrationInfo first and than free checkpoint_recorder
       if(g_cr != NULL) {
         qapi_free_MigrationInfo(g_cr->info);
         qapi_free_checkpoint_recorder(g_cr);
       }
       g_cr = g_malloc0(sizeof(checkpoint_recorder));
       g_cr->info = g_malloc0(sizeof(MigrationInfo));
       g_cr->init_time = get_clock();
      
        return;
      }
   ```
      
      ```c
      /**
       * @Brief: qmp interface
       * @param: errp [Error **] error handle
       * @return: [checkpoint_recorder *]
       * @Author: mengsen
       * @Date: 2020-10-22 15:26:43
      **/
      checkpoint_recorder *qmp_info_checkpoint(Error **errp) {
          if(g_cr == NULL) {
              error_setg(errp, "global checkpoint_recorder error");
              return NULL;
          }
          checkpoint_recorder *cr = g_malloc0(sizeof(checkpoint_recorder));
          cr->info = g_malloc0(sizeof(MigrationInfo));
          memcpy(cr, g_cr, sizeof(checkpoint_recorder));
          return cr;
      }
      
      ```

3. ## 修改`colo.c`源码

   1. 声明全局变量

      ```c
      extern checkpoint_recorder *g_cr;
      ```

      