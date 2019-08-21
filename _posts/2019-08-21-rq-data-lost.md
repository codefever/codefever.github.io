---
title: 从源码角度看rq会丢任务吗？
tags:
  - rq
  - web开发
  - 系统设计
---

> 在web开发中，对于一些繁重的计算任务，我们一般不会将其放在页面请求处理的主干流程中，而是通过异步任务转化为异步计算，这个过程一般会用[rq](https://github.com/rq/rq)。
> 本文从源码角度看使用rq会不会出现丢任务的情况……

## rq client侧异步任务入队
相关函数：
* [Queue.enqueue_job](https://github.com/rq/rq/blob/master/rq/queue.py#L339-L367)
* [Queue.push_job_id](https://github.com/rq/rq/blob/master/rq/queue.py#L214-L221)

```python
def push_job_id(self, job_id, pipeline=None, at_front=False):
        """Pushes a job ID on the corresponding Redis queue.
        'at_front' allows you to push the job onto the front instead of the back of the queue"""
        connection = pipeline if pipeline is not None else self.connection
        if at_front:
            connection.lpush(self.key, job_id)
        else:
            connection.rpush(self.key, job_id)
```

一个异步任务`job`被创建之后经过各种`enqueue_*`的调用，最后会落到`push_job_id`的地方。可以看到，这里使用了redis-list的lpush（或rpush）完成入队。

## rq worker侧异步任务出队
相关函数：
* [Queue.dequeue_any](https://github.com/rq/rq/blob/master/rq/queue.py#L453-L489)
* [Queue.lpop](https://github.com/rq/rq/blob/master/rq/queue.py#L423-L451)

在worker侧之前的异步任务`job`通过`dequeue_any`出队，最后调用到Queue上实现的`lpop`。

```python
    @classmethod
    def lpop(cls, queue_keys, timeout, connection=None):
        """Helper method.  Intermediate method to abstract away from some
        Redis API details, where LPOP accepts only a single key, whereas BLPOP
        accepts multiple.  So if we want the non-blocking LPOP, we need to
        iterate over all queues, do individual LPOPs, and return the result.
        Until Redis receives a specific method for this, we'll have to wrap it
        this way.
        The timeout parameter is interpreted as follows:
            None - non-blocking (return immediately)
             > 0 - maximum number of seconds to block
        """
        connection = resolve_connection(connection)
        if timeout is not None:  # blocking variant
            if timeout == 0:
                raise ValueError('RQ does not support indefinite timeouts. Please pick a timeout value > 0')
            result = connection.blpop(queue_keys, timeout)
            if result is None:
                raise DequeueTimeout(timeout, queue_keys)
            queue_key, job_id = result
            return queue_key, job_id
        else:  # non-blocking variant
            for queue_key in queue_keys:
                blob = connection.lpop(queue_key)
                if blob is not None:
                    return queue_key, blob
            return None
```

这里可以看到，出队逻辑直接使用redis-list的`lpop`。这样会有一个潜在的问题，如果redis服务收到`lpop`命令，完成出队操作并已经要回包了，此时若网络出现问题或者worker重启，这个任务的出队将永远没法到达worker。这里的主要原因是，redis在lpop的结果达到调用方前已经永久地改变了自身的持久化状态。

从任务或消息队列的设计角度来看，redis-list并不能确保数据能可靠地到达客户端，反倒是像[kafka](https://kafka.apache.org)那样的，通过ack机制将数据的回包和消费解耦开来，能更好地应对类似这样的情况。

## 最后
天知道我经历过什么才知道这个！

如果想避免这种情况，可以尝试使用celery并配合[acks_late=true](https://docs.celeryproject.org/en/latest/reference/celery.app.task.html#celery.app.task.Task.acks_late)特性，这样只有当任务完成才会完全清理，否则会被延后重试，当然这样做的话也需要任务计算是符合[幂等性](https://en.wikipedia.org/wiki/Idempotence)才能避免重试带来的副作用。