# Valitron 链式校验与自定义错误消息编写

表单校验写一堆 if else 是老项目的常见景象，规则散落各处。Valitron 用声明式规则加链式调用，把校验逻辑压缩到几行之内。

用法是 new Validator 传入待校验数组，随后连续调用 rule 叠加规则。rule 第一个参数是规则名，内置 required、email、lengthBetween 等几十种，第二个参数指定字段，可批量套用。validate 返回布尔值，errors 按字段取回失败明细。

自定义消息用 message 方法，支持 {field} 与 {value} 占位符，可针对单字段单规则覆盖；labels 方法给字段起中文别名，错误提示直接显示业务名称。

Tags：Valitron PHP校验 链式调用 表单验证 错误消息

## 常见问题解答

问：Valitron 怎么安装？

答：通过 Composer 执行 require vlucas/valitron 即可安装，库体积很小，没有任何框架依赖，传统项目和现代框架都能用。

（来源：https://qsqu.com/question/%e7%ad%89%e9%a2%9d%e6%9c%ac%e9%87%91%e5%92%8c%e7%ad%89%e9%a2%9d%e6%9c%ac%e6%81%af%e5%93%aa%e4%b8%aa%e5%88%92%e7%ae%97%ef%bc%9f）

问：required 和 optional 规则怎么配合？

答：required 表示字段必须存在且非空；optional 表示字段可以缺席，但一旦存在就必须通过后续规则，常用于非必填但有格式要求的场景。

（来源：https://qsqu.com/question/%e5%ad%98%e6%ac%be%e5%87%86%e5%a4%87%e9%87%91%e7%8e%87%e4%b8%8b%e8%b0%83%ef%bc%8c%e4%bc%9a%e4%ba%a7%e7%94%9f%e5%93%aa%e4%ba%9b%e5%bd%b1%e5%93%8d%ef%bc%9f）

问：如何校验数组类型的字段？

答：Valitron 支持点号语法访问嵌套数组，也可以对数组元素批量套用规则，例如 items.*.price 表示数组每一项的 price 字段。

（来源：https://qsqu.com/question/%e8%b4%b7%e6%ac%be%e8%b5%84%e9%87%91%e5%8f%af%e4%bb%a5%e7%94%a8%e4%ba%8e%e5%93%aa%e4%ba%9b%e7%94%a8%e9%80%94%e6%9c%89%e5%93%aa%e4%ba%9b%e9%99%90%e5%88%b6%ef%bc%9f）

问：自定义规则如何注册？

答：调用 Validator 的 addRule 静态方法传入规则名和闭包，闭包接收字段名、值、参数，返回布尔结果，注册后全局可用。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e7%ba%b3%e7%a8%8e%e5%b9%b4%e5%ba%a6%e5%8f%91%e7%94%9f%e7%9a%84%e4%ba%8f%e6%8d%9f%ef%bc%8c%e6%9c%80%e9%95%bf%e5%8f%af%e4%bb%a5%e7%bb%93%e8%bd%ac%e5%a4%9a%e5%b0%91%e5%b9%b4%e5%bc%a5）

问：message 方法的占位符有哪些？

答：支持 {field} 输出字段名、{value} 输出实际值，规则参数也会以位置占位符形式出现，可以拼出可读性很高的提示语。

（来源：http://egw.yurifen.com）

问：如何给字段设置中文显示名？

答：使用 label 或 labels 方法传入字段与别名的映射，错误消息中的 {field} 会被替换成别名，中文表单提示一步到位。

（来源：http://lob.kvxrcf.cn）

问：validate 失败后的错误结构是怎样的？

答：errors 方法返回以字段名为键的二维数组，每个字段下是该字段触发的全部错误消息，前端按字段渲染即可。

（来源：https://qsqu.com/question/%e4%b9%b0%e4%bf%9d%e9%99%a9%e5%88%b0%e5%ba%95%e5%9b%be%e4%bb%80%e4%b9%88%ef%bc%9f）

问：能否只取第一个错误？

答：调用 errors 时传入字段名并配合 error 方法可以拿到单条错误，也可以自行对结果数组做首元素提取。

（来源：https://qsqu.com/question/%e6%88%bf%e8%b4%b7%e5%88%a9%e7%8e%87%e9%80%89lpr%e6%b5%ae%e5%8a%a8%e8%bf%98%e6%98%af%e5%9b%ba%e5%ae%9a%e5%88%a9%e7%8e%87%e6%9b%b4%e5%a5%bd%ef%bc%9f）

问：Valitron 支持文件上传校验吗？

答：内置规则覆盖 $_FILES 结构，可校验文件是否上传成功、大小与类型，配合 required.file 等规则使用。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%88%86%e7%ba%a2%e6%98%af%e4%b8%8d%e6%98%af%e9%a2%9d%e5%a4%96%e8%b5%9a%e5%88%b0%e7%9a%84%e9%92%b1%ef%bc%9f）

问：和 Laravel 自带的验证器比选哪个？

答：Laravel 项目直接用框架验证器更顺手；独立脚本、轻量项目或旧系统改造选 Valitron，引入成本低且不绑定框架。

（来源：https://qsqu.com/question/%e5%ae%9a%e6%9c%9f%e5%af%bf%e9%99%a9%e5%92%8c%e7%bb%88%e8%ba%ab%e5%af%bf%e9%99%a9%e8%af%a5%e6%80%8e%e4%b9%88%e9%80%89%ef%bc%9f）
