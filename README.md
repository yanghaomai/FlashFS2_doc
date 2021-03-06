# FlashFS2

## IO500-ISC22 
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
| network                | 2x200G Infiniband              |

[OceanStor Pacific](https://e.huawei.com/cn/products/storage/distributed-storage/oceanstor-pacific-series/oceanstor-pacific-9950)：
| parameters              | configuration |
| ----------------------- | ------------- |
| capacity per OceanStor  | 188T          |
| bandwidth per OceanStor | 160GB/s       |
| count                   | 2             |


## Overview

FlashFS is a highly scalable distributed file system temporarily deployed for HPC applications. The schematic diagram of the architecture is as follows:

![overview1](./overview.png)

The overall architecture adopts the separation of client and server.
Before starting the upper-level application, each server node needs to run the background process Flash_server.

The client preloads Flash_client.so and uses [syscall_intercept](https://github.com/pmem/syscall_intercept) to intercept the file system access request of the user process. If the request is performed in the directory mounted by FlashFS, the FlashFS client will handle the execution.


## Distributed Data Management

FlashFS2 divides each file into 256KB file blocks. The data distribution of data and metadata is shown in the following figure:

![data-route1](./data-route.png)

File metadata is routed to a data server by hashing the absolute path of the file. The data server is responsible for accessing and persisting the metadata of the file.

The data block of the file is routed to a data server according to the hash value calculated by the absolute path of the file and the data block ID. The data server is responsible for the access and persistence of this file block.

It should be noted that in order to optimize the storage of small files (files whose file size is less than one file block), the data server where the first file block of the file is located and the data server where the file metadata is located are the same.

## RPC

FlashFS uses self-developed high-performance RPC based on two-side RDMA, which can be used on RoCE or Infiniband networks.
RDMA adopts RC connection mode, and the single-machine throughput can reach up to 11Mops in the Mellanox ConnectX-6 Infiniband network during the test.

## Stroage Engine

In the data server, each server contains up to 32 threads to process RPC requests. Each thread can create up to 1024 stackless coroutines to access the storage engine concurrently.

Each thread is bound to a Storage Engine interface, and Storage Engine interacts with [DPC filesystem](https://e.huawei.com/cn/products/storage/topic/2021/oceanstor-pacific-hpc-high-density) through libaio. The schematic diagram of the storage engine is as follows:

![storage-engine1](./storage-engine.png)

Components and what they do:
* Metadata Storage
    * MCache(Metadata Cache): Cache hot metadata in MetaDB.
    * Wal Log: Metadata persistence is guaranteed through Wal Log.
    * Batch Engine: Multiple Metadata can be aggregated and written to Wal Log to speed up persistence.
    * MetaDB(Metadata Database): Persistent metadata index. Generated by Wal Log asynchronous scan.
* Data Block Storage
    * Data Log: Stores data blocks of size 256KB in the form of Logs.
    * IndexDB(Index Database): Used to quickly index the data in the Data Log, which can be restored by scanning the Data Log.
    * ICache(Index Cache): Cache of IndexDB.
    * Block Cache: The cache of the Data Log, the cache block size is 256KB, which is generated when reading.

### Create (modify) file metadata
1. Create a cache in MCache while passing metadata to Batch Engine.
2. When the Batch Engine collects a certain amount of metadata or triggers a timeout mechanism, the metadata is written to the Wal Log.
3. The client is notified via RPC that the file creation (metadata modification) is complete.
4. Wal Log is applied to MetaDB asynchronously.
### Get file metadata
1. First get the metadata from MCache, if the cache hits, return the file metadata information through RPC.
2. If the cache misses, access MetaDB to obtain metadata, and return file metadata information through RPC.
### Write file block
1. File data is written to Data Log
2. Create a Data Log index in ICache and return client ACK via RPC.
3. ICache content is asynchronously flushed into IndexDB.
### Read file block
1. Get the Data Log index from ICache. If there is a cache miss, the Data Log index is fetched from IndexDB.
2. Read the file blocks from the Data Log according to the index information, and create the corresponding file block cache in the Block Cache.
3. Return data from the Block Cache to the client via RPC.

## Concurrency control

In order to reduce the overhead of locking, FlashFS performs metadata operations on the same file by the same thread. The schematic diagram is as follows:

![cc1](./cc.png)
Take file A as an example:
* Server_ID = a = hash1(file-name) % Server_Num
* Thread_ID = b = hash2(file-num) % Thread_Num

Metadata modification requests for file A need to be processed by thread *b* on Data Server *a*.

If thread *c* receives a modification request for file A, *c* will forward the request to the coroutine on thread *b* for processing.

Through the above request transfer operation, the serialization of the metadata modification of file A is guaranteed.

Concurrency control is also accomplished in a similar manner for the write operation of the data block of file A.

## write-fsync

To improve the write performance of FlashFS, FlashFS optimizes the write-fsync combined operation. As shown below:

![write-sync1](./write_sync.png)

After each write request is sent through RPC, it will return to the client that the write is successful. After the data server completes the data persistence, it returns an ACK to the client, which is asynchronously received by the client's RDMA network card hardware. When the client performs fsync on a file, it is determined whether the data persistence is completed by comparing the number of ACKs received by the client and the execution times of the write request. After all the data is persisted, fsync returns.

## Configure and run

To run the program, you need to configure `/etc/flashfs/cfg.json` first. The configuration parameters are described as follows:
```
{
    // FlashFS cluster master node IP address
    "master-ip": "n30311",
    // RPC port of the master node of the FlashFS cluster
    "master-port": 8989,
    // RPC port for FlashFS data nodes
    "server-port": 8990,
    // RDMA RoCE gid-idx
    "gid-idx": 1,
    // Maximum number of supported clients
    "max-cli-num": 10000,
    // List of data server nodes ip/hostname
    "server-list": [
        "n30321",
        "n30322",
        "n30324",
        "n30325"
    ],
    // List of RDMA device names
    "dev-list": [
        "mlx5_0",
        "mlx5_1"
    ],
    // Database data storage path(MetaDB, IndexDB)
    "db-path": [
        "/mnt/dpc1",
        "/mnt/dpc2",
    ],
    // Available core masks
    "core-mask": "000ffffffff",
    // Number of threads in the data server-side thread pool
    "io-num": 32,
    // FlashFS mount point
    "mount-point": "/mnt/flashfs",
    // Data Log storage path
    "data-path": [
        "/mnt/dpc1",
        "/mnt/dpc2",
    ],
    // Used to configure the interval for reporting cluster status information
    "master-report-sec": 1,
    // Hugepage file system mount point
    "hugetlbfs-dir": "/mnt/huge",
    // Timeout of Batch Engine
    "wal-sync-timeout-us": 20,
}
```

Program running steps:
1. Start the Master node by executing `sh run.sh master`
2. Start the server node by executing `sh run.sh server` on the data server
3. Execute the upper application



## Contact us

email: yhm12345@foxmail.com

QQ Group: 724071725