---
layout: post
title: Kafka消息存储概览
category: 源码阅读
tags: kafka 
date: 2016-08-15
---
{% include JB/setup %}


* 目录
{:toc}

---

### 摘要

Kafka作为一个消息中间件系统，面临的首要问题就是消息如何持久化，如何方便地进行读写和解析。本文将就Kafka的消息存储问题开一个头，后续将会对重要的代码部分一一讲解。Kafka的消息概念，首先我们在此谈论的不是网络传递中的消息，而更多偏向于记录的意思，也就是消费者和生产者所在意的实际对象。消息是Kafka造作的最小单元，并不允许更改消息的实际内容，一条消息本质上是一个键值可缺省的键值对。

### 消息格式
下面首先以Kafka 0.10.0版本为例来解释一下其消息格式：
 CRC+magic+attributes+wrapperTimestamp(optional)+key(长度+内容)+payload(长度+内容)

 下面依次列出每一部分
 
```
 * 1. 4 byte CRC32 校检值
 * 2. 1 byte "magic" 标识符来显示消息格式是否发生了改动,值为0/1(也可以看作是版本号)
 * 3. 1 byte "attributes" 标识符,包含以下内容:
 *    bit 0 ~ 2 : 压缩编码方式
 *      0 : no compression
 *      1 : gzip
 *      2 : snappy
 *      3 : lz4
 *    bit 3 : 时间戳类型
 *      0 : create time
 *      1 : log append time
 *    bit 4 ~ 7 : 保留部分
 * 4. (可选) 8 byte 时间戳,只有magic为1时才携带该部分
 * 5. 4 byte key length, 指定Key部分的长度
 * 6. K byte key
 * 7. 4 byte payload length, 指定值的长度
 * 8. V byte payload
```

Kafka的消息格式在设计上允许多重嵌套，这种嵌套是通过压缩实现的。试想一下，某个消息的key为空，然后它的value部分是一个压缩后的MessageSet，那么经过解压缩并读取后它就是一个键值对集合了，这有些类似于json了。但实际上Kafka的消息只允许二重嵌套，这并非由其消息格式的局限性决定，而是考虑到读取消息时解析的复杂度决定。嵌套消息使得Kafka传递复杂类型的对象成为可能，但是出于性能因素的考虑，对象序列化和过于复杂的数据格式并不适合消息系统这一业务，或者说kafka在性能方面和表达能力上做了一个漂亮的妥协。

同时，我们还要重点来谈一谈时间戳相关，时间戳有三种取值，分别是-1代表不带时间戳，0代表该消息创建的时间，1代表它持久化时间（也可以理解为入库被kafka处理的时间）。对于嵌套的消息来说，若我们选择时间戳类型为入库时间，则被压缩消息的时间戳和其外层消息一致；若我们选择时间戳类型为创建时间，则应该从字节码流中读取；若magic值为0.则不管如何，都应该认为时间戳为-1，时间戳类型为CREATE_TIME。

### Message类的代码设计

虽然看起来消息格式好像比较简单，但实际上代码却相对有些复杂，最重要的问题是1、要兼容magic为0的情况，2、要能为后续的版本升级留出扩展。我们首先思考一下，Message类应该具有哪些功能呢大致分为下面几个部分：

#### 预定义的变量

由于操作的是bytes，大量的取值需要位操作，所以我们应该预定义好一些位操作辅助变量和一些重要的偏移位置。这里面有几个比较重要的预定义变量需要着重强调一下：

- key length byte的位置，因为它是头和体的分割点
- 从attributs byte读取时间戳类型和压缩编码方式的辅助变量

下面上代码

```java
/**
   * Specifies the mask for the compression code. 3 bits to hold the compression codec.
   * 0 is reserved to indicate no compression
   */
  val CompressionCodeMask: Int = 0x07
  /**
   * Specifies the mask for timestamp type. 1 bit at the 4th least significant bit.
   * 0 for CreateTime, 1 for LogAppendTime
   */
  val TimestampTypeMask: Byte = 0x08
  val TimestampTypeAttributeBitOffset: Int = 3


    public byte updateAttributes(byte attributes) {
        return this == CREATE_TIME ?
            (byte) (attributes & ~Record.TIMESTAMP_TYPE_MASK) : (byte) (attributes | Record.TIMESTAMP_TYPE_MASK);
    }

    public static TimestampType forAttributes(byte attributes) {
        int timestampType = (attributes & Record.TIMESTAMP_TYPE_MASK) >> Record.TIMESTAMP_TYPE_ATTRIBUTE_OFFSET;
        return timestampType == 0 ? CREATE_TIME : LOG_APPEND_TIME;
    }


 def compressionCodec: CompressionCodec = 
    CompressionCodec.getCompressionCodec(buffer.get(AttributesOffset) & CompressionCodeMask)
```

#### 各个属性的get方法

这个不用多说，需要注意的就是magic的值不同，有些取值的偏移位置不一样，所以需要事先写好静态方法快捷获得不同magic下的位置偏移。

#### 合法性检查

主要检查以下几个方面：

- magic值和时间戳类型和时间戳值的组合是否一致
- CRC 校检是否通过

#### 不同magic值下的message相互转换


```
1. 计算新message需要的空间大小并分配
2. 写入新的magic值
3. 取原来的attribute并用设置的TimestampType更新它，然后写入attribute值
4. 若是0->1则写入时间戳　
５.写入原来的消息体
６.计算新的ＣＲＣ值并填充
```

#### 一系列的构造方法

主要的构造途径有两条：

- 主要用于构造嵌套消息，直接传入buffer数据，可以设定时间戳和时间戳类型
- 主要用于构造原子消息，传入键值对数据，并设定主要参数（magic、压缩编码、时间戳类型、时间戳）

### 消息相关的主要类
![Message相关类](https://static.oschina.net/uploads/img/201608/15201836_hfex.png "Message相关类")

下面就依次介绍每个类的作用

- MessageAndOffset：在Message之外附加它处于set中的偏移
- MessageAndMeta：主要功能是包装解码器将message解码为Key和Value对象
- MessageSet：管理message集合，事先集合的顺序读和批量写，但个人认为其代码的重心在于如何解决消息嵌套的解析
- ByteBufferMessageSet：用ByteBuffer来存储序列的Message，主要为了方便读Message的操作
- ByteBufferBackedInputStream：将buffer的读写包装成流的模式
