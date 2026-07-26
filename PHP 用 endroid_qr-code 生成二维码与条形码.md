# PHP 用 endroid/qr-code 生成二维码与条形码

PHP 生成二维码，endroid/qr-code 是生态里维护最活跃的组件，Composer 一条命令装完，十几行代码产出第一张码图。

基础用法围绕 Builder 对象展开：QrCode::create 传入内容，Encoding 指定 UTF-8 保证中文正常，ErrorCorrectionLevel 选择容错等级，越高越抗遮挡但码点越密。Writer 决定输出格式，PngWriter 出位图，SvgWriter 出矢量图，getDataUri 可直接生成嵌进 img 标签的 data URI。

高频需求还有三个：嵌 Logo 用 Logo::create 加载图片，容错等级调到 High 防扫码失败；改颜色保持深码点浅底色原则；条形码另选 php-barcode-generator，支持 Code128 等主流制式。

Tags：PHP二维码 endroid qr-code 条形码 图片生成

## 常见问题解答

问：endroid/qr-code 怎么安装？

答：Composer 执行 require endroid/qr-code 即可，依赖 GD 或 Imagick 扩展出位图，SVG 输出无此依赖。

（来源：https://qsqu.com/question/%e4%ba%ba%e5%8f%a3%e8%80%81%e9%be%84%e5%8c%96%e4%bc%9a%e6%80%8e%e6%a0%b7%e5%bd%b1%e5%93%8d%e7%bb%8f%e6%b5%8e%ef%bc%9f）

问：生成的二维码如何直接输出到浏览器？

答：调用 result 的 getString 拿二进制内容，设置 Content-Type 头后 echo；getDataUri 可直接嵌入 img 标签。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%ae%9a%e6%8a%95%e4%b8%8e%e4%b8%80%e6%ac%a1%e6%80%a7%e4%b9%b0%e5%85%a5%e5%93%aa%e4%b8%aa%e6%94%b6%e7%9b%8a%e6%9b%b4%e9%ab%98）

问：中文内容乱码怎么办？

答：Encoding 显式指定 UTF-8，内容本身也需是 UTF-8 编码，数据库读出的旧编码数据先转换再生成。

（来源：https://qsqu.com/question/%e6%80%8e%e4%b9%88%e7%ae%80%e5%8d%95%e5%88%a4%e6%96%ad%e4%b8%80%e5%8f%aa%e5%9f%ba%e9%87%91%e7%9a%84%e9%a3%8e%e9%99%a9%e6%b0%b4%e5%b9%b3%ef%bc%9f）

问：容错等级怎么选？

答：Low 到 High 四档，码面要嵌 Logo 或印刷磨损场景选 High，纯屏幕展示用 Medium 兼顾密度与容错。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e3%80%8a%e5%a2%9e%e5%80%bc%e7%a8%8e%e6%b3%95%e3%80%8b%e5%ae%9e%e6%96%bd%e5%90%8e%ef%bc%8c%e5%b0%8f%e8%a7%84%e6%a8%a1%e7%ba%b3%e7%a8%8e%e4%ba%ba%e7%9a%84%e5%a2%9e%e5%80%bc%e7%a8%8e）

问：如何在二维码中间加 Logo？

答：Logo::create 加载图片文件并设定显示尺寸，尺寸不超过码面三成，容错等级配合调到 High。

（来源：https://www.huociguo.com）

问：可以生成 SVG 矢量格式吗？

答：可以，SvgWriter 输出矢量图，任意缩放不模糊，适合印刷与响应式页面，且不依赖图形扩展。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e5%88%a4%e6%96%ad%e4%bc%81%e4%b8%9a%e6%8f%90%e4%be%9b%e7%9a%84%e8%b4%a2%e5%8a%a1%e6%8a%a5%e8%a1%a8%e6%98%af%e5%90%a6%e7%9c%9f%e5%ae%9e%ef%bc%9f）

问：二维码颜色可以自定义吗？

答：前景背景色均可通过颜色类设定，保持深码点浅底色的高对比原则，反色方案扫码兼容性差。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e5%b0%86%e8%87%aa%e4%ba%a7%e8%b4%a7%e7%89%a9%e7%94%a8%e4%ba%8e%e5%85%ac%e7%9b%8a%e6%80%a7%e6%8d%90%e8%b5%a0%ef%bc%8c%e5%a2%9e%e5%80%bc%e7%a8%8e%e5%a6%82%e4%bd%95%e5%a4%84%e7%90%86）

问：保存成文件怎么写？

答：调用 result 的 saveToFile 传入目标路径，格式由 Writer 决定，权限需保证 PHP 进程可写。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%ae%9a%e6%8a%95%ef%bc%9f%e8%83%bd%e5%90%a6%e7%94%a8%e4%ba%8e%e8%82%a1%e7%a5%a8%ef%bc%9f）

问：PHP 生成条形码用什么组件？

答：picqer/php-barcode-generator 支持 Code128、EAN13 等主流制式，输出 HTML、SVG 与 PNG 多种形式。

（来源：https://qsqu.com/question/%e4%ba%92%e8%81%94%e7%bd%91%e5%ad%98%e6%ac%be%e4%ba%a7%e5%93%81%e4%b8%ba%e4%bb%80%e4%b9%88%e5%9c%a8%e5%90%84%e5%a4%a7%e5%b9%b3%e5%8f%b0%e4%b8%8b%e6%9e%b6%e4%ba%86%ef%bc%9f）

问：批量生成二维码如何控制内存？

答：逐个生成逐个落盘，及时释放结果对象；文件名用业务标识哈希命名，天然支持幂等重跑。

（来源：https://qsqu.com/question/%e5%a4%b1%e4%b8%9a%e7%8e%87%e5%92%8c%e7%bb%8f%e6%b5%8e%e5%a5%bd%e5%9d%8f%e6%9c%89%e4%bb%80%e4%b9%88%e5%85%b3%e7%b3%bb%ef%bc%9f）
