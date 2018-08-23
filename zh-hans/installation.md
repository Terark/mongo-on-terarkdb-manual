## 安装部署

### 1.环境要求


### 2.下载安装包
- 进入 [Terark.com](terark.com)，选择 TerarkMongo
- 点击"下载"，进入下载页面
- 填写表单即可获取最新的下载链接，如果在下载页面没有看到适合您平台的安装包（可通过文件名识别），请联系我们进一步支持
  - 目前我们仅提供 Linux 环境下，64 位操作系统的安装版本
  - 文件名中的 `bmi2-0` 表示不支持 `bmi2` CPU 指令，`bmi2-1` 表示支持 `bmi2` CPU 指令, 您可以通过`cat /proc/cpuinfo  | grep bmi2` 查看您的 CPU 是否支持 `bmi2`

### 3.安装
目前我们提供针对不同平台打包好的压缩包，您可以在下载后解压到任意目录即可直接使用。

### 4.启动
解压后根目录下有个启动脚本 `mongod.sh`，直接运行该脚本即可启动，如果希望以后台方式启动，修改该脚本，在命令行加入 `--fork`，并且，需要将 `config` 文件中的 `dbPath` 和 `localTempDir` 改为绝对路径。

```bash
env LD_LIBRARY_PATH=`pwd`/lib:$LD_LIBRARY_PATH  \
    TerarkUseDivSufSort=1                       \
    ./mongod -f config --fork
# --fork 表示以后台方式启动
# -f config 中的 "config" 是配置文件名
```

### 5. 

配置文件说明：

如下 config 中的配置选项是按照 `系统内存 = 16G` 来优化的，也可以正常工作于内存 8G 以上(包括大于16G)的系统：


```
net:
  bindIp: "0.0.0.0"
  port: 27017
storage:
  dbPath: "data" # 必须提前创建数据目录，如果以 --fork 方式启动，必须是绝对路径
  engine: "rocksdb"
  rocksdb:
    maxWriteMBPerSec: 200 # 文件写入速度限流，根据 SSD 的性能和业务需求调整
    terarkdb:
      enabled: true                   # 如果为 false，表示使用原版 rocksdb
      localTempDir: "data/terark-tmp" # 必须提前创建，如果以 --fork 方式启动，必须是绝对路径
      cleanTempDir: true              # 启动时自动清理残留 temp 文件（mongod异常终止时会有残留文件）
      softZipWorkingMemLimit: 2147483648  #   2G，如果不设置，默认是总内存的 1/8
      hardZipWorkingMemLimit: 4294967296  #   4G，如果不设置，默认是总内存的 1/4
      smallTaskMemory: 536870912          # 512M，如果不设置，默认是总内存的 1/32
```


