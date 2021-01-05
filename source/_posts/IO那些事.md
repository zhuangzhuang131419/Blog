---
title: IO那些事
date: 2020-07-10 12:43:01
tags:
---

本来计划七月OKR了解一下经常面经中看到但是不很了解的“I/O多路复用”，发现自己在整理序列化，反序列化的时候连基本的I/O也没有整明白，于是搜集资料准备好好梳理一下

# 为什么使用I/O
* 当我们的程序需要从硬盘，网络，或其他应用程序中读取或写入数据时候，数据传输量可能很大，而我们的内存或带宽有限，**无法一次性读取获取写入大量数据**。
* 而流（Stream）可以实现一点一点的逐步传输数据。
* 想想我们是怎样下载一个大文件的, 下载软件(例如x雷)并不会占用你内存很大的空间, 而只是在内存划分一个缓冲区, 一点一点地下载到自己的内存(缓冲区满了再写到硬盘)。

# I/O在Java中的架构
## I/O 的分类
### 从流的方向划分
* 输入流 (I)

    * 
        {% asset_img Input.jpg Input %}
* 输出流 (O)

    * 
        {% asset_img Output.jpg Output %}
### 从流的传输单位来分
* 字节流 (8位字节)
    * 每次读取（写出）一个字节，当传输资源有中文时，就会出现乱码。
    * 具体参见 为什么会乱码 // TODO
* 字符流 (16位字节)
    * 每次读取（写出）两个字节，有中文时，使用流程就可以正确传输显示中文。
## Java中的分类
* 四大基本类型
    | | 输入流 | 输出流 | 
    | - | - | - |
    | 字节流 | InputStream 字节输入流 | OutputStream 字符输出流 |
    | 字符流 | Reader 字符输入流 | Writer 字符输出流 | 
* 各自的子类都以父类作为自己的后缀，比如文件的字节输出流：FileOutputStream
## Java中的架构
* {% asset_img JavaIO体系架构图.png JavaIO体系架构图 %}
* 流的选择
    * 选用输入流还是输出流，根据具体的使用场景判断，如果是写程序到别的地方，那么就使用输出流。反之就是输入流。
    * 如果传输的数据有中文，那么选择字符流。
    * 再根据额外需要的功能具体选择。
## Java底层实现
### 字节输入流 InputStream
* 
    ```Java
    public abstract class InputStream implements Closeable {

        // Reads the next byte of data from the input stream.
        public abstract int read() throws IOException;


        // Closes this stream and releases any system resources associated with it.
        public void close() throws IOException {}

        /**
        Skips over and discards n bytes of data from this inputstream.
        @return     the actual number of bytes skipped.
        */
        public long skip(long n) throws IOException {
            // ...
        }


        // Returns an estimate of the number of bytes that can be read (or skipped over) from this input stream without blocking by the next invocation of a method for this input stream.
        public int available() throws IOException {
            return 0;
        }###

        // Marks the current position in this input stream
        // 如果从标记处开始往后，已经获取或者跳过了readLimit个字节，那么这个标记失效，不允许再重新通过reset回到这个位置。
        public synchronized void mark(int readlimit) {}

        public synchronized void reset() throws IOException {
            throw new IOException("mark/reset not supported");
        }
    }
    ```
#### ByteArrayInputStream
字节数组输入流，该类的功能就是从字节数组 byte[] 中进行以字节为单位的读取，也就是将资源文件都以字节形式存入到该类中的字节数组中去，我们拿数据也是从这个字节数组中拿。
#### PipedInputStream
管道字节输入流，它和 PipedOutputStream 一起使用，能实现多线程间的管道通信。
#### FilterInputStream
装饰者模式中充当装饰者的角色，具体的装饰者都要继承它，所以在该类的子类下都是用来装饰别的流的，也就是处理类。
#### BufferedInputStream
缓冲流，对处理流进行装饰、增强，内部会有一个缓冲区，用来存放字节，每次都是将缓冲区存满然后发送，而不是一个字节或两个字节这样发送，效率更高。
#### DataInputStream
数据输入流，用来装饰其他输入流，它允许通过数据流来读写Java基本类型。
#### FileInputStream
文件输入流，通常用于对文件进行读取操作。
#### File
对指定目录的文件进行操作。
#### ObjectInputStream
对象输入流，用来提供对“基本数据或对象”的持久存储。通俗点讲，就是能直接传输Java对象（序列化、反序列化用。
### 字节输出流 OutputStream
* 
    ```Java
    public abstract class OutputStream implements Closeable, Flushable {

        // Writes the specified byte to this output stream.
        public abstract void write(byte b[]) throws IOException;

        // Flushes this stream by writing any buffered output to the underlying stream.
        public void flush() throws IOException {}

        // Closes this stream and releases any system resources associated with it.
        public void close() throws IOException {}

    }
    ```
#### 缓冲区的作用
为了**加快数据传输速度**，**提高数据输出效率**，又是输出数据流会在提交数据之前把所要输出的数据先暂时保存在内存缓冲区中，然后成批进行输出，每次传输过程都以某特定数据长度为单位进行传输，在这种方式下，数据的末尾一般都会有一部分数据由于数量不够一个批次，而存留在缓冲区里，调用 flush() 方法可以将这部分数据强制提交。

### 字符输入流 Reader
{% asset_img 字符输入流.png 字符输入流 %}
Reader 中各个类的用途和使用方法基本和InputStream 中的类使用一致。

### 字符输出流 Writer
{% asset_img 字符输出流.png 字符输出流 %}

## 字节流与字符流的区别
* 字节流操作的基本单元为字节；字符流操作的基本单元为Unicode码元。
* 字节流默认不使用缓冲区；字符流使用缓冲区。
* 字节流通常用于处理二进制数据，实际上它可以处理任意类型的数据，但它不支持直接写入或读取Unicode码元；字符流通常处理文本数据，它支持写入及读取Unicode码元。

## 字节流与字符流使用场景
* 字节流：图像，视频，PPT, Word, 纯文本
* 字符流：纯文本类型（TXT), 不能处理图像视频等非文本类型的文件

# 参考文献
https://juejin.im/post/5d4ee73ae51d4561c94b0f9d
https://zhuanlan.zhihu.com/p/109941007