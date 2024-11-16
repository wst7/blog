---
title: 在Rust中使用TcpListener处理TCP连接
date: 2024-11-16 20:17:08
tags:
  - Rust
  - 网络编程
  - TCP
categories: Rust
---

在Rust中，TcpListener是标准库中提供的一个结构体，用于监听TCP连接，它是网络编程的核心工具之一，允许你接受来自客户端的连接并处理它们。

通过 TcpListener，你可以：
* 创建服务器并监听指定的地址和端口。
* 接收客户端连接，并通过 TcpStream 进行通信。
* 使用多种方法（如 accept 或 incoming）处理单个或多个连接。

## 1、TcpListener::bind
TcpListener::bind 用于绑定到指定的地址和端口。以下是最小的示例：
```rust
use std::net::TcpListener;

fn main() -> std::io::Result<()> {
    // 创建一个 TCP 监听器，绑定到 127.0.0.1:7878
    let listener = TcpListener::bind("127.0.0.1:7878")?;

    println!("Server listening on 127.0.0.1:7878");
    Ok(())
}

```
当服务器启动后，它会监听 127.0.0.1:7878。然而，绑定了端口后，还需要处理客户端连接，这可以通过两种方法实现：accept 和 incoming。

## 2、使用 accept
accept 是 TcpListener 提供的一个阻塞方法，用于逐个处理客户端连接。每次调用 accept，服务器都会等待一个新的客户端连接并返回：
* TcpStream：用于与客户端通信的流。
* SocketAddr：客户端的地址信息。
这种方式非常适合需要完全控制单个连接的场景。
```rust
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write};

fn handle_client(mut stream: TcpStream) {
    let mut buffer = [0; 512];
    // 从流中读取数据
    match stream.read(&mut buffer) {
        Ok(_) => {
            println!("Received: {}", String::from_utf8_lossy(&buffer));
            stream.write(b"Hello from server!").unwrap(); // 向客户端发送数据
        }
        Err(e) => println!("Failed to read from client: {}", e),
    }
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7878")?;

    println!("Server listening on 127.0.0.1:7878");

    for _ in 0..3 { // 只处理 3 个连接示例
        let (stream, addr) = listener.accept()?; // 阻塞等待一个连接
        println!("New connection from {}", addr);
        handle_client(stream); // 处理连接
    }

    Ok(())
}
```

### 特点
* 逐个连接：每次调用 accept 都阻塞线程，等待下一个客户端连接。
* 灵活性高：可以完全控制连接的处理逻辑。

### 适用场景
* 单线程模型或逐步调试。
* 对每个连接的处理逻辑需要高度定制。

## 3、使用incoming
incoming 方法通过迭代器处理多个连接。它每次迭代返回一个新的 TcpStream，用于与客户端通信。
以下代码展示了如何使用 incoming 方法处理多个客户端，并使用线程并发处理：
```rust
use std::net::TcpListener;
use std::io::{Read, Write};
use std::thread;

fn handle_client(mut stream: std::net::TcpStream) {
    let mut buffer = [0; 512];
    if let Ok(_) = stream.read(&mut buffer) {
        println!("Received: {}", String::from_utf8_lossy(&buffer));
        stream.write(b"Response from server").unwrap();
    }
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7878")?;
    println!("Server listening on 127.0.0.1:7878");

    // 使用 incoming 处理多个连接
    for stream in listener.incoming() {
        match stream {
            Ok(stream) => {
                println!("New connection: {}", stream.peer_addr().unwrap());
                // 使用线程处理连接
                thread::spawn(move || handle_client(stream));
            }
            Err(e) => println!("Connection failed: {}", e),
        }
    }

    Ok(())
}
```

### 特点
* 迭代器模式：通过迭代器自然地处理多个连接。
* 便于并发：结合线程或线程池实现高效的多连接处理。

### 适用场景
* 多线程或并发处理。
* 大量客户端连接需要同时处理。

## 4、 非阻塞模式
TcpListener 可以配置为非阻塞模式，通过调用 `set_nonblocking(true)`。在非阻塞模式下：
如果没有客户端连接，请求不会阻塞，而是立即返回 `io::ErrorKind::WouldBlock`。
```rust
use std::net::TcpListener;
use std::io;

fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7878")?;
    listener.set_nonblocking(true)?;

    loop {
        match listener.accept() {
            Ok((stream, addr)) => println!("New connection from {}", addr),
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                // 没有连接，非阻塞模式继续循环
                continue;
            }
            Err(e) => eprintln!("Error: {}", e),
        }
    }
}
```
## 5、异步支持
Rust 提供了异步网络编程框架（如 tokio 和 async-std），可以实现更高效的非阻塞网络服务。
示例（基于 tokio）：
```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> tokio::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7878").await?;
    println!("Server listening on 127.0.0.1:7878");

    loop {
        let (mut socket, addr) = listener.accept().await?;
        println!("New connection from {}", addr);

        tokio::spawn(async move {
            let mut buffer = [0; 512];
            if let Ok(_) = socket.read(&mut buffer).await {
                println!("Received: {}", String::from_utf8_lossy(&buffer));
                socket.write_all(b"Hello from async server!").await.unwrap();
            }
        });
    }
}
```
## 6、总结

* TcpListener::accept：逐个处理单个连接，适合简单或逐步调试。
* TcpListener::incoming：通过迭代器高效处理多个连接，适合并发或多线程场景。
* 非阻塞模式：使用 set_nonblocking 实现事件驱动的网络处理。
* 异步框架支持：结合 tokio 或 async-std，构建高性能网络应用。