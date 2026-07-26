# PHP UUID 有序与随机版本对 InnoDB 索引影响

用 UUID 替代自增 ID 做主键在分布式系统里几乎成了默认选择，但随机 UUID 在 InnoDB 聚簇索引上埋着性能地雷，版本选错表越大写入越痛苦。

InnoDB 主键即数据的物理排序。自增 ID 新行永远追加到 B+ 树末尾；UUID v4 完全随机，插入位置飘忽，页分裂频发，填充率跌到五成，缓冲池装不下，磁盘 IO 随之放大。

解法是让 UUID 有序：v6、v7 把时间戳放在最高有效位，字节序与时间序一致，写入回归尾部追加，ramsey/uuid 已支持生成。存量 v4 迁移不动时，主键保留自增 BIGINT，UUID 降级为对外标识加唯一索引。

Tags：PHP UUID InnoDB 索引优化 MySQL 分布式主键

## 常见问题解答

问：为什么随机 UUID 会伤害 InnoDB 写入性能？

答：聚簇索引按主键物理排序，随机值插入位置分散，页分裂频繁、填充率低，表体积膨胀且缓冲池效率下降。

（来源：https://qsqu.com/question/%e6%8c%87%e6%95%b0%e5%9f%ba%e9%87%91%e5%92%8c%e4%b8%bb%e5%8a%a8%e5%9f%ba%e9%87%91%e5%93%aa%e4%b8%aa%e6%9b%b4%e9%80%82%e5%90%88%e9%95%bf%e6%9c%9f%e6%8c%81%e6%9c%89）

问：UUID v4 的主要问题是什么？

答：完全随机的值导致插入点遍布整棵 B+ 树，每次写入都可能触发页分裂与随机 IO，数据量越大越明显。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e7%9a%84%e6%9d%a0%e6%9d%86%e6%95%88%e5%ba%94%e6%98%af%e6%80%8e%e4%b9%88%e5%9b%9e%e4%ba%8b%ef%bc%9f）

问：UUID v7 为什么对索引友好？

答：最高有效位是毫秒级时间戳，字节序与时间先后一致，新值天然落在索引尾部，写入退化为顺序追加。

（来源：https://qsqu.com/question/%e8%b6%85%e9%95%bf%e6%9c%9f%e7%89%b9%e5%88%ab%e5%9b%bd%e5%80%ba%e6%94%af%e6%8c%81%e6%b6%88%e8%b4%b9%e5%93%81%e4%bb%a5%e6%97%a7%e6%8d%a2%e6%96%b0%e7%9a%84%e8%b5%84%e9%87%91%e5%ae%89%e6%8e%92%e5%a6%82）

问：UUID v1 有序吗？

答：包含时间戳但字段排列经过重排，字节序不随时间单调递增，对索引顺序性的改善有限。

（来源：http://vyk.wpxdeq.cn）

问：PHP 怎么生成 UUID v7？

答：ramsey/uuid 库支持 v6 与 v7，调用对应工厂方法即可，MySQL 8 之后也可在数据库侧生成。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e7%bb%8f%e8%90%a5%e6%b4%bb%e5%8a%a8%e7%8e%b0%e9%87%91%e6%b5%81%e5%87%80%e9%a2%9d-%e5%87%80%e5%88%a9%e6%b6%a6%e6%af%94%e7%8e%87%ef%bc%9f）

问：UUID 主键应该用哪种字段类型存？

答：BINARY(16) 存二进制形式最省空间且比较高效，CHAR(36) 可读性好但体积翻倍，索引开销更大。

（来源：https://qsqu.com/question/%e7%ad%89%e9%a2%9d%e6%9c%ac%e6%81%af%e5%92%8c%e7%ad%89%e9%a2%9d%e6%9c%ac%e9%87%91%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%8c%e5%93%aa%e4%b8%aa%e6%9b%b4%e5%88%92%e7%ae%97%ef%bc%9f）

问：不想改主键方案还有退路吗？

答：主键保留自增 BIGINT 供内部关联，UUID 作为对外业务标识建唯一索引，内外隔离各取所长。

（来源：https://qsqu.com/question/%e4%b8%aa%e4%ba%ba%e4%bf%a1%e7%94%a8%e8%b4%b7%e6%ac%be%e9%a2%9d%e5%ba%a6%e6%98%af%e6%80%8e%e4%b9%88%e6%a0%b8%e5%ae%9a%e7%9a%84%ef%bc%9f）

问：表已经用了 v4 怎么补救？

答：评估定期 OPTIMIZE TABLE 重建索引的收益，新表一律改用 v7，老表在停机窗口做迁移或接受现状。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e3%80%8a%e5%a2%9e%e5%80%bc%e7%a8%8e%e6%b3%95%e3%80%8b%e5%ae%9e%e6%96%bd%e5%90%8e%ef%bc%8c%e5%b0%8f%e8%a7%84%e6%a8%a1%e7%ba%b3%e7%a8%8e%e4%ba%ba%e7%9a%84%e5%a2%9e%e5%80%bc%e7%a8%8e）

问：雪花算法和 UUID v7 怎么选？

答：雪花 ID 更短且趋势递增，但需要发号器协调；v7 标准化程度高、跨语言生态成熟，无中心依赖。

（来源：https://qsqu.com/question/%e9%93%b6%e8%a1%8c%e5%ae%a1%e6%89%b9%e8%b4%b7%e6%ac%be%e4%b8%bb%e8%a6%81%e7%9c%8b%e5%93%aa%e4%ba%9b%e6%96%b9%e9%9d%a2%ef%bc%9f）

问：UUID 会影响 join 性能吗？

答：二进制 16 字节的比较成本高于 8 字节 BIGINT，宽 join 链路多的系统应把这点纳入方案权衡。

（来源：https://qsqu.com/question/%e9%93%b6%e8%a1%8c%e8%b4%b7%e6%ac%be%e5%ae%a1%e6%89%b9%e4%b8%80%e8%88%ac%e9%9c%80%e8%a6%81%e5%a4%9a%e9%95%bf%e6%97%b6%e9%97%b4%ef%bc%9f）
