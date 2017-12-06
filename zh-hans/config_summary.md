# 配置参数

Terark 相关配置全都都在 storage.rocksdb.terarkdb 下，包含以下条目

|arg name                |default|comments
|-----------------------:|------:|-
|enabled                 |false  |总开关，如果关闭，则使用 rocksdb
|indexNestLevel          |3      |索引嵌套层数嵌套越高压缩率越高，同时性能损失越大
|indexTempLevel          |4      |级别越高构建时使用计划外内存越小，同时构建速度小幅度下降
|checksumLevel           |3      |数据校验级别。加载 sst 文件时会对数据进行校验
|minPreadLen             |16     |pread开关，-1禁用，0总是启动，大于0，平均长度超过后启用
|cacheCapacityBytes      |0      |user space cache 尺寸，0 表示禁用，请勿设置过小
|cacheShards             |17     |sharding 是为了减小 user space cache 中的 lru 锁冲突<br/>应该设置为一个`素数`
|entropyAlgo             |none   |熵编码设置，开启后小幅度提升压缩率，读取性能会下降一些
|terarkZipMinLevel       |-1     |超过在该层则使用 Terark 构建 sst 文件
|useSuffixArrayLocalMatch|false  |后缀数组匹配数据，小幅度提高压缩率，压缩速度会下降很多
|warmUpIndexOnOpen       |true   |加载 sst 预热索引
|warmUpValueOnOpen       |false  |加载 sst 预热数据，内存不足以装下所有 sst 文件时请勿启用
|estimateCompressionRatio|0.2    |估算压缩率
|sampleRatio             |0.03   |数据采样率
|localTempDir            |/tmp   |临时文件目录，推荐把临时目录和数据目录放置在不同的磁盘
|cleanTempDir            |false  |启动时清空临时目录。多实例共享临时目录请勿打开该选项
|indexType               |IL_256 |索引使用的 Succinct 实现
|initialWriteBufferSize  |1G     |initial sync 时设置 memtable 到该大小
|softZipWorkingMemLimit  |16G    |软限制 compact 的内存用量
|hardZipWorkingMemLimit  |32G    |硬限制 compact 的内存用量
|smallTaskMemory         |1.2G   |小 compact (或 memtable flush) 的优先级更高
|indexCacheRatio         |0      |典型值 0 或者较小的值，如 0.003
|zipThreads              |8      |压缩数据使用的线程数


