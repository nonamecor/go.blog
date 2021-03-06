# Go手动内存分配

用Go的时候，有时候又想自己管理内存。所以决定写个手动内存管理的包吧。就当无聊练练手...

## 总体设计

两级分配。较大内存以页为单位分配，每页4k。分配出去的大块内存只能是1页，2页，3页...较小内存使用buddy算法结合分配池的方式进行分配。buddy算法主要是方便回收，对于各种不规则大小则分别维护一个free的链表。

整体上基本类似于Go本身使用的内存管理算法，除了没有引入垃圾回收标记信息，以及对小对象使用buddy分配算法。

## buddy算法

管理结点使用的内存跟最小最大结点相关。如果最小单元太小，则浪费过多的管理结点。管理结点数目=最小单元数目*2   如果最大单元过大，则需要用更长的整型来记录大小，单个管理结点的大小增加。

buddy算法管理的每块内存大小为选择为4k。更大内存则用页分配器管理。最小单元大小选择为32B，这样可以用8位存储结点大小(8位最多可以表示255个单元结点，4k包含128个大小32的单元结点)。

管理结点使用uint8，一页中有64个结点，所以4k的管理结点只需要64B，很省！

```Go
size [64]uint8
```

## 改进的buddy算法

本来很喜欢buddy分配器的，不过buddy适用性不那么好，能分配的大小类型只有32B，64B，128B，256B，512B，1024B，2048B，4096B。基本的buddy算法有个很严重的缺点，内部碎片浪费很严重。比如申请66B的内存，会分配128B，太浪费了！

所以我做了一些改动，用不同的最小单元的buddy分配器进行插值。比如说基准的最小单无大小是16，那么它负责的大小是:

16 32 64 128 256 512 1024 2048

如果我选择另一个最小单元为24的buddy分配器，它负责的大小是：

24 48 96 192 384 768 1536 3072 

二者配合一起用，内存利用率会高很多。依此下去，可以弄一些最小单元大小不一的buddy分配器，那么它就可以弥补2的指数倍导致过多内部碎片的问题了。我计算了一个，最小单元使用到56的时候内存浪费率已经不会超过1:1.125，大概是1.1111多的样子。

但是这样做引出了另一个问题，外部碎片。就是各个buddy分配器之间，必须要正好凑满4096B才不至于浪费。

	16 32 64 128 256 512 1024 2048
	24 48 96 192 384 768 1536 3072 		(768 + 1280)	A
	40 80 160 320 640 1280 2560  		(640 + 1408)	B
	56 112 224 448 896 1792 3584 		(896 + 1152)	C
	72 144 288 576 1152 2304 			(1152 + 896)	D
	88 176 352 704 1408 2816	(704 1344)  (1408 640)	E
	104 208 416 832 1664 3328 	(1664 384)				F
	120 240 480 960 1920 3840	()						G
	168 336 672 1344 2688 								H

所以我精心设计了一些组合，刚好凑起来4096B的。

	组合 1408 1408 1280 E E B
	组合 1536 1280 1280 A B B
	组合 896 1152 896 1152 C D C D

	组合 768 1280 768 1280 A B A B
	组合 1408  1344 1344	E H H
	组合 896 1152 C D 

事实上只用到E就够了，也就是前三个组合。所以选用的buddy分配器为：

	B 1280
	E 1408
	A 1536
	D 1152
	C 896

## 小对象分配池

buddy的使用其实更多是想方便回收时的内存合并。为了加快分配速度，维护一个分配池，对每个大小类别维持一个free链表，分配时直接从链表中划一个出去，回收看是否需要进行合并。

一个大的buddy管理的块被切小后，变成好多小对象挂到free链表中。

## 页分配器

这个就跟Go语言采用同样的方式弄了。每次分配时是几个页大小的内存，回收用位图的方式来进行合并回收。

好吧，不多废话，上[代码](http://github.com/tiancaiamao/memory)