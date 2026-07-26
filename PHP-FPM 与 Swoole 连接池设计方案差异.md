# PHP-FPM 与 Swoole 连接池设计方案差异

连接池要复用昂贵的数据库连接，但 PHP-FPM 与 Swoole 运行模型不同，连接池设计从出发点就分道扬镳。

PHP-FPM 一次请求一次生命周期，无法跨请求共享对象，所谓连接池只能退化为 PDO 持久连接，无法控制容量，进程数一多就把数据库连接打满。

Swoole 常驻内存，协程下用 Channel 实现真连接池：预建连接塞进 Channel，用时 pop、用完 push，容量天然限制并发上限，归还时做健康检查。长连场景还要用心跳对抗 wait_timeout 断链。

容量规划的经验值是数据库 max_connections 除以实例数再留三成余量，池过大加剧切换，过小则请求排队。

Tags：PHP-FPM Swoole 连接池 协程 常驻内存

## 常见问题解答

问：PHP-FPM 下能实现真正的连接池吗？

答：不能跨请求共享内存对象，只能依赖 PDO 持久连接做进程级复用，严格意义上不算池，因为无法统一控制容量与归还。

（来源：http://24zbwang.com.cn）

问：PDO 持久连接怎么开启？

答：构造 PDO 时在 options 数组中设置 PDO::ATTR_PERSISTENT 为 true，此后同一 FPM 子进程的新请求会复用已有连接。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f）

问：持久连接的最大风险是什么？

答：每个 FPM 子进程都持有若干长连接，进程数乘以每进程连接数可能超过数据库 max_connections，引发拒绝连接故障。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e6%8a%95%e8%b5%84%e4%b8%ad%e7%9a%84%e6%9c%80%e5%a4%a7%e5%9b%9e%e6%92%a4%e6%98%af%e4%bb%80%e4%b9%88%e6%84%8f%e6%80%9d%ef%bc%9f）

问：Swoole 连接池为什么用 Channel？

答：Channel 是协程安全的队列，容量固定且支持阻塞等待，天然契合连接的借出与归还语义，超容量时协程挂起而非报错。

（来源：https://qsqu.com/question/%e6%b0%91%e9%97%b4%e5%80%9f%e8%b4%b7%e5%88%a9%e7%8e%87%e5%a4%9a%e9%ab%98%e4%bb%a5%e5%86%85%e5%8f%97%e6%b3%95%e5%be%8b%e4%bf%9d%e6%8a%a4%ef%bc%9f）

问：Swoole 连接池如何处理断线重连？

答：借出连接时先执行一次轻量查询探测活性，失败则销毁重建；同时启动定时器对池内空闲连接发送心跳维持在线。

（来源：https://qsqu.com/question/%e7%a0%8d%e5%a4%b4%e6%81%af%e5%90%88%e6%b3%95%e5%90%97%ef%bc%9f）

问：MySQL wait_timeout 对长连接有什么影响？

答：服务端会主动关闭空闲超时的连接，客户端不知情继续复用就会触发 gone away 错误，心跳保活或缩短池内空闲时间是标准对策。

（来源：https://qsqu.com/question/etf%e5%92%8c%e5%9c%ba%e5%a4%96%e5%9f%ba%e9%87%91%e4%ba%a4%e6%98%93%e6%88%90%e6%9c%ac%e5%b7%ae%e5%a4%9a%e5%b0%91）

问：连接池大小设置多少合适？

答：经验值是数据库 max_connections 除以应用实例数再留三成余量，池子过大反而加剧上下文切换，过小则请求排队。

（来源：http://txq.taizhouxx.com）

问：FPM 模式还有其他复用方案吗？

答：可以在 PHP 与数据库之间加一层 ProxySQL 之类的中间件，由中间件维护到数据库的连接池，PHP 侧保持短连接。

（来源：http://banqiubif.com.cn）

问：Swoole 多进程下每个进程都要有池吗？

答：是的，连接对象无法跨进程共享，每个 worker 进程各自维护一个池，容量规划时要把 worker 数量纳入计算。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%8c%87%e6%95%b0%e5%9f%ba%e9%87%91%ef%bc%9f%e5%ae%83%e5%92%8c%e4%b8%bb%e5%8a%a8%e5%9e%8b%e5%9f%ba%e9%87%91%e6%9c%89%e4%bb%80%e4%b9%88%e4%b8%8d%e5%90%8c%ef%bc%9f）

问：如何监控连接池健康度？

答：暴露池内空闲数、借出数、等待协程数等指标，接入 Prometheus 之类的监控体系，等待时间过长时及时告警扩容。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e4%bf%a1%e7%94%a8%e8%b4%b7%e6%ac%be%e5%92%8c%e6%8a%b5%e6%8a%bc%e8%b4%b7%e6%ac%be%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）
