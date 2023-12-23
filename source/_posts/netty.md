---
title: netty
thumbnail: /img/nio/netty-logo.jpg
cover: /img/nio/netty-RoundCorner.png
categories: 
- java
- 网络编程
tags: 网络编程
toc: true
---
nio 虽然使用了多路复用，但是真正实现一个网络编程还需要考虑很多问题，所以出现了netty这种基于nio的网络编程框架，来帮助我们简化开发网络相关问题的难度，netty使用异步的方式来接收各种事件，并提供链式pipeline的方式帮助我们处理数据，并且增强了nio的ByteBuffer，还提供了一系列例如心跳、异步等方式来帮助我们构建代码
<!--more-->

### netty架构
从网上找了两张图，netty的总体架构，以及netty的线程模型如下

![](/img/nio/0041.png)


这张图表明了netty可以有多个eventloopGroup 每个eventLoopGroup中包含多个eventLoop，可以帮助我们异步执行一些io事件，并且还表明netty中有多条流水线，其中每个流水线pipline都必然有一个head和tail的双向链表来对应入站和出站，一般来说，出战需要编码，入站需要解码，如果pipeline上的事件需要处理很久，那么可以交给链上的下一个处理器处理，也可以放入任务队列执行，这样变相提高了处理速度
![](/img/nio/nio_eventloop.png)

通过eventLoop可以拿到channel，进而处理我们的io事件


### 上一个栗子
#### 最简单的netty服务器
一个简单的netty的网络编程🌰
```java
// 服务器端
package com.hyf.demo.netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import lombok.extern.slf4j.Slf4j;

import java.nio.charset.Charset;

@Slf4j
public class NettyServerDemo {


    public static void main(String[] args) throws InterruptedException {

        // 可以给某个handler指定特定的NioEventLoopGroup
        EventLoopGroup defaultEventLoopGroup = new DefaultEventLoopGroup();
        //  线程数默认是这么多 private static final int DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt("io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));
        ChannelFuture channelFuture = new ServerBootstrap()
                // 一个只处理accept 事件  一个处理其他事件  accept事件的NioEventLoopGroup不管设置几，其实到最后都是1
                .group(new NioEventLoopGroup(2), new NioEventLoopGroup(2))
                .channel(NioServerSocketChannel.class)
                // 这里添加的处理器都是给SocketChannel的 而不是给ServerSocketChannel的
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel nioSocketChannel) throws Exception {

                        // 加个日志
                        nioSocketChannel.pipeline().addLast(new LoggingHandler());
                        // 拿到pipeline 设置解码器
//                        nioSocketChannel.pipeline().addLast(new StringDecoder());
                        // 这里为什么用ChannelInboundHandlerAdapter 而不是ChannelInboundHandler
                        // 是因为使用了适配器模式，可以指定哪些方法我可以实现，而不是强制实现
                        // 这里如果方法要向下传递 其实可以使用ChannelInboundHandlerAdapter  如果要直接释放 可以使用SimpleChannelInboundHandler
                        nioSocketChannel.pipeline().addLast("handler1", new ChannelInboundHandlerAdapter() {

                            @Override
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                ByteBuf buf = (ByteBuf) msg;
                                log.info(buf.toString(Charset.defaultCharset()));
                                // 会向下一个handler传递数据  // 现在使用的ChannelInboundHandlerAdapter 所以需要调用这一句 如果是SimpleChannelInboundHandler就不需要了
                                ctx.fireChannelRead(msg);
                            }
                        }).addLast(defaultEventLoopGroup, "handler2", new ChannelInboundHandlerAdapter() {

                            @Override
                            public void channelRead(ChannelHandlerContext channelHandlerContext, Object msg) throws Exception {
                                ByteBuf buf = (ByteBuf) msg;
                                // 这里打出的线程和handler1 打出来的不一样
                                log.info(buf.toString(Charset.defaultCharset()));
                            }
                        });

                    }
                    // 绑定服务端端口
                }).bind(9999);

        // 能在关闭后做一些处理
        channelFuture.channel().closeFuture().sync();
        // 确保是在连接关闭做的处理
        log.info("close do something...");
        // 优雅关闭一些eventLoopGroup中的线程
        defaultEventLoopGroup.shutdownGracefully();

    }
}


// client 
package com.hyf.demo.netty;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.handler.logging.LoggingHandler;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class NettyClientDemo {

    public static void main(String[] args) throws InterruptedException {

        NioEventLoopGroup eventExecutors = new NioEventLoopGroup(2);
        ChannelFuture channelFuture = new Bootstrap().group(eventExecutors)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<Channel>() {
                             @Override
                             protected void initChannel(Channel channel) throws Exception {
                                 channel.pipeline().addLast(new LoggingHandler());
                                 channel.pipeline().addLast(new StringEncoder());
                             }
                         }
                )
                // 异步非阻塞
                .connect("localhost", 9999)
                // 由于netty中都是异步的 所以要sync等待
                .sync()
                // 获取抽象过后的channel通道
                .channel()
                // 写入消息并清空缓冲区
                .writeAndFlush("hello");

        // 能在关闭后做一些处理
        channelFuture.channel().closeFuture().sync();
        // 确保是在连接关闭做的处理
        log.info("close do something...");
        // 优雅关闭一些eventLoopGroup中的线程
        eventExecutors.shutdownGracefully();
    }
}

```
上面这个例子是一个最简单的服务器、客户端编程模版代码，一般来说，我们的代码也就是根据这个模版来扩展了，他的具体流程如下

### 代码流程图
![](/img/nio/netty_liuchen.png)

以上流程说明了以下几点
1. netty是异步的，可以分配workGroup 线程池 和 boosGroup线程池，启动是通过BootStrap来启动的
2. NioEventLoopGroup 提供了next接口，可以按照负载均衡策略来指定哪个NioEventLoop来工作
 * 因为实现了ScheduledExecutorService 所以可以执行定时任务、普通任务 以及Io事件的处理 但是DefaultEventLoopGroup处理不了IO事件
3. 把 channel 理解为数据的通道
4. 把 msg 理解为流动的数据，最开始输入是 ByteBuf，但经过 pipeline 的加工，会变成其它类型对象，最后输出又变成 ByteBuf
5.  把 handler 理解为数据的处理工序
  * 工序有多道，合在一起就是 pipeline，pipeline 负责发布事件（读、读取完成...）传播给每个 handler， handler 对自己感兴趣的事件进行处理（重写了相应事件处理方法）
  * handler 分 Inbound 和 Outbound 两类
* 把 eventLoop 理解为处理数据的工人
  * 工人可以管理多个 channel 的 io 操作，并且一旦工人负责了某个 channel，就要负责到底（绑定）
  * 工人既可以执行 io 操作，也可以进行任务处理，每位工人有任务队列，队列里可以堆放多个 channel 的待处理任务，任务分为普通任务、定时任务
  * 工人按照 pipeline 顺序，依次按照 handler 的规划（代码）处理数据，可以为每道工序指定不同的工人


### ByteBuf
> ByteBuf是netty在nio ByteBuffer上做的增强

针对于ByteBuf 有以下几点需要注意
1. ByteBuf可以开辟直接内存  也可以开辟堆内存 直接内存需要手动管理释放
2. ByteBuf具备池化和非池化能力 池化提高工作效率，但是复杂度变高
3. 可以使用slice切片  来模拟半包现象
4. 他有很多和零拷贝相关的类，比如copy\duplicate\slice等


### 粘包半包现象原因

粘包半包出现的原因有以下几点
1. 应用层有接收和发送缓冲区限制
2. 传输层有滑动窗口机制
3. 网络层有MTU MCC限制

### 粘包半包解决方案

在netty中有以下几种解决方案
1. 短连接 每次发送完数据就断开连接 可以解决粘包问题 解决不了半包问题
2. netty提供的定长解码器 ，可以解决问题 但是浪费空间
3. 分隔符解码器 ，可以解决问题 但是效率低 需要找到分隔符
4. 帧解码器:*LengthFieldBasedFrameDecoder* 这个解码器提供的参数解释如下
 * maxFrameLength 超过这个长度了还没完 就报错
 * lengthFieldOffset 长度字段偏移量（就是代表实际内容有多长的的长度字段是从第几个字节开始的）
 * lengthFieldLength 用来记录内容长度的字段的实际字节是多少
 * lengthAdjustment 长度字段为基准 还有几个字段是内容
 * initialBytesToStrip 从头剥离几个字节（相当于在最终的结果是没有剥离的那几个字节的）

 ### 自定义http协议的一个例子
 ```java
 NioEventLoopGroup boss = new NioEventLoopGroup();
NioEventLoopGroup worker = new NioEventLoopGroup();
try {
    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.channel(NioServerSocketChannel.class);
    serverBootstrap.group(boss, worker);
    serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            ch.pipeline().addLast(new LoggingHandler(LogLevel.DEBUG));
            ch.pipeline().addLast(new HttpServerCodec());
            ch.pipeline().addLast(new SimpleChannelInboundHandler<HttpRequest>() {
                @Override
                protected void channelRead0(ChannelHandlerContext ctx, HttpRequest msg) throws Exception {
                    // 获取请求
                    log.debug(msg.uri());

                    // 返回响应
                    DefaultFullHttpResponse response =
                            new DefaultFullHttpResponse(msg.protocolVersion(), HttpResponseStatus.OK);

                    byte[] bytes = "<h1>Hello, world!</h1>".getBytes();

                    response.headers().setInt(CONTENT_LENGTH, bytes.length);
                    response.content().writeBytes(bytes);

                    // 写回响应
                    ctx.writeAndFlush(response);
                }
            });
            /*ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                    log.debug("{}", msg.getClass());

                    if (msg instanceof HttpRequest) { // 请求行，请求头

                    } else if (msg instanceof HttpContent) { //请求体

                    }
                }
            });*/
        }
    });
    ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
    channelFuture.channel().closeFuture().sync();
} catch (InterruptedException e) {
    log.error("server error", e);
} finally {
    boss.shutdownGracefully();
    worker.shutdownGracefully();
}
 ```

运行后访问浏览器 lcoalhost:8080 浏览器打出hello word

### 自定义协议的例子

自定义协议需要满足以下要求

* 魔数，用来在第一时间判定是否是无效数据包
* 版本号，可以支持协议的升级
* 序列化算法，消息正文到底采用哪种序列化反序列化方式，可以由此扩展，例如：json、protobuf、hessian、jdk
* 指令类型，是登录、注册、单聊、群聊... 跟业务相关
* 请求序号，为了双工通信，提供异步能力
* 正文长度
* 消息正文

一个🌰
```java
package com.hyf.demo.netty;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.ByteBufAllocator;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageCodec;
import io.netty.handler.codec.MessageToMessageCodec;
import lombok.extern.slf4j.Slf4j;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.List;

@Slf4j
public class MessageCodecSharable extends MessageToMessageCodec<ByteBuf, Message> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Message msg, List<Object> outList) throws Exception {
        // 将msg内容 写入netty帮助我们创建的Bytebuf中
//        ByteBuf out = ctx.alloc().buffer();
        ByteBuf out  = ByteBufAllocator.DEFAULT.buffer(1024);
        // 1. 4 字节的魔数
        out.writeBytes(new byte[]{1, 2, 3, 4});
        // 2. 1 字节的版本,
        out.writeByte(1);
        // 3. 1 字节的序列化方式 jdk 0 , json 1
        out.writeByte(0);
        // 4. 1 字节的指令类型
        out.writeByte(msg.getMessageType());
        // 5. 4 个字节
        out.writeInt(msg.getSequenceId());
        // 无意义，对齐填充
        out.writeByte(0xff);
        // 6. 获取内容的字节数组
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(msg);
        byte[] bytes = bos.toByteArray();
        // 7. 长度
        out.writeInt(bytes.length);
        // 8. 写入内容
        out.writeBytes(bytes);
        outList.add(out);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        int magicNum = in.readInt();
        byte version = in.readByte();
        byte serializerType = in.readByte();
        byte messageType = in.readByte();
        int sequenceId = in.readInt();
        in.readByte();
        int length = in.readInt();
        byte[] bytes = new byte[length];
        in.readBytes(bytes, 0, length);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bytes));
        Message message = (Message) ois.readObject();
        log.debug("{}, {}, {}, {}, {}, {}", magicNum, version, serializerType, messageType, sequenceId, length);
        log.debug("{}", message);
        // 为了给下一个handler使用
        out.add(message);
    }
}

// 测试类
package com.hyf.demo.netty;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.ByteBufAllocator;
import io.netty.buffer.Unpooled;
import io.netty.channel.embedded.EmbeddedChannel;
import io.netty.handler.logging.LoggingHandler;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class Test {

    public static void main(String[] args) throws Exception {
        EmbeddedChannel channel = new EmbeddedChannel(
                new LoggingHandler(),
//                new LengthFieldBasedFrameDecoder(
//                        1024, 12, 4, 0, 0),
                new MessageCodecSharable()
        );
// encode
//        Message message = new Message("zhangsan", "123");
        Message message = new Message(1, 2);

//        channel.writeOutbound(message);
// decode
        ByteBuf buf = ByteBufAllocator.DEFAULT.buffer();
        List<Object> messageList = new ArrayList<>();
        new MessageCodecSharable().encode(null, message, messageList);
//
        System.out.println(messageList);


        channel.writeInbound(convertListToByteBuf(messageList));
//        ByteBuf s1 = buf.slice(0, 100);
//        ByteBuf s2 = buf.slice(100, buf.readableBytes() - 100);
//        s1.retain(); // 引用计数 2
//        channel.writeInbound(s1); // release 1
//        channel.writeInbound(s2);
    }

    public static ByteBuf convertListToByteBuf(List<Object> list) {
        // 创建一个新的ByteBuf
        ByteBuf byteBuf = Unpooled.buffer();

        // 创建一个字节数组输出流
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();

        try {
            // 创建ObjectMapper对象
            ObjectMapper objectMapper = new ObjectMapper();

            // 将列表转换为字节数组
            byte[] byteArray = objectMapper.writeValueAsBytes(list);


            // 将字节数组输出流的内容写入ByteBuf
            byteBuf.writeBytes(byteArrayOutputStream.toByteArray());

            byteBuf.writeBytes(byteArray);

        } catch (IOException e) {
            e.printStackTrace();
        }

        return byteBuf;
    }
}
```