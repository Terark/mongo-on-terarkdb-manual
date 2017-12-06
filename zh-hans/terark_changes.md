# Terark 的修改

## 对 MongoDB 的修改
1. `db.adminCommand({setParameter:1, rocksdbCompact: 2})` compact 所有 sst 到最底层
1. `db.adminCommand({setParameter:1, terarkZipMinLevel: -1})` 修改 `terarkZipMinLevel`
1. Initial sync 开始与结束调用 engine 虚函数 prepareInitialSync/finishInitialSync
  - 这两个虚函数是 Terark 新增的，用于优化引擎的 initial sync

## 对 MongoRocks 的修改
1. 分离 mongo 的 oplog 到独立的 ColumnFamily ，该 ColumnFamily 禁用自动 compact
1. oplog collection 在 Truncate 之后 Flush 并整体 lv0->lv0 compact
1. 增加 `MongoRocksOplogPropertiesCollector` ，按 prefix 收集必要的 oplog 信息
1. 启动时修正 oplog collection 的 numRecords 与 dataSize
1. 禁用 `RocksEngine::initRsOplogBackgroundThread` 创建后台线程的逻辑
1. 在 RocksEngine 开启一个独立的线程统一清理 oplog
