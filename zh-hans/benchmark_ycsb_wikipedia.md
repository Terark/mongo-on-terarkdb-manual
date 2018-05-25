## 简介

YCSB 的英文全称是 Yahoo! Cloud Serving Benchmark，是 Yahoo 公司的一个用来对云服务进行基础测试的工具。目标是促进新一代云数据服务系统的性能比较。本测试使用修改版的 YCSB 分别向官方原版 MongoDB 和 [TerarkMongo](http://terark.com/zh/databases/mongodb)导入 **38,508,221** 条 [wikipedia](https://dumps.wikimedia.org/backup-index.html) 文章数据，并测试在不同内存下两者的读写性能。

由于原版 YCSB 的数据都是纯随机生成的字符串，离用户的真是场景相差较大，所以我们修改了 [YCSB](https://github.com/Terark/YCSB/tree/dev) 并添加了一个 [FileWorkload](https://github.com/Terark/YCSB/blob/master/README-terark.md)，以使用接近真实场景的数据来对数据库进行测试。

测试的数据库有 [TerarkMongo](http://terark.com/zh/databases/mongodb) 和官方原版的 [MongoDB](https://www.mongodb.com/)，MongoDB 的版本为 **v3.2.13**。

## 测试平台

<table>
  <tr>
    <th>CPU</th>
    <td>Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz （共 24 核 48 线程）</td>
  </tr>
  <tr>
    <th>内存</th>
    <td>DDR4 16G @ 1866 MHz x 12 （共 <strong>128 G</strong>）</td>
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

测试中使用了两台以上配置的服务器，并在这两台服务器上分别搭建了一主一从的 MonoDB 和 TerarkMongo 集群。YCSB 测试客户端均跑在同内网下的不同机器中。

下文 G, GB 指 2<sup>30</sup>，而非 10<sup>9</sup>。

## 数据导入

原版的 YCSB 使用随机生成的字符串作为数据源，这样的数据无法体现压缩算法的优劣。通过使用修改后的 YCSB，我们以 [wikipedia](https://dumps.wikimedia.org/backup-index.html) dump 出来的文章数据作为数据源（数据示例可见**附录1**），这些数据共有这些数据共有 **38,508,221** 条，总大小为 **102.1 GB**，平均每条约 **2.8 KB**。其中，将数据中前三个字段(id、namepsace、title)作为主键（_id）。

数据导入后，数据库的尺寸大小比较如下：

<table>
<tr>
  <th colspan="2" align="right">数据库尺寸</th>
  <th>压缩率</th>
  <th rowspan="3"></th>
  <th>数据条数</th>
  <th>单条尺寸</th>
  <th>总尺寸</th>
</tr>
<tr>
  <td align="right">TerarkMongo</td>
  <td align="right">27.3 G</td>
  <td align="right">26.7% 或 3.74倍</td>
  <td align="center" rowspan="2">38,508,221</td>
  <td align="center" rowspan="2">2.8 KB</td>
  <td align="center" rowspan="2">102.1 G</td>
</tr>
<tr>
  <td align="right">MongoDB</td>
  <td align="right">58.3 G</td>
  <td align="right">57.1% 或 1.75倍</td>
</tr>
</table>

将数据库磁盘占用以及测试中最大内存用量用图表展示如下：

![size_memory](../images/benchmark_ycsb_wikipedia/size_memory.svg)

## 测试结果

我们进行了两种测试：

- 随机读测试（read）
- 批量随机读测试，每次 query 随机读取 20 条数据（batch_read）
- 随机读、写混合测试，读写比例为 9 ：1（read_write）

这两种测试分别在 128G、24G 内存下运行，其中 24G 内存限制为使用内存挤占工具挤占一定数量的内存（不可换出）确保各数据库能使用的内存为 24G。

每次测试中 MonoDB 的 **storage.wiredTiger.engineConfig.cacheSizeGB** 总是设置为可用内存的 **60%** - 1GB（60% of RAM minus 1 GB），TerarkMongo 的 **softZipWorkingMemLimit** 和 **hardZipWorkingMemLimit** 分别设置为可用内存的 **1/8** 和 **1/4**。

所有的测试均使用 32 个线程，每次测试持续 30 分钟。

测试结果总览如下：

<table>
    <tr>
             <th>内存</th><th>测试类型</th><th>TerarkMongo</th><th>MongoDB</th>
    </tr>
    <tr align="right">
             <td rowspan="3">128G</td> <td align="left">read</td> <td>134,188</td> <td>131,485</td>
    </tr>
    <tr align="right">
             <td align="left">batch_read</td> <td>317,480</td> <td>336,620</td>
    </tr>
    <tr align="right">
             <td align="left">read_write</td> <td>56,336</td> <td>10,601</td>
    </tr>
    <tr align="right">
             <td rowspan="3">24G</td><td align="left">read</td> <td>25,192</td> <td>2,822</td>
    </tr>
    <tr align="right">
             <td align="left">batch_read</td> <td>22,500</td> <td>2,860</td>
    </tr>
    <tr align="right">
             <td align="left">read_write</td> <td>10,831</td> <td>3,073</td>
    </tr>
</table>

将上表数据以图标形式展示如下：

### 1. 128G memory

![rps_128g](../images/benchmark_ycsb_wikipedia/qps_128g.svg)

128G 内存对 TerarkMongo 和原版 Mongo 都够用，实际上，TerarkMongo 只使用了 27G，而原版 Mongo 则使用了 117G 内存（进程内存 + 系统缓存）。

随机读 95/99 分位延迟如下：

![read_latency_128g](../images/benchmark_ycsb_wikipedia/read_latency_128g.svg)

批量随机读 95/99 分位延迟如下：

![batchread_latency_128g](../images/benchmark_ycsb_wikipedia/batchread_latency_128g.svg)

读写混合 95/99 分位延迟如下：

![readwrite_latency_128g](../images/benchmark_ycsb_wikipedia/readwrite_latency_128g.svg)

<hr />

### 2. 24G memory

![rps_24g](../images/benchmark_ycsb_wikipedia/qps_24g.svg)

24G 内存对 TerarkMongo 和原版 Mongo 都不够用。因为测试时使用的磁盘为共享网盘，其 IO 较低，故数据库性能相较于 128G 内存下都有较大的下降，但是原版 Mongo 的内存缺口比 TerarkMongo 大的多，从而 TerarkMongo 的性能远高于原版 Mongo。

随机读 95/99 分位延迟如下：

![read_latency_24g](../images/benchmark_ycsb_wikipedia/read_latency_24g.svg)

批量随机读 95/99 分位延迟如下：

![batchread_latency_24g](../images/benchmark_ycsb_wikipedia/batchread_latency_24g.svg)

读写混合 95/99 分位延迟如下：

![readwrite_latency_24g](../images/benchmark_ycsb_wikipedia/readwrite_latency_24g.svg)
