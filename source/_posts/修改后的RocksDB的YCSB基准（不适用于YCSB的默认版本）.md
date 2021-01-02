---
title: 修改后的RocksDB的YCSB基准（不适用于YCSB的默认版本）
date: 2021-01-02 09:35:03
tags: ['RocksDB','YCSB']
categories:
cover: https://www.researchgate.net/profile/Daehyeok_Kim/publication/326906419/figure/tbl2/AS:671484594880515@1537105811577/The-percentage-of-read-update-insert-modify-read-and-update-and-scan-operations-in.png
---
<meta name="referrer" content="no-referrer" />

# 修改后的RocksDB的YCSB基准（不适用于YCSB的默认版本）

为了评估未经修改在Facebook上发布的RocksDB版本的性能，只需参考YCSB [README](https://github.com/brianfrankcooper/YCSB/blob/master/rocksdb/README.md)文档。Maven还可以下载RocksDB软件包并运行YCSB。

但是，要通过修改RocksDB的代码来评估修改后的RocksDB版本的性能，您需要使用JAVA构建RocksDB，创建jni程序包，然后使用该程序包运行YCSB。

就我而言，为了将开放通道SSD用作RocksDB的存储，我修复了RocksDB的存储后端，以使用外部库（liblightnvm）而不是posix I / o使用新的后端。需要其他工作。但是，对于常规代码修改或算法修改，本文中的说明就足够了。

## RocksDB jni软件包生成

首先，您需要无论什么版本以及如何修改它，都能正常工作的RocksDB源代码。移动到下一个之前，让我们运行RocksDB源代码，我有`make release`和`db_bench`看它是否工作。

如果修改后的RocksDB版本正常运行，则需要JAVA编译RocksDB。

在RocsDB源代码中，java目录与jni包相关。由于压缩方法未全部使用五种方法，因此仅保留了所需的方法并被注释掉（以节省时间）。

- 如下例所示修改Makefile（仅快照压缩）。

```makefile
# A version of each $(LIBOBJECTS) compiled with -fPIC and a fixed set of static compression libraries
java_static_libobjects = $(patsubst %,jls/%,$(LIBOBJECTS))
CLEAN_FILES += jls

ifneq ($(ROCKSDB_JAVA_NO_COMPRESSION), 1)
# JAVA_COMPRESSIONS = libz.a libbz2.a libsnappy.a liblz4.a libzstd.a
JAVA_COMPRESSIONS = libsnappy.a
endif

# JAVA_STATIC_FLAGS = -DZLIB -DBZIP2 -DSNAPPY -DLZ4 -DZSTD
JAVA_STATIC_FLAGS = -DSNAPPY
# JAVA_STATIC_INCLUDES = -I./zlib-$(ZLIB_VER) -I./bzip2-$(BZIP2_VER) -I./snappy-$(SNAPPY_VER) -I./lz4-$(LZ4_VER)/lib -I./ zstd-$(ZSTD_VER)/lib
JAVA_STATIC_INCLUDES = -I./snappy-$(SNAPPY_VER)
```

对于Java编译，请使用`Makefile`rocksdbjavastaticrelease规则。由于它不是跨平台的，因此该行已被注释掉。通过删除此行，跨平台的Java包被取消，因此`ibrocksdbjni-*.jnilib`删除该部分。

- 结果，如下面的示例所示修改Makefile。

```makefile
rocksdbjavastaticrelease: rocksdbjavastatic
  # cd java/crossbuild && vagrant destroy -f && vagrant up linux32 && vagrant halt linux32 && vagrant up linux64 && vagrant halt linux64
  cd java;jar -cf target/$(ROCKSDB_JAR_ALL) HISTORY*.md
  # cd java/target;jar -uf $(ROCKSDB_JAR_ALL) librocksdbjni-*.so librocksdbjni-*.jnilib
  cd java/target;jar -uf $(ROCKSDB_JAR_ALL) librocksdbjni-*.so
  cd java/target/classes;jar -uf ../$(ROCKSDB_JAR_ALL) org/rocksdb/*.class org/rocksdb/util/*.class
```

如果正常修改了Makefile，请`make rocksdbjavastaticrelease`尝试通过命令进行构建。如果您有足够的核心`-j16`，则可以提供类似的选项。

如果构建成功，那么`rocksdb/java/target/rocksdbjni-5.18.3.jar`将创建jni软件包文件。`5.18.3`根据您正在使用的RocksDB的源代码，此处创建的版本会有所不同。

以这种方式创建的jni包将在以后的YCSB操作中使用。

### 注意

在实验室openstack的新开虚拟机上可能会有些问题，尤其是java环境的问题（默认java环境是需要自己配的），要先把java8的环境完整配置好，然后maven里面如果显示缺少assertj的jar包，也需要额外下载。



## YCSB编译

### 获取YCSB代码并进行配置

通过以下命令接收YCSB代码

```shell
git clone https://github.com/brianfrankcooper/YCSB.git
git checkout 0.15.0
```

要下载YCSB的代码模型依赖项，请将以下内容添加`YCSB/core/pom.xml`到文件的`<dependencies>`下部。(pom里面本身是有这两行的，如果没有的话就只能自己单独下。)

```html
<dependency>
      <groupId>org.apache.htrace</groupId>
      <artifactId>htrace-core4</artifactId>
      <version>4.1.0-incubating</version>
</dependency>
<dependency>
      <groupId>org.hdrhistogram</groupId>
      <artifactId>HdrHistogram</artifactId>
      <version>2.1.4</version>
</dependency>
```

### YCSB编译

通过Maven绑定RocksDB。请注意，这`mvm clean package`花费了很长时间，因为仅使用命令即可下载所有数据库应用程序（HBase，MongoDB等）的模块的依赖项。

- 以下命令是绑定RocksDB的命令。

```shell
mvn -pl com.yahoo.ycsb:rocksdb-binding -am clean package
```

您可以看到由于添加了依赖关系而在`YCSB/rocksdb/target/dependency/`路径中`htrace-core4-4.1.0-incubating.jar` `HdrHistogram-2.1.4.jar`创建了一个文件。

如果您查看YCSB目录中*YCSB / pom.xml*文件中的部分`<rocksdb.version>5.11.3</rocksdb.version>`，则可以看到YCSB下载并使用了RocksDB版本5.11.3的jni包。通过修改此部分，可以通过接收所需版本的jni软件包来运行YCSB。（但是它仍然是在Facebook上发布的版本。我们想要的是通过直接修改源代码来评估RocksDB的性能。）

在Linux系统上，`bin/ycsb.sh`您可以通过运行文件来操作多个YCSB基准测试。

- 使用以下命令加载YCSB-workloada的数据。`workloads/workloada`在文件中指定了工作负载a的配置，并且还提供了将存储数据的目录作为选项。

```shell
./bin/ycsb.sh load rocksdb -s -P workloads/workloada -p rocksdb.dir=/home/ubuntu/ycsbdata
```

如果检查LOG文件的前一部分，则可以看到RocksDB的5.11.3版本（YCSB的默认版本）仍在运行。

**（重要！）将在上一个过程中创建的jni软件包文件复制`rocksdb/java/target/rocksdbjni-5.18.3.jar`到该`YCSB/rocksdb/target/dependency/`路径。并`rocksdbjni-5.11.3.jar`删除现有文件。**

然后，`rocsdb.dir`删除RocksDB数据路径中指定的目录路径中的所有文件，然后再次执行laod命令。

```shell
./bin/ycsb.sh load rocksdb -s -P workloads/workloada -p rocksdb.dir=/home/ubuntu/ycsbdata
```

如果查看LOG文件的前面，可以看到出现了RocksDB所需的版本（我修改和构建的版本）。

加载数据后，您可以通过轮流多个工作负载来执行性能评估。

- 以下命令对`home/rocky/ycsbdata`目录中现有的数据库执行工作负载-a。

```shell
./bin/ycsb.sh run rocksdb -s -P workloads/workloada -p rocksdb.dir=/home/ubuntu/ycsbdata
```

### YCSB下的参数设置

由于字段计数和文件长度较小，因此工作负载的默认配置在很短的时间内完成，因此必须将其更改为足够的大小。`YCSB/worklaods/`可以通过修改路径中的文件来更改每个工作负载参数。

`YCSB\rocksdb\src\main\java\com\yahoo\ycsb\db\rocksdb\RocksDBClient.java`可以在文件中设置RocksDB的各种参数和配置。

`initRocksDB()`该功能下的以下代码与RocksDB选项相关。查找改变参照该选项的功能，`java/src/main/java/org/rocksdb/Options.java`文件和`java/src/main/java/org/rocksdb/DBOptions.java`功能的文件在原型RocksDB源代码并更改所需的选项。

```java
    if(cfDescriptors.isEmpty()) {
      final Options options = new Options()
          .optimizeLevelStyleCompaction()
          .setCreateIfMissing(true)
          .setCreateMissingColumnFamilies(true)
          .setIncreaseParallelism(rocksThreads)
          .setMaxBackgroundCompactions(rocksThreads)
          .setInfoLogLevel(InfoLogLevel.INFO_LEVEL);
      dbOptions = options;
      return RocksDB.open(options, rocksDbDir.toAbsolutePath().toString());
    } else {
      final DBOptions options = new DBOptions()
          .setCreateIfMissing(true)
          .setCreateMissingColumnFamilies(true)
          .setIncreaseParallelism(rocksThreads)
          .setMaxBackgroundCompactions(rocksThreads)
          .setInfoLogLevel(InfoLogLevel.INFO_LEVEL);
      dbOptions = options;

      final List<ColumnFamilyHandle> cfHandles = new ArrayList<>();
      final RocksDB db = RocksDB.open(options, rocksDbDir.toAbsolutePath().toString(), cfDescriptors, cfHandles);
      for(int i = 0; i < cfNames.size(); i++) {
        COLUMN_FAMILIES.put(cfNames.get(i), new ColumnFamily(cfHandles.get(i), cfOptionss.get(i)));
      }
      return db;
    }
```

如果`RocksDBClient.java`您已经修改了文件，请输入以下命令来重新绑定（`mvn -pl com.yahoo.ycsb:rocksdb-binding -am clean package`命令中`clean`省略了该命令。）

```shell
mvn -pl com.yahoo.ycsb:rocksdb-binding -am package
```

然后`rocksdb/java/target/rocksdbjni-5.11.3.jar`删除重新创建的文件。 `load`如果`run`运行命令，您将能够检查RocksDB是否可以使用所需的选项。