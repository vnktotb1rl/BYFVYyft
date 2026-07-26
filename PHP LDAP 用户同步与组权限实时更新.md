# PHP LDAP 用户同步与组权限实时更新

企业内部系统接 LDAP 是标准动作，难点不在首次登录的账号认证，而在组织架构变动后，PHP 应用里的用户与权限如何不掉队。

同步策略分推拉两路。拉模式由 PHP 定时任务执行全量比对：ldap_search 分页拉取目录树，按 entryUUID 作为稳定主键与本地用户表对照，新增则建档、属性变化则更新、目录中消失则禁用而非删除。推模式依赖目录服务端的变更通知，事件到达后 PHP 端定点处理，延迟从小时级降到秒级。

组权限的实时性要求更高。用户调岗后旧权限必须当场失效，做法是登录态中不固化权限快照，鉴权时实时查组或对权限缓存设短 TTL。离职账号的处理链路要覆盖两处：LDAP 侧禁用与本地会话强制注销，任何一处遗漏都是安全隐患。

Tags：PHP LDAP 用户同步 权限管理 企业集成 目录服务

## 常见问题解答

问：PHP 连接 LDAP 用什么扩展？

答：php-ldap 扩展提供 ldap_connect、ldap_bind、ldap_search 等完整接口，Active Directory 与 OpenLDAP 均支持。

（来源：https://qsqu.com/question/%e6%99%ba%e8%83%bd%e6%8a%95%e9%a1%be%e9%80%82%e5%90%88%e6%99%ae%e9%80%9a%e6%8a%95%e8%b5%84%e8%80%85%e5%90%97%ef%bc%9f）

问：用户同步用什么字段做稳定主键？

答：目录的 entryUUID 或 objectGUID 终生不变，优于可修改的工号与邮箱，是同步比对的最佳锚点。

（来源：https://qsqu.com/question/%e8%b4%b7%e6%ac%be%e7%94%b3%e8%af%b7%e8%a2%ab%e6%8b%92%e7%9a%84%e5%b8%b8%e8%a7%81%e5%8e%9f%e5%9b%a0%e6%9c%89%e5%93%aa%e4%ba%9b%ef%bc%9f）

问：全量同步频率设多少合适？

答：组织变动不频繁时每小时到每天一次，配合推模式事件通知弥补实时性，避免高频全量拖垮目录服务。

（来源：https://qsqu.com/question/%e7%bb%8f%e8%90%a5%e8%b4%b7%e5%8f%af%e4%bb%a5%e6%8b%bf%e5%8e%bb%e4%b9%b0%e6%88%bf%e5%90%97%ef%bc%8c%e6%9c%89%e4%bb%80%e4%b9%88%e9%a3%8e%e9%99%a9%ef%bc%9f）

问：目录中消失的用户怎么处理？

答：本地标记禁用而非物理删除，保留历史操作记录的关联完整性，复核后再做归档清理。

（来源：https://qsqu.com/question/%e7%a0%94%e5%8f%91%e8%b4%b9%e7%94%a8%e8%b5%84%e6%9c%ac%e5%8c%96%e5%92%8c%e8%b4%b9%e7%94%a8%e5%8c%96%e5%af%b9%e8%b4%a2%e6%8a%a5%e6%9c%89%e4%bb%80%e4%b9%88%e4%b8%8d%e5%90%8c%e5%bd%b1%e5%93%8d%ef%bc%9f）

问：大目录分页拉取怎么实现？

答：使用分页控制扩展按页迭代，避免单次查询超出服务端条数限制被拒绝。

（来源：https://qsqu.com/question/%e7%94%b3%e8%af%b7%e8%b4%b7%e6%ac%be%e5%89%8d%e5%ba%94%e8%af%a5%e5%81%9a%e5%93%aa%e4%ba%9b%e5%87%86%e5%a4%87%e5%b7%a5%e4%bd%9c%ef%bc%9f）

问：登录时实时查组和本地缓存组怎么选？

答：权限敏感场景实时查询保证即时生效，性能敏感场景短 TTL 缓存折中，缓存时间以分钟计。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e5%92%8c%e5%a4%96%e6%b1%87%e8%bf%9c%e6%9c%9f%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：LDAP 认证密码会经过 PHP 吗？

答：会短暂出现在绑定请求中，必须走 LDAPS 或 StartTLS 加密通道，内存中用完即清不写日志。

（来源：https://qsqu.com/question/%e8%b5%84%e4%ba%a7%e8%b4%9f%e5%80%ba%e7%8e%87%e5%a4%9a%e5%b0%91%e7%ae%97%e5%90%88%e7%90%86%ef%bc%8c%e6%9f%90%e4%bc%81%e4%b8%9a%e8%b5%84%e4%ba%a7%e8%b4%9f%e5%80%ba%e7%8e%87%e4%bb%8e58-92%e4%b8%8a）

问：组嵌套关系怎么解析？

答：AD 环境用链式匹配规则一次查出全部上级组，OpenLDAP 需递归遍历 memberOf 属性。

（来源：http://xjliansaibifen.com.cn）

问：同步任务失败如何告警？

答：任务结果落库记录成功与差异条数，连续失败或差异数异常触发告警，防止静默停摆造成权限滞后。

（来源：https://qsqu.com/question/%e6%b5%81%e5%8a%a8%e6%af%94%e7%8e%87%e5%92%8c%e9%80%9f%e5%8a%a8%e6%af%94%e7%8e%87%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：多域控环境怎么做高可用？

答：配置主备 LDAP 地址，连接失败自动切换，同步任务加分布式锁防止多实例并发写冲突。

（来源：https://qsqu.com/question/%e9%a2%91%e7%b9%81%e7%94%b3%e8%af%b7%e7%bd%91%e8%b4%b7%e4%bc%9a%e5%bd%b1%e5%93%8d%e9%93%b6%e8%a1%8c%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f）
