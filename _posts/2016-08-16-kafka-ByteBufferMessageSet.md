
---
layout: post
title: Kafka消息存储之ByteBufferMessageSet
category: 源码阅读
tags: kafka 
date: 2016-08-16
---
{% include JB/setup %}


* 目录
{:toc}

---

### 摘要

MessageSet是Kafka在底层操作message非常重要的一个层级概念，从名称上可以看出来它是消息的集合体，但是代码中的处理逻辑更多的是在考虑到嵌套消息的处理问题。MessageSet的主要功能是提供Message的顺序读和批量写，操作的基本对象是MessageAndOffset。

### MessageSet

首先我们要讲一讲message在MessageSet中是如何分布的，每一条消息的前端都会被加入一个Long来表示消息在set中的位置偏移，紧接着是一个Int来指明消息的大小。MessageSet正是通过读取这些来分割消息的。

同时MessageSet实现了Iterable接口，它最主要的两个方法即是返回迭代器和写入Channel，下面贴代码。

```java
/** Write the messages in this set to the given channel starting at the given offset byte. 
    * Less than the complete amount may be written, but no more than maxSize can be. The number
    * of bytes written is returned */
  def writeTo(channel: GatheringByteChannel, offset: Long, maxSize: Int): Int
  
  /**
   * Provides an iterator over the message/offset pairs in this set
   */
  def iterator: Iterator[MessageAndOffset]
  
  /**
   * Gives the total size of this message set in bytes
   */
  def sizeInBytes: Int
```

在这里我主要想提几个问题：

- 为什么是实现iterable而不是直接实现iterator接口？
- MessageSet理应可以将更多的API集成到该虚类中，比如在特定Offset插入消息和消息集，删除特定Offset的消息，该不该引入这些API呢？如果需要，应该引入到哪些程度？

首先看看第一个问题，我们不难Collection接口也是继承Iterable的，因为Iterator接口的核心方法next()或者hasNext() 是依赖于迭代器的当前迭代位置的。 如果Collection直接实现Iterator接口，势必导致集合对象中包含当前迭代位置的数据(指针)。  当集合在不同方法间被传递时，由于当前迭代位置不可预置，那么next()方法的结果会变成不可预知。

对于第二个问题，目前我所想到的是由于有嵌套消息的存在，无论是插入或者删除都需要经过复杂的检查操作，而对于消息队列来说，消息的插入和消费必定是顺序的，且发生在头和尾，将这样的API设定加入到父虚类未免代价太高。不过这只是我个人的一点想法，不一定正确。

### iterator的生成

接下来我们看看ByteBufferMessageSet是如何实现顺序读的，我们在这里直接考虑最复杂的情形，即需要解析嵌套消息解压缩的情形。

```
1. 从buffer的position头部读取Long，得到整个set的初始偏移
2.读取一个Int，得到第一个message的大小；进行检查，若该大小小于规定的消息头大小则出现错误，若buffer剩余的大小小于该大小，则说明最后一条消息被截断，解析完毕
3. 根据得到的消息大小生成MessageAndOffset，若它是非压缩的则直接作为下一条消息；否则进入4.
4. 生成一个内迭代器负责嵌套消息中的消息迭代。这里我们需要注意几点，一是要根据外层消息的时间戳和时间戳类型来修改内层消息的这两个属性；二是要注意偏移的转换。偏移的转换我会放在下面重点讲一讲。
5. 内迭代器首先从压缩字节码流里解压缩读取所有的内层消息，并在外层请求next()时一一迭代，直到结束重置内迭代器，并返回外层迭代逻辑
```

偏移定位问题是我们在这里需要重点提到的问题，首先这个偏移到底指什么：从设计上来讲，有两种选择：1.类似于消息的序号，决定了消息是第几条。2.消息在字节码流中的起始位置。为了更好的服务于上层读取之后的处理逻辑，Kafka选择了消息序号。但是这里的主要问题是对于嵌套消息，它解压缩之后，这些内部消息存储的是相对位移（相对于外层的序号），需要修改它们的相对位移到绝对位移。另外嵌套消息的位移又该是什么呢，若是仅仅考虑外层，那么它解压缩之后所有后面的消息位移都需要增加，所以不合理，一种可取的方式是选择其内部消息的最后一条的绝对位移。

```java

/**
RO = IO_of_a_message - IO_of_the_last_message
AO = AO_Of_Last_Inner_Message + RO


**/

override def makeNext(): MessageAndOffset = {
        messageAndOffsets.pollFirst() match {
          case null => allDone()
          case nextMessage@ MessageAndOffset(message, offset) =>
            if (wrapperMessage.magic > MagicValue_V0) {
              val relativeOffset = offset - lastInnerOffset
              val absoluteOffset = wrapperMessageOffset + relativeOffset
               /** 这里做的非常巧妙，这个lastInnerOffset其实就是整个外层消息等价的内部相对位移**/
              new MessageAndOffset(message, absoluteOffset)
            } else {
              nextMessage
            }
        }
      }
```

### 构造函数和写函数

MessageSet的写并不困难，首先写入偏移，再写入大小，最后写入本体。对于压缩消息，则再给它加个头部。但是构造函数的设计其实体现了这个类的功能，ByteBufferMessageSet到底会用在什么地方呢？

- 直接从Buffer中获取原始数据，把它当做messageSet进行解析，也就是读操作
- 从一系列消息中组装（可选整体的时间戳和magic值），最终写入channel中

ByteBufferMessageSet最特别的用处是用来读写嵌套消息的，它为嵌套消息设定相对偏移，检查所有内部消息的magic值，在读取的时候转换内部消息的时间戳，对内部消息集做压缩并加上嵌套消息头。

看到这里，构造函数的设计呼之欲出了。

- 直接从Buffer或bytes中构造，主要完成读操作
- 从一系列消息中构造，可以指定它们是否组成嵌套消息，也就是指定整个set的压缩方式，是否压缩；更重要的是指定偏移
- 若是需要压缩，就指定外部嵌套消息的消息头相关属性，这一部分就可以参考Message的构造函数了

下面贴一下后者的构造函数代码

```java
private def create(offsetAssigner: OffsetAssigner,
                     compressionCodec: CompressionCodec,
                     wrapperMessageTimestamp: Option[Long],
                     timestampType: TimestampType,
                     messages: Message*): ByteBuffer = {
    if (messages.isEmpty)
      MessageSet.Empty.buffer
    else if (compressionCodec == NoCompressionCodec) {
// 不为嵌套消息时，只是集合一系列消息，仅仅通过offsetAssigner指定预计偏移
      val buffer = ByteBuffer.allocate(MessageSet.messageSetSize(messages))
      for (message <- messages) writeMessage(buffer, message, offsetAssigner.nextAbsoluteOffset())
      buffer.rewind()
      buffer
    } else {
      val magicAndTimestamp = wrapperMessageTimestamp match {
        case Some(ts) => MagicAndTimestamp(messages.head.magic, ts)
        case None => MessageSet.magicAndLargestTimestamp(messages)
      }
      var offset = -1L
      val messageWriter = new MessageWriter(math.min(math.max(MessageSet.messageSetSize(messages) / 2, 1024), 1 << 16))

// 写嵌套消息头
      messageWriter.write(codec = compressionCodec, timestamp = magicAndTimestamp.timestamp, timestampType = timestampType, magicValue = magicAndTimestamp.magic) { outputStream =>
        val output = new DataOutputStream(CompressionFactory(compressionCodec, magicAndTimestamp.magic, outputStream))
        try {
          for (message <- messages) {
            offset = offsetAssigner.nextAbsoluteOffset()
            if (message.magic != magicAndTimestamp.magic)
              throw new IllegalArgumentException("Messages in the message set must have same magic value")
            // Use inner offset if magic value is greater than 0
            if (magicAndTimestamp.magic > Message.MagicValue_V0)

            //注意这里，内部消息写入的是相对位移
              output.writeLong(offsetAssigner.toInnerOffset(offset))
            else
              output.writeLong(offset)
            output.writeInt(message.size)
            output.write(message.buffer.array, message.buffer.arrayOffset, message.buffer.limit)
          }
        } finally {
          output.close()
        }
      }
      val buffer = ByteBuffer.allocate(messageWriter.size + MessageSet.LogOverhead)

//注意这里，写嵌套消息大小和位移，嵌套消息位移是它最后一条内部消息的绝对位移
      writeMessage(buffer, messageWriter, offset)
      buffer.rewind()
      buffer
    }
  }

可以说上面这段代码几乎就包括了整个ByteBufferMessageSet的设计目的，和读写方式，这一段大有深意啊。知道看懂这一段我的诸多疑惑才得到解答。

```

### 验证与校正

这里主要完成下面几个任务：

- 检查时间戳与时间戳类型
- 对于嵌套内层消息，需要检查它是否有key
- 可以重新设定时间戳类型和时间戳并修改
- 需要做偏移校正，可以为整个Set设定整体的一个起始偏移，重新检查所有消息的位移是否合理

Kafka 0.10.0的代码中将这些功能混杂在一次遍历中，对于含压缩的MessageSet许多操作比如offset校正等不可行只能返回一个新的解压缩之后的MessageSet。我个人认为这可能并不是好的做法，应该巴这些功能区分独立出来，必要的校检做一块，校正做一块，最后重新设定做一块。下面以kafka 0.8.0的代码为例展示一下如何做偏移校正。

```java
 /**
   * Update the offsets for this message set. This method attempts to do an in-place conversion
   * if there is no compression, but otherwise recopies the messages
   */
private[kafka] def assignOffsets(offsetCounter: AtomicLong, codec: CompressionCodec):            ByteBufferMessageSet = {
    if(codec == NoCompressionCodec) {
      // do an in-place conversion
      var position = 0
      buffer.mark()
      while(position < sizeInBytes - MessageSet.LogOverhead) {
        buffer.position(position)
        buffer.putLong(offsetCounter.getAndIncrement())
        position += MessageSet.LogOverhead + buffer.getInt()
      }
      buffer.reset()
      this
    } else {
      // messages are compressed, crack open the messageset and recompress with correct offset
      val messages = this.internalIterator(isShallow = false).map(_.message)
      new ByteBufferMessageSet(compressionCodec = codec, offsetCounter = offsetCounter, messages = messages.toBuffer:_*)
    }
  }
```
