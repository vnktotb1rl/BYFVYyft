# PHP 多字节正则 \p{L} 与 \w 匹配差异

用正则处理中文时，\w 与 \p{L} 的差异常让人困惑：一个漏掉汉字，一个照单全收，根源在定义域不同。

\w 默认只覆盖 ASCII 的字母数字下划线，PCRE 按字节处理时中文一个也匹配不上；加 u 修饰符开启 UTF-8 模式后，\w 才扩展到 Unicode 字母数字范畴。

\p{L} 是 Unicode 属性写法，专指任意语言的字母，不含数字与下划线，必须配合 u 修饰符。选型清晰：中英文加数字用 [\p{L}\p{N}_]+，纯词语用 \p{L}+，只匹配中文用 \p{Han}。

切记没加 u 修饰符时 \p{L} 属于字节级的未定义行为，匹配结果不可依赖，这一条是所有写法的大前提。

Tags：PHP正则 多字节 Unicode PCRE 字符匹配

## 常见问题解答

问：\w 默认能匹配中文吗？

答：不能，未加 u 修饰符时 PCRE 按字节处理，\w 仅覆盖 ASCII 的字母数字下划线，汉字完全匹配不到。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e6%8a%a5%e5%91%8a%e4%b8%8a%e6%9c%89%e9%80%be%e6%9c%9f%e8%ae%b0%e5%bd%95%e8%bf%98%e8%83%bd%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f）

问：加了 u 修饰符后 \w 有什么变化？

答：进入 UTF-8 模式后 \w 扩展为 Unicode 字母与数字加下划线，此时可以匹配中文，但同时也放进了各国文字。

（来源：https://qsqu.com/question/%e8%b5%84%e4%ba%a7%e8%b4%9f%e5%80%ba%e7%8e%87%e5%a4%9a%e5%b0%91%e7%ae%97%e5%90%88%e7%90%86%ef%bc%8c%e6%9f%90%e4%bc%81%e4%b8%9a%e8%b5%84%e4%ba%a7%e8%b4%9f%e5%80%ba%e7%8e%87%e4%bb%8e58-92%e4%b8%8a）

问：\p{L} 具体匹配哪些字符？

答：匹配 Unicode 定义的全部字母，包括拉丁字母、汉字、日文假名、西里尔字母等，不含数字与下划线。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e5%88%b0%e6%9c%9f%e6%80%8e%e4%b9%88%e4%ba%a4%e5%89%b2%ef%bc%9f）

问：\p{L} 必须配合 u 修饰符吗？

答：是的，Unicode 属性在字节模式下无意义，缺少 u 修饰符时匹配行为不可预期，属于错误用法。

（来源：https://qsqu.com/question/%e7%8e%b0%e5%9c%a8%e6%88%bf%e8%b4%b7%e5%88%a9%e7%8e%87%e6%98%af%e5%a4%9a%e5%b0%91%ef%bc%9f）

问：只想匹配中文怎么写？

答：使用 [\x{4e00}-\x{9fff}] 的区间写法或 \p{Han} 脚本属性，都要带 u 修饰符，后者语义更准确。

（来源：https://qsqu.com/question/%e6%88%bf%e4%ba%a7%e6%8a%b5%e6%8a%bc%e8%b4%b7%e6%ac%be%e4%b8%80%e8%88%ac%e8%83%bd%e8%b4%b7%e5%a4%9a%e5%b0%91%ef%bc%9f）

问：校验中英文混合用户名推荐什么写法？

答：/^[\p{L}\p{N}_]{2,20}$/u 这种组合兼顾中英文与数字，长度限制一并收紧，兼容性与严谨性平衡较好。

（来源：https://qsqu.com/question/%e5%9b%bd%e5%8a%a1%e9%99%a2%e5%b8%b8%e5%8a%a1%e4%bc%9a%e8%ae%ae%e5%af%b9%e5%a4%96%e8%b4%b8%e6%9c%89%e4%bd%95%e6%9c%80%e6%96%b0%e9%83%a8%e7%bd%b2%ef%bc%9f）

问：\p{N} 和 \d 有什么区别？

答：\d 默认只匹配 ASCII 数字，加 u 后含义不变；\p{N} 匹配 Unicode 全部数字字符，包括全角数字与其他文字的数字系统。

（来源：http://xcc.hcsggy.com）

问：preg_match 返回 false 是什么原因？

答：模式语法错误或不加 u 修饰符时传入非法 UTF-8 字节序列会导致返回 false，应用 === 严格区分 false 与 0。

（来源：https://qsqu.com/question/%e5%af%bf%e9%99%a9%e4%bf%9d%e9%a2%9d%e6%80%8e%e4%b9%88%e7%ae%97%e6%89%8d%e4%b8%8d%e5%9d%91%ef%bc%9f）

问：mb_ereg 系列函数还有必要用吗？

答：旧的多字节正则扩展已被 PCRE 的 u 修饰符全面超越，新代码统一走 preg 系列，老函数只存在于历史项目。

（来源：https://qsqu.com/question/%e6%8a%95%e4%bf%9d%e6%97%b6%e4%b8%8d%e5%a6%82%e5%ae%9e%e5%91%8a%e7%9f%a5%e5%81%a5%e5%ba%b7%e7%8a%b6%e5%86%b5%e4%bc%9a%e6%80%8e%e6%a0%b7%ef%bc%9f）

问：性能上 \p{L} 比 \w 慢吗？

答：Unicode 属性查表成本略高，但在常规文本规模下差异微乎其微，可读性与正确性远比这点开销重要。

（来源：http://wnv.qfyyds.com）
