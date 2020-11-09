# Flink-chdfs

Flink-chdfs 是腾讯云 云CHDFS 针对Flink的文件系统实现，并且支持了recoverwriter接口。 Flink可以基于该文件系统实现读写CHDFS上的数据以及作为流应用的状态后端。

## 使用环境

### 系统环境

支持Linux系统

### 软件依赖

Flink 1.10.0
Flink 1.11.0


## 使用方法

### 获取Flink-chdfs 发行包

    下载地址：[Flink-chdfs release](https://github.com/tencentyun/flink-chdfs/releases)


### 安装Flink-chdfs 依赖

1.执行`mkdir ${FLINK_HOME}/plugins/chdfs-hadoop`,  在`${FLINK_HOME}/plugins`目录下创建flink-chdfs-hadoop插件目录；

2.将对应版本的预编译包（flink-chdfs-hadoop-{flink.version}-{version}.jar）拷贝到`${FLINK_HOME}/plugins/chdfs-hadoop`目录下；

3.在${FLINK_HOME}/conf/flink-conf.yaml中添加一些CHDFS相关配置以确保flink能够访问到CHDFS，这里的配置键与CHDFS完全兼容，可参考[hadoop-chdfs:[云CHDFS-操作指南-挂载CHDFS](https://cloud.tencent.com/document/product/1105/36368)](https://cloud.tencent.com/document/product/1105/36368)，必须配置信息如下：

```yaml
fs.AbstractFileSystem.ofs.impl: com.qcloud.chdfs.fs.CHDFSDelegateFSAdapter
fs.ofs.impl: com.qcloud.chdfs.fs.CHDFSHadoopFileSystemAdapter
fs.ofs.tmp.cache.dir: /data/emr/hdfs/tmp/chdfs/
fs.ofs.upload.flush.flag: true
fs.ofs.user.appid: 123456789
```

4.在作业的write或sink路径中填写格式为：```ofs://test/path```的路径信息即可，例如：

```java
        ...
        StreamingFileSink<String> fileSink  =  StreamingFileSink.forRowFormat(
                new Path("ofs://test/sink-test"),
                new SimpleStringEncoder<String>("UTF-8"))
                .build();
        ...
```

### 使用示例

以下给出Flink Job读写chdfs的示例代码：

```Java
// Read from CHDFS 
env.readTextFile("ofs://<dir-name>/<file-name>");

// Write to CHDFS
stream.writeAsText("ofs://<dir-name>/<file-name>");

// Use CHDFS as FsStatebackend
env.setStateBackend(new FsStateBackend("ofs://<dir-name>/<file-name>"));

// Use the streamingFileSink which supports the recoverable writer
StreamingFileSink<String> fileSink  =  StreamingFileSink.forRowFormat(
                new Path("ofs://<dir-name>/<file-name>"),new SimpleStringEncoder<String>("UTF-8"))
                .withRollingPolicy(build).build();

```


## 所有配置说明

| 属性键                             | 说明                | 默认值 | 必填项 |
|:-----------------------------------:|:--------------------|:-----:|:---:|
|fs.ofs.upload.flush.flag|chdfs调用flush的时候是否刷数据, 默认false。但是在flink 场景下，该配置需要设置为true。|无|是|
|fs.ofs.user.appid| 配置账户appid。 | 无  | 是|
|fs.ofs.impl                      | chdfs对FileSystem的实现类，固定为 com.qcloud.chdfs.fs.CHDFSHadoopFileSystemAdapter。 | 无|是|
|fs.AbstractFileSystem.ofs.impl   | chdfs对AbstractFileSystem的实现类，固定为com.qcloud.chdfs.fs.CHDFSDelegateFSAdapter。 | 无 |是|
|fs.ofs.tmp.cache.dir           | 本地cache的临时目录, 对于读写数据, 当内存cache不足时会写入本地硬盘, 这个路径若不存在会自动创建。 | 无 | 是|
|fs.ofs.data.transfer.https                   | 数据流是否使用tls, 默认为false | false | 否|
|fs.ofs.data.transfer.thread.count                | 读写数据线程大小, 默认32 |32 | 否 |
|fs.ofs.prev.read.block.count             | 预读数据块的数量| 4 | 否 |
|fs.ofs.reload.range.size       | range下载的range大小| 1048576（1MB）|否|

## 重要注意事项

Flink-chdfs v1.10-0.1.0 版本从flink checkpoint恢复时需要等待1min以上，该时间是CHDFS后端的session过期时间，server端可以配置调整。如果没有等待而从checkpoint进行恢复，可能会出现不能open文件的异常。</br>

Flink-chdfs v1.10-0.1.1 版本优化添加主动释放session，从flink checkpoint恢复时不需要再等待1min（session过期时间）。</br>

## FAQ

- Flink 既可以通过[chdfs-hadoop](https://github.com/tencentyun/chdfs-hadoop-plugin)读写chdfs中的对象文件，也可以通过flink-chdfs-hadoop来读写，这两种有什么区别？

chdfs-hadoop实现了Hadoop的兼容文件系统语义，Flink可以通过写Hadoop兼容文件系统的形式写入数据到chdfs中，但是这种方式不支持的flink的recoverable writer写入，当你使用streamingFileSink写入数据时，要求底层文件系统支持recoverable writer。 因此，flink-chdfs-hadoop基于chdfs-hadoop扩展实现了flink的recoverable writer，完整地支持了Flink文件系统的语义，因此推荐使用它来访问chdfs文件。
