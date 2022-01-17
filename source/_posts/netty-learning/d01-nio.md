---
title: "Netty（一）NIO"
date: 2021-08-17T11:14:25+08:00
lastmod: 2021-08-17T11:14:25+08:00
categories: ["Netty"]
---

<!--more-->

`java.nio` 全称 Java non-blocking IO（实际上是 New IO），是指 JDK 1.4 及以上版本里提供的新 API（New IO） ，为所有的原始类型（ `boolean` 类型除外）提供缓存支持的数据容器，使用它可以提供非阻塞式的高伸缩性网络。

## 三大组件

`Buffer`、`Channel`、`Selector` 是 NIO 的三大组件。

### Buffer

`Buffer` 用于缓冲读写数据，常见的 `Buffer` 有：

+ `ByteBuffer`
  + `MappedByteBuffer`
  + `DirectByteBuffer`
  + `HeapByteBuffer`
+ `ShortBuffer`
+ `IntBuffer`
+ `LongBuffer`
+ `FloatBuffer`
+ `DoubleBuffer`
+ `CharBuffer`

### Channel

`Channel` 有一点类似于 `Stream`，它就是读写数据的**双向通道**，可以从 `Channel` 将数据读入 `Buffer`，也可以将 `Buffer` 的数据写入 `Channel`，而之前的 `Stream` 要么是输入，要么是输出，`Channel` 比 `Stream` 更为底层。

{% mermaid graph LR %}
Channel --> Buffer
Buffer --> Channel
{% endmermaid %}

### Selector

`Selector` 单从字面意思不好理解，需要结合服务器的设计演化来理解它的用途。

#### 多线程服务器设计

{% mermaid graph TD %}
subgraph 多线程服务器设计
t1(thread) --> s1(socket1)
t2(thread) --> s2(socket2)
t3(thread) --> s3(socket3)
end
{% endmermaid %}

存在以下缺点：

+ 内存占用高
+ 线程上下文切换成本高
+ 只适合连接数少的场景

#### 线程池服务器设计

{% mermaid graph TD %}
subgraph 线程池服务器设计
t4(thread) --> s4(socket1)
t5(thread) --> s5(socket2)
t4(thread) -.-> s6(socket3)
t5(thread) -.-> s7(socket4)
end
{% endmermaid %}

存在以下缺点：

+ 阻塞模式下，线程仅能处理一个 Socket 连接。
+ 仅适合短连接场景

#### Selector 服务器设计

{% mermaid graph TD %}
subgraph Selector 服务器设计
thread --> selector
selector --> c1(channel)
selector --> c2(channel)
selector --> c3(channel)
end
{% endmermaid %}

`Selector` 的作用就是配合一个线程来管理多个 `Channel`，获取这些 `Channel` 上发生的事件，这些 `Channel` 工作在非阻塞模式下，不会让线程吊死在一个 `Channel` 上。**适合连接数特别多，但流量低的场景（low traffic）**。

调用 `Selector` 的 `select()` 会阻塞直到 `Channel` 发生了读写就绪事件；当这些事件发生，`select()` 方法就会返回这些事件交给 `Thread` 来处理。

## Hello World

通过一个简单的入门案例来演示 NIO 的基本使用；有一普通文本文件 data.txt，内容为：

```txt
1234567890abcd
```
使用 `FileChannel` 来读取文件内容：

```java
@Slf4j(topic = "d01.ByteBufferHelloWorld")
public class ByteBufferHelloWorld {
    public static void main(String[] args) {

        try (FileChannel channel = new FileInputStream("d01/data.txt").getChannel()) {

            /*
            申请10个字节的缓存区。
             */
            final ByteBuffer buffer = ByteBuffer.allocate(10);

            while (true) {

                /*
                从 FileChannel 中读取数据并写入到 ByteBuffer 中，每次读取10个字节；此时为写模式，读取完毕返回 -1。
                 */
                final int length = channel.read(buffer);

                log.debug("读取到的字节数为：{}", length);

                if (-1 == length) {
                    break;
                }

                /*
                切换至读模式。
                 */
                buffer.flip();

                while (buffer.hasRemaining()) {
                    final byte b = buffer.get();

                    log.debug("读取到的字节值为：{}", (char) b);
                }

                /*
                清空，切换至写模式。
                 */
                buffer.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

```shell
14:46:42.123 d01.ByteBufferHelloWorld [main] - 读取到的字节数为：10
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：1
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：2
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：3
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：4
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：5
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：6
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：7
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：8
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：9
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：0
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节数为：4
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：a
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：b
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：c
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节值为：d
14:46:42.131 d01.ByteBufferHelloWorld [main] - 读取到的字节数为：-1

Process finished with exit code 0
```

## ByteBuffer

通过对 `ByteBuffer` 的深入学习，能更加深入的理解 NIO 数据传输的过程。

### ByteBuffer 正确姿势

按照以下步骤正确使用 `ByteBuffer` ，该过程中需要注意的是读写模式的切换：

1. 向 `ByteBuffer` 写入数据，例如调用 `channel.read(buffer)`。
2. 调用 `flip()` 切换至**读模式**。
3. 从 `ByteBuffer` 读取数据，例如调用 `buffer.get()`。
4. 调用 `clear()` 或 `compact()` 切换至**写模式**。
5. 重复 1~4 步骤。

### ByteBuffer 内部结构

`ByteBuffer` 有以下重要属性：

+ capacity
+ position
+ limit

当一个容量为`10`个字节的 `ByteBuffer` 刚被创建时：

![](bytebuffer-new.png)

[bytebuffer-new.drawio](bytebuffer-new.drawio)

切换至写模式后，`position` 切换为写入位置，`limit` 切换为写入限制切等于 `capacity`，下图为写入`4`个字节后的状态：

![](bytebuffer-write-4-bytes.png)

[bytebuffer-write-4-bytes.drawio](bytebuffer-write-4-bytes.drawio)

切换至读模式后，`position` 切换为读取位置，`limit` 切换为读取限制，调用 `flip()` 可切换为读模式，flip 的字面意思为浏览，如图所示：

![](bytebuffer-flip.png)

[bytebuffer-flip.drawio](bytebuffer-flip.drawio)

下图为读取`4`个字节后的状态：

![](bytebuffer-read-4-bytes.png)

[bytebuffer-read-4-bytes.drawio](bytebuffer-read-4-bytes.drawio)

调用 `clear()` 方法后将切换至写模式并重置属性，此时状态如图所示：

![](bytebuffer-clear.png)

[bytebuffer-clear.drawio](bytebuffer-clear.drawio)

调用 `compact()` 方法将切换至写模式并把未读完的部分向前压缩，如下图所示：

![](bytebuffer-compact.png)

[bytebuffer-compact.drawio](bytebuffer-compact.drawio)

### ByteBuffer 调试工具

提供一个可直观显示 `ByteBuffer` 内部结构属性的调式工具源码：

```java
public abstract class ByteBufferUtil {
    private static final char[] BYTE2CHAR = new char[256];
    private static final char[] HEXDUMP_TABLE = new char[256 * 4];
    private static final String[] HEXPADDING = new String[16];
    private static final String[] HEXDUMP_ROWPREFIXES = new String[65536 >>> 4];
    private static final String[] BYTE2HEX = new String[256];
    private static final String[] BYTEPADDING = new String[16];

    static {
        final char[] DIGITS = "0123456789abcdef".toCharArray();
        for (int i = 0; i < 256; i++) {
            HEXDUMP_TABLE[i << 1] = DIGITS[i >>> 4 & 0x0F];
            HEXDUMP_TABLE[(i << 1) + 1] = DIGITS[i & 0x0F];
        }

        int i;

        // Generate the lookup table for hex dump paddings
        for (i = 0; i < HEXPADDING.length; i++) {
            int padding = HEXPADDING.length - i;
            StringBuilder buf = new StringBuilder(padding * 3);
            for (int j = 0; j < padding; j++) {
                buf.append("   ");
            }
            HEXPADDING[i] = buf.toString();
        }

        // Generate the lookup table for the start-offset header in each row (up to 64KiB).
        for (i = 0; i < HEXDUMP_ROWPREFIXES.length; i++) {
            StringBuilder buf = new StringBuilder(12);
            buf.append(NEWLINE);
            buf.append(Long.toHexString((long) i << 4 & 0xFFFFFFFFL | 0x100000000L));
            buf.setCharAt(buf.length() - 9, '|');
            buf.append('|');
            HEXDUMP_ROWPREFIXES[i] = buf.toString();
        }

        // Generate the lookup table for byte-to-hex-dump conversion
        for (i = 0; i < BYTE2HEX.length; i++) {
            BYTE2HEX[i] = ' ' + StringUtil.byteToHexStringPadded(i);
        }

        // Generate the lookup table for byte dump paddings
        for (i = 0; i < BYTEPADDING.length; i++) {
            int padding = BYTEPADDING.length - i;
            StringBuilder buf = new StringBuilder(padding);
            for (int j = 0; j < padding; j++) {
                buf.append(' ');
            }
            BYTEPADDING[i] = buf.toString();
        }

        // Generate the lookup table for byte-to-char conversion
        for (i = 0; i < BYTE2CHAR.length; i++) {
            if (i <= 0x1f || i >= 0x7f) {
                BYTE2CHAR[i] = '.';
            } else {
                BYTE2CHAR[i] = (char) i;
            }
        }
    }

    /**
     * 打印所有内容
     *
     * @param buffer {@link ByteBuffer}
     */
    public static void debugAll(ByteBuffer buffer) {
        int oldlimit = buffer.limit();
        buffer.limit(buffer.capacity());
        StringBuilder origin = new StringBuilder(256);
        appendPrettyHexDump(origin, buffer, 0, buffer.capacity());
        System.out.println("+--------+-------------------- all ------------------------+----------------+");
        System.out.printf("position: [%d], limit: [%d]\n", buffer.position(), oldlimit);
        System.out.println(origin);
        buffer.limit(oldlimit);
    }

    /**
     * 打印可读取内容
     *
     * @param buffer {@link ByteBuffer}
     */
    public static void debugRead(ByteBuffer buffer) {
        StringBuilder builder = new StringBuilder(256);
        appendPrettyHexDump(builder, buffer, buffer.position(), buffer.limit() - buffer.position());
        System.out.println("+--------+-------------------- read -----------------------+----------------+");
        System.out.printf("position: [%d], limit: [%d]\n", buffer.position(), buffer.limit());
        System.out.println(builder);
    }

    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);
        buffer.put(new byte[]{97, 98, 99, 100});
        debugAll(buffer);
    }

    private static void appendPrettyHexDump(StringBuilder dump, ByteBuffer buf, int offset, int length) {
        if (isOutOfBounds(offset, length, buf.capacity())) {
            throw new IndexOutOfBoundsException(
                    "expected: " + "0 <= offset(" + offset + ") <= offset + length(" + length
                            + ") <= " + "buf.capacity(" + buf.capacity() + ')');
        }
        if (length == 0) {
            return;
        }
        dump.append("         +-------------------------------------------------+").append(NEWLINE).append("         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |").append(NEWLINE).append("+--------+-------------------------------------------------+----------------+");

        final int fullRows = length >>> 4;
        final int remainder = length & 0xF;

        // Dump the rows which have 16 bytes.
        for (int row = 0; row < fullRows; row++) {
            int rowStartIndex = (row << 4) + offset;

            // Per-row prefix.
            appendHexDumpRowPrefix(dump, row, rowStartIndex);

            // Hex dump
            int rowEndIndex = rowStartIndex + 16;
            for (int j = rowStartIndex; j < rowEndIndex; j++) {
                dump.append(BYTE2HEX[getUnsignedByte(buf, j)]);
            }
            dump.append(" |");

            // ASCII dump
            for (int j = rowStartIndex; j < rowEndIndex; j++) {
                dump.append(BYTE2CHAR[getUnsignedByte(buf, j)]);
            }
            dump.append('|');
        }

        // Dump the last row which has less than 16 bytes.
        if (remainder != 0) {
            int rowStartIndex = (fullRows << 4) + offset;
            appendHexDumpRowPrefix(dump, fullRows, rowStartIndex);

            // Hex dump
            int rowEndIndex = rowStartIndex + remainder;
            for (int j = rowStartIndex; j < rowEndIndex; j++) {
                dump.append(BYTE2HEX[getUnsignedByte(buf, j)]);
            }
            dump.append(HEXPADDING[remainder]);
            dump.append(" |");

            // Ascii dump
            for (int j = rowStartIndex; j < rowEndIndex; j++) {
                dump.append(BYTE2CHAR[getUnsignedByte(buf, j)]);
            }
            dump.append(BYTEPADDING[remainder]);
            dump.append('|');
        }

        dump.append(NEWLINE).append("+--------+-------------------------------------------------+----------------+");
    }

    private static void appendHexDumpRowPrefix(StringBuilder dump, int row, int rowStartIndex) {
        if (row < HEXDUMP_ROWPREFIXES.length) {
            dump.append(HEXDUMP_ROWPREFIXES[row]);
        } else {
            dump.append(NEWLINE);
            dump.append(Long.toHexString(rowStartIndex & 0xFFFFFFFFL | 0x100000000L));
            dump.setCharAt(dump.length() - 9, '|');
            dump.append('|');
        }
    }

    public static short getUnsignedByte(ByteBuffer buffer, int index) {
        return (short) (buffer.get(index) & 0xFF);
    }
}
```

### ByteBuffer 常见方法

在使用 `ByteBuffer ` 的读取和写入相关方法时，务必先阅读源码，部分方法在调用后会切换读写模式，如果使用不当，排查非常困难！

#### 分配空间

可以使用 `allocate(int capacity)` 方法为 `ByteBuffer` 分配空间，其他 `Buffer` 类也有该方法：

+ `allocate(int capacity)` 分配的是堆内存，效率较低，需要进行一次拷贝。
+ `allocateDirect(int capacity)` 分配的是直接内存，无需进行拷贝，但使用不当容易造成内存泄漏。

示例代码：

```java
public class ByteBufferAllocate {
    public static void main(String[] args) {
        /*
        class java.nio.HeapByteBuffer，java 堆内存，读写效率较低，受到 GC 的影响。
         */
        System.out.println(ByteBuffer.allocate(16).getClass());
        
        /*
        class java.nio.DirectByteBuffer，直接内存，读写效率高（少一次拷贝），不会受 GC 影响，分配的效率低。
         */
        System.out.println(ByteBuffer.allocateDirect(16).getClass());
    }
}
```

输出结果：

```shell
class java.nio.HeapByteBuffer
class java.nio.DirectByteBuffer

Process finished with exit code 0
```

#### 写入数据

向 `ByteBuffer` 写入数据：

+ 调用 `Channel` 的 `read()` 方法，从 `Channel` 中读取出数据写入到 `Buffer` 中。
  ```java
  int readBytes = channel.read(buf);
  ```
+ 调用 `ByteBuffer` 自己的 `put()` 方法。
  ```java
  buf.put((byte)127);
  ```
  

示例代码：

```java
public class ByteBufferWrite {
    public static void main(String[] args) {

        ByteBuffer buffer = ByteBuffer.allocate(10);

        /*
        1. 调用 FileChannel 的 read() 方法，从 Channel 中读取出数据写入到 ByteBuffer 中。d01/rw.txt 内容为：12345 共 5 字节。
         */

        try (RandomAccessFile file = new RandomAccessFile("d01/rw.txt", "rw")) {
            final FileChannel channel = file.getChannel();

            /*
            该方法返回读取到的字节数，如果没有读取到数据则返回 -1。
             */
            final int readBytes = channel.read(buffer);

            /*
            输出 5。
             */
            System.out.println(readBytes);
        } catch (IOException e) {
            e.printStackTrace();
        }

        /*
        position: [5], limit: [10]。
         */
        ByteBufferUtil.debugAll(buffer);

        /*
        2. 调用 ByteBuffer 自己的 put() 方法。
         */

        /*
        调用单个字节参数方法 写入 'a' 0x61。
         */
        buffer.put((byte) 0x61);

        /*
        position: [6], limit: [10]。
         */
        ByteBufferUtil.debugAll(buffer);

        /*
        调用字节数组参数方法 写入 'b' 0x62 'c' 0x63 'd' 0x64。
         */
        buffer.put(new byte[]{(byte) 0x62, (byte) 0x63, (byte) 0x64});

        /*
        position: [9], limit: [10]。
         */
        ByteBufferUtil.debugAll(buffer);
    }
}
```

输出结果：

```shell
5
+--------+-------------------- all ------------------------+----------------+
position: [5], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 31 32 33 34 35 00 00 00 00 00                   |12345.....      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [6], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 31 32 33 34 35 61 00 00 00 00                   |12345a....      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [9], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 31 32 33 34 35 61 62 63 64 00                   |12345abcd.      |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0

```

#### 读取数据

向 `ByteBuffer` 读取数据：

+ 调用 `FileChannel` 的 `write()` 方法，从 `ByteBuffer` 中读取出数据写入到 `FileChannel` 中。

  ```java
  int writeBytes = channel.write(buf);
  ```

+  调用 `ByteBuffer` 自己的 `get()` 方法。

  ```java
  byte b = buf.get();
  ```

   `get()` 方法会让 `position` 读指针后移一位，如果想重复读取数据：

  + 可以调用 `rewind() ` 方法将 `position` 重新置为 `0`。
  + 或者调用 有参`get(int i)` 方法获取索引 `i` 的内容，它不会移动读指针。

示例代码：

```java
public class ByteBufferRead {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);

        buffer.put(new byte[]{'c', 'd', 'e', 'f'});

        buffer.flip();

        /*
        这里必须创建一个输出流，数据的走向是从 ByteBuffer 中读取数据，通过 FileChannel 写入到文件中，因此也必须在做下面的操作前切换至读模式。
         */
        try (FileChannel channel = new FileOutputStream("d01/r.txt").getChannel()) {
            /*
            1. 调用 Channel 的 write() 方法，从 ByteBuffer 中读取出数据写入到 FileChannel 中。
             */
            final int writeBytes = channel.write(buffer);

            /*
            4，共从 Buffer 中读取了 4个字节的数据写入到文件 d01/r.txt 中。
             */
            System.out.println(writeBytes);
        } catch (IOException e) {
            e.printStackTrace();
        }

        /*
        position: [4], limit: [4]。
         */
        ByteBufferUtil.debugAll(buffer);

        /*
        2. 调用 ByteBuffer 自己的 get() 方法。
         */

        ByteBuffer buf = ByteBuffer.allocate(10);

        buf.put(new byte[]{'c', 'd', 'e', 'f'});

        buf.flip();

        /*
        调用无参 get() 方法会使 position + 1。
         */
        buf.get();
        buf.get();

        /*
        position: [2], limit: [4]。
         */
        ByteBufferUtil.debugAll(buf);

        /*
        调用有参 get(int i) 方法只会获取对应位置的数据，position 的值不变，下面的代码输出 'e'。
         */

        System.out.println((char) buf.get(2));

        /*
        position: [2], limit: [4]。
         */
        ByteBufferUtil.debugAll(buf);
    }
}
```

输出结果：

```shell
4
+--------+-------------------- all ------------------------+----------------+
position: [4], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 63 64 65 66 00 00 00 00 00 00                   |cdef......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [2], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 63 64 65 66 00 00 00 00 00 00                   |cdef......      |
+--------+-------------------------------------------------+----------------+
e
+--------+-------------------- all ------------------------+----------------+
position: [2], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 63 64 65 66 00 00 00 00 00 00                   |cdef......      |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0
```

#### 紧凑模式

`compact()`方法会将未读完的数据向前压缩，并切换至写模式：

```java
public class ByteBufferCompact {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);

        buffer.put(new byte[]{'a', 'b', 'c', 'd'});

        /*
        position: [4], limit: [10]。
         */
        ByteBufferUtil.debugAll(buffer);

        buffer.flip();

        /*
        'a'。
         */
        System.out.println((char) buffer.get());

        /*
        position: [1], limit: [4]。
         */
        ByteBufferUtil.debugAll(buffer);

        /*
        将未读完的部分向前压缩（这里只是变更 position 的值不做数据清空），并切换至写模式。
         */
        buffer.compact();

        /*
        position: [3], limit: [10]，此时内存中的 ByteBuffer 值为 [98, 99, 100, 100, 0, 0, 0, 0, 0, 0]。
         */
        ByteBufferUtil.debugAll(buffer);
    }
}
```
输出结果：

```shell
+--------+-------------------- all ------------------------+----------------+
position: [4], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+
a
+--------+-------------------- all ------------------------+----------------+
position: [1], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [3], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 62 63 64 64 00 00 00 00 00 00                   |bcdd......      |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0

```

#### 倒带重读

`rewind()` 方法可以应用于需要重复读取数据的场景，就像播放磁带的倒带功能一样：

```java
public class ByteBufferRewind {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);

        buffer.put(new byte[]{'a', 'b', 'c', 'd'});

        /*
        position: [4], limit: [10]。
         */
        ByteBufferUtil.debugAll(buffer);

        buffer.flip();

        /*
        position: [0], limit: [4]。
         */
        ByteBufferUtil.debugAll(buffer);

        buffer.get();
        buffer.get();

        /*
        position: [2], limit: [4]。
         */
        ByteBufferUtil.debugAll(buffer);

        /*
        将 position 置为 0，便于重新读取数据。
         */
        buffer.rewind();

        /*
        position: [0], limit: [4]。
         */
        ByteBufferUtil.debugAll(buffer);
    }
}
```

输出结果：

```shell
+--------+-------------------- all ------------------------+----------------+
position: [4], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [2], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0
```

#### 标记回退

`mark()` 和`reset()` 方法组合使用可以实现回退到指定位置的功能：

```java
public class ByteBufferMarkReset {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);

        buffer.put(new byte[]{'a', 'b', 'c', 'd'});

        buffer.flip();

        buffer.get();
        buffer.get();

        /*
        position: [2], limit: [4]。
         */
        ByteBufferUtil.debugAll(buffer);

        /*
        在 position[2] 做标记
         */
        buffer.mark();

        buffer.get();
        buffer.get();

        /*
        回退到 position[2]
         */
        buffer.reset();

        /*
        position: [2], limit: [4]。
         */
        ByteBufferUtil.debugAll(buffer);
    }
}
```

输出结果：

```shell
+--------+-------------------- all ------------------------+----------------+
position: [2], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [2], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0
```

#### 清空重置

`clear()` 方法将会重置 `ByteBuffer`，并切换为写模式：

```java
public class ByteBufferClear {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);

        buffer.put(new byte[]{'a', 'b', 'c', 'd'});

        /*
        position: [4], limit: [10]。
         */
        ByteBufferUtil.debugAll(buffer);

        buffer.flip();

        buffer.get();
        buffer.get();
        buffer.get();

        /*
        position: [3], limit: [4]。
         */
        ByteBufferUtil.debugAll(buffer);

        /*
        清空并切换至写模式。
         */
        buffer.clear();

        /*
        position: [0], limit: [10]。
         */
        ByteBufferUtil.debugAll(buffer);
    }
}
```

输出结果：

```shell
+--------+-------------------- all ------------------------+----------------+
position: [4], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [3], limit: [4]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 64 00 00 00 00 00 00                   |abcd......      |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0
```

#### 转换操作

下面的例子演示 `ByteBuffer` 与 `String` 之间的互相转换：

```java
public class ByteBufferStringConvertion {
    public static void main(String[] args) {
        /*
        1. 字符串转为 ByteBuffer
         */
        ByteBuffer buffer1 = ByteBuffer.allocate(10);
        buffer1.put("hello".getBytes());
        ByteBufferUtil.debugAll(buffer1);

        /*
        2. Charset
         */
        ByteBuffer buffer2 = StandardCharsets.UTF_8.encode("hello");
        ByteBufferUtil.debugAll(buffer2);

        /*
        3. wrap
         */
        ByteBuffer buffer3 = ByteBuffer.wrap("hello".getBytes());
        ByteBufferUtil.debugAll(buffer3);

        /*
        4. 转为字符串
         */
        String str1 = StandardCharsets.UTF_8.decode(buffer2).toString();
        System.out.println(str1);

        buffer1.flip();
        String str2 = StandardCharsets.UTF_8.decode(buffer1).toString();
        System.out.println(str2);
    }
}
```

输出结果：

```shell
+--------+-------------------- all ------------------------+----------------+
position: [5], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 68 65 6c 6c 6f 00 00 00 00 00                   |hello.....      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [5]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 68 65 6c 6c 6f                                  |hello           |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [5]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 68 65 6c 6c 6f                                  |hello           |
+--------+-------------------------------------------------+----------------+
hello
hello

Process finished with exit code 0
```

### ByteBuffer 分散读取

分散读取（Scattering Reads），代码不是重点，其中的思想值得借鉴，有文本文件 `words.txt` ，其内容如下：

```txt
onetwothree
```

使用如下方式读取，可以将数据填充至多个 `ByteBuffer ` 中：

```java
public class ByteBufferScatteringReads {
    public static void main(String[] args) {
        try (FileChannel channel = new RandomAccessFile("d01/words.txt", "r").getChannel()) {
            ByteBuffer buf1 = ByteBuffer.allocate(3);
            ByteBuffer buf2 = ByteBuffer.allocate(3);
            ByteBuffer buf3 = ByteBuffer.allocate(5);

            final ByteBuffer[] buffers = {buf1, buf2, buf3};

            /*
            传入 ByteBuffer[] 数组，通过 FileChannel 将文件中的数据分散到各个 ByteBuffer 中。
             */
            channel.read(buffers);

            for (ByteBuffer b : buffers) {
                b.flip();

                ByteBufferUtil.debugAll(b);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

```shell
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [3]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 6f 6e 65                                        |one             |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [3]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 74 77 6f                                        |two             |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [5]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 74 68 72 65 65                                  |three           |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0
```

### ByteBuffer 集中写入

与分散读取相对应的是集中写入（Gathering Writes），下面的代码将多个 `ByteBuffer` 的数据集中写入到文件 `words_zh_CN.txt` 中：

```java
public class ByteBufferGatheringWrites {
    public static void main(String[] args) {

        try (FileChannel channel = new RandomAccessFile("d01/words_zh_CN.txt", "rw").getChannel()) {

            final ByteBuffer buf1 = StandardCharsets.UTF_8.encode("你好");
            final ByteBuffer buf2 = StandardCharsets.UTF_8.encode("小");
            final ByteBuffer buf3 = StandardCharsets.UTF_8.encode("苹果");

            channel.write(new ByteBuffer[]{buf1, buf2, buf3});
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### ByteBuffer 黏包半包

网络上有多条数据发送给服务端，数据之间使用 `\n` 进行分隔，但由于某种原因这些数据在接收时，被进行了重新组合，例如原始数据有 `3` 条为：

* `Hello,world\n`
* `I'm Zhang San\n`
* `How are you?\n`

变成了下面的两个 `ByteBuffer` （黏包，半包）：

* `Hello,world\nI'm Zhang San\nHo`
* `w are you?\n`

现在要求你编写程序，将错乱的数据恢复成原始的按 `\n` 分隔的数据。

示例代码：

```java
public class ByteBufferPacketHandle {
    public static void main(String[] args) {
         /*
         网络上有多条数据发送给服务端，数据之间使用 \n 进行分隔
         但由于某种原因这些数据在接收时，被进行了重新组合，例如原始数据有 3 条为
             Hello,world\n
             I'm Zhang San\n
             How are you?\n
         变成了下面的两个 ByteBuffer (黏包，半包)
             Hello,world\nI'm Zhang San\nHo
             w are you?\n
         现在要求你编写程序，将错乱的数据恢复成原始的按 \n 分隔的数据
         */
        ByteBuffer source = ByteBuffer.allocate(32);
        source.put("Hello,world\nI'm Zhang San\nHo".getBytes());
        split(source);
        source.put("w are you?\n".getBytes());
        split(source);
    }

    private static void split(ByteBuffer source) {

        source.flip();

        for (int i = 0; i < source.limit(); i++) {
            final byte b = source.get(i);

            if (b == '\n') {
                /*
                计算一条完整的消息长度
                 */
                int len = i + 1 - source.position();

                ByteBuffer target = ByteBuffer.allocate(len);

                /*
                从 source 读取数据写入到 target
                 */

                for (int j = 0; j < len; j++) {
                    target.put(source.get());
                }

                ByteBufferUtil.debugAll(target);
            }
        }

        source.compact();
    }
}
```

输出结果：

```shell
+--------+-------------------- all ------------------------+----------------+
position: [12], limit: [12]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 65 6c 6c 6f 2c 77 6f 72 6c 64 0a             |Hello,world.    |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [14], limit: [14]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 49 27 6d 20 5a 68 61 6e 67 20 53 61 6e 0a       |I'm Zhang San.  |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [13], limit: [13]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 6f 77 20 61 72 65 20 79 6f 75 3f 0a          |How are you?.   |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0
```

### ByteBuffer 线程安全

⚠️ `ByteBuffer` 是非线程安全类，使用时要注意！！！

## 文件编程

### FileChannel

⚠️  `FileChannel` 只能工作在阻塞模式下。

#### 获取

不能直接打开 `FileChannel`，必须通过 `FileInputStream`、`FileOutputStream` 或者 `RandomAccessFile` 来获取 `FileChannel`，它们都有 `getChannel `方法：

* 通过 `FileInputStream` 获取的 `Channel` 只能读。
* 通过 `FileOutputStream` 获取的 `Channel` 只能写。
* 通过 `RandomAccessFile` 是否能读写根据构造 `RandomAccessFile` 时的读写模式决定。

#### 读取

调用 `FileChannel` 的读取方法时会从 `FileChannel` 读取数据填充至 `ByteBuffer`，返回值表示读到了多少字节，`-1` 表示到达了文件的末尾。

```java
int bytes = channel.read(buffer);
```

#### 写入

当调用 `Channel` 的写入方法时，需要考虑 `Channel` 的写入能力，其正确姿势如下：

```java
ByteBuffer buffer = ...;
buffer.put(...); // 存入数据
buffer.flip();   // 切换读模式

/*
write 方法并不能保证一次将 buffer 中的内容全部写入 channel,因此需要循环检测 buffer 中是否还有剩余数据
*/
while(buffer.hasRemaining()) {
    channel.write(buffer);
}
```

#### 关闭

`Channel` 必须关闭，不过调用了 `FileInputStream`、`FileOutputStream `或者 `RandomAccessFile` 的 `close()` 方法会间接地调用 `Channel` 的 ` close()` 方法，推荐使用 try with resource 优雅的关闭。

示例代码：

```java
public class FileChannelTwr {
    public static void main(String[] args) {
        try (FileChannel channel = new RandomAccessFile("d01/non-exist-file.txt", "rw").getChannel()) {
            ByteBuffer buffer = ByteBuffer.allocate(10);

            channel.read(buffer);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

对其字节码反编译如下：

```java
public class FileChannelTwr {
    public FileChannelTwr() {
    }

    public static void main(String[] args) {
        try {
            FileChannel channel = (new RandomAccessFile("d01/non-exist-file.txt", "rw")).getChannel();
            Throwable var2 = null;

            try {
                ByteBuffer buffer = ByteBuffer.allocate(10);
                channel.read(buffer);
            } catch (Throwable var12) {
                var2 = var12;
                throw var12;
            } finally {
                if (channel != null) {
                    if (var2 != null) {
                        try {
                            channel.close();
                        } catch (Throwable var11) {
                            var2.addSuppressed(var11);
                        }
                    } else {
                        channel.close();
                    }
                }

            }
        } catch (IOException var14) {
            var14.printStackTrace();
        }

    }
}
```

#### 位置

获取当前位置：

```java
long pos = channel.position();
```

设置当前位置：

```java
long newPos = ...;
channel.position(newPos);
```

设置当前位置时，如果设置为文件的末尾：

* 这时读取会返回 `-1` 。
* 这时写入，会追加内容，但要注意如果 position 超过了文件末尾，再写入时在新内容和原末尾之间会有空洞（00）。

示例代码：

```java
@Slf4j(topic = "d01.FileChannelPosition")
public class FileChannelPosition {
    public static void main(String[] args) {
        /*
        创建一个不存在的 non-exist-file.txt 随机文件读写对象
         */
        try (FileChannel channel = new RandomAccessFile("d01/non-exist-file.txt", "rw").getChannel()) {
            long position = channel.position();

            if (log.isDebugEnabled()) {
                log.debug("当前位置为：{}", position);
            }

            ByteBuffer buffer = ByteBuffer.allocate(10);

            /*
            从 Channel 中读取数据写入到 Buffer 中，由于读取的是一个不存在的文件，因此读取到的字节数为 0
             */
            final int readBytes = channel.read(buffer);

            /*
            position: [0], limit: [10]
             */
            ByteBufferUtil.debugAll(buffer);

            if (log.isDebugEnabled()) {
                log.debug("读取字节数：{}", readBytes);
            }

            /*
            设置 Channel 当前位置为 1，这超出了文件末尾
             */
            channel.position(1);

            buffer.put(new byte[]{'a', 'b', 'c'});

            /*
            position: [3], limit: [10]
             */
            ByteBufferUtil.debugAll(buffer);

            buffer.flip();

            /*
            position: [0], limit: [3]
             */
            ByteBufferUtil.debugAll(buffer);

            /*
            从 Buffer 中读取数据并通过 Channel 写入到文件中，因此必须先切换至 Buffer 读模式
             */
            channel.write(buffer);

            /*
            position: [3], limit: [3]
             */
            ByteBufferUtil.debugAll(buffer);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

```shell
16:50:12.632 d01.FileChannelPosition [main] - 当前位置为：0
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 00 00 00 00 00 00 00 00 00 00                   |..........      |
+--------+-------------------------------------------------+----------------+
16:50:12.649 d01.FileChannelPosition [main] - 读取字节数：-1
+--------+-------------------- all ------------------------+----------------+
position: [3], limit: [10]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 00 00 00 00 00 00 00                   |abc.......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [0], limit: [3]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 00 00 00 00 00 00 00                   |abc.......      |
+--------+-------------------------------------------------+----------------+
+--------+-------------------- all ------------------------+----------------+
position: [3], limit: [3]
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 62 63 00 00 00 00 00 00 00                   |abc.......      |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0
```

文件内容如下，出现了空洞：

```shell
 abc
```

通过编辑器打开显示如下：

![](filechannel-position-gt-end.png)

#### 大小

使用 `Channel` 的 `size()` 方法可以获取文件的大小。

#### 强制写入

操作系统出于性能的考虑，会将数据缓存，不是立刻写入磁盘。可以调用 `channel.force(true);` 方法将文件内容和元数据（文件的权限等信息）立刻写入磁盘。

### FileChannel 传输数据

`FileChannel` 的 `transferTo` 方法可以高效率的将源文件数据传输至目标文件中：

```java
public class FileChannelTransferTo {
    public static void main(String[] args) {
        try (
                FileChannel from = new RandomAccessFile("d01/from.txt", "rw").getChannel();
                FileChannel to = new FileOutputStream("d01/to.txt").getChannel()
        ) {
            final ByteBuffer buffer = ByteBuffer.allocate(4);

            buffer.put(new byte[]{'a', 'b', 'c', 'd'});

            buffer.flip();


            from.write(buffer);


            long start = System.nanoTime();

            /*
            效率高，底层利用操作系统的零拷贝进行优化
             */

            from.transferTo(0, from.size(), to);

            long end = System.nanoTime();

            System.out.println("transferTo 用时：" + (end - start) / 1000_000.0);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

```shell
transferTo 用时：1.044499

Process finished with exit code 0
```

