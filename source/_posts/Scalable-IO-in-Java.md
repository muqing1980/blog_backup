---
title: Scalable IO in Java
---

[原文](http://gee.cs.oswego.edu)

原文作者：Doug Lea

## 网络服务

在web服务，分布式系统等 大多数系统都有类似的数据处理流程：

- 读取请求

- 解码请求

- 服务处理

- 编码响应

- 发送响应

但是事实上每一步都有不同的处理。

XML 解析，文件传输，web页面生成，计算服务等

## 典型的服务端设计

典型的设计是 PPC(Process Per Connection)  或者  TPC(Thread Per Connection)

![典型的服务端设计](/images/PPC-TCP.png)

即每个连接在他们自己的处理线程中进行处理。

典型的服务端socket 循环的代码：

```java
class Server implements Runnable{
    public void run(){
        try{
            ServerSocket ss = new SoerverSocket(PORT);
            while(!Thread.interrupted()){
                new  Thread(new Handler(ss.accept())).start(); //TPC
            }catch(IOException e){
                //deal with exception                
            }
        }
    }
    static class Handler implements Runnable{
        final Socket socket;
        Handler(Socket s){
            socket = s;
        }
        public void run(){
            try{
                byte] input = new byte[MAX_INPUT];
                socket.getInputStream().read(input);
                byte[] output = process(input);
                socket.getOutputStream().write(output);
            }cache(IOException e){
                //deal with exception
            }
        }
        private byte[] process(byte[] cmd){
            //deal with input
        }
    }
}
```

## 灵活的目标

- 在负载增加(多客户端)下的优雅降级

- 资源不断增加下的持续改进(CPU，内存，磁盘，带宽)

- 也会满足能力和性能的目标

  - 短延迟

  - 满足高峰需求

  - 可调节的服务能力

分而治之通常是实现任何可伸缩性目标的最佳方法

## 分治思想

- 把处理分解成小任务

  - 每个任务执行一个无阻塞的action

- 在每个任务准备好时执行它

  - IO事件通常作为触发器

  ![handler](/images/handler.png)

- java.nio 支持的基础机制

  - 非阻塞的读和写

  - 与IO事件相关联的调度任务

- 无止境的变化

  - 事件驱动的设计系列

## 事件驱动设计

- 通常比替代方法更有效

  - 极少的资源占用

    - 不需要每个客户端一个线程

  - 较少的开销

    - 较少的上下文切换，更少的锁

  - 但是调度可能较慢

    - 需要手动绑定action和事件

- 编程比较困难

  - 必须分解为简单的非阻塞的action

    - 类似GUI的事件驱动行为

    - 无法消除所有阻塞：GC、页面错误等

  - 必须跟踪服务的逻辑状态

## AWT中的事件

![Event in AWT](/images/event-awt.png)

IO的事件驱动的想法类似于AWT， 但设计是不同的。

## 反应堆模式

- 反应堆通过调度适当的处理程序来响应IO事件

  - 类似于AWT的线程

- 处理程序执行非阻塞的action

  - 类似于AWT的ActionListener

- 管理处理程序绑定到事件

  - 类似于AWT的 addActionListener

> 可以看Schmidt et al, Pattern-Oriented Software Architecture, Volume 2 (POSA2)
> 
> 或者 Richard Stevens's 网络书籍, Matt Welsh's SEDA framework等

## 基础的反应堆设计

单线程版本

![Single Thread Reactor](/images/singleThreadReactor.png)

## java.nio 的支持

- Channels

  - 连接到文件，socket等 支持非阻塞的读

- Buffers

  - 类似于array对象可以由channel直接读或写

- Selectors

  - 告诉一组通道中的有IO事件

- SelectionKeys

  - 维护IO事件状态和绑定

## Reactor 1:Setup

```java
class Reactor implements Runnable{
    final Selector selector;
    final ServerSocketChannel serverSocket;
    Reactor(int port)throws IOException{
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(new InetSocketAddress(port));
        serverSocket.configureBlocking(false);
        SelectionKey sk=serverSocket.register(selector,SelectionKey.OP_ACCEPT);
        sk.attach(new Acceptor());
    }
```

## Reactor 2:Dispatch Loop

```java
public void run(){
    try{
        while(!Thread.interrupted()){
            selector.select();
            Set selected = selector.selectedKeys();
            Iterator it = selected.iterator();
            while(it.hasNext()){
                dispatch((SelectionKey)it.next());
            }
            selected.clear();
        }
    }catch(IOException ex){}
}
void dispatch(SelectionKey k){
    Runnable r = (Runnable)k.attachment();
    if(r!=null){
        r.run();
    }
}
```

## Reactor 3:Acceptor

```java
class Acceptor implements Runnable{
    public void run(){
        try{
            SocketChannel c = serverSocket.accept();
            if(c!=null){
                new Handler(selector,c);
            }
        }catch(IOException ex){}
    }
}
}
```

![Single Thread Reactor](/images/singleThreadReactor.png)

## Reacotr 4:Handler setup

```java
final class Handler implements Runnable{
    final SocketChannel socket;
    final SelectionKey sk;
    ByteBuffer input = ByteBuffer.allocate(MAX_IN);
    ByteBuffer output = ByteBuffer.allocate(MAX_OUT);
    static final int READING = 0,SENDING=1;
    int state = READING;

    Handler(Selector sel,SocketChannel c)throws IOException{
        socket = c;
        c.configureBlocking(false);
        sk = socket.register(sel,0);
        sk.attach(this);
        sk.interestOps(SelectionKey.OP_READ);
        sle.wakeup();        
    }
    boolean inputIsComplete(){}
    boolean outputIsComplete(){}
    void process(){}
```

## Reactor 5:Request handling

```java
public void run(){
    try{
        if(state == READING){
            read();
        }else if(state == SENDING){
            send();
        }
    }catch(IOException ex){}
}
void read() throws IOException{
    socket.read(input);
    if(inputIsComplete()){
        process();
        state = SENDING;
        sk.interestOps(SelectionKey.OP_WRITE);
    }
}
void send() throws IOException{
    socket.write(output);
    if(outputIsComplete()){
        sk.cancel();
    }
}
```

## 状态处理程序

- GoF状态-对象模式的简单使用

  重新绑定合适的处理程序为附件

```java
class Handler{
    public void run(){ // initial state is reader
        socket.read(input);
        if(inputIsComplete()){
            process();
            sk.attach(new Sender());
            sk.interest(SelectionKey.OP_WRITE);
            sk.selector().wakeup();
        }
    }
    class Sender implements Runnable{
        public void run(){
            socket.write(output);
            if(outputIsComplete()){
                sk.cancel();
            }
        }
    }
}
```

## 多线程设计

- 添加线程可以提升伸缩性

  - 主要适用于多处理器

- 工作线程

  - Reactor 要快速触发处理程序

    - 处理程序拖慢Reactor

  - 非IO的处理安排到其他线程

- 多Reactor线程

  - Reactor线程只做IO

  - 负荷分担到其他Reactor

    - 负荷分担以均衡CPU和IO速率

## 工作线程

- 卸载非IO处理以加速

  - Reactor 线程，类似于POSA2 的 proactor 设计；

- 采用work线程比将计算绑定处理重写为事件驱动简单

  - 还是纯的非阻塞计算

  处理能力大于开销

- 多个IO同时处理是比价困难的

  - 一个比较好的方法是读取输入数据到buffer

- 使用线程池进行调优和控制

  - 通常使用远比客户端少的线程

## Worker Thread Pools

![](/images/workerThreadPools.png)

## Handler with Thread Pool

```java
class Handler implements Runnable{
    static PooledExecutor pool = new PooledExecutor(...);
    static final int PROCESSING = 3;

    synchronized void read(){
        socket.read(input);
        if(inputIsComplete()){
            state = PROCESSING;
            pool.execute(new Processer());
        }
    }
    synchronized void processAndHandOff(){
        process();
        state = SENDING;
        sk.interest(SelectedKey.OP_WRITE)
    }
    class Processer implements Runnable{
        public void run(){
            processAndHandOff();
        }
    }
}
```

## 任务协调

- Handoffs(传递)

  - 每个任务启用、触发或调用下一个任务

  - 通常最快，但脆弱

- Callbacks，对每个处理程序的回调

  - 状态集合，附件等；

  - 一个中介者模式的变体

- Queues 队列

  - 例如，跨越阶段的处理缓冲

- Futures

  - 每个任务产生结果

  - 加入或等待/通知的协调

## 使用线程池

- 可调节的工作线程池

- Main方法 `execute(Runnable r)`

- 控制：

  - 任务队列的类型

  - 最大线程数

  - 最小线程数

  - 活动线程

  - 空闲线程保活间隔

    - 如需则由新的线程替换旧的

  - 饱和策略

    - 阻塞，忽略，生产者运行，等

## 多Reactor 线程

- 使用Reactor池

  - 匹配CPU和IO的速率

  - 静态或动态构造

    - 每个都有自己的Selector，Thread，dispatch loop

  - 主 acceptor 分发到其他的reactor

    ```java
    Selector[] selectors;
    int next=0;
    class Acceptor {
        public synchronized void run(){
            Socket connection = serverSocket.accept();
            if(connection !=null){
                new Handler(selectors[next],connection);
            }
            if(++next == selectors.length){
                next = 0;
            }
        }
    }
    ```

## 使用多Reactors

![Multiple Reactors](/images/multipleReactors.png)

## 使用其他nio特性

- 每个reactor 多selector

  - 为不同的IO事件绑定不同handler

  - 为了协调需要注意同步问题

- 文件传输

  - 自动化的 文件到网络 或 网络到文件的拷贝

- 内存映射文件

  - 通过buffer访问文件

- Direct Buffer

  - 有时可以实现零拷贝传输

  - 但是有设置和终结开销

  - 最适合具有长期连接的应用程序

## 基于连接的扩展

- 非单服务的请求

  - 客户端连接

  - 客户端发送一系列消息/请求

  - 客户端断开连接

- 例子

  - 数据库和事物监视器

  - 多人游戏，聊天，等

- 可以扩展基本的网络服务模式

  - 处理许多相对长生命周期的客户端

  - 跟踪客户端和会话状态（包括删除）

  - 跨多个主机分发服务
