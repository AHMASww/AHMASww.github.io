---
layout: post
title: "workflow poller 理解"
date: 2023-06-26 17:50:00 +0800
categories: jekyll update
---

# workflow poller 理解

## char data[0] 柔性数组

```c++
struct __poller_message
{
	int (*append)(const void *, size_t *, poller_message_t *);
	char data[0];
};
```
`char data[0]`从语法逻辑上来看是完全没有意义的，但在结构体中却能完成变长度的作用，即结构体的长度是不定的，可以用来构造
缓冲区、数据包等等。对于编译器而言，长度为0的数组不占用空间，数组名本身不占用空间，只是一个偏移量。通过`poll_message->
data`可以定位到数据的开始。

```c++
#include <iostream>


struct buffer
{
    int32_t length;
    char data[0];
};

int main()
{
    buffer my_buffer = {
        .length = 10
    };
    
    // 输出地址
    std::cout << std::hex;
    std::cout << "&my_buffer.length : " << &my_buffer.length << std::endl;
    std::cout << "&my_buffer.data   : " << &my_buffer.data << std::endl;

    return 0;
}
```

```
&my_buffer.length : 0x7fffc83ade84
&my_buffer.data   : 0x7fffc83ade88
```

## union

union 是一个用户定义类型，其中所有成员都共享同一个内存位置。 此定义意味着在任何给定时间，union 都不能包含来自其成员列表
的多个对象。 这还意味着，无论 union 具有多少成员，它始终仅使用足以存储最大成员的内存。

