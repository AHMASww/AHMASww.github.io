---
layout: post
title: "Antlr4 C++ Visitor方法"
date: 2023-06-14 18:05:00 +0800
categories: jekyll update
---

# workflow Executor

Executor需要涉及到对线程池和链表的理解。

## `ExecQueue` `ExecSession` `Executor`

```c++
class ExecQueue
{
...
private:
    // 给类提供了列表的头节点
    struct list_head session_list;
    // 作为该队列操作的唯一mutex来提供锁
    pthread_mutex_t mutex;
...
};
```

```c++
class ExecSession
{
private:
    // 实际操作处理
    virtual void execute() = 0;
    // 状态处理
    virtual void handle(int state, int error) = 0;

private:
    // 列表
    ExecQueue *queue;
...
};
```

```c++
class Executor
{
...
public:
    // 任务提出
    int request(ExecSession *session, ExecQueue *queue);
...
};
```

## `Executor`函数理解

首先有一个重要的`struct`，是整个类串联起来的关键。

### `ExecSessionEntry`
```c++
struct ExecSessionEntry
{
    struct list_head list;
    ExecSession *session;
    thrdpool_t *thrdpool;
};
```

### `Executor::request`
```c++
int Executor::request(ExecSession *session, ExecQueue *queue)
{
    struct ExecSessionEntry *entry;

    session->queue = queue;
    // 开辟ExecSessionEntry空间
    entry = (struct ExecSessionEntry *)malloc(sizeof (struct ExecSessionEntry));
    if (entry)
    {
        entry->session = session;
        entry->thrdpool = this->thrdpool;
        pthread_mutex_lock(&queue->mutex);
        // 将entry挂载到queue末尾
        list_add_tail(&entry->list, &queue->session_list);
        // 如果queue下一个就是当前所挂载，则push到线程池中
        if (queue->session_list.next == &entry->list)
        {
            struct thrdpool_task task = {
                .routine = Executor::executor_thread_routine,
                .context = queue
            };
            if (thrdpool_schedule(&task, this->thrdpool) < 0)
            {
                list_del(&entry->list);
                free(entry);
                entry = NULL;
            }
        }

        pthread_mutex_unlock(&queue->mutex);
    }

    return -!entry;
}
```

### `Executor::executor_thread_routine`

```c++
void Executor::executor_thread_routine(void *context)
{
    ExecQueue *queue = (ExecQueue *)context;
    struct ExecSessionEntry *entry;
    ExecSession *session;
    
    pthread_mutex_lock(&queue->mutex);
    // queue->session_list.nexts实际上就是ExecSessionEntry的list
    entry = list_entry(queue->session_list.next, struct ExecSessionEntry, list);
    // 去除当前节点
    list_del(&entry->list);
    // 获取要执行的session
    session = entry->session;
    // 判断链表上是否还有节点，有的话继续放入到线程池中
    if (!list_empty(&queue->session_list))
    {
        struct thrdpool_task task = {
            .routine = Executor::executor_thread_routine,
            .context = queue
        };
        __thrdpool_schedule(&task, entry, entry->thrdpool);
    }
    else
        free(entry);
    
    pthread_mutex_unlock(&queue->mutex);
    // 实际任务执行
    session->execute();
    // 状态处理
    session->handle(ES_STATE_FINISHED, 0);
}
```

### `Executor::executor_cancel`

```c++
void Executor::executor_cancel(const struct thrdpool_task *task)
{
    ExecQueue *queue = (ExecQueue *)task->context;
    struct ExecSessionEntry *entry;
    struct list_head *pos, *tmp;
    ExecSession *session;

    // 遍历当前链表
    list_for_each_safe(pos, tmp, &queue->session_list)
    {
        // 找到每个任务的入口
        entry = list_entry(pos, struct ExecSessionEntry, list);
        // 删除当前节点
        list_del(pos);
        session = entry->session;
        free(entry);
        // 状态处理
        session->handle(ES_STATE_CANCELED, 0);
    }
}
```

