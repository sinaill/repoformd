title: io中的装饰者模式
categories: java基础
tags: 
	- 设计模式
	- io
---

### java io中的装饰者模式

以`inputStream`输入流为例，`inputStream`中的装饰者模式结构图：

![](http://wx1.sinaimg.cn/large/96b7c0f4gy1g0aqr1yy2fj20ge06j74k.jpg)

**装饰者模式的各个角色**:

**抽象构件（Component）角色**：由`InputStream`扮演。这是一个抽象类，为各种子类型流处理器提供统一的接口。
**具体构件（ConcreteComponent）角色**：由`ByteArrayInputStream`、`FileInputStream`、PipedInputStream以及StringBufferInputStream等原始流处理器扮演。他们实现了抽象构件角色所规定的接口，可以被链接流处理器所装饰。
**抽象装饰（Decorator）角色**：由`FilterInputStream`扮演。它实现了`InputStream`所规定的接口。
**具体装饰（ConcreteDecorator）角色**：由几个类扮演，分别是`DataInputStream`、`BufferInputStream`以及两个不常用的类`LineNumberInputStream`和`PushBackInputStream`。


----------


得到这样一个结构图就能更好的理解io，以装饰类`BufferedInputStream`为例，如果使用继承的方法，那么要为具体构件例如`ByteArrayInputStream`，`FileInputStream`等都添加缓冲功能，需要为他们都设计一个子类，而现在，只需要将他们传入`BufferedInputStream`中，就能获取相应的带缓冲的流。


首先看构造方法，需要传入一个普通字节输入流

```
public BufferedInputStream(InputStream in) {
    this(in, DEFAULT_BUFFER_SIZE);
}

public BufferedInputStream(InputStream in, int size) {
    super(in);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```

接着，装饰类中新定义一个字节数组作为缓冲区

`protected volatile byte buf[]`

接下来我们看装饰类是怎么让普通的输入流用上缓冲的,这是装饰类中的read方法，在调用普通输入流的read方法前后进行了一系列操作(对read方法进行修饰)。

```
public synchronized int read(byte b[], int off, int len)
    throws IOException
{
    getBufIfOpen(); // Check for closed stream
    if ((off | len | (off + len) | (b.length - (off + len))) < 0) {
        throw new IndexOutOfBoundsException();
    } else if (len == 0) {
        return 0;
    }

    int n = 0;
    for (;;) {
        int nread = read1(b, off + n, len - n);//此处跳入read1方法
        if (nread <= 0)
            return (n == 0) ? nread : n;
        n += nread;
        if (n >= len)
            return n;
        // if not closed but no bytes available, return
        InputStream input = in;
        if (input != null && input.available() <= 0)
            return n;
    }
}

private int read1(byte[] b, int off, int len) throws IOException {
    int avail = count - pos;
    if (avail <= 0) {
        /* If the requested length is at least as large as the buffer, and
           if there is no mark/reset activity, do not bother to copy the
           bytes into the local buffer.  In this way buffered streams will
           cascade harmlessly. */
        if (len >= getBufIfOpen().length && markpos < 0) {
            return getInIfOpen().read(b, off, len);//这里调用了普通输入流的read方法，读取长度超过缓冲默认长度时，不使用缓冲直接读取。
        }
        fill();//填充缓冲区，进入这个方法
        avail = count - pos;
        if (avail <= 0) return -1;
    }
    int cnt = (avail < len) ? avail : len;
    System.arraycopy(getBufIfOpen(), pos, b, off, cnt);
    pos += cnt;
    return cnt;
}


private void fill() throws IOException {
    byte[] buffer = getBufIfOpen();
    if (markpos < 0)
        pos = 0;            /* no mark: throw away the buffer */
    else if (pos >= buffer.length)  /* no room left in buffer */
        if (markpos > 0) {  /* can throw away early part of the buffer */
            int sz = pos - markpos;
            System.arraycopy(buffer, markpos, buffer, 0, sz);
            pos = sz;
            markpos = 0;
        } else if (buffer.length >= marklimit) {
            markpos = -1;   /* buffer got too big, invalidate mark */
            pos = 0;        /* drop buffer contents */
        } else if (buffer.length >= MAX_BUFFER_SIZE) {
            throw new OutOfMemoryError("Required array size too large");
        } else {            /* grow buffer */
            int nsz = (pos <= MAX_BUFFER_SIZE - pos) ?
                    pos * 2 : MAX_BUFFER_SIZE;
            if (nsz > marklimit)
                nsz = marklimit;
            byte nbuf[] = new byte[nsz];
            System.arraycopy(buffer, 0, nbuf, 0, pos);
            if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
                // Can't replace buf if there was an async close.
                // Note: This would need to be changed if fill()
                // is ever made accessible to multiple threads.
                // But for now, the only way CAS can fail is via close.
                // assert buf == null;
                throw new IOException("Stream closed");
            }
            buffer = nbuf;
        }
    count = pos;
    int n = getInIfOpen().read(buffer, pos, buffer.length - pos);//在这个地方又再次调用普通输入流的read方法，一次读取默认长度8192个长度的数据到缓冲区
    if (n > 0)
        count = n + pos;
}
```
就这样，BufferedInputStream就成了带缓冲区的字节输入流，通过使用装饰者模式，对普通输入流的read方法添加修改，扩充了普通输入流的功能。