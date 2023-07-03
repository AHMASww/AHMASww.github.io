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

从Linux2.6.27版本开始，`timerfd_create`中用以改变行为的参数`flags`是位宽的下述值。

`TFD_NONBLOCK` 在新打开的文件描述符设置`O_NONBLOCK`文件描述标识，使用该标识将节省额外的`fcntl(2)`调用来达到相同结果。

`TFD_CLOEXEC` 在新的文件描述符设置`close-on-exec(FD_CLOEXEC)`文件描述标识，查询设置`O_CLOEXEC`标识的理由可以查看`open(2
)`。

Linux版本2.6.26以前(包括2.6.26)，`flag`必须被设置为0。

#### `timerfd_settime()`

`timerfd_settime()`通过文件描述符开始或停止定时器。

定时器中的`new_value`参数用来表示失效时间和间隔时间。`itimer`结构体包含使用`timespec`结构体两部分。

```c++
struct timespec {
    time_t  tv_sec;                 /* Seconds */
    long    tv_nsec;                /* Nanoseconds */
};

struct itimerspec {
    struct timespec it_interval;    /* Interval for periodic timer */
    struct timespec it_value;       /* Initial expiration */
};
```

`new_value.it_value`用秒和纳秒来表示定时器初始失效时间。`new_value.it_value`任意一部分不为0值开始定时器，两部分都为0值
停止定时器。

设置`new_value.it_interval`非0值用来表示自初始失效期后重复定时器的周期。如果该值为0，该定时器在`new_value.it_value`定时
的时间上仅失效一次。通常来讲，初始失效时间是和调用定时器时钟的当前时间相关联(即`new_value.it_value`表示的时间适合当前的
`clockid`的值相关联)。可以通过`flags`参数选择一个绝对超时时间。

`flags`是一个下述值的位掩码：

`TFD_TIMER_ABSTIME` 将`new_value.it_value`解释为一个在定时器上的绝对值。该定时器将在到达这个值设定的值上失效。

`TFD_TIMER_CANCEL_ON_SET` 如果该标识同时伴有`TFD_TIMER_ABSTIME`和时钟是`CLOCK_REALTIME`或者`CLOCK_REALTIME_ALARM`，标记
该定时器为可取消的如果实时时钟接口不连续的变化。当该变化产生，可用`read(2)`从文件描述符读出`ECANCELED`错误。

如果`old_value`参数非空，它`itimerspec`结构体返回之前定时器设置的失效值。后面`timerfd_gettime()`有描述。

#### `timerfd_gettime()`

`timerfd_gettime()`用`curr_value`参数返回和文件描述符`fd`相关的定时器设定的失效值。

`it_value`用来表示到下一次失效时的总时间。如果两项值都是0，则表明该定时器当前是停止的。这项永远包含一个相对时间，不论是
否`TFD_TIMER_ABSTIME`是否被设置。

`it_interval`项用来返回定时器的间隔。如果值为0，说明定时器仅在`curr_value.it_value`表示的时间上失效一次。

#### 时间文件描述符上的操作

使用`timerfd_create()`返回的文件描述符支持下述操作：

##### `read(2)`

如果该定时器已经失效过一次或者多次自使用`timerfd_settime()`修改或者上次成功`read(2)`，那么`read(2)`返回的buffer返回8位
无符号整数表示发生失效的次数。

如果在使用`read(2)`没有定时器失效，将阻塞到下一个定时失效，或者返回错误`EAGAIN`当这些文件描述符被设置为非阻塞时。

如果提供的buffer小于8个字节，`read(2)`将返回`EINVAL`错误。

如果相关时钟是`CLOCK_REALTIME`或者`CLOCK_REALTIME_ALARM`，定时器是绝对的(`TFD_TIMER_ABSTIME`)并且标识是`TFD_TIMER_CANCE
L_ON_SET`，`read(2)`将返回错误`ECANCELED`如果实时时钟接受不连续变化。如果标识不是`TFD_TIMER_CANCEL_ON_SET`，那么该始终
拒绝不连续变化(即`clock_settime(2)`可能导致`read(2)`非阻塞，返回0值，if the clock change occurs after the time expired,
but before the `read(2)` on the file descriptor)。

#### `poll(2), select(2)`...

