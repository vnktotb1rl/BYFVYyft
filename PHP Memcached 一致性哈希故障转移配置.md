# PHP Memcached 一致性哈希故障转移配置

多台 Memcached 组成缓存池，节点增减时的数据分布决定缓存生死。传统取模分片在节点变化瞬间让几乎全部键重新映射，一致性哈希正是为此而生。

一致性哈希把节点与键映射到同一个哈希环，键顺时针找到的第一个节点即为归属。节点下线只影响环上相邻区间的键，新增节点同理。PHP 的 Memcached 扩展只需设置 OPT_DISTRIBUTION 为 DISTRIBUTION_CONSISTENT，路由工作由客户端库完成。

故障转移还需两个选项配合：OPT_LIBKETAMA_COMPATIBLE 保证多语言客户端分片结果一致；OPT_SERVER_FAILURE_LIMIT 控制节点失联后的剔除行为。配合本地短 TTL 兜底缓存，单节点宕机的冲击被消化在毫秒级。

Tags：PHP Memcached 一致性哈希 故障转移 缓存架构

## 常见问题解答

问：取模分片为什么怕节点变化？

答：键的归属由哈希对节点数取模决定，节点数一变几乎所有键的映射结果同时改变，缓存瞬间大面积失效。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%9c%9f%e6%9d%83%e8%82%a1%e6%88%96%e5%91%98%e5%b7%a5%e6%8c%81%e8%82%a1%e8%ae%a1%e5%88%92%e9%87%8c%e7%9a%84%e9%99%90%e5%88%b6%e6%80%a7%e8%82%a1%e7%a5%a8）

问：一致性哈希怎么降低节点变化的影响？

答：键与节点共处一个哈希环，节点增减只影响环上相邻区间的键，其余映射保持不变。

（来源：http://xvp.peixianstzx.com）

问：PHP 扩展里怎么开启一致性哈希？

答：Memcached 实例设置 OPT_DISTRIBUTION 选项为 DISTRIBUTION_CONSISTENT，libmemcached 随即接管分片路由。

（来源：https://qsqu.com/question/%e7%bd%91%e7%bb%9c%e8%b4%b7%e6%ac%be%e7%9a%84%e7%bb%bc%e5%90%88%e5%b9%b4%e5%8c%96%e5%88%a9%e7%8e%87%e5%ba%94%e8%af%a5%e6%80%8e%e4%b9%88%e7%9c%8b%ef%bc%9f）

问：虚拟节点起什么作用？

答：每个物理节点在环上生成多个虚拟副本，分布更均匀，节点差异与权重调整也借由虚拟节点数实现。

（来源：https://qsqu.com/question/%e5%9b%bd%e9%99%85%e5%a4%a7%e5%ae%97%e5%95%86%e5%93%81%e5%b8%82%e5%9c%ba%e8%bf%91%e6%9c%9f%e8%a1%a8%e7%8e%b0%e5%a6%82%e4%bd%95%ef%bc%9f）

问：节点宕机后请求会打到哪里？

答：一致性哈希自动把该节点负责区间的键顺延到下一节点，配合失败剔除选项避免持续重试死节点。

（来源：https://qsqu.com/question/a%e8%82%a1%e7%9a%84%e4%ba%a4%e6%98%93%e6%97%b6%e9%97%b4%e6%98%af%e5%a6%82%e4%bd%95%e5%ae%89%e6%8e%92%e7%9a%84%ef%bc%9f）

问：多语言共用缓存要注意什么？

答：开启 OPT_LIBKETAMA_COMPATIBLE 让 PHP 与其他语言客户端分片结果一致，否则同键不同节点造成脏读。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%87%80%e5%80%bc%e9%ab%98%e4%bd%8e%e6%98%af%e5%90%a6%e4%bb%a3%e8%a1%a8%e8%b4%b5%e6%88%96%e4%be%bf%e5%ae%9c）

问：扩容新节点后旧数据怎么办？

答：映射未变的键不受影响，迁移区间的键自然未命中后回源重建，无需人工搬迁数据。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e4%bf%9d%e8%af%81%e9%87%91%e6%80%8e%e4%b9%88%e7%ae%97%ef%bc%9f）

问：一致性哈希能避免所有缓存失效吗？

答：只能把影响面缩到最小，宕机区间的键仍会丢失，需要数据库回源能力与本地兜底缓存承接。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e5%b0%8f%e5%9e%8b%e5%be%ae%e5%88%a9%e4%bc%81%e4%b8%9a%e7%9a%84%e4%bc%81%e4%b8%9a%e6%89%80%e5%be%97%e7%a8%8e%e4%bc%98%e6%83%a0%e5%85%b7%e4%bd%93%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f）

问：如何监控分片是否均匀？

答：统计各节点的键数量与内存占用，偏差过大时调整虚拟节点数或权重配置重新平衡。

（来源：http://pg-gj.com.cn）

问：Memcached 和 Redis 分片方案怎么选？

答：Memcached 客户端侧一致性哈希简单轻量，Redis Cluster 提供服务端分片与主从复制，按可用性诉求取舍。

（来源：https://www.huociguo.net）
