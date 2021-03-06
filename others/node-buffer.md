## 关于Node中Buffer的理解

> 搬运自：https://blog.csdn.net/qq_34629352/article/details/88037778
>
> 个人感觉这是一篇比较通俗的理解，比较node的初学者

你是不是和我一样，对Node.js中的**Buffer, Stream,** 和 **二进制数据**一直都是很模糊的印象? 或者有的时候觉得，哎，我会用就行了，这些原理、底层的东西，应该交给Node.js的工程师们去理解。

的确，这些名词可能会比较初学者感到恐惧和陌生，特别是那些刚从前端转全栈，做Node.js，却没有计算机基础的同学来说。

但是很遗憾，很多教程或者书籍都会直接跳过这些原理和解释的部分，直接教你怎么使用Node.js的一些库、工具或者API，但是对于核心的部分、为什么这样处理和使用，却只字未提。甚至有些直接告诉你：“你根本不需要理解这些，因为你在工作中可能**永远不会**直接使用它”

是的，如果你想一辈子做一个平庸的程序员，的确可以在工作中不直接使用。

然而，如果那些迷惑和模糊的概念，能引起你的好奇，并不断保持这种好奇心去学习和探索，那么你对Node.js的理解就会更上一层楼，然后你就会更愿意去学习和了解Node.js一些核心的、原理性的东西，比如**Buffer, Stream**。 这也就是我写这篇文章的原因–去帮助你更好的、更深入的去理解Node.js。

当说到**Buffer**，官方是这么说的：

> …JavaScript 语言没有读取或操作二进制数据流的机制。 `Buffer` 类被引入作为 Node.js API 的一部分，使其可以在 TCP 流或文件系统操作等场景中处理二进制数据流。

嗯..尴尬，除非你已经有一些计算机基础，否则上面这句话说了只能让你脑袋更大。我们尝试简化一下，把主要含义提炼一下，可以这么说：

`Buffer`类被引入到Node.js的API中，让其与二进制数据流的操作和交互成为可能

这样是不是简单的多了？ 但是…Buffer，streams和二进制数据又是什么东西呢？我们从后向前，一个一个解释下。

## 二进制数据是什么鬼？

你应该已经知道，计算机存储和表示数据使用二进制的。比如，下面这些是5个二进制数，5个不同的1和0序列：

```
10`, `01`, `001`, `1110`, `00101011
```

二进制中的每个数字，0或1叫做位(bit)，也就是**Binary digIT**的缩写。

为了能够存储和表示这些数据，计算机需要将数据转换为二进制形式。比如，要存储数字12，计算机需要将12转化为二进制`1100`

计算机怎么知道要如何去转换？这就完全是一个数学问题了。计算机是知道怎么去处理的，有兴趣的可以自己查阅。

但是，我们日常工作的数据类型不仅仅是数字。我们还有字符、图片甚至视频。计算机是知道如何将这些表示为二进制的。就拿字符来说，比如计算机如何用二进制来表示”L“这个字母。为了将数据存储为二进制形式，无论任何类型的数据都会先被转换为数字，然后将数字转为二进制形式。所以为了表示”L“，计算机首先将**L**转换为数字表示，我们看下怎么做到这一点。

打开你的浏览器控制台，然后粘贴下面的代码：`"L".charCodeAt(0)`。你看到了什么？数字76？这就是字母**L**的数字编码。但是计算机怎么知道具体哪个数字代表那个字母呢？

**字符集**

字符集就是定义数字所代表的字符的一个规则表，同样定义了怎样用二进制存储和表示。那么，用**多少位**来表示一个数字，这个就叫**字符编码(Character Encoding)**

有一种字符编码叫做**UTF-8**。它规定了，字符应该以字节为单位来表示。一个字节是8位(bit)。所以8个1和0组成的序列，应该用二进制来存储和表示任意一个字符。

为了更好的理解，举个例子： 比如之前提到的12的二进制表示是`1100`。 所以，使用UTF-8的格式来表示，应该使用一个字节，也就是8位来完整表示，也即`00001100`， 没有错吧？

因此，76在计算机中的存储形式应该是`01001100`。

这就是计算机将字符存储成二进制的方式。当然，计算机也有一些特殊规则，将图片、视频等存储为二进制的，总之，计算机会将无论图片、视频或其他数据都转换为二进制并存储，这就是我们说的**二进制数据**。

如果你对字符编码非常感兴趣，那你可以[参考一下这篇文章](https://www.w3.org/International/questions/qa-what-is-encoding)

现在我们了解了什么是二进制数据，但是我们介绍buffer的时候，说的**二进制数据流(streams of binary data)**又是什么呢？

### Stream

在Node.js中，流(stream)就是一系列从A点到B点移动的数据。完整点的说，就是当你有一个很大的数据需要传输、搬运时，你不需要等待所有数据都传输完成才开始下一步工作。

实际上，巨型数据会被分割成小块(chunks)进行传输。所以，buffer的原始定义中所说的(“streams of binary data… in the context of… file system”)意思就是说二进制数据在文件系统中的传输。比如，将file1.txt的文字存储到file2.txt中。

但是，buffer到底在流(stream)中，是如何操作二进制数据的？buffer到底是个什么呢？

### Buffer

我们已经知道数据流(stream of data)是从一个地方向另一个地方传输数据的过程，但是这个具体是怎么样的一个过程？

通常情况下，我们传输数据往往是为了处理它，或者读它，或者基于这些数据做处理等。但是，在每次传输过程中，有一个数据量的问题。因此当数据到达的时间比数据理出的时间快的时候，这个时候我们处理数据就需要等待了。

领域覅那个面，如果处理数据的时间比到达的时间快，这一时刻仅仅到达了一小部分数据，那这小部分数据需要等待剩下的数据填满，然后再送过去统一处理。

这个”等待区域”就是buffer! 它是你电脑上的一个很小的物理地址，一般在RAM中，在这里数据暂时的存储、等待，最后在流(stream)中，发送过去并处理。

我们可以把整个流(stream)和buffer的配合过程看作公交站。在一些公交站，公车在没有装满乘客前是不会发车的，或者在特定的时刻才会发车。当然，乘客也可能在不同的时间，人流量大小也会有所不同，有人多的时候，有人少的时候，乘客或公交站都无法控制人流量。

不论何时，早到的乘客都必须等待，直到公车接到指令可以发车。当乘客到站，发现公车已经装满，或者已经开走，他就必须等待下一班车次。

总之，这里总会有一个等待的地方，这个等待的区域就是Node.js中的**Buffer** Node.js不能控制数据什么时候传输过来，传输速度，就好像公交车站无法控制人流量一样。他只能决定什么时候发送数据。如果时间还不到，那么Node.js就会把数据放入buffer–”等待区域”中，一个在RAM中的地址，直到把他们发送出去进行处理。

一个关于buffer很典型的例子，就是你在线看视频的时候。如果你的网络足够快，数据流(stream)就可以足够快，可以让buffer迅速填满然后发送和处理，然后处理另一个，再发送，再另一个，再发送，然后整个stream完成。

但是当你网络连接很慢，当处理完当前的数据后，你的播放器就会暂停，或出现”缓冲”(buffer)字样，意思是正在收集更多的数据，或者等待更多的数据到来，才能下一步处理。当buffer装满并处理好，播放器就会显示数据，也就是播放视频了。在播放当前内容的时候，更多的数据也会源源不断的传输、到达和在buffer等待。

如果播放器已经处理完或播放完前一个数据，buffer仍然没有填满，”buffering”(缓冲)字符就会再次出现，等待和收集更多的数据。

这就是**Buffer！**

从原始的定义，我们知道，buffer可以在stream中与二进制数据进行交互和操作。那么到底可以进行什么样的操作呢？在Node.js中又应该如何进行刚才所描述的一些东西呢？我们来瞧一瞧。

与Buffer共舞

你甚至可以做你自己的buffer！ 在stream中，Node.js会自动帮你创建buffer之外，你可以创建自己的buffer并操作它，是不是很有趣？ 我们来搞一个！

根据你的需求，这里有不同的创建方式，我们一起看一下吧：

```
// 创建一个大小为10的空buffer
// 这个buffer只能承载10个字节的内容

const buf1 = Buffer.alloc(10);

// 根据内容直接创建buffer

const buf2 = Buffer.from("hello buffer");
```

创建了之后，你就可以操作buffer了

```
// 检查下buffer的结构

buf1.toJSON()
// { type: 'Buffer', data: [ 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 ] }
// 一个空的buffer

buf2.toJSON()
// { type: 'Buffer',data: [ 104, 101, 108, 108, 111, 32, 98, 117, 102, 102, 101, 114 ] }
// the toJSON() 方法可以将数据进行Unicode编码并展示
   
// 检查buffer的大小

buf1.length // 10

buf2.length //12 根据数据自动盛满并创建

//写入数据到buffer
buf1.write("Buffer really rocks!")

//解码buffer

buf1.toString() // 'Buffer rea'

//哦豁，因为buf1只能承载10个字节的内容，所有多处的东西会被截断

//比较两个buffers
```

当然，在Node.js中，还有更多更丰富的方法来操作buffer，你可以[参考这里](https://nodejs.org/dist/latest-v8.x/docs/api/buffer.html)，然后去尝试更多的方法。

最后，我想给你一个小小的挑战：去阅读[zlib.js的源码](https://github.com/nodejs/node/blob/master/lib/zlib.js)，一个Node.js的核心库，去看一下它是如何利用buffer这个神器去操作二进制数据流的。处理后，最后变成gziped文件。 

