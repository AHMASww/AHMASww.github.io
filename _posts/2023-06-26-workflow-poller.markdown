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

## timerfd

### 名称(NAME)

`timerfd_create, timerfd_setting, timerfd_gettime` - 使用文件描述符来通知的定时器

### 概要(SYNOPSIS)

```c++
#include <sys/timerfd.h>

    int timerfd_create(int clockid, int flags);

    int timerfd_settime(int fd, int flags,
                        const struct itimerspec *new_value,
                        struct itimerspec *old_value);

    int timerfd_gettime(int fd, struct itimerspec *curr_value);
```

### 描述(DESCRIPTION)

这些系统调用通过文件描述符来传递定时器失效通知。这些可作为`settimer(2)`和`timer_create(2)`的替代，同时文件描述符可以被
`select(2)，poll(2), epoll(2)`监视。

这些系统调用和`timer_create(2), timer_settime(2), timer_gettime(2)`是相似的。(并不和`timer_getoverrun(2)`相似)

#### `timerfd_create()`

`timerfd_create()`创建一个全新的定时器实例并将和该定时器相关联的文件描述符作为返回值。`clockid`参数表示该程序使用的时钟
并且必须是下列值之一:

`CLOCK_REALTIME`: 一个可设定的系统级实时时钟。

`CLOCK_MONOTONIC`: 一个不可设置的单调递增时钟，用于测量过去某个未指定点的时间，该时间点在系统启动后不会改变。

`CLOCK_BOOTTIME(Linux 3.15以后)`: 和`CLOCK_MONOTONIC`类似是一个单调递增时钟。`CLOCK_MONOTONIC`不能在系统挂起的时候使用，
`CLOCK_BOOTTIME`却可以在系统挂起时使用。这在应用需要挂起唤醒时候是有用的。`CLOCK_REALTIME`在系统时钟是非实时计数是不可
用的。

`CLOCK_REALTIME_ALARM(Linux 3.11以后)`: 类似于`CLOCK_REALTIME`，但是当它挂起时会通知系统，调用系统必须有`CAP_WAKE_ALARM
`能力去设置对应的定时器。

`CLOCK_BOOTTIME_ALARM(Linux 3.11以后)`: 类似于`CLOCK_BOOTTIME`，但是当它挂起时会通知系统，调用系统必须有`CAP_WAKE_ALARM
`能力去设置对应的定时器。

可以查看`clock_getres(2)`去获得更多关于时钟的细节。

每个时钟的当前值都可以通过使用`clock_gettime(2)`去检索。

从Linux2.6.27版本开始，`timerfd_create`中用以改变行为的参数`flags`是为

