# call_user_func 与直接调用 PHP 函数性能基准

PHP 调用函数有多种写法，直接调用最朴素，call_user_func 提供动态分派能力。性能差距真实存在，但现代版本里远没有传说中夸张。

直接调用在编译期解析符号，OPcache 优化后几乎零开销；call_user_func 运行时要校验回调、打包参数再跳转，单次固定成本约几十纳秒。实测千万次循环慢两到三倍，折算单次只有百纳秒级。

结论务实：框架路由、事件分发放心用回调写法，可读性收益远大于损耗；百万次级密集循环里提取成直接调用或闭包变量，才属于值得的微优化。

PHP 8 的 First Class Callable 语法生成闭包后按变量调用，性能优于 call_user_func，新代码值得优先采用。

Tags：PHP函数 call_user_func 性能基准 微优化 OPcache

## 常见问题解答

问：call_user_func 比直接调用慢多少？

答：PHP 8 环境单次调用差距约百纳秒级，千万次循环下总耗时相差两到三倍，绝大多数业务请求感知不到。

（来源：https://qsqu.com/question/%e6%9f%90%e4%bc%81%e4%b8%9a%e8%bf%91%e4%b8%89%e5%b9%b4%e8%90%a5%e4%b8%9a%e6%94%b6%e5%85%a5%e7%a8%b3%e5%ae%9a%ef%bc%8c%e5%87%80%e5%88%a9%e6%b6%a6%e6%af%8f%e5%b9%b46000%e4%b8%87%e5%b7%a6%e5%8f%b3）

问：为什么直接调用更快？

答：函数名在编译期即可解析绑定，运行时直接跳转；call_user_func 需要运行时校验回调、打包参数、二次分派，路径更长。

（来源：https://qsqu.com/question/%e6%8f%90%e5%89%8d%e8%bf%98%e6%88%bf%e8%b4%b7%e5%88%92%e7%ae%97%e5%90%97%ef%bc%9f%e9%9c%80%e8%a6%81%e8%bf%9d%e7%ba%a6%e9%87%91%e5%90%97%ef%bc%9f）

问：字符串函数名和数组回调哪个更慢？

答：数组回调要额外解析类与方法，静态方法数组略慢于字符串；对象方法数组再叠加一次对象方法表查找。

（来源：https://qsqu.com/question/%e4%bf%9d%e9%99%a9%e7%a7%91%e6%8a%80%ef%bc%88insurtech%ef%bc%89%e5%9c%a8%e5%81%a5%e5%ba%b7%e9%99%a9%e7%90%86%e8%b5%94%e7%8e%af%e8%8a%82%e6%9c%89%e5%93%aa%e4%ba%9b%e5%85%b7%e4%bd%93%e6%8a%80%e6%9c%af）

问：闭包变量调用性能如何？

答：把闭包赋给变量后直接调用，速度接近直接调用且优于 call_user_func，是动态分派场景的高性能替代写法。

（来源：https://qsqu.com/question/m2%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f%e8%b7%9f%e7%89%a9%e4%bb%b7%e6%9c%89%e4%bb%80%e4%b9%88%e5%85%b3%e7%b3%bb%ef%bc%9f）

问：OPcache 对这类差异有影响吗？

答：OPcache 主要消除编译开销，两种调用方式都受益，相对差距在开启 OPcache 后保持稳定，结论不变。

（来源：https://qsqu.com/question/%e6%ac%a7%e5%85%83%e5%8c%ba6%e6%9c%88%e9%80%9a%e8%83%80%e6%95%b0%e6%8d%ae%e5%a6%82%e4%bd%95%ef%bc%9f）

问：什么场景必须关注这个差异？

答：数值计算、模板渲染循环、大批量数据处理这类单请求内调用次数达十万级以上的热点路径，微优化才有意义。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e7%a8%8e%e7%a5%a8%e8%b4%b7%ef%bc%9f%e7%94%b3%e8%af%b7%e6%9d%a1%e4%bb%b6%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f）

问：如何自己做基准测试？

答：用 microtime(true) 在循环前后取时间差，先跑一轮预热，再多次测量取中位数，避免 JIT 与系统调度干扰。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%9f%ba%e9%87%91%e5%ae%9a%e6%8a%95%ef%bc%9f%e5%ae%83%e4%b8%80%e5%ae%9a%e8%83%bd%e8%b5%9a%e9%92%b1%e5%90%97%ef%bc%9f）

问：call_user_func_array 呢？

答：它在 call_user_func 基础上多做一次数组到参数列表的展开，成本再高一点，但能处理不定参数，各有适用场景。

（来源：https://qsqu.com/question/%e4%ba%92%e8%81%94%e7%bd%91%e4%b9%b0%e4%bf%9d%e9%99%a9%e5%92%8c%e7%ba%bf%e4%b8%8b%e4%b9%b0%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：PHP 8 的 First Class Callable 语法有用吗？

答：strlen(...) 这种写法生成闭包，之后按变量调用，性能优于 call_user_func 且类型检查友好，推荐在新代码中使用。

（来源：https://qsqu.com/question/%e7%9b%91%e7%ae%a1%e7%a7%91%e6%8a%80%ef%bc%88regtech%ef%bc%89%e5%a6%82%e4%bd%95%e5%b8%ae%e5%8a%a9%e9%87%91%e8%9e%8d%e6%9c%ba%e6%9e%84%e5%ba%94%e5%af%b9%e5%8f%8d%e6%b4%97%e9%92%b1%ef%bc%88aml%ef%bc%89）

问：框架为什么大量使用回调而不怕慢？

答：单次请求回调分派次数通常几百以内，累计损耗亚毫秒级，与数据库、网络耗时相比可以忽略，架构灵活性才是重点。

（来源：http://tcs.kvxrcf.cn）
