---
title: IO模型和NIO
mermaid: true
math: false
comments: true
hide: false
excerpt: 
date: 2025-03-26 22:09:36
tags: [Netty, NIO]
categories: Netty
---



# 铺垫

在学IO模型前，我想先讲清楚为什么要有I/O模型，或者说I/O的核心在解决什么问题。

首先，我想大概温习一下操作系统的用户态（user model）和核心态（Kernel Mode）的职责。

| **操作类型** | **用户态** | **内核态** | **实现机制**               |
| -------- | ------- | ------- | ---------------------- |
| 直接访问硬件   | ❌       | ✅       | 特权指令集限制                |
| 修改MMU页表  | ❌       | ✅       | CR3寄存器保护               |
| 发起系统调用   | ✅       | ➡️      | 软中断门(SYSCALL/SYSENTER) |
| 处理硬件中断   | ❌       | ✅       | IDT(中断描述符表)配置          |
| 修改进程内存空间 | ❌       | ✅       | 页表项特权标志                |
| 调度线程     | ❌       | ✅       | 任务状态段(TSS)             |
大概我们看出来了，一个程序执行超越本程序之外的功能，就会涉及到操作系统用户态到核心态的切换。那什么是本程序范围内的功能呢？例如：计算，内存访问，函数调用等就属于范围内的功能。那么结合本文要说的I/O操作，如果一个程序要进行I/O操作，例如：微信网络传输一个文件，Adobe加载本地一个文件等，都会涉及操作系统用户态到核心态的切换。所以I/O模型的核心确实是**内核空间与用户空间之间的数据交互方式问题**。

既然这样，那么如果我设计一个程序，能否服务端采用AIO的方式，客户端采用NIO方式呢？答案是：当然可以了！ 所以I/O模型解决的是在本机系统状态数据交互的问题。


# 基本概念

- BIO  同步阻塞I/O：每个连接由一个独立的用户线程处理。当该线程执行I/O操作（如read/write）时，会通过系统调用进入内核态，如果数据未就绪，内核会将线程置于阻塞状态，直到数据准备好后才唤醒线程继续执行。

- NIO 同步非阻塞I/O：用户线程通过Selector（用户态对象）调用select()方法时，会触发系统调用进入内核态，内核通过epoll等机制监控注册的Channel。当有I/O事件就绪时，select()返回，用户线程遍历就绪的Channel集合，然后从对应的用户态Buffer中读取或写入数据。

- AIO 异步非阻塞I/O：程序发起I/O操作后立即返回，无需等待操作完成。内核会全权负责I/O操作的执行，包括数据准备和拷贝工作。当操作真正完成时，内核通过回调机制主动通知应用程序处理结果。

##  **内核在三种I/O模型中的职责对比**

| 模型  | 内核职责范围          | 用户线程参与程度      | 典型系统调用               |
| --- | --------------- | ------------- | -------------------- |
| BIO | 仅负责数据就绪检测       | 必须全程等待并执行数据拷贝 | `read()`/`write()`   |
| NIO | 负责就绪事件通知        | 需要主动发起数据拷贝    | `select()`+`read()`  |
| AIO | 全流程处理（检测+拷贝+通知） | 只需提交请求和接收结果   | `io_submit()`（Linux） |
```java

// BIO/NIO的数据流
硬件设备 → 内核缓冲区 →（用户线程介入）→ 用户缓冲区

// AIO的数据流
硬件设备 → 内核缓冲区 → 用户缓冲区
（无用户线程介入）

```

- BIO/NIO：每个完整I/O需要至少2次用户态↔内核态切换

- AIO：只需1次提交+1次回调通知


# NIO
流程：
1. 创建Selector
2. 创建channel，绑定端口
3. Channel注册到Selector
4. Selector处理事件

```java

public class NioServer {  
  
    private static final int BUFFER_SIZE = 1024;  
    private static final int PORT = 8080; 
     
    public static void main(String[] args) {  
        try {  
            // 1. 创建Selector  
            Selector selector = Selector.open();  
            // 2. 创建ServerSocketChannel并配置为非阻塞模式  
            ServerSocketChannel serverChannel = ServerSocketChannel.open();  
            serverChannel.configureBlocking(false);  
            serverChannel.bind(new InetSocketAddress(PORT));  
            // 3. 将ServerSocketChannel注册到Selector，监听ACCEPT事件  
            serverChannel.register(selector, SelectionKey.OP_ACCEPT);  
            System.out.println("服务器启动，监听端口: " + PORT);  
            while (true) {  
                // 4. 阻塞等待就绪的Channel  
                selector.select();  
                // 5. 获取就绪的SelectionKey集合  
                Set<SelectionKey> selectedKeys = selector.selectedKeys();  
                Iterator<SelectionKey> iter = selectedKeys.iterator();  
                while (iter.hasNext()) {  
                    SelectionKey key = iter.next();  
                    // 6. 处理ACCEPT事件  
                    if (key.isAcceptable()) {  
                        handleAccept(key, selector);  
                    }  
                    // 7. 处理READ事件  
                    if (key.isReadable()) {  
                        handleRead(key);  
                    }  
                    // 8. 从集合中移除已处理的key  
                    iter.remove();  
                }  
            }  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
  
    private static void handleAccept(SelectionKey key, Selector selector) throws IOException {  
        ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();  
  
        // 接受客户端连接  
        SocketChannel clientChannel = serverChannel.accept();  
        clientChannel.configureBlocking(false);  
  
        // 注册读事件  
        clientChannel.register(selector, SelectionKey.OP_READ);  
        System.out.println("客户端连接: " + clientChannel.getRemoteAddress());  
    }  
  
    private static void handleRead(SelectionKey key) throws IOException {  
        SocketChannel clientChannel = (SocketChannel) key.channel();  
        ByteBuffer buffer = ByteBuffer.allocate(BUFFER_SIZE);  
  
        try {  
            // 读取客户端数据  
            int bytesRead = clientChannel.read(buffer);  
  
            if (bytesRead == -1) {  
                // 客户端关闭连接  
                System.out.println("客户端断开连接: " + clientChannel.getRemoteAddress());  
                clientChannel.close();  
                return;  
            }  
            // 处理接收到的数据  
            buffer.flip();  
            byte[] bytes = new byte[buffer.remaining()];  
            buffer.get(bytes);  
            String message = new String(bytes);  
            System.out.println("收到消息: " + message);  
  
            // 回显给客户端  
            ByteBuffer response = ByteBuffer.wrap(("服务器回复: " + message).getBytes());  
            clientChannel.write(response);  
            buffer.clear();  
        } catch (IOException e) {  
            System.out.println("客户端异常断开: " + clientChannel.getRemoteAddress());  
            clientChannel.close();  
        }  
    }  
}
```


```java

  
public class NioClient {  
    private static final int BUFFER_SIZE = 1024;  
    private static final String HOST = "localhost";  
    private static final int PORT = 8080;  
  
    public static void main(String[] args) {  
        try {  
            // 1. 创建SocketChannel  
            SocketChannel socketChannel = SocketChannel.open();  
            socketChannel.configureBlocking(false);  
  
            // 2. 连接服务器  
            socketChannel.connect(new InetSocketAddress(HOST, PORT));  
  
            // 等待连接完成  
            while (!socketChannel.finishConnect()) {  
                System.out.println("等待连接...");  
            }  
  
            System.out.println("已连接到服务器");  
  
            // 3. 创建缓冲区  
            ByteBuffer buffer = ByteBuffer.allocate(BUFFER_SIZE);  
            Scanner scanner = new Scanner(System.in);  
  
            while (true) {  
                System.out.print("请输入消息(输入exit退出): ");  
                String message = scanner.nextLine();  
  
                if ("exit".equalsIgnoreCase(message)) {  
                    break;  
                }  
  
                // 4. 发送消息到服务器  
                buffer.clear();  
                buffer.put(message.getBytes());  
                buffer.flip();  
                while (buffer.hasRemaining()) {  
                    socketChannel.write(buffer);  
                }  
  
                // 5. 接收服务器响应  
                buffer.clear();  
                int bytesRead = socketChannel.read(buffer);  
                if (bytesRead > 0) {  
                    buffer.flip();  
                    byte[] bytes = new byte[buffer.remaining()];  
                    buffer.get(bytes);  
                    System.out.println("服务器响应: " + new String(bytes));  
                }  
            }  
  
            // 6. 关闭连接  
            socketChannel.close();  
            scanner.close();  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
}

```


## 传统Java NIO的缺点

1. 需要手动管理Selector、Channel、Buffer和事件循环，开发效率低
2. Java NIO在Linux平台下，空轮询Bug会导致CPU使用率异常升高，严重影响系统性能。（Java11已经修复）
3.  断线重连需要手动实现
4. 性能瓶颈
	1.  线程模型单一：通常只有一个Reactor线程处理I/O
	2.  锁竞争：Selector的selectedKeys()操作需要同步