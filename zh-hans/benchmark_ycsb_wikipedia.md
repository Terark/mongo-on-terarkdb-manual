## 简介

YCSB 的英文全称是 Yahoo! Cloud Serving Benchmark，是 Yahoo 公司的一个用来对云服务进行基础测试的工具。目标是促进新一代云数据服务系统的性能比较。本测试使用修改版的 YCSB 分别向官方原版 MongoDB 和 [TerarkMongo](http://terark.com/zh/databases/mongodb) 导入 **38,508,221** 条 [wikipedia](https://dumps.wikimedia.org/backup-index.html) 文章数据，并测试在不同内存下两者的读写性能。

由于原版 YCSB 的数据都是纯随机生成的字符串，离用户的真实场景相差较大，所以我们修改了 [YCSB](https://github.com/Terark/YCSB/tree/dev) 并添加了一个 [FileWorkload](https://github.com/Terark/YCSB/blob/master/README-terark.md)，以使用接近真实场景的数据来对数据库进行测试。

测试的数据库有:
 - [TerarkMongo](http://terark.com/zh/databases/mongodb)，存储引擎为 TerarkDB，**target_file_size_base** 设为 **2G**，后记为 TerarkDB_2G
 - [TerarkMongo](http://terark.com/zh/databases/mongodb)，存储引擎为 TerarkDB，**target_file_size_base** 设为 **24G**，后记为 TerarkDB_24G
 - 官方原版的 [MongoDB](https://www.mongodb.com/)，版本为 **v3.2.13**，存储引擎为 WiredTiger，后记为 WiredTiger

TerarkDB 的 **target_file_size_base** 选项用于设置数据压缩后生成的数据文件（sst）的大小。当 **target_file_size_base** 设置为 **2G** 时 TerarkDB 的写放大较小，生成的数据文件较小（≈ 2G），便于管理；当 **target_file_size_base** 设置为 **24G** 时，能将所有的 wikipedia 文章数据压缩到一个数据文件（sst）中，此时 TerarkDB 有最高的压缩率和最好的性能。

## 测试平台

<table>
  <tr>
    <th>CPU</th>
    <td>Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz （共 24 核 48 线程）</td>
  </tr>
  <tr>
    <th>内存</th>
    <td>DDR4 16G @ 2400 MHz x 8 （共 <strong>128 G</strong>）</td>
  </tr>
  <tr>
    <th>File system</th>
    <td>lenovo's web file system</td>
  </tr>
  <tr>
    <th>操作系统</th>
    <td>CentOS 7.3.1611</td>
  </tr>
</table>

测试中使用了两台上表配置且在同一内网的服务器，并在这两台服务器上分别搭建了一主一从的 MongoDB 和 TerarkMongo 集群。YCSB 测试程序均跑在同内网下的不同机器中。

下文 G, GB 指 2<sup>30</sup>，而非 10<sup>9</sup>。

## 数据导入

原版的 YCSB 使用随机生成的字符串作为数据源，这样的数据无法体现压缩算法的优劣。通过使用修改后的 YCSB，我们以 [wikipedia](https://dumps.wikimedia.org/backup-index.html) dump 出来的文章数据作为数据源（数据示例可见**附录1**），这些数据共有这些数据共有 **38,508,221** 条，总大小为 **102.1 GB**，平均每条约 **2.8 KB**。其中，将数据中前三个字段(id、namepsace、title)作为主键（_id）。

数据导入后，数据库的尺寸大小比较如下：
<table>
<tr>
  <th colspan="2" align="right">数据库尺寸</th>
  <th>压缩率</th>
  <th rowspan="4"></th>
  <th>数据条数</th>
  <th>单条尺寸</th>
  <th>总尺寸</th>
</tr>
<tr>
  <td align="right">TerarkDB_2G</td>
  <td align="right">27.3 G</td>
  <td align="right">26.7% 或 3.74倍</td>
  <td align="center" rowspan="3">38,508,221</td>
  <td align="center" rowspan="3">2.8 KB</td>
  <td align="center" rowspan="3">102.1 G</td>
</tr>
<tr>
  <td align="right">TerarkDB_24G</td>
  <td align="right">23.4 G</td>
  <td align="right">22.9% 或 4.36倍</td>
</tr>
<tr>
  <td align="right">WiredTiger</td>
  <td align="right">58.3 G</td>
  <td align="right">57.1% 或 1.75倍</td>
</tr>
</table>

将数据库磁盘占用以及随机读测试中最大内存用量用图表展示如下：

![size_memory](../images/benchmark_ycsb_wikipedia/size_memory.svg)

## 测试结果

我们进行了两种测试：

- 随机读测试（read）
- 批量随机读测试，每次 query 随机读取 20 条数据（batch_read）
- 随机读、写混合测试，读写比例为 9 ：1（read_write）

这三种测试分别在 128G、24G 内存下运行，其中 24G 内存限制为使用内存挤占工具挤占一定数量的内存（不可换出）确保各数据库能使用的内存为 24G。

每次测试中 WiredTiger 的 **cacheSizeGB** 总是设置为可用内存的 **60% - 1GB**（60% of RAM minus 1 GB），TerarkDB 的 **softZipWorkingMemLimit** 和 **hardZipWorkingMemLimit** 分别设置为可用内存的 **1/8** 和 **1/4**。

所有的测试均使用 32 个线程，每次测试持续 30 分钟。

测试结果总览如下：
<table>
    <tr>
             <th>内存</th><th>测试类型</th><th>TerarkDB_2G</th><th>TerarkDB_24G</th><th>WiredTiger</th>
    </tr>
    <tr align="right">
             <td rowspan="3">128G</td> <td align="left">read</td> <td>134,188</td> <td>140,948</td> <td>131,485</td>
    </tr>
    <tr align="right">
             <td align="left">batch_read</td> <td>317,480</td> <td>328,220</td> <td>336,620</td>
    </tr>
    <tr align="right">
             <td align="left">read_write</td> <td>56,336</td> <td>74,703</td> <td>10,601</td>
    </tr>
    <tr align="right">
             <td rowspan="3">24G</td><td align="left">read</td> <td>25,192</td> <td>28,059</td> <td>2,822</td>
    </tr>
    <tr align="right">
             <td align="left">batch_read</td> <td>22,500</td> <td>23,260</td> <td>2,860</td>
    </tr>
    <tr align="right">
             <td align="left">read_write</td> <td>10,831</td> <td>6,702</td> <td>3,073</td>
    </tr>
</table>

将上表数据以图表形式详细展示如下：

### 1. 128G memory

![rps_128g](../images/benchmark_ycsb_wikipedia/qps_128g.svg)

128G 内存对 TerarkDB 和 WiredTiger 都够用，实际上，TerarkDB_2G（**target_file_size_base** 为 **2G** 的 TerarkDB） 使用了 27G 内存，TerarkDB_24G（**target_file_size_base** 为 **24G** 的 TerarkDB） 仅使用了 23G 内存，而 WiredTiger 则使用了 117G 内存（进程内存 + 系统缓存）。

随机读 95/99 分位延迟如下：

![read_latency_128g](../images/benchmark_ycsb_wikipedia/read_latency_128g.svg)

批量随机读 95/99 分位延迟如下：

![batchread_latency_128g](../images/benchmark_ycsb_wikipedia/batchread_latency_128g.svg)

读写混合 95/99 分位延迟如下：

![readwrite_latency_128g](../images/benchmark_ycsb_wikipedia/readwrite_latency_128g.svg)

<hr />

### 2. 24G memory

![rps_24g](../images/benchmark_ycsb_wikipedia/qps_24g.svg)

24G 内存对 TerarkDB_2G（**target_file_size_base** 为 **2G** 的 TerarkDB） 和 WiredTiger 都不够用，故数据库性能相较于 128G 内存下都有较大的下降，但是 WiredTiger 的内存缺口比 TerarkDB_2G 大的多，从而 TerarkDB_2G 的性能远高于 WiredTiger。TerarkDB_24G（**target_file_size_base** 为 **24G** 的 TerarkDB） 在不限制内存的情况下最大内存使用仅有 23G， 24G 内存本该足够 TerarkDB_24G 将所有数据加载进内存，但是因磁盘及操作系统的原因，TerarkDB_24G 的最大内存使用只能达到 19G，既仍有部分数据未能加载进内存。所以 TerarkDB_24G 相对与 TerarkDB_2G 没有显著的性能提升，但是仍远高于 WiredTiger。

测试时使用的磁盘为共享网盘，其 IO 较低，且对于单个文件的随机读 IOPS 有一定上限，多个线程同时读一个文件的性能较低，在进行测试前都进行了足够长时间的预热。而 TerarkDB_24G 则刚好将所有数据压缩成一个文件，共享网盘对 TerarkDB_24G 的影响更大，Terark_24G 的读写混合测试结果较 Terark_2G 有较大程度的下降。

随机读 95/99 分位延迟如下：

![read_latency_24g](../images/benchmark_ycsb_wikipedia/read_latency_24g.svg)

批量随机读 95/99 分位延迟如下：

![batchread_latency_24g](../images/benchmark_ycsb_wikipedia/batchread_latency_24g.svg)

读写混合 95/99 分位延迟如下：

![readwrite_latency_24g](../images/benchmark_ycsb_wikipedia/readwrite_latency_24g.svg)

## 附录1

数据示例：
```
13	0	'AfghanistanHistory'	'#REDIRECT [[History of Afghanistan]] {{R from CamelCase}}'	'cat rd'	750223	'Rory096'	'20060908041552'	''	0	1	0	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79939091958447'
14	0	'AfghanistanGeography'	'#REDIRECT [[Geography of Afghanistan]] {{R from CamelCase}}'	'1 revision from [[:nost:AfghanistanGeography]]: import old edit, see [[User:Graham87/Import]]'	194203	'Graham87'	'20110110035619'	''	0	1	1	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79889889964380'
15	0	'AfghanistanPeople'	'#REDIRECT [[Demographics of Afghanistan]] {{R from CamelCase}}'	'Robot: Fixing double redirect to [[Demographics of Afghanistan]]'	279219	'RussBot'	'20140710191843'	''	0	1	1	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79859289808156'
18	0	'AfghanistanCommunications'	'#REDIRECT [[Communications in Afghanistan]] {{R from CamelCase}}'	'cat rd'	750223	'Rory096'	'20060908041442'	''	0	1	0	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79939091958557'
19	0	'AfghanistanTransportations'	'#REDIRECT [[Transport in Afghanistan]] {{R from CamelCase}} {{R unprintworthy}}'	''	930338	'Asklepiades'	'20110122003720'	''	0	1	0	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79889877996279'
20	0	'AfghanistanMilitary'	'#REDIRECT [[Afghan Armed Forces]] {{R from CamelCase}}'	'Robot: Fixing double redirect to [[Afghan Armed Forces]]'	11292982	'EmausBot'	'20130604184503'	''	0	1	1	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79869395815496'
21	0	'AfghanistanTransnationalIssues'	'#REDIRECT [[Foreign relations of Afghanistan]] {{R from CamelCase}}'	'{{R from CamelCase}}'	241822	'Gurch'	'20060401120842'	''	0	1	1	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79939598879157'
23	0	'AssistiveTechnology'	'#REDIRECT [[Assistive_technology]] {{R from CamelCase}}'	'cat rd'	750223	'Rory096'	'20060908041700'	''	0	1	0	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79939091958299'
24	0	'AmoeboidTaxa'	'#REDIRECT [[Amoeba]] {{R from CamelCase}}'	'Bot: Fixing double redirect to [[Amoeba]]'	15934865	'Invadibot'	'20140929222603'	''	0	1	1	RAND()	DATE_ADD('1970-01-01', INTERVAL UNIX_TIMESTAMP() SECOND)+0	'79859070777396'
```

更多数据示例在[这里](https://raw.githubusercontent.com/Terark/mongo-on-terarkdb-manual/master/zh-hans/ycsb_dateset_example.txt)
