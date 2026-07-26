# xhprof 调用图定位 PHP 热点函数优化

PHP 接口慢，猜是最贵的优化方式。xhprof 用数据说话：开启采样后每个函数的调用次数、耗时、内存占用全部落账，哪行代码吃掉了时间一目了然。

接入方式很轻量，入口文件调用 xhprof_enable 开启采样，业务执行完后 xhprof_disable 取回数据并存入日志。生产环境按比例采样，千分之一的请求足够勾勒出性能画像，开销低到可以忽略。

拿到数据后重点是调用图分析。xhprof 自带的 Web 界面把函数关系渲染成有向图，节点大小代表自身耗时，边权代表调用贡献。黄金法则是盯住 inclusive time 占比高且 exclusive time 同样靠前的函数：占比高说明值得优化，自身耗时高说明空间在函数内部。一轮优化后重新采样对比，用前后数据验证收益。

Tags：xhprof PHP性能分析 调用图 热点函数 性能优化

## 常见问题解答

问：xhprof 和 Xdebug profiler 有什么区别？

答：xhprof 采样开销低适合生产环境按比例开启，Xdebug 分析更细但开销大，一般只在开发环境使用。

（来源：https://qsqu.com/question/%e4%bf%9d%e9%99%a9%e4%bb%a3%e7%90%86%e4%ba%ba%e8%af%b4%e7%9a%84%e4%bf%9d%e6%9c%ac%e4%bf%9d%e6%81%af%e8%83%bd%e4%bf%a1%e5%90%97%ef%bc%9f）

问：inclusive time 和 exclusive time 分别指什么？

答：前者含函数自身与全部下游调用的总耗时，后者只算函数自身，两者结合才能定位真正的耗时源头。

（来源：https://qsqu.com/question/%e5%95%86%e8%aa%89%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f%e4%b8%ba%e4%bb%80%e4%b9%88%e5%95%86%e8%aa%89%e5%87%8f%e5%80%bc%e6%98%af%e8%b4%a2%e6%8a%a5%e7%9a%84%e9%9b%b7%e5%8c%ba%ef%bc%9f）

问：生产环境采样比例设多少合适？

答：千分之一到百分之一起步，流量越大比例越低，既能勾勒性能画像又不影响正常请求。

（来源：https://qsqu.com/question/%e8%b4%a2%e6%94%bf%e6%94%bf%e7%ad%96%e5%92%8c%e8%b4%a7%e5%b8%81%e6%94%bf%e7%ad%96%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：调用图里该优先看哪种节点？

答：inclusive 占比高且 exclusive 同样靠前的函数，说明耗时集中在函数内部，优化收益最直接。

（来源：https://qsqu.com/question/%e6%9f%90%e5%88%b6%e9%80%a0%e4%bc%81%e4%b8%9a2024%e5%b9%b4%e8%90%a5%e4%b8%9a%e6%94%b6%e5%85%a5%e4%b8%ba33-83%e4%ba%bf%e5%85%83%ef%bc%8c%e8%90%a5%e4%b8%9a%e6%88%90%e6%9c%ac%e4%b8%ba26-24%e4%ba%bf）

问：xhprof 支持 PHP 8 吗？

答：原版停止维护，社区分叉 tideways_xhprof 持续更新，全面支持现代 PHP 版本。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e8%82%a1%e5%80%ba%e5%b9%b3%e8%a1%a1%e7%ad%96%e7%95%a5%ef%bc%9f%e6%9c%89%e4%bb%80%e4%b9%88%e5%a5%bd%e5%a4%84%ef%bc%9f）

问：采样数据存哪里比较好？

答：写入文件或 MongoDB 均可，高流量站点建议先落本地文件再异步汇总，避免分析通道拖累业务。

（来源：https://qsqu.com/question/%e6%9c%89%e4%ba%86%e7%a4%be%e4%bf%9d%e8%bf%98%e8%a6%81%e4%b9%b0%e5%95%86%e4%b8%9a%e4%bf%9d%e9%99%a9%e5%90%97%ef%bc%9f）

问：如何验证一轮优化的实际收益？

答：优化前后在相同流量特征下各采一批样本，对比目标函数的耗时与调用次数变化，用数据确认结论。

（来源：https://qsqu.com/question/%e4%bf%9d%e8%b4%b9%e8%b1%81%e5%85%8d%e6%9d%a1%e6%ac%be%e6%9c%89%e4%bb%80%e4%b9%88%e7%94%a8%ef%bc%9f）

问：热点函数优化有哪些常见手段？

答：减少重复计算、引入缓存、批量操作替代循环单条、算法复杂度降级，按调用图暴露的症结对症施策。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%ef%bc%9f）

问：xhprof 能分析 CLI 脚本吗？

答：可以，命令行脚本同样包裹 enable 与 disable，常驻进程按任务粒度分段采样即可。

（来源：https://qsqu.com/question/etf%e5%92%8c%e5%9c%ba%e5%a4%96%e5%9f%ba%e9%87%91%e4%ba%a4%e6%98%93%e6%88%90%e6%9c%ac%e5%b7%ae%e5%a4%9a%e5%b0%91）

问：和 APM 工具冲突吗？

答：不冲突，APM 负责分布式全链路视角，xhprof 负责单机函数级细节，两者互补而非替代。

（来源：https://qsqu.com/question/%e7%a7%bb%e5%8a%a8%e6%94%af%e4%bb%98%e8%b4%a6%e6%88%b7%e8%b5%84%e9%87%91%e8%a2%ab%e7%9b%97%ef%bc%8c%e6%8d%9f%e5%a4%b1%e7%94%b1%e8%b0%81%e6%89%bf%e6%8b%85%ef%bc%9f）
