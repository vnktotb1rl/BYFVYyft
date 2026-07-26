# composer.lock 锁定与语义化版本策略运用

团队开发里"在我机器上是好的"这类事故，半数源于依赖版本漂移。composer.lock 记录每个包的精确版本与依赖树快照，任何人执行 composer install 得到的都是同一份依赖。

composer.json 的约束遵循语义化版本：^2.1 允许 2.x 内兼容更新不跨 3.0，~2.1.3 收紧到 2.1.x；主版本号为 0 的包变动快，^0.8 只允许 0.8.x 升级。

lock 必须提交版本库，服务器只跑 composer install --no-dev，绝不执行 update。升级是开发期主动行为，本地 update 指定包名、跑完测试再提交新 lock，composer audit 挂进 CI 扫描已知漏洞。

Tags：composer.lock 语义化版本 依赖管理 PHP包管理 供应链安全

## 常见问题解答

问：composer.lock 要不要提交到 Git？

答：必须提交，它是团队与部署环境依赖一致性的唯一保障，应用项目提交 lock，纯类库项目可以不提交。

（来源：https://qsqu.com/question/%e4%b9%b0%e5%9f%ba%e9%87%91%e9%9c%80%e8%a6%81%e5%85%b3%e6%b3%a8%e5%93%aa%e4%ba%9b%e8%b4%b9%e7%94%a8%ef%bc%9f）

问：install 和 update 的本质区别是什么？

答：install 严格按 lock 文件安装已锁定版本，lock 缺失时才解析生成；update 忽略现有 lock 重新解析约束并刷新锁定结果。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e5%88%a4%e6%96%ad%e8%b4%b7%e6%ac%be%e5%88%a9%e7%8e%87%e6%98%af%e5%90%a6%e5%90%88%e7%90%86%ef%bc%9f）

问：^2.1 和 ~2.1 有什么区别？

答：^2.1 允许升级到 2.x 内任意更高版本；~2.1 只允许 2.x 系列，写成 ~2.1.3 时进一步收紧到 2.1.x。

（来源：http://pg-yl.com.cn）

问：0.x 版本的脱字符约束为什么特殊？

答：语义化版本约定主版本号为 0 时不保证兼容性，Composer 因此把 ^0.8 限制在 0.8.x 范围内，防止破坏性升级。

（来源：https://qsqu.com/question/%e5%85%88%e6%81%af%e5%90%8e%e6%9c%ac%e5%92%8c%e7%ad%89%e9%a2%9d%e6%9c%ac%e6%81%af%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%e6%80%8e%e4%b9%88%e9%80%89%ef%bc%9f）

问：生产环境部署用什么命令？

答：composer install --no-dev --optimize-autoloader，跳过开发依赖并生成优化后的自动加载映射，速度与安全兼顾。

（来源：http://fczuqiusj.com.cn）

问：想升级单个包怎么做？

答：执行 composer update 包名，只刷新该包及其必要的联动依赖，其余锁定版本保持不动，变更范围可控。

（来源：https://qsqu.com/question/%e6%8a%b5%e6%8a%bc%e8%b4%b7%e6%ac%be%e5%92%8c%e4%bf%a1%e7%94%a8%e8%b4%b7%e6%ac%be%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：lock 文件冲突怎么解决？

答：不要手工编辑，选择一方的 lock 为基础，重新执行 composer update 相关包让 Composer 重新解析生成一致快照。

（来源：https://qsqu.com/question/%e8%82%a1%e7%a5%a8%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f）

问：composer audit 是干什么的？

答：检查 lock 中依赖是否命中已知安全漏洞数据库，输出受影响包与修复版本建议，适合接入 CI 强制卡点。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%85%b6%e4%bb%96%e5%ba%94%e6%94%b6%e6%ac%be%e5%92%8c%e5%85%b6%e4%bb%96%e5%ba%94%e4%bb%98%e6%ac%be%ef%bc%8c%e4%b8%ba%e4%bb%80%e4%b9%88%e5%ae%83%e4%bb%ac%e5%ae%b9）

问：为什么 CI 里建议跑 composer validate？

答：该命令校验 composer.json 与 lock 是否同步、约束是否合法，提前拦截忘记提交新 lock 的低级失误。

（来源：https://qsqu.com/question/2026%e5%b9%b41-5%e6%9c%88%e4%b8%ad%e5%9b%bd%e6%9c%8d%e5%8a%a1%e8%b4%b8%e6%98%93%e8%a1%a8%e7%8e%b0%e5%a6%82%e4%bd%95%ef%bc%9f）

问：私有包也受 lock 管理吗？

答：一样管理，无论来自 Packagist 还是私有仓库，锁定机制完全相同，版本与来源信息都会写入 lock 快照。

（来源：https://qsqu.com/question/%e9%a2%91%e7%b9%81%e7%94%b3%e8%af%b7%e7%bd%91%e8%b4%b7%e4%bc%9a%e5%bd%b1%e5%93%8d%e9%93%b6%e8%a1%8c%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f）
