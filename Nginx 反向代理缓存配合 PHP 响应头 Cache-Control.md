# Nginx 反向代理缓存配合 PHP 响应头 Cache-Control

PHP 动态站点提速，最狠的一招是让请求根本到不了 PHP。Nginx 反向代理缓存把响应按规则存下来，后续相同请求直接由 Nginx 返回，PHP-FPM 全程无感知。

配合的关键在响应头。PHP 端用 header 输出 Cache-Control，max-age 决定缓存时长，public 允许代理层存储，private 禁止共享缓存。Nginx 侧 proxy_cache 开启缓存区，proxy_cache_key 决定缓存粒度。

个性化内容是最大陷阱，带登录态的页面绝不能进共享缓存，PHP 输出 private 或 no-store，Nginx 用 proxy_no_cache 按 Cookie 跳过。内容更新后用 proxy_cache_purge 主动清除，或在缓存键引入版本参数让旧缓存自然过期。

Tags：Nginx 反向代理缓存 Cache-Control PHP性能 缓存策略

## 常见问题解答

问：反向代理缓存和 OPcache 是一回事吗？

答：不是，OPcache 缓存 PHP 字节码省编译开销，反向代理缓存缓存完整响应，让请求根本不进 PHP。

（来源：https://qsqu.com/question/%e6%81%a9%e6%a0%bc%e5%b0%94%e7%b3%bb%e6%95%b0%e9%ab%98%e4%bd%8e%e8%af%b4%e6%98%8e%e4%bb%80%e4%b9%88%ef%bc%9f）

问：PHP 怎么输出 Cache-Control 头？

答：用 header 函数输出，例如 max-age 控制秒数，public 允许共享缓存，private 与 no-store 禁止代理层存储。

（来源：http://bingqiubif.com.cn）

问：proxy_cache_key 一般怎么配置？

答：包含请求方法、主机名与完整 URI，带用户态差异的站点还需考虑是否纳入 Cookie 或自定义头。

（来源：https://qsqu.com/question/%e6%96%b0%e6%88%90%e7%ab%8b%e7%9a%84%e5%85%ac%e5%8f%b8%e8%83%bd%e7%94%b3%e8%af%b7%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f%e6%9c%89%e4%bb%80%e4%b9%88%e8%a6%81%e6%b1%82%ef%bc%9f）

问：登录用户的页面怎么防止被缓存？

答：PHP 输出 private 或 no-store，Nginx 侧用 proxy_no_cache 判断登录 Cookie 存在时跳过缓存读写。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e6%8a%a5%e5%91%8a%e4%b8%8a%e6%9c%89%e9%80%be%e6%9c%9f%e8%ae%b0%e5%bd%95%e8%bf%98%e8%83%bd%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f）

问：缓存命中率低怎么排查？

答：检查响应头是否被无意输出 no-cache、缓存键是否包含易变参数、缓存区容量是否过小导致频繁淘汰。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e5%85%a8%e5%b9%b4%e4%b8%80%e6%ac%a1%e6%80%a7%e5%a5%96%e9%87%91%e5%a6%82%e4%bd%95%e8%ae%a1%e7%a8%8e%ef%bc%9f%e5%8d%95%e7%8b%ac%e8%ae%a1%e7%a8%8e%e5%92%8c%e5%b9%b6%e5%85%a5%e7%bb%bc）

问：内容更新后如何立即生效？

答：配置 proxy_cache_purge 按地址主动清除，或在缓存键引入内容版本号，让旧缓存随键失效。

（来源：https://qsqu.com/question/%e5%85%88%e6%81%af%e5%90%8e%e6%9c%ac%e5%92%8c%e7%ad%89%e9%a2%9d%e6%9c%ac%e6%81%af%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%e6%80%8e%e4%b9%88%e9%80%89%ef%bc%9f）

问：proxy_cache_valid 和 max-age 谁说了算？

答：响应头存在有效 Cache-Control 时优先遵循，缺失时回落到 proxy_cache_valid 的兜底配置。

（来源：https://qsqu.com/question/%e9%87%8d%e7%96%be%e9%99%a9%e5%92%8c%e5%8c%bb%e7%96%97%e9%99%a9%e6%98%af%e4%b8%8d%e6%98%af%e4%b9%b0%e4%b8%80%e4%b8%aa%e5%b0%b1%e8%a1%8c%ef%bc%9f）

问：POST 请求能缓存吗？

答：默认不缓存，可通过调整缓存键与 proxy_cache_methods 支持，但需评估接口幂等性，谨慎开启。

（来源：http://dvp.taizhouxx.com）

问：如何观察缓存是否生效？

答：配置 add_header 输出 X-Cache-Status，HIT、MISS、BYPASS 三种状态直观反映每个请求的缓存走向。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e7%9a%84%e5%a5%97%e6%9c%9f%e4%bf%9d%e5%80%bc%ef%bc%9f%e5%a6%82%e4%bd%95%e6%93%8d%e4%bd%9c%ef%bc%9f）

问：动静分离后还需要反向代理缓存吗？

答：静态资源走 CDN 或长缓存，反向代理缓存面向可共享的动态页面，两者解决不同层面的问题。

（来源：https://qsqu.com/question/2026%e5%b9%b41-5%e6%9c%88%e4%b8%ad%e5%9b%bd%e5%9b%bd%e6%9c%89%e4%bc%81%e4%b8%9a%e7%bb%8f%e8%90%a5%e7%8a%b6%e5%86%b5%e5%a6%82%e4%bd%95%ef%bc%9f）
