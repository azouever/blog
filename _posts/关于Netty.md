---

title: 关于Netty

date: 2020-09-01 20:59:13

---

> 阻塞、非阻塞，同步、异步通过使用A和B两个角色来进行介绍，暂时还没想好具体是什么。

###  关于阻塞和非阻塞

1. 阻塞：A问了B一个问题，在B没有给A回答之前，A什么都不做，就在等着。这个时候就是A被阻塞了。
2. 非阻塞：A问了B一个问题，B没有快速的响应A结果，A就去做别的事情去了，可能等一会儿会再去问B。这个时候A就是非阻塞。

###  关于同步和异步

1. 同步：A问了B一个问题，B快速的响应A，或者没有及时的响应，过了一会还是A继续来问B，B可能会给A响应，也可能没有，就是这个动作一直是A来进行发起，这个时候就是A获取答案就是一个同步的过程
2. 异步：A问了B一个问题，同时告诉B，如果你有答案，就是一会喊我一下，然后A就不管了，就去做别的事情了，等到B有了答案后就会自动的去告诉A。


### Netty框架理解


#### Buffer的继承体系

池化内存的分配

https://www.cnblogs.com/wuzhenzhao/category/1528244.html

#### Channel的继承体系

#### ChannelHandler的继承体系

#### ChannelHandlerContext

#### ChannelPipeline

#### ChannelOutboundInvoker/ChannelInboundInvoker

ChannelContext和Channelpipeline 都可以是Invoker


#### ChannelFuture

#### ChannelPromise



#### EventLoop


### 配置

ChannelConfig





### 对于Buffer的理解




### 参考
- 

