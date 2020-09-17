介绍类文档（了解）

QOM(QEMU Object Model)——必看（看了这个框架，对理解源码的实现逻辑有帮助）：
https://wiki.qemu.org/Features/QOM/Machine
qemu QOM(qemu object model)和设备模拟
https://blog.csdn.net/ayu_ag/article/details/52870393


Documentation/GettingStartedDevelopers. 开发者需知:
https://wiki.qemu.org/Documentation/GettingStartedDevelopers
开发者相关的文档:
https://wiki.qemu.org/Category:Developer_documentation
Qemu Detailed Study
https://lists.gnu.org/archive/html/qemu­devel/2011­04/pdfhC5rVdz7U8.pdf


源码docs需要关注的文档：（熟悉操作以后需要看）
迁移   docs/devel/migration.rst
Block I/O error injection using blkdebug   docs/devel/blkdebug.txt
Block driver correctness testing with blkverify   docs/devel/blkverify.txt
The memory API models the memory and I/O buses and controllers of a QEMU machine
docs/devel/migration.rst
COLO,   docs/COLO­FT.txt （重点关注）
block­replication, 块复制。 docs/block­replication.txt


一些内部功能(还有一些人工监视/控制界面)实现在QAPI和QMP
Features/QAPI: https://wiki.qemu.org/Features/QAPI
Documentation/QMP:   https://wiki.qemu.org/Documentation/QMP




开发测试方法/工具（熟悉基本常规操作以后，需要重点关注开发测试手段，和常用测试工具）
CoLo 部署和测试
https://wiki.qemu.org/Features/COLO
手动配置：
https://wiki.qemu.org/Features/COLO/Manual_HOWTO
https://wiki.qemu.org/Features/COLO/Managed_HOWTO


debug 加分析(对内存分布相关的图较多)
https://blog.csdn.net/huang987246510/article/details/103791410
https://blog.csdn.net/huang987246510/category_9645395.html
调试：
gdb https://blog.csdn.net/Shirleylinyuer/article/details/79115187
event https://blog.csdn.net/Shirleylinyuer/article/details/80283008
stap toolkit
https://github.com/detailyang/systemtap­toolkit


Debug QEMU: 怎么去debugQEMU的。就是valgrind和gdb。
https://wiki.qemu.org/Documentation/Debugging
https://wiki.qemu.org/Documentation/Debugging_with_Valgrind
QEMU源码文档：
QEMU SystemTap trace tool， docs\tools\qemu­trace­stap.rst
Tracing framework is described at Features/Tracing­­­
https://wiki.qemu.org/Features/Tracing
关于追踪qemu 源码函数路径的一个方 法：
https://www.cnblogs.com/jusonalien/p/4764798.html
虚拟机管理工具：
Virt Tools | Blogging about open source virtualization
https://planet.virt­tools.org/
QEMU测试
https://wiki.qemu.org/Testing
gdb­qemu 调试
https://github.com/ehabkost/gdb­qemu


社区文档（当工具书用）
qemu文档:
https://wiki.qemu.org/Documentation
qemu官方博客:
https://www.qemu.org/blog/
QEMU TODO
https://wiki.qemu.org/ToDo/CodeTransitions




推荐博客（QEMU 流程、框架类的资料这部分资料特别多，这块儿有个大概了解就行，目标就是：后面有问题知道去哪里查资料，知道查资料的关键字）
我见过最全的剖析QEMU原理的文章
https://blog.csdn.net/u014022631/article/details/81001263
https://people.cs.nctu.edu.tw/~chenwj/dokuwiki/doku.php?id=qemu
下面这几个资料是同一个作者:
https://blog.csdn.net/ayu_ag/article/details/52808349
vl.c:main，Qom init，参数解析，初始化内存
https://blog.csdn.net/ayu_ag/article/details/52808349
qemu­kvm部分流程/源代码分析: 这个里面有很多详细的流程图，很不错，之后跟代码的时
可以去当做工具用。
https://blog.csdn.net/sdulibh/article/details/51839410
qemu整体流程，源码架构解读: 整体逻辑讲的很不错:
https://blog.csdn.net/robinblog/article/details/8876599
http://blog.chinaunix.net/uid­26941022­id­3510672.html
http://blog.chinaunix.net/uid­8679615­id­5710883.html
北方南方的资料不错:   https://me.csdn.net/u011414616
另外推荐 苏易北 的博客
https://abelsu7.top/2019/06/04/qemu-src-notes/

