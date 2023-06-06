---
layout: post
title: "workflow list"
date: 2023-05-17 15:12:00 +0800
categories: jekyll update
---

## 简介
list是workflow中使用的具有头结点的单向链表和双向链表。无论是单项链表还是双向链表都是只有指针而没有具体的数据项，实际
使用的时候都需要加上数据项，例如
```c++
struct test_node{
    struct slist_node;
    int test_node_data;
}
```

### 单向链表 Single-linked list

普通节点和头结点
```c++
struct slist_node {
    struct slist_node *next;
}

struct slist_head {
    struct slist_node first, *last;
}
```

#### `INIT_SLIST_HEAD`头结点初始化

![slist-init](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/slist_init.png)

#### `slist_add_after` `slist_add_head`和`slist_add_tail`
`slist_add_after`是一个通用函数，将node节点添加到prev节点后面。

![slist-add-after](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/slist_add_after.png)

`slist_add_head`是`slist_add_after`特例化，将node节点添加到head节点后面。

![slist-add-head](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/slist_add_head.png)

`slist_add_tail`将node节点添加到最后。似乎将`slist_add_tail`写成特例化应该也是可以:
```c++
static inline void slist_add_tail(struct slist_node *node
                                  struct slist_head *list)
{
    slist_add_after(node, list->last, list);
}
```

![slist-add-tail](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/slist_add_tail.png)

测试程序
```c++
#include <iostream>

#include "list.h"

// single list
struct myListNode
{
    struct slist_node node;
    int val;
    
    myListNode()
    {
        node.next = nullptr;
        val = -1;
    }

    myListNode(int _val)
    {
        node.next = nullptr;
        val = _val;
    }
};

void echoList(slist_head* head)
{
    slist_node* pos = nullptr;
    slist_for_each(pos, head)
        std::cout << ((myListNode*)pos)->val << " ";
    std::cout << std::endl;
}


int main(int argc, char** argv)
{
    slist_head head1, head2;
    INIT_SLIST_HEAD(&head1);
    INIT_SLIST_HEAD(&head2);

    // build head1
    for (int i = 0; i < 5; ++i)
    {
        myListNode* newNode = new myListNode(i);
        slist_add_head(&newNode->node, &head1);
    }
    // echo head1
    std::cout << "head1 : " << std::endl;
    echoList(&head1);

    // build head2
    for (int i = 5; i < 10; ++i)
    {
        myListNode* newNode = new myListNode(i);
        slist_add_tail(&newNode->node, &head2);
    }
    // echo head2
    std::cout << "head2 : " << std::endl;
    echoList(&head2);

    // TODO free memory

    return 0;
}
```

结果
```
head1 :
4 3 2 1 0
head2 :
5 6 7 8 9
```

#### `slist_del_afer`和`slist_del_head`
类似`slist_add_after`和`slist_add_head`

#### `__slist_splice`

TODO

### 双向链表 doubly linked list

#### 结点

```c++
struct list_head {
    struct list_head *next, *prev;
}
```

#### `INIT_LIST_HEAD` 头结点初始化

![init-list-head](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/list_init.png)

#### `__list_add` `list_add`和`list_add_tail`

`__list_add`是通用添加节点的方式,`list_add`是特化在头结点后添加节点(即栈的模式)，`list_add_tail`是特化在末尾添加节点
(即队列的模式)。

![list-add](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/list_add.png)
![list-add-tail](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/list_add_tail.png)

#### `__list_del` `list_move`和`list_move_tail`

通用删除节点方式，删除前后两个节点之间的节点。`list_move`移除一个节点并将其加入到另外一个队列头结点后；`list_move_tail`
移除一个节点并将其加入到另一个队列末尾。

#### `__list_splice` `list_splice` `list_splice_init`

TODO

#### `list_entry`

```c++
#define list_entry(prt, type, member) \
    (((type *)((char *)(ptr)-(unsigned long)(&((type *)0)->member)))
```
在`Executor`中有一处使用
```c++
struct ExecSessionEntry *entry;
...
entry = list_entry(queue->session_list.next, struct ExecSessionEntry, list);
```
