# FlashFS2 概述

## IO500-ISC22 English Version
Schematic diagram of the test environment:
![test-env1](./test-env.png)

Client node, data node test environment parameters:
| parameters             | configuration                  |
| ---------------------- | ------------------------------ |
| memory capacity        | 503G                           |
| CPU model              | Intel(R) Xeon(R) Platinum 8358 |
| CPU frequency          | 2.60GHz                        |
| sockets                | 2                              |
| cores per socket       | 32                             |
| threads per core       | 2                              |
| OS                     | Centos7                        |
| kernel version         | Linux3.10                      |
| number of client nodes | 10                             |
| number of data nodes   | 30                             |
| client network         | 4x200G Infiniband              |
| data server network    | 2x200G Infiniband              |

OceanStor Pacific：
| parameters              | configuration |
| ----------------------- | ------------- |
| capacity per OceanStor  | 188T          |
| bandwidth per OceanStor | 160GB/s       |
| count                   | 2             |

## IO500-ISC22 中文版
测试环境示意图：
![test-env2](./test-env.png)

客户端节点、数据服务器测试环境参数：
| 属性           | 配置                           |
| -------------- | ------------------------------ |
| 内存容量       | 503G                           |
| CPU型号        | Intel(R) Xeon(R) Platinum 8358 |
| CPU频率        | 2.60GHz                        |
| CPU槽数        | 2                              |
| 每槽核数       | 32                             |
| 每核线程数     | 2                              |
| 操作系统       | Centos7                        |
| 内核版本       | Linux3.10                      |
| 客户端数量     | 10                             |
| 数据服务器数量 | 30                             |
| 客户端网络     | 4x200G Infiniband              |
| 数据服务器网络 | 2x200G Infiniband              |

OceanStor Pacific参数：
| 属性     | 参数    |
| -------- | ------- |
| 容量每台 | 188T    |
| 带宽每台 | 160GB/s |
| 个数     | 2       |


## 整体架构

FlashFS是一个针对HPC应用程序临时部署的，高度可扩展的分布式文件系统。架构示意图如下：

![overview](./overview.png)

整体架构采用客户端和服务端分离。
在启动上层应用程序之前，每个服务端节点都需要运行文件系统的后台进程Flash_server。

客户端通过预加载（Preload）Flash_client.so，调用
[syscall_intercept](https://github.com/pmem/syscall_intercept)拦截用户进程的文件系统访问请求。如果该请求在FlashFS所挂载的目录下执行操作，则将有FlashFS客户端处理执行。

## 数据分布

FlashFS2将每个文件切分成一个个大小为256KB的文件块。数据和元数据的数据分布如下图所示：

![data-route](./data-route.png)

文件元数据通过计算文件绝对路径的哈希值，路由到一台对象的数据服务器上。由该数据服务器负责该文件元数据的访问与持久化。

文件的数据块则是根据文件绝对路径和数据块ID共同计算出的哈希值，路由到一台服务器上。该服务器负责该文件快的访问与持久化。

需要注意的是为了优化小文件存储（文件大小小于一个文件块的文件），文件的第一个文件块所在的数据服务器和文件元数据所在的数据服务器是同一台。

## RPC

FlashFS使用自研的基于双边RDMA实现的高性能RPC，可搭建在RoCE或者Infiniband网络上。
RDMA采用RC连接模式，测试时在Mellanox ConnectX-6 Infiniband网络中单机吞吐最高可达11Mops。

## 存储引擎

在数据服务器，每个服务器都包含最多32个线程用来处理访问请求。每个线程最多可以创建1024个无栈协程来并发处理访问请求。
每个线程和一个存储引擎（Storage Engine）接口绑定，Storage Engine通过libaio同[DPC Filesystem](https://e.huawei.com/cn/products/storage/topic/2021/oceanstor-pacific-hpc-high-density)进行交互。存储引擎的示意图如下：

![storage-engine](./storage-engine.png)

组件与作用：
* Metadata Storage
    * MCache（元数据缓存）：缓存MetaDB中的热元数据。
    * Wal Log（写前日志）：通过Wal Log保证元数据持久化。
    * Batch Engine（批处理引擎）：可以聚合多个Metadata写入Wal Log，加速持久化。
    * MetaDB（元数据数据库）：元数据持久化引擎。由Wal Log异步刷入生成。
* Data Block Storage
    * Data Log（数据日志）：以Log的形式存储大小为256KB的数据块。
    * IndexDB（索引数据库）：用来索引Data Log中数据，可以通过Data Log还原。
    * ICache（索引缓存）：IndexDB的缓存。
    * Block Cache（块缓存）：Data Log的缓存，缓存块大小为256KB，在读取时生成。

### 创建（修改）文件元数据
1. 在MCache创建缓存，同时将元数据传递给Batch Engine。
2. 当Batch Engine收集元数据达到批处理数量或者触发超时机制时，将元数据写入Wal Log。
3. 调用RPC通知客户端文件创建（元数据修改）完成。
4. Wal Log异步应用到MetaDB。
### 读取文件元数据
1. 访问MCache，如果缓存命中，返回文件元数据信息。
2. 如果缓存未命中，访问MetaDB，并返回文件元数据信息。
### 写入文件数据
1. 文件数据写入Data Log
2. 在ICache创建Data Log索引，返回客户端ACK。
3. ICache内容异步刷入IndexDB。
### 读取文件数据
1. 从ICache获取文件索引数据。如果缓存未命中，则从IndexDB获取Data Log索引。
2. 根据索引信息读取Data Log，在Block Cache中创建缓存。
3. 从Block Cache中返回数据给客户端。

## 并发控制
为了减少锁的开销，FlashFS对于同一个文件的元数据操作都由同一个线程执行。示意图如下：
![cc](./cc.png)
对于文件A来说：
* Server_ID = a = hash1(file-name) % Server_Num
* Thread_ID = b = hash2(file-num) % Thread_Num

如果线程c收到了对文件A的修改请求，C将会把请求转交给线程b上的协程来处理。

通过上述的请求转移操作，保证了文件A的元数据修改序列化。

对于文件A的数据块的写入操作也使用类似的方式完成并发控制。

## write-fsync

为了提升FlashFS的写入性能，FlashFS优化了write-fsync组合操作。如下图所示：

![write-sync](./write_sync.png)

每次write请求发送完成之后便返回客户端写入成功。数据服务端完成数据持久化之后返回Write OK，由客户端RDMA网卡硬件异步接收。在客户端对某文件执fsync时，通过比较客户端所有到的write OK个数和write请求的执行次数是否相同，来判断是否完成数据的持久化。等到数据全部持久化完毕之后，fsync执行返回。

## 运行

程序运行需要配置`/etc/flashfs/cfg.json`。配置参数说明如下：
```
{
    // FlashFS集群主节点IP地址
    "master-ip": "n30311",
    // FlashFS集群主节点的RPC端口
    "master-port": 8989,
    // FlashFS数据节点的RPC端口
    "server-port": 8990,
    // RDMA RoCE gid-idx
    "gid-idx": 1,
    // 最大支持的客户端数量
    "max-cli-num": 10000,
    // 数据服务端节点的列表ip/hostname
    "server-list": [
        "n30321",
        "n30322",
        "n30324",
        "n30325"
    ],
    // RDMA设备名称列表
    "dev-list": [
        "mlx5_0",
        "mlx5_1"
    ],
    // 数据库数据路径
    "db-path": [
        "/mnt/dpc1",
        "/mnt/dpc2",
    ],
    // 可用的核心掩码
    "core-mask": "000ffffffff",
    // 数据服务器端线程池中线程数量
    "io-num": 32,
    // FlashFS挂载路径
    "mount-point": "/mnt/flashfs",
    // Data Log存储路径
    "data-path": [
        "/mnt/dpc1",
        "/mnt/dpc2",
    ],
    // 用于信息收集统计的时间间隔
    "master-report-sec": 1,
    // 大页文件系统挂载目录
    "hugetlbfs-dir": "/mnt/huge",
    // Batch Engine的超时时间
    "wal-sync-timeout-us": 20,
}
```

程序运行步骤：
1. 启动Master节点，执行`sh run.sh master`
2. 启动服务端节点，在数据服务器执行`sh run.sh server`
3. 执行应用程序


## 交流

邮箱：yhm12345@foxmail.com

QQ群组：724071725