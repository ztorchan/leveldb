# Change Log

## Task 1: 简单将SST存储路径分开，将Level 0和Level 1的SST放置在SSD下，其余放置在HDD下。
### 需要改动的地方：
  1. 需要添加冷热分离的开关，添加SSD和HDD的路径设置选项，令SSD为主要路径，即MANIFEST等文件的存放路径。
  2. 对于Minor Compaction，因为不一定会直接落入Level 0，需要对最终落入的Level进行判断，以此改变落盘路径。~~在这过程中可能会读取不同Level的SST文件，需要修改读取路径~~（这里不会读取文件，而是直接通过file meta内的smallest和largest进行判断）。（相关过程可以参考[LevelDB源码解析(12) Memtable落盘](https://www.huliujia.com/blog/124132a9b3/)）
  3. 对于Major Compaction，要根据哪一Level发生了Compaction以及新SST的目标Level，修改输入SST的读取路径其输出SST的落盘路径。（相关过程可以参考[LevelDB源码解析(13) BackgroundCompaction SST文件合并](https://www.huliujia.com/blog/4496bd928e/)，阅读源码从`MaybeScheduleCompaction()`开始）
  4. 要注意到，Major Compaction中有一种可能的情况，合并发起的Level不为0，所输入的SST文件数目为1，而下一级Level没有与之重叠的SST，也就是说下一级Le夫拉弗vel没有需要参与Compaction的SST（相关判断`IsTrivialMove()`）。这种情况下，Leveldb仅需要修改一下Version中该SST所处的level即可，无需额外操作。可是对于本次任务而言，是可能发生SST需要从SSD迁移到HDD的情况。这里需要作出修改。
  5. `RemoveObsoleteFile()`用于删除废弃的文件，它会检查数据库下所有的文件，这里也需要改动检查路径。
  
### 存在问题：
  1. 对于Minor Compaction，Leveldb先将ImmuMemTable构建成SST落入硬盘，后续再根据该SST中是否与下一级Level文件有重叠，决定将其放置在哪个Level，那么就没办法提前得知它应当落入SSD还是HDD。  
    **解决方法**：可以将判断过程前置。原过程是在`BuildTable`的过程中记录最大key和最小key，完成以后再通过`PickLevelForMemTableOutput`过程决定将其设置为哪个Level。要提前判断的话，要再`BuildTable`前构建一个MemTable的迭代器，找到最大key和最小key，直接判断Level。但这带来的额外开销暂且未知。

### 实现思路要点：  

1. **关于开关和选项**  
    在`options`中添加了   
    ``` C++
      // Leveldb will wirte hot data in ssd, and will write cold data in hdd
      // if shot_cold_separation is true, sd_path and hdd_path must both have values 
      // if shot_cold_separation is false, db will behave as the original Leveldb
      bool hot_cold_separation = false;
      std::string ssd_path;
      std::string hdd_path;
    ```      
    在使用`Open()`打开数据库时，如果设置了`hot_cold_separation`为true，需要保证`ssd_path`和`hdd_path`有值，而且要保证`Open()`函数内的`dbname`要和`ssd_path`相同（设置了检查，这个地方用起来有点麻烦，等待改进）。  

2. **关于Minor Compaction**
    原生的Minor Compaction的主要调用大致如下：`CompactMemTable()`开始flush过程，调用`WriteLevel0Table()`。`WriteLevel0Table()`先调用`BuildTable()`完成构建SST文件，再调用`PickLevelForMemTableOutput()`选择文件所处的Level。最后回到`CompactMemTable()`会调用`RemoveObsoleteFiles()`清理不需要的文件。  

    重新写了以下几个函数代替：  
    - `WriteLevel0TableWithSeparation()`：当`hot_cold_separation`开启时会调用这个函数。和原来的不同，因为要事先确定文件输出到SSD还是HDD下，要先行获取Memtable的迭代器得到最大最小的key，判别所处Level，再传递对应路径构建SST文件。
    - `BuildTableWithSeparation()`：`WriteLevel0TableWithSeparation()`内会调用这个函数构建SST文件，和原来不同的点在于多传递了一个Level参数，因为函数内会对刚生成的SST构造一个迭代器进行测试，需要Level参数确定文件处于哪个路径下，以成功查找文件。  

3. **关于Major Compaction**
    Major Compaction的入口是`MaybeScheduleCompaction()`，整个流程会递归调用这个函数直到没有需要Major Compaction的情况。后台线程调用的真正开始工作的函数是`BackgroundCompaction()`，流程暂且不细说。最主要的改动就是要实现根据Level查找到输入文件的位置，代码上的体现就是正确构建迭代器。对于从level到level+1的Major Compaction，level层的输入文件要逐个调用`NewIterator()`构建迭代器，对于level+1层的输入文件会调用`NewTwoLevelIterator`一次性构建一个TwoLevel迭代器。关于这部分的改动如下：
    - `FindTableWithSeparation()`：添加了`level`参数，用于在`hot_cold_separation`开启时，根据level寻找文件。
    - `NewIteratorWithSeparation()`：也是多加了一个`level`参数，主要是为了能给`FindTableWithSeparation()`传递level。
    - `TwoLevelIteratorWithSeparation`类和`NewTwoLevelIteratorWithSeparation()`：新的TwoLevel迭代器，主要也是添加了`level`变量用于在不同路径下查找文件。
    - `GetFileIteratorWithSeparation()`：传递给`TwoLevelIteratorWithSeparation`类的获取指定SST文件迭代器的函数，会调用到`NewIteratorWithSeparation()`。

4. **Major Compaction的特殊情况**
    在一开始有提到，Major Compaction存在一种特殊情况，即level层仅有一个输入文件，level+1层没有输入文件，这时候原生的Leveldb只需要在Version里改一下文件的层级即可。但是由于可能会存在L2向L3的迁移，因此添加了判断，在这种情况下，会调用`filesystem`库的`std::rename`实现SSD到HDD的迁移（后续应该可以换一个函数）。

5. **关于RemoveObsoleteFile**
    不论是在Minor Compaction还是Major Compaction，最后都会调用`RemoveObsoleteFile`清理不需要的文件，新写了一个`RemoveObsoleteFileWithSeparation()`用于在`hot_cold_separation`开启时的清理。