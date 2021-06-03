介绍Redis的三种复杂数据类型：位图、HyperLoLog和GeoHash

### 一、位图

位图的数据结构是为了节省存储空间的，对于需要签到、记录人数这类场景，可以使用位图数组的数据结构，相比于byte数组，大大节约了空间。

<font size=4>**常见指令**</font>

```
1.>setbit s 1 1 #设置key为s的value位置索引1是1
2.>getbit s 1 1  #取出kye为s的value位置索引是1
(integer) 1
3.>get s  #取出位数组前8位构成的字符
"@"
4.>bitcount w #统计位图中出现1的个数
(integer) 21
5.>bitcount w 0 2 #统计位图中前三个字符中出现1的个数
6.>bitpos w 0 0 2 #统计位图中前三个字符中0出现的bit位  
7.>bitfield w get u4 0 get u3 2 get i4 0 get i3 2  #可以一次性执行多条指令，分别是从第一个字符开始，取出w的无符号4个位
第3个字符开始取3个无符号位；第1个字符开始，4个有符号位；第3个字符开始，3个有符号位（这里有符号指的是位数组中第一个是符号位）
```

<font size=4>**应用场景**</font>

1. 签到

比如设置key是某用户名，然后按照分配一个360天的位数组，每次签到的话，就将位数组的那一位置为1，最后统计整个位数组中1的个数。

### 二、**HyperLogLog** 

对于需要统计大型网站的UV数据，因为需要去重，这样每次请求的时候都需要带上用户ID，但是用户的ID不适宜放到Set集合里面，因为页面访问量极大，存储空间惊人，因此可以利用HyperLogLog 来进行统计，但是统计不是很精确。

<font size=4>**常见指令**</font>

```
1.>pfadd code user1
2.>pfcount code
(integer) 1
3.>pfmerge code3 code1 code2 #将code1和code2数值合并到code3中
```

<font size=4>**内存占用12k？**</font>

Redis的HyperLogLog中用到了16384个桶，每个桶需要6个bit存储，所以总共占用内存是12k个字节

<font size=4>**应用场景**</font>

因为如果要使用它的话，至少12K内存，因此如果单个用户计数是不划算的，所以如果统计大型网站的UV是非常可以的

### 三、**GeoHash** 

这个数据类型可以存储经纬度，然后计算两个点的距离。

<div align="center">  <img src="https://i.bmp.ovh/imgs/2021/06/1e88b4f341cd09e7.png" width="400"/> </div><br>

<font size=4>**常见指令**</font>

```
1.>geoadd company 116.489033 40.007669 meituan 116.334255 40.027400 xiaomi #用来增加多个经纬度和名称
(integer) 2
2.>geodist company meituan xiaomi km  #用来计算两个元素之间的距离
"13.3659"
3.>geopos company meituan  #获取元素位置
1) 1) "116.48903220891952515"
   2) "40.00766997707732031"
4.>geohash company meitian  #获取元素的hash值，将经纬度编码成字符串，然后可以利用这个编码进入某个网址取直接定位
1) "wx4gdg0tx40"
5.>georadiusbymember company meituan 20 km count 3 asc  #查询指定元素附近的其他元素
```

<font size=4>**应用场景**</font>

用来计算某两个位置的距离，实现附近的人和附近的餐馆功能。