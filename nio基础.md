title: nio基础
categories: java基础
tags: 
	- io
---

### 什么是nio

Java NIO是java 1.4之后新出的一套IO接口，这里的的新是相对于原有标准的Java IO和Java Networking接口。NIO提供了一种完全不同的操作方式。

**NIO中的N可以理解为Non-blocking，不单纯是New**

标准的IO编程接口是面向字节流和字符流的。而NIO是面向通道和缓冲区的，数据总是从通道中读到buffer缓冲区内，或者从buffer写入到通道中

### nio核心组件

#### Channel通道

普通io是通过输入流和输出流来进行传输，nio只需要一个channel，channel是双向的，既可以写也可以读

特点
- 通道可以读也可以写，流一般来说是单向的(只能读或者写)
- 通道可以异步读写
- 通道总是基于缓冲区Buffer来读写

#### Buffer缓冲区

Java NIO Buffers用于和NIO Channel交互，我们从channel中读取数据到buffers里，从buffer把数据写入到channels。

buffer中有三个属性，position，limit，capacity

1. 初始状态下，position为0，limit=capacity
2. 使用put写入数据后，position为写入数据最后一个字节的索引(即写入数据的长度)
3. 在读取buffer中数据或者向channel写入buffer，需要调用flip，作用是将`limit=position`,然后position归0，保证只作用于有效数据。

#### Selector选择器

Selector是Java NIO中的一个组件，用于检查一个或多个NIO Channel的状态是否处于可读、可写。如此可以实现单线程管理多个channels,也就是可以管理多个网络链接。

##### 创建一个Selector

`Selector selector = Selector.open();`

##### 注册Channel到Selector上

```
channel.configureBlocking(false);//必须设置非阻塞通道
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

register的第二个参数，这个参数是一个“关注集合”，代表我们关注的channel状态，有四种基础类型可供监听：

- Connect
- Accept
- Read
- Write

一个channel触发了一个事件也可视作该事件处于就绪状态。因此当channel与server连接成功后，那么就是“连接就绪”状态。server channel接收请求连接时处于“可连接就绪”状态。channel有数据可读时处于“读就绪”状态。channel可以进行数据写入时处于“写就绪”状态。

上述的四种就绪状态用SelectionKey中的常量表示如下：

- SelectionKey.OP_CONNECT
- SelectionKey.OP_ACCEPT
- SelectionKey.OP_READ
- SelectionKey.OP_WRITE

如果对多个事件感兴趣可利用位的或运算结合多个常量，比如：

`int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;    `

##### SelectionKey

register方法把Channel注册到了Selectors上，这个方法的返回值是SelectionKey，这个返回的对象包含了以下属性：

- The interest set
- The ready set
- The Channel
- The Selector
- An attached object (optional)

Interest Set实际上就是我们希望处理的事件的集合，它的值就是注册时传入的参数，我们可以用按为与运算把每个事件取出来：

```
int interestSet = selectionKey.interestOps();

boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE; 
```

Ready Set中的值是当前channel处于就绪的值，为触发该channel的就绪状态

`int readySet = selectionKey.readyOps();`

Channel + Selector可以这样获得

```
Channel  channel  = selectionKey.channel();
Selector selector = selectionKey.selector(); 
```

Attaching Objects，我们可以给一个SelectionKey附加一个Object，这样做一方面可以方便我们识别某个特定的channel，同时也增加了channel相关的附加信息。例如，可以把用于channel的buffer附加到SelectionKey上：

```
selectionKey.attach(theObject);

Object attachedObj = selectionKey.attachment();
```

附加对象的操作也可以在register的时候就执行：

`SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);`

##### 获取就绪channel

一旦我们向Selector注册了一个或多个channel后，就可以调用select来获取channel。select方法会返回所有处于就绪状态的channel。 select方法具体如下：

- int select()
- int select(long timeout)
- int selectNow()

select()方法在返回channel之前处于阻塞状态。 select(long timeout)和select做的事一样，不过他的阻塞有一个超时限制。

selectNow()不会阻塞，根据当前状态立刻返回合适的channel。

select()方法的返回值是一个int整形，代表有多少channel处于就绪了。也就是自上一次select后有多少channel进入就绪。举例来说，假设第一次调用select时正好有一个channel就绪，那么返回值是1，并且对这个channel做任何处理，接着再次调用select，此时恰好又有一个新的channel就绪，那么返回值还是1，现在我们一共有两个channel处于就绪，但是在每次调用select时只有一个channel是就绪的。

在调用select并返回了有channel就绪之后，可以通过选中的key集合来获取channel，这个操作通过调用selectedKeys()方法：

`Set<SelectionKey> selectedKeys = selector.selectedKeys();    `

还记得在register时的操作吧，我们register后的返回值就是SelectionKey实例，也就是我们现在通过selectedKeys()方法所返回的SelectionKey。

遍历这些SelectionKey可以通过如下方法：

```
Set<SelectionKey> selectedKeys = selector.selectedKeys();

Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

while(keyIterator.hasNext()) {

    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
}
```

由于调用select而被阻塞的线程，可以通过调用Selector.wakeup()来唤醒即便此时已然没有channel处于就绪状态。具体操作是，在另外一个线程调用wakeup，被阻塞与select方法的线程就会立刻返回。

当操作Selector完毕后，需要调用close方法。close的调用会关闭Selector并使相关的SelectionKey都无效。channel本身不管被关闭。