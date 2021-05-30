Redis有5种基本的数据结构，分别是String、List、Set、Zset、Hash，其中这些'数据结构指的是<font color=red>value，Redis中的所有key都是字符串作为名称。</font>

接下来就一一介绍这些基本数据结构的常见指令和应用场景

### 一、String (字符串）

​	<font size=4>**常见指令**</font>

```
1.>set name java
2.>get name 
 "java"
3.>expire name 5.  #5s后过期
4.>get name 
 (nil)
5.>setex name 5 java #5s后过期,等价于1和3结合
6.>setnx name java #如果name这个key已经存在，那么就不会set成功，否则就会设置key是name，value是java

7.>set count 10 #虽然说10是字符串，但是存的就是10数值
8.>incr count
 (integer) 11
9.>incrby count 5 #在当前基础上+5，但是范围大小介于有符号long的最大值和最小值之间，超过，redis就会报错
 (integer) 16
10.>incrby count -5
 (integer) 11
	
11.>getrange name start end #获取key的name的value索引是[start,end]的子串，如果要到最后，可以用-1替代end
12.>mset name1 java name2 python #同时设置一个或多个 key-value
13.>append name value2 #如果key存在，将value2附加到name的value上
```

<font size=4>**应用场景**</font>

1. 计数

利用Redis中incrby命令实现原子性递增，比如限制一个手机号发送多少天短信，一个接口一分钟限制的请求数量。

2. 热点数据、token缓存

因为Redis读取速度超级快，因此可以把热点数据缓存，token也可以存储，通过expire可以实现过期操作。但是这里热点数据缓存会造成如果数据库更新了，但是热点数据还没有更新，也就是缓存不一致，解决方案自行搜索哦！

3. 分布式锁

利用setnx命令，如果成功是1，否则返回0。两台机器，一台通过setnx设置了一个lock锁，成功之后去执行自己的业务了，另一台机器如果也想执行业务，也需要setnx，但是由于刚才的lock锁没有释放，只能干巴巴的等着了。

### 二、List (列表）

Redis中的列表相当于LinkedList，当列表里面最后一个元素被弹出，该List也就会自动被删除。

​	<font size=4>**常见指令**</font>

```
1.>rpush name java python golang #列表里面存储3个元素,右边进
2.>llen name
 (integer) 3
3.>lpop name #左边出,右进坐出[队列]
  "java"
4.>rpop name #右边出，右进右出[栈]
  "golang"
5.>lindex name 1 #获取列表中索引是1的vaule值,时间复杂度是o(n)
6.>llen name #获取列表长度

7.>lrem name count value #count>0,从表头搜索count个值等于value，然后移除;count<0,从表尾开始;count=0，移除所有值为value
8.>lrange name 0 -1 #获取列表中所有元素
9.>blpop name 10 #如果队列没有元素，就会阻塞等待至最多10秒，然后从左边移除元素，brpop同理，但是在这没出来之前当时的解决方案是
ThreadSleep(1),这就会导致如果消费者较少，等待时间就会挺长，但是blpop是一旦来了元素就瞬间弹出，没有就等待。
10.>lset name index value #设置列表中索引为index的值是value
```

​	<font size=4>**应用场景**</font>

1. 队列

利用lpush、rpop可以较好实现队列操作

### 三、Hash (字典）

Redis中的字典相当于HashMap，即数组+链表的二维结构。

​	<font size=4>**常见指令**</font>

```
1.>hset name java "thind in java" #这里的java "thind in java"相当于hashmap的key和value
2.>hset name python "python go"
3.>hgetall name #这里出现的是entry，key和value间隔出现
 1) "java"
 2) "thind in java"
 3) "python"
 4) "python go"
4.>hlen name
 (integer) 2
5.>hget name java 
 "thind in java"
6.>hset name java "java 40 days"

7.>hset user zhang 20
8.>hincrby user zhang 2
 (integer) 22
9.>hdel user zhang
```

​	<font size=4>**应用场景**</font>

1. 存储对象

利用hmset user age 20 sex boy不需要像字符串一次性序列化整个对象，可以直接对用户结构的每一个字段单独存储，取出用户信息的时候也可以部分获取，在一定程度上节省了网络流量。

### 四、Set (集合）

Redis中的集合相当于HashSet，由于Java中的HashSet相当于HashMap的value为null，这里的Set同理，当集合中最后一个元素被删除之后，数据结构也就会自动删除。

​	<font size=4>**常见指令**</font>

```
1.>sadd name java
2.>sadd name java #重复，返回0
3.>sadd name python go
4.>smembers name #返回的顺序无序
 1) "java"
 2) "python"
 3) "go"
5.>sismember name java #查询某个value是否存在，存在返回1
6.>scard	name #获取name这个set的长度
 (integer) 3
7.>spop books #随机弹出一个
 "java"
```

​	<font size=4>**应用场景**</font>

1. 用户、点赞去重

由于set存储的元素不可能重复，因此可以避免相同的用户同时中奖或者点赞重复。

2. 好友共同关注

微博中用户关注的人存在一个set结合中，然后很容易求sinter key1 key2，求出用户共同关注。

### 五、Zset (有序列表）

Redis中的有序列表类似于SortedSet和HashMap的结合，这是由于它一方面可以保持内部value的唯一性，另一方面每一个value赋予一个score，代表它的排序。

​	<font size=4>**常见指令**</font>

```
1.>zadd name 9.0 "think in java"
2.>zadd name 8.0 "java 40 days"
3.>zadd name 8.6 "java fighting"

4.>zrange name 0 -1 #按照得分升序排列
  1) "java 40 days"
  2) "java fighting"
  3) "think in java"

5.>zrevrange name 0 -1
  1) "think in java"
  2) "java fighting"
  3) "java 40 days"

6.>zcard name #获取列表长度
7.>zscore name “think in java" #获取指定value的score
8.>zrank name "think in java"	#获取指定value的排名
9.>zrem name “think in java" #删除value
```

​	<font size=4>**应用场景**</font>

1. 排行榜相关问题

比如要展示某个部门的点赞排行榜，可以设置value是用户id，点赞数作为score。
