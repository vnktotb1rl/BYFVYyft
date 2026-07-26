# PHP 数组键整数字符串隐式转换规则及预防

PHP 数组键的类型处理是隐式转换高发区。核心规则只有一条：合法的整数字符串被静默转成整数键，其余字符串保持原样。

键名 "8" 写入后存为整数 8，"08" 因前导零保持字符串，浮点键强制取整，true 变 1、null 变空字符串，几类键撞车是常事。典型事故是 JSON decode 后的字符串键再写回数组，类型悄然改变，严格比较全盘失效。

预防手段很直接：需保真键名时改用对象属性，拿到键后显式 strval 固定类型；判断键存在用 array_key_exists 而非 isset。

关键路径开启 strict_types 虽管不到数组键，但能让函数边界的类型问题尽早暴露，整体代码更稳健。

Tags：PHP数组 隐式转换 数组键 类型安全 strict_types

## 常见问题解答

问：字符串 "8" 作为数组键会发生什么？

答：它被静默转换为整数 8，写入和读取时 PHP 自动做等价映射，表面上字符串键也能访问到同一元素。

（来源：https://qsqu.com/question/%e6%95%b0%e5%ad%97%e4%ba%ba%e6%b0%91%e5%b8%81%ef%bc%88e-cny%ef%bc%89%e4%b8%8e%e7%ac%ac%e4%b8%89%e6%96%b9%e6%94%af%e4%bb%98%e5%b9%b3%e5%8f%b0%e5%9c%a8%e8%b4%a7%e5%b8%81%e5%b1%9e%e6%80%a7%e4%b8%8e）

问：为什么 "08" 不会被转成整数？

答：前导零的写法不符合 PHP 对合法整数字符串的定义，转换规则不适用，键保持字符串原样。

（来源：https://qsqu.com/question/%e7%a4%be%e4%bc%9a%e8%9e%8d%e8%b5%84%e8%a7%84%e6%a8%a1%e8%bf%99%e4%b8%aa%e6%8c%87%e6%a0%87%e7%9c%8b%e4%bb%80%e4%b9%88%ef%bc%9f）

问：浮点数作为键会怎样？

答：浮点键被强制转换为整数，小数部分直接截断，1.9 和 1.1 会撞成同一个键，后写入的值覆盖前一个。

（来源：https://qsqu.com/question/%e5%85%ac%e7%a7%af%e9%87%91%e8%b4%b7%e6%ac%be%e5%92%8c%e5%95%86%e4%b8%9a%e8%b4%b7%e6%ac%be%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：布尔值和 null 能做数组键吗？

答：语法上允许，true 转为 1，false 转为 0，null 转为空字符串，都属于容易踩坑的隐式行为。

（来源：https://qsqu.com/question/%e8%b4%a2%e6%94%bf%e6%94%bf%e7%ad%96%e5%92%8c%e8%b4%a7%e5%b8%81%e6%94%bf%e7%ad%96%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：json_decode 后的键名类型是什么？

答：JSON 对象键经 decode 后是字符串，但一旦用作 PHP 数组键，整数字符串键立刻被转成整数，类型就此改变。

（来源：https://qsqu.com/question/%e5%9c%ba%e5%86%85%e5%9f%ba%e9%87%91%e5%92%8c%e5%9c%ba%e5%a4%96%e5%9f%ba%e9%87%91%e4%b8%bb%e8%a6%81%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：如何检查字符串键是否存在？

答：使用 array_key_exists 并传入字符串键即可，PHP 内部做同样的转换，能找到对应元素；isset 遇到 null 值会漏判。

（来源：https://qsqu.com/question/%e6%88%bf%e4%ba%a7%e6%8a%b5%e6%8a%bc%e8%b4%b7%e6%ac%be%e4%b8%80%e8%88%ac%e8%83%bd%e8%b4%b7%e5%87%ba%e5%87%a0%e6%88%90%ef%bc%9f）

问：foreach 拿到的键还能保持字符串吗？

答：不能，遍历返回的是数组内部实际存储的键，整数字符串键早已变成整数，严格类型比较会失败。

（来源：http://cps.guolupipe.com）

问：怎样真正保留字符串键？

答：改用 stdClass 对象属性或 SplObjectStorage 语义，对象属性名不做整数化转换；或干脆把键做一层前缀包装。

（来源：https://qsqu.com/question/%e5%85%b7%e8%ba%ab%e6%99%ba%e8%83%bd%e4%ba%a7%e4%b8%9a%e5%8f%91%e5%b1%95%e6%83%85%e5%86%b5%e5%a6%82%e4%bd%95%ef%bc%9f）

问：strict_types 能阻止键转换吗？

答：不能，strict_types 只约束函数参数与返回值的标量类型，数组键转换是语言底层行为，只能编码时主动规避。

（来源：https://qsqu.com/question/%e7%bb%8f%e8%90%a5%e8%b4%b7%e5%92%8c%e6%b6%88%e8%b4%b9%e8%b4%b7%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

问：排序函数会受键类型影响吗？

答：ksort 默认按常规规则比较，整数键与字符串键混排结果可能出乎意料，必要时指定 SORT_STRING 或 SORT_NUMERIC 标志。

（来源：https://qsqu.com/question/%e5%85%ac%e7%a7%af%e9%87%91%e8%b4%b7%e6%ac%be%e5%88%a9%e7%8e%87%e6%98%af%e5%a4%9a%e5%b0%91%ef%bc%9f）
