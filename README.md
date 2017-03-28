![](docs/header.jpg)

Java has a lot of different ways to represent a stream of bytes.  Depending on the author and age of a library, it might use `byte[]`, `InputStream`, `ByteBuffer`, or `ReadableByteChannel`.  If the bytes represent strings, there's also `String`, `Reader`, and `CharSequence` to worry about.  Remembering how to convert between all of them is a thankless task, made that much worse by libraries which define their own custom representations, or composing them with Clojure's lazy sequences and stream representations.

这个库是一个工具集合，可以让你忘记那些乱七八糟的api
完整的文档地址[这里](http://aleph.io/codox/byte-streams/)

### 使用

```clj
[byte-streams "0.2.2"]
```

### 类型转换

使用`byte-streams/convert`，从一种类型转换成另外一种

```clj
byte-streams> (convert "abcd" java.nio.ByteBuffer)
#<HeapByteBuffer java.nio.HeapByteBuffer[pos=0 lim=4 cap=4]>
byte-streams> (convert *1 String)
"abcd"
```

调用`(convert data to-type options?)`来转换，如果可能，从当前类型转换为目的类型。
目的类型可以是Java类或者是一个Clojure协议。
因为没有直接的方法来从String转化成byte-buffer，可以使用`convert`来实现：

```clj
byte-streams> (conversion-path String java.nio.ByteBuffer)
([java.lang.String [B]
 [[B java.nio.ByteBuffer])
```

我们不是直接把string转换成`ByteBuffer`，我们可先转换成`byte[]`然后从`byte[]`到`ByteBuffer`
调用方法会选择最小的路径来转换的。
通常的方法有`to-byte-buffer`,`to-byte-buffers`,`to-byte-array`,`to-input-stream`,
`to-readable-channel`,`to-char-sequence`,`to-string`和`to-line-seq`

任何类型都可以存在，甚至是一个序列。例如我们可以创建一个`InputStream`的无限序列

```clj
byte-stream> (to-input-stream (repeat "hello"))
#<InputStream byte_streams.InputStream@3962a02c>
```

然后我们转化成一个惰性的`ByteBuffers`：

```clj
byte-streams> (take 2
                (convert *1
                  (seq-of java.nio.ByteBuffer)
                  {:chunk-size 128}))
(#<HeapByteBuffer java.nio.HeapByteBuffer[pos=0 lim=128 cap=128]>
 #<HeapByteBuffer java.nio.HeapByteBuffer[pos=0 lim=128 cap=128]>)
```

注意我们使用`(seq-of type)`来描述序列，并且使用一个map来描述可选项：

* `:chunk-size` - 每次转化的块大小, 默认`4096`
* `:direct?` - `ByteBuffers`可以直接转化，具体看[direct](http://stackoverflow.com/a/5671880/228387),默认是`false`
* `:encoding` - 编码方法, 默认是`"UTF-8"`

创建一个[Manifold stream](https://github.com/ztellman/manifold), 使用`(stream-of type)`。
转化core.async channel, 使用 `manifold.stream/->source`。

### 定制转化

以上是一些通用的字节类型转化，通过`byte-streams/def-conversion`来定制扩展。

```clj
;; 一个从byte-buffers到my-byte-representation的转化
(def-conversion [ByteBuffer MyByteRepresentation]
  [buf options]
  (buffer->my-representation buf options))

;; everything that can be converted to a ByteBuffer is transitively fair game now
(convert "abc" MyByteRepresentation)
```

这个可以定制任何转化机制，只要觉得可行。

### 传输

简答的转化非常好用，但是有时我们需要保存更多的字节到内存中。
当你需要写一些字节到文件中时，socket，或者其他终端，你可以使用`byte-streams/transfer`

```clj
byte-streams> (def f (File. "/tmp/salutations"))
#'byte-streams/f
byte-streams> (transfer "hello" f {:append? false})
nil
byte-streams> (to-string f)
"hello"
```

`(transfer source sink options?)` 允许你管道化任何东西。也可以定制：

```clj
(def-transfer [InputStream MyByteSink]
  [stream sink options]
  (send-stream-to-my-sink stream sink))
```

### 工具函数

`byte-streams/print-bytes` 会打印16禁止和ascii码

```clj
byte-streams> (print-bytes (-> #'print-bytes meta :doc))
50 72 69 6E 74 73 20 6F  75 74 20 74 68 65 20 62      Prints out the b
79 74 65 73 20 69 6E 20  62 6F 74 68 20 68 65 78      ytes in both hex
20 61 6E 64 20 41 53 43  49 49 20 72 65 70 72 65       and ASCII repre
73 65 6E 74 61 74 69 6F  6E 73 2C 20 31 36 20 62      sentations, 16 b
79 74 65 73 20 70 65 72  20 6C 69 6E 65 2E            ytes per line.
```

`(byte-streams/compare-bytes a b)` 比较两个字节是否相同

```clj
byte-streams> (compare-bytes "abc" "abd")
-1
```

`bytes-streams/bytes=` 比较字节是否相等

`byte-streams/conversion-path` 返回转化路径:

```clj
;; each element is a conversion tuple of to/from
byte-streams> (conversion-path java.io.File String)
([java.io.File java.nio.channels.ReadableByteChannel]
 [#'byte-streams/ByteSource java.lang.CharSequence]
 [java.lang.CharSequence java.lang.String])

;; but if a conversion is impossible...
byte-streams> (conversion-path java.io.OutputStream java.io.InputStream)
nil
```

`byte-streams/possible-conversions` 类型可以转化的目的类型列表.

```clj
byte-streams> (possible-conversions String)
(java.lang.String java.io.InputStream java.nio.DirectByteBuffer java.nio.ByteBuffer (seq-of java.nio.ByteBuffer) java.io.Reader java.nio.channels.ReadableByteChannel [B java.lang.CharSequence)
```

`byte-streams/optimized-transfer?` 返回两个类型是否有转化的方式:

```clj
byte-streams> (optimized-transfer? String java.io.File)
true
```

### license

Copyright © 2014 Zachary Tellman

Distributed under the [MIT License](http://opensource.org/licenses/MIT)
