# JsonSerializable 接口控制 PHP 输出字段格式

API 直接 json_encode 领域对象，输出要么是空对象，要么把内部字段和盘托出。JsonSerializable 接口把序列化的主动权交还给类本身。

实现只有一个方法：类实现 JsonSerializable 并定义 jsonSerialize，返回可序列化数组。金额格式化成两位小数字符串，时间统一 ISO8601，密码哈希、成本价等内部字段直接不出现。json_encode 遇到该接口自动调用，嵌套对象与集合逐层生效。

进阶用法是按场景输出不同视图：给类增加输出档位设定方法，jsonSerialize 按档位组装数组，列表接口走精简版，详情接口走完整版。序列化逻辑内聚在模型里，字段调整只改一处，接口契约稳定性有了结构保障。

Tags：PHP JsonSerializable API开发 JSON序列化 接口设计

## 常见问题解答

问：JsonSerializable 解决了什么问题？

答：让类自己决定 JSON 输出的字段与格式，避免私有属性输出为空对象或敏感字段被意外暴露。

（来源：http://lok.jinxinggs.com）

问：实现接口需要写哪个方法？

答：只需实现 jsonSerialize 方法，返回可序列化的数据结构，json_encode 会自动调用。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e7%94%b3%e8%af%b7%e9%93%b6%e8%a1%8c%e8%b4%b7%e6%ac%be%e9%9c%80%e8%a6%81%e6%bb%a1%e8%b6%b3%e5%93%aa%e4%ba%9b%e5%9f%ba%e6%9c%ac%e6%9d%a1%e4%bb%b6%ef%bc%9f）

问：嵌套对象也会被自动处理吗？

答：会，返回数组里包含的 JsonSerializable 对象在编码时逐层调用各自的方法，集合同样生效。

（来源：https://qsqu.com/question/%e6%88%bf%e8%b4%b7%e5%88%a9%e7%8e%87%e9%80%89lpr%e6%b5%ae%e5%8a%a8%e8%bf%98%e6%98%af%e5%9b%ba%e5%ae%9a%e5%88%a9%e7%8e%87%e6%9b%b4%e5%a5%bd%ef%bc%9f）

问：如何隐藏敏感字段？

答：jsonSerialize 的返回数组里不包含这些字段即可，密码哈希、内部成本等信息天然被隔离。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%85%b6%e4%bb%96%e5%ba%94%e6%94%b6%e6%ac%be%e5%92%8c%e5%85%b6%e4%bb%96%e5%ba%94%e4%bb%98%e6%ac%be%ef%bc%8c%e4%b8%ba%e4%bb%80%e4%b9%88%e5%ae%83%e4%bb%ac%e5%ae%b9）

问：时间字段推荐输出什么格式？

答：ISO8601 是跨语言兼容最好的选择，带时区信息，前端与移动端解析都有现成支持。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e5%88%a4%e6%96%ad%e8%b4%b7%e6%ac%be%e5%88%a9%e7%8e%87%e6%98%af%e5%90%a6%e5%90%88%e7%90%86%ef%bc%9f）

问：金额字段为什么要转成字符串？

答：浮点数在 JSON 传输中存在精度风险，定点字符串由前端按需解析，金额计算链路更安全。

（来源：https://qsqu.com/question/%e4%b8%ba%e4%bb%80%e4%b9%88%e6%9c%89%e5%85%88%e4%bf%9d%e9%9a%9c%e5%90%8e%e7%90%86%e8%b4%a2%e7%9a%84%e8%af%b4%e6%b3%95%ef%bc%9f）

问：列表和详情想输出不同字段怎么办？

答：给类增加输出档位设定方法，jsonSerialize 按档位组装数组，一套模型支撑多种视图。

（来源：https://qsqu.com/question/%e6%84%8f%e5%a4%96%e9%99%a9%e5%93%aa%e4%ba%9b%e6%83%85%e5%86%b5%e4%b8%8d%e8%b5%94%ef%bc%9f）

问：和框架的资源类是什么关系？

答：Laravel Resource 等机制是同一思路的框架化封装，原生 JsonSerializable 更轻量，无框架依赖。

（来源：https://qsqu.com/question/%e7%8e%b0%e5%9c%a8%e7%9a%84%e6%88%bf%e8%b4%b7lpr%e5%88%a9%e7%8e%87%e6%98%af%e5%a4%9a%e5%b0%91%ef%bc%9f）

问：jsonSerialize 里能抛异常吗？

答：可以但应谨慎，序列化阶段的异常往往发生在响应输出途中，前置校验比失败兜底更稳妥。

（来源：https://qsqu.com/question/%e4%b9%b0%e4%bf%9d%e9%99%a9%e5%88%b0%e5%ba%95%e5%9b%be%e4%bb%80%e4%b9%88%ef%bc%9f）

问：接口版本演进时字段怎么兼容？

答：老字段保留并逐步标记废弃，新字段按档位或版本参数控制输出，序列化层集中管理兼容策略。

（来源：https://qsqu.com/question/%e8%b4%b7%e6%ac%be%e8%a2%ab%e6%8b%92%e5%90%8e%e5%a4%9a%e4%b9%85%e5%8f%af%e4%bb%a5%e5%86%8d%e6%ac%a1%e7%94%b3%e8%af%b7%ef%bc%9f）
