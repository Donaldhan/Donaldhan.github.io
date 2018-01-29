---
layout: page
title: NIO总结
subtitle: NIO总结
date: 2018-11-04 15:17:19
author: donaldhan
catalog: true
category: Java
categories:
    - Java
tags:
    - NIO
---

# 引言
在JDK1.4之前，Java的socket通信与文件的IO操作都是阻塞的，即一个动作的执行，必须等待先前的动作完成；从JDK1.4开始加入nio包，主要以通道的思想完整socket通信与文件的io操作，并且是非阻塞的，在线程执行一个动作，不用等待动作执行完，可以去做别的事情。NIO中通道主要有Socket，ServerSocket和Datagram通道，及文件通道和管道，通道读写操作主要利用ByteBuffer存放字节序列。
java NIO相关的文章列表如下:  
* [Java Socket读写缓存区Writer和Reader][]    
* [Java NIO ByteBuffer详解][]      
* [Java序列化与反序列化实例分析][]     
* [Java序列化与反序列化详解][]    
* [Java序列化与反序列化详解后续][]   
* [NIO-TCP通信实例(单线程，多线程Server)][]    
* [NIO-TCP简单实例][]  
* [Channel接口定义][]  
* [AbstractInterruptibleChannel接口定义][]  
* [SelectableChannel接口定义][]  
* [SelectionKey定义][]  
* [SelectorProvider定义][]  
* [AbstractSelectableChannel定义][]  
* [NetworkChannel接口定义][]  
* [ServerSocketChannel定义][]  
* [ServerSocketChannelImpl解析][]  
* [AbstractSelector定义][]  
* [SelectorImpl分析][]       
* [WindowsSelectorImpl解析一(FdMap，PollArrayWrapper)][]      
* [SocketChannel接口定义][]  
* [WindowsSelectorImpl解析二(选择操作，通道注册，通道反注册，选择器关闭等)][]  
* [SocketChannelImpl解析一(通道连接，发送数据)][]  
* [SocketChannelImpl解析二(发送数据后续)][]  
* [SocketChannelImpl解析三(接收数据)][]    
* [SocketChannelImpl解析四(关闭通道等)][]    
* [MembershipKey定义][]  
* [MulticastChanne接口定义][]  
* [MembershipKeyImpl简介][]  
* [DatagramChannel定义][]  
* [DatagramChannelImpl解析一(初始化)][]  
* [DatagramChannelImpl解析二(报文发送与接收)][]  
* [DatagramChannelImpl解析三(多播)][]    
* [DatagramChannelImpl解析四(地址绑定，关闭通道等)][]    







![nio](/image/NIO/nio.png)



# 目录
* [Java Socket通信实例](#Java Socket通信实例)
* [Java Socket读写缓存区Writer和Reader](#Java Socket读写缓存区Writer和Reader)
* [Java NIO ByteBuffer详解](#Java NIO ByteBuffer详解)      
* [Java序列化与反序列化实例分析](#Java序列化与反序列化实例分析)    
* [Java序列化与反序列化详解](#Java序列化与反序列化详解)    
* [Java序列化与反序列化详解后续](#Java序列化与反序列化详解后续)  
* [NIO-TCP通信实例(单线程，多线程Server)](#NIO-TCP通信实例(单线程，多线程Server))    
* [NIO-TCP简单实例](#NIO-TCP简单实例)  
* [Channel接口定义](#Channel接口定义)
* [AbstractInterruptibleChannel接口定义](#[AbstractInterruptibleChannel接口定义)  
* [SelectableChannel接口定义](#SelectableChannel接口定义)  
* [SelectionKey定义](#SelectionKey定义)
* [SelectorProvider定义](#[SelectorProvider定义)  
* [AbstractSelectableChannel定义](#AbstractSelectableChannel定义)  
* [NetworkChannel接口定义](#NetworkChannel接口定义)
* [ServerSocketChannel定义](#ServerSocketChannel定义)
* [ServerSocketChannelImpl解析](#ServerSocketChannelImpl解析)  
* [AbstractSelector定义](#AbstractSelector定义)
* [SelectorImpl分析](#SelectorImpl分析)      
* [WindowsSelectorImpl解析一(FdMap，PollArrayWrapper)](#WindowsSelectorImpl解析一(FdMap，PollArrayWrapper))      
* [SocketChannel接口定义](#SocketChannel接口定义)  
* [WindowsSelectorImpl解析二(选择操作，通道注册，通道反注册，选择器关闭等)](#WindowsSelectorImpl解析二(选择操作，通道注册，通道反注册，选择器关闭等))  
* [SocketChannelImpl解析一(通道连接，发送数据)](#SocketChannelImpl解析一(通道连接，发送数据))
* [SocketChannelImpl解析二(发送数据后续)](#SocketChannelImpl解析二(发送数据后续))  
* [SocketChannelImpl解析三(接收数据)](#SocketChannelImpl解析三(接收数据))    
* [SocketChannelImpl解析四(关闭通道等)](#SocketChannelImpl解析四(关闭通道等))   
* [MembershipKey定义](#MembershipKey定义)  
* [MulticastChanne接口定义](#MulticastChanne接口定义)  
* [MembershipKeyImpl简介](#MembershipKeyImpl简介)  
* [DatagramChannel定义](#DatagramChannel定义)
* [DatagramChannelImpl解析一(初始化)](#DatagramChannelImpl解析一(初始化))  
* [DatagramChannelImpl解析二(报文发送与接收)](#DatagramChannelImpl解析二(报文发送与接收))  
* [DatagramChannelImpl解析三(多播)](#DatagramChannelImpl 解析三(多播))    
* [DatagramChannelImpl解析四(地址绑定，关闭通道等)](#DatagramChannelImpl 解析四(地址绑定，关闭通道等))    

## Java Socket通信实例
java socket编程是阻塞模式的，即BIO，从socket获取InputStream，OutputStream，而InputStream，OutputStream要经过(BufferedInputStream,BufferedOutputStream),(DataInputStream，DataOutputStream)等的包装才可以写读socket的缓冲区；输入流skip函数，可以丢掉一些不必要的包，mark，reset函数可以标记读取位置，从标记位置从新读取；flush函数发送缓冲区里的所有数据，我们这里是强制清空缓冲区，实际不要这样做，以免影响数据传输效率。

## Java Socket读写缓存区Writer和Reader   
构造PrintWriter实际为初始化writer和是否自动刷新缓存，及初始化换行符，同时通过父类Writer，初始化写缓存同步锁；在构造PrintWriter的过程中，writer实际为
BufferedWriter，初始化BufferedWriter的过程，实际为初始化writer，缓冲区，缓冲区大小及位置和换行符，在构造BufferedWriter也需要传入writer，
这个writer实际为OutputStreamWriter，OutputStreamWriter初始化主要是初始化字节流编码器，字节流编码器StreamEncoder初始化，实际为初始化输出流是否打开状态，字节编码集，字节编码器,字节缓冲区,输出流OutputStream(从socket获取)。PrintWriter发送字符串，实际为将字符串通过BufferedWriter发送，BufferedWriter现将
字符串写入到其字节缓冲区中，如果缓冲区满，则发送缓存数据，发送委托给OutputStreamWriter，而OutputStreamWriter委托给StreamEncoder，有StreamEncoder将字节数组包装成CharBuffer，在通过编码器，编码字节到编码到字节流缓冲区bb(ByteBuffer)，如果字节流缓冲区bb(ByteBuffer)已满，则发送缓存数据。
InputStreamReader的构造实际上为初始化流解码器StreamDecoder，StreamDecoder初始化主要是，初始化输入流状态，字节流缓冲区，socket输入流；
BufferedReader构造的主要是，初始化缓冲区，缓冲区大小，及位置和创建Reader读缓冲区同步锁。从缓存读取数据实际上，先从socket输入流缓冲区通过流解码器StreamDecoder读取数据，解码填充到BufferedReader缓冲区中，BufferedReader从缓冲区中，读取一行数据。

## Java NIO ByteBuffer详解  
get*（int index）方法不改变mark，limit和capacity的值;put则回改变position的位置，put操作后position的位置为，put操作之前position+length（put 操作数）；
mark操作会改变mark的值，reset操作，则是将position定位到mark；clear操作并不会清空缓冲空间，而是将position复位0，limit为capacity，mark为-1；remain操作返回的是可用的空间大小为capacity-position；如put后，超出缓冲区大小，则抛出BufferOverflowException异常。
compact操作一般在一下情况调用，当out发送数据，即读取buf的数据，write方法可能只发送了部分数据，buf里还有剩余，这时调用buf.compact()函数将position与limit之间的数据，copy到buf的0到limit-position，进行压缩（非实际以压缩，只是移动），以便下次 向写入缓存。当position与limit之间的数据为空时，则不改变原缓冲区，否则copy相应数据。HeapByteBuffer向缓存中写入占多字节的原始类型Char，int，float等时，HeapByteBuffer，通过Bit将原始类型字节拆分存入到ByteBuffer的缓存中。

## Java序列化与反序列化实例分析   
反序列对象的时候，没有调用构造函数，而是使用字节流将对象属性，直接赋值。同时可以看sex（private transient String），由于有transient标识符，而没有被序列化 。
JDK中提供了另一个序列化接口Externalizable，使用该接口之后，之前基于Serializable接口的序列化机制就将失效。Externalizable序列化和反序列化调用的分别是对象的writeExternal，readExternal，而非writeObject和readObject，通是使用Externalizable进行序列化时，当读取对象时，会调用被序列化类的无参构造器去创建一个新的对象，然后再将被保存对象的字段的值分别填充到新对象中。这就是为什么在此次序列化过程中Person类的无参构造器会被调用。由于这个原因，实现Externalizable接口的类必须要提供一个无参的构造器，且它的访问权限为public。使用ObjectOutputStream和ObjectInputStream，序列化对象及原始类型，在网络中传输，没有任何问题。无论是实现Serializable接口，或是Externalizable接口，当从I/O流中读取对象时，readResolve()方法都会被调用到。实际上就是用readResolve()中返回的对象直接替换在反序列化过程中创建的对象

## Java序列化与反序列化详解
ObjectOutputStream的构造，主要是构造对象，对象输出流BlockDataOutputStream，BlockDataOutputStream主要是写序列化的流数据，BlockDataOutputStream中有一个存放序列化数据
的缓存区，头部数据缓冲区，原始类型数据缓冲区，其中关联一个原始类型输出流DataOutputStream，（DataOutput）主要用于写原始类型数据，int，float，String，double等;同时写流版本号和流魔数。然后，写类流对象描述，主要写类名，序列版本号，类型，属性个数，属性描述；然后写对象属性值，当对象父接口是Serializable，如果Serializable实现了WriteObject方法，则直接调用WriteObject序列化属性值，否则，通过反射获取对象属性的值，写到缓存中，如果对象父接口Externalizable，则直接调用对象writeExternal方法序列化对象。

## Java序列化与反序列化详解后续
ObjectInputStream初始化主要是，构造BlockDataInputStream，从缓冲中读取数据，然后读取流魔数与版本号；然后从缓冲中读取对象流描述信息，包括类型，属性名，及每个属性对应的读取位置；在加载流对象描述信息类；如果是Externalizable，则调用对象的readExternal的方法，如果是Serializable，有ReadObject，则通过反射调用ReadObject方法，没有从缓冲中读取属性数据，并给属性赋值；如果对象中有readResolve方法，则调用readResolve，并返回。

## NIO-TCP通信实例(单线程，多线程Server)   
在操作缓冲区Buffer时，要注意从通道读数据到缓冲区，及写缓冲区，或从缓冲区写数据到通道，即读取缓冲区，缓冲区读写模式转换是要调用flip函数，进行切换模式，
limit定位到position位置，然后position回到0；意思为缓冲区可读可写的数据量。put操作为写缓存区，get操作为读缓存区，当重用缓冲区，记得clear缓冲区，clear并不为清空缓冲区，至少将position至少为0，mark为-1，limit为capacity，再次写数据是将覆盖以前的数据。

## Channel接口定义
一个通道表示对一个实体的打开连接，比如硬件设备，文件，网络socket，或者一个应用组件可能执行一个或多个不同的IO操作，比如读写。通道有两个状态一个打开，一个关闭。通道在创建时打开，一旦关闭将会关闭。如果通道已经关闭，尝试执行IO操作，将会引起ClosedChannelException异常。判断一个通道是否打开，可以用isOpen方法。一般情况下，在实现Channel的具体接口和类中，必须保证多线程安全访问。如果当前线程close时，其他线程已将调用close，则当前线程阻塞，直至先前线程完成close。当前线程的close将无效。

## AbstractInterruptibleChannel接口定义
AbstractInterruptibleChannel是一个可以异步关闭和中断IO阻塞线程的通道，所有具体的通道实现，如果想要可以异步关闭和中断，必须实现此类。AbstractInterruptibleChannel内部有一个Open布尔值用于表示通道是否打开。在通道关闭时调用implCloseChannel，implCloseChannel方法完成实际的关闭通道工作。有个中断处理器用于记录中断阻塞IO操作线程的线程，完成实际的关闭通道工作。有一组协调方法为begin和end方法，一般在一个可能阻塞的IO操作的开始调用begin，之后调用end方法，这些操作一般用一个try语句块，组合使用。begin方法主要初始化中断处理器，end方法根据IO操作是否完成和Open状态，及中断线程处理器，中断线程判断是抛出AsynchronousCloseException异常还是ClosedByInterruptException。

## SelectableChannel接口定义
SelectableChannel是一个可选择的通道，可以注册到选择器，通道在创建时为阻塞模式，必选先通过configureBlocking方法，设置通道为非阻塞模式，才可以注册到注册器。我们可以通过validOps方法验证通道注册到选择器的事件，是否为通道支持的事件，可以通过isRegistered方法判断是否注册到选择器，用isBlocking方法判断通道是否为阻塞模式，用register方法将通道感兴趣的事件注册到选择器中，并返回一个注册器SelectionKey。

## SelectionKey定义
SelectionKey表示一个可选择通道与选择器关联的注册器，可以简单理解为一个token。SelectionKey包含两个操作集，分别是兴趣操作事件集interestOps和通道就绪操作事件集readyOps，每个操作集用一个Integer来表示。interestOps用于选择器判断在下一个选择操作的过程中，操作事件是否是通道关注的。兴趣操作事件集在SelectionKey创建时，初始化为注册选择器时的opt值，这个值可能通过interestOps(int)会改变。SelectionKey的readyOps表示一个通道已经准备就绪的操作事件，但不能保证在没有引起线程阻塞的情况下，就绪的操作事件会被线程执行。在一个选择操作完成后，大部分情况下就绪操作事件集会立即更新。如果外部的事件或在通道有IO操作，就绪操作事件集可能不准确。如果需要经常关联一些应用的特殊数据到SelectionKey，比如一个object表示一个高层协议的状态，object用于通知实现协议处理器。所以，SelectionKey支持通过attach方法将一个对象附加的SelectionKey的attachment上。attachment可以通过#attachment方法进行修改。SelectionKey定义了所有的操作事件，但是具体通道支持的操作事件依赖于具体的通道。所有可选择的通道都可以通过validOps方法，判断一个操作事件是否被通道所支持。测试一个不被通道所支持的通道，将会抛出相关的运行时异常。SelectionKey多线程并发访问时，是线程安全的。读写兴趣操作事件集的操作都将同步到，选择器的具体操作。同步器执行过程是依赖实现的：在一个本地实现版本中，如果一个选择操作正在进行，读写兴趣操作事件集也许会不确定地阻塞；在一个高性能的实现版本中，可能会简单阻塞。无论任何时候，一个选择操作在操作开始时，选择器总是占用着兴趣操作事件集的值。SelectionKey可以简单理解为通道和选择器的映射关系，并定义了相关的操作事件，分别为OP_READ，OP_WRITE，OP_CONNECT，OP_ACCEPT值分别是，int的值的第四为分别为1，级1,4，8,16。用一个AtomicReferenceFieldUpdater原子更新attachment。

## SelectorProvider定义
SelectorProvider就是为了创建DatagramChannel，Pipe，Selector，ServerSocketChannel，SocketChannel，System.inheritedChannel()而服务的，在相应的通道和选择器的open方法中调用系统默认的SelectorProvider相关的open*方法，创建相应的通道和选择器。SelectorProvider的provider方法主要是实例化SelectorProvider，过程为：判断java.nio.channels.spi.SelectorProvider系统属性是否被定义为一个具体的SelectorProvider实现类的唯一类名，是则加载此类，实例化，如果加载实例化失败，返回一个错误。如果无没有选择器提供者属性配置，则在SelectorProvider的实现且对系统类加载器可见Jar包中，的资源文件META-INF/services的目录下，提供了provider-configuration文件java.nio.channels.spi.SelectorProvider，则文件的第一个class类将会被加载和实例化，如果加载实例化失败，返回一个错误。上两步失败，则加载系统默认的选择器提供者。inheritedChannel方法主要是更具系统网络服务，更具具体的网络请求，创建不同的可继承实例，如果继承通道表示一个面向流的连接Socket，则SocketChannel将会被返回。SocketChannel初始化为阻塞模式，绑定一个socket地址，连接一个peer。如果继承通道表示一个面向流的监听socket，则ServerSocketChannel将会被返回。ServerSocketChannel初始化为阻塞模式，绑定一个socket地址。如果继承的通道是一个面向报文的Socket，则DatagramChannel将会被返回，初始化为阻塞模式，绑定一个socket地址。

## AbstractSelectableChannel定义
AbstractSelectableChannel有一个SelectorProvider类型的变量provider，主要是为创建通道而服务。一个选择key数组keys，保存与通道相关的选择key，一个key计数器keyCount，记录当前通道注册到选择器，生成的选择key。一个布尔blocking记录当前通道的阻塞模式。一个keyLock拥有控制选择key数据的线程安全访问。同时还有一个regLock控制通道注册选择器和配置通道阻塞模式的线程安全访问。提供了选择key集合keys的添加和移除，判断通道是否注册到选择器，及获取注册到选择器的选择key。注册通道到选择器过程为：首先验证通道是否打开，关注的操作事件是否有效，如果通道打开且事件有效，判断通道是注册到选择器，如果通道已经注册到选择器，则更新兴趣操作事件集，和附件对象，否则调用选择器的注册方法，并将返回的选择key添加到通道选择key集合。关闭通道所做的工作主要是，遍历通道的选择key数组，取消选择key。

## NetworkChannel接口定义

网络通道NetworkChannel，主要作用是绑定socket的local地址，获取绑定的地址，
以及设置或获取socket选项。

## ServerSocketChannel定义
ServerSocketChannel主要是绑定socket地址，监听Socket连接。 

## ServerSocketChannelImpl解析
## AbstractSelector定义
## SelectorImpl分析]     
## WindowsSelectorImpl解析一(FdMap，PollArrayWrapper)     
## SocketChannel接口定义
## WindowsSelectorImpl解析二(选择操作，通道注册，通道反注册，选择器关闭等)
## SocketChannelImpl解析一(通道连接，发送数据)
## SocketChannelImpl解析二(发送数据后续)
## SocketChannelImpl解析三(接收数据)  
## SocketChannelImpl解析四(关闭通道等)   
## MembershipKey定义
## MulticastChanne接口定义
## MembershipKeyImpl简介
## DatagramChannel定义
## [DatagramChannelImpl解析一(初始化)
## DatagramChannelImpl解析二(报文发送与接收)
## DatagramChannelImpl解析三(多播)
## DatagramChannelImpl解析四(地址绑定，关闭通道等)




## 总结



[Java Socket通信实例]:http://donald-draper.iteye.com/blog/2356695 "Java Socket通信实例"  
[Java Socket读写缓存区Writer和Reader]:http://donald-draper.iteye.com/blog/2356885 "Java Socket读写缓存区Writer和Reader"  
[Java NIO ByteBuffer详解]:http://donald-draper.iteye.com/blog/2356885 "Java NIO ByteBuffer详解"  
[Java序列化与反序列化实例分析]:http://donald-draper.iteye.com/blog/2357515 "Java序列化与反序列化实例分析"  
[Java序列化与反序列化详解]:http://donald-draper.iteye.com/blog/2357540 "Java序列化与反序列化详解"  
[Java序列化与反序列化详解后续]:http://donald-draper.iteye.com/blog/2357891 "Java序列化与反序列化详解后续"  
[NIO-TCP简单实例]:http://donald-draper.iteye.com/blog/2369044 "NIO-TCP简单实例"  
[NIO-TCP通信实例(单线程，多线程Server)]:http://donald-draper.iteye.com/blog/2369052 "NIO-TCP通信实例(单线程，多线程Server)"  
[Channel接口定义]:http://donald-draper.iteye.com/blog/2369111  "Channel接口定义"  
[AbstractInterruptibleChannel接口定义]:http://donald-draper.iteye.com/blog/2369238 "AbstractInterruptibleChannel接口定义"  
[SelectableChannel接口定义]:http://donald-draper.iteye.com/blog/2369317 "SelectableChannel接口定义"  
[SelectionKey定义]:http://donald-draper.iteye.com/blog/2369499 "SelectionKey定义"  
[SelectorProvider定义]:http://donald-draper.iteye.com/blog/2369615 "SelectorProvider定义"  
[AbstractSelectableChannel定义]:http://donald-draper.iteye.com/blog/2369742 "AbstractSelectableChannel定义"  
[NetworkChannel接口定义]:http://donald-draper.iteye.com/blog/2369773 "NetworkChannel接口定义"  
[ServerSocketChannel定义]:http://donald-draper.iteye.com/blog/2369836 "ServerSocketChannel定义"  
[ServerSocketChannelImpl解析]:http://donald-draper.iteye.com/blog/2370912 "ServerSocketChannelImpl解析"  
[Selector定义]:http://donald-draper.iteye.com/blog/2370015 "Selector定义"  
[AbstractSelector定义]:http://donald-draper.iteye.com/blog/2370138 "AbstractSelector定义"       
[SelectorImpl分析]:http://donald-draper.iteye.com/blog/2370519 "SelectorImpl分析"     

[WindowsSelectorImpl解析一(FdMap，PollArrayWrapper)]:http://donald-draper.iteye.com/blog/2370811 "WindowsSelectorImpl解析一(FdMap，PollArrayWrapper)"  
[WindowsSelectorImpl解析二(选择操作，通道注册，通道反注册，选择器关闭等)]:http://donald-draper.iteye.com/blog/2370862 "WindowsSelectorImpl解析二(选择操作，通道注册，通道反注册，选择器关闭等)"  
[SocketChannel接口定义]:http://donald-draper.iteye.com/blog/2371218 "SocketChannel接口定义"  
[SocketChannelImpl解析一(通道连接，发送数据)]:http://donald-draper.iteye.com/blog/2372364 "SocketChannelImpl解析一(通道连接，发送数据)"  
[SocketChannelImpl解析二(发送数据后续)]:http://donald-draper.iteye.com/blog/2372548 "SocketChannelImpl解析二(发送数据后续)"  
[SocketChannelImpl解析三(接收数据)]:http://donald-draper.iteye.com/blog/2372590 "SocketChannelImpl解析三(接收数据)"     
[SocketChannelImpl解析四(关闭通道等)]:http://donald-draper.iteye.com/blog/2372717 "SocketChannelImpl解析四(关闭通道等)"      
[MembershipKey定义]:http://donald-draper.iteye.com/blog/2372947 "MembershipKey定义"  
[MulticastChanne接口定义]:http://donald-draper.iteye.com/blog/2373009 "MulticastChanne接口定义"  
[MembershipKeyImpl简介]:http://donald-draper.iteye.com/blog/2373066 "MembershipKeyImpl 简介"  
[DatagramChannel定义]:http://donald-draper.iteye.com/blog/2373046 "DatagramChannel定义"  
[DatagramChannelImpl解析一(初始化)]:http://donald-draper.iteye.com/blog/2373245 "DatagramChannelImpl解析一(初始化)"  
[DatagramChannelImpl解析二(报文发送与接收)]:http://donald-draper.iteye.com/blog/2373281 "DatagramChannelImpl解析二(报文发送与接收)"  
[DatagramChannelImpl解析三(多播)]:http://donald-draper.iteye.com/blog/2373507 "DatagramChannelImpl解析三(多播)"    
[DatagramChannelImpl解析四(地址绑定，关闭通道等)]:http://donald-draper.iteye.com/blog/2373519 "DatagramChannelImpl解析四(地址绑定，关闭通道等)"  
