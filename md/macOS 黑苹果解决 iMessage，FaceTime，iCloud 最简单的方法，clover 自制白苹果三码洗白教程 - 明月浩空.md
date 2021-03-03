> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [lmih.cn](https://lmih.cn/Serial_Number.html)

首先看洗白之后，iMessage 正常发消息，FaceTime 可以正常登陆聊天，iCloud 可正常登陆

[![](https://lmih.cn/content/uploadfile/201810/6f6d1539954966.png)](https://lmih.cn/content/uploadfile/201810/6f6d1539954966.png)

洗白教程如下：

找到 Clover 的 config.plist，使用 Clover Configurator 打开，选择左侧 SMBIOS，使用右侧上下箭头（老版本应该是个魔术棒图标）选择适合自己的 SMBIOS 型号

复制 Serial Number 前往 [Apple 官网查询序列号](https://checkcoverage.apple.com/cn/zh)，如果显示有购买日期和机型等保修信息，说明这个序列号别人在用，不建议使用，点击 Generate New 生成新的 Serial Number，直到出现提示：很抱歉，这个序列号无效。请检查您的信息并再试一次。（一般都提示这个）

[![](https://lmih.cn/content/uploadfile/201810/77a51539955732.png)](https://lmih.cn/content/uploadfile/201810/77a51539955732.png)

如果你运气特别好，有可能生成出来官方正版且未被激活的白苹果序列号

[![](https://lmih.cn/content/uploadfile/201810/f7c21539955731.png)](https://lmih.cn/content/uploadfile/201810/f7c21539955731.png)

选择 Clover Configurator 左侧 System Parameters，生成一个 UUID，系统生成随机生成都可以

[![](https://lmih.cn/content/uploadfile/201810/cdc91539955909.png)](https://lmih.cn/content/uploadfile/201810/cdc91539955909.png)

复制生成等 UUID 粘贴到 SMBIOS 的 SmUUID 里，使用 Win+s 保存或者点击左下角 save

[![](https://lmih.cn/content/uploadfile/201810/c8131539956012.png)](https://lmih.cn/content/uploadfile/201810/c8131539956012.png)

复制 SMBIOS 里的 Board Serial Number，粘贴到 Rt Variables 的 MLB 里，保存之后，点击 ROM 的获取，确保下方三码正常匹配自己生成的，保存重启即可

[![](https://lmih.cn/content/uploadfile/201810/61aa1539956732.png)](https://lmih.cn/content/uploadfile/201810/61aa1539956732.png)

本文标签：这篇文章木有标签

版权声明：若无特殊注明，本文为《[明月浩空](https://lmih.cn/author/1 "博客管理员")》原创，转载请保留文章出处。  
本文链接：https://lmih.cn/Serial_Number.html

正文到此结束