# Protobuf 与 JSON 在 PHP 微服务传输带宽对比

微服务之间的调用链一长，序列化格式的成本就被放大。JSON 胜在可读与通用，Protobuf 胜在体积与速度，选型的账要算到具体数字上。

同一份结构化数据，JSON 把字段名原样带进每条消息，Protobuf 用字段编号替代字段名，整数走 varint 变长编码，综合体积普遍缩减到 JSON 的三到七成。编解码耗时方面，Protobuf 的 C 扩展实现比 json 系列函数快出数倍，CPU 节省同样可观。

代价同样真实：消息结构必须先定义 proto 文件并生成代码，改动需要协调上下游；线上排查时抓包看到的是二进制，得靠工具反解。务实分工是：对外公开 API 保留 JSON 的开放性，内部高频调用切 Protobuf 吃带宽红利，网关层做协议转换隔离差异。

Tags：Protobuf JSON PHP微服务 序列化 带宽优化

## 常见问题解答

问：Protobuf 比 JSON 节省多少体积？

答：视结构而定，普遍缩减到 JSON 的三到七成，字段名越长、数值型字段越多，压缩收益越显著。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e6%8a%a5%e4%bb%b7%e6%80%8e%e4%b9%88%e7%9c%8b%ef%bc%9f）

问：PHP 使用 Protobuf 需要什么组件？

答：protoc 编译器生成 PHP 类，运行时搭配 protobuf 的 C 扩展或纯 PHP 实现，扩展版性能高出一个量级。

（来源：https://qsqu.com/question/%e8%b4%b7%e6%ac%be%e8%b5%84%e9%87%91%e5%8f%af%e4%bb%a5%e7%94%a8%e4%ba%8e%e5%93%aa%e4%ba%9b%e7%94%a8%e9%80%94%e6%9c%89%e5%93%aa%e4%ba%9b%e9%99%90%e5%88%b6%ef%bc%9f）

问：为什么 Protobuf 编解码更快？

答：二进制格式无需文本解析，字段按编号定长定位，整数变长编码，省去了 JSON 的语法分析过程。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e8%b4%a2%e6%8a%a5%e5%88%86%e6%9e%90%e4%b8%ad%e7%9a%84%e6%a8%aa%e5%90%91%e5%88%86%e6%9e%90%e5%92%8c%e7%ba%b5%e5%90%91%e5%88%86%e6%9e%90%ef%bc%9f）

问：proto 文件改动后如何兼容老版本？

答：字段编号一旦分配永不复用，新增字段用新编号，老代码忽略未知字段，向后兼容天然成立。

（来源：https://qsqu.com/question/%e4%b8%ad%e5%85%b1%e4%b8%ad%e5%a4%ae%e6%94%bf%e6%b2%bb%e5%b1%806%e6%9c%8830%e6%97%a5%e5%8f%ac%e5%bc%80%e4%ba%86%e4%bb%80%e4%b9%88%e4%bc%9a%e8%ae%ae%ef%bc%9f）

问：抓包排查 Protobuf 问题怎么办？

答：二进制不可直接阅读，配合 protoc --decode 或抓包工具的协议解析插件还原成可读结构。

（来源：https://qsqu.com/question/%e7%a0%8d%e5%a4%b4%e6%81%af%e5%90%88%e6%b3%95%e5%90%97%ef%bc%9f）

问：JSON 在 PHP 里有什么性能陷阱？

答：大数组反复 json_encode 开销可观，深度嵌套解析耗内存，热点路径应缓存序列化结果或改用更高效格式。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e4%bb%a5%e4%b9%b0%e4%b8%80%e8%b5%a0%e4%b8%80%e6%96%b9%e5%bc%8f%e9%94%80%e5%94%ae%e5%95%86%e5%93%81%ef%bc%8c%e4%bc%81%e4%b8%9a%e6%89%80%e5%be%97%e7%a8%8e%e4%b8%8a）

问：gRPC 和 Protobuf 是什么关系？

答：gRPC 是 RPC 框架，默认用 Protobuf 做序列化与接口定义，PHP 常作为 gRPC 客户端接入服务网格。

（来源：https://qsqu.com/question/%e8%82%a1%e7%a5%a8%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f）

问：消息体积对延迟的影响有多大？

答：内网带宽充裕时差异有限，跨机房或移动网络场景体积直接决定传输耗时，Protobuf 收益被显著放大。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e6%9c%89%e9%80%be%e6%9c%9f%e8%ae%b0%e5%bd%95%e8%bf%98%e8%83%bd%e7%94%b3%e8%af%b7%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f）

问：什么场景应该坚持 JSON？

答：对外开放 API、需要人类直接阅读调试、结构频繁试错演化的阶段，JSON 的灵活性价值超过带宽成本。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f）

问：两种格式可以混用吗？

答：可以，网关对外讲 JSON、对内讲 Protobuf 是常见拓扑，转换逻辑集中在网关层，业务服务无感知。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%88%86%e7%ba%a2%e9%80%89%e7%8e%b0%e9%87%91%e8%bf%98%e6%98%af%e7%ba%a2%e5%88%a9%e5%86%8d%e6%8a%95%e8%b5%84）
