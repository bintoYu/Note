### NIO和BIO

IO的方式通常分为几种，同步阻塞的BIO、同步非阻塞的NIO、异步非阻塞的AIO
BIO和NIO的区别：
1. 面向流与面向缓冲
  IO是面向流的，NIO是面向缓冲区的。 Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节。而NIO则可以将数据先放入缓存区，稍后再读取想要的部分。（但其缺陷在于Thread需要反复检查buffer中是否包含所有想要处理的数据）
2. 阻塞与非阻塞
  BIO是阻塞的，意味如果数据为空或满时，读和写会发生阻塞，阻塞期间不能做其他任何事情。
  而NIO是非阻塞的，例如读的流程为：线程向channel发送读取请求，channel将数据放入buffer中，线程检查buffer中是否包含所有想要的数据，没有的话做其他事情，提高效率（写操作也是如此）。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190226092539555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
3. BIO：“一个线程一个连接”，即一个socket连接对应一个线程，NIO：“一个线程多个连接”，一个工作线程可以包含多个channel，即对应着多个连接。
### NIO中的buffer、channel、selector详解

##### Buffer：
- Buffer本质上是一块可读可写的内存。
- buffer的工作原理：
  --capacity ：容量
  -- position（位置）：从写切换到读时，position会重置为0，因此每次读都能读到之前放入的所有数据。
  --limit：限制写操作的数据大小
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190302130806727.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
- buffer的具体操作：
1. 分配buffer： `XXXBuffer buf =XXXBuffer.allocate(1024); `
2. 向Buffer中写数据：

```java
//方法一：channel --> buffer
int bytesRead = inChannel.read(buf); //read into buffer.  
//方法二：使用put方法
buf.put(127);  
```
3. 从写模式切换到读模式：
  ① flip()方法  : ，并将position重置为0，limit也重置为0；
  ② rewind()方法 ：并将position重置为0，limit保持不变；
4. 从Buffer中读取数据 ：同写数据

```
//方法一： buffer --> channel
int bytesWritten = inChannel.write(buf);  
//方法二： 使用get方法
byte aByte = buf.get();  
```

7.读模式切换到写模式：
① clear()方法：清空buffer，然后进入写模式：
②compact()方法：只会清除已经读过的数据；未读的数据会移到buffer的开头，新写入的数据会放到未读数据的后面。 

```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");  
FileChannel inChannel = aFile.getChannel();  
  
//create buffer with capacity of 48 bytes  
ByteBuffer buf = ByteBuffer.allocate(48);  
  
int bytesRead = inChannel.read(buf); //read into buffer.  
while (bytesRead != -1) {  
  
  buf.flip();  //make buffer ready for read  
  
  while(buf.hasRemaining()){  
      System.out.print((char) buf.get()); // read 1 byte at a time  
  }  
  
  buf.clear(); //make buffer ready for writing  
  bytesRead = inChannel.read(buf);  
}  
aFile.close();  
```

##### Channel：就是一个双向通道
 <img src="https://img-blog.csdnimg.cn/20190302132136783.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_4,color_FFFFFF,t_70" width="40%">

##### selector： 
Java NIO的选择器允许一个单独的线程来监视多个输入通道。
 <img src="https://img-blog.csdnimg.cn/20190302152749777.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70" width="60%">
下面是一个channel注册到selector然后获取信息的示例：

```java
//开启selector
Selector selector = Selector.open();  
channel.configureBlocking(false);  
//channel注册到selector上，并获得selectionKey
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);  
while(true) {  
  int readyChannels = selector.select();  
  if(readyChannels == 0) continue;  
  Set selectedKeys = selector.selectedKeys();  
  Iterator keyIterator = selectedKeys.iterator();  
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
```



### AIO
AIO是发出IO请求后，**由操作系统自己去获取IO权限并进行IO操作**；NIO则是发出IO请求后，由线程不断尝试获取IO权限，获取到后通知应用程序自己进行IO操作。