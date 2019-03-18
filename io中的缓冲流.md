title: io中的缓冲流
categories: java基础
tags: 
	- io
---

### 缓冲流的作用

io中有几个带有缓冲功能的装饰类，`BufferedInputStream`，`BufferedInputStream`，`BufferedReader`，BufferedWriter，不带缓冲的操作，每读一个字节就要写入一个字节，由于涉及磁盘的IO操作相比内存的操作要慢很多，所以不带缓冲的流效率很低。带缓冲的流，可以一次读很多字节，但不向磁盘中写入，只是先放到内存里。等凑够了缓冲区大小的时候一次性写入磁盘，这种方式可以减少磁盘操作次数，速度就会提高很多

### 源码分析

实现缓冲的主要承载是数组，在带缓冲的字节流和字符流中分别为

```
protected volatile byte buf[]
private char cb[];
```

他们的默认长度为
```
private static int DEFAULT_BUFFER_SIZE = 8192;
private static int defaultCharBufferSize = 8192;
```

查看字节流中关于从输入流中读取数据的源码，如下

```
/**
 * Reads a byte of data from this input stream. This method blocks
 * if no input is yet available.
 *
 * @return     the next byte of data, or <code>-1</code> if the end of the
 *             file is reached.
 * @exception  IOException  if an I/O error occurs.
 */
public int read() throws IOException {
    return read0();
}

private native int read0() throws IOException;

/**
 * Reads a subarray as a sequence of bytes.
 * @param b the data to be written
 * @param off the start offset in the data
 * @param len the number of bytes that are written
 * @exception IOException If an I/O error has occurred.
 */
private native int readBytes(byte b[], int off, int len) throws IOException;
```

`native`是用做java 和其他语言（如c++）进行协作时使用的，也就是`native` 后的函数的实现不是用java写的。从
文档中可以知道，`read0()`每次在输入流中读取一个字节的数据，而`readBytes(byte b[], int off, int len)`读取一段数据。

查看字符流中关于从输入流中读取数据的源码，找到字节流转化为字符流的类`InputStreamReader`

```
/**
 * Reads a single character.
 *
 * @return The character read, or -1 if the end of the stream has been
 *         reached
 *
 * @exception  IOException  If an I/O error occurs
 */
public int read() throws IOException {
    return sd.read();
}

/**
 * Reads characters into a portion of an array.
 *
 * @param      cbuf     Destination buffer
 * @param      offset   Offset at which to start storing characters
 * @param      length   Maximum number of characters to read
 *
 * @return     The number of characters read, or -1 if the end of the
 *             stream has been reached
 *
 * @exception  IOException  If an I/O error occurs
 */
public int read(char cbuf[], int offset, int length) throws IOException {
    return sd.read(cbuf, offset, length);
}
```

字节流转化为字符流依靠`StreamDecode`r这个类，创建`InputStreamRreader`时可以制定编码模式。可以看出两种方法也是同样分为了一次读取一个和一段的区别。

对于输出流也一样，就不再写了。

带缓冲作用的类在调用`read方`法的时候会首先从输入流读取默认长度8192字节或字符数据进入缓存数组中，即`read`方法中的`fill()`方法(当我们调用read方法一次获取字节或字符数据长度超过缓冲数组长度时，不使用缓冲数组，直接从输入流中获取我们需要的长度的数据，跳过`fill()`方法)，然后再从缓冲中取出我们需要的字节或字符数据。

### 效率比较

用来测试的文件为tif格式，大小为549MB字节的图片

1. 仅仅使用字节流，且不使用数组接收(使用数组的话，会调用readBytes方法，一次从输入流读取多个数据)

```
@org.junit.Test
public void test() {
	FileInputStream fis = null;
	FileOutputStream fos = null;
	try {
		fis = new FileInputStream("C:/Users/18813/Desktop/copy.tif");
		fos = new FileOutputStream("C:/Users/18813/Desktop/copy2.tif");
		int b;
		long start = System.currentTimeMillis();
		while((b=fis.read()) != -1){
			fos.write(b);
		}
		long cost = System.currentTimeMillis() - start;
		System.out.println("耗时:"+cost);
	} catch (Exception e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}finally {
		try {
			fis.close();
			fos.close();
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
}
```

耗时:2859775ms，约为47.66分钟，这种每次读取一个字节，然后写入一个字节的模式很耗时间

2. 下面使用普通字节流，但使用数组读取写入(io效率提升，每次读取和写入都是调用readBytes和writeBytes方法，一次读写多个数据)

```
@org.junit.Test
public void test2(){
	FileInputStream fis = null;
	FileOutputStream fos = null;
	try {
		fis = new FileInputStream("C:/Users/18813/Desktop/copy.tif");
		fos = new FileOutputStream("C:/Users/18813/Desktop/copy2.tif");
		byte b[] = new byte[4096];
		long start = System.currentTimeMillis();
		while((fis.read(b)) != -1){
			fos.write(b);
		}
		long cost = System.currentTimeMillis() - start;
		System.out.println("耗时:"+cost);
	} catch (Exception e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}finally {
		try {
			fis.close();
			fos.close();
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
}
```

耗时:1640ms，约为1.64秒，单看读写上的区别，第一种每次读写1个字节，第二种每次读写4096个字节，效率提升4096倍，但一次读写4096字节的数据花费时间也比读写1个字节多，所以实际只提升了1743倍。

3. 下面使用带缓冲的字节流，默认缓冲长度8192，但每次向缓冲中只读写1个字节。

```
@org.junit.Test
public void test3(){
	BufferedInputStream bis = null;
	BufferedOutputStream bos = null;
	try {
		bis = new BufferedInputStream(new FileInputStream("C:/Users/18813/Desktop/copy.tif"));
		bos = new BufferedOutputStream(new FileOutputStream("C:/Users/18813/Desktop/copy2.tif"));
		int b;
		long start = System.currentTimeMillis();
		while((b=bis.read()) != -1){
			bos.write(b);
		}
		long cost = System.currentTimeMillis() - start;
		System.out.println("耗时:"+cost);
	} catch (Exception e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}finally {
		try {
			bis.close();
			bos.close();
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
}
```

耗时:21837ms，约为21.837秒，比起第二种方法多使用了20秒，从源码上看区别是，这种方法在读写过程中需要通过输入流缓冲数组，它先读取8192字节数据到缓冲数组，然后再从缓冲数组中取1字节数据(全部取完后，重新读取8192字节数组到输入流缓冲数组)，再将这个字节写入到8192个字节的输出流缓冲数组，输出流缓冲数组满了后将缓冲数组中所有数组一次写入输出流。而第二种方法是直接从输入流和输出流中一次读写4096字节数组。

4. 将第三种方法改进，每次读写4096字节(8192字节的话，和默认缓冲数组长度相等，将不使用缓冲)

```
@org.junit.Test
public void test4(){
	BufferedInputStream bis = null;
	BufferedOutputStream bos = null;
	try {
		bis = new BufferedInputStream(new FileInputStream("C:/Users/18813/Desktop/copy.tif"));
		bos = new BufferedOutputStream(new FileOutputStream("C:/Users/18813/Desktop/copy2.tif"));
		byte b[] = new byte[4096];
		long start = System.currentTimeMillis();
		while((bis.read(b)) != -1){
			bos.write(b);
		}
		long cost = System.currentTimeMillis() - start;
		System.out.println("耗时:"+cost);
	} catch (Exception e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}
}
```
耗时:1098ms，约为1s，时间比第二种短的原因在默认的缓冲数组的长度为8192，每次读写8192字节长度数据，并且每次从缓冲流的读写为4096字节。但如果第二种方法每次读取长度不是4096而是大于8192，即缓冲数组的长度，则第二种方法速度更快。这样看来第二种方法有种手动使用缓冲的意思。