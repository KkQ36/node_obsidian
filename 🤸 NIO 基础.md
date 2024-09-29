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
## Path & Paths
jdk7 引入了 Path 和 Paths 类
- Path 用来表示文件路径
- Paths 是工具类，用来获取 Path 实例
```java
Path source = Paths.get("1.txt"); // 相对路径 使用 user.dir 环境变量来定位 1.txt

Path source = Paths.get("d:\\1.txt"); // 绝对路径 代表了  d:\1.txt

Path source = Paths.get("d:/1.txt"); // 绝对路径 同样代表了  d:\1.txt

Path projects = Paths.get("d:\\data", "projects"); // 代表了  d:\data\projects
```
## Files
检查文件是否存在
```java
Path path = Paths.get("helloword/data.txt");
System.out.println(Files.exists(path));
```
创建一级目录
```java
Path path = Paths.get("helloword/d1");
Files.createDirectory(path);
```
- 如果目录已存在，会抛异常 FileAlreadyExistsException
- 不能一次创建多级目录，否则会抛异常 NoSuchFileException
创建多级目录用
```java
Path path = Paths.get("helloword/d1/d2");
Files.createDirectories(path);
```
拷贝文件
```java
Path source = Paths.get("helloword/data.txt");
Path target = Paths.get("helloword/target.txt");

Files.copy(source, target);
```
- 如果文件已存在，会抛异常 FileAlreadyExistsException
如果希望用 source 覆盖掉 target，需要用 StandardCopyOption 来控制
```java
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
```
移动文件
```java
Path source = Paths.get("helloword/data.txt");
Path target = Paths.get("helloword/data.txt");

Files.move(source, target, StandardCopyOption.ATOMIC_MOVE);
```
- StandardCopyOption.ATOMIC_MOVE 保证文件移动的原子性
删除文件
```java
Path target = Paths.get("helloword/target.txt");

Files.delete(target);
```
- 如果文件不存在，会抛异常 NoSuchFileException
删除目录
```java
Path target = Paths.get("helloword/d1");

Files.delete(target);
```
- 如果目录还有内容，会抛异常 DirectoryNotEmptyException
遍历目录文件
```java
public static void main(String[] args) throws IOException {
    Path path = Paths.get("C:\\Program Files\\Java\\jdk1.8.0_91");
    AtomicInteger dirCount = new AtomicInteger();
    AtomicInteger fileCount = new AtomicInteger();
    Files.walkFileTree(path, new SimpleFileVisitor&lt;Path&gt;(){
	    // 访问目录之前执行
        @Override
        public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) 
            throws IOException {
            System.out.println(dir);
            dirCount.incrementAndGet();
            return super.preVisitDirectory(dir, attrs);
        }
        
		// 访问文件时执行
        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) 
            throws IOException {
            System.out.println(file);
            fileCount.incrementAndGet();
            return super.visitFile(file, attrs);
        }
    });
    System.out.println(dirCount); // 133
    System.out.println(fileCount); // 1479
}
```
删除多级目录
```java
Path path = Paths.get("d");
Files.walkFileTree(path, new SimpleFileVisitor&lt;Path&gt;(){
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) 
        throws IOException {
        Files.delete(file);
        return super.visitFile(file, attrs);
    }

    @Override
    public FileVisitResult postVisitDirectory(Path dir, IOException exc) 
        throws IOException {
        Files.delete(dir);
        return super.postVisitDirectory(dir, exc);
    }
});
```
拷贝多级目录
```java
String source = "D:\\Snipaste-1.16.2-x64";
String target = "D:\\Snipaste-1.16.2-x64aaa";

Files.walk(Paths.get(source)).forEach(path -&gt; {
    try {
        String targetName = path.toString().replace(source, target);
        // 是目录
        if (Files.isDirectory(path)) {
            Files.createDirectory(Paths.get(targetName));
        }
        // 是普通文件
        else if (Files.isRegularFile(path)) {
            Files.copy(path, Paths.get(targetName));
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
});
```
# 网络编程
## 演示阻塞模式
Server，存在两个阻塞的位置，分别是 `accept()` 和 `read()` 方法
```java
public class Server {  
    public static void main(String[] args) {  
        ByteBuffer buffer = ByteBuffer.allocate(16);  
        try (ServerSocketChannel ssc = ServerSocketChannel.open()) {  
            ssc.bind(new InetSocketAddress(8080));  
            List<SocketChannel> channels = new ArrayList<>();  
            while (true) {  
                SocketChannel accept = ssc.accept(); // 阻塞，等待连接  
                channels.add(accept);  
                for (SocketChannel channel : channels) {  
                    channel.read(buffer); // 阻塞，等待读取数据  
                    buffer.flip();  
                    while (buffer.hasRemaining()) {  
                        System.out.print((char) buffer.get());  
                    }  
                    buffer.clear();  
                    System.out.println("\n------");  
                }  
  
            }  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```
Client
```java
public class Client {  
    public static void main(String[] args) {  
        try (SocketChannel sc = SocketChannel.open()){  
            sc.connect(new InetSocketAddress("localhost", 8080));  
            sc.write(StandardCharsets.UTF_8.encode("hello"));  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```
## 非阻塞模式演示
要改为非阻塞模式，只需要设置 `ServerSocketChannel` 和 `SocketChannel`为非阻塞模式，可以分别让 `accept()` 方法和 `read()` 方法变为非阻塞的模式。
- accept 如果没有获取到连接，返回的是 null。
- read 如果没有读取到数据的话，返回的是 0（读取到 0 字节）。
