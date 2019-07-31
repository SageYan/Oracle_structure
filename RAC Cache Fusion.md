[TOC]

# RAC Cache Fusion 

## 定义：

![](C:\Users\yancong\Desktop\824142-20161204180422193-302021095.png)

```sql
RAC是一个数据库执行在多个实例上。通过DLM(Distributed Lock Management):分布式锁管理器 ，来解决并发问题。
RAC各个节点间的共享资源，为了保证每一个节点訪问数据的一致性。所以须要使用DLM来协调各个实例间的资源竞争访问。 
--这个DLM在RAC中就叫Cache Fusion。
```

## RAC并发实现机制：

### 资源分类

```powershell
在DLM中，根据资源数量，活动密集程度，把资源分成两类：Cache Fusion和Non-Cache Fusion。


Cache Fusion Resource指数据块这种资源，包括普通数据块，索引数据块，段头块(Segment Header),undo 数据块。 
Non-Cache Fusion Resource是所有的非数据库块资源, 包括数据文件，控制文件，数据字典，Library Cache，share Pool的Row Cache等。
--Row Cache 中存放的是数据字典，它的目的是在编译过程中减少对磁盘的访问。
```

### DLM管理cache fusion资源

```SQL
在Cache Fusion中，每一个数据块都被映射成一个Cache Fusion资源，Cache Fusion 资源实际就是一个数据结构，资源的名称就是数据块地址。

每个数据请求动作都是分步完成的：
首先把数据块地址X转换成Cache Fusion 资源名称，然后把这个Cache Fusion 资源请求提交给DLM， DLM 进行Global Lock的申请和释放活动，
只有进程获得了PCM Lock才能继续下一步。
--即：实例要获得数据块的使用权


Cache Fusion要解决的首要问题就是：数据块拷贝在集群节点间的状态分布图， 这是通过GRD 实现的。 
GRD（Global Resource Directory）
可以把GRD 看作一个内部数据库，这里记录的是每一个数据块在集群间的分布图，它位于每一个实例的SGA中，但是每个实例SGA中都是部分GRD，所有实例的GRD汇总在一起就是一个完整的GRD。
所以：
RAC 会根据每个资源的名称从集群中选择一个节点作为它的Master Node，而其他节点叫作Shadow Node。 
Master Node 的GRD中记录了该资源在所有节点上的使用信息，而Shadow Node的GRD中只需要记录资源在该节点上的使用情况，这些信息实际就是PCM Lock信息。
PCM Lock 有3个属性： Mode，Role 和 PI(Past Image)
```



## RAC架构

### SGA

```powershell
和传统的单实例相比， RAC Insance的SGA 最显著的变化就是多了一个GRD部分。 
Oracle 中的数据操作都是在内存的SGA区完成的，
和传统的单实例不同，RAC 是有多个实例，每个数据块可以在任何一个Instance 的SGA中都有拷贝。
所以RAC必须知道这些拷贝的分布版本，状态，而GRD就是这些信息的内存区。

GRD 虽然位于SGA中，但是不像Buffer Cache 或者 Log buffer等SGA 组件，有明确的参数来对应，
每个节点中都只有部分GRD内容，所有的节点合在一起才构成完整的GRD.
```

### 后台进程的变化

#### 1 LMSn 

Lock Manager Service

```
这个进程是Cache Fusion的主要进程，负责数据块在实例间的传递，对应的服务叫作GCS(Global Cache Service), 这个进程的名称来源与Lock Manager Service。 
从Oracle 9开始，Oracle 对这个服务重新命名成Global Cache Service, 但是进程名字确没有调整。 
这个进程的数量是通过参数GCS_SERVER_PROCESSES 来控制，缺省值是2个，取值范围为0-9. 
```

#### 2 LMD

lock monitor daemon process

```sql 
这个进程负责的是Global Enqueue Service（GES），
具体来说，这个进程负责在多个实例之间协调对数据块的访问顺序，保证数据的一致性访问。 
--它和LMSn进程的GCS服务还有GRD共同构成RAC最核心的功能Cache Fusion。
```

#### 3 LCK

lock process

```
这个进程负责Non-Cache Fusion 资源的同步访问，每个实例有一个LCK 进程
```

#### 4 LMON

lock monitor process

```sql
各个实例的LMON进程会定期通信，以检查集群中各个节点的健康状态，当某个节点出现故障时，负责集群重构，GRD恢复等操作，它提供的服务叫作：Cluster Group Services（CGS）。

LMON 主要借助两种心跳机制来完成健康检查：
1.节点间的网络心跳（Network Heartbeat）： 可以想象,节点间定时的发送ping包检测节点状态，如果能在规定时间内收到回应，就认为对方状态正常
2.通过控制文件的磁盘心跳（Controlfile Heartbeat）： 每个节点的CKPT进程每隔3秒更新一次控制文件一个数据块，
                                               这个数据块叫作Checkpoint Progress Record，控制文件是共享的，
                                               所以实例间可以相互检查对方是否及时更新来判断。
```

#### 5 DIAG

```
DIAG 进程监控实例的健康状态，并在实例出现运行错误时,收集诊断数据记录到alert.log 文件
```

#### 6 GSD

```
这个进程负责客户端工具，比如srvctl 接收用户命令，为用户提供管理接口。
```





## RAC文件

Oracle 文件包括二进制执行文件，参数文件（pfile和spfile），密码文件，控制文件，数据文件，联机日志，归档日志，备份文件

### 1 spfile

这个参数文件需要被所有节点访问，需要放在共享存储上

### 2 Redo Thread

```powershell
RAC 环境下有多个实例，每个实例都需要有自己的一套Redo log 文件来记录日志。这套Redo Log 就叫作一个Redo Thread，
其实单实例下也是Redo Thread，只是Thread 这个词很少被提及，每个实例一套Redo Thread的设计就是为了避免资源竞争造成性能瓶颈。

Redo Thread有两种：
一种是Private 的，创建语法： alter database add logfile .. Thread n; 
另一种是public，创建语法：alter database add logfile...; 

RAC 中每个实例都要设置thread 参数，该参数默认值为0. 如果设置了这个参数，则实例启动时，会使用等于该Thread的Private Redo Thread。
如果没有设置这个参数，则使用缺省值0，启动实例后选择使用Public Redo Thread，并且实例会用独占的方式使用该Redo Thread。

RAC 中每个实例都需要一个Redo Thread，每个Redo Log Thread至少需要两个Redo Log Group，每个Log Group 成员大小应该相等，每组最好有2个以上成员，这些成员应放在不同的磁盘上，以避免单点失败。

要注意的是，在RAC 环境下，Redo Log Group是在整个数据库级别进行编号的，比如实例1有1，2，3三个日志组，那么实例2的日志组就应该从4开始编号，不能在使用1，2，3这三个编号。

在RAC 环境下，所有实例的联机日志必须放在共享存储上，因为如果某个节点异常关闭，
剩下的节点要进行Crash Recovery， 执行Crash Recovery的这个节点必须能够访问到故障节点的连接日志，只有把联机日志放在共享存储上才能满足这个要求。
```

### 3 archive log

```
RAC中的每个实例都会产生自己的归档日志，归档日志只有在执行Media Recovery时才会用到，
所以归档日志不必放在共享存储上，每个实例可以在本地存放归档日志。
但是如果在单个实例上进行备份归档日志或者进行Media Recovery操作，
又要求在这个节点必须能够访问到所有实例的归档日志，在RAC环境下，配置归档日志可以有多种选择。

1）使用NFS
这种方式实际是归档到共享存储，比如2个节点，每个节点都有2个目录，Arch1，Arch2分别对应实例1和实例2的日志。每个实例只配置一个归档位置，归档到本地，然后通过NFS 把对方的目录挂到本地。

2）实例间归档（CIA: Cross Instance Archive）
这种方式是上一种方式的变种，也是比较常见的一种配置方法。两个节点都创建2个目录Arch1和Arch2 分别对应实例1和实例2产生的归档日志。 每个实例都配置两个归档位置。位置1对应本地归档目录，位置2对应另一个实例。

3）使用ASM
这种方式也是归档到共享存储，只是通过Oracle 提供的ASM,把上面的复杂性隐藏了，但原理都一样。
```



### 4 Undo Tablespace

```
和Redo Log 一样，在RAC 环境下，每个实例都需要有一个单独的回滚表空间，这个是通过参数SID.undo_tablespace 来配置的。
```



## SCN(System Change Number)

```SQL
在RAC中，有GCS 负责全局维护SCN的产生。
缺省用的是Lamport SCN生成算法，该算法大致原理是： 在所有节点间的通信内容中都携带SCN， 
每个节点把接收到的SCN和本机的SCN对比，如果本机的SCN小，则调整本机的SCN和接收的一致，如果节点间通信不多，还会主动地定期相互通报。 
故即使节点处于Idle 状态，还是会有一些Redo log 产生。 
还有一个广播算法（Broadcast），这个算法是在每个Commit操作之后，节点要想其他节点广播SCN，虽然这种方式会对系统造成一定的负载，但是确保每个节点在Commit之后都能立即查看到SCN.

这两种算法各有优缺点，Lamport虽然负载小，但是节点间会有延迟，广播虽然有负载，但是没有延迟。 
Oracle 10g RAC 缺省选用的是BroadCast 算法，可以从alert.log 日志中看到相关信息：

Picked broadcast on commit scheme to generate SCNS
```



## Cache Fusion， GCS， GES 关系

```SQL
Cache Fusion（内存融合）是通过高速的Private Interconnect，在实例间进行数据块传递，它是RAC 最核心的工作机制，
它把所有实例的SGA虚拟成一个大的SGA区。每当不同的实例请求相同的数据块时，这个数据块就通过Private Interconnect 在实例间进行传递。

整个Cache Fusion 有两个服务组成：GCS 和GES。 
--GCS 负责数据库在实例间的传递，GES 负责锁管理
```

