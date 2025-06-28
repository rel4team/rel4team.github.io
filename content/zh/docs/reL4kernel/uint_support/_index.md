---
weight: 4
bookCollapseSection: true
title: "在rel4中添加用户态中断支持"
commentsId: 1 
---

# Riscv-N扩展学习

参考资料：
 - [廖东海的提交](https://github.com/rel4team/rel4_kernel/commit/2c50e6fed1ba4dd56c6be06b27e3e36c5082d623#diff-20a770e31a3c0cf5778a11019c3dcb98ee3e690b4946b833a457217241caf3aa)
 - [riscv-N扩展](https://gallium70.github.io/rv-n-ext-impl/ch1_1_priv_and_trap.html)

## 标准实施情况
1. N扩展草案
RISC-V 特权级指令规范 v1.11 版本中存在 N 扩展，但是在 RISC-V 规范 v1.12 版本中，N 扩展被移除。

2. CLIC扩展草案
N扩展还需要考虑各类外设中断，CLIC扩展是支持这个的

## 寄存器情况和代码实现阅读

包括如下的寄存器：
 - ustatus
 bit0 - UIE：用户态中断使能位，当其设为1的时候才能被启用
 
 bit4 - UPIE：跟SPIE很像，在中断发生的时候需要保存原先的中断使能情况，不需要UPP位（SPP指示之前的状态位）
 
 需要做的操作：在init的时候必须置位ustatus的bit0，在uret的时候会自动管理UPIE，不过，如果需要嵌套，我认为也需要做保存操作
 
 - utvec
 跟stvec相同，中断向量，最后两位都是mode位（因为需要对齐到4字节），0表示direct模式，而1表示vector模式

 - uip与uie
 同样的类似sip和sie。
 
 uip存储待处理的中断信息，包括
 
 bit0 - USIP 软件中断
 
 bit4 - UTIP 时钟中断
 
 bit8 - UEIP 外部中断

 在廖的实现中，他写了个sip.rs的文件，里面的实现也包括了sip的中断和部分uip的中断（但是没有uip的文件，应该是因为，这个文件包含了sip和uip的共同代码）

 - sedeleg与sideleg

 只有委托给s态的对应位才可以委托给U态。
 
 对于这两个寄存器，尤其是sideleg寄存器，这里面涉及到上面uip所需要的三个中断，因此，如果需要在用户态处理这些寄存器，我认为有两种方式，1，直接委托给用户态，2，不委托给用户态，但是在内核态接到相关中断的时候，可以允许它写入UIP对应的位。

 在廖的代码实现中，对这两个寄存器的操作进行了一些封装。包括sedeleg的寄存器的对应异常相关位的读和写
 
 不过，在sideleg的设置中，廖的代码只设置了写入软件中断和外部中断的实现（时钟中断应该给内核了吧）



 - ucause寄存器
 中断发生时写入ucause，最高位是interrupt位，置位了表示是1，后面跟着exception code，其中interrupt置位的时候，0,4,8分别表示用户态的软件，时钟和外部中断等内容
 而不置位的时候则表明包含了一大堆的用户态的异常

 - uepc
 记录中断异常发生的时候的指令。
 - uscratch
 没有额外的说明，估计就是用于类似于sscratch的缓存作用的
 - utval
 这个寄存器跟标准有关，会写入一些指令/加载/存储地址未对齐/访问错误/页错误异常产生时，导致错误的虚拟地址等内容。

 - uret指令
 将pc设置为uepc，uie设置为upie

 - uipi指令
 在廖的实现中，包括五个接口：

 uipi_send

 uipi_read

 uipi_write

 uipi_activate

 uipi_deactivate

 - 其他
 另外，在廖的实现中，还有三个寄存器，suicfg（0x1B2），suirs（0x1B1），suist（0x1B0）
 我不太确定这几个寄存器的具体功能（TODO）



### 其他代码

在boot的时候需要启动uint，
```
#[cfg(feature = "ENABLE_UINTC")]
crate::uintc::init();
```

在进入和restore的时候需要考虑uint
restore
（在fastpath restore中也需要这么做）
```
crate::uintc::uintr_return();
```

进入的时候需要
```
crate::uintc::uintr_save();
```

在decode invocation中对Notification的cap的处理，也增加了
```
#[cfg(feature = "ENABLE_UINTC")] {
    if get_currenct_thread().uintr_inner.uist.is_none() {
        crate::uintc::regiser_sender(cap);
        }
    }
```

在invoke_tcb_bind_notification中增加了
```
#[cfg(feature = "ENABLE_UINTC")]
{
    tcb.uintr_inner.utvec = crate::uintr::utvec::read().bits();
    tcb.uintr_inner.uscratch = crate::uintr::uscratch::read();
}
```

#### 对于Notification的cap的更改
额，这扯到一个代码的历史原因，简单来说，下面的代码描述方式，是曾经使用的方法，但是会存在比如字段填充错误导致的问题，因此已经弃用，改为，从pbf从头开始解析的方式。
```
从
plus_define_bitfield! {
    notification_t, 4, 0, 0, 0 => {
        new, 0 => {
            bound_tcb, get_bound_tcb, set_bound_tcb, 3, 0, 39, 0, true,
            msg_identifier, get_msg_identifier, set_msg_identifier, 2, 0, 64, 0, false,
            queue_head, get_queue_head, set_queue_head, 1, 0, 39, 0, true,
            queue_tail, get_queue_tail, set_queue_tail, 0, 25, 39, 0, true,
            state, get_usize_state, set_state, 0, 0, 2, 0, false
        }
    }
}
改到了
plus_define_bitfield! {
    notification_t, 5, 0, 0, 0 => {
        new, 0 => {
            bound_tcb, get_bound_tcb, set_bound_tcb, 3, 0, 39, 0, true,
            msg_identifier, get_msg_identifier, set_msg_identifier, 2, 0, 64, 0, false,
            queue_head, get_queue_head, set_queue_head, 1, 0, 39, 0, true,
            queue_tail, get_queue_tail, set_queue_tail, 0, 25, 39, 0, true,
            state, get_usize_state, set_state, 0, 0, 2, 0, false,
            uintr_flag, get_uintr_flag, set_uintr_flag, 4, 9, 1, 0, false,
            recv_idx, get_recv_idx, set_recv_idx, 4, 0, 9, 0, false
        }
    }
}
```
在这段代码中，首先是增加了Notification_t的这个长度，从4->5
然后在最后增加了uint_flag和ricv_idx的字段。
从逻辑上来说，如果要加入uintr，这个工作应该避免不了，因为uint作为一个内核相关的对象，肯定也需要额外的cap来进行管理。

但是如果可以，我想更加期望的方式不是使用各类config弄得到处都是，而是使用模块化的方法去完成

#### 对于tcb的更改
这个commit中，对于tcb中，增加了相关的字段和对应的结构体
```
pub uintr_inner:uintr_tcb_inner;

pub struct uintr_tcb_inner {
	pub uepc: usize;
	pub utvec: usize;
	pub uscratch: usize;
	pub uist: Option<usize>
}
```
#### 常量
这部分需要在rel4 config中去增加,不过也不绝对，上述是基于原先廖的设计的代码





