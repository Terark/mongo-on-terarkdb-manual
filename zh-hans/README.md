## TerarkMongo 使用手册

### 1.产品简介
TerarkMongo 是一款使用 TerarkDB 存储引擎的 MongoDB 修改版，基于 TerarkDB 存储引擎的高压缩高随机读特点，我们将 MongoDB 的压缩率和随机读能力也全面的提高了

TerarkMongo 使用了 MongoRocks 作为 TerarkDB 和 MongoDB 的适配层：

- MongoRocks 是 MongoDB 的一个存储引擎适配层，将 RocksDB 对接到 MongoDB，使用该引擎的 MongoDB 占用空间更小、性能更高。为了不失简洁，我们也简称 MongoRocks 就是搭配了 MongoRocks 的 MongoDB。
- TerarkDB 是基于 Terark 可检索压缩专利技术的 RocksDB 发行版，完全兼容 RocksDB，为 MongoRocks 提供更高效的压缩率和随机读性能。


### 2.优势

TerarkDB 完全兼容 RocksDB，所以 TerarkMongo 在功能上完全等同于 MongoRocks 官方版。

作为 MongoDB 存储引擎，RocksDB 对比 WiredTiger，在某些场景下已表现出巨大的优势，TerarkDB 则进一步扩大了这个优势:

- TerarkDB 的存储压缩率 远高于 官方 RocksDB
  - 即使单比较磁盘文件，TerarkDB 的压缩率也远高于官方 RocksDB
  - 进一步比较内存占用，TerarkDB 的对比优势更加明显：可检索压缩解决了双缓存问题（传统块压缩很难解决）
  - 相比传统存储引擎，使用相同的内存，TerarkDB 可以装入所有数据，而其它存储引擎只能装入一小部分数据
- TerarkDB 的随机读性能 远高于 官方 RocksDB
  - TerarkDB 的可检索压缩专利技术，直接在压缩的数据上执行搜索，无需预先解压（传统块压缩需要先解压数据块）
  - 引擎层面随机读性能相比官方 RocksDB 提升 200 倍以上，MongoDB 层有固定的性能损耗，最终提升一般在 10 倍以上

### 3.理想场景
TerarkMongo 在大多数场景下都表现优异，但我们也有自己的理想场景：

- 文本数据较多的场景，如用户信息、点评数据、论坛数据、标签数据等
- 读远多于写的场景
  - TerarkDB 存储引擎的压缩相对较慢（但读的时候不需要解压）
- 随机读较多的场景

### 4.联系我们
- contact@terark.com
- [www.terark.com](http://www.terark.com)
