---
weight: 15
title: "sel4 ipc延迟回复机制"
commentsId: 1 
---

# sel4 ipc延迟回复机制

sel4 的进程之间通过 Endpoint 和 Notification 来通信，在 rel4-linux-kit 中的消息是同步的，因此当 client1 发送一个消息到 server 的时候，client1 会阻塞，而 server 此时 reply cap 的状态对应 client1，这个时候就无法接收第二个 client 发送的 ipc, 如果舍弃这个去等待第二个 client 的 ipc, 那么 client1 就会丢失 reply cap 进入无休止的等待。

而服务程序经常会遇到无法立即回复的情况（比如等待资源或完成任务），所以必须延迟回复。为了解决这个问题， rel4-linux-kit 中使用了 IpcSaver 机制并形成了一个 mod，这个机制利用了 sel4 可以将 reply cap 保存在一个指定的 slot 的机制，实现了简单的延迟回复机制。

## IPCSaver

```rust
/// 保存未能及时回复的 IPC
#[derive(Debug)]
pub struct IpcSaver {
    /// 等待队列
    queue: VecDeque<LeafSlot>,
    /// 闲置的 slot
    free_slots: Vec<LeafSlot>,
}
```

上述代码构建了一个简单的结构，第一个 field `queue` 是等待的队列，这个队列中每一个元素都是一个 reply cap。第二个 field `free_slots` 是回收的空 slot，reply cap 被使用后就会被清空，然后保存在这里以便下次使用。

```rust
    /// 保存一个调用者的回复能力
    pub fn save_caller(&mut self) -> Result<(), sel4::Error> {
        let slot = match self.free_slots.pop() {
            Some(slot) => slot,
            None => alloc_slot(),
        };
        slot.save_caller()?;
        self.queue.push_back(slot);
        Ok(())
    }
```

当程序需要保存 replier cap 的时候就调用上述的函数，在程序初始化的时候其中还没有 slot，这个时候就需要申请新的 slot 来用于保存，然后调用 sel4 中的 syscall 将 reply cap 保存在这个 slot 中。当程序完成某个条件的时候就会调用 `reply_one` 来实现回复。

```rust
    /// 回复一个 Endpoint
    ///
    /// - `msg`  [MessageInfo] 需要回复的消息
    pub fn reply_one(&mut self, msg: MessageInfo) -> Result<(), sel4::Error> {
        let reply_cap = self.queue.pop_front();

        if let Some(slot) = reply_cap {
            Endpoint::from(slot).send(msg);
            self.free_slots.push(slot);
        }
        Ok(())
    }
```

上述代码，就是程序在完成某个条件的时候进行回复的机制，虽然这里最后采用的是 Endpoint 对象的接口，但是事实上这里并不是一个真正的 endpoint, 而是一个 reply cap。

## 使用场景

这个使用场景多用于中断和异步相关的场景，比如单独的串口程序，有多个任务都在等待输入的时候。