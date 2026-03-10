## This documentation is based on the implementation of MCU porting LittleFs
### 0. LittleFs的功能和特点
LittleFs的主要特点为：实现了均衡损耗与掉电安全。
### 1. LittleFs源码解读
#### 1.1关键术语解读
① **无边界存储与有边界存储**：
“无边界存储”（Unbounded Storage）并不是指内存（RAM），而是指存储介质（Flash）的写入方式，指的是文件系统允许在一个固定的存储块上**无限制地进行擦除和写入操作，而没有考虑Flash硬件的物理寿命限制**。
    有边界存储： 承认Flash硬件有物理上限（即每个存储单元有擦写次数限制，通常是1万到10万次）。文件系统的设计需要围绕这个“边界”来优化，确保磨损均衡，避免某个区域被频繁擦写而提前报废。
    无边界存储： 假设存储空间是无限的、可无限重写的（像硬盘或内存那样）。如果按照这个思路设计文件系统，就会频繁地更新同一块区域的数据，这在Flash上是非常危险的。


### 2. 基于STM32F407ZET6MCU移植LittleFs文件系统
  STM32裸机对于程序的执行以前后台方式来实现，对于STM32的MCU而言，移植文件系统有两种方案，移植到MCU内部Flash或者外部的SPI Flash上，此处我先以内部
Flash作为嵌入式场景的移植案例。
#### 2.1 移植架构分析
  首先分析整体架构，STM32原本操作内部Flash的方式为，使用标准库或HAL库中提供的函数，按照Flash操作流程直接进行操作；移植文件系统以后意味着在以后的工程
中，我们可以通过文件系统来操作MCU内部的Flash，以这种方式操作可以兼具LittleFs均衡损耗与掉电安全的特性。
原先的方式 Main(应用) -> HAL/STD Funs(驱动) ->MCU Flash(硬件)

  我们的目标就是基于STM32CubeMax实现一个工程基座，通过移植文件系统(lfs.c lfs.h lfs_util.c lfs_util.h)相关源码，并创建文件lfs_port.c实现作为文件
系统和原本HAL/STD操作Flash的适配层功能。
移植后方案 Main(应用) -> LittleFs Funs(系统) ->.read = lfs_flash_read(以read为例，实现在lfs_port.c中) -> HAL/STD Funs ->MCU Flash
#### 2.2 具体适配实现
1. 适配工作以struct lfs_config 为核心展开：
用HAL/STD Lib完成四个操作函数(read prog earse sync)，并用指针对四个操作函数进行指向；对硬件参数和性能参数进行配置；
另外需要确认,1. 为LittleFs提供静态缓冲区; 2. 在函数操作里(适配层lfs_port.c); 3. 做地址保护,sync错误码返回的意义。

2. STM32F407ZET6 Flash布局及文件系统管理位置
  STM32F407ZET6的Flash布局不太规整，扇区0-3大小为16KB，扇区4大小为64KB，扇区5-7均为128KB；首先应该阅读.map文件，避免文件系统和代码段互相占用造成芯片损坏。其次注意编写适配层文件时，我们会编辑block_size = 扇区大小，因为STM32的硬件强制要求按扇区擦除，这里的值选择计划分配扇区空间大小即可。
##### 此处应该注意，避免让文件系统管理大小不一致的扇区，必须只使用相同大小的扇区作为 LittleFS 的管理区域，如果混用(比如16KB和128KB)会造成逻辑与物理实际混乱，浪费空间，并且磨损均衡策略也会失效。。
  根据 LittleFS 的规范，它将存储介质视为一个由大小均匀的块（evenly sized blocks）组成的数组，这些块是文件系统进行分配、垃圾回收和磨损均衡的逻辑单元，在 lfs_config 中配置的 block_size 参数，就是告诉 LittleFS 每一个“块”有多大。LittleFS 在运行时，会假设它所管理的每一个块（从块0到块 block_count-1）的大小都等于 block_size。



