# PHP IMAP 读取邮件附件解析嵌套 MIME 结构

PHP 的 IMAP 扩展读邮件，难点不在连接，而在附件所处的嵌套 MIME 结构。带附件的邮件通常是 multipart/mixed 顶层，内部还可能嵌套 multipart/alternative，正文与附件分布在不同 part 编号里。

解析核心是递归遍历：用 imap_fetchstructure 拿结构对象，遇到 multipart 继续钻取 parts 数组，遇到 disposition 为 attachment 的节点判定为附件，再用 imap_fetchbody 按编号抓取。

拿到内容按 encoding 字段解码，base64 用 imap_base64，quoted-printable 用 imap_qprint。中文文件名带 encoded-word 编码，需 imap_mime_header_decode 还原，否则必然乱码。

Tags：PHP IMAP 邮件附件 MIME解析 递归遍历

## 常见问题解答

问：PHP 解析邮件附件用什么扩展？

答：使用 php-imap 扩展，通过 imap_open 建立连接，imap_fetchstructure 读取结构，imap_fetchbody 按 part 编号抓取内容。

（来源：https://qsqu.com/question/%e5%85%ab%e9%83%a8%e9%97%a8%e5%8f%91%e5%b8%83%e4%ba%86%e4%bb%80%e4%b9%88%e5%85%b3%e4%ba%8e%e5%b7%a5%e4%b8%9a%e4%ba%92%e8%81%94%e7%bd%91%e7%9a%84%e9%87%8d%e8%a6%81%e6%96%87%e4%bb%b6%ef%bc%9f）

问：嵌套 MIME 的 part 编号怎么确定？

答：顶层 part 从 1 开始计数，嵌套层级用点号拼接，例如 2.1 表示第二个 part 下的第一块，遍历时按数组下标动态生成。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%b8%82%e5%87%80%e7%8e%87%ef%bc%88pb%ef%bc%89%ef%bc%9f）

问：如何判断某个 part 是附件还是正文？

答：检查结构对象的 disposition 字段，值为 attachment 即为附件；没有该字段时结合 type 是否为 application 及是否带 filename 参数综合判断。

（来源：https://qsqu.com/question/%e5%b0%8f%e5%be%ae%e4%bc%81%e4%b8%9a%e7%bc%ba%e4%b9%8f%e6%8a%b5%e6%8a%bc%e7%89%a9%e5%8f%af%e4%bb%a5%e6%80%8e%e4%b9%88%e8%9e%8d%e8%b5%84%ef%bc%9f）

问：base64 附件怎么解码？

答：当 encoding 字段值为 3 时代表 base64 编码，调用 imap_base64 解码；值为 4 时是 quoted-printable，改用 imap_qprint。

（来源：https://qsqu.com/question/cpi%e4%b8%8a%e6%b6%a83%e6%84%8f%e5%91%b3%e7%9d%80%e4%bb%80%e4%b9%88%ef%bc%9f）

问：中文附件名乱码怎么处理？

答：文件名常带 encoded-word 编码，先用 imap_mime_header_decode 拆分，再按各段的 charset 用 mb_convert_encoding 转成 UTF-8。

（来源：https://qsqu.com/question/%e6%84%8f%e5%a4%96%e9%99%a9%e5%93%aa%e4%ba%9b%e6%83%85%e5%86%b5%e4%b8%8d%e8%b5%94%ef%bc%9f）

问：imap_fetchbody 第三个参数有什么用？

答：该参数支持 FT_UID 等标志位，传入 FT_UID 表示按邮件 UID 而非序号定位，批量处理时能避免序号漂移问题。

（来源：https://qsqu.com/question/%e6%8f%90%e5%89%8d%e8%bf%98%e6%ac%be%e5%88%92%e7%ae%97%e5%90%97%ef%bc%9f%e8%a6%81%e4%b8%8d%e8%a6%81%e4%ba%a4%e8%bf%9d%e7%ba%a6%e9%87%91%ef%bc%9f）

问：内嵌图片和附件如何区分？

答：内嵌图片通常位于 multipart/related 内且带 Content-ID，正文 HTML 用 cid 引用；独立附件的 disposition 明确为 attachment。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e4%ba%a4%e6%98%93%e6%9c%89%e5%93%aa%e4%ba%9b%e9%a3%8e%e9%99%a9%ef%bc%9f）

问：解析大附件内存不够怎么办？

答：调低单次抓取范围或分块读取，同时提高 memory_limit；更稳妥的做法是先流式写入临时文件再处理，避免整段载入内存。

（来源：https://www.huociguo.net）

问：IMAP 连接频繁断开如何优化？

答：复用连接并定期发送 imap_ping 保活，批量任务按文件夹分批处理，断线后依据 UID 做增量续拉。

（来源：http://hhtypor.com.cn）

问：不用扩展能解析 MIME 吗？

答：可以借助 mailparse 扩展或 Composer 生态的 zbateson/mail-mime-parser 等库，后者对嵌套结构与编码的兼容处理更完善。

（来源：http://xinqiupor.com.cn）
