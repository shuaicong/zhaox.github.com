---
layout: post
title: "数据去重技术原理分析"
description: ""
category: Storage
tags: [Storage]
---
{% include JB/setup %}

数据去重又称重复数据删除，是指在一个数字文件集合中，找出重复的数据并将其删除，只保存唯一的数据单元。在删除的同时，要考虑数据重建，即虽然文件的部分内容被删除，但当需要时，仍然将完整的文件内容重建出来，这就需要保留文件与唯一数据单元之间的索引信息。
![去重原理描述][1]

###应用数据去重技术的好处

 1. 节省存储空间。通过重复数据删除，可以大大降低需要的存储介质数量，进而降低成本。甚至可以使基于硬盘的存储系统成本低于磁带库，同时提供更好的性能。因此，支持数据去重技术的存储系统，特别适合用来做数据的备份。
 2. 提升写入性能。磁盘的写入性能是有限的，通常顺序写入在100MB/s左右，如果在写入数据的时候就进行数据去重，可以避免一部分的数据写入磁盘，从而提升写入性能。
 3. 节省网络带宽。如果在客户端进行数据去重，仅将新增的数据传输到存储系统，可以减少网络上的数据传输量，从而节省网络带宽。

###数据去重的粒度

 1. 文件级别的数据去重。最粗粒度也是最容易实现的一种，通过为文件整体计算一个hash值，对于相同的hash值的文件只存储一份。缺点是去重效果比较差。比较适合变动不太频繁的文件或者小文件。大家都用的百度云盘采用这个级别的数据去重，程序员都用的git也采用这个级别的数据去重。
 2. 固定分块的数据去重。将文件按照offset切分为固定大小的数据块，比如4MB，比如512KB，然后在数据块的级别做去重。这种方法实现简单，还可以用来实现断点续传和并发传输。缺点是去重效果还是比较差，难以应对在文件中间insert数据的情况。360云盘采用512KB的固定分块去重。
 3. 可变分块的数据去重。通过对数据的每一个滑动窗口计算rolling hash，并选取具有满足固定模式的hash值的窗口作为boundary，这样就实现了基于内容的数据分块。然后对数据分块计算hash值，在分块的级别上实现数据去重。这种方式的优点是去重效果好，可以应对数据的各种变化情况。缺点是技术要复杂，包括高效的具有好的区分度的rolling hash，合适的分块大小的选取，性能和存储量之间的折衷等，可以展开写一篇长文了。著名的[Data Domain][2]的存储系统，就使用了这种去重方式。[LBFS][3]也是用这种方式的数据去重，并选用Rabin Fingerprint算法作为rolling hash。
 4. rsync。类unix系统上，大家常用来做备份的rsync，其实也应用了数据去重技术，它通过在服务器端固定分块，在客户端逐字节比较来实现去重。我不确定是否应该算是固定分块还是可变分块，所以单独列了出来。rsync的缺点，是必须有明确的历史版本才能实现去重，不能实现全局去重。rsync只能检测到重复数据，并不能减少存储量。要减少存储量还要使用delta encoding。通过是用类似rsync的算法，得到新增文件与其历史文件的变化值delta，可以不必立即重建这个新增文件并存储，而是只存储这个delta数据，在需要时候重建。进而减少数据存储量。最成功的网盘Dropbox使用这种方式实现数据去重。

###数据去重技术的用途

 1. 备份系统。
 2. 网盘系统。
 3. HTTP页面加载速度提升。可以看看[这个][4]。
 
数据去重与通常说的大家常用的数据压缩的区别，主要在于去重的粒度。数据压缩技术在比较小的范围内以比较小的粒度查找重复数据，粒度一般为几个比特到几个字节。而重复数据删除是在比较大的范围内查找大块的重复数据，一般重复数据块尺寸在1KB以上。
![数据去重VS数据压缩][7]

一个好的数据去重系统，尤其是基于内容的可变分块的数据去重系统，实现难度是比较大的。就目前来看，国内的存储系统厂商基本上还没看到好的产品出来。

相对于应用到备份系统上，如果将数据去重应用到云存储系统上，其实现难度会更拿到，因为云存储系统是一个online系统，不仅要关注系统的吞吐量，更要关注每一次请求的响应延时。贝尔实验室在今年刚刚发表了一篇论文[SEARS][5]来论证这种可能性。

####reference
[重复数据删除—维基百科][6]


  [1]: http://7lryjt.com1.z0.glb.clouddn.com/dedup.png
  [2]: http://www.emc.com/domains/datadomain/index.htm
  [3]: http://cis.poly.edu/cs623/lbfs.pdf
  [4]: https://code.google.com/p/diffable/
  [5]: http://arxiv.org/pdf/1508.01182.pdf
  [6]: https://zh.wikipedia.org/wiki/%E9%87%8D%E5%A4%8D%E6%95%B0%E6%8D%AE%E5%88%A0%E9%99%A4
  [7]: http://7lryjt.com1.z0.glb.clouddn.com/dedup%20vs%20compression.png