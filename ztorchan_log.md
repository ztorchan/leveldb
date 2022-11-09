# Change Log

## Task 1: 简单将SST存储路径分开，将Level 0和Level 1的SST放置在SSD下，其余放置在HDD下。
### 需要改动的地方：
  1. 需要添加冷热分离的开关，添加SSD和HDD的路径设置选项，令SSD为主要路径，即MANIFEST等文件的存放路径。
  2. 对于Minor Compaction，因为不一定会直接落入Level 0，需要对最终落入的Level进行判断，以此改变落盘路径。~~在这过程中可能会读取不同Level的SST文件，需要修改读取路径~~（这里不会读取文件，而是直接通过file meta内的smallest和largest进行判断）。（相关过程可以参考[LevelDB源码解析(12) Memtable落盘](https://www.huliujia.com/blog/124132a9b3/)）
  3. 对于Major Compaction，要根据哪一Level发生了Compaction以及新SST的目标Level，修改输入SST的读取路径其输出SST的落盘路径。（相关过程可以参考[LevelDB源码解析(13) BackgroundCompaction SST文件合并](https://www.huliujia.com/blog/4496bd928e/)，阅读源码从`MaybeScheduleCompaction`开始）
  4. 要注意到，Major Compaction中有一种可能的情况，合并发起的Level不为0，所输入的SST文件数目为1，而下一级Level没有与之重叠的SST，也就是说下一级Le夫拉弗vel没有需要参与Compaction的SST（相关判断`IsTrivialMove()`）。这种情况下，Leveldb仅需要修改一下Version中该SST所处的level即可，无需额外操作。可是对于本次任务而言，是可能发生SST需要从SSD迁移到HDD的情况。这里需要作出修改。
  5. `RemoveObsoleteFile`用于删除废弃的文件，它会检查数据库下所有的文件，这里也需要改动检查路径。
  
### 存在问题：
  1. 对于Minor Compaction，Leveldb先将ImmuMemTable构建成SST落入硬盘，后续再根据该SST中是否与下一级Level文件有重叠，决定将其放置在哪个Level，那么就没办法提前得知它应当落入SSD还是HDD。  
    **解决方法**：可以将判断过程前置。原过程是在`BuildTable`的过程中记录最大key和最小key，完成以后再通过`PickLevelForMemTableOutput`过程决定将其设置为哪个Level。要提前判断的话，要再`BuildTable`前构建一个MemTable的迭代器，找到最大key和最小key，直接判断Level。但这带来的额外开销暂且未知。

### 实现思路：
  1. 在`options`中添加  
    ``` C++
      // Leveldb will wirte hot data in ssd, and will write cold data in hdd
      // if shot_cold_separation is true, sd_path and hdd_path must both have values 
      // if shot_cold_separation is false, db will behave as the original Leveldb
      bool hot_cold_separation = false;
      std::string ssd_path;
      std::string hdd_path;
    ```