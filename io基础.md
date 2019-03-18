title: 装饰者模式
categories: java基础
tags: 
	- 设计模式
---

### 流的概念

流是一种有顺序的，有起点和终点的字节集合，是对数据传输的总成或抽象。即数据在两设备之间的传输称之为流，流的本质是数据传输，根据数据传输的特性讲流抽象为各种类，方便更直观的进行数据操作。

###io流的分类

根据数据处理类的不同分为：字符流和字节流。
根据数据流向不同分为：输入流和输出流。

### 字符流和字节流

实际上最根本的流都是字节流，因为计算机上数据存储的最终形式实际上都是字节。而字符流的由来是因为：不同文字的数据编码不同，所以才有了对字符进行高效操作的流对象。本质其实就是基于字节流读取时，去查了指定的码表。所以，字节流和字符流的区别在于：

1. 读写单位不同：字节流以字节（8bit）为单位，字符流以字符为单位，根据码表映射字符，一次可能读多个字节。
2. 处理对象不同：字节流能处理所有类型的数据（如图片、avi等），而字符流只能处理字符类型，即纯文本的数据。

### java流类结构图

通过一张关系图来了解一下Java中的IO流构成：

![](http://wx4.sinaimg.cn/large/96b7c0f4gy1fzorzopl97j20j30l70tu.jpg)

### IO流的具体应用

#### FileinputStream/FileoutputStream

从类名就看出是用来操作文件的字节流，将c盘中一张jpg格式的图片复制并更改文件名

```
public class Copy {
	public static void main(String[] args){
		InputStream in = null;
		OutputStream out = null;
		try {
			in = new FileInputStream("c:/1.jpg");
			out = new FileOutputStream(new File("c:/2.jpg"));
			byte buffer[] = new byte[20];
			while(in.read(buffer) != -1){
				out.write(buffer);
			}
		} catch (FileNotFoundException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}catch (IOException e) {
			// TODO: handle exception
			e.printStackTrace();
		}finally {
			try {
				in.close();
				out.close();
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
			
		}
	}
}
```

#### FileReader/FileWriter

从类名能看出是操作文件的字符流(实际上是简单装饰了父类`InputStreamReader/OutputStreamWriter`，这两个字符流仅仅只有构造函数没有覆写方法，可以当作可接收字符流的`InputStreamReader/OutputStreamWriter`)，将一个txt文件的内容复制到一个新建的名为test2的txt文件里。

```
public class Copy2 {
	public static void main(String[] args) {
		Reader reader = null;
		Writer writer = null;
		try {
			reader = new FileReader("c:/test.txt");
			writer = new FileWriter(new File("c:/test2.txt"));
			char buff[] = new char[20];
			while(reader.read(buff) != -1){
				writer.write(buff);
			}
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}finally {
			try {
				reader.close();
				writer.close();
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	}
}
```

#### BufferedReader/BufferedWriter

这两个类为带缓冲的字符流，采用装饰者模式对普通字符流进行装饰，构造方法传入一个`Reader/Writer`，内部维护一个8192长度的字符数组作为缓冲，当使用`public int read(char cbuf[], int offset, int length)`一次读取字符数超过缓冲默认长度8192，将不使用缓冲，直接读取到我们自定义的字符数组中(另一个输出流同理)。关闭输出流时，`close`方法清空缓冲并将缓冲中剩余字符写入。

```
public class BufferCopy {
	public static void main(String[] args) {
		Reader reader = null;
		Writer writer = null;
		try {
			reader = new BufferedReader(new FileReader("c:/test.txt"));
			writer = new BufferedWriter(new FileWriter("c:/test2.txt"));
			int ch;
			while((ch = reader.read()) != -1){
				writer.write(ch);
			}
		} catch (Exception e) {
			// TODO: handle exception
		}finally {
			try {
				reader.close();
				writer.close();
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	}
}
```

#### BufferedInputStream/BufferedOutputStream

带缓冲的字节流，采用装饰者模式对普通字节流进行装饰，构造方法传入一个`InputStream`，内部维护8192长度的字节数组作为缓冲，当使用`public synchronized int read(byte b[], int off, int len)`一次读取字节数超过8192个字节，将不使用缓冲，直接读取到我们自定义的字节数组中(另一个输出流同理)，关闭输出流时，`close`方法清空缓冲并将剩余字节写入。

```
public class BufferCopy2 {
	public static void main(String[] args) {
		BufferedInputStream bis = null;
		BufferedOutputStream bos = null;
		try {
			bis = new BufferedInputStream(new FileInputStream("c:/test.txt"));
			bos = new BufferedOutputStream(new FileOutputStream("c:/test2.txt"));
			int b;
			while((b=bis.read()) != -1){
				bos.write(b);
			}
		} catch (Exception e) {
			// TODO: handle exception
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
}
```

#### InputStreamReader/OutputStreamWriter

这两个类是转化流，就是将字节流转化为字符流，通过构造函数传入的字节流和编码，使用`StreamDecoder/StreamEncode`将传进来的字节输入流/字节输出流编码为字符流。`StreamDecoder`的`read/write`实现字符的读写。

```
public class Transfer {
	public static void main(String[] args) {
		Reader isr = null;
		Writer osw = null;
		try {
			isr = new InputStreamReader(new FileInputStream("c:/test.txt"));
			osw = new OutputStreamWriter(new FileOutputStream("c:/test2.txt"));
			int ch;
			while((ch = isr.read()) != -1){
				osw.write(ch);
			}
		} catch (Exception e) {
			// TODO: handle exception
		}finally {
			try {
				isr.close();
				osw.close();
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	}
}
```