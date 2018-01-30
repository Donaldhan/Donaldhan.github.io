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
* [Selector定义][]  
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
* [Selector定义](#Selector定义)
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
ServerSocketChannelImpl的初始化主要是初始化ServerSocket通道线程thread，地址绑定，接受连接同步锁，默认创建ServerSocketChannelImpl的状态为未初始化，文件描述和文件描述id，如果使用本地地址，则获取本地地址。bind首先检查ServerSocket是否关闭，是否绑定地址，如果既没有绑定也没关闭，则检查绑定的socketaddress是否正确或合法；然后通过Net工具类的bind（native）和listen（native），完成实际的ServerSocket地址绑定和开启监听，如果绑定是开启的参数小于1，则默认接受50个连接。accept方法主要是调用accept0（native）方法接受连接，并根据接受来接
文件描述的地址构造SocketChannelImpl，并返回。

## Selector定义
选择器接口主要提供了，打开选择器，选择、唤醒操作，获取注册到选择器的key和已经选择的选择key集合。

## AbstractSelector定义
AbstractSelector取消的key放在一个set集合中，对集合进行添加操作时，必须同步取消key set集合。反注册选择key完成的实际工作是，将key，从key对应的通道的选择key数组中移除。

## SelectorImpl分析]    
SelectorImpl有4个集合分别为就绪key集合，key集合，key集合的代理publicKeys及就绪key集合的代理publicSelectedKeys；实际是两个集合就绪key集合和key集合，publicSelectedKeys和publicKeys是其他线程访问上述两个集合的代理。SelectorImpl构造的时候，初始化选择器提供者SelectorProvider，创建就绪key集合和key集合，然后初始化就绪key和key集合的代理，初始化过程为，如果nio包的JDK版本存在bug问题，则就绪ke和key集合的代理集合直接引用就绪key和key集合。否则将当前key集合包装成不可修改的代理集合publicKes，将就绪key集合包装成容量固定的集合publicSelectedKeys。其他线程获取选择器的就绪key和key集合，实际上返回的是key集合的代理publicKeys和就绪key集合的代理publicSelectedKeys。
select方法的3中操作形式，实际上委托给为lockAndDoSelect方法，方法实际上是同步的，可安全访问，获取key集合代理publicKeys和就绪key代理集合publicSelectedKeys，然后交给doSelect(long l)方法，这个方法为抽象方法，待子类扩展。实际的关闭选择器操作implCloseSelector方法，首先唤醒等待选择操作的线程，唤醒方法wakeup待实现，同步选择器，就绪key和key集合的代理publicKeys，publicSelectedKeys，调用implClose完成实际的关闭通道工作，待子类实现。可选通道注册方法，首先注册的通道必须是AbstractSelectableChannel类型，并且是SelChImpl实例。更具可选择通道和选择器构造选择key，设置选择key的附加物，同步key集合代理，调用implRegister方法完成实际的注册工作，implRegister方法待子类实现。
processDeregisterQueue方法，主要是遍历取消key集合，反注册取消key，实际的反注册工作由implDereg方法，implDereg方法待子类扩展。成功，则从集合中移除。

## WindowsSelectorImpl解析一(FdMap，PollArrayWrapper)     
WindowsSelectorImpl默认加载net和nio资源库；WindowsSelectorImpl内锁4个，分别为关闭锁closeLock，中断锁interruptLock，startLock，finishLock后面两个的作用，目前还不清楚，后面再说；一个唤醒管道，作用尚不明确；一个注册到选择器的通道计数器totalChannels；updateCount计数器作用，尚不明确；通道集合channelArray，存放的元素实际为通道关联的选择key；pollWrapper用于存储选择key和相应的兴趣事件，及唤醒管道的源通道，唤醒管道的源通道存放在pollWrapper的索引0位置上。FdMap主要是存储选择key的，FdMap实际上是一个HashMap，key为选择key的文件描述id，value为MapEntry，MapEntry为选择key的包装Entry，里面含有更新计数器updateCount和清除计数器clearedCount。PollArrayWrapper存放选择key和通道及其相关的操作事件。PollArrayWrapper通过AllocatedNativeObject来存储先关的文件描述及其兴趣事件，AllocatedNativeObject为已分配的底层内存空间，AllocatedNativeObject的内存主要NativeObject来分配，NativeObject实际是通过Unsafe来分配内存。PollArrayWrapper作用即存放选择key和选择key关注的事件，用选择key的文件描述id，表示选择key，文件描述id为int，所以占4个字节，选择key的兴趣操作事件也为int，即4个字节，所以SIZE_POLLFD为8，文件描述id开始位置FD_OFFSET为0，兴趣事件开始位置EVENT_OFFSET为4；FD_OFFSET和EVENT_OFFSET都是相对于SIZE_POLLFD的。PollArrayWrapper同时存储唤醒等待选择操作的选择器的通道和唤醒通道关注事件即通道注册选择器事件，即添加选择key事件。当有通道注册到选择器，则唤醒通道，唤醒等待选择操作的选择器。

## WindowsSelectorImpl解析二(选择操作，通道注册，通道反注册，选择器关闭等)
implRegister方法，首先同步关闭锁，以防在注册的过程中，选择器被关闭；检查选择器是否关闭，没有关闭，则检查是否扩容，需要则扩容为pollWrapper为原来的两倍；检查过后，添加选择key到选择器通道集合，设置key在选择器通道集合的索引，添加选择key到文件描述fdMap，添加key到key集合，将选择key添加到文件描述信息及关注操作事件包装集合pollWrapper，通道计数器自增。
implDereg方法首选判断反注册的key是不是在通道key尾部，不在交换，并将交换信息更新到pollWrapper，从fdMap，keys，selectedKeys集合移除选择key，并将key从通道中移除。
SubSelector主要有两个方法以poll从pollWrapper拉取关注读写事件的选择key；processSelectedKeys方法主要是更新关注读写事件的选择key的相关通道的已经就绪的操作事件集。
StartLock主要控制选择线程，startThreads方法为唤醒所有等待选择操作的线程，运行计数器runsCounter自增，waitForStart方法为，判断选择线程是否需要等待开始锁。
FinishLock用于控制线程集合中的选择线程，完成锁只有在所有线程集合中的执行完，才释放，waitForHelperThreads方法为等待完成锁，threadFinished方法为当前选择线程
已结束，更新完成的选择线程计数器threadsToFinish（减一），reset方法重置threadsToFinish为线程集合大小。SelectThread线程启动时等待startLock，从pollWrapper拉取索引index对应的关注读写事件的选择key如果运行异常，则设置finishLock的finishLock，运行结束则更新完成选择操作线程计数器（自减）。doSelect方法将选择操作分成多个选择线程SelectThread放在选择线程放在threads集合中，每个SelectThread使用SubSelector从当前注册到选择器的通道中选取SubSelector索引所对应的批次的通道已经就绪的通道并更新操作事件。整个选择过程有startLock和finishLock来控制。再有在一个选择操作的所有子选择线程执行完，才释放finishLock。下一个选择操作才能开始，即startLock可用。wakeup主要是通过sink通道发送信息给source通道（native实现），通知子选择线程可以进行选择操作。子选择线程选择主要处理相应批次的1024个通道就绪事件（每批次通道关联到source通道）。implClose方法主要关闭唤醒管道的sink和source通道，反注册选择器的所有通道，释放所有通道空间，结束所有选择线程集合中的线程

## SocketChannel接口定义
socket通道继承的接口 ByteChannel， ByteChannel主要是继承了可读（ReadableByteChannel）可写（WritableByteChannel）通道接口和分散（ScatteringByteChannel）聚集（ScatteringByteChannel）通道接口；可读通道接口，可以从通道读取字节序列写到缓存区；可写通道接口，可以从缓存区读取字节序列写到通道；分散通道可以从通道读取字节序列，写到一组缓存区中，聚集通道可以从一组缓存区读取字节序列，写到通道。
socket通道接口主要提供了的连接，完成连接，是否正在建立连接，读缓冲区写到通道，聚集写，读通道写缓冲区等操作。

## SocketChannelImpl解析一(通道连接，发送数据)
SocketChannelImpl构造主要是初始化读写及状态锁和通道socket文件描述。connect连接方法首先同步读锁和写锁，确保socket通道打开，并没有连接；然后检查socket地址的正确性与合法性，然后检查当前线程是否有Connect方法的访问控制权限，最后尝试连接socket地址。从缓冲区读取字节序列写到通道write（ByteBuffer），首先确保通道打开，且输出流没有关闭，然后委托给IOUtil写字节序列；IOUtil写字节流过程为首先通过Util从当前线程的缓冲区获取可以容下字节序列的临时缓冲区（DirectByteBuffer），如果没有则创建一个DirectByteBuffer，将字节序列写到临时的DirectByteBuffer中，然后将写操作委托给nativedispatcher（SocketDispatcher），将DirectByteBuffer添加到当前线程的缓冲区，以便重用，因为DirectByteBuffer实际上是存在物理内存中，频繁的分配将会消耗更多的资源。

## SocketChannelImpl解析二(发送数据后续)
SocketChannelImpl写ByteBuffer数组方法，首先同步写锁，确保通道，输出流打开，连接建立委托给IOUtil，将ByteBuffer数组写到输出流中，这一过程为获取存放i个字节缓冲区的IOVecWrapper，遍历ByteBuffer数组m，将字节缓冲区添加到iovecwrapper的字节缓冲区数组中，如果ByteBuffer非Direct类型，委托Util从当前线程的缓冲区获取容量为j2临时DirectByteBuffer，并将ByteBuffer写到DirectByteBuffer，并将DirectByteBuffer添加到iovecwrapper的字节缓冲区（Shadow-Direct）数组中，将字节缓冲区的起始地址写到iovecwrapper，字节缓冲区的实际容量写到iovecwrapper；遍历iovecwrapper的字节缓冲区（Shadow-Direct）数组，将Shadow数组中的DirectByteBuffer通过Util添加到本地线程的缓存区中，并清除DirectByteBuffer在iovecwrapper的相应数组中的信息；最后通过SocketDispatcher，将iovecwrapper的缓冲区数据，写到filedescriptor对应的输出流中。

## SocketChannelImpl解析三(接收数据)  
读输入流到buffer，首先同步读写，确保通道，输入流打开，通道连接建立，清除原始读线程，获取新的本地读线程，委托IOUtil读输入流到buffer；IOUtil读输入流到buffer，首先确保buffer是可写的，否则抛出IllegalArgumentException，然后判断buffer是否为Direct类型，是则委托给readIntoNativeBuffer，否则通过Util从当前线程缓冲区获取一个临时的DirectByteBuffer，然后通过readIntoNativeBuffer读输入流数据到临时的DirectByteBuffer，这一个过程是通过SocketDispatcher的read方法实现，读写数据到DirectByteBuffer中后，将DirectByteBuffer中数据，写到原始buffer中，并将DirectByteBuffer添加到添加临时DirectByteBuffer到当前线程的缓冲区，以便重用，因为重新DirectByteBuffer为直接操作物理内存，频繁分配物理内存，将耗费过多的资源。
从输入流读取数据，写到ByteBuffer数组的read方法，首先同步写锁，确保通道，连接建立，输入流打开，委托给IOUtil，从输入流读取数据写到ByteBuffer数组中；IOUtil首先获取存放i个字节缓冲区的IOVecWrapper，遍历ByteBuffer数组m，将buffer添加到iovecwrapper的字节缓冲区数组中，如果ByteBuffer非Direct类型，委托Util从当前线程的缓冲区获取容量为j2临时DirectByteBuffer，并将ByteBuffer写到DirectByteBuffer，并将DirectByteBuffer添加到iovecwrapper的字节缓冲区（Shadow-Direct）数组中，将字节缓冲区的起始地址写到iovecwrapper，字节缓冲区的实际容量写到iovecwrapper；遍历iovecwrapper的字节缓冲区（Shadow-Direct）数组，将Shadow数组中的DirectByteBuffer通过Util添加到本地线程的缓存区中，并清除DirectByteBuffe在iovecwrapper的相应数组中的信息；最后通过SocketDispatcher，从filedescriptor对应的输入流读取数据，写到iovecwrapper的缓冲区中。

## SocketChannelImpl解析四(关闭通道等)   
实际关闭通道，同步状态锁，置输入流和输出流打开状态为false，如果通道没有关闭，则通过SocketDispatcher预先关闭fd，通知读线程，关闭输入流，通知写线程，输出流关闭，如果当前没有注册到任何选择器，则调用kill完成实际关闭工作，即SocketDispatcher关闭fd。

## MembershipKey定义
多播关系key接口主要提供了判断key是否有效，阻塞和解阻塞socket地址等操作。

## MembershipKeyImpl简介
MembershipKeyImpl内部有一个多播关系key关联的多播通道和多播分组地址，及多播报文源地址，及一个地址阻塞集。MembershipKeyImpl主要操作为drop关系key，直接委托个多播通道drop方法；block地址，首先判断多播关系key中的阻塞Set中是否包含对应的地址，有，则直接返回，否则委托给DatagramChannelImpl的block方法，完成实际的阻塞工作，然后添加地址的多播关系key阻塞set；unblock，首先判断多播关系key中的阻塞Set中是否包含对应的地址，无，则直接返回，有则委托给DatagramChannelImpl的unblock方法，完成实际的的解除阻塞工作，并从多播关系key中的阻塞Set移除对应的地址。

## MulticastChanne接口定义
多播通道提供了关闭多播通道和加入多播组操作。

## DatagramChannel定义
报文通道接口，主要定义了绑定，连接socket地址，断开连接，读写字节Buffer，接受和发送字节buf操作。

## [DatagramChannelImpl解析一(初始化)
DatagramChannelImpl主要成员有报文socket分发器，这个与SocketChannleImpl中的socket分发器原理基本相同，报文socket分发器可以理解为报文通道的静态代理；网络协议family表示当前报文通道的网络协议family；多播关系注册器MembershipRegistry，主要是通过一个Map-HashMap<InetAddress,LinkedList<MembershipKeyImpl>>来管理多播组和多播组成员关系key的映射（关系）；通道本地读写线程记录器，及读写锁控制通道读写，一个状态锁，当通道状态改变时，需要获取状态锁。DatagramChannelImpl构造方法，主要是初始化读写线程，及读写锁和状态锁，初始化网络协议family，及报文通道描述符和文件描述id。DatagramChannelImpl(SelectorProvider selectorprovider)与其他两个不同的是构造时更新当前报文socket的数量。

## DatagramChannelImpl解析二(报文发送与接收)
send（发送报文）方法，首先同步写锁，确保通道打开，然后检查地址，如果系统安全管理器不为null，则更具地址类型检查相应的权限，如果地址为多播地址，则检查多播权限，否则检查连接到socketaddress的权限；如果发送的buffer为direct类型，则直接发送，否则从当前线程缓冲区获取一个临时DirectByteBuffer，并将buffer中的数据写到临时DirectByteBuffer中，然后发送，发送后，释放临时DirectByteBuffer，即添加到当前线程缓存区以便重用。receive（接收报文）方法，首先同步读锁，确保通道打开，如果本地地址为null，则绑定local地址，并初始化报文通道的localAddress；获取buffer当前可用空间remaining，如果buffer为direct类型，则直接接收报文，否则，从当前线程缓冲区获取临时DirectByteBuffer，接收报文，写到临时缓冲区临时DirectByteBuffer，读取临时DirectByteBuffer，写到buffer中，时DirectByteBuffer，即添加DirectByteBuffer到当前线程缓存区，以便重用。
send（发送报文）和receive（接收报文）方法不需要通道已经处于连接状态，而read和write需要通道建立连接状态，这种方式与SocketChannel的读写操作相同，这样与SocketChannel无异，如果要不如使用SocketChannel。如果使用DatagramChannel,建议使用send和recieve方法进行报文的发送和接收。

## DatagramChannelImpl解析三(多播)
join(报文通道加入多播组)方法，首先检查加入的多播组地址是否正确，然后校验源地址，检查多播成员关系注册器中是否存在多播地址为inetaddress，网络接口为networkinterface，源地址为inetaddress1的多播成员关系key，有则直接返回，否则根据网络协议族family，网络接口，源地址构造多播成员关系MembershipKeyImpl，添加到注册器MembershipRegistry。
阻塞源地址报文与解除源地址报文阻塞，首先检查源地址，再将实际的阻塞与解除阻塞工作委托给Net完成。drop方法，首先判断多播成员关系key是否有效，如果有效，判断多播组为ip4还是ip6，然后委托给Net完成实际的drop工作。

## DatagramChannelImpl解析四(地址绑定，关闭通道等)
关闭通道实际完成的工作为更新系统报文socket计数器，即自减1；注册器不为null，则使注册器中的所有多播组无效；通知本地读写线程，通道已关闭；委托报文分发器DatagramDispatcher关闭文件描述。


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
