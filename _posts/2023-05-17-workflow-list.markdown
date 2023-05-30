---
layout: post
title: "workflow list"
date: 2023-05-17 15:12:00 +0800
categories: jekyll update
---

## 简介
list是workflow中使用的具有头结点的单向链表和双向链表。

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

#### `slist_del_afer`和`slist_del_head`
类似`slist_add_after`和`slist_add_head`

#### `__slist_splice`

### 双向链表 doubly linked list

#### 结点

```c++
struct list_head {
    struct list_head *next, *prev;
}
```

#### `INIT_LIST_HEAD` 头结点初始化

TODO
![init-list-head](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/)

#### `__list_add` `list_add`和`list_add_tail`

`__list_add`是通用添加节点的方式,`list_add`是特化在头结点后添加节点(即栈的模式)，`list_add_tail`是特化在末尾添加节点
(即队列的模式)。

TODO
![list-add](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/)
![list-add-tail](https://github.com/AHMASww/AHMASww.github.io/blob/master/_photos/)

#### `__list_del` `list_move`和`list_move_tail`

通用删除节点方式，删除前后两个节点之间的节点。
