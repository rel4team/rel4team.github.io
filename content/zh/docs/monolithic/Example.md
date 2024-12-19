---
title: "实例和测试"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 1. 网络模块的本地测试

> 相关 commit 为：[网络模块实现](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/commit/aae140336bd3ecef54fc1943f4223b220289f1f0)

目前网络模块的初始化代码如下：

```rust
#[export_name = "_start"]
fn main(ipc_buffer: IPCBuffer) -> sel4::Result<!> {
    static LOGGER: sel4_logging::Logger = sel4_logging::LoggerBuilder::const_default()
        .write(|s| sel4::debug_print!("{}", s))
        .level_filter(log::LevelFilter::Trace)
        .fmt(fmt_with_module)
        .build();
    LOGGER.set().unwrap();
    set_ipc_buffer(ipc_buffer);
    debug_println!("[Net Thread] Net-Thread");

    let virtio_net = VirtIoNetDev::<VirtIoHalImpl, MmioTransport, 32>::try_new(unsafe {
        MmioTransport::new(NonNull::new(VIRTIO_NET_ADDR as *mut VirtIOHeader).unwrap()).unwrap()
    })
    .expect("failed to create net driver");

    smoltcp_impl::init(virtio_net);
  
    // Modify here
    ipc::run_ipc();

    unreachable!()
}
```



我们提供了两个测试程序，分别测试作为 client 和 server 的表现，相关代码位于 [test_net](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/blob/main/crates/net-thread/src/smoltcp_impl/test.rs)

- client

  测试代码如下：

  ```rust
  fn test_client() {
      const REQUEST: &str = "\
      GET / HTTP/1.1\r\n\
      Host: ident.me\r\n\
      Accept: */*\r\n\
      \r\n";
  
      let tcp_socket = TcpSocket::new();
      tcp_socket
          .connect(SocketAddr::new(
              IpAddr::V4(Ipv4Addr::new(49, 12, 234, 183)),
              80,
          ))
          .unwrap();
      let request_buf = REQUEST.as_bytes();
      tcp_socket.send(request_buf).unwrap();
      let mut response_buf = [0; 256];
      let cnt = tcp_socket.recv(&mut response_buf).unwrap();
      let response = core::str::from_utf8(&response_buf[..cnt]).unwrap();
      debug_println!("response: {:?}", response);
  }
  ```

  运行方式：在上文提到的网络模块初始化代码中，将 ipc 入口修改为 test_client 的入口，即修改为：

  ```rust
  #[export_name = "_start"]
  fn main(ipc_buffer: IPCBuffer) -> sel4::Result<!> {
      xxx
      // Modify here
      // ipc::run_ipc();
      smoltcp_impl::test::test_client();
      unreachable!()
  }
  ```

  然后运行

  ```shell
  $ make -C docker/ exec ARCH=aarch64
  ```

  

- Server

  测试代码同上，将 IPC 入口修改为 test_server 入口即可。在运行内核之后，我们会在主机的 6379 号端口上启动 qemu，并且监听来自外部的网络请求。此时可以另开一个终端，运行如下指令：

  ```shell
  $ echo "Hello, World!" | nc localhost 6379
  ```

  即可观测到微内核的 server 接收到了请求，并且做出了响应。



## 2. 独立宏内核测试

> 相关 commit 为：[网络模块实现](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/commit/aae140336bd3ecef54fc1943f4223b220289f1f0)

目前的宏内核用户程序进行了对应的网络测试，详见 [test-thread](https://github.com/Azure-stars/rust-root-task-demo-mi-dev/tree/main/crates/test-thread/src)。在这里他通过 vsyscall handler 新建了一个 socket，并且按照 syscall 的语义，开始监听指定的端口并响应对应的请求。即我们利用宏内核的 syscall 建立了一个 server。在启动之后我们可以像网络模块的本地测试一样，通过运行如下指令：

```rust
$ echo "Hello, World!" | nc localhost 6379
```

来对宏内核的 Server 进行访问测试。


## 3. 多个宏内核并存测试

> 相关 commit 为：TODO

TODO