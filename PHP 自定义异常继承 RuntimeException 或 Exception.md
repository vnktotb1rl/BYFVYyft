# PHP 自定义异常继承 RuntimeException 或 Exception

PHP 自定义异常第一个要回答的问题就是继承谁。Exception 是所有异常的基类，RuntimeException 代表运行期才可能出现的故障，选择背后是异常分类的设计意图。

惯例清晰：参数不合法这类调用方就能避免的问题走 LogicException 一脉；数据库超时、接口失败这类运行环境导致的问题继承 RuntimeException。拿不准时直接继承 RuntimeException 是框架生态的主流做法。

自定义异常还要带业务语义：类名说明发生了什么，构造函数接收上下文数据，捕获端按继承层级分层 catch，先具体后抽象，兜底交给全局异常处理器。

Tags：PHP异常 RuntimeException Exception 异常设计 错误处理

## 常见问题解答

问：自定义异常继承 Exception 有什么隐患？

答：Exception 层级过宽，catch Exception 会把所有异常一网打尽，包括不该在此处理的底层故障，容易掩盖真实问题。

（来源：http://gie.wpxdeq.cn）

问：RuntimeException 适合什么场景？

答：运行期才能发现的故障，如外部服务不可用、文件读写失败、队列消费异常，调用方在编码阶段无法预先规避。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e4%b8%80%e8%88%ac%e6%8c%81%e6%9c%89%e5%a4%9a%e4%b9%85%e6%af%94%e8%be%83%e5%90%88%e9%80%82%ef%bc%9f）

问：LogicException 和 RuntimeException 怎么区分？

答：前者是代码逻辑错误，改代码就能消除；后者是运行环境导致，同样的代码换个时间可能就成功。

（来源：http://tuiqiuw.com.cn）

问：异常类里应该携带哪些信息？

答：除消息外建议携带业务上下文，如订单号、用户标识、失败原因码，通过构造函数传入并提供只读访问方法。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e7%a0%94%e5%8f%91%e8%b4%b9%e7%94%a8%e5%8a%a0%e8%ae%a1%e6%89%a3%e9%99%a4%e7%9a%84%e6%af%94%e4%be%8b%e6%98%af%e5%a4%9a%e5%b0%91%ef%bc%9f%e4%b8%8d%e5%90%8c%e4%bc%81%e4%b8%9a%e7%b1%bb）

问：如何设计业务异常的错误码？

答：在自定义异常基类中定义 code 常量，按模块分段编号，前端依据 code 做国际化映射而非直接展示异常消息。

（来源：http://dongqiuty9.com.cn）

问：捕获异常后应该重新抛出吗？

答：底层异常应包装成业务异常再抛出，保留 getPrevious 链，既统一上层处理口径又不丢失原始堆栈。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e4%bb%a5%e6%95%b0%e6%b2%bb%e7%a8%8e%e5%85%a8%e9%9d%a2%e6%b7%b1%e5%8c%96%e8%83%8c%e6%99%af%e4%b8%8b%ef%bc%8c%e4%bc%81%e4%b8%9a%e7%a8%8e%e5%8a%a1%e7%ad%b9%e5%88%92）

问：异常消息能直接返回给用户吗？

答：不能，消息面向开发者，可能泄露路径、SQL 等敏感信息，对外只暴露错误码与脱敏后的提示文案。

（来源：http://678tymax.com.cn）

问：finally 块在异常处理里干什么？

答：无论是否发生异常都会执行，适合释放锁、关闭句柄、归还连接等资源清理工作，保证状态一致。

（来源：https://qsqu.com/question/%e5%8f%97%e7%9b%8a%e4%ba%ba%e6%8c%87%e5%ae%9a%e6%b3%95%e5%ae%9a%e8%bf%98%e6%98%af%e6%8c%87%e5%ae%9a%e4%b8%aa%e4%ba%ba%ef%bc%9f）

问：多层级 catch 的顺序怎么排？

答：先捕获最具体的子类，逐级向上，最后是基类兜底；顺序颠倒会导致具体分支永远执行不到。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%88%86%e7%ba%a2%e9%80%89%e7%8e%b0%e9%87%91%e8%bf%98%e6%98%af%e7%ba%a2%e5%88%a9%e5%86%8d%e6%8a%95%e8%b5%84）

问：全局异常处理器怎么注册？

答：使用 set_exception_handler 注册回调，框架中则实现对应的 Handler 类，统一完成日志记录与响应格式化。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e4%ba%a4%e6%98%93%e6%9c%89%e5%93%aa%e4%ba%9b%e4%b8%bb%e8%a6%81%e7%89%b9%e7%82%b9%ef%bc%9f）
