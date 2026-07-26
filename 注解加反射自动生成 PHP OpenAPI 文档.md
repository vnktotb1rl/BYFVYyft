# 注解加反射自动生成 PHP OpenAPI 文档

手写 OpenAPI 文档最大的痛点是代码改了文档忘改，时间一长接口文档与实际行为脱节。利用 PHP 8 原生注解配合反射，可以让文档直接从代码里长出来。

实现分三层：先用 Attribute 定义 Route、Param 等注解类；再写扫描器遍历控制器，通过 ReflectionMethod 的 getAttributes 读取注解，汇总成中间数组；最后渲染成 OpenAPI 规范的 JSON。

参数类型可从方法签名推导，标量映射 string、integer，对象类型继续反射生成 schema。生成动作做成命令行脚本接入 CI，每次合并自动产出最新 openapi.json。

Tags：PHP注解 反射 OpenAPI 接口文档 自动化

## 常见问题解答

问：PHP 8 注解和旧的 docblock 注释有什么区别？

答：注解是语言级特性，写在方法上方并能通过反射 API 直接读取为对象实例；docblock 只是文本注释，解析依赖正则或第三方库。

（来源：https://qsqu.com/question/%e6%99%ba%e8%83%bd%e6%8a%95%e9%a1%be%e9%80%82%e5%90%88%e6%99%ae%e9%80%9a%e6%8a%95%e8%b5%84%e8%80%85%e5%90%97%ef%bc%9f）

问：如何读取方法上的注解？

答：先通过 ReflectionClass 的 getMethod 拿到 ReflectionMethod，再调用 getAttributes 传入注解类名，返回数组后用 newInstance 实例化。

（来源：https://qsqu.com/question/%e8%af%81%e7%9b%91%e4%bc%9a%e5%af%b9%e8%b5%84%e6%9c%ac%e5%b8%82%e5%9c%ba%e8%b4%a2%e5%8a%a1%e9%80%a0%e5%81%87%e6%9c%89%e4%bd%95%e6%9c%80%e6%96%b0%e9%83%a8%e7%bd%b2%ef%bc%9f）

问：扫描控制器目录性能会不会很差？

答：开发环境实时扫描无所谓，生产环境应把生成结果缓存为静态 JSON 文件，只在部署时重新生成，运行时零开销。

（来源：https://qsqu.com/question/%e8%b4%a2%e6%94%bf%e8%b5%a4%e5%ad%97%e6%98%af%e4%bb%80%e4%b9%88%e6%84%8f%e6%80%9d%ef%bc%9f%e8%b5%a4%e5%ad%97%e9%ab%98%e4%ba%86%e6%9c%89%e4%bb%80%e4%b9%88%e5%90%8e%e6%9e%9c%ef%bc%9f）

问：方法参数类型怎么映射成 OpenAPI 类型？

答：用 ReflectionNamedType 的 getName 获取类型名，int 映射 integer，float 映射 number，bool 映射 boolean，其余默认 string。

（来源：https://qsqu.com/question/2026%e5%b9%b47%e6%9c%88%e5%88%b8%e5%95%86%e9%87%91%e8%82%a1%e9%85%8d%e7%bd%ae%e6%9c%89%e4%bd%95%e5%8f%98%e5%8c%96%ef%bc%9f）

问：嵌套对象的响应结构如何生成？

答：对返回的 DTO 类继续做反射，遍历其公开属性的类型递归构建 schema，数组类型结合注解中的元素类型声明处理。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%89%93%e6%96%b0%e5%8f%8a%e5%85%b6%e5%b8%82%e5%80%bc%e8%a7%84%e5%88%99%ef%bc%9f）

问：有没有现成的库可以直接用？

答：zircote/swagger-php 是主流方案，支持 PHP 8 注解写法，扫描代码即可输出 OpenAPI 文档，配合 swagger-ui 直接可视化。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e5%88%86%e6%9e%90%e4%bc%81%e4%b8%9a%e7%9a%84%e6%88%90%e9%95%bf%e6%80%a7%ef%bc%8c%e5%93%aa%e4%ba%9b%e6%8c%87%e6%a0%87%e6%9c%80%e5%85%b3%e9%94%ae%ef%bc%9f）

问：注解参数太多显得臃肿怎么办？

答：把通用响应结构抽成可复用的注解组合或基类，业务接口只标注差异部分，保持代码清爽。

（来源：https://qsqu.com/question/%e8%82%a1%e7%a5%a8%e9%80%80%e5%b8%82%e4%ba%86%e6%80%8e%e4%b9%88%e5%8a%9e%ef%bc%9f）

问：生成的 JSON 如何接 Swagger UI？

答：把 openapi.json 放到可访问路径，Swagger UI 初始化时指向该地址即可，Docker 官方镜像一条命令就能起服务。

（来源：https://qsqu.com/question/%e6%99%ba%e8%83%bd%e6%8a%95%e9%a1%be%ef%bc%88robo-advisor%ef%bc%89%e4%b8%8e%e4%bc%a0%e7%bb%9f%e4%ba%ba%e5%b7%a5%e7%90%86%e8%b4%a2%e9%a1%be%e9%97%ae%e6%9c%89%e4%bd%95%e6%9c%ac%e8%b4%a8%e5%8c%ba）

问：CI 里怎么校验文档是否过期？

答：流水线中重新生成文档并与仓库内版本做 diff，出现差异则判定构建失败，强制开发者同步更新。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e4%bb%80%e4%b9%88%e6%97%b6%e9%97%b4%e4%ba%a4%e6%98%93%ef%bc%9f）

问：反射会影响线上运行性能吗？

答：只在构建期执行反射生成文档，线上请求链路完全不经过反射，对运行时性能没有影响。

（来源：https://qsqu.com/question/%e5%b0%8f%e5%be%ae%e4%bc%81%e4%b8%9a%e6%b2%a1%e6%9c%89%e6%8a%b5%e6%8a%bc%e7%89%a9%e8%83%bd%e7%94%b3%e8%af%b7%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f）
