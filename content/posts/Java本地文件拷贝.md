---
title: "Java 四种本地文件拷贝方式比较和测试"
date: 2020-04-13T13:31:23+08:00
draft: false
featured_image: "https://images.unsplash.com/photo-1517430816045-df4b7de11d1d?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2071&q=80"
tags: [I/O,NIO]
categories: Java
description: 学习完 Java 中 NIO 的基本使用，总结了四种最基本的本地文件拷贝方式，并简单比较性能
---

## 准备工作

---

为了便于测试，定义一个文件拷贝的接口让测试类去继承它：

```java
  public interface IFileCopy {
      void copyFile(File source, File dest);
  }
```

无论是 Stream 还是 Channel ，用完都需要调用 `close()`方法关闭

为了防止重复代码，这里统一写一个关闭的方法：

```java
    static void close(Closeable closeable){
        if(closeable != null){
            try {
                closeable.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

## 实现

---

### 不带缓冲区的流拷贝

- 最朴素的文件拷贝方式，不使用带任何缓冲区的流装饰

- 从源文件的输入流一个字节一个字节读取

  只要还有数据 （`read()` 返回 ≠ -1）就循环读取并写入目标文件的输出流

```java
    static class noBufferedStreamCopy implements IFileCopy{

        @Override
        public void copyFile(File source, File dest) {
            InputStream is = null;
            OutputStream os = null;
            try {
                //创建文件输入流
                is = new FileInputStream(source);
                //创建文件输出流
                os = new FileOutputStream(dest);
                int result;
                //一个字节一个字节读，读到就输出到文件输出流
                while ((result = is.read()) != -1) {
                    os.write(result);
                }
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                close(is);
                close(os);
            }
        }
    }
```



### 带缓冲区的流拷贝

- 使用装饰器模式，用带缓冲区的流对输入输出流进行包裹
- 需要定义缓冲区，即每次写入的次数

```java
	static class  bufferedStreamCopy implements IFileCopy{
        @Override
        public void copyFile(File source, File dest) {
            InputStream is = null;
            OutputStream os = null;
            try {
                is = new BufferedInputStream(new FileInputStream(source));
                os = new BufferedOutputStream(new FileOutputStream(dest));
                byte[] buffer = new byte[1024];
                int len;
                while((len = is.read(buffer)) != -1){
                    os.write(buffer, 0, len);
                }
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            } finally{
                close(is);
                close(os);
            }
        }
    }
```

### 使用 Buffer 的 Channel 拷贝

#### NIO 模型

同步非阻塞式通信模型，`java.nio` 包下提供了 Buffer、Channel 等非阻塞式通信模型的类

核心思路是使用双向的 Channel 代替单向的 Stream，Channel 和 Stream 相比：

- 流是有方向的，一个流只能读或写；而 Channel 通道是双向的，既能读又能写

- 流的读写方法都是阻塞的

  - Channel 通道的读写有阻塞和非阻塞两种模式

  - 由于支持非阻塞调用，允许个线程里处理多个 Channel 的 I/O

#### Buffer

Channel 的读写操作可以通过 Buffer 来实现，Buffer 代表内存中一段可以读写的缓冲区域

Buffer 的基本操作：

- `flip()`
- `clear()`
- `compact()`

**下图详细介绍了他们的用法：**

![](https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202201231936810.png){:height 497, :width 564}

```java
 static class nioBufferedStreamCopy implements IFileCopy{
        @Override
        public void copyFile(File source, File dest) {
            FileChannel sourceChannel = null;
            FileChannel destChannel = null;

            try {
                sourceChannel = new FileInputStream(source).getChannel();
                destChannel = new FileOutputStream(dest).getChannel();

                ByteBuffer buffer = ByteBuffer.allocate(1024);
                while(sourceChannel.read(buffer) !=- 1)
                //从源文件的 channel 读取数据写入 byteBuffer
                {
                    buffer.flip();
                    //从 byteBuffer 中读取数据写入 目标文件的 channel
                    while(buffer.hasRemaining()){
                        destChannel.write(buffer);
                    }
                    buffer.clear();
                }
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            } finally{
                close(sourceChannel);
                close(destChannel);
            }
        }
    }
```

### 使用 Transfer 的 Channel 拷贝

Channel 除了通过 Buffer 进行读写操作，也可以不显式通过 Buffer，直接在 Channel 层面进行数据交换

- 核心方法：

```java
long transferTo(long position, long count,
                                    WritableByteChannel target)
```

- 常用的 Channel：
  - FileChannel
  - ServerSocketChannel
  - SocketChannel

```java
 static class nioTransferCopy implements IFileCopy{
        @Override
        public void copyFile(File source, File dest) {
            FileChannel sourceChannel = null;
            FileChannel destChannel = null;

            try {
                sourceChannel = new FileInputStream(source).getChannel();
                destChannel = new FileOutputStream(dest).getChannel();
                long copyed = 0;
              	//transferTo 方法不保证一次性全部传输完
                while(copyed < sourceChannel.size()){
                    copyed  += sourceChannel.transferTo(0, sourceChannel.size(), destChannel);
                }
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            } finally{
                close(sourceChannel);
                close(destChannel);
            }
        }
    }
```

## 测试

简单起见，编写一个测试方法，直接在 main 方法中调用。拷贝 5 次统计平均时间：

``` java
    public static void main(String[] args) {
        File smallFile = new File("/Users/admin/Downloads/test.mp4");
        File smallFileCopy = new File("/Users/admin/Downloads/testCopy/test.mp4");
        File bigFile = new File("/Users/admin/Downloads/golangDoc.zip");
        File bigFileCopy = new File("/Users/admin/Downloads/testCopy/golangDoc.zip");

        System.out.println("=====Small file (<1M) Copy test=======");
        benchmark(new noBufferedStreamCopy(), smallFile, smallFileCopy);
        benchmark(new bufferedStreamCopy(), smallFile, smallFileCopy);
        benchmark(new nioBufferedStreamCopy(), smallFile, smallFileCopy);
        benchmark(new nioTransferCopy(), smallFile, smallFileCopy);

        System.out.println("=====Big file (>10M) Copy test=======");
        benchmark(new noBufferedStreamCopy(), bigFile, bigFileCopy);
        benchmark(new bufferedStreamCopy(), bigFile, bigFileCopy);
        benchmark(new nioBufferedStreamCopy(), bigFile, bigFileCopy);
        benchmark(new nioTransferCopy(), bigFile, bigFileCopy);
    }

    static void benchmark(IFileCopy test,File source,File target){
        long elapsed = 0L;
        for (int i = 0; i < TEST_ROUNDS; i++) {
            long startTime = System.currentTimeMillis();
            test.copyFile(source,target);
            elapsed += System.currentTimeMillis() - startTime;
            target.delete();
        }
        System.out.println(test.getClass().getName()+"平均用时:"+elapsed/TEST_ROUNDS+"ms");
    }
```

> 测试结果：
>
> ![](https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202201241327030.png)

不使用缓冲区一个字节一个字节拷贝大文件实在太慢了....

### 总结

- 缓冲区的使用对性能的提升效果非常明显

  - 除了 `noBufferStreamCopy` 其他方法拷贝方式性能表现差不多

  - 但是随着文件增大，nioTransferCopy 的性能更好一些

- 其他拷贝性能差距不大的原因是：
  - NIO 在 JDK4 版本中推出，和 JDK3 中 BIO 方法对比性能有较大提升
  - 但 Java 之后的版本，传统 IO 的底层实现已经使用 NIO 新的方式重新实现