
![MistLogo](http://can.sfhfpc.com/mistLogo-300px.png)

# Mist 薄雾算法

薄雾算法是不同于 snowflake 的全局唯一 ID 生成算法。相比 snowflake ，薄雾算法具有更高的数值上限和更长的使用期限。

## 考量了什么业务场景和要求呢？

用到全局唯一 ID 的场景不少，这里引用美团 Leaf 的场景介绍:

> 在复杂分布式系统中，往往需要对大量的数据和消息进行唯一标识。如在美团点评的金融、支付、餐饮、酒店、猫眼电影等产品的系统中，数据日渐增长，对数据分库分表后需要有一个唯一 ID 来标识一条数据或消息，数据库的自增 ID 显然不能满足需求；特别一点的如订单、骑手、优惠券也都需要有唯一 ID  做标识。此时一个能够生成全局唯一ID 的系统是非常必要的。

引用微信 seqsvr 的场景介绍：

> 微信在立项之初，就已确立了利用数据版本号实现终端与后台的数据增量同步机制，确保发消息时消息可靠送达对方手机。

爬虫数据服务的场景介绍：

> 数据来源各不相同，且并发极大的情况下难以生成统一的数据编号，同时数据编号又将作为爬虫下游整个链路的溯源依据，在爬虫业务链路中十分重要。


这里参考美团 [Leaf](https://tech.meituan.com/2017/04/21/mt-leaf.html) 的要求：

1、全局唯一性：不能出现重复的 ID 号，既然是唯一标识，这是最基本的要求；

2、趋势递增：在 MySQL InnoDB 引擎中使用的是聚集索引，由于多数 RDBMS 使用 B-tree 的数据结构来存储索引数据，在主键的选择上面我们应该尽量使用有序的主键保证写入性能；

3、单调递增：保证下一个 ID 一定大于上一个 ID，例如事务版本号、IM 增量消息、排序等特殊需求；

4、信息安全：如果 ID 是连续的，恶意用户的爬取工作就非常容易做了，直接按照顺序下载指定 URL 即可；如果是订单号就更危险了，竞对可以直接知道我们一天的单量。所以在一些应用场景下，会需要 ID 无规则、不规则；

可以用“全局不重复，不可猜测且呈递增态势”这句话来概括描述要求。

## 薄雾算法的设计思路是怎么样的？

薄雾算法采用了与 snowflake 相同的位数——64，在考量业务场景和要求后并没有沿用 1-41-10-12 的占位，而是采用了 1-47-8-8 的占位。即：
```
* 1      2                                                     48         56       64
* +------+-----------------------------------------------------+----------+----------+
* retain | increas                                             | salt     | sequence |
* +------+-----------------------------------------------------+----------+----------+
* 0      | 0000000000 0000000000 0000000000 0000000000 0000000 | 00000000 | 00000000 |
* +------+-----------------------------------------------------+------------+--------+
```
- 第一段为最高位，占 1 位，保持为 0，使得值永远为正数；
- 第二段放置自增数，占 47 位，自增数在高位能保证结果值呈递增态势，遂低位可以为所欲为；
- 第三段放置随机因子，占 8 位，上限数值 255，使结果值不可预测；
- 第四段放置序列号，占 8 位，上限数值 255，表示同一毫秒生成的第 N 个数；


![mistSturct](http://can.sfhfpc.com/mist.png)

## 薄雾算法生成的数值是什么样的？

薄雾自增数为 1～10 的运行结果如下：

```
193024
221185
289026
343811
438788
464389
588038
611079
677896
744457
```

根据运行结果可知，薄雾算法能够满足“全局不重复，不可猜测且呈递增态势”的场景要求。

## 薄雾算法 mist 和雪花算法 snowflake 有何区别？

snowflake 是由 Twitter 公司提出的一种全局唯一 ID 生成算法，它具有“递增态势、不依赖数据库、高性能”等特点，自 snowflake 推出以来备受欢迎，算法被应用于大大小小公司的服务中。snowflake 高位为时间戳的二进制，遂完全收到时间戳的影响，倘若时间回拨（当前服务器时间回到之前的某一时刻），那么 snowflake 有极大概率生成与之前同一时刻的重复 ID，这直接影响整个业务。

snowflake 受时间戳影响，使用上限不超过 70 年。

薄雾算法 Mist 由书籍《Python3 反爬虫原理与绕过实战》的作者韦世东综合 [百度 UidGenerator](https://github.com/baidu/uid-generator)、 [美团 Leaf](https://tech.meituan.com/2017/04/21/mt-leaf.html) 和 [微信序列号生成器 seqsvr](https://www.infoq.cn/article/wechat-serial-number-generator-architecture) 中介绍的技术点，同时考虑高性能分布式序列号生成器架构后设计的一款“递增态势、不依赖数据库、高性能且不受时间回拨影响”的全局唯一序列号生成算法。

薄雾算法不受时间戳影响，受到数值大小影响。薄雾算法高位数值上限计算方式为`int64(1<<47 - 1)`，上限数值`140737488355327` 百万亿级，假设每天消耗 10 亿，薄雾算法能使用 385+ 年。

## 为什么薄雾算法不受时间回拨影响？

snowflake 受时间回拨影响的根本原因是高位采用时间戳的二进制值，而薄雾算法的高位是按序递增的数值。结果值的大小由高位决定，遂薄雾算法不受时间回拨影响。

## 为什么说薄雾算法的结果值不可预测？

考虑到“不可预测”的要求，薄雾算法的中间位是 8 位随机值，且末 8 位是同一毫秒时间戳的递增序列号，因此结果值不可预测。

## 当程序重启，薄雾算法的值会重复吗？

snowflake 受时间回拨影响，一旦时间回拨就有极大概率生成重复的 ID。薄雾算法中的高位是按序递增的数值，程序重启会造成按序递增数值回到初始值，因此一定会生成重复的 ID。

## 薄雾算法的值会重复，那我要它干嘛？

无论是什么样的全局唯一 ID 生成算法，都会有优点和缺点。在实际的应用当中，没有人会将全局唯一 ID 生成算法完全托付给程序，而是会用数据库存储关键值或者所有生成的值。全局唯一 ID 生成算法大多都采用分布式架构或者主备架构提供发号服务，这时候就不用担心它的重复问题。

## 是否提供薄雾算法的工程实践或者架构实践？

是的，作者的另一个项目 [mistkv](https://github.com/asyncins/mistkv) 是薄雾算法与 Redis 的结合，实现了“全局不重复”，你再也不用担心程序重启带来的问题。

## 薄雾算法的分布式架构，推荐 CP 还是 AP？

CAP 是分布式架构中最重要的理论，C 指的是一致性、A 指的是可用性、P 指的是分区容错性。CAP 当中，C 和 A 是互相冲突的，且 P 一定存在，遂我们必须在 CP 和 AP 中选择。**实际上这跟具体的业务需求有关**，但是对于全局唯一 ID 发号服务来说，大多数时候可用性比一致性更重要，也就是选择 AP 会多过选择 CP。至于你怎么选，还是得结合具体的业务场景考虑。


### 薄雾算法的性能测试

采用 Golnag（1.14） 自带的 Benchmark 进行测试，测试机硬件环境如下：

```
内存 16 GB 2133 MHz LPDDR3
处理器 2.3 GHz 双核Intel Core i5
操作系统 MacBook Pro (13-inch, 2017, Two Thunderbolt 3 ports)
```


进行了多轮测试，随机取 3 轮测试结果。以此计算平均值，得 `单次执行时间 8804 ns/op`。以下是随机 3 轮测试的结果：


```
goos: darwin
goarch: amd64
pkg: mist
BenchmarkGenerate-4       118484              8863 ns/op
PASS
ok      mist    1.345s
```

```
goos: darwin
goarch: amd64
pkg: mist
Benchmark_Mist-4          132512              8768 ns/op
PASS
ok      mist    1.382s
```

```
goos: darwin
goarch: amd64
pkg: mist
Benchmark_Mist-4          133716              8782 ns/op
PASS
ok      mist    1.394s
```


