# OPcache 预加载列表配置及命中率监控

OPcache 预加载的完整落地包含两件功课：一份精确的预加载清单，一套持续运转的命中率监控。清单决定内存花得值不值，监控决定这个判断是否持续成立。

清单生成走数据驱动：预发环境挂自动加载记录器，回放核心请求后按加载频次排序，头部几百个文件就是候选。清单脚本用 opcache_compile_file 逐个编译，路径必须是解析符号链接后的真实绝对路径，框架的容器与路由编译产物务必列入。

监控围绕 opcache_get_status 三个指标：内存占用看 memory_usage，命中率看 hits 与 misses 比值，健康水位百分之九十九以上；restarts 计数反映频繁重启的失衡信号。指标接入监控系统定期采集，预加载从一次性配置变成持续调优的运营项。

Tags：OPcache PHP预加载 命中率监控 性能调优 FPM

## 常见问题解答

问：预加载清单应该包含哪些文件？

答：请求命中率最高的框架内核、容器与路由编译产物、高频基础组件，低频业务类交给常规缓存按需加载。

（来源：https://qsqu.com/question/%e5%bc%80%e6%94%be%e9%93%b6%e8%a1%8c%ef%bc%88open-banking%ef%bc%89%e7%9a%84%e6%8a%80%e6%9c%af%e6%9e%b6%e6%9e%84%e4%b8%8e%e5%95%86%e4%b8%9a%e4%bb%b7%e5%80%bc%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f）

问：清单数据从哪里来？

答：开发环境记录自动加载日志，按加载频次排序取头部集合，数据驱动比拍脑袋清单可靠得多。

（来源：http://dongqiuty9.com.cn）

问：为什么清单里必须用绝对路径？

答：相对路径依赖启动时的工作目录，符号链接目录在部署切换后会失效，真实路径保证清单稳定。

（来源：http://shandiankanq.com.cn）

问：opcache_compile_file 和 require 有何区别？

答：前者只编译不执行，后者执行并建立类继承关系；有继承依赖的类建议通过 require 入口类统一加载。

（来源：https://qsqu.com/question/%e5%9f%ba%e5%b0%bc%e7%b3%bb%e6%95%b0%e6%98%af%e8%a1%a1%e9%87%8f%e4%bb%80%e4%b9%88%e7%9a%84%ef%bc%9f%e6%95%b0%e5%80%bc%e5%a4%9a%e5%b0%91%e7%ae%97%e5%90%88%e7%90%86%ef%bc%9f）

问：命中率监控看哪些指标？

答：opcache_get_status 输出的 hits、misses、memory_usage 与 restarts 计数，四者组合判断缓存健康度。

（来源：https://qsqu.com/question/%e5%85%b7%e8%ba%ab%e6%99%ba%e8%83%bd%e4%ba%a7%e4%b8%9a%e5%8f%91%e5%b1%95%e6%83%85%e5%86%b5%e5%a6%82%e4%bd%95%ef%bc%9f）

问：命中率多少算健康？

答：稳态百分之九十九以上，持续走低说明代码变更后未重启或内存不足导致条目被淘汰。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%8a%bd%e8%b4%b7%ef%bc%9f%e5%a6%82%e4%bd%95%e9%81%bf%e5%85%8d%ef%bc%9f）

问：内存满了会发生什么？

答：新文件无法缓存且可能触发重启清空，命中率骤降，需要调大 memory_consumption 或精简预加载规模。

（来源：https://qsqu.com/question/%e5%85%ab%e9%83%a8%e9%97%a8%e5%8f%91%e5%b8%83%e4%ba%86%e4%bb%80%e4%b9%88%e5%85%b3%e4%ba%8e%e5%b7%a5%e4%b8%9a%e4%ba%92%e8%81%94%e7%bd%91%e7%9a%84%e9%87%8d%e8%a6%81%e6%96%87%e4%bb%b6%ef%bc%9f）

问：指标如何接入监控系统？

答：写一个内部状态页输出 opcache_get_status 结果，采集端定时抓取转成时序指标，配合阈值告警。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e6%8a%a5%e5%91%8a%e4%b8%ad%e7%9a%84%e8%bf%9e%e4%b8%89%e7%b4%af%e5%85%ad%e6%98%af%e4%bb%80%e4%b9%88%e6%84%8f%e6%80%9d%ef%bc%9f）

问：部署新版本后需要做什么？

答：平滑重启 FPM 使预加载清单重建，直接热改代码不重启会让旧字节码继续服务，属于高危操作。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%8c%87%e6%95%b0%e5%9f%ba%e9%87%91%ef%bc%9f%e5%ae%83%e5%92%8c%e4%b8%bb%e5%8a%a8%e5%9e%8b%e5%9f%ba%e9%87%91%e6%9c%89%e4%bb%80%e4%b9%88%e4%b8%8d%e5%90%8c%ef%bc%9f）

问：预加载和 JIT 能同时开吗？

答：可以叠加，预加载管加载开销，JIT 管执行效率，生产环境两者并存是高性能 PHP 的常见配置。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e6%9c%89%e9%80%be%e6%9c%9f%e8%ae%b0%e5%bd%95%ef%bc%8c%e5%a4%9a%e4%b9%85%e8%83%bd%e6%b6%88%e9%99%a4%ef%bc%9f）
