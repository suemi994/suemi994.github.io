---
layout: post
title: Kafka消息存储之FileMessageSet
category: 源码阅读
tags: kafka 
date: 2016-08-18
---
{% include JB/setup %}


* 目录
{:toc}

---

### 摘要

看过前面几篇博客的盆友可能会问，逼逼了这么多还不知道消息到底存到哪儿了，分明标题党嘛。这一次我们就来看与存储切实相关的底层操作类FileMessageSet。它是MessageSet的一个子类，操作消息和文件之间的读写操作。想想我们也知道，这特么就是要写增删改查啊。这一次的代码确实没啥好说的，但是FileMessageSet确是比较重要的一个类，还是简短讲一下吧。

### FileMessageSet的功能
- 消息的增删改查
- 进行必要的检查，比如是否是指定的消息格式（检查Magic值）
- 进行消息格式的转换

对于最核心的功能——增删改查，我们在这里进一步展开。首先FileMessageSet只处理最外层的消息，而不考虑嵌套的消息，嵌套消息会移交给之前的ByteBufferMessageSet处理。某种程度上，我们也可以把ByteBufferMessageSet看做是嵌套消息。

FileMessageSet的删除也分为两种，一种是从特定位置截断，一种是直接删除整个文件。其查询主要是从消息的序号也就是offset获得其在文件中的位置。其增加只允许向尾部追加，若想在中间添加，必须先截断。

我们列一下几个重要的原子操作吧

- read(buffer,position,length),read(position,length):FileMessageSet
- writeTo(channel,position,size)
- truncate(size)
- search(offset):position
- close
- flush


### FileMesssage的设计

FileMessageSet使用FileChannel来进行读写，我们的操作依赖于position进行，需要首先定位。同样，FileMessageSet允许支持切片，也就是截取文件中的一部分，指定start和end。但是这样每次检查末尾都需要考虑end了。

这里首先要注意的第一点是channel的游标应该始终定位在set的尾部，这是为了保证写入是顺序的，所以在初始化的时候就应该将游标移到尾部。

第二点是在关闭channel的时候需要先做flush然后截断。这一点可能不太好理解，这里举个例子，如果我使用了分片，并在位置end后写入了一条新消息，由于必须保证消息是有序的，所以后面所有的消息必须丢弃。这也是保证消息的顺序写特性。

```scala
 def close() {
    flush()
    trim()
    channel.close()
  }
```

第三点是迭代的过程，这里面几乎所有的原子操作均是从遍历实现的，遍历中需要进行较多的检查操作，主要是以下几点。

- 如果当前读取的messageSize小于最小的消息头大小，说明消息出现错误
- 如果当前读取的messageSize大于剩余的容量，说明最后一条消息不完整
- 如果剩下的容量小于offsetSize+MessageSizeLength，说明已经没有消息了

但是这里的容量需要同时考虑指定的end和channel的结尾，下面以生成迭代器为例。

```scala
override def makeNext(): MessageAndOffset = {
        //最后一条消息出现在end之后
        if(location + sizeOffsetLength >= end)
          return allDone()


        // read the size of the item
        sizeOffsetBuffer.rewind()
        channel.read(sizeOffsetBuffer, location)

        //最后一条消息出现在下一文件中
        if(sizeOffsetBuffer.hasRemaining)
          return allDone()

        sizeOffsetBuffer.rewind()
        val offset = sizeOffsetBuffer.getLong()
        val size = sizeOffsetBuffer.getInt()

        //最后一条消息被end截断或消息大小出现问题
        if(size < Message.MinMessageOverhead || location + sizeOffsetLength + size > end)
          return allDone()
      //消息过大
        if(size > maxMessageSize)
          throw new CorruptRecordException("Message size exceeds the largest allowable message size (%d).".format(maxMessageSize))

        // read the item itself
        val buffer = ByteBuffer.allocate(size)
        channel.read(buffer, location + sizeOffsetLength)

        //最后一条消息被文件截断
        if(buffer.hasRemaining)
          return allDone()
        buffer.rewind()

        // increment the location and return the item
        location += size + sizeOffsetLength
        new MessageAndOffset(new Message(buffer), offset)
      }
```

第四条是追加是以ByteBufferMessageSet为单位的，这主要是将嵌套消息和一般消息还有批量写入统一在一个方法下。


第五条是一个有趣的代码细节

```scala
def delete(): Boolean = {
    CoreUtils.swallow(channel.close())
    file.delete()
  }

def swallow(log: (Object, Throwable) => Unit, action: => Unit) {
    try {
      action
    } catch {
      case e: Throwable => log(e.getMessage(), e)
    }
  }
```
这里将代码块包裹在try catch中，通过这种方法调用的形式，非常简洁优美，有点类似于使用AOP收集异常，值得借鉴。

### 消息读入的过程

写到这儿，让我们来回顾一下整个消息存储的内容并整理出完整的流程吧。

1. 首先FileMessageSet读取最外层消息
2. 若该消息是嵌套消息，则生成ByteBufferMessageSet解压缩并生成原子消息集
3. 通过调用message自身的方法进行检验和获取基本信息比如消息格式
4. 通过MessageAndMeta加上译码器获得key-value对象


### 消息写入的过程

1. 首先MessageWriter写入key-value和消息头生成buffer
2. 对于嵌套消息使用刚刚的buffer生成 ByteBufferMessageSet并convert压缩成新的ByteBufferMessageSet
3. 再使用FileMessageSet追加ByteBUfferMessageSet
