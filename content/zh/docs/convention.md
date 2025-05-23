
---
weight: 201
bookCollapseSection: false
bookFlatSection: true
title: "reL4 编程公约草案"
---

# reL4 编程公约草案

## 🧭 命名规范（Naming Conventions）

| 类型                | 命名风格               | 示例                     |
| ------------------- | ---------------------- | ------------------------ |
| 变量 / 函数         | `snake_case`           | `calculate_sum`          |
| 常量 / 静态变量     | `SCREAMING_SNAKE_CASE` | `MAX_BUFFER_SIZE`        |
| 类型 / 枚举 / Trait | `CamelCase`            | `HttpRequest`, `Display` |
| 生命周期标识符      | `'lowercase`           | `'a`, `'static`          |
| 模块名              | `snake_case`           | `utils`, `io_helper`     |
| Feature             | `snake_case`           | `smp`, `mcs`, `have_cpu` |

---

## 🧼 代码风格（Code Style）

- 每行不超过 **100 字符**
- 使用 `rustfmt` 格式化代码：
```sh
  rustup component add rustfmt
  cargo fmt
```

## 🔍 错误处理 (Error Handling)

- 使用 Result<T, E> 处理可预期错误
- 使用 ? 操作符传播错误
- 避免使用 unwrap() / expect()，除非确定无误

## 📦 模块与 Crate 结构（Project Structure）

- 使用 `workspace` 拆分大型项目
- 每个模块职责单一，避免循环依赖
- `lib.rs` 只作为入口，不承载业务逻辑，`main.rs` 中相关逻辑

## 🧰 工具建议（Tooling）

- `rustfmt`	自动格式化代码
- `clippy`	静态分析和潜在错误提示

## 📝 文档规范（Documentation）

- 使用 /// 注释 public API
- 使用 //! 注释 crate 或模块入口
- 添加 # Examples，自动作为测试用例

```
/// 获取两个数字之和
///
/// # Examples
///
/// ```
/// assert_eq!(my_crate::add(2, 3), 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

## 适当使用抽象

> 草案

- 对于不需要加锁的场景适当使用抽象，例如启动核专属的变量，就可以使用一个 struct 来定义, e.g. `BootCore<usize>`
- 对于 `unsafe` 代码进行收集，减少 `unsafe` 代码的分布
- 减少直接指针的使用，使用一个类型或者抽象进行计算
- 使用 `cargo clippy` 检测语法的异常
- 对于确认无误的代码，在 mod 或 crate 上加入 `#![deny(warnings)]` 和 `#![deny(missing_cos)]`，确保后续代码不出现问题
