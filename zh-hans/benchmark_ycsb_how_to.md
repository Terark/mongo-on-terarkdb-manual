## 1. 下载，安装部署 Mongo on TerarkDB

点击这里：[下载，安装部署 Mongo on TerarkDB](installation.md)

## 2. 创建测试所需的数据库以及表

- 创建测试用户及测试数据库

```bash
cd /path/to/mongo
./mongo
use database
db.createUser({user: "root", pwd: "root", roles: [ ]})
db.auth("root", "root")
db.createCollection("usertable")
```

## 3. 准备原始数据
用户可以根据自己的需要，自行选择测试数据集，一般来说要求测试数据集每行是一条完整记录，强烈建议使用真实数据进行测试。

我们为用户提供 wikipedia 的全量数据（103GB），经过 xz 高度压缩后大约 17GB，如果需要请联系我们获取下载地址。


## 4. 下载并编译 Terark 修改版的 YCSB

```bash
git clone git@github.com:Terark/YCSB.git
cd YCSB
mvn -pl com.yahoo.ycsb:mongodb-binding -am clean package
```

这个步骤会编译必须的一些模块。

## 5. 开始测试
整个测试过程，可以大致分为三个阶段，分别是：

- 数据加载（只写）
- 随机读测试（只读）
- 读写混合测试

### 5.1. 配置文件准备
根据上面三个阶段，我们可以在YCSB根目录创建三个不同的配置文件，如：

`wikipedia_load.conf`, `wikipedia_read.conf`,`wikipedia_readwrite.conf`

完整的配置选项参考[这里](https://github.com/brianfrankcooper/YCSB/wiki/Core-Properties)

每个配置文件的配置项目各不相同, 参数的大致说明：
```
workload：指定我们定制的 workload：FileWorkload
datafile：源数据的路径
fieldnames：每行数据中使用指定分隔符分割的数据项，每一个使用“,”分割，数量需要和源数据中数据项相同
writerate：读写比例
batchread：当其等于 1 时，使用 read；大于 1 时，使用 batchRead
delimiter：源数据中分割各数据项的分割符，如：“\t”，“\0”
mongodb.upsert=true：在混合测试中会把随机读出的数据插入到一个临时表中，而可能会导致主键冲突，故将 upsert 设为 true

```

mongodb.url: 参考 [YCSB/MongoDB Configuration Parameters](https://github.com/Terark/YCSB/tree/master/mongodb#mongodb-configuration-parameters)


### 5.2. 加载数据
配置文件示例(`wikipedia_load.conf `)：

```
# wikipedia_load.conf

# 需要插入的数据条数，一般是数据源的行数
recordcount=38508221

workload=com.yahoo.ycsb.workloads.FileWorkload


mongodb.url=mongodb://root:root@127.0.0.1:27017/database
mongodb.upsert=true

# 准备插入的数据源
datafile=/path/to/wikipedia.txt
# 数据源文件中，单行数据代表的各个字段名称
# 对于 wikipedia 数据可以是：cur_id,cur_namespace,cur_title,cur_text,cur_comment,cur_user,cur_user_text,cur_timestamp,cur_restrictions,cur_counter,cur_is_redirect,cur_minor_edit,cur_random,cur_touched,inverse_timestamp
fieldnames=fieldname1,fieldname2,fieldname3,fieldname4,fieldname5,fieldname6,...
# 单行数据中多个字端的字段分隔符
delimiter=\t
```


启动脚本：

```bash
./bin/ycsb load mongodb -s -P wikipedia_load.conf -threads 16 > load_wikipedia_thread_16.txt
```

输出文件在测试完成后会输出统一的统计信息，在测试过程中控制台会显示实时写入速度，和预估的写入剩余时间。

### 5.2. 随机读测试
配置文件示例(`wikipedia_read.conf `)：

```
# wikipedia_read.conf
# 数据源的行数，用来内部计算一些hash值试用，与其他阶段数值需要相同
recordcount=38508221

# 总操作数，自由定义
operationcount=64000000
workload=com.yahoo.ycsb.workloads.FileWorkload

# 使用 uniform 分布进行读测试
requestdistribution=uniform

mongodb.url=mongodb://root:root@127.0.0.1:27017/database
mongodb.upsert=true

# 批量度设置，一次读取多少数据
batchread=1
```

启动测试脚本：

```bash
./bin/ycsb run mongodb -s -P wikipedia_read.conf -threads 16 > read_wikipedia_thread_16.txt
```


### 5.4. 读写混合测试

配置文件示例(`wikipedia_readwrite.conf `)：

```
# wikipedia_readwrite.conf
# 数据源的行数，用来内部计算一些hash值试用，与其他阶段数值需要相同
recordcount=38508221
# 总操作数，自由定义
operationcount=64000000
workload=com.yahoo.ycsb.workloads.FileWorkload

# 使用 uniform 分布进行读测试
requestdistribution=uniform

mongodb.url=mongodb://root:root@127.0.0.1:27017/database
mongodb.upsert=true

# 写占的比例，0.05 表示 5%
writerate=0.05
# 批量度设置，一次读取多少数据
batchread=1
```

启动测试脚本：

```bash
./bin/ycsb run mongodb -s -P wikipedia_readwrite.conf -threads 16 > readwrite_wikipedia_thread_16.txt
```

读写混合测试说明：
1. 程序会在每次请求前生成 KEY 进行随机读取
2. 读取到的数据，会按照预定的比例写回到另一个临时表中(`usertable_for_write`)
3. 每次开始测试之前，需要先清空临时表 `usertable_for_write`
