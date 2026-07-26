# PHP 慢查询日志定位与执行计划索引类型解读

PHP 接口变慢，瓶颈多数在数据库。MySQL 开启 slow_query_log 并把 long_query_time 设到 0.5 秒，慢 SQL 就会带着耗时与扫描行数落盘，再用 pt-query-digest 按指纹聚合，高负载语句立刻现形。

定位到 SQL 后执行 EXPLAIN，重点看 type、key、rows、Extra 四列。type 从好到差是 const、eq_ref、ref、range、index、ALL，出现 ALL 说明全表扫描必须处理。

Extra 出现 Using filesort 或 Using temporary，说明排序分组没走索引。建联合索引遵循最左前缀，等值条件在前、范围条件在后，多数慢查询迎刃而解。

Tags：PHP 慢查询 MySQL索引 执行计划 性能优化

## 常见问题解答

问：MySQL 慢查询日志怎么开启？

答：在配置文件中设置 slow_query_log=1，指定 slow_query_log_file 路径，并把 long_query_time 调整为业务可接受的阈值，动态修改可用 SET GLOBAL 命令。

（来源：https://qsqu.com/question/%e7%bd%91%e8%b4%b7%e5%92%8c%e9%93%b6%e8%a1%8c%e8%b4%b7%e6%ac%be%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：long_query_time 设多少合适？

答：线上接口类业务建议 0.5 秒起步，分析期可临时设为 0 记录全部 SQL，采集足够样本后再调回，避免日志膨胀。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%aft1%e4%ba%a4%e6%98%93%e5%88%b6%e5%ba%a6%ef%bc%9f）

问：pt-query-digest 是干什么的？

答：这是 Percona Toolkit 中的日志分析工具，能把慢日志按 SQL 指纹聚合，输出总耗时、平均耗时、执行次数等排名，快速锁定高负载语句。

（来源：http://lok.jinxinggs.com）

问：EXPLAIN 的 type 列 ALL 意味着什么？

答：ALL 代表全表扫描，优化器没有找到可用索引，数据量增长后性能会线性恶化，是必须处理的信号。

（来源：https://qsqu.com/question/%e6%8f%90%e5%89%8d%e8%bf%98%e8%b4%b7%e6%98%af%e5%90%a6%e4%b8%80%e5%ae%9a%e5%88%92%e7%ae%97%ef%bc%9f）

问：ref 和 range 有什么区别？

答：ref 表示通过索引做等值匹配，可能命中多行；range 表示索引范围扫描，常见于大于、小于、BETWEEN 条件，两者都属于可接受的访问方式。

（来源：https://qsqu.com/question/%e5%ba%94%e6%94%b6%e8%b4%a6%e6%ac%be%e5%91%a8%e8%bd%ac%e7%8e%87%e9%99%8d%e4%bd%8e%e5%af%b9%e4%bc%81%e4%b8%9a%e6%9c%89%e4%bb%80%e4%b9%88%e5%bd%b1%e5%93%8d%ef%bc%9f）

问：Using filesort 怎么消除？

答：让 ORDER BY 的字段顺序与某个索引的列顺序一致，排序就能借助索引天然有序的特性完成，Extra 中不再出现 filesort。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%ef%bc%9f）

问：联合索引为什么要遵循最左前缀？

答：索引按列定义顺序组织数据，查询条件从第一列开始连续命中才能利用索引，跳过首列会导致后续列失效。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e7%9b%88%e4%ba%8f%e6%80%8e%e4%b9%88%e8%ae%a1%e7%ae%97%ef%bc%9f）

问：rows 列很大但查询很快是怎么回事？

答：rows 是优化器基于统计信息的预估值，可能失真；执行 ANALYZE TABLE 更新统计后重新查看，若仍偏大则考虑索引选择性不足。

（来源：https://qsqu.com/question/%e5%a4%ae%e8%a1%8c%e9%99%8d%e6%81%af%ef%bc%8c%e9%92%b1%e8%a2%8b%e5%ad%90%e4%bc%9a%e5%8f%97%e5%93%aa%e4%ba%9b%e5%bd%b1%e5%93%8d%ef%bc%9f）

问：覆盖索引是什么意思？

答：查询所需的全部列都包含在索引中，无需回表取数据，Extra 会显示 Using index，响应速度明显提升。

（来源：https://qsqu.com/question/%e6%9c%89%e4%ba%86%e5%9f%ba%e6%9c%ac%e5%8c%bb%e7%96%97%e4%bf%9d%e9%99%a9%ef%bc%8c%e8%bf%98%e9%9c%80%e8%a6%81%e4%b9%b0%e5%95%86%e4%b8%9a%e5%8c%bb%e7%96%97%e9%99%a9%e5%90%97%ef%bc%9f）

问：PHP 代码层面能配合做什么？

答：避免在循环中执行 SQL，能用一次 JOIN 或 IN 查询解决的绝不拆成多次请求；同时给 ORM 查询开启日志，便于与慢日志对照定位。

（来源：https://qsqu.com/question/%e4%bf%9d%e8%b4%b9%e8%b1%81%e5%85%8d%e6%9d%a1%e6%ac%be%e6%9c%89%e4%bb%80%e4%b9%88%e7%94%a8%ef%bc%9f）
