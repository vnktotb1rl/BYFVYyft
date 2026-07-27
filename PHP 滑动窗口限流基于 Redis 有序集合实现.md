# PHP 滑动窗口限流基于 Redis 有序集合实现

滑动窗口限流是接口保护中精度较高的一种方案，相比固定窗口计数器，它能够避免窗口边界处的流量突刺问题。在 PHP 中利用 Redis 有序集合（ZSET）实现该算法时，通常以当前时间戳作为 score，以唯一请求标识作为 member，通过 ZREMRANGEBYSCORE 清理窗口外的过期记录，再用 ZCARD 统计窗口内请求数，超过阈值即拒绝请求。

工程实践中需要特别注意原子性问题。清理、计数、写入三步操作如果分开发送，在并发场景下会出现竞态，导致限流不准。推荐将逻辑封装进 Lua 脚本通过 EVAL 一次性执行，既保证原子性，又减少网络往返。同时应为每个 key 设置略大于窗口时长的过期时间，防止冷 key 长期占用内存。

在高并发接口上，这种方案的额外开销约为每次请求一次 Redis 往返，延迟通常在毫秒级以内。对于秒杀、短信发送、开放 API 等场景，滑动窗口配合令牌桶可以形成多层防护。需要注意的是，PHP-FPM 模式下应使用持久连接或连接池，避免频繁建连拖垮 Redis。

Tags：PHP Redis 滑动窗口 限流 有序集合

## 内链

## 快讯

Elastic APM 分布式追踪 PHP 事务与跨度关系
Elastic APM 代理自动捕获 PHP 请求事务，按 Trace ID 串联子跨度，还原跨服务链路并定位慢节点。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/Elastic%20APM%20%E5%88%86%E5%B8%83%E5%BC%8F%E8%BF%BD%E8%B8%AA%20PHP%20%E4%BA%8B%E5%8A%A1%E4%B8%8E%E8%B7%A8%E5%BA%A6%E5%85%B3%E7%B3%BB.md

JsonSerializable 控制 PHP 输出字段格式
实现该接口后 json_encode 采用 jsonSerialize 返回值，可按场景裁剪字段并统一日期金额输出格式。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/JsonSerializable%20%E6%8E%A5%E5%8F%A3%E6%8E%A7%E5%88%B6%20PHP%20%E8%BE%93%E5%87%BA%E5%AD%97%E6%AE%B5%E6%A0%BC%E5%BC%8F.md

Nginx 反向代理缓存与 PHP 响应头协同
PHP 输出 Cache-Control 响应头后，Nginx 反向代理据此缓存热点页面，显著降低后端 PHP 请求压力。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/Nginx%20%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86%E7%BC%93%E5%AD%98%E9%85%8D%E5%90%88%20PHP%20%E5%93%8D%E5%BA%94%E5%A4%B4%20Cache-Control.md

OPcache 预加载列表配置方法与命中率监控要点
OPcache 预加载在 PHP-FPM 启动时把核心类编译进共享内存，配合命中率监控可及时发现容量不足。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/OPcache%20%E9%A2%84%E5%8A%A0%E8%BD%BD%E5%88%97%E8%A1%A8%E9%85%8D%E7%BD%AE%E5%8F%8A%E5%91%BD%E4%B8%AD%E7%8E%87%E7%9B%91%E6%8E%A7.md

OPcache 预加载配置有效缩减 PHP 框架启动文件数
框架引导需加载数百个文件，预加载脚本将高频类一次性驻留内存，请求阶段省去磁盘读取与编译，启动耗时明显下降。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/OPcache%20%E9%A2%84%E5%8A%A0%E8%BD%BD%E9%85%8D%E7%BD%AE%E7%BC%A9%E5%87%8F%20PHP%20%E6%A1%86%E6%9E%B6%E5%90%AF%E5%8A%A8%E6%96%87%E4%BB%B6%E6%95%B0.md

PHP IMAP 扩展读取邮件附件解析嵌套 MIME 结构
IMAP 扩展按 part 递归遍历嵌套 MIME 结构，依据 Content-Type 与编码逐层解码，提取邮件附件。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20IMAP%20%E8%AF%BB%E5%8F%96%E9%82%AE%E4%BB%B6%E9%99%84%E4%BB%B6%E8%A7%A3%E6%9E%90%E5%B5%8C%E5%A5%97%20MIME%20%E7%BB%93%E6%9E%84.md

PHP LDAP 用户同步方案与组权限实时更新机制
通过 ldap_search 定期拉取目录用户与组关系，比对本地差异后增量写入，配合事件触发可实现组织权限的近实时同步。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20LDAP%20%E7%94%A8%E6%88%B7%E5%90%8C%E6%AD%A5%E4%B8%8E%E7%BB%84%E6%9D%83%E9%99%90%E5%AE%9E%E6%97%B6%E6%9B%B4%E6%96%B0.md

PHP Memcached 一致性哈希与故障转移配置实践
Memcached 客户端开启一致性哈希后节点增减仅影响少量键分布，配合自动摘除与重试策略可在故障时平滑转移缓存流量。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20Memcached%20%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E6%95%85%E9%9A%9C%E8%BD%AC%E7%A7%BB%E9%85%8D%E7%BD%AE.md

PHP Redis 分布式锁借助 Lua 脚本实现原子释放
释放锁需先校验唯一标识再删除键，两步合成 Lua 脚本原子执行可避免误删他人锁，加锁时设置唯一值与过期时间是前提。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20Redis%20%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%20Lua%20%E8%84%9A%E6%9C%AC%E5%8E%9F%E5%AD%90%E9%87%8A%E6%94%BE%E7%A4%BA%E4%BE%8B.md

PHP UUID 有序与随机版本对 InnoDB 索引影响
随机 UUID 作主键会引发 InnoDB 页分裂与插入随机化，有序 UUIDv7 保持索引局部性，更适合高写入表。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20UUID%20%E6%9C%89%E5%BA%8F%E4%B8%8E%E9%9A%8F%E6%9C%BA%E7%89%88%E6%9C%AC%E5%AF%B9%20InnoDB%20%E7%B4%A2%E5%BC%95%E5%BD%B1%E5%93%8D.md

PHP 多字节正则中 p{L} 与 w 的匹配范围差异
开启 u 修饰符后 w 仍只匹配字母数字下划线，中文需用 p{L} 等 Unicode 属性类，混用会造成匹配偏差。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E5%A4%9A%E5%AD%97%E8%8A%82%E6%AD%A3%E5%88%99%20p%7BL%7D%20%E4%B8%8E%20w%20%E5%8C%B9%E9%85%8D%E5%B7%AE%E5%BC%82.md

PHP 多级环境变量覆盖策略与配置缓存刷新机制
系统级、容器注入与 .env 文件三级环境变量存在覆盖顺序，明确优先级并规范缓存刷新流程，可避免配置漂移与排查困难。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E5%A4%9A%E7%BA%A7%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E8%A6%86%E7%9B%96%E7%AD%96%E7%95%A5%E4%B8%8E%E7%BC%93%E5%AD%98%E5%88%B7%E6%96%B0.md

PHP 8.4 属性钩子 get set 注入验证与变更日志
PHP 8.4 属性钩子允许在 get、set 中内联校验与副作用逻辑，无需样板方法即可记录字段变更日志。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E5%B1%9E%E6%80%A7%E9%92%A9%E5%AD%90%20get_set%20%E6%B3%A8%E5%85%A5%E9%AA%8C%E8%AF%81%E4%B8%8E%E5%8F%98%E6%9B%B4%E6%97%A5%E5%BF%97.md

慢查询日志定位问题 SQL 与执行计划索引类型解读
开启慢查询日志并设置阈值可收集耗时 SQL，EXPLAIN 的 type 从 const 到 ALL 逐级变差反映索引效率。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E6%85%A2%E6%9F%A5%E8%AF%A2%E6%97%A5%E5%BF%97%E5%AE%9A%E4%BD%8D%E4%B8%8E%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E7%B4%A2%E5%BC%95%E7%B1%BB%E5%9E%8B%E8%A7%A3%E8%AF%BB.md

PHP 数组键整数字符串隐式转换规则与预防措施
PHP 数组键中合法整数字符串会被转为整型，带前导零与浮点键另有规则，混合键型易引发覆盖，建议统一键类型并加严格校验。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E6%95%B0%E7%BB%84%E9%94%AE%E6%95%B4%E6%95%B0%E5%AD%97%E7%AC%A6%E4%B8%B2%E9%9A%90%E5%BC%8F%E8%BD%AC%E6%8D%A2%E8%A7%84%E5%88%99%E5%8F%8A%E9%A2%84%E9%98%B2.md

PHP 死信队列失败任务的手动重跑后台实现方案
消费失败超限的消息转入死信队列后，后台提供查询筛选与手动重跑入口，重跑前校验幂等键，可防止重复执行造成数据错乱。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E6%AD%BB%E4%BF%A1%E9%98%9F%E5%88%97%E5%A4%B1%E8%B4%A5%E4%BB%BB%E5%8A%A1%E6%89%8B%E5%8A%A8%E9%87%8D%E8%B7%91%E5%90%8E%E5%8F%B0%E5%AE%9E%E7%8E%B0.md

PHP 用 endroid/qr-code 生成二维码
endroid/qr-code 支持设置尺寸、容错级别与中心 Logo，输出 PNG、SVG 格式，还能叠加文本水印。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E7%94%A8%20endroid_qr-code%20%E7%94%9F%E6%88%90%E4%BA%8C%E7%BB%B4%E7%A0%81%E4%B8%8E%E6%9D%A1%E5%BD%A2%E7%A0%81.md

PHP 自定义异常继承基类的选择标准
运行期错误继承 RuntimeException，业务异常继承 Exception，分层定义便于分类捕获与统一兜底处理。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E8%87%AA%E5%AE%9A%E4%B9%89%E5%BC%82%E5%B8%B8%E7%BB%A7%E6%89%BF%20RuntimeException%20%E6%88%96%20Exception.md

PHP 自定义错误页面同时记录完整堆栈追踪信息
注册异常与错误处理器后，生产环境向用户展示友好错误页，后台记录完整堆栈与请求上下文，兼顾访问体验与排查效率。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E8%87%AA%E5%AE%9A%E4%B9%89%E9%94%99%E8%AF%AF%E9%A1%B5%E9%9D%A2%E5%B9%B6%E8%AE%B0%E5%BD%95%E5%AE%8C%E6%95%B4%E5%A0%86%E6%A0%88%E8%BF%BD%E8%B8%AA.md

PHP 装饰器模式动态扩展功能且不违背单一职责
装饰器模式通过包装原对象按需叠加缓存、日志等能力，各装饰器职责单一，运行期可自由组合，避免继承层级无限膨胀。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E8%A3%85%E9%A5%B0%E5%99%A8%E6%A8%A1%E5%BC%8F%E6%89%A9%E5%B1%95%E5%8A%9F%E8%83%BD%E4%B8%8D%E8%BF%9D%E8%83%8C%E5%8D%95%E4%B8%80%E8%81%8C%E8%B4%A3.md

## 外链
## 常见问题

滑动窗口限流和固定窗口限流有什么区别？
固定窗口按自然时间段计数，窗口边界处可能出现两倍阈值的流量突刺；滑动窗口以当前时刻向前回溯一个窗口长度统计，任意时间点都不超限，控制更平滑精确。
来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%9f%ba%e9%87%91%e5%ae%9a%e6%8a%95%ef%bc%9f%e5%ae%83%e4%b8%80%e5%ae%9a%e8%83%bd%e8%b5%9a%e9%92%b1%e5%90%97%ef%bc%9f

为什么用 Redis 有序集合而不是普通计数器？
有序集合按 score 排序存储，可以精确删除窗口外的历史记录，从而实现真正的滑动窗口；普通计数器只能按固定周期清零，无法表达时间维度。
来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%90%8d%e7%a7%b0%e5%90%8e%e9%9d%a2%e7%9a%84%e5%ad%97%e6%af%8da%e5%92%8cc%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f

为什么要用 Lua 脚本执行限流逻辑？
清理过期记录、统计数量、写入新记录涉及多个命令，Lua 脚本在 Redis 内原子执行，避免并发请求之间互相干扰，同时把多次网络往返合并为一次。
来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%88%86%e7%ba%a2%e6%98%af%e4%b8%8d%e6%98%af%e9%a2%9d%e5%a4%96%e8%b5%9a%e5%88%b0%e7%9a%84%e9%92%b1%ef%bc%9f

有序集合的 member 应该怎么设计？
member 需要保证唯一性，常使用时间戳加随机串或自增序列，避免同一毫秒内的多个请求因 member 相同而被去重覆盖，导致计数偏少。
来源：https://qsqu.com/question/%e6%80%8e%e4%b9%88%e7%ae%80%e5%8d%95%e5%88%a4%e6%96%ad%e4%b8%80%e5%8f%aa%e5%9f%ba%e9%87%91%e7%9a%84%e9%a3%8e%e9%99%a9%e6%b0%b4%e5%b9%b3%ef%bc%9f

限流 key 不设置过期时间会有什么后果？
长期不活跃的限流对象会残留空 key 或过期数据，持续占用 Redis 内存。设置略大于窗口时长的 TTL 可以让空闲 key 自动回收。
来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%8c%87%e6%95%b0%e5%9f%ba%e9%87%91%ef%bc%9f%e5%ae%83%e5%92%8c%e4%b8%bb%e5%8a%a8%e5%9e%8b%e5%9f%ba%e9%87%91%e6%9c%89%e4%bb%80%e4%b9%88%e4%b8%8d%e5%90%8c%ef%bc%9f

PHP-FPM 模式下连接 Redis 有什么注意事项？
每次请求结束后连接会释放，高并发下频繁建连开销明显。可使用 pconnect 持久连接，或借助 Swoole 等常驻内存框架维护连接池。
来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e6%8a%95%e8%b5%84%e4%b8%ad%e7%9a%84%e6%9c%80%e5%a4%a7%e5%9b%9e%e6%92%a4%e6%98%af%e4%bb%80%e4%b9%88%e6%84%8f%e6%80%9d%ef%bc%9f

滑动窗口限流适合哪些业务场景？
适合对精度要求较高的接口保护，例如短信验证码发送频率控制、开放平台 API 调用配额、登录防爆破以及秒杀活动的入口限流。
来源：https://qsqu.com/question/%e5%9c%ba%e5%86%85%e5%9f%ba%e9%87%91%e5%92%8c%e5%9c%ba%e5%a4%96%e5%9f%ba%e9%87%91%e4%b8%bb%e8%a6%81%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f

Redis 不可用时限流逻辑怎么处理？
应设计降级策略，例如 Redis 异常时放行并记录告警，或退化为本地内存计数。直接抛错会导致业务整体不可用，风险反而更大。
来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e8%82%a1%e5%80%ba%e5%b9%b3%e8%a1%a1%e7%ad%96%e7%95%a5%ef%bc%9f%e6%9c%89%e4%bb%80%e4%b9%88%e5%a5%bd%e5%a4%84%ef%bc%9f

集群部署时按用户限流的 key 如何分布？
可以使用 Redis Cluster 的 hash tag 让同一用户或接口的 key 落在同一槽位，保证 Lua 脚本操作单 key 时不会跨节点报错。
来源：https://qsqu.com/question/%e4%b9%b0%e5%9f%ba%e9%87%91%e9%9c%80%e8%a6%81%e5%85%b3%e6%b3%a8%e5%93%aa%e4%ba%9b%e8%b4%b9%e7%94%a8%ef%bc%9f

滑动窗口算法的主要缺点是什么？
每个请求都要写入一条记录，内存占用与窗口内请求量成正比。窗口大、阈值高时存储开销明显，可换用滑动日志的近似算法或令牌桶降低消耗。
来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e4%b8%80%e8%88%ac%e6%8c%81%e6%9c%89%e5%a4%9a%e4%b9%85%e6%af%94%e8%be%83%e5%90%88%e9%80%82%ef%bc%9f
