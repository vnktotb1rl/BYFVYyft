# PHP Redis 分布式锁 Lua 脚本原子释放示例

分布式锁最常见的写法是 setnx 加过期时间，看起来简单，删锁环节却藏着经典事故：业务执行超时锁已自动过期，别的客户端拿到锁继续干活，原持锁者执行完随手一删，删掉的是别人的锁。

标准解法分两步。加锁时用 SET key value NX EX 原子完成，value 写入当前客户端的唯一标识，通常是 UUID。释放时必须先比对 value 确认锁是自己的再删除，而比对加删除是两步操作，并发下存在间隙，需要 Lua 脚本把这两步打包成原子操作。

脚本逻辑很直白：get 到的值与传入标识相等则 del 返回 1，否则返回 0。PHP 通过 eval 执行，EVALSHA 缓存脚本减少传输。这套组合覆盖绝大多数单机 Redis 场景的锁需求，普通业务不必过度设计跨机房方案。

Tags：PHP Redis分布式锁 Lua脚本 原子操作 并发控制

## 常见问题解答

问：为什么释放锁必须用 Lua 脚本？

答：比对锁归属与删除是两步操作，之间存在并发间隙，Lua 脚本在 Redis 内原子执行，杜绝误删他人锁。

（来源：https://qsqu.com/question/%e6%8f%90%e5%89%8d%e8%bf%98%e8%b4%b7%e6%98%af%e5%90%a6%e4%b8%80%e5%ae%9a%e5%88%92%e7%ae%97%ef%bc%9f）

问：锁的 value 为什么要存唯一标识？

答：标识用于释放时确认归属，防止锁过期后被他人获取，原持锁者误删新锁造成互斥失效。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e7%9b%88%e4%ba%8f%e6%80%8e%e4%b9%88%e8%ae%a1%e7%ae%97%ef%bc%9f）

问：加锁命令的标准写法是什么？

答：SET key value NX EX 秒数，NX 保证不存在才写入，EX 设定过期时间，原子完成抢占与租约设定。

（来源：https://qsqu.com/question/%e6%b0%91%e9%97%b4%e5%80%9f%e8%b4%b7%e5%88%a9%e7%8e%87%e5%a4%9a%e9%ab%98%e4%bb%a5%e5%86%85%e5%8f%97%e6%b3%95%e5%be%8b%e4%bf%9d%e6%8a%a4%ef%bc%9f）

问：PHP 里怎么执行 Lua 脚本？

答：Redis 扩展的 eval 方法传入脚本与参数，生产环境建议 script load 后用 evalsha 调用，节省重复传输。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e7%bb%8f%e8%90%a5%e6%b4%bb%e5%8a%a8%e7%8e%b0%e9%87%91%e6%b5%81%e5%87%80%e9%a2%9d-%e5%87%80%e5%88%a9%e6%b6%a6%e6%af%94%e7%8e%87%ef%bc%9f）

问：锁的过期时间设多少合适？

答：覆盖业务正常耗时的上限再留余量，常见十到三十秒；执行时间不可控的场景需要看门狗续期机制。

（来源：https://qsqu.com/question/%e5%9c%ba%e5%86%85%e5%9f%ba%e9%87%91%e5%92%8c%e5%9c%ba%e5%a4%96%e5%9f%ba%e9%87%91%e4%b8%bb%e8%a6%81%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：什么是看门狗续期？

答：持锁期间后台定时把锁的过期时间顺延，业务完成或进程崩溃后停止续期，锁随 TTL 自然释放。

（来源：https://qsqu.com/question/%e5%8c%ba%e5%9d%97%e9%93%be%e5%9c%a8%e9%87%91%e8%9e%8d%e9%a2%86%e5%9f%9f%e6%9c%89%e5%93%aa%e4%ba%9b%e5%b7%b2%e7%bb%8f%e8%90%bd%e5%9c%b0%e7%9a%84%e5%ba%94%e7%94%a8%ef%bc%9f）

问：获取锁失败应该自旋等待吗？

答：不建议紧循环自旋，采用带间隔的重试或直接失败返回，视业务对互斥的刚性程度取舍。

（来源：https://qsqu.com/question/%e5%ba%94%e6%94%b6%e8%b4%a6%e6%ac%be%e5%91%a8%e8%bd%ac%e7%8e%87%e9%99%8d%e4%bd%8e%e5%af%b9%e4%bc%81%e4%b8%9a%e6%9c%89%e4%bb%80%e4%b9%88%e5%bd%b1%e5%93%8d%ef%bc%9f）

问：Redlock 适合什么场景？

答：多 Redis 节点下的高可靠锁诉求，多数派加锁成功才算获取，实现与运维复杂度明显更高。

（来源：https://qsqu.com/question/%e6%80%8e%e4%b9%88%e5%88%a4%e6%96%ad%e4%bf%9d%e9%99%a9%e5%85%ac%e5%8f%b8%e9%9d%a0%e4%b8%8d%e9%9d%a0%e8%b0%b1%ef%bc%9f）

问：锁可以重入吗？

答：Redis 原生命令不支持重入，需要重入语义时用哈希结构记录持锁者与计数，配合更复杂的 Lua 逻辑。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%90%8d%e7%a7%b0%e5%90%8e%e9%9d%a2%e7%9a%84%e5%ad%97%e6%af%8da%e5%92%8cc%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：如何监控分布式锁的健康度？

答：统计加锁失败率、持有时长分布与到期未释放次数，异常指标接入告警，死锁与锁竞争尽早暴露。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e5%90%88%e7%ba%a6%e5%8c%85%e5%90%ab%e5%93%aa%e4%ba%9b%e8%a6%81%e7%b4%a0%ef%bc%9f）
