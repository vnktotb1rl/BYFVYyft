# PHP 迭代器管道实现 filter/map/reduce 延迟计算

用 array_filter、array_map 链式处理大数组，每一步都生成完整中间数组，内存随数据量膨胀。迭代器管道把每步改造成生成器，数据逐条穿过各环节，全程只占一份内存。

每个管道环节是接收 iterable 返回 Generator 的函数：filter 命中条件才 yield，map 变换后 yield，reduce 作为终点站把流收敛成单一结果。外层循环开始时整条链才真正运转。

延迟计算还带来短路能力，find 类操作命中即停，后续数据不再加载。配合数据库游标或文件逐行读取，处理百万级记录也游刃有余，nikic/iter 等库提供了现成实现。

Tags：PHP迭代器 生成器 延迟计算 管道模式 内存优化

## 常见问题解答

问：生成器和普通数组的本质区别是什么？

答：数组一次性承载全部数据，生成器只在被迭代时逐条产出，内存占用与数据总量脱钩。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e9%98%b2%e8%8c%83%e8%b4%b7%e6%ac%be%e8%bf%87%e7%a8%8b%e4%b8%ad%e7%9a%84%e8%af%88%e9%aa%97%e9%a3%8e%e9%99%a9%ef%bc%9f）

问：filter 管道函数怎么写？

答：定义接收 iterable 与断言闭包的函数，内部 foreach 遍历并用 yield 输出通过断言的元素，返回类型声明为 Generator。

（来源：http://qyw.kvxrcf.cn）

问：map 和 filter 的执行顺序有讲究吗？

答：先 filter 再 map 可以减少变换次数；反之若过滤依赖变换后的值，则必须先 map，按数据依赖关系排布。

（来源：https://qsqu.com/question/%e5%95%86%e8%aa%89%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f%e4%b8%ba%e4%bb%80%e4%b9%88%e5%95%86%e8%aa%89%e5%87%8f%e5%80%bc%e6%98%af%e8%b4%a2%e6%8a%a5%e7%9a%84%e9%9b%b7%e5%8c%ba%ef%bc%9f）

问：reduce 为什么无法延迟？

答：聚合必须看完全部数据才能得出结果，它是管道的消费端而非中间环节，放在链条末尾执行。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e9%85%8d%e7%bd%ae%e5%9f%ba%e9%87%91%e7%bb%84%e5%90%88%e9%99%8d%e4%bd%8e%e9%a3%8e%e9%99%a9）

问：管道能和数据库查询结合吗？

答：可以，用 PDO 的逐行 fetch 包装成生成器作为管道源头，百万行结果集也能以常量内存处理。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e5%b0%8f%e5%9e%8b%e5%be%ae%e5%88%a9%e4%bc%81%e4%b8%9a%e7%9a%84%e4%bc%81%e4%b8%9a%e6%89%80%e5%be%97%e7%a8%8e%e4%bc%98%e6%83%a0%e5%85%b7%e4%bd%93%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f）

问：生成器可以被遍历两次吗？

答：不能，生成器是一次性的，再次 foreach 会抛异常；需要复用时把结果收集成数组或重建生成器。

（来源：https://qsqu.com/question/%e7%ad%89%e9%a2%9d%e6%9c%ac%e6%81%af%e5%92%8c%e7%ad%89%e9%a2%9d%e6%9c%ac%e9%87%91%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：短路操作怎么实现？

答：在消费端循环中命中目标后 break，上游生成器随之停止产出，未读取的数据根本不会加载。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e6%9c%89%e9%80%be%e6%9c%9f%e8%ae%b0%e5%bd%95%ef%bc%8c%e5%a4%9a%e4%b9%85%e8%83%bd%e6%b6%88%e9%99%a4%ef%bc%9f）

问：延迟计算会降低总耗时吗？

答：不一定，单条处理开销略增；它的价值在于内存可控与提前终止，大数据量下整体反而更快。

（来源：https://qsqu.com/question/%e6%80%8e%e4%b9%88%e5%88%a4%e6%96%ad%e4%bf%9d%e9%99%a9%e5%85%ac%e5%8f%b8%e9%9d%a0%e4%b8%8d%e9%9d%a0%e8%b0%b1%ef%bc%9f）

问：有哪些现成库可用？

答：nikic/iter 提供 map、filter、reduce 等惰性操作，Laravel 的 LazyCollection 也是同一思想的封装。

（来源：http://xjliansaibifen.com.cn）

问：什么场景不适合迭代器管道？

答：数据量小且需要多次随机访问的场景，数组的随机存取与复用能力更合适，管道反而增加复杂度。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e6%b5%81%e6%b0%b4%e5%af%b9%e8%b4%b7%e6%ac%be%e9%a2%9d%e5%ba%a6%e6%9c%89%e4%bb%80%e4%b9%88%e5%bd%b1%e5%93%8d%ef%bc%9f）
