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
    ...
    // slist_del_after
    slist_node* pos = nullptr;
    slist_node* prev = nullptr;
    slist_for_each_safe(pos, prev, &head1)
    {
        if (((myListNode*)pos)->val == 2)
            slist_del_after(prev, &head1);
    }
    // echo head1
    std::cout << "slist_del_after head1 : " << std::endl;
    echoList(&head1);

    // slist_del_head
    slist_del_head(&head2);
    // echo head2
    std::cout << "slist_del_head head2 : " << std::endl;
    echoList(&head2);
    
    ...
}
```
结果
```
head1 : 
4 3 2 1 0 
head2 : 
5 6 7 8 9 
slist_del_after head1 : 
4 3 1 0 
slist_del_head head2 : 
6 7 8 9 
```

#### `__slist_splice` `slist_splice`

`slist_splice`调用`__slist_splice`，合并两个链表，合并后以head为头结点。

测试程序
```c++
    // slist_splice
    slist_splice(&head1, &head2.first, &head2);
    //echo head2
    std::cout << "slist_splice head1 head2 : " << std::endl;
    echoList(&head2);
```
结果
```
slist_splice head1 head2 : 
4 3 1 0 6 7 8 9 
```

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

测试程序
```c++
struct myDoubleLinkedListNode
{
    struct list_head node;
    int val;

    myDoubleLinkedListNode()
    {
        INIT_LIST_HEAD(&node);
        val = -1;
    }

    myDoubleLinkedListNode(int _val)
    {
        INIT_LIST_HEAD(&node);
        val = _val;
    }
};

void echoDoubleLinkedList(list_head* head)
{
    list_head* pos = nullptr;
    list_for_each(pos, head)
        std::cout << ((myDoubleLinkedListNode*)pos)->val << " ";
    std::cout << std::endl;
}


int main(int argc, char** argv)
{
    ...

    list_head dhead1;
    list_head dhead2;
    INIT_LIST_HEAD(&dhead1);
    INIT_LIST_HEAD(&dhead2);

    // list_add
    for (int i = 0; i < 5; ++i)
    {
        myDoubleLinkedListNode* newNode = new myDoubleLinkedListNode(i);
        list_add(&newNode->node, &dhead1);
    }
    // echo dhead1
    std::cout << "double linked list head1 : " << std::endl;
    echoDoubleLinkedList(&dhead1);

    // list_add_tail
    for (int i = 5; i < 10; ++i)
    {
        myDoubleLinkedListNode* newNode = new myDoubleLinkedListNode(i);
        list_add_tail(&newNode->node, &dhead2);
    }

    // echo dhead2
    std::cout << "double linked list head2 : " << std::endl;
    echoDoubleLinkedList(&dhead2);

    ...

    return 0;
}

```
结果
```
double linked list head1 : 
4 3 2 1 0 
double linked list head2 : 
5 6 7 8 9 
```

#### `list_del` `list_move`和`list_move_tail`

`list_del`是通用删除节点方式，删除前后两个节点之间的节点。`list_move`移除一个节点并将其加入到另外一个队列头结点后；`list_move_tail`
移除一个节点并将其加入到另一个队列末尾。

测试程序
```c++
    // list_del
    list_head* dpos = nullptr;
    list_head* dn = nullptr;
    list_for_each_safe(dpos, dn, &dhead1)
    {
        if (((myDoubleLinkedListNode*)dpos)->val == 2)
            list_del(dpos);
    }
    // echo dhead1
    std::cout << "double linked list head1 : " << std::endl;
    echoDoubleLinkedList(&dhead1);
    std::cout << "------------------------------------" << std::endl;

    // list_move
    dpos = nullptr;
    dn = nullptr;
    list_for_each_safe(dpos, dn, &dhead1)
    {
        if (((myDoubleLinkedListNode*)dpos)->val == 3)
            list_move(dpos, &dhead2);
    }
    // echo dhead1
    std::cout << "double linked list head1 : " << std::endl;
    echoDoubleLinkedList(&dhead1);
    // echo dhead2
    std::cout << "double linked list head2 : " << std::endl;
    echoDoubleLinkedList(&dhead2);
    std::cout << "------------------------------------" << std::endl;

    // list_move_tail
    dpos = nullptr;
    dn = nullptr;
    list_for_each_safe(dpos, dn, &dhead1)
    {
        if (((myDoubleLinkedListNode*)dpos)->val == 1)
            list_move_tail(dpos, &dhead2);
    }
    // echo dhead1
    std::cout << "double linked list head1 : " << std::endl;
    echoDoubleLinkedList(&dhead1);
    // echo dhead2
    std::cout << "double linked list head2 : " << std::endl;
    echoDoubleLinkedList(&dhead2);
    std::cout << "------------------------------------" << std::endl;

```
结果
```
------------------------------------
double linked list head1 : 
4 1 0 
double linked list head2 : 
3 5 6 7 8 9 
------------------------------------
double linked list head1 : 
4 0 
double linked list head2 : 
3 5 6 7 8 9 1 
------------------------------------
```

#### `__list_splice` `list_splice` `list_splice_init`

`list_splice_init`调用`__list_splice`和`list_splice`，和单链表类似，同样是合并链表。

测试程序
```c++
```
结果
```
```

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

`struct`内成员地址是连续的。

这里首先`queue->session_list`在传入时实际也是`ExecSessionEntry`，所以才能做地址转换；其次`(unsigned long)(&((type*)0)
->member)))`这里是计算出该`member`之前所有成员所占的地址，从而计算出整个指针的首地址，才能做后面的地址转换。
