- BIO
	1. ServerSocket.accept等待
	2. socket.getInptStream.read()时会再次等待
	3 .如果是单线程，一次只能响应一个客户端，但即使是有线程池进行响应如果客户端没有发送数据，该线程也会一直等待
- NIO
	1. 
	```
	ServerSocketChannel serverSocketChannel = ServerSockertChannel.open();
	ServerSocket serverSocket = serverSocket.socket();
	// 设置为非阻塞
	serverSocketChannel.configuraBlocking(false)
	// 底层为linux的accept
	while(true) {
		SocketChannel socketChannel = serverSocketChannel.accept();
		// 设置读非阻塞
		socketChannel.configuraBlocking(false)
		// 将socketChannel存起来然后循环读，即使是单线程也可以响应多个客户端，但无论有没有数据都会被循环
	}
	```
- 多路复用器
	```
	// 上述NIO不变，将多路复用注册到NIO
	Selector selector = Selector.open();
	// 注册连接事件
	serverSocketChannel.register(selector), OP_ACCET)
	while(true) {
		// 阻塞等待事件
		selector.select();
		// 如果是select()、poll()是循环所有，效率较低，在1.5之后采用了epoll()
		// 此时获取到的都是有事件的，可能是读，可能是写可能是连接。连接事件处理过一次是不会再有的
		Set<SelectionKey> selectionKeys = select.selectKeys();
		// 分别处理读、写事件，如果是连接事件循环再去注册读、写事件
	}
	```
// TODO linux版本的open()->provider()->create()会返回epoll
- AsynchronousServerSocketChannel
	accept直接变为回调