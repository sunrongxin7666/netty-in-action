# Netty源码分析-1: ByteBuf
@(Java)[netty]

## 1. 为什么会有ByteBuf
buffer即**缓冲区**，在数据传输时，内存里开辟的一块临时保存数据的区域。它其实是一种**化同步为异步**的机制，可以解决数据传输的速率不对等以及不稳定的问题。

就Java来说，我们非常熟悉的Old I/O–InputStream&OutputStream系列API，基本都是在内部使用到了buffer，`outputStream.write()`只将内容写入了buffer中，必须调用`outputStream.flush()`，才能保证数据写入生效！

而NIO中则直接将buffer这个概念封装成了对象，其中最常用的大概是**ByteBuffer**了。于是使用方式变为了：将数据写入Buffer，`flip()`一下，然后将数据读出来。于是，buffer的概念更加深入人心了！

Netty中的也有缓存区buffer的概念，不同的是，Netty的buffer专为网络通讯而生，被成为**ByteBuf**。 Netty的ByteBuf缓冲区实现地比Java本身的ByteBuffer更加灵活，方便。它的类结构也比较复杂，这里只集合源码说一下最核心的要点。


## 2. Netty的数据容器类：ByteBuf
ByteBuf不是对ByteBuffer的封装，而是重新实现了一个缓冲区，作为ByteBuffer的提到。ByteBuffer只使用了一个position指针来记录当前的读写位置，ByteBuf使用了两个指针`readerIndex`, `writerIndex`分别来记录当前的读写位置，使用起来更加简单和方便。这是《Netty In Action》中的示意图：
![Alt text](./1517299339838.png)
![Alt text](./1517299326014.png)

由于ByteBuf内部保存了一个读指针readerIndex和一个写指针writerIndex，可以**同时进行读和写**，而不需要使用`flip()`进行读写切换。

和ByteBuffer一样，ByteBuf也支持**堆内缓冲区**和堆外直接缓冲区（**直接缓存区**），根据经验来说，底层IO处理线程的缓冲区使用堆外直接缓冲区，减少一次IO复制。业务消息的编解码使用堆内缓冲区，分配效率更高，而且不涉及到内核缓冲区的复制问题。

堆内缓冲区内部是以数组形式保存数据，可以调用ByteBuf的`hasArray()`来判断是否为堆内缓冲区，如果`true`则为堆内缓冲区，否者就是直接缓存区。

比如`UnpooledHeapByteBuf`类是典型的堆内缓冲区，其`hasArray()`方法直接返回为真：
```
    @Override
    public boolean hasArray() {
        return true;
    }
```
同样，作为直接缓冲去的一种，`PooledDirectByteBuf`的`hasArray()`方法直接返回为假：
```
    @Override
    public boolean hasArray() {
        return false;
    }

```

## 3. 字节级的操作
ByteBuffer是一个固定长度的缓冲区，当put方法要写的数据大于可写的容量时会抛出异常。ByteBuf改进了这个设计，支持**自动扩容**。每次写入之前会检查是否可以完全写入，如果不能，就会自动扩展ByteBuf的容量，保证写入数据的方法不会抛出异常。

下面以堆内缓冲区`UnpooledHeapByteBuf`的源码为例，解析ByteBuf的各种字节操作。

### 3.1 随机化访问数据
ByteBuf的数据索引从0开始，最后一个字节的索引为`buffer.capacity()-1`。
```
        ByteBuf buffer = BYTE_BUF_FROM_SOMEWHERE; //get reference form somewhere
        for (int i = 0; i < buffer.capacity(); i++) {
            byte b = buffer.getByte(i);
            System.out.println((char) b);
        }
```
抽象类**ByteBuf**提供`getByte(int)`方法来获得指定索引处的数据，且不改变`readerIndex`和`writerIndex`。
`UnpooledHeapByteBuf`类中`getByte`方法如下：
```
    @Override
    public byte getByte(int index) {
	    //检查该Buffer是否可用
        ensureAccessible();
        //返回数据
        return _getByte(index);
    }

    @Override
    protected byte _getByte(int index) {
	    //array为底层支持的数组；
        return HeapByteBufUtil.getByte(array, index);
    }
```
1. 首先`getByte`方法调用 `ensureAccessible()`来判断当前的缓存区是否可用，所有访问换出去数据的方法都要求先执行该方法。如果 `ensureAccessible()`方法发现该缓存区已经不可用，则直接抛出异常。
```
    /**
     * Should be called by every method that tries to access the buffers content to check
     * if the buffer was released before.
     */
    protected final void ensureAccessible() {
        if (checkAccessible && refCnt() == 0) {
            throw new IllegalReferenceCountException(0);
        }
    }
```

2. 随后调用工具类的静态方法`HeapByteBufUtil.getByte(array, index)`获得数据，其中`array`就是堆内缓冲区底层支持的数据。

>`HeapByteBufUtil`是堆内缓存区的工具类，与之对应，直接缓存区的工具类为`UnsafeByteBufUtil`。

### 3.2 读写数据

ByteBuf提供了两大类读写操作：
1. `get/set`类方法：对参数传入的索引处进行读写操作，不改变读写索引；
2. `read/write`类方法：根据读写索引指定的位置进行读写操作，并在操作后改变读写索引；

以对int型数据的操作为例分析源码。

```
    @Override
    public int getInt(int index) {
        ensureAccessible();
        return _getInt(index);
    }

    @Override
    protected int _getInt(int index) {
        return HeapByteBufUtil.getInt(array, index);
    }
```
首先调用`ensureAccessible();`保证缓存区的可用性，以后这个步骤不再复述。然后调用`HeapByteBufUtil`中封装好的`getInt`方法将4byte的Int型数据取出。
```
    static int getInt(byte[] memory, int index) {
        return  (memory[index]     & 0xff) << 24 |
                (memory[index + 1] & 0xff) << 16 |
                (memory[index + 2] & 0xff) <<  8 |
                memory[index + 3] & 0xff;
    }
```
`setInt`方法的过程与之类似
```
    @Override
    public ByteBuf setInt(int index, int   value) {
        ensureAccessible();
        _setInt(index, value);
        return this;
    }

    @Override
    protected void _setInt(int index, int value) {
        HeapByteBufUtil.setInt(array, index, value);
    }
```

```
    static void setInt(byte[] memory, int index, int value) {
        memory[index]     = (byte) (value >>> 24);
        memory[index + 1] = (byte) (value >>> 16);
        memory[index + 2] = (byte) (value >>> 8);
        memory[index + 3] = (byte) value;
    }
```
通过以上源码可知`get/set`方法是对参数传入的索引处进行读写操作，不改变读写索引。

与之相对，`read/write`方法是根据读写索引（readIndex/writeIndex）指定的位置进行读写操作，并在操作后改变读写索引。`read/write`是在父类`AbstractByteBuf`中实现的，源码如下：
```
    @Override
    public ByteBuf writeInt(int value) {
        ensureWritable0(4);
        _setInt(writerIndex, value);
        writerIndex += 4;
        return this;
    }
```
1. 确保当前缓存区的大小能够放下一个int型（4byte），如果不够大buffer就会扩大为原来的二倍;
2. 根据`writerIndex`指示的位置，写入int数据;
3. `writerIndex`增加4；

其中`ensureWritable0`方法用于确保缓存区大小，并在需要的时候扩大缓存区。
```
    final void ensureWritable0(int minWritableBytes) {
        ensureAccessible();
        //如果当前缓存区的可写入大小满足要求，直接返回
        if (minWritableBytes <= writableBytes()) {
            return;
        }
		//如果要求的写入的数据大小已经超过还能扩展的最大限制，则抛出异常；
        if (minWritableBytes > maxCapacity - writerIndex) {
            throw new IndexOutOfBoundsException(String.format(
                    "writerIndex(%d) + minWritableBytes(%d) exceeds maxCapacity(%d): %s",
                    writerIndex, minWritableBytes, maxCapacity, this));
        }
		//将缓存区大小扩大为原来的2倍；
        // Normalize the current capacity to the power of 2.
        int newCapacity = alloc().calculateNewCapacity(writerIndex + minWritableBytes, maxCapacity);
		//调整缓存区；
        // Adjust to the new capacity.
        capacity(newCapacity);
    }
```
同理，`readInt()`也是类似的过程：
1. 确保当前缓存区还有一个int型（4byte）的数据等待读取；
2. 根据`readerIndex `指示的位置，读出int数据;
3. `readerIndex `增加4；
```
    @Override
    public int readInt() {
        checkReadableBytes0(4);
        int v = _getInt(readerIndex);
        readerIndex += 4;
        return v;
    }
```
```
    private void checkReadableBytes0(int minimumReadableBytes) {
        ensureAccessible();
        if (readerIndex > writerIndex - minimumReadableBytes) {
            throw new IndexOutOfBoundsException(String.format(
                    "readerIndex(%d) + length(%d) exceeds writerIndex(%d): %s",
                    readerIndex, minimumReadableBytes, writerIndex, this));
        }
    }
```
### 3.3 索引管理
随着读写操作在缓存区上进行，缓存区的大小终究会耗尽，为了重复利用缓存区，`ByteBuf`提供了索引管理方法。
`AbstractByteBuf`类中实现了`discardReadBytes()`用以回收已经读取的空间，并将读写索引一致向缓存区头部移动。
```
    @Override
    public ByteBuf discardReadBytes() {
        ensureAccessible();
        if (readerIndex == 0) {
            return this;
        }

        if (readerIndex != writerIndex) {
            setBytes(0, this, readerIndex, writerIndex - readerIndex);
            writerIndex -= readerIndex;
            adjustMarkers(readerIndex);
            readerIndex = 0;
        } else {
            adjustMarkers(readerIndex);
            writerIndex = readerIndex = 0;
        }
        return this;
    }
```
需要注意的是，该方法由于底层使用了内存复制，频繁调用的话可能带来不必要的开销。如果只是希望将读写缓存重置，可以调用`clear() `方法。
```
    @Override
    public ByteBuf clear() {
        readerIndex = writerIndex = 0;
        return this;
    }
```

## 4. ByteBuf的分类
ByteBuf的堆内缓冲区又分为内存池缓冲区`PooledByteBuf`和普通内存缓冲区`UnpooledHeapByteBuf`。`PooledByteBuf`采用二叉树来实现一个内存池，集中管理内存的分配和释放，不用每次使用都新建一个缓冲区对象。`UnpooledHeapByteBuf`每次都会新建一个缓冲区对象。在高并发的情况下推荐使用PooledByteBuf，可以节约内存的分配。在性能能够保证的情况下，可以使用`UnpooledHeapByteBuf`，实现比较简单。

缓存区的分配，通过`ByteBufAllocator`类来完成，`AbstractByteBufAllocator`类中实现了分配Buffer的方法`buffer()` 。
```
    @Override
    public ByteBuf buffer() {
        if (directByDefault) {
            return directBuffer();
        }
        return heapBuffer();
    }

```
该方法根据要分配的缓存区类型，分别调用方法生成堆内缓存区和直接缓存区。
```
    @Override
    public ByteBuf heapBuffer(int initialCapacity, int maxCapacity) {
        if (initialCapacity == 0 && maxCapacity == 0) {
            return emptyBuf;
        }
        validate(initialCapacity, maxCapacity);
        //调用子类的具体实现
        return newHeapBuffer(initialCapacity, maxCapacity);
    }
    @Override
    public ByteBuf directBuffer(int initialCapacity, int maxCapacity) {
        if (initialCapacity == 0 && maxCapacity == 0) {
            return emptyBuf;
        }
        validate(initialCapacity, maxCapacity);
        //调用子类的具体实现
        return newDirectBuffer(initialCapacity, maxCapacity);
    }
```
再确保要分配的空间大小不高过最大限制之后，便调用抽象方法`newHeapBuffer`和`newDirectBuffer`，以子类的具体实现来完成buffer的分配。

在Netty框架中，默认是使用池化的缓冲区：在创建工作线程时，会为每个线程开辟专属的堆内缓冲池和直接缓冲池，并将其放入线程局部变量中，以做到线程封闭。当需要开辟缓冲区时，就在已有缓存池内复用ByteBuf;

以生成堆内缓冲区为例，对于内存池缓冲区的实现`PooledByteBufAllocator`类，其会先获取当前线程的线程变量`PoolThreadCache`，这是一种`ThreadLoacl`, 如果该线程的堆内缓冲次`heapArena`不为空，则堆内缓冲此内分配ByteBuffer；
```
    @Override
    protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
	    //1. 获取线程变量
        PoolThreadCache cache = threadCache.get();
        //2. 获取堆内缓冲池
        PoolArena<byte[]> heapArena = cache.heapArena;

        final ByteBuf buf;
        //3. 如果堆内缓冲池不为空
        if (heapArena != null) {
	        //4. 则直接在缓冲池内复用ByteBuf
            buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
        } else { //5. 如果堆内缓冲池为空，则创建新的ByteBuf
            buf = PlatformDependent.hasUnsafe() ?
                    new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                    new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }
```

同理反观非池化的 `UnpooledByteBufAllocator`其每次都直接分配新的ByteBuf。
```
    @Override
    protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
        return PlatformDependent.hasUnsafe() ?
                new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
    }
```

1. [Netty源码解读（二）Netty中的buffer](http://ifeve.com/netty-2-buffer/)
2. [Netty5源码分析（五） -- ByteBuf缓冲区](http://blog.csdn.net/iter_zc/article/details/39478111)
3. [源码之下无秘密 ── 做最好的 Netty 源码分析教程](https://segmentfault.com/a/1190000007282628)
