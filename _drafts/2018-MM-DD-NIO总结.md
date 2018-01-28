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
## Java序列化与反序列化详解
## Java序列化与反序列化详解后续
## NIO-TCP通信实例(单线程，多线程Server)   
## NIO-TCP简单实例
## Channel接口定义
## AbstractInterruptibleChannel接口定义
## SelectableChannel接口定义
## SelectionKey定义
## SelectorProvider定义
## AbstractSelectableChannel定义
## NetworkChannel接口定义
## ServerSocketChannel定义
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
