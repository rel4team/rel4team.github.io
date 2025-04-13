+++
weight = 20
date = '2025-01-25T09:47:15+08:00'
title = 'CMake 兼容性设计'
+++


## 1. 简介

为了方便后续开发和对比，我实现了 reL4 Kernel 新旧模式的兼容。简单来说，通过一个 CMake Option 选择编译哪种模式，默认还是使用旧模式 (reL4 作为 lib)。这样的好处是，在 **纯 rust** 模式移植过程中，可以不断的 merge 进 master，而无需等所有功能，所有 CPU 架构相关代码都移植完成后才能合入，后者显然需要一个非常长的时间。

两种模式会并行一段时间，Pure Rust 模式会逐渐实现所有功能，等完成所有功能移植并测试稳定后，会将旧模式废弃。

下面介绍下对编译系统的一些改动

## 2. 编译命令
```
# 在 rel4test 中
# 编译旧模式没有变化
cd rel4_kernel
cargo xtask build -p spike

# kernel.elf 在 build/kernel 路径下

# 编译新模式加一个参数
cargo xtask build -p spike --bin

# kernel.elf 在 build/rel4_kernel 下

# 最后生成的 simulate 文件是完全一样的
```

## 3. 兼容性设计

如上所说，为了兼容新旧模式，主要修改了以下部分

### 3.1 ReL4 Kernel 中增加一个 CMakeLists.txt

这个 CMakeLists 相当于一个 compile wrapper ，将 Cargo 编译产物接入到 seL4 CMake 编译系统中。除了 kernel，我们不想改变 seL4 原生的其他项目，因此在编译整个镜像 （opensbi + kernel + userspace）时，仍然使用 seL4 提供的编译系统。

这样做的好处是，只替换了 kernel，可以证明 reL4 能够实现 seL4 kernel 的功能。

这个 CMakeList 重要内容如下

```
# rel4 编译命令
add_custom_command(
    OUTPUT ${CMAKE_BINARY_DIR}/reL4/kernel.elf
    COMMAND ${CMAKE_OBJCOPY} --remove-section=.riscv.attributes ${KERNEL_ELF_PATH} ${CMAKE_BINARY_DIR}/reL4/kernel.elf
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMENT "Build and prepare reL4 kernel.elf"
)

# rel4 编译 target，依赖 kernel.elf 会触发上面的 cmake custom command
add_custom_target(
    build_reL4
    DEPENDS ${CMAKE_BINARY_DIR}/reL4/kernel.elf
)

# rel4 编译 target wrapper，供其他函数调用 reL4_kernel.elf
add_executable(reL4_kernel.elf IMPORTED GLOBAL)
set_target_properties(reL4_kernel.elf PROPERTIES
    IMPORTED_LOCATION ${CMAKE_BINARY_DIR}/reL4/kernel.elf
)
```


### 3.2 sel4test 中引用 rel4_kernel CMake 模块

重要的地方是，如果有 REL4_KERNEL 这个编译选项，就引入 reL4 module

```
if(REL4_KERNEL)
    find_package(reL4 REQUIRED)
endif()

if(REL4_KERNEL)
    rel4_import_kernel()
endif()
```


### 3.3 elfloader 里实现新旧模式兼容

elfloader 将所有 elf 文件打包成一个完整镜像，我们只替换 kernel elf，opensbi 和 userspace elf 文件保持不变，这样才可以验证 reL4 kernel 实现了对 seL4 kernel 的替代。

在下面所示的地方，同样根据 REL4_KERNEL 编译选项，选择使用旧模式生成的 kernel elf 还是新模式生成的

```
if(REL4_KERNEL)
    list(APPEND cpio_files "$<TARGET_FILE:reL4_kernel.elf>")
else()
    list(APPEND cpio_files "$<TARGET_FILE:kernel.elf>")
endif()
```

这就是为什么我们要在 reL4 CMakeList 中加一个 **rel4 编译 target wrapper**，就是供 **$<TARGET_FILE:reL4_kernel.elf>** 调用的，这样可以很方便的找到 kernel.elf 文件路径，无需写一个固定路径。

## 4. 一些说明

### 4.1 为什么还保留 seL4 kernel

虽然我们不使用任何 seL4 kernel 的 C 和汇编代码，但是我们仍然还依赖 seL4 kernel 仓库中的两个东西

- libseL4，这个是供用户态 sel4test 使用的，所以我们也不可能用 rust 实现，但是它又被放在 seL4 kernel 仓库中
- CMake 脚本，目前还依赖 kernel 中的 cmake 脚本，生成一堆宏定义，供 sel4test 和 libseL4 使用。本质上 reL4 kernel 是可以不依赖的，但是目前为了省事，也用了一点 seL4 kernel CMake 生成的宏定义。例如 KernelHaveFPU KernelIsMCS 等，但这部分很少且可控，后续去除也很容易。

后续看是否需要将 seL4 kernel 中有用的 CMakeList 和 libseL4 剥离出来。
