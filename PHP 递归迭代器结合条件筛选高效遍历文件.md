# PHP 递归迭代器结合条件筛选高效遍历文件

处理包含数万张图片或日志的目录时，递归函数一次性把路径塞进数组的写法既占内存又难维护。PHP 标准库提供的 RecursiveDirectoryIterator 配合 RecursiveIteratorIterator，可以以惰性方式逐层展开目录树，每次只产出当前一个文件节点，内存占用保持恒定，遍历百万级文件也不会触顶 memory_limit。

真正的灵活性来自 FilterIterator。继承该类并实现 accept 方法，可以把扩展名、文件大小、修改时间等条件组合成一条过滤管道，套在递归迭代器外层之后，业务循环里拿到的就是已经符合条件的文件。CallbackFilterIterator 进一步允许直接传入闭包，免去手写子类的繁琐，适合一次性的筛选逻辑。

工程实践中还需注意两个细节：RecursiveDirectoryIterator 默认会带出点目录，应通过 SKIP_DOTS 标志跳过；符号链接可能形成循环引用，需要关闭 FOLLOW_SYMLINKS 或自行记录已访问路径。配合生成器把筛选结果包装成惰性序列，整条遍历链路从读取到消费都保持流式，是构建命令行清理工具与扫描器的可靠范式。

Tags：PHP 迭代器 SPL 文件遍历 递归过滤

## 内链

## 快讯

PHP IMAP 读取附件需解析嵌套 MIME 结构
imap_fetchstructure 返回邮件的 MIME 树形结构，附件常嵌套在 multipart/mixed 之下，需递归遍历并按编码方式解码内容。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20IMAP%20%E8%AF%BB%E5%8F%96%E9%82%AE%E4%BB%B6%E9%99%84%E4%BB%B6%E8%A7%A3%E6%9E%90%E5%B5%8C%E5%A5%97%20MIME%20%E7%BB%93%E6%9E%84.md

PHP LDAP 用户同步与组权限实时更新方案
通过 ldap_search 拉取目录服务中的用户与组条目，与本地记录比对后增量更新，可让权限数据与企业组织架构保持一致。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20LDAP%20%E7%94%A8%E6%88%B7%E5%90%8C%E6%AD%A5%E4%B8%8E%E7%BB%84%E6%9D%83%E9%99%90%E5%AE%9E%E6%97%B6%E6%9B%B4%E6%96%B0.md

PHP Memcached 一致性哈希故障转移配置要点
客户端开启一致性哈希后，节点增减只影响少量键的分布，再配合重试次数与故障转移参数，可降低单点宕机的冲击面。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20Memcached%20%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E6%95%85%E9%9A%9C%E8%BD%AC%E7%A7%BB%E9%85%8D%E7%BD%AE.md

PHP Redis 分布式锁用 Lua 脚本原子释放
释放锁时先比对唯一标识再删除键，两步操作须保证原子性，Lua 脚本在 Redis 内原子执行，可避免误删其他客户端的锁。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20Redis%20%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%20Lua%20%E8%84%9A%E6%9C%AC%E5%8E%9F%E5%AD%90%E9%87%8A%E6%94%BE%E7%A4%BA%E4%BE%8B.md

UUID 有序与随机版本对 InnoDB 索引的影响
随机 UUID 作主键会造成 InnoDB 聚簇索引频繁页分裂，有序 UUID 或 ULID 让插入集中在索引尾部，写入性能更平稳。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20UUID%20%E6%9C%89%E5%BA%8F%E4%B8%8E%E9%9A%8F%E6%9C%BA%E7%89%88%E6%9C%AC%E5%AF%B9%20InnoDB%20%E7%B4%A2%E5%BC%95%E5%BD%B1%E5%93%8D.md

PHP 多字节正则中 p{L} 与 w 匹配差异解析
开启 u 修饰符后 \w 可匹配 Unicode 字母与数字，\p{L} 仅匹配各语言字母，处理多语言文本时两者范围差异需注意。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E5%A4%9A%E5%AD%97%E8%8A%82%E6%AD%A3%E5%88%99%20p%7BL%7D%20%E4%B8%8E%20w%20%E5%8C%B9%E9%85%8D%E5%B7%AE%E5%BC%82.md

PHP 多级环境变量覆盖策略与缓存刷新机制
开发、测试、生产配置按优先级逐级覆盖，修改环境变量后需刷新配置缓存，否则旧值继续生效容易造成排查困难。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E5%A4%9A%E7%BA%A7%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E8%A6%86%E7%9B%96%E7%AD%96%E7%95%A5%E4%B8%8E%E7%BC%93%E5%AD%98%E5%88%B7%E6%96%B0.md

PHP 属性钩子 get set 注入验证与变更日志
PHP 8.4 属性钩子允许在 get 与 set 访问时执行自定义逻辑，可在赋值时完成数据校验并记录变更日志，精简样板代码。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E5%B1%9E%E6%80%A7%E9%92%A9%E5%AD%90%20get_set%20%E6%B3%A8%E5%85%A5%E9%AA%8C%E8%AF%81%E4%B8%8E%E5%8F%98%E6%9B%B4%E6%97%A5%E5%BF%97.md

慢查询日志定位与执行计划索引类型解读
开启 MySQL 慢查询日志捕获超时语句，再用 EXPLAIN 查看执行计划，type 列从 ALL 到 const 反映索引利用程度。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E6%85%A2%E6%9F%A5%E8%AF%A2%E6%97%A5%E5%BF%97%E5%AE%9A%E4%BD%8D%E4%B8%8E%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E7%B4%A2%E5%BC%95%E7%B1%BB%E5%9E%8B%E8%A7%A3%E8%AF%BB.md

PHP 数组键整数字符串隐式转换规则与预防
数字形式的字符串键会被 PHP 隐式转为整数，含前导零或小数点则保持字符串，混用键类型时遍历结果可能出乎意料。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E6%95%B0%E7%BB%84%E9%94%AE%E6%95%B4%E6%95%B0%E5%AD%97%E7%AC%A6%E4%B8%B2%E9%9A%90%E5%BC%8F%E8%BD%AC%E6%8D%A2%E8%A7%84%E5%88%99%E5%8F%8A%E9%A2%84%E9%98%B2.md

PHP 死信队列失败任务手动重跑后台实现
多次重试仍失败的任务转入死信队列，后台提供按条件筛选与手动重跑入口，重放前需先确认任务的幂等性。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E6%AD%BB%E4%BF%A1%E9%98%9F%E5%88%97%E5%A4%B1%E8%B4%A5%E4%BB%BB%E5%8A%A1%E6%89%8B%E5%8A%A8%E9%87%8D%E8%B7%91%E5%90%8E%E5%8F%B0%E5%AE%9E%E7%8E%B0.md

PHP 用 endroid_qr-code 生成二维码与条形码
endroid/qr-code 基于 GD 或 Imagick 渲染二维码，支持容错级别、尺寸与 logo 嵌入设置，可输出 PNG 与 SVG 格式。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E7%94%A8%20endroid_qr-code%20%E7%94%9F%E6%88%90%E4%BA%8C%E7%BB%B4%E7%A0%81%E4%B8%8E%E6%9D%A1%E5%BD%A2%E7%A0%81.md

自定义异常继承 RuntimeException 还是 Exception
运行期才能发现的问题适合继承 RuntimeException，可预见的业务错误继承 Exception，语义区分有利于分类捕获处理。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E8%87%AA%E5%AE%9A%E4%B9%89%E5%BC%82%E5%B8%B8%E7%BB%A7%E6%89%BF%20RuntimeException%20%E6%88%96%20Exception.md

PHP 自定义错误页面并记录完整堆栈追踪
set_exception_handler 接管未捕获异常，向用户展示友好错误页面的同时，将堆栈与请求上下文写入日志便于回溯。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E8%87%AA%E5%AE%9A%E4%B9%89%E9%94%99%E8%AF%AF%E9%A1%B5%E9%9D%A2%E5%B9%B6%E8%AE%B0%E5%BD%95%E5%AE%8C%E6%95%B4%E5%A0%86%E6%A0%88%E8%BF%BD%E8%B8%AA.md

PHP 装饰器模式扩展功能不违背单一职责
装饰器模式把缓存、日志、重试等横切逻辑包裹在原对象外层，原类保持单一职责，行为可按需灵活组合叠加。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E8%A3%85%E9%A5%B0%E5%99%A8%E6%A8%A1%E5%BC%8F%E6%89%A9%E5%B1%95%E5%8A%9F%E8%83%BD%E4%B8%8D%E8%BF%9D%E8%83%8C%E5%8D%95%E4%B8%80%E8%81%8C%E8%B4%A3.md

PHP 迭代器管道实现 filter map reduce 延迟计算
基于生成器串联 filter 与 map 操作，数据逐项流过管道而不产生中间数组，处理大数据集时内存占用保持恒定。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP%20%E8%BF%AD%E4%BB%A3%E5%99%A8%E7%AE%A1%E9%81%93%E5%AE%9E%E7%8E%B0%20filter_map_reduce%20%E5%BB%B6%E8%BF%9F%E8%AE%A1%E7%AE%97.md

PHP-FPM 与 Swoole 连接池设计方案差异分析
PHP-FPM 请求结束后资源即释放，连接难以跨请求复用；Swoole 常驻内存，可在协程间维护长连接池提升吞吐。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/PHP-FPM%20%E4%B8%8E%20Swoole%20%E8%BF%9E%E6%8E%A5%E6%B1%A0%E8%AE%BE%E8%AE%A1%E6%96%B9%E6%A1%88%E5%B7%AE%E5%BC%82.md

Protobuf 与 JSON 在微服务传输中的带宽对比
Protobuf 采用二进制编码且无字段名开销，相同数据结构体积小于 JSON，高并发微服务间传输可显著节省带宽。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/Protobuf%20%E4%B8%8E%20JSON%20%E5%9C%A8%20PHP%20%E5%BE%AE%E6%9C%8D%E5%8A%A1%E4%BC%A0%E8%BE%93%E5%B8%A6%E5%AE%BD%E5%AF%B9%E6%AF%94.md

Valitron 链式校验与自定义错误消息编写方法
Valitron 通过链式 rule 声明字段校验规则，支持自定义错误消息与多语言包，轻量无依赖适合小型项目输入校验。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/Valitron%20%E9%93%BE%E5%BC%8F%E6%A0%A1%E9%AA%8C%E4%B8%8E%E8%87%AA%E5%AE%9A%E4%B9%89%E9%94%99%E8%AF%AF%E6%B6%88%E6%81%AF%E7%BC%96%E5%86%99.md

WebSocket 压缩帧与消息合并降低推送带宽
开启 permessage-deflate 扩展可压缩 WebSocket 帧内容，配合短窗口内消息合并推送，能明显降低高频推送带宽占用。
来源：https://github.com/vnktotb1rl/BYFVYyft/blob/main/WebSocket%20%E5%8E%8B%E7%BC%A9%E5%B8%A7%E4%B8%8E%E6%B6%88%E6%81%AF%E5%90%88%E5%B9%B6%E9%99%8D%E4%BD%8E%20PHP%20%E6%8E%A8%E9%80%81%E5%B8%A6%E5%AE%BD.md

## 外链

## 常见问题

RecursiveIteratorIterator 的三个遍历模式有什么区别？
LEAVES_ONLY 只产出叶子节点即文件，SELF_FIRST 先产出父级再产出子级，CHILD_FIRST 则相反。遍历目录做删除操作时常用 CHILD_FIRST，保证先删子文件再删空目录。
来源：https://qsqu.com/question/2026%e5%b9%b4%e5%85%a8%e5%b9%b4%e4%b8%80%e6%ac%a1%e6%80%a7%e5%a5%96%e9%87%91%e5%a6%82%e4%bd%95%e8%ae%a1%e7%a8%8e%ef%bc%9f%e5%8d%95%e7%8b%ac%e8%ae%a1%e7%a8%8e%e5%92%8c%e5%b9%b6%e5%85%a5%e7%bb%bc

为什么遍历时会出现 UnexpectedValueException？
目录在遍历过程中被删除或权限不足时会抛出该异常。可以在 RecursiveDirectoryIterator 构造时传入 CATCH_GET_CHILD 标志忽略无权限的子目录，或者用 try-catch 包裹迭代过程。
来源：https://qsqu.com/question/2026%e5%b9%b47%e6%9c%881%e6%97%a5%e8%b5%b7%ef%bc%8c%e4%b8%89%e6%b5%81%e5%90%88%e4%b8%80%e5%8f%8d%e5%90%91%e5%bc%80%e7%a5%a8%e4%b8%aa%e4%ba%ba%e6%89%80%e5%be%97%e7%a8%8e%e9%a2%84

如何只筛选指定扩展名的文件？
继承 FilterIterator 并在 accept 中判断 getExtension 返回值即可，更简洁的方式是使用 RegexIterator 对路径做正则匹配，或者直接使用 GlobIterator 处理单层目录的通配符匹配。
来源：https://qsqu.com/question/2026%e5%b9%b4%e4%bb%a5%e6%95%b0%e6%b2%bb%e7%a8%8e%e5%85%a8%e9%9d%a2%e6%b7%b1%e5%8c%96%e8%83%8c%e6%99%af%e4%b8%8b%ef%bc%8c%e4%bc%81%e4%b8%9a%e7%a8%8e%e5%8a%a1%e7%ad%b9%e5%88%92

迭代器遍历和 glob 函数相比有什么优势？
glob 会一次性返回全部匹配结果，目录巨大时内存暴涨；迭代器逐个产出路径，内存恒定。迭代器还能获取 SplFileInfo 对象，直接读取大小、权限、修改时间等元信息。
来源：https://qsqu.com/question/%e8%82%a1%e7%a5%a8%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f

如何避免遍历符号链接造成的死循环？
不要传入 FOLLOW_SYMLINKS 标志，默认配置下符号链接不会被展开。若业务必须跟随链接，应在遍历时记录已访问的真实路径，遇到重复路径主动跳过。
来源：https://qsqu.com/question/a%e8%82%a1%e7%9a%84%e4%ba%a4%e6%98%93%e6%97%b6%e9%97%b4%e6%98%af%e5%a6%82%e4%bd%95%e5%ae%89%e6%8e%92%e7%9a%84%ef%bc%9f

多个筛选条件如何组合使用？
FilterIterator 支持层层嵌套，把文件大小过滤器包在扩展名过滤器外面即可。条件较多时建议改用 CallbackFilterIterator 传闭包，把多个判断写在同一个 accept 逻辑里，层次更清晰。
来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%aft1%e4%ba%a4%e6%98%93%e5%88%b6%e5%ba%a6%ef%bc%9f

遍历时如何跳过特定目录如 vendor？
在自定义 FilterIterator 的 accept 方法里判断当前节点是否为目录且名称在排除列表中，返回 false 即可阻止递归进入该目录，过滤对文件和目录同样生效。
来源：https://qsqu.com/question/%e8%82%a1%e7%a5%a8%e7%9a%84%e6%b6%a8%e8%b7%8c%e5%b9%85%e9%99%90%e5%88%b6%e6%98%af%e5%a4%9a%e5%b0%91%ef%bc%9f

SKIP_DOTS 标志是必须加的吗？
强烈建议加上。不加时迭代器会产出每个目录下的点与双点条目，若业务代码未做判断，可能在拼接路径或递归删除时引发意外行为甚至安全风险。
来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%b8%82%e7%9b%88%e7%8e%87%ef%bc%88pe%ef%bc%89%ef%bc%9f

迭代器能和生成器结合使用吗？
可以。把迭代器包在生成器函数里用 yield 逐条产出，再叠加业务层二次过滤，整条链路保持惰性求值。处理结果无需全部载入内存，适合导出清单或批量迁移场景。
来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%b8%82%e5%87%80%e7%8e%87%ef%bc%88pb%ef%bc%89%ef%bc%9f

遍历超大目录时如何统计进度？
迭代器本身无法预知总数，常见的做法是先单独跑一次统计文件数量的遍历，或者用 du、find 命令预估规模，再在主遍历时按已处理数量计算百分比输出进度。
来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e8%82%a1%e6%81%af%e7%8e%87%ef%bc%9f%e5%ae%83%e4%b8%8e%e5%88%86%e7%ba%a2%e7%8e%87%e6%9c%89%e4%bd%95%e5%8c%ba%e5%88%ab%ef%bc%9f
