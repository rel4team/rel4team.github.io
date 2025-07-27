---
weight: 10
bookCollapseSection: false
title: "Slot 管理"
commentsId: 1 
---

# Slot 管理

要讨论 Capability 管理，就不可避免的要接触 CSpace, 而每一个 Cap 都会放在 Slot 中，每个 Slot 仅能放置一个 Cap。对于 Cap 的管理本质上就是针对 Slot 进行管理。

## Slot 操作

Slot 管理需要下面的基本功能：

- 申请 Slot
- 回收 Slot
- 清空 Slot
- 从一个 Slot 复制 Cap
- 获取 Slot 中的 Cap
- 针对 Cap 的 revoke, mint 等操作

## CSpace 分级管理

Slot 查找的过程和虚拟地址翻译的过程非常相似，都是根据一个地址中的部分 bit 来一级一级的翻译，在 rel4-linux-kit 中使用了两级 Cnode, 每一级 占 12 bit，也就是目前最高能有 12^12 个 slot。

```plain
+-----------+-----------+---------+
|  63..24   |  23..12   |  11..0  |
+-----------+-----------+---------+
| Not Used  |  Level 1  | Level 0 |
+-----------+-----------+---------+
```

## Slot 申请

在程序进行初始化的时候会将所有可用的 Slot 的范围传入 `SlotManager` 进行初始化，在需要申请 Cap 或者从某个 Cap 中复制 Cap 的时候，首先会申请一个 Slot, rel4-linux-kit 中将这个可用的 Slot 命名为 `LeafSlot`，意为在最后一级的 Slot，在空白的 slot 中可以放置申请或者复制的 Slot。

## Slot 释放

由于一个 slot 中可以只放置一个 Cap，所以当一个 Cap 生命周期走向结束的时候，就可以进行将这个 LeafSlot 释放，最终进行重新利用。在下次申请 Slot 的时候会首先在回收池中进行查找，如果有可用的 Slot，则将这个 Slot 分配出去，而不必在 SlotManager 中减少可分配的范围。

## 边缘处理

由于采用了两级 Slot 管理，但是提前将所有的 Slot 都分配好，就需要 1 + 12 = 13 个 2 ^ 12 空间的 CNode，所以直接放置会消耗许多内存，因此在 rel4-linux-kit 中采用了动态分配的方式，在初始化的时候仅会分配 2 个 CNode，将前 4096 个空间安排，在申请到 4096+ 的时候再进行扩展，在 rel4-linux-kit 中的做法是在遇到 n % 4096 = 0 的时候会将一个 CNode 放在下一个 23..12 的位置，扩展出 4096 个可用的 Slot，这种方式目前仅限于 23..12，并不会影响最大可用大小。


## 未来扩展

1. 更多层的 CSpace，扩展为更大的空间
2. 在需要扩展 CSpace 的时候再进行扩展，使用的级别动态扩展：1,2,3,4,5