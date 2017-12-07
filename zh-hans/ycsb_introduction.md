# YCSB 简介

YCSB 的英文全称是：**Yahoo! Cloud Serving Benchmark (YCSB) **。是 Yahoo 公司的一个用来对云服务进行基础测试的工具。目标是促进新一代云数据服务系统的性能比较。

YCSB 支持对绝大部分主流数据库进行测试，包括 MySQL, MongoDB, Cassandra, HBase 等等

## 随机生成数据与真实数据

测试的第一步是灌数据，灌的数据怎么来呢？YCSB 数据是随机生成的，为了更接近真实情况，我们对 YCSB 做了一些修改，让它可以使用用户提供的真实数据。

在这个测试中，我们使用的真实数据有:
- [Amazon movie data](https://snap.stanford.edu/data/web-Movies.html)
  - 原始数据 9.1G, 数据清洗后 8G，数据条数约 800万，平均每条约 1K
- Wikipedia 英文版全部文本
  - 原始数据约 107G，数据条数约 4000万，平均每条约 2.8K
