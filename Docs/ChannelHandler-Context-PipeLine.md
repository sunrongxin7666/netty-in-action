# Netty源码分析-2: ChannelHandler && ChannelPipeline
@(Java)[netty]

## ChannelHandler
### SimpleChannelInboundHandler
```
// SimpleChannelInboundHandler可以指定要处理的消息的类型I
public abstract class SimpleChannelInboundHandler<I> extends ChannelInboundHandlerAdapter {

    private final TypeParameterMatcher matcher;//类型匹配器
    private final boolean autoRelease;//是否自动回收资源

	//默认构造器中是启动自动回收资源的
    protected SimpleChannelInboundHandler() {
        this(true);
    }

    protected SimpleChannelInboundHandler(boolean autoRelease) {
        matcher = TypeParameterMatcher.find(this, SimpleChannelInboundHandler.class, "I");//TypeParameterMatcher为Netty自定义类，用于检查传输而来的消息是否为指定类型
        this.autoRelease = autoRelease;
    }
    
    public boolean acceptInboundMessage(Object msg) throws Exception {
        return matcher.match(msg);//检查msg是否为I类型
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        boolean release = true;
        try {
            if (acceptInboundMessage(msg)) {//检查msg是否为要处理的类型
	            //如果是，则将msg转化为I类型，然后调用channelRead0进程业务逻辑处理
                @SuppressWarnings("unchecked")
                I imsg = (I) msg;
                channelRead0(ctx, imsg);
            } else {//如果不是，则直接调用下一个处理器
                release = false;
                ctx.fireChannelRead(msg);
            }
        } finally {
	        //如果启动类自动回收资源，且该Handler真的消费msg，就回收msg
            if (autoRelease && release) {
                ReferenceCountUtil.release(msg);
            }
        }
    }

    /**
     * <strong>Please keep in mind that this method will be renamed to
     * {@code messageReceived(ChannelHandlerContext, I)} in 5.0.</strong>
     *
     * Is called for each message of type {@link I}.
     */
    protected abstract void channelRead0(ChannelHandlerContext ctx, I msg) throws Exception; //用户应该事先的业务逻辑
}
```

如果一个消息被消费了或是被丢弃了，没有传递给笑一个ChannelHanlder，就应该负责将该消息释放掉。

对于ChanelOutHandler，如果丢弃了消息应该通知`ChannelPromise`。
### ChannelPromise
```
public interface ChannelPromise extends ChannelFuture, Promise<Void>
```
其中`ChannelFuture`继承自`Future<Void>`，而`Promise<V> extends Future<V>`，所以`ChannelPromise`接口就是一种可写入状态的`Future`

```
@Sharable
public class DiscardOutboundHandler
    extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        ReferenceCountUtil.release(msg);
        //ChannelPromise是ChannelFuture的子类，用于设置执行结果
        //在消费该消息后，通知ChannelPromise，该消息已经被处理
        promise.setSuccess();
    }
}
```

## ChannelPipeline
#### addFirst
```
    @Override
    public final ChannelPipeline addFirst(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
	        //handler是否已经被其他PipeLine所绑定
            checkMultiplicity(handler);
            //坚持名字是否合法，是否重复
            name = filterName(name, handler);
			//生成新的ChannelHandlerContext
            newCtx = newContext(group, name, handler);
			//将Context出入队列头部
            addFirst0(newCtx);

            // If the registered is false it means that the channel was not registered on an eventloop yet.
            // In this case we add the context to the pipeline and add a task that will call
            // ChannelHandler.handlerAdded(...) once the channel is registered.
            if (!registered) {
	            //如果没有注册，这等待注册之后在执行
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                newCtx.setAddPending();
                //交给指定的进程来处理该事件；
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callHandlerAdded0(newCtx);
                    }
                });
                return this;
            }
        }
        //添加
        callHandlerAdded0(newCtx);
        return this;
    }
```

```
    private void addFirst0(AbstractChannelHandlerContext newCtx) {
        AbstractChannelHandlerContext nextCtx = head.next;
        newCtx.prev = head;
        newCtx.next = nextCtx;
        head.next = newCtx;
        nextCtx.prev = newCtx;
    }
```
ChannelHandlerContext的双向链式结构；
```
abstract class AbstractChannelHandlerContext extends DefaultAttributeMap
        implements ChannelHandlerContext, ResourceLeakHint {
    volatile AbstractChannelHandlerContext next;
    volatile AbstractChannelHandlerContext prev;
```

```
    @Override
    public EventExecutor executor() {
        if (executor == null) {
            return channel().eventLoop();
        } else {
            return executor;
        }
    }
```

```
    private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
        return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
    }
```
ctx已经在PipeLine的Context链表中，而ctx中已有handler，再将handler中设置ctx，双向引用
```
    private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
        try {
            ctx.handler().handlerAdded(ctx);
            ctx.setAddComplete();
        } 
        ......
    }
```

同理删除的时候，也要双向解除引用
```
    private void callHandlerRemoved0(final AbstractChannelHandlerContext ctx) {
        // Notify the complete removal.
        try {
            try {
                ctx.handler().handlerRemoved(ctx);
            } finally {
                ctx.setRemoved();
            }
        } catch (Throwable t) {
            fireExceptionCaught(new ChannelPipelineException(
                    ctx.handler().getClass().getName() + ".handlerRemoved() has thrown an exception.", t));
        }
    }
```

出站操作 从尾部开始
```
@Override
public final ChannelPipeline read() {
    tail.read();
    return this;
}

@Override
public final ChannelFuture write(Object msg) {
    return tail.write(msg);
}
```
入栈操作，从头部开始
```
    @Override
    public final ChannelPipeline fireChannelRead(Object msg) {
        AbstractChannelHandlerContext.invokeChannelRead(head, msg);
        return this;
    }

```

```
    static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRead(m);
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRead(m);
                }
            });
        }
    }
```

AbstractChannelHandlerContext
```
    private void invokeChannelRead(Object msg) {
        if (invokeHandler()) {
            try {
                ((ChannelInboundHandler) handler()).channelRead(this, msg);
            } catch (Throwable t) {
                notifyHandlerException(t);
            }
        } else {
            fireChannelRead(msg);
        }
    }
```

```
    @Override
    public ChannelHandlerContext fireChannelRead(final Object msg) {
        invokeChannelRead(findContextInbound(), msg);
        return this;
    }
    
    private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;
        } while (!ctx.inbound);
        return ctx;
    }

```

1. [Netty5源码分析（四） -- 事件分发模型](http://blog.csdn.net/iter_zc/article/details/39474727)
2. [Netty5源码分析（三） -- Channel如何注册OP_ACCEPT, OP_READ, OP_WRITE](http://blog.csdn.net/iter_zc/article/details/39396169)
3. [Netty 源码分析之 二 贯穿Netty 的大动脉 ── ChannelPipeline (一)](https://segmentfault.com/a/1190000007308934)
4. [Netty源码解读（三）Channel与Pipeline](http://ifeve.com/channel-pipeline/)
