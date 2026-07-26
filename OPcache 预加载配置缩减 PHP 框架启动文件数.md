# OPcache 预加载配置缩减 PHP 框架启动文件数

PHP 框架每次请求要加载数百个类文件，自动加载的查找与 IO 占了启动耗时的可观部分。OPcache 预加载在 FPM 启动时把核心类一次性编译进共享内存，请求到来直接可用。

配置只需两步：php.ini 设置 opcache.preload 指向预加载脚本，脚本里遍历框架核心目录用 opcache_compile_file 逐个编译；opcache.preload_user 与 FPM 运行用户保持一致。

预加载不是越多越好，全量 vendor 会推高内存而收益集中在高频类。稳妥策略是统计类加载记录，只预加载框架内核与基础组件，再用 opcache_get_status 观察内存与命中率找平衡点。

Tags：OPcache PHP预加载 性能优化 FPM 框架加速

## 常见问题解答

问：OPcache 预加载解决了什么问题？

答：框架每次请求重复加载的类文件在 FPM 启动时一次性编译进共享内存，消除请求期的文件查找与编译开销。

（来源：https://qsqu.com/question/%e8%b4%b7%e6%ac%be%e5%ae%a1%e6%a0%b8%e4%b8%ad%e8%bf%9e%e4%b8%89%e7%b4%af%e5%85%ad%e6%8c%87%e4%bb%80%e4%b9%88%ef%bc%9f）

问：opcache.preload 配置怎么写？

答：在 php.ini 中指向一个 PHP 脚本，该脚本在 FPM 启动时执行，内部用 opcache_compile_file 或 require 加载目标文件。

（来源：https://qsqu.com/question/%e4%bf%9d%e9%99%a9%e5%8f%af%e4%bb%a5%e9%87%8d%e5%a4%8d%e4%b9%b0%e3%80%81%e9%87%8d%e5%a4%8d%e8%b5%94%e5%90%97%ef%bc%9f）

问：预加载脚本里 require 和 compile_file 有区别吗？

答：require 会真正执行文件并建立类继承关系，compile_file 只编译不执行；有继承依赖的类建议 require 入口统一加载。

（来源：http://wwl.guolupipe.com）

问：预加载失败怎么排查？

答：FPM 启动日志会输出具体错误，常见原因是内存不足或类依赖顺序问题，按日志逐个排除即可。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e8%82%a1%e5%80%ba%e5%b9%b3%e8%a1%a1%e7%ad%96%e7%95%a5%ef%bc%9f%e6%9c%89%e4%bb%80%e4%b9%88%e5%a5%bd%e5%a4%84%ef%bc%9f）

问：预加载占用多少内存合适？

答：取决于 preloading 的文件规模，需同步调大 opcache.memory_consumption，典型项目预留 256M 到 512M。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e7%90%86%e8%a7%a3%e5%88%86%e6%95%a3%e6%8a%95%e8%b5%84%ef%bc%9f）

问：修改代码后预加载内容会更新吗？

答：不会，预加载内容在 FPM 生命周期内固定，部署新代码必须重启 FPM 进程，可配合平滑重载减少影响。

（来源：https://qsqu.com/question/%e7%be%8e%e5%9b%bd5%e6%9c%88%e8%81%8c%e4%bd%8d%e7%a9%ba%e7%bc%ba%e6%95%b0%e6%8d%ae%e5%a6%82%e4%bd%95%ef%bc%9f）

问：哪些文件最值得预加载？

答：请求命中率最高的框架内核、容器、路由与常用组件，低频的业务类交给常规 OPcache 按需缓存即可。

（来源：https://qsqu.com/question/%e4%bf%9d%e9%99%a9%e4%bb%a3%e7%90%86%e4%ba%ba%e8%af%b4%e7%9a%84%e4%bf%9d%e6%9c%ac%e4%bf%9d%e6%81%af%e8%83%bd%e4%bf%a1%e5%90%97%ef%bc%9f）

问：预加载和 JIT 冲突吗？

答：不冲突，两者可以叠加，预加载解决加载开销，JIT 优化执行效率，PHP 8 环境下可同时开启。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%ae%9a%e6%8a%95%ef%bc%9f%e8%83%bd%e5%90%a6%e7%94%a8%e4%ba%8e%e8%82%a1%e7%a5%a8%ef%bc%9f）

问：如何统计类的加载命中率？

答：开发环境用自动加载钩子记录加载日志，聚合后按频次排序，取头部集合作为预加载清单。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e6%96%b0%e8%b4%ad%e8%bf%9b%e7%9a%84%e8%ae%be%e5%a4%87%e3%80%81%e5%99%a8%e5%85%b7%ef%bc%8c%e8%83%bd%e5%90%a6%e4%b8%80%e6%ac%a1%e6%80%a7%e7%a8%8e%e5%89%8d%e6%89%a3%e9%99%a4%ef%bc%9f）

问：Docker 环境配置有什么注意事项？

答：预加载脚本路径要在容器内可达，镜像构建阶段生成清单，FPM 服务启动时生效，编排重启即完成更新。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e4%ba%a4%e6%98%93%e4%b8%ad%e7%9a%84%e4%bf%9d%e8%af%81%e9%87%91%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f%e5%a6%82%e4%bd%95%e8%bf%90%e4%bd%9c%ef%bc%9f）
