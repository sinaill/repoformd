title: nio与io简单聊天室demo
categories: java基础
tags: 
	- io
---

### io聊天室demo

client端

```
public static void main(String[] args) throws Exception {
	Socket Client = new Socket("localhost", 8888);
	DataInputStream dis = new DataInputStream(Client.getInputStream());
	DataOutputStream dos = new DataOutputStream(Client.getOutputStream());
	BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
	//开启一个线程用于读服务器传来的数据
	new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				DataInputStream dis = new DataInputStream(Client.getInputStream());
				while (true) {
					String msgr = dis.readUTF();
				}
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	}).start();;
	//开启一个线程 用来读取键盘输入然后传输给服务器
	new Thread(new Runnable() {
		
		@Override
		public void run() {
			String msg;
			try {
				while (true) {
					
					msg = br.readLine();
					dos.writeUTF(msg);
					dos.flush();
				}
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	}).start();;
//		dis.close();
//		dos.close();
//		br.close();
//		Client.close();
}
```

server端

```java
public static void main(String[] args) throws Exception {
	List<Socket> sockets = new ArrayList<Socket>();
	ServerSocket server = new ServerSocket(8888);
	while(true) {
		Socket socket = server.accept();
		sockets.add(socket);
		//由于io阻塞，每一个客户端连接都开启一个线程监听
		new Thread(new Runnable() {
			Socket mySocket = socket;
			@Override
			public void run() {
				DataInputStream dis = null;
				DataOutputStream dos = null;
				String name = "";
				try {
					dis = new DataInputStream(mySocket.getInputStream());
					dos = new DataOutputStream(mySocket.getOutputStream());
					dos.writeUTF("输入名字");
					name = dis.readUTF();
				} catch (IOException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
				
				while(true) {
					String msg = "";
					try {
						msg = dis.readUTF();
						//消息分发给其他客户端
						for (Socket socket : sockets) {
							if(mySocket == socket) {
								continue;
							}
							dos = new DataOutputStream(socket.getOutputStream());
							dos.writeUTF(name+":"+msg);
							dos.flush();
						}
					} catch (IOException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
					
				}
			}
		}).start();
	}


}
```

### nio聊天室demo

client端

```java
public static void main(String[] args) {
	
	try {
		SocketChannel sc = SocketChannel.open();
		sc.connect(new InetSocketAddress("127.0.0.1", 8888));
		Selector selector = Selector.open();
		sc.configureBlocking(false);
		sc.register(selector, SelectionKey.OP_READ);

		BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
		ByteBuffer writeBuffer = ByteBuffer.allocate(50);
		ByteBuffer readBuffer = ByteBuffer.allocate(50);
		//开启线程用于读取键盘输入然后写入服务器
		new Thread(new Runnable() {
			@Override
			public void run() {
				while (true) {
					String msg;
					try {
						msg = br.readLine();
						writeBuffer.clear();
						writeBuffer.put(msg.getBytes());
						writeBuffer.flip();
						sc.write(writeBuffer);
					} catch (IOException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
				}
			}
		}).start();
		//selector监听channel
		while (true) {
			selector.select();
			Set<SelectionKey> selectedKeys = selector.selectedKeys();
			Iterator<SelectionKey> it = selectedKeys.iterator();
			while (it.hasNext()) {
				SelectionKey selectionKey = (SelectionKey) it.next();
				if (selectionKey.isReadable()) {
					SocketChannel channel = (SocketChannel) selectionKey.channel();
					readBuffer.clear();
					channel.read(readBuffer);
					System.out.println(new String(readBuffer.array()));
				}
				it.remove();
			}
		}
		
	} catch (Exception e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}
}
```

服务端

```java
public static void main(String[] args) {
	List<SocketChannel> socketChannels;
	ServerSocketChannel ssc = null;
	Selector selector = null;
	try {
		//存放所有客户端socketchannel，用于消息分发
		socketChannels = new ArrayList<SocketChannel>();
		ssc = ServerSocketChannel.open();
		ssc.socket().bind(new InetSocketAddress(8888));
		selector = Selector.open();
		ssc.configureBlocking(false);
		ssc.register(selector, SelectionKey.OP_ACCEPT);
		ByteBuffer buff = ByteBuffer.allocate(50);
		while (true) {
			selector.select();
			Set<SelectionKey> selectedKeys = selector.selectedKeys();
			Iterator<SelectionKey> it = selectedKeys.iterator();
			while (it.hasNext()) {
				SelectionKey key = it.next();
				if(key.isAcceptable()) {
					ServerSocketChannel sscTemp = (ServerSocketChannel) key.channel();
					SocketChannel sc = sscTemp.accept();
					sc.configureBlocking(false);
					sc.register(selector, SelectionKey.OP_READ);
					socketChannels.add(sc);
				}
				
				if(key.isReadable()) {
					
					buff.clear();
					SocketChannel sc = (SocketChannel) key.channel();
					sc.read(buff);
					//消息分发
					socketChannels.forEach(socketChannel->{
						if(sc!=socketChannel) {
							try {
								buff.flip();
								socketChannel.write(buff);
							} catch (IOException e) {
								// TODO Auto-generated catch block
								e.printStackTrace();
							}
						}
					});
					System.out.println(new String(buff.array()));
					
				}
				
				it.remove();
			}
		}
	} catch (Exception e) {
		e.printStackTrace();
	}
}
```

### nio非阻塞特性

不用像io一样，因为读写都是阻塞的，所以对每个客户端连接都要建一个线程来监听，而nio只需要一个线程使用selector监听所有channel，减轻服务器负担