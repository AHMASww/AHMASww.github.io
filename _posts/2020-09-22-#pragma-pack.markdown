---
layout: post
title:	"#pragma pack"
date:	2020-09-22 22:44:00 +0800
categories: jekyll update
---

计算机对基本类型数据在内存中存放的位置有限制，会要求这些数据的首地址的值是某个数`k`（通常为4或8的倍数），这就是内存对齐，而这个`k`则被称为该数据类型的对齐模数（alignment modulus）。

n字节边界对齐的意思是说一个成员的地址必须安排在成员的尺寸的整数倍地址上或者是n的整数倍地址上，取它们中的最小值，也就是：
{% highlight cpp %}
min(sizeof(member), n)
{% endhighlight %}
实际上，1字节边界对齐也就表示了结构成员之间没有空洞。
要专门对某些结构定义使用对齐选项，可以使用`#pramga pack`编译指令。

(1) #pragma pack ([n])
该指令指定结构和联合成员的紧凑对齐。该编译指示在其出现后的第一个结构或联合说明处生效。

(2) #pragma pack (show)
该指令显示当前字节对齐数目。

(3) #pragma pack (push, n)
该指令将n字节对齐压入栈，保证之后的字节对齐都为n字节。

(4) #pragma pack (pop)
该指令弹出栈中的n字节对齐，恢复之前的字节对齐数目。一般和#pragma pack (push, n) 一起使用。
