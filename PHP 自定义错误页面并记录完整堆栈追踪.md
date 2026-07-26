# PHP 自定义错误页面并记录完整堆栈追踪

线上白屏与调试页直连生产是新手项目的标志。面向用户的错误页应当友好克制，面向开发的错误信息应当详尽完整，两件事用两套通道分别处理。

错误页按状态码分层：404 引导返回有效内容，500 给出安抚文案与错误编号，绝不暴露路径、SQL 或堆栈。框架中注册异常处理器映射状态码渲染视图；原生项目用 set_exception_handler 加 register_shutdown_function 双保险。

堆栈走日志通道完整保留：getTraceAsString 提供调用链，配合请求地址、参数摘要、时间戳写入结构化日志，敏感字段脱敏。错误编号同时出现在页面与日志，报障时凭编号秒级定位现场。

Tags：PHP错误页面 异常处理 堆栈追踪 日志记录 Monolog

## 常见问题解答

问：为什么不能向用户展示堆栈信息？

答：堆栈包含文件路径、类名甚至查询片段，属于攻击者绘制系统画像的现成素材，对外必须隐藏。

（来源：https://qsqu.com/question/%e7%a0%94%e5%8f%91%e8%b4%b9%e7%94%a8%e8%b5%84%e6%9c%ac%e5%8c%96%e5%92%8c%e8%b4%b9%e7%94%a8%e5%8c%96%e5%af%b9%e8%b4%a2%e6%8a%a5%e6%9c%89%e4%bb%80%e4%b9%88%e4%b8%8d%e5%90%8c%e5%bd%b1%e5%93%8d%ef%bc%9f）

问：set_exception_handler 能捕获所有错误吗？

答：只能捕获未被 catch 的异常，致命错误如内存溢出要靠 register_shutdown_function 配合 error_get_last 兜底。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e5%88%a4%e6%96%ad%e8%b4%b7%e6%ac%be%e4%b8%ad%e4%bb%8b%e6%98%af%e5%90%a6%e9%9d%a0%e8%b0%b1%ef%bc%9f）

问：404 页面除了文案还应该有什么？

答：返回首页与核心栏目的链接、搜索入口，并保证 HTTP 状态码确实是 404，避免搜索引擎误判软 404。

（来源：https://qsqu.com/question/%e6%95%b0%e5%ad%97%e4%ba%ba%e6%b0%91%e5%b8%81%e5%92%8c%e6%94%af%e4%bb%98%e5%ae%9d%e3%80%81%e5%be%ae%e4%bf%a1%e6%94%af%e4%bb%98%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：错误编号有什么作用？

答：页面展示编号、日志记录同一编号，用户反馈问题时凭编号直接检索到完整堆栈，沟通成本骤降。

（来源：https://qsqu.com/question/%e6%9c%89%e4%ba%86%e7%a4%be%e4%bf%9d%e8%bf%98%e8%a6%81%e4%b9%b0%e5%95%86%e4%b8%9a%e4%bf%9d%e9%99%a9%e5%90%97%ef%bc%9f）

问：Monolog 怎么记录异常堆栈？

答：把异常对象放入 context 数组，formatter 会自动展开消息与 trace，生产环境配合 JSON 格式便于采集。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%ae%9a%e6%8a%95%e4%b8%8e%e4%b8%80%e6%ac%a1%e6%80%a7%e4%b9%b0%e5%85%a5%e5%93%aa%e4%b8%aa%e6%94%b6%e7%9b%8a%e6%9b%b4%e9%ab%98）

问：日志里的请求参数要全量记录吗？

答：记录摘要即可，密码、身份证号、令牌等敏感字段必须脱敏或剔除，防止日志成为新的泄露源。

（来源：https://qsqu.com/question/%e7%a7%bb%e5%8a%a8%e6%94%af%e4%bb%98%e8%b4%a6%e6%88%b7%e8%b5%84%e9%87%91%e8%a2%ab%e7%9b%97%ef%bc%8c%e6%8d%9f%e5%a4%b1%e7%94%b1%e8%b0%81%e6%89%bf%e6%8b%85%ef%bc%9f）

问：display_errors 线上应该是什么状态？

答：必须关闭，错误只进日志不进响应；开发环境开启并配合 Whoops 之类的调试页提升排查效率。

（来源：https://qsqu.com/question/%e4%bf%9d%e9%99%a9%e5%90%88%e5%90%8c%e9%87%8c%e7%9a%84%e7%ad%89%e5%be%85%e6%9c%9f%e6%98%af%e4%bb%80%e4%b9%88%e6%84%8f%e6%80%9d%ef%bc%9f）

问：ajax 请求的错误页怎么处理？

答：区分请求类型，ajax 返回 JSON 结构的错误码与提示，页面请求才渲染 HTML 错误视图。

（来源：https://qsqu.com/question/%e7%bb%99%e5%ad%a9%e5%ad%90%e4%b9%b0%e4%bf%9d%e9%99%a9%e9%9c%80%e8%a6%81%e6%b3%a8%e6%84%8f%e4%bb%80%e4%b9%88%ef%bc%9f）

问：通知渠道怎么接入？

答：日志处理器挂载邮件、钉钉或企业微信 Webhook，严重错误实时推送，普通错误聚合后定时汇总。

（来源：https://qsqu.com/question/%e5%ae%b6%e5%ba%ad%e9%85%8d%e7%bd%ae%e4%bf%9d%e9%99%a9%ef%bc%8c%e5%ba%94%e8%af%a5%e4%bc%98%e5%85%88%e7%bb%99%e8%b0%81%e4%b9%b0%ef%bc%9f）

问：错误日志需要定期清理吗？

答：需要，按天滚动切割并设置保留周期，Monolog 的 RotatingFileHandler 开箱即用，防止磁盘被日志占满。

（来源：https://qsqu.com/question/%e9%87%8f%e5%8c%96%e4%ba%a4%e6%98%93%e4%b8%8e%e9%ab%98%e9%a2%91%e4%ba%a4%e6%98%93%ef%bc%88hft%ef%bc%89%e5%9c%a8%e9%87%91%e8%9e%8d%e7%a7%91%e6%8a%80%e7%9b%91%e7%ae%a1%e6%a1%86%e6%9e%b6%e4%b8%8b）
