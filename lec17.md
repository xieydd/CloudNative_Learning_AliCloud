* [Lec17：深入理解 ETCD之etcd 性能优化实践](#lec17深入理解-etcd之etcd-性能优化实践)
   * [1. 理解 ETCD 性能](#1-理解-etcd-性能)
      * [1. 哪些组件会带来性能损耗](#1-哪些组件会带来性能损耗)
   * [2. 服务端 ETCD 优化](#2-服务端-etcd-优化)
      * [1. 硬件](#1-硬件)
      * [2. 服务端](#2-服务端)
      * [3. 客户端](#3-客户端)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

### Lec17：深入理解 ETCD之etcd 性能优化实践

#### 1. 理解 ETCD 性能

##### 1. 哪些组件会带来性能损耗

1. RAFT 间通信受带宽限制 
2. WAL 日志的写受磁盘IO 写入延时的影响
3. Storage 磁盘 IO fdatasync 延迟
4. 索引层锁的 block 
5. blotdb Tx 的锁
6. blotdb 自身性能
7. 所在宿主机内核参数
8. grpc api层延时

#### 2. 服务端 ETCD 优化

前提： [理解 etcd api 的设计](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/api.md)

##### 1. 硬件

升级CPU、选取性能好的SSD、网络带宽升级、独占部署减少其他程序的干扰

[etcd hardward suggestion](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md)

##### 2. 服务端

1. 内存索引层 [pr 9511](https://github.com/etcd-io/etcd/pull/9511) 优化内部锁减少等待时间
2. lease 规模化使用 [pr 9418](https://github.com/etcd-io/etcd/pull/9418) 优化 lease revoke 和过期失效算法，解决 lease 规模性问题
3. 后端 blotdb 使用优化 [commit](https://github.com/etcd-io/etcd/commit/3faed211e535729a9dc36198a8aab8799099d0f3) 可根据不同的硬件和负载配置； 完全并发读 优化调用blotdb tx 锁使用，提升性能 [pr 10523](https://github.com/etcd-io/etcd/pull/10523) 
4. ali 做的一个优化 [基于 segregated hashmap 的 etcd  内部存储 freelist 分配回收算法](https://www.cncf.io/blog/2019/05/09/performance-optimization-of-etcd-in-web-scale-data-scenario/)
   1. 页面大小 4kb 
   2. freelist 是暂存未使用页面提升性能,因为获取 freelist 的算法为遍历会导致寻址复杂度为 O(n) ，在数据大且碎片化的时候，性能会急剧下降；
   3. 通过将连续的页面大小作为 hashmap 的 key value 是页面起始id的集合，查询页面 size 为n 时可以 以 O(1) 时间复杂度获取，回收的时候可以通过 merge 的 方法合并获得大的连续的页面，从 O(nlogn) 到 O(1)

##### 3. 客户端

1. Put 时避免大的 value ，例如 k8s crd 的使用
2. 避免创建频繁变化的 key/value 例如 node 上报的信息
3. 避免创建大量的 Lease, 尽量选择复用例如 k8s event 数据管理
