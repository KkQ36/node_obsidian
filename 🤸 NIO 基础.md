# 三大组件
## Channel & Buffer
channel 有一点类似于 stream，它就是读写数据的**双向通道**，可以从 channel 将数据读入 buffer，也可以将 buffer 的数据写入 channel，而之前的 stream 要么是输入，要么是输出，channel 比 stream 更为底层。
![[Pasted image 20240925190450.png]]
常见的 Channel 有：
```java
FileChannel
DatagramChannel
SocketChannel
ServerSocketChannel
```
buffer 则用来缓冲读写数据，常见的 buffer 有
```java
ByteBuffer
ShortBuffer
IntBuffer
LongBuffer
FloatBuffer
DoubleBuffer
CharBuffer
```
## Selector
`Selector` 是 NIO 中的一个关键抽象，它允许单个线程管理多个通道（Channels）。
Selector 可以监听一个或多个注册到它的通道，并能够识别出哪些通道已经准备好执行读取或写入等操作。
Selector 与单个线程去处理单个连接的阻塞模式相比，极大的提升了线程的利用率，线程在一个 socket 无操作的时候，不会被阻塞，而是会去处理其他已经准备好执行读取和写入操作的线程。
![[Pasted image 20240925191525.png]]
# ByteBuffer
## 示例代码
Channel 与 Buffer 的基本使用方式：
```java
public class Main {  
    public static void main(String[] args) {  
        try (FileInputStream inputStream =  new FileInputStream("ryan-demo-netty-nio/src/main/resources/data.txt")) {  
            FileChannel channel = inputStream.getChannel();  
            // 准备缓冲区，单位为字节，分配 1KB 空间  
            ByteBuffer buffer = ByteBuffer.allocate(10);  
            int read = 0;  
            while (read != -1) {  
                // 从 Channel 中读取数据  
                read = channel.read(buffer);  
                // 将 Buffer 切换到读模式，打印 Buffer 的内容  
                buffer.flip();  
                while (buffer.hasRemaining()) {  
                    byte b = buffer.get();  
                    System.out.println((char) b);  
                }  
                // 切换到写模式，清空缓冲区  
                buffer.clear();  
            }  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```
## ByteBuffer 实现原理
[[💬 ByteBuffer 源码剖析]]
## 分散读取 ScatteringReads
```java
private static void scatteringRead() {  
    // 文件中的数据为：onetwothree，希望将其分别写入到三个缓冲区中  
    try (FileInputStream inputStream =  new FileInputStream("ryan-demo-netty-nio/src/main/resources/scatteringRead.txt")) {  
        FileChannel channel = inputStream.getChannel();  
        // 准备缓冲区，单位为字节，分配 1KB 空间  
        ByteBuffer[] buffers = new ByteBuffer[]{  
                ByteBuffer.allocate(3),  
                ByteBuffer.allocate(3),  
                ByteBuffer.allocate(5)  
        };  
        long read = channel.read(buffers);  
        System.out.println(read);  
        for (ByteBuffer buffer : buffers) {  
            buffer.flip();  
            while (buffer.hasRemaining()) {  
                byte b = buffer.get();  
                System.out.println((char) b);  
            }  
        }  
    } catch (IOException e) {  
        throw new RuntimeException(e);  
    }  
}
```
## 集中写入 GatheringWrites
```java
private static void gatheringWrite() {  
    ByteBuffer buffer1 = StandardCharsets.UTF_8.encode("hello");  
    ByteBuffer buffer2 = StandardCharsets.UTF_8.encode("world");  
    ByteBuffer buffer3 = StandardCharsets.UTF_8.encode("你好");  
  
    try (FileOutputStream outputStream = new FileOutputStream("ryan-demo-netty-nio/src/main/resources/gatheringWrite.txt")) {  
        FileChannel channel = outputStream.getChannel();  
        // 准备缓冲区，单位为字节，分配 1KB 空间  
        ByteBuffer[] buffers = new ByteBuffer[]{  
                buffer1,  
                buffer2,  
                buffer3  
        };  
        long write = channel.write(buffers);  
        System.out.println(write);  
    } catch (IOException e) {  
        System.out.println(e);  
    }  
}
```
## 案例：ByteBuffer 解决黏包半包问题
```java
public class ByteBufferExam {  
    private static StringBuilder forStr = new StringBuilder();  
    public static void main(String[] args) {  
        ByteBuffer source = ByteBuffer.allocate(32);  
        source.put("HelloWorld\nI'm Ryan\nHo".getBytes());  
        split(source);  
        source.put("w are you?\n".getBytes());  
        split(source);  
    }  
    private static void split(ByteBuffer source) {  
        // 切换到读模式  
        source.flip();  
        while (source.hasRemaining()) {  
            byte b = source.get();  
            if (b == '\n') {  
                System.out.println(forStr);  
                forStr = new StringBuilder();  
            } else {  
                forStr.append((char) b);  
            }  
        }  
        source.clear();  
    }  
}
```
# 文件编程
## FileChannel
**FileChannel 的获取**：不能直接打开 FileChannel，必须通过 FileInputStream、FileOutputStream 或者 RandomAccessFile 来获取 FileChannel，它们都有 getChannel 方法：
- 通过 FileInputStream 获取的 channel 只能读
- 通过 FileOutputStream 获取的 channel 只能写
- 通过 RandomAccessFile 是否能读写根据构造 RandomAccessFile 时的读写模式决定
**FileChannel 读取数据**：会从 channel 读取数据填充 ByteBuffer，返回值表示读到了多少字节，-1 表示到达了文件的末尾
```java
int readBytes = channel.read(buffer);
```
**FileChannel 的写入**：写入的正确姿势如下， SocketChannel
```java
ByteBuffer buffer = ...;
buffer.put(...); // 存入数据
buffer.flip();   // 切换读模式

while(buffer.hasRemaining()) {
    channel.write(buffer);
}
```
在 while 中调用 channel.write 是因为 write 方法并不能保证一次将 buffer 中的内容全部写入 channel
**FileChannel 的关闭**：channel 必须关闭，不过调用了 FileInputStream、FileOutputStream 或者 RandomAccessFile 的 close 方法会间接地调用 channel 的 close 方法
**FileChannel 的位置**：
```java
// 获取当前位置
long pos = channel.position();
// 设置当前位置
long newPos = ...;
channel.position(newPos);
```
设置当前位置时，如果设置为文件的末尾
- 这时读取会返回 -1
- 这时写入，会追加内容，但要注意如果 position 超过了文件末尾，再写入时在新内容和原末尾之间会有空洞（00）
**文件大小获取**：使用 size 方法获取文件的大小
**FileChannel 的强制写入**：操作系统出于性能的考虑，会将数据缓存，不是立刻写入磁盘。可以调用 `force(true)` 方法将文件内容和元数据（文件的权限等信息）立刻写入磁盘。
## Channel 之间的数据传输
```java
public class FileChannelTest {  
    public static void main(String[] args) {  
        try (  
                FileInputStream from = new FileInputStream("ryan-demo-netty-nio/src/main/resources/data.txt");  
                FileOutputStream to = new FileOutputStream("ryan-demo-netty-nio/src/main/resources/to.txt")  
        ) {  
            FileChannel fromChannel = from.getChannel();  
            FileChannel toChannel = to.getChannel();  
            // 底层应用零拷贝进行优化  
            fromChannel.transferTo(0, fromChannel.size(), toChannel);  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```
## 大文件传输的优化
当文件超过 2G 的时候，传输可能会出现问题，此时就需要采用分次传输的方式：
```java
public class FileChannelTest {  
    public static void main(String[] args) {  
        try (  
                FileInputStream from = new FileInputStream("ryan-demo-netty-nio/src/main/resources/data.txt");  
                FileOutputStream to = new FileOutputStream("ryan-demo-netty-nio/src/main/resources/to.txt")  
        ) {  
            FileChannel fromChannel = from.getChannel();  
            FileChannel toChannel = to.getChannel();  
            // 底层应用零拷贝进行优化  
            long size = fromChannel.size();  
            int count = (int) (size / 1024) + 1;  
            for (long left = size; left > 0;) {  
                left -= fromChannel.transferTo((size - left), count, toChannel);  
            }  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```
