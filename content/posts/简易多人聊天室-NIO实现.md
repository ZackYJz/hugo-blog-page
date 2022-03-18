---
title: "简易多人聊天室[2]-NIO模型实现"
date: 2020-05-02T13:31:23+08:00
draft: false
featured_image: "https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202203142007922.png"
tags: [IO]
categories: Java
description: 利用NIO包下的 Buffer 和 Channel 实现简单的多人聊天室，和 BIO 实现进行比较
---

## 总体流程

---

- `ServerSocketChannel` 注册 ACCEPT 事件监听到 Selector 中，用于监听 accept 事件

  - 当有一个客户端发出连接请求，服务器接收了客户端连接请求时，即触发了 `ServerSocketChannel` 的 accept 事件
  - 与 BIO 模型中 `ServerSocket.accept()` 执行的事件相同，即接收了该客户端的连接请求

- `ServerSocketChannel` 触发 accept 事件后，服务器端<u>处理</u>新建立连接的客户端

  - 得到客户端对应的 SocketChannel
  - 将新连接的客户端的 SocketChannel 注册 READ 事件在 Selector 中

  > 即让 Selector 监控客户端的 SocketChannel 是否触发 READ (可读) 事件
  
  > 触发时机：当该客户端像服务器发送了数据，其 SocketChannel 上有了可供服务器读取的数据时，触发 READ 事件

  - 对可读事件触发后的处理操作和 BIO 类似：读取 SocketChannel 的数据并转发给当前连接到服务器的其他客户端

    但在 NIO 中，处理客户端连接的操作都是**在同一个线程中进行**

**注意**

- 虽然 NIO 模型中各个读写操作支持非阻塞式的调用，但 Selector 监听各个 Channel  的 select() 方法是阻塞调用的

- 也就是说，Selector 上监听的所有 Channel 都没有触发其注册监听的事件，此时 Selector 会阻塞，直到有所要监听事件发生

## 服务端

---

### 准备工作

#### 成员 & 构造方法

```java
    private static final String QUIT = "bye";
    private static final int DEFAULT_PORT = 9988;
    private static final int BUFFER_SIZE = 1024;

    private ServerSocketChannel serverChannel;
    private Selector selector;
    //ServerSocketChannel 从 SocketChannel 中读取数据
    private ByteBuffer readBuffer = ByteBuffer.allocate(BUFFER_SIZE);
    //用于向其他的 SocketChannel 中写数据,负责实现转发消息时写入其他客户端
    private ByteBuffer writeBuffer = ByteBuffer.allocate(BUFFER_SIZE);
    //声明标准化字符集
    private Charset charset = Charset.forName("UTF-8");
    //自定监听的端口
    private int port;

    public Server(){
        this(DEFAULT_PORT);
    }

    public Server(int port) {
        this.port = port;
    }
```

#### Help func

- 统一的关闭操作

  ```java
  private void close(Closeable closeable){
      if(closeable != null){
          try {
            closeable.close();
          } catch (IOException e) {
            e.printStackTrace();
          }
      }
  }
  ```

### 主流程：`start()`
---

现在开始实现核心的方法 start

1. 调用静态方法 `ServerSocketChannel.open() `创建一个 ServerSocketChannel

   ```java
   serverChannel = ServerSocketChannel.open()
   serverChannel.configureBlocking(false);
   ```

   默认情况下创建的 ServerSocketChannel 为阻塞式调用，这里我们需要设置调用模式为非阻塞式

2. 将 ServerSocketChannel 关联的 ServerSocket 绑定到监听端口

   ```java
   serverChannel.socket().bind(new InetSocketAddress(port));
   ```

3. 创建 selector 对象，并将 ServerSocketChannel 的 ACCEPT事件注册到 selector 对象中

   ```java
   selector = Selector.open();
   serverChannel.register(selector,SelectionKey.OP_ACCEPT);
   ```

4. 开始监听连接请求

   ```java
   while(true){
     selector.select();
     Set<SelectionKey> keys = selector.selectedKeys();
   }
   ```

   - 当没有channel 触发事件时 select() 会阻塞，直到有 channel 的事件触发。服务端需要不断监听，需要在循环中调用
   - selectedKeys 返回本次监听到的所有触发了事件的 channel 对应的 selectedKey Set 集合

5. 处理触发事件

   ```java
   for (SelectionKey key : keys) {
     handles(key);
   }
   keys.clear();;
   ```

   - 对于集合中每一个触发事件，调用 handles() 进行处理，将在后文实现。
   - 处理完后，需要手动将 selectedKeys 集合清空，保证下一轮不处理过期的重复事件

### 事件处理：`handles()`
---
#### ACCEPT 事件

- 发生在服务器通道 `serverSocketChannel` 上
  - 触发时机： 和客户端建立了连接

- 处理流程：

  - 调用传入的 selectedKey 的 `channel()` 获得触发这个事件的 ServerSocketChannel 

  - 调用 `accept()` 方法，返回对应的客户端通道 SocketChannel
    - 客户端的 channel 也要切换到非阻塞调用模式

  - 接着客户端的 channel 也需要注册到 selector 上，事件为 READ

```java
if(key.isAcceptable()){
    //得到触发了这个事件的通道
    ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
    SocketChannel clientChannel = serverChannel.accept();
    clientChannel.configureBlocking(false);
    clientChannel.register(selector,SelectionKey.OP_READ);
    System.out.println(getClientName(clientChannel)+"已连接");
}
```

#### READ 事件

1. 利用 Buffer 从客户端 channel 中读出发来的消息，进行打印等处理

   > 关于 Buffer 的操作在之前本地文件拷贝的文章中有介绍：传送门

2. 将消息转发到其他客户端

3. 检查是否退出等业务操作

```java
else if(key.isReadable()){
    SocketChannel clientChannel = (SocketChannel) key.channel();
    String fwdMsg = receive(clientChannel);
    if(fwdMsg.isEmpty()){
        //消息为空，出现异常，不再监听该 channel 的事件
        key.cancel();
        //更新 selector 状态，立即返回 select() 方法（多线程下）
        selector.wakeup();
    }else{
        System.out.println(getClientName(clientChannel) + ":" + fwdMsg);
        forwardMessage(clientChannel, fwdMsg);
        // 检查用户是否退出
        if (checkQuit(fwdMsg)) {
            key.cancel();
            selector.wakeup();
            System.out.println(getClientName(clientChannel) + "已断开");
        }
    }
}
```

##### Buffer 接收数据

```java
    private String receive(SocketChannel client) throws IOException {
        //清空缓冲区防止污染
        readBuffer.clear();
        //只要 channel 还有字节读，就一直读
        while(client.read(readBuffer) > 0);
        readBuffer.flip();
        return String.valueOf(charset.decode(readBuffer));
    }
```

#### 消息转发

```java
    private void forwardMessage(SocketChannel client, String fwdMsg) throws IOException {
        //遍历所有监听的 selectionKey
        for (SelectionKey key: selector.keys()) {
            Channel connectedClient = key.channel();
          	//跳过服务器的 channel
            if (connectedClient instanceof ServerSocketChannel) {
                continue;
            }
						//跳过发送者客户端的channel
            if (key.isValid() && !client.equals(connectedClient)) {
                writeBuffer.clear();
                writeBuffer.put(charset.encode(getClientName(client) + ":" + fwdMsg));
                writeBuffer.flip();
                while (writeBuffer.hasRemaining()) {
                    ((SocketChannel)connectedClient).write(writeBuffer);
                }
            }
        }
    }
```

## 客户端

---

### 主流程：`start()`

1. 获取 SocketChannel，配置为非阻塞模式
2. 创建 selector，注册 clientChannel 的 CONNECT 事件
3. 正式向服务器发送连接请求 clientChannel
4. 不停 select() 监听触发的事件，遍历这些事件进行处理

```java
client = SocketChannel.open();
client.configureBlocking(false);

selector = Selector.open();
client.register(selector, SelectionKey.OP_CONNECT);
client.connect(new InetSocketAddress(host, port));

while (true) {
    selector.select();
    Set<SelectionKey> selectionKeys = selector.selectedKeys();
    for (SelectionKey key : selectionKeys) {
      	handles(key);
    }
    selectionKeys.clear();
}
```

###  事件处理：`handles()`

##### CONNECT

连接就绪时触发该事件

- 获取触发该事件的 clientChannel

- 需要额外的线程来处理键盘的阻塞输入
- 注册该 clientChannel 的 READ 事件到 selector 中

```java
if (key.isConnectable()) {
    SocketChannel client = (SocketChannel) key.channel();
    //建立连接的过程是否就绪
    if (client.isConnectionPending()) {
        client.finishConnect();
        // 处理用户的输入
        new Thread(new UserInputHandler(this)).start();
    }
    client.register(selector, SelectionKey.OP_READ);
}
```
**处理键盘输入的线程：**

```java
class UserInputHandler implements Runnable{
        private Client chatClient;
        public UserInputHandler(Client chatClient) {
            this.chatClient = chatClient;
        }
        @Override
        public void run() {
            try {
                // 等待用户输入消息
                BufferedReader consoleReader =
                        new BufferedReader(new InputStreamReader(System.in));
                while (true) {
                    String input = consoleReader.readLine();
                    // 向服务器发送消息
                    chatClient.send(input);
                    // 检查用户是否准备退出
                    if (chatClient.checkQuit(input)) {
                        break;
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

**利用 Buffer 发送消息到服务器：**

```java
public void send(String msg) throws IOException {
    if (msg.isEmpty()) {
      return;
    }

    writeBuffer.clear();
    writeBuffer.put(charset.encode(msg));
    writeBuffer.flip();
    while (writeBuffer.hasRemaining()) {
      client.write(writeBuffer);
    }

    // 检查用户是否准备退出
    if (checkQuit(msg)) {
      close(selector);
    }
}
```

##### READ

服务器转发消息来时触发该事件

```java
 else if (key.isReadable()) {
     SocketChannel client = (SocketChannel) key.channel();
     String msg = receive(client);
     if (msg.isEmpty()) {
       // 服务器异常
       close(selector);
     } else {
       System.out.println(msg);
     }
 }
```

利用 Buffer 接收服务器的消息：

```java
private String receive(SocketChannel client) throws IOException {
    readBuffer.clear();
    while (client.read(readBuffer) > 0);
    readBuffer.flip();
    return String.valueOf(charset.decode(readBuffer));
}
```

## 测试

---

运行客户端时需要在运行配置中运行多个实例运行：

![](https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202201242042716.png)

![](https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202201242043942.png)

![](https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202201242058243.png)