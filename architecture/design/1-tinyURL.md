# 短链接

## 使用场景(Scenario)

微博和Twitter都有140字数的限制，如果分享一个长网址，很容易就超出限制，发布出去。短网址服务可以把一个长网址变成短网址，方便在社交网络上传播。

## 需求(Needs)

很显然，要尽可能的短。长度设计为多少才合适呢？

## 短网址的长度

当前互联网上的网页总数大概是 45亿(参考 短网址_短网址资讯`mrw.so`)，45亿 超过了 `2^{32}=4294967296232=4294967296`，但远远小于64位整数的上限值，那么用一个64位整数足够了。微博的短网址服务用的是长度为 `7` 的字符串，这个字符串可以看做是62进制的数，那么最大能表示`{62}^7=3521614606208627=3521614606208`个网址，远远大于 45亿。所以长度为7就足够了。一个64位整数如何转化为字符串呢？，假设我们只是用大小写字母加数字，那么可以看做是62进制数，`log_{62{(2^{64}-1)=10.7log62(264−1)=10.7`，即字符串最长11就足够了。实际生产中，还可以再短一点，比如新浪微博采用的长度就是7，因为 62^7=3521614606208627=3521614606208，这个量级远远超过互联网上的URL总数了，绝对够用了。现代的web服务器（例如Apache, Nginx）大部分都区分URL里的大小写了，所以用大小写字母来区分不同的URL是没问题的。因此，正确答案：长度不超过7的字符串，由大小写字母加数字共62个字母组成。

## 一对一还是一对多映射？

一个长网址，对应一个短网址，还是可以对应多个短网址？ 这也是个重大选择问题。一般而言，一个长网址，在不同的地点，不同的用户等情况下，生成的短网址应该不一样，这样，在后端数据库中，可以更好的进行数据分析。如果一个长网址与一个短网址一一对应，那么在数据库中，仅有一行数据，无法区分不同的来源，就无法做数据分析了。

以这个7位长度的短网址作为唯一ID，这个ID下可以挂各种信息，比如生成该网址的用户名，所在网站，HTTP头部的 User Agent等信息，收集了这些信息，才有可能在后面做大数据分析，挖掘数据的价值。短网址服务商的一大盈利来源就是这些数据。

正确答案：一对多

## 如何计算短网址

现在我们设定了短网址是一个长度为7的字符串，如何计算得到这个短网址呢？

最容易想到的办法是哈希，先hash得到一个64位整数，将它转化为62进制整，截取低7位即可。但是哈希算法会有冲突，如何处理冲突呢，又是一个麻烦。这个方法只是转移了矛盾，没有解决矛盾，抛弃。

正确答案：分布式发号器(`Distributed ID Generator`)

## 如何存储

如果存储短网址和长网址的对应关系？以短网址为 `primary key`, 长网址为`value`, 可以用传统的关系数据库存起来，例如`MySQL,PostgreSQL`，也可以用任意一个分布式 KV 数据库，例如`Redis, LevelDB`。

## 301还是302重定向

这也是一个有意思的问题。这个问题主要是考察你对301和302的理解，以及浏览器缓存机制的理解。

301是永久重定向，302是临时重定向。短地址一经生成就不会变化，所以用301是符合http语义的。但是如果用了301， `Google`，`百度`等搜索引擎，搜索的时候会直接展示真实地址，那我们就无法统计到短地址被点击的次数了，也无法收集用户的`Cookie`, `User Agent` 等信息，这些信息可以用来做很多有意思的大数据分析，也是短网址服务商的主要盈利来源。

所以，正确答案是302重定向。

可以抓包看看mrw.so的短网址是怎么做的，使用 Chrome 浏览器，访问这个URL `http://mrw.so/4UD39p`，是我事先发微博自动生成的短网址。来抓包看看返回的结果是啥，可见新浪微博用的就是302临时重定向。