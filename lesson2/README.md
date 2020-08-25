# lesson 2

## 测试环境

### 集群拓扑及配置
本次测试所使用的集群拓扑如下所示：

| ip              | 机器配置          | 集群角色                                         |
| --------------- | ----------------- | ------------------------------------------------ |
| `10.227.71.147` | 8 Cores, 16GB Mem | `grafana`, `alertmanager`, `prometheus`, 1个`pd` |
| `10.227.67.58`  | 8 Cores, 16GB Mem | 1个`tikv`, 1个`tidb`                             |
| `10.227.67.62`  | 8 Cores, 16GB Mem | 1个`tikv`, 1个`tidb`                             |

### 测试环境配置

除了配置好特定角色无密码`sudo`权限，还需要将承载数据和log等目录的磁盘进行格式化并重挂载。磁盘需要被格式化为`ext4`格式，并且添加`nodelalloc`和`noatime`挂载参数。

`nodelalloc`主要是控制当需要写入的数据抵达`kernel page cache`时分配磁盘block的行为。当`nodelalloc`参数被设置时，磁盘block将会在抵达`kernel page cache`时被立刻被分配。因为在一般的操作系统使用场景中，通常是少量数据被以非常高的频率写入，延迟分配将会有效减少分配操作的次数。但在`tinykv`这种单次写入量较大，且需要精确控制`page cache`行为的场景时，手动控制这些细节将会为我们提供更大的优化空间。

### 安装

`tiup`工具为我们提供了一个可以在生产环境下使用的快速部署`tidb`集群的能力，首先我们需要根据集群拓扑定义一个[YAML配置文件](./tidb-test.yaml)，然后利用`tiup`的`cluster`组件完成安装：

```bash
# tiup内建了一个ssh工具，有些情况下这个工具会出现读不到或错误读取本地配置的情况
# 此时可以使用 --native-ssh 选项来使用系统本地的ssh命令来连接远程机器
tiup cluster deploy tidb-test v4.0.0 ${path_to_yaml} -u ${remote_user}
```

安装完成后，我们可以使用`tiup cluster start ${cluster_name}`和`tiup cluster stop ${cluster_name}`来启动和关闭集群，还可以使用`tiup cluster display`来查看集群状态。比如在本次实验中，我们可以看到该拓扑下的集群信息：![](./images/topo.png)


## sysbench测试

### 测试步骤

测试使用的config文件在[这里](./sysbench.conf)，为了加快导入速度，我们通过一下命令将tidb的事务模型设置为乐观模型，导入完成之后再将其设置回悲观模型。

```mysql
set global tidb_disable_txn_auto_retry = off;
set global tidb_txn_mode="optimistic";
```

数据导入命令如下：
```bash
sysbench --config-file=./sysbench.conf oltp_point_select --tables=32 --table-size=100000 prepare
```

数据导入过程中（持续时间从5:38~6:00），TiDB的QPS状况如下图：
![](./images/sysbench-import/sysbench-tidb-qps.png)
TiDB的查询延迟如下图：
![](./images/sysbench-import/sysbench-tidb-duration.png)
TiKV的cpu使用情况如下图：
![](./images/sysbench-import/sysbench-tikv-cpu.png)
TiKV的qps如下图：
![](./images/sysbench-import/sysbench-tikv-qps.png)
TiKV的grpc延迟如下图：
![](./images/sysbench-import/sysbench-tikv-grpc-duration.png)
TiKV的grpc qps如下图：
![](./images/sysbench-import/sysbench-tikv-grpc-qps.png)

测试命令如下：
```bash
sysbench --config-file=./sysbench.conf oltp_point_select --tables=32 --table-size=100000 run
```

测试过程中，TiDB的QPS状况如下图：
![](./images/sysbench-test/tidb-qps.png)
TiDB的查询延迟如下图：
![](./images/sysbench-test/tidb-duration.png)
TiKV的cpu使用情况如下图：
![](./images/sysbench-test/tikv-cpu.png)
TiKV的qps如下图：
![](./images/sysbench-test/tikv-qps.png)
TiKV的grpc延迟如下图：
![](./images/sysbench-test/tikv-grpc-duration.png)
最终测试结果如下图：
![](./images/sysbench-test/result.png)

## ycsb测试

数据导入命令如下：
```bash
./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=500000 -p mysql.host=10.227.67.58 -p mysql.port=4000 --threads=8
```

数据导入过程中，TiDB的QPS状况如下图：
![](./images/ycsb-import/tidb-qps.png)
TiDB的查询延迟如下图：
![](./images/ycsb-import/tidb-duration.png)
TiKV的cpu使用情况如下图：
![](./images/ycsb-import/tikv-cpu.png)
TiKV的qps如下图：
![](./images/ycsb-import/tikv-qps.png)
TiKV的grpc延迟如下图：
![](./images/ycsb-import/tikv-rpc-duration.png)
TiKV的grpc qps如下图：
![](./images/ycsb-import/tikv-grpc-qps.png)

测试命令如下：
```bash
./bin/go-ycsb run mysql -P workloads/workloada -p recordcount=500000 -p mysql.host=10.227.67.58 -p mysql.port=4000 --threads=8
```

## tpc-c测试

数据导入命令如下：
```bash
go-tpc tpcc -D tpcc -H 10.227.67.58 --warehouses 10 prepare
```

数据导入过程中，TiDB的QPS状况如下图：
![](./images/tpcc-import/tidb-qps.png)
TiDB的查询延迟如下图：
![](./images/tpcc-import/tidb-duration.png)
TiKV的cpu使用情况如下图：
![](./images/tpcc-import/tikv-cpu.png)
TiKV的qps如下图：
![](./images/tpcc-import/tikv-qps.png)
TiKV的grpc延迟如下图：
![](./images/tpcc-import/tikv-grpc-duration.png)
TiKV的grpc qps如下图：
![](./images/tpcc-import/tikv-grpc-qps.png)

测试命令如下：
```bash
go-tpc tpcc -D tpcc -H 10.227.67.58 --warehouses 10 run
```

数据导入过程中，TiDB的QPS状况如下图：
![](./images/tpcc-test/tidb-qps.png)
TiDB的查询延迟如下图：
![](./images/tpcc-test/tidb-duration.png)
TiKV的cpu使用情况如下图：
![](./images/tpcc-test/tikv-cpu.png)
TiKV的qps如下图：
![](./images/tpcc-test/tikv-qps.png)
TiKV的grpc延迟如下图：
![](./images/tpcc-test/tikv-grpc-duration.png)
TiKV的grpc qps如下图：
![](./images/tpcc-test/tikv-grpc-qps.png)

## 分析

在sysbench和ycsb的数据导入过程中有一个非常有意思的现象——导入过程中sysbench的每语句执行时延明显比高的多（5倍以上），QPS比ycsb更低，其数据导入时间却比ycsb少了50%。要知道，实际导入数据sysbench一共需要导入320万行，而ycsb只需要导入10万行数据。经过分析两个基础测试导入程序的代码发现，sysbench在一条insert语句中会导入更多的数据（类似batch commit的思路），而ycsb则是通过频繁的insert单条记录的方式插入数据。很显然，在这种无冲突的大批量的插入过程中，使用Batch Commit的思路会显著提高程序性能。

另一个有意思的现象是，无论哪一个数据导入程序，其初始时CPU和QPS都存在一个非常明显的“尖峰”，该“尖峰”是由于多线程瞬间开始提交数据导致的，有点类似于“秒杀”。之后，由于每一个并发线程负责的任务执行进度不同，不再有瞬时大量的写入，因此后续时间的CPU和QPS趋于平缓。这种场景在设计分布式系统时是一个必需谨慎考虑的点（短时瞬时高并发量）

最后是在tpcc测试过程中，我们从TiKV的grpc ops图中可以发现，该测试中存在大量的悲观锁操作。这严重拖慢了测试程序的执行。原因在学习视频中也有提到，我们在测试时选择warehouse数量为10，并发量为默认值16——此时warehouse的数量显著小于并发量，是的程序在模拟提交订单时会产生大量的锁冲突，从而拖慢了数据库整体性能，预计该问题在warehouse显著大于并发时会缓解。

在集群拓扑中，虽然我们部署了两个tidb节点，但是由于在测试时总是访问同一个tidb节点，因此没有最大程度利用其分布式的特点。如果在测试时，将测试程序拆分成访问两个不同tidb的节点，其性能应当有一定程度的提升。但由于所有的查询最终都会落到两个tikv上且测试语句本身并不复杂（Parse和Schedule压力并不大）这种优化所得到的性能提升比较有限。

从几个测试的TiKV CPU使用情况和QPS的图中，我们不难发现，其中一个节点的处理压力明显比另一个更大，这一特点在数据导入过程中表现的尤为明显。猜测是因为在数据导入时，存在数据由一个Tikv流向另一个TiKV的情况。至于查询中出现的这种情况，则可能是因为副本间维持数据一致性的关系，leader的开销会更大。