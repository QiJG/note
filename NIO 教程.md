# NIO 教程

## SelectionKey

SelectionKey是一个表示SelectableChannel 注册到Selector中的一个token。当一个channnel被注册到selector中后就会创建一个selectionKey。

### SelectionKey中维护的集合

* interest 集合

  ```java
  // interest 集合在注册完成之后可以通过 SelectionKey.interestOps(int) 方法进行修改
  private volatile int interestOps;
  ```

* ready集合

  ```java
  // ready集合表示channnel中的哪个操作已经准备好了，不能够被直接修改
  private int readyOps;
  ```

* 判断Channel支持的操作类型

  ```java
  SelectableChannel.validOps();
  // ServerSocketChannel支持 SelectionKey.OP_ACCEPT
  // SocketChannel支持 SelectionKey.OP_CONNECT,SelectionKey.OP_READ,SelectionKey.OP_WRITE
  ```

### 注册用法

* 客户端

  ```java
  // 连接就绪时间，当第一次连接未连接上时会进入连接就绪的准备
  SocketChannel socketChannel = SocketChannel.open();
          socketChannel.configureBlocking(false);
          Selector selector = Selector.open();
          socketChannel.register(selector, SelectionKey.OP_CONNECT);
          boolean established = socketChannel.connect(new InetSocketAddress("localhost", 8888));
          boolean loop = true;
          while (loop){
              if(selector.select() <=0){
                  continue;
              }
              Set<SelectionKey> selectionKeys = selector.selectedKeys();
              for (SelectionKey key : selectionKeys) {
                  if(key.isConnectable()){
                      SocketChannel channel = (SocketChannel) key.channel();
                      System.out.println("isConnected:" + channel.isConnected());
                      if(channel.isConnectionPending()){
                          System.out.println("ConnectionPending then invoke finishConnect");
                          if(channel.finishConnect()){
                              System.out.println("已经连接上客户端");
                              loop = false;
                          }
                      }
                  }
                  selectionKeys.remove(key);
              }
          }
  ```

* 服务端

### SelectionKey无效

* SelectionKey.cancel();

  会将 SelectionKey加入到Selector中的cancelledKeys，在下次select时进行移除

* Channel.close();

* Selector.close();

## Selector

### 三个集合：

* key set

  channel 注册时会加入到这里，candelled Keys会在下次select 操作时从key set中移除

* selectedKey set

  通过select操作会将key添加到selectedkEY SET

* cancelledKey set

  channel 关闭时或者 key调用cansel方法时会将key加入到candelledKey 集合

### Seclect 操作

select操作会执行三个步骤：

1. 将cancelledKey 集合中的key从 key 集合中移除，将对应的channel进行注销，这个步骤会清空candelledKey 集合
   1. 从keys移除该key
   2. 从selectedkeys移除该Key
   3. 从channel当中的keys移除该Key
   4. 修改该key的 valid为false
   5. 如果该key对应的channel是打开的，且是注册的，那么kill掉该channel
2. seletion操作开始后会由底层操作系统根据channel感兴趣的操作类型进行更新是否已经完成准备
3. 如果在步骤2中有key加入到candelledKey集合，在真正select操作后也会做一个清理cancelledKey的操作

* select()

  阻塞选择，唤醒情况为：channel被选择到；selector.wakeup()

* select(long)

  阻塞选择，唤醒情况为：channel被选择到；selector.wakeup();超时

* selectNow

  非阻塞选择

```
```



### wakeup操作

如果另一个线程由于select() select(long) 阻塞,那么wakeup会唤醒该阻塞，如果当前线程未被阻塞，那么会唤醒下一次的select操作

### close操作

* 取消channel的注册
* 如果channle处于开启状态，且没有任何注册，那么将channel进行kill

## Channel

### SelectableChannel

#### 方法介绍

```java
// 返回支持的操作类型
// ServerSocketChannel支持 SelectionKey.OP_ACCEPT
// SocketChannel支持 SelectionKey.OP_READ/OP_WRITE/OP_CONNECT
public abstract int validOps();
// 判断是否已经注册
// Channel当中会维护变量 SelectionKey[] keys和 int keyCount
// 根据keyCount判断是否完成注册
public abstract boolean isRegistered();
// 根据selector找到对应的SelectionKey
public abstract SelectionKey keyFor(Selector sel);
// 进行channel注册
// 1.将channel注册到selector
// 2.将selectionKey维护在channel中的 SelectionKey[] keys集合中
public abstract SelectionKey register(Selector sel, int ops, Object att);
// 配置channel为非阻塞
public abstract SelectableChannel configureBlocking(boolean block);
// 判断channel是阻塞或者非阻塞
public abstract boolean isBlocking();
```

### ServerSocketChannel

ServerSocketChannel通过该类的open方法进行创建。

ServerSocketChannel支持以下option选项

* SO_RCVBUF Socket receive buffer大小
* SO_REUSEADDR 重用地址

ServerSocketChannel是线程安全的。

ServerSocketChannel只支持 SelectionKey.OP_ACCEPT

```java
// 返回一个ServerSocketChannel
public static ServerSocketChannel open();
// 绑定一个本地地址
public final ServerSocketChannel bind(SocketAddress local);
// 设置channel的属性
public abstract <T> ServerSocketChannel setOption(SocketOption<T> name,T value);
// 如果当前channel 是非阻塞的，那么会立即返回一个Null(如果没有可用的connections)
// 如果当前channel 是阻塞的，那么它会阻塞直到一个连接可用
public abstract SocketChannel accept();
// 返回当前服务端地址
public abstract SocketAddress getLocalAddress();
```

### SocketChannel

SocketChannel通过该类的open方法创建，阻塞模式下，可以通过connect方法建立连接，可以通过isConnected()方法判断该channel是否已经连接上了。

非阻塞模式下：可以通过connect方法以及finishConnect方法建立连接，判断是否正在建立连接，可以通过isConnectionPending方法进行检测

SocketChannel支持 option选项为：

* SO_SNDBUF ：socket sen buffer大小
* SO_RCVBUF：socket receive buffer大小
* SO_KEEPALIVE：保持长链接
* SO_REUSEADDR：重用地址
* SO_LINGEG：延迟关闭时间
* NODELY：禁用nagle算法

```java
// 返回 SelectionKey.OP_CONNECT/OP_READ/OP_WRITE
public final int validOps();
// 关闭连接的读，但是并不关闭channel
// 一旦关闭连接的读，那么channel的read 会返回-1
public abstract SocketChannel shutdownInput();
// 关闭连接的写，但是不关闭channel
// 一旦关闭写操作，当往channel写数据时会抛出ClosedChannelException
public abstract SockChannel shutdownOutput();
// 判断channel是否已经连接上
public abstract boolean isConnected();
// 判断channel的连接操作是不是正在进行中
// 如果返回true，表示连接已经初始化，但是并没有连接上，可以通过finshConnect方法完成后续连接
public abstract boolean isConnectionPending();
// 如果channel是非阻塞的，这个方法的调用也是非阻塞的，如果立即建立了连接，那么会返回treu，否则会返回false,后续建立连接可以通过finshConnect方法完成
// 如果channel是阻塞的，那么会一直阻塞，直到连接成功
public abstract boolean connect(SocketAddress remote);
```

> 当channel关闭时 会获取channel当绑定的SelectionKey 并调用SelectionKey.cancel()方法