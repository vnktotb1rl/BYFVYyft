# PHP 股票行情推送数据压缩与心跳包设计

股票行情推送对实时性和带宽利用率要求极高，PHP 侧通常借助 Swoole 或 Workerman 构建 WebSocket 长连接服务，向客户端持续分发行情快照与逐笔更新。行情数据频次高、字段重复度大，若直接推送 JSON 明文，带宽成本会随订阅数线性膨胀，因此压缩与协议设计是系统成败的关键。

压缩层面有多级手段。应用层可采用增量推送，只发送相对上一笔变化的字段，或用字段编号替代字段名、数组代替对象来缩小报文；传输层可开启 WebSocket 的 permessage-deflate 扩展进行 Deflate 压缩，热点行情还可按档位批量合并，把一秒内的多次变动聚合成一帧下发。实测这些手段叠加后带宽可下降五至八成。

心跳机制用于甄别真假断连。服务端可每隔三十秒发送 ping 帧，客户端按时回应 pong，连续多个周期未响应即主动关闭连接并回收资源。需要注意运营商 NAT 会静默清理空闲连接，心跳间隔不宜超过六十秒；同时要区分网络空闲与业务空闲，行情休市期间连接仍需保活，避免客户端反复重连造成接入层抖动。

Tags：PHP WebSocket 行情推送 数据压缩 心跳包

## 内链

## 快讯

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

PHP 迭代器管道实现延迟计算
基于生成器串联 filter、map、reduce 形成惰性管道，元素逐个流过各阶段，处理大数据集内存占用恒定。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E8%BF%AD%E4%BB%A3%E5%99%A8%E7%AE%A1%E9%81%93%E5%AE%9E%E7%8E%B0%20filter_map_reduce%20%E5%BB%B6%E8%BF%9F%E8%AE%A1%E7%AE%97.md

PHP-FPM 与 Swoole 在连接池设计上的方案差异
PHP-FPM 请求结束即释放资源，连接难以跨请求复用；Swoole 常驻内存可在进程内维护连接池，降低建连开销。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP-FPM%20%E4%B8%8E%20Swoole%20%E8%BF%9E%E6%8E%A5%E6%B1%A0%E8%AE%BE%E8%AE%A1%E6%96%B9%E6%A1%88%E5%B7%AE%E5%BC%82.md

Protobuf 与 JSON 微服务传输带宽对比
相同数据结构下 Protobuf 报文体积通常仅为 JSON 的三到五成且序列化更快，但可读性较差需权衡。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/Protobuf%20%E4%B8%8E%20JSON%20%E5%9C%A8%20PHP%20%E5%BE%AE%E6%9C%8D%E5%8A%A1%E4%BC%A0%E8%BE%93%E5%B8%A6%E5%AE%BD%E5%AF%B9%E6%AF%94.md

Valitron 链式校验用法与自定义错误消息编写技巧
Valitron 以链式调用声明校验规则，支持按字段定制错误消息与多语言输出，可在任意 PHP 项目中接入。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/Valitron%20%E9%93%BE%E5%BC%8F%E6%A0%A1%E9%AA%8C%E4%B8%8E%E8%87%AA%E5%AE%9A%E4%B9%89%E9%94%99%E8%AF%AF%E6%B6%88%E6%81%AF%E7%BC%96%E5%86%99.md

WebSocket 压缩帧与消息合并降低 PHP 推送带宽
开启 permessage-deflate 压缩帧并按时间窗合并消息，高频推送带宽可下降过半，代价是 CPU 占用上升。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/WebSocket%20%E5%8E%8B%E7%BC%A9%E5%B8%A7%E4%B8%8E%E6%B6%88%E6%81%AF%E5%90%88%E5%B9%B6%E9%99%8D%E4%BD%8E%20PHP%20%E6%8E%A8%E9%80%81%E5%B8%A6%E5%AE%BD.md

call_user_func 与 PHP 函数直调性能基准
基准测试显示 call_user_func 比直接调用慢约两到三成，PHP 8 后差距缩小，热点路径仍建议直接调用。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/call_user_func%20%E4%B8%8E%E7%9B%B4%E6%8E%A5%E8%B0%83%E7%94%A8%20PHP%20%E5%87%BD%E6%95%B0%E6%80%A7%E8%83%BD%E5%9F%BA%E5%87%86.md

composer.lock 锁定机制与语义化版本策略的运用
composer.lock 固化依赖精确版本，保证各环境安装结果一致；版本约束遵循语义化规则，平衡安全更新与稳定性。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/composer.lock%20%E9%94%81%E5%AE%9A%E4%B8%8E%E8%AF%AD%E4%B9%89%E5%8C%96%E7%89%88%E6%9C%AC%E7%AD%96%E7%95%A5%E8%BF%90%E7%94%A8.md

xhprof 调用图定位 PHP 热点函数指导性能优化
xhprof 采样生成函数级调用图，按耗时与调用次数排序可快速锁定热点函数，优化前后对比报告能量化每一步改动收益。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/xhprof%20%E8%B0%83%E7%94%A8%E5%9B%BE%E5%AE%9A%E4%BD%8D%20PHP%20%E7%83%AD%E7%82%B9%E5%87%BD%E6%95%B0%E4%BC%98%E5%8C%96.md

命令模式结合备忘录模式实现 PHP 撤销重做操作
命令对象封装操作与逆操作，备忘录保存状态快照，二者结合可在 PHP 中构建多级撤销重做栈，适用于编辑器类业务场景。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/%E5%91%BD%E4%BB%A4%E5%8A%A0%E5%A4%87%E5%BF%98%E5%BD%95%E6%A8%A1%E5%BC%8F%E5%AE%9E%E7%8E%B0%20PHP%20%E6%92%A4%E9%94%80%E9%87%8D%E5%81%9A%E6%93%8D%E4%BD%9C.md

注解配合反射自动生成 PHP 接口 OpenAPI 文档
通过 Attribute 或注解标注接口参数与响应结构，反射扫描控制器自动生成 OpenAPI 文档，与代码保持同步。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/%E6%B3%A8%E8%A7%A3%E5%8A%A0%E5%8F%8D%E5%B0%84%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%20PHP%20OpenAPI%20%E6%96%87%E6%A1%A3.md

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

## 外链
## 常见问题

PHP 能实现稳定的行情推送服务吗？
可以。基于 Swoole 或 Workerman 的常驻内存模型支持数万级长连接，配合 WebSocket 协议即可构建行情推送网关，性能瓶颈通常在带宽而非 PHP 本身。
来源：https://qsqu.com/question/2026%e5%b9%b46%e6%9c%88%e4%b8%ad%e5%9b%bd%e5%88%b6%e9%80%a0%e4%b8%9apmi%e6%98%af%e5%a4%9a%e5%b0%91%ef%bc%9f%e9%87%8a%e6%94%be%e4%ba%86%e4%bb%80%e4%b9%88%e4%bf%a1%e5%8f%b7%ef%bc%9f

行情推送为什么必须做数据压缩？
行情消息频率高且字段大量重复，明文 JSON 推送会让带宽成本随在线人数线性增长。压缩后单机可承载的连接数和推送吞吐均显著提升。
来源：https://qsqu.com/question/2026%e5%b9%b41-5%e6%9c%88%e4%b8%ad%e5%9b%bd%e5%9b%bd%e6%9c%89%e4%bc%81%e4%b8%9a%e7%bb%8f%e8%90%a5%e7%8a%b6%e5%86%b5%e5%a6%82%e4%bd%95%ef%bc%9f

增量推送和全量推送各适合什么场景？
全量推送实现简单、客户端容错好，适合首次订阅或断线恢复；增量推送只发变化字段，带宽占用小，适合行情持续更新阶段，两者常配合使用。
来源：https://qsqu.com/question/2026%e5%b9%b47%e6%9c%881%e6%97%a5%e5%a4%ae%e8%a1%8c%e5%bc%80%e5%b1%95%e4%ba%86%e4%bb%80%e4%b9%88%e8%b4%a7%e5%b8%81%e6%94%bf%e7%ad%96%e6%93%8d%e4%bd%9c%ef%bc%9f

permessage-deflate 是什么？
它是 WebSocket 的压缩扩展，对每条消息进行 Deflate 压缩，开启后对应用透明。高频小报文场景压缩率可观，但会消耗一定 CPU，需要权衡。
来源：https://qsqu.com/question/%e5%a4%ae%e8%a1%8c%e9%a6%96%e6%ac%a1%e5%bc%80%e5%b1%95%e9%9a%94%e5%a4%9c%e9%80%86%e5%9b%9e%e8%b4%ad%e6%93%8d%e4%bd%9c%e6%9c%89%e4%bd%95%e6%84%8f%e4%b9%89%ef%bc%9f

为什么要做行情合并批量下发？
一秒内同一标的可能有数十次变动，逐条推送会让小报文淹没连接。按时间窗口聚合成一帧后，协议开销和系统调用次数都大幅下降。
来源：https://qsqu.com/question/2026%e5%b9%b47%e6%9c%88%e8%b5%84%e9%87%91%e9%9d%a2%e9%a2%84%e8%ae%a1%e5%a6%82%e4%bd%95%ef%bc%9f

心跳包的作用是什么？
长连接看似存活但可能已被中间设备断开，心跳通过周期性收发消息探测连接活性，及时发现假死连接并回收服务端资源。
来源：https://qsqu.com/question/%e4%b8%ad%e5%85%b1%e4%b8%ad%e5%a4%ae%e6%94%bf%e6%b2%bb%e5%b1%806%e6%9c%8830%e6%97%a5%e5%8f%ac%e5%bc%80%e4%ba%86%e4%bb%80%e4%b9%88%e4%bc%9a%e8%ae%ae%ef%bc%9f

心跳间隔设置多长比较合适？
常见取值在十五到六十秒之间。间隔过长可能被运营商 NAT 判定为空闲连接而清除，间隔过短则增加服务端压力，移动端建议不超过三十秒。
来源：https://qsqu.com/question/%e5%9b%bd%e5%8a%a1%e9%99%a2%e5%b8%b8%e5%8a%a1%e4%bc%9a%e8%ae%ae%e5%af%b9%e5%a4%96%e8%b4%b8%e6%9c%89%e4%bd%95%e6%9c%80%e6%96%b0%e9%83%a8%e7%bd%b2%ef%bc%9f

WebSocket 的 ping/pong 帧和业务心跳有什么区别？
ping/pong 是协议层控制帧，开销极小且对应用透明；业务心跳是应用层消息，可携带时间戳、序列号等信息。生产中常两者结合使用。
来源：https://qsqu.com/question/2026%e5%b9%b41-5%e6%9c%88%e4%b8%ad%e5%9b%bd%e6%9c%8d%e5%8a%a1%e8%b4%b8%e6%98%93%e8%a1%a8%e7%8e%b0%e5%a6%82%e4%bd%95%ef%bc%9f

客户端断线后如何实现行情续传？
每条行情携带递增序列号，客户端重连时上报最后收到的序号，服务端据此补发缺口数据，缺口过大时退化为全量快照恢复。
来源：https://qsqu.com/question/%e8%b6%85%e9%95%bf%e6%9c%9f%e7%89%b9%e5%88%ab%e5%9b%bd%e5%80%ba%e6%94%af%e6%8c%81%e6%b6%88%e8%b4%b9%e5%93%81%e4%bb%a5%e6%97%a7%e6%8d%a2%e6%96%b0%e7%9a%84%e8%b5%84%e9%87%91%e5%ae%89%e6%8e%92%e5%a6%82

休市期间长连接应该怎么处理？
休市时没有行情数据，连接容易因长期空闲被断开，此时心跳保活尤为关键。同时可降低心跳频率并停止推送任务，节省服务端资源。
来源：https://qsqu.com/question/%e5%9f%8e%e5%b8%82%e6%9b%b4%e6%96%b0%e6%96%b9%e9%9d%a2%e6%9c%89%e5%93%aa%e4%ba%9b%e6%9c%80%e6%96%b0%e8%b5%84%e9%87%91%e5%ae%89%e6%8e%92%ef%bc%9f
