---
title: "mht和chm文件打不开的解决方法"
date: 2010-01-20T00:00:00+08:00
draft: false
type: "post"
tags:
    - "Tips"
---

先引入这两种文件：

* `mht` IE保存的文件格式，也不知道是什么版本开始成为默认的网页保存格式。
* `chm` 不用解释了吧。

情况是这样：双击`mht`文件打不开，双击`chm`文件打开后无法显示。

怎么办呢？

仔细检查了一下，发现了一些问题：文件名中是包含了一些特殊的字符，例如文件名为`C#测试.mht`，这样的话就会出现上述的问题。

为了进一步研究，尝试用正常的文件名，但是将路径中的目录名改为包含特殊的字符，也出现了上述的问题。

看来在文件名和路径中不能出现特殊的字符，那么在IE中将这些特殊的字符进行转义（或者称之为Encode更为合适）行不行呢？答案是：不行。

接下来有理由怀疑所谓的“特殊字符”绝不止`#`这一个字符，如果大家遇到类似的问题可以尝试检查文件名或者路径，估计这样可以解决99%的所谓`mht`文件打不开的故障吧。

（上述问题已经反馈到微软，暂时没有一个完整解决方案，例如补丁什么的）

说完.mht，回过头来说.chm。

我的解决方案就一条：参照上述解决方法，原因是我已经用这种方法解决了，如果不行，请Google之。