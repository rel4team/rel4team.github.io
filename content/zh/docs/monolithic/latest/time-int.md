---
weight: 14
title: "aarch64 用户态定时器设计"
commentsId: 1 
---

# aarch64 定时器设计

为了实现这个支持需要在 sel4 中开启 KernelArmExportPCNTUser 和 KernelArmExportPTMRUser 支持，开启后可以在用户态访问 `cntp` 系列寄存器。

## 中断设计

在 rel4-linux-kit 中中断需要在绑定在 Notification 上，而 Notification 可以执行等待操作来等待中断的发生，这种方式需要在特定的地方等待中断的发生，但是会阻塞当前的程序，并不适合在用户态单线程的程序中，如果需要等待的地方就启动一个新的线程专用于等待又太过于奢侈。所以需要找到一种方式让中断操作能够异步的进行。

好在 sel4 支持将 Notification 绑定在 tcb 上，在线程等待 Endpoint 的时候，如果这个时候发生了中断，那么就会立刻返回，由于发生的是 Notification 所以 Message 中没有信息，消息在 badge 中分辨。在 rel4-linux-kit 中的设计是将一个专用于中断的 Notification 绑定在 TCB 上，然后将不同的中断注册在这个 Notification 上，利用不同的 badge 来区分。

```rust
        let (message, tid) = DEFAULT_SERVE_EP.recv(());
        match tid {
            u64::MAX => handle_timer(),
            _ => spawner
                .spawn_local(exception::waiting_and_handle(tid, message))
                .unwrap(),
        };
```

上述代码的设计就是将 Notification 绑定在 TCB 上的操作，当发生中断的时候 recv 接收到的就是 (message, badge)，这个时候就可以根据 badge 来区分是哪个中断， -1 就是时钟中断，当发生 ipc 的时候接收到的就是 (message, tid)，这里将 badge 作为 tid 来区分不同的任务。

## 设置定时器

在系统需要设计定时器的时候，需要考虑多个不同的程序同时定时的情况，如果多个程序都设置了等待的时间，那么就需要根据不同程序的定时时间来分配哪个程序最先被唤醒，所以需要先对等待的队列进行排序，将最先唤醒的程序放在队头。

```rust
/// 设置进程定时器
pub fn set_process_timer(pid: usize, next: Duration) {
    TIME_QUEUE.lock().push((next, TimerType::ITimer(pid)));
    TIME_QUEUE
        .lock()
        .sort_by(|(dura_a, ..), (dura_b, ..)| dura_a.cmp(dura_b));
    log::debug!(
        "set next timer curr: {:?}  next: {:?}",
        current_time(),
        next
    );
    set_timer(TIME_QUEUE.lock().first().unwrap().0);
}
```

当程序中断触发的时候需要就会就会调用 `handle_timer` 函数。

```rust

/// 处理时钟中断
///
///
pub fn handle_timer() {
    // 处理已经到时间的定时器
    let curr_time = current_time();

    TIME_QUEUE.lock().retain(|(duration, timer_ty)| {
        if curr_time >= *duration {
            match timer_ty {
                TimerType::WaitTime(_tid, waker) => waker.wake_by_ref(),
                TimerType::ITimer(pid) => handle_process_timer(curr_time, *pid),
            };
        }
        curr_time < *duration
    });

    // 设置下一个定时器
    let next = TIME_QUEUE
        .lock()
        .first()
        .map(|x| x.0)
        .unwrap_or(Duration::ZERO);
    set_timer(next);

    TIMER_IRQ_SLOT
        .cap::<sel4::cap_type::IrqHandler>()
        .irq_handler_ack()
        .unwrap();
}

```

在 `handle_timer` 函数中针对不同函数的定时情况进行区分处理。将已经完成的移除，然后设置下一个需要的定时器。