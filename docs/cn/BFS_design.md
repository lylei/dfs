# BFS整体设计文档

## 背景
百度的核心数据库Tera将数据持久化在分布式文件系统上，分布式文件系统的性能、可用性和扩展性对整个上层搜索业务的稳定性与效果有着至关重要的影响。现有的分布式文件系统无法很好地满足这几方面的要求，所以我们从Tera需求出发，开发了百度自己的分布式文件系统。

## 系统架构
系统主要由Nameserver，Chunkserver和ClientSDk三部分构成。其中Nameserver是BFS的“大脑”，负责整个文件系统的目录树及每个文件的元信息的持久化存储和更新；Chunkserver是提供文件读写服务的实体；ClientSDk包括提供给管理员使用的管理命令工具bfs_client和给用户使用的SDK。除此之外，BFS还提用一个可视化Web界面用于实时反映集群的状态。

## Nameserver
Nameserver是BFS的“大脑”，负责存储及更新目录树及每个文件的元信息，所有需要读取或更新元信息的操作都需要通过Nameserver，而Nameserver的不可用将导致整个文件系统瘫痪。因此，在BFS的设计中，我们重点关注Nameserver的性能和可用性。

### Nameserver中的模块
Nameserver中主要有三个模块，分别负责管理目录树，文件块以及Chunkserver的管理。

#### 目录树
BFS的设计目标是存储海量文件，因此目录树结构可能无法全部存在内存中。而如前面所说，作为全局元信息的唯一管理者，目录树结构需要支持大吞吐的访问。因此，我们选择leveldb来存储BFS的目录树。Leveldb的高性能随机读写很适合BFS目录树的应用场景，而且可以使目录树直接持久化在磁盘，解决了内存限制目录树规模的问题，同时免去了启动时目录树构建的过程。因此BFS不需要联邦机制就可以承载海量文件的存储需求。

##### 目录树存储
在BFS的目录树中，每一个目录或文件的存储格式为 `父目录 entry_id + 目录/文件名 -> 目录/文件元信息`。其中`entry_id`是用来唯一标识某个目录或文件的id，根目录的`entry_id`为1，`父目录entry_id`为1，目录的元信息包括目录的`entry_id`、读写权限及创建时间等；文件的元信息还会包括文件的大小，版本，副本数量，及文件对应的所有block的`block_id`等。例如形如`/home/dir/file`的一个目录结构在BFS中的实际存储如下：
> 0/  
> 1home -> meta(`entry_id`=2, ctime=...)  
> 2dir -> meta(`entry_id`=3, ctime=...)  
> 3file -> meta(`entry_id`=4, block=1,2,3, size=.

##### 文件查找
查找文件时按层级查找，先查找到父目录的`entry_id`，再以父目录`entry_id` + 子目录名为key查找子目录的`entry_id`，以此类推。例如，查找`/home/dir/file`的过程如下：
> 查找根目录的`entry_id`，再以根目录的`entry_id`+home，也就是`1home`，读取/home目录的meta信息，获得/home的`entry_id`，2  
> 同上查找`2dir`读取`entry_id`，3  
> 最后取得`3file`  

查找文件的复杂度等于目录深度。

##### list目录
因为目录树在leveldb中存储的key为`父目录entry_id+名字`，所以同一目录下的目录和文件都有同样的前缀。只需查找到父目录的`entry_id`再扫描出以此`entry_id`开头的数据，就可以获得该目录下的所有目录和文件。

##### rename
Rename操作只需要将被改名文件或目录文件夹的key中的父目录id/文件名更改，而不需要对其子目录或者父目录做任何操作。

#### BlockMapping
在BFS中，文件被切分成block，每一个block都有一个唯一的id作为标识，称为block_id。每个block都会有多个副本，存储在多个Chunkserver上（默认为三个），以提高数据的可靠性和可用性，每一个副本称为一个replica。BlockMapping维护了block和其replica所在Chunkserver的映射关系。同时，BlockMapping为每个block维护一个状态：
	* NotInRecover：当前Block处于正常状态。
	* Writing：当前Block正在被写，正在被写的Block不会被副本恢复。
	* HiRecover：当前Block只剩下一副本，需要高优副本恢复。
	* LoRecover：当前Block不到默认副本数，需要副本恢复。
	* Incomplete：当前Block之前正在被写，但是其中有副本故障，等待被关闭。
副本恢复是分布式文件系统保证文件可靠的核心。BFS中只对已经关闭的文件进行副本恢复，以简化恢复的逻辑和副本一致性的复杂度。当副本不足默认副本数的时候，BlockMapping会将该Block放到高优或者普通恢复队列，等待Chunkserver挑选相应Block恢复。

#### ChunkserverManager
ChunkserverManager主要维护集群中所有Chunkserver的状态。包括Chunkserver存货信息的维护，当前的负载等。在新的文件块被创建或者被副本恢复的时候，ChunkserverManger负责挑选出Chunkserver。当前ChunkserverManager挑选Chunkserver的策略是以较大概率挑选较低负载的Chunkserver，以避免简单挑选负载最低的Chunkserver带来的压力峰值。


### Nameserver中的主要流程
#### 创建文件
SDK端向Nameserver发起创建文件的操作，Nameserver首先检查文件路径的合法性，然后由ChunkserverManager根据负载均衡逻辑选取Chunkserver，记录在BlockMapping里，然后返回给SDK，SDK再向三个Chunkserver发起写请求。

#### 删除文件
SDK端向Nameserver发起创建文件的操作，Nameserver首先检查文件路径的合法性，然后删除namespace中对应条目，向用户返回成功。真正Chunkserver上物理文件的删除为异步后台操作。

#### BlockReport
BlockReport是一个重要的流程，Chunkserver会定期将自己的Block信息向Nameserver汇报，包括Block的大小，状态等。Nameserver会：
	1. 把BlockReport携带的Block信息和namespace中的信息作对比，如果不一致或者该Block已被删除，则会要求Chunkserver删除该文件。
	2. 如果检查过程中发现Chunkserver丢失了Block，则会对该Block发起副本恢复操作。
	3. 如果有需要该Chunkserver做副本恢复的Block，会把副本恢复的信息告知该Chunkserver。
	4. 如果当前Block写异常，需要关闭，也会通过BlockReport的返回rpc告知Chunkserver
#### 副本恢复
BFS只会对已经关闭的文件进行副本恢复，正在写的文件异常时需要先将文件关闭才能进行副本恢复。目前副本恢复有两个触发入口，Chunkserver宕机和BlockReport。


