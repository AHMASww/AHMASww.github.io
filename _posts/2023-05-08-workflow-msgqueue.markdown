---
layout: post
title: "workflow msgqueue 个人理解"
date: 2023-05-08 17:53:00 +0800
categories: jekyll update
---

### 简介
msgqueue是workflow中用于机器资源协调调度的队列实现，完整的介绍可以参照[消息队列新实现：Workflow msgqueue代码详解](
https://zhuanlan.zhihu.com/p/525985268)，
以下主要列出的是个人在理解过程的经历的一些疑惑。

### `msgqueue_create()`
```c++
msgqueue_t *msgqueue_create(size_t maxlen, int linkoff)
{
    // 各种初始化，最后设置queue的成员变量如下：
    queue->msg_max = maxlen;
    queue->linkoff = linkoff;
    queue->head1 = NULL;
    queue->head2 = NULL;
    // 借助两个head分别作为两个内部队列的位置
    queue->get_head = &queue->head1;
    queue->put_head = &queue->head2;
    // 一开始队列为空，所以生产者队尾也等于队头
    queue->put_tail = &queue->head2;
    queue->msg_cnt = 0;
    queue->nonblock = 0;
    ...
}
```

这里的疑惑在于`queue->get_head``queue->put_head``queue->put_tail`对`NULL`取地址。在C中,`NULL`被定义为0，这就意味着
`queue->head1 = 0``queue->head2 = 0`，但是内存中也要有地址来存储这个`0`值,并且这两个`NULL`的地址是不一样的。

测试程序:
```c++
int main(int argc, char** argv)
{
    void* head1 = NULL;
    void* head2 = NULL;

    printf("address of head1 : %p\naddress of head2 : %p\n", &head1, &head2);

    return 0;
}
```
结果:
```
address of head1 : 0x7ffe9eaf3608
address of head2 : 0x7ffe9eaf3610
```

### `msgqueue_put()`

```c++
void msgqueue_put(void *msg, msgqueue_t *queue)
{
    // 1. 通过create的时候传进来的linkoffset，算出消息尾部的偏移量
    void **link = (void **)((char *)msg + queue->linkoff);

    // 2. 设置为空，用于表示生产者队列末尾的后面没有其他数据
    *link = NULL;
    // 3. 加生产者锁
    pthread_mutex_lock(&queue->put_mutex);
    // 4. 如果当前已经有msg_max个消息的话
    //    就要等待消费者通过put_cond来叫醒我
    while (queue->msg_cnt > queue->msg_max - 1 && !queue->nonblock)                
        pthread_cond_wait(&queue->put_cond, &queue->put_mutex);

    // 5. put_tail指向这条消息尾部，维护生产者队列的消息个数
    *queue->put_tail = link;
    queue->put_tail = link;
    queue->msg_cnt++;
    pthread_mutex_unlock(&queue->put_mutex);
    // 6. 如果有消费者在等，通过get_cond叫醒他～
    pthread_cond_signal(&queue->get_cond);
} 
```

#### `void **link = (void **)((char *)msg + queue->linkoff);`
`link`是一个二级指针，将`(char*)msg + queue->linkoff`转为二级指针，实际值还是一个地址，即`*link`指向的应该是数据的
末尾，这也是后面`msg = (char*)queue->get_head - queue->linkoff`可以得到原本消息的原因。

#### `*link = NULL;`
`*link`实际上指的就是数据的末尾预留的指针的位置，作为`next`含义，如果该值不为`NULL`，则是下一个`msg`的末尾预留指针
的位置。

#### `*queue->put_tail = link;`
#### `queue->put_tail = link;`
`queue->put_tail`是一个二级指针,存的是指向`queue->head2`的地址(初始情况)，`*queue->put_tail`就等于是`queue->head2`，
这句话在开始时就可以看做`queue->head2 = link`，即队列的末尾指向下一个数据的预留指针的位置，这就是开发者所说的类似
`queue_tail->next = msg`的含义。
`queue->put_tail = link`是将二级指针的值存储为`link`的值，`*queue->put_tail`指向的就是最后一个数据预留指针的位置,
这也就是开发者所说的类似`queue_tail = msg`的含义。
综上所述`**put_tail`实现的一个链表结构。

### `signal/broadcast`在持有锁和不持有锁的情况下
在`msgqueue_put()`中`signal`在不持有锁的情况下唤醒线程，在`__msgqueue_swap()`中是在持有锁的情况下唤醒线程的，按照个
人理解，先释放锁可能使得其他线程获得锁，而有条件锁的线程会进一步等待，直到获得其他线程释放的锁。似乎在可预测性上比较
困难。

