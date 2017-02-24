---
layout: post
title: Kafka消息存储之MessageWriter
category: 源码阅读
tags: kafka 
date: 2016-08-17
---
{% include JB/setup %}


* 目录
{:toc}

---

### 摘要

MessageWriter是Kafka进行消息写的工具类，这一部分代码倒是和整个系统设计没有多大关系，但是从局部来看，有许多有意思的细节，所以也开一篇短博客来讲一讲。

### MessageWriter的设计意图

首先让我们列出在消息写的过程中可能出现的变化情况，也就是这个类的设计需求：

- 输入源不同，有bytes[] stream 基本的数据类型（Int,Long,Byte，bytes）等，均需要支持。
- 写入的数据大小不确定，所以需要考虑自适应容量机制
- 需要一定的自动保障机制，比如在写入数据后自动生成CRC并填充到头部；自动计算大小并填充到头部

我们将这三个需求再切分一下层次，绝大部分的基本类型写入都可以归结为字节的写入，各种类型的写入和自适应容量的功能较为底层，可以实现的更加普适，而相对的第三个需求则相对上层，可以分开来实现。

实际上，Kafka也是这么做的，MessagheWriter继承了一个父类BufferingOutputStream，该类主要用于从各类输入源中写入数据到缓存，然后批量写入Buffer中。

### BufferingOutputStream解析

写入的消息会先暂存在BufferingOutputStream内部，他的容量控制是通过字节数组来构成链表完成的，每个字节数组均是定长的（长度由构造函数传入），同时为每个字节数组配备一个游标来标示已被写入多少字节内容，同时采用一个引用表明当前正在写的数组。下面就给出这种基本结构的定义。

```scala
protected final class Segment(size: Int) {
    val bytes = new Array[Byte](size)
    var written = 0
    var next: Segment = null
    def freeSpace: Int = bytes.length - written
  }
```
BufferingOutputStream的控制策略非常简单，那就是当前的Segment写满，立即增加新的Segment。值得注意的是，这种写入是不可逆的，就是当你回退后再写入也会创建新的segment而非重用原有的segment。

那么更加有效和复杂的控制策略是否值得被引入呢？Kafka采用这种简单的策略是由于消息的写入是一次性的，一个序列的消息使用一个writer来写入，而并非复用writer。另外就是Kafka的性能更多地受限于网络带宽，所以它应该采取尽量简单的读写策略提高读写的效率而不必费尽心机来减少内存的创建和释放（GC并不是它的主要问题）。

### 基本写入

下面我们以byte数组的写入为例，上代码：

```scala
override def write(b: Array[Byte], off: Int, len: Int) {
    if (off >= 0 && off <= b.length && len >= 0 && off + len <= b.length) {
      var remaining = len
      var offset = off
      while (remaining > 0) {
        if (currentSegment.freeSpace <= 0) addSegment()

        val amount = math.min(currentSegment.freeSpace, remaining)
        System.arraycopy(b, offset, currentSegment.bytes, currentSegment.written, amount)
        currentSegment.written += amount
        offset += amount
        remaining -= amount
      }
    } else {
      throw new IndexOutOfBoundsException()
    }
  }
```

对于基本类型的写入，MessageWriter使用了位操作，一个字节一个字节的写入，确保它们构建在字节写入的基础之上，这是一种漂亮的归纳，我们也以32位Int的写入为例。

```scala
 private def writeInt(value: Int): Unit = {
    write(value >>> 24)
    write(value >>> 16)
    write(value >>> 8)
    write(value)
  }
```

### 保障机制的实现

我们还是先贴代码再废话吧

```scala

  def write(key: Array[Byte] = null,
            codec: CompressionCodec,
            timestamp: Long,
            timestampType: TimestampType,
            magicValue: Byte)(writePayload: OutputStream => Unit): Unit = {
    withCrc32Prefix {
      // write magic value
      write(magicValue)
      // write attributes
      var attributes: Byte = 0
      if (codec.codec > 0)
        attributes = (attributes | (CompressionCodeMask & codec.codec)).toByte
      if (magicValue > MagicValue_V0)
        attributes = timestampType.updateAttributes(attributes)
      write(attributes)
      // Write timestamp
      if (magicValue > MagicValue_V0)
        writeLong(timestamp)
      // write the key
      if (key == null) {
        writeInt(-1)
      } else {
        writeInt(key.length)
        write(key, 0, key.length)
      }
      // write the payload with length prefix
      withLengthPrefix {
        writePayload(this)
      }
    }
  }

private def withLengthPrefix(writeData: => Unit): Unit = {
    // get a writer for length value
    val lengthWriter = reserve(ValueSizeLength)
    // save current size
    val oldSize = size
    // write data
    writeData
    // write length value
    writeInt(lengthWriter, size - oldSize)
  }
```

从上面这段代码可以看出scala的with非常类似于Python的decrator，将代码块当做无返回无参的函数在with中进行调用。但我想提的是另一个问题，那就是如何记录之前的位置呢。我们说过写入的过程是不可逆的，写入的游标不能回退，但是我们必须在写完数据之后再写入CRC，那么我们就需要类似于buffer的mark和reset机制一样的东东。

但是我们不能像buffer那样直接移动游标，因为我们需要顺利写入下一条消息，移动游标再复位实在代价太大。那我们能不能提前把这一小段内存截取出来赋给另一个引用，写入的时候向新引用写就行了，独立于原有的数据写入过程。MessageWriter就是这样做的，好吧让我们介绍那所谓的一小段内存吧。

```scala
protected class ReservedOutput(seg: Segment, offset: Int, length: Int) extends OutputStream {
    private[this] var cur = seg
    private[this] var off = offset
    private[this] var len = length //预留的内存大小

    override def write(value: Int) = {
      if (len <= 0) throw new IndexOutOfBoundsException()
      if (cur.bytes.length <= off) {
        cur = cur.next
        off = 0
      }
      cur.bytes(off) = value.toByte
      off += 1
      len -= 1
    }
  }

```

但是你们一定看出了上面的问题了，这样写入不是将数据覆盖了吗，所以我们在写数据时需要先预留一部分内存，这时就需要跳过一部分内存空间了。

```scala
private def skip(len: Int): Unit = {
    if (len >= 0) {
      var remaining = len
      while (remaining > 0) {
        if (currentSegment.freeSpace <= 0) addSegment()

        val amount = math.min(currentSegment.freeSpace, remaining)
        currentSegment.written += amount
        remaining -= amount
      }
    } else {
      throw new IndexOutOfBoundsException()
    }
  }
```

预留操作在此处

```scala
def reserve(len: Int): ReservedOutput = {
    val out = new ReservedOutput(currentSegment, currentSegment.written, len)
    skip(len)
    out
  }
```

我们再来看withLengthPrefix，先预留ValueSize大小的内存，然后写入数据，最后计算整个的内存变化也就是写入的数据的大小，写入预留内存中，是不是很完美？
