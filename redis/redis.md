## 一、redis的基本概念

redis是一个内存数据库

### 1、NOSQL

即 Not-Only SQL（ 泛指非关系型的数据库），作为关系型数据库的补充。

### 2、主要作用

（1）为热点数据加速查询（主要场景）。如热点商品、热点新闻、热点资讯、推广类等高访问量信息等。

（2）即时信息查询。如各位排行榜、各类网站访问统计、公交到站信息、在线人数信息（聊天室、网站）、设备信号等。

（3）时效性信息控制。如验证码控制、投票控制等。

（4）分布式数据共享。如分布式集群架构中的 session 分离消息队列。



## 二、redis基本数据类型（value的数据类型）

### 1、String

以key-value的方式进行存储



![redis存储空间2.png](https://cdn.nlark.com/yuque/0/2021/png/2396018/1629295033198-5e314148-1f28-413b-9c79-b9fa7e583ce9.png)



### 2、hash

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599094139416-93c73156-cdf5-42c7-bb8d-4712496052a7.png)

### 3、list

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599094215214-a0c00951-ea27-4fed-80c1-d78ee54a7e97.png)

### 4、set

set类型：与hash存储结构完全相同，仅存储键，不存储值（nil），并且值是不允许重复的

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599094283116-5a0be673-4192-416f-93c1-690fd46962bc.png)



![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599094319802-c1dc5b8e-1b5d-410e-997a-2e0f2926e56f.png)

### 5、zset



## 三、Jedis

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599107827055-bc35a1ca-ade9-4bd0-98aa-2e61f4870d10.png)

### 1、基于连接池的连接

JedisPool：Jedis提供的连接池技术 

poolConfig:连接池配置对象 

host:redis服务地址

port:redis服务端口号



创建jedis的配置文件：jedis.properties

```
jedis.host=192.168.40.130  
jedis.port=6379  
jedis.maxTotal=50  
jedis.maxIdle=10
```

 创建JedisUtils：com.itheima.util.JedisUtils，使用静态代码块初始化资源

```
public class JedisUtils {
    private static int maxTotal;
    private static int maxIdel;
    private static String host;
    private static int port;
    private static JedisPoolConfig jpc;
    private static JedisPool jp;

    static {
        ResourceBundle bundle = ResourceBundle.getBundle("redis");
        maxTotal = Integer.parseInt(bundle.getString("redis.maxTotal"));
        maxIdel = Integer.parseInt(bundle.getString("redis.maxIdel"));
        host = bundle.getString("redis.host");
        port = Integer.parseInt(bundle.getString("redis.port"));
        //Jedis连接池配置
        jpc = new JedisPoolConfig();
        jpc.setMaxTotal(maxTotal);
        jpc.setMaxIdle(maxIdel);
        jp = new JedisPool(jpc,host,port);
    }

}
```

 对外访问接口，提供jedis连接对象，连接从连接池获取，在JedisUtils中添加一个获取jedis的方法：getJedis

```
public static Jedis getJedis(){
	Jedis jedis = jedisPool.getResource();
	return jedis;
}
```

## 四、数据删除淘汰策略

### 1、定时删除

创建一个定时器，当key设置有过期时间，且过期时间到达时，由定时器任务立即执行对键的删除操作

- **优点**：节约内存，到时就删除，快速释放掉不必要的内存占用
- **缺点**：CPU压力很大，无论CPU此时负载量多高，均占用CPU，会影响redis服务器响应时间和指令吞吐量

- **总结**：用处理器性能换取存储空间（拿时间换空间）



### 2、惰性删除

数据到达过期时间，不做处理。等下次访问该数据时，我们需要判断

1. 如果未过期，返回数据
2. 发现已过期，删除，返回不存在

- **优点**：节约CPU性能，发现必须删除的时候才删除
- **缺点**：内存压力很大，出现长期占用内存的数据

- **总结**：用存储空间换取处理器性能（拿空间换时间）



### 3、定期删除

- Redis启动服务器初始化时，读取配置server.hz的值，默认为10
- 每秒钟执行server.hz次**serverCron()**-------->**databasesCron()**--------->**activeExpireCycle()**

- **activeExpireCycle()**对每个expires[*]逐一进行检测，每次执行耗时：250ms/server.hz
- 对某个expires[*]检测时，随机挑选W个key检测

```
  如果key超时，删除key

  如果一轮中删除的key的数量>W*25%，循环该过程

  如果一轮中删除的key的数量≤W*25%，检查下一个expires[*]，0-15循环

  W取值=ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP属性值
```

- 参数current_db用于记录**activeExpireCycle()** 进入哪个expires[*] 执行
- 如果activeExpireCycle()执行时间到期，下次从current_db继续向下执行

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599114024877-10e18e25-c31d-4681-91a4-2d8e28495bc0.png)

总的来说：定期删除就是周期性轮询redis库中的时效性数据，采用随机抽取的策略，利用过期数据占比的方式控制删除频度

- **特点1**：CPU性能占用设置有峰值，检测频度可自定义设置
- **特点2**：内存压力不是很大，长期占用内存的冷数据会被持续清理

- **总结**：周期性抽查存储空间（随机抽查，重点抽查）



### 4、数据淘汰策略（逐出算法）

当新数据进入redis时，如果内存不足怎么办？在执行每一个命令前，会调用**freeMemoryIfNeeded()**检测内存是否充足。如果内存不满足新 加入数据的最低存储要求，redis要临时删除一些数据为当前指令清理存储空间。清理数据的策略称为逐出算法。**（3类8种）**

**第一类**：检测易失数据（可能会过期的数据集server.db[i].expires ）

```
volatile-lru：挑选最近最少使用的数据淘汰
volatile-lfu：挑选最近使用次数最少的数据淘汰
volatile-ttl：挑选将要过期的数据淘汰
volatile-random：任意选择数据淘汰
```

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599114604805-ede6c9a0-f6ce-45c7-8b29-5e11d97e12dd.png)

**第二类**：检测全库数据（所有数据集server.db[i].dict ）

```
allkeys-lru：挑选最近最少使用的数据淘汰
allkeLyRs-lfu：：挑选最近使用次数最少的数据淘汰
allkeys-random：任意选择数据淘汰，相当于随机
```

**第三类**：放弃数据驱逐

```
no-enviction（驱逐）：禁止驱逐数据(redis4.0中默认策略)，会引发OOM(Out Of Memory)
```

## 五、redis集群

### 1、主从复制

单机的 redis，能够承载的 QPS 大概就在上万到几万不等。对于缓存来说，一般都是用来支撑**读高并发**的。因此架构做成主从(master-slave)架构，一主多从，主负责写，并且将数据复制到其它的 slave 节点，从节点负责读。所有的**读请求全部走从节点**。这样也可以很轻松实现水平扩容，**支撑读高并发**。

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599179290745-364c331d-e1e7-48d6-b5ed-e8be09ac648f.png)

#### 1）主从复制的作用

- 读写分离：master写、slave读，提高服务器的读写负载能力
- 负载均衡：基于主从结构，配合读写分离，由slave分担master负载，并根据需求的变化，改变slave的数 量，通过多个从节点分担数据读取负载，大大提高Redis服务器并发量与数据吞吐量

- 故障恢复：当master出现问题时，由slave提供服务，实现快速的故障恢复
- 数据冗余：实现数据热备份，是持久化之外的一种数据冗余方式

- 高可用基石：基于主从复制，构建哨兵模式与集群，实现Redis的高可用方案



#### 2）主从复制工作流程

主从复制过程大体可以分为3个阶段

- 建立连接阶段（即准备阶段）
- 数据同步阶段

- 命令传播阶段（反复同步）

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599179450095-e5927368-5d42-4286-8aca-7e31de159d8a.png)

##### 阶段一：建立连接

建立slave到master的连接，使master能够识别slave，并保存slave端口号

流程如下：

1. 步骤1：设置master的地址和端口，保存master信息
2. 步骤2：建立socket连接

1. 步骤3：发送ping命令（定时器任务）
2. 步骤4：身份验证

1. 步骤5：发送slave端口信息

至此，主从连接成功！

当前状态：

slave：保存master的地址与端口

master：保存slave的端口

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599179544889-48043514-6677-4aef-8933-9df875174123.png)

##### 阶段二：数据同步

- 在slave初次连接master后，复制master中的所有数据到slave
- 将slave的数据库状态更新成master当前的数据库状态

同步过程如下：

1. 步骤1：请求同步数据
2. 步骤2：创建RDB同步数据

1. 步骤3：恢复RDB同步数据
2. 步骤4：请求部分同步数据

1. 步骤5：恢复部分同步数据

至此，数据同步工作完成！

当前状态：

slave：具有master端全部数据，包含RDB过程接收的数据

master：保存slave当前数据同步的位置

总体：之间完成了数据克隆

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599200916848-0dbfba87-130c-417b-b5fd-9274e219e24c.png)

**数据同步阶段master说明**

1：如果master数据量巨大，数据同步阶段应避开流量高峰期，避免造成master阻塞，影响业务正常执行

2：复制缓冲区大小设定不合理，会导致数据溢出。如进行全量复制周期太长，进行部分复制时发现数据已经存在丢失的情况，必须进行第二次全量复制，致使slave陷入死循环状态。

```
repl-backlog-size ?mb
```

1. master单机内存占用主机内存的比例不应过大，建议使用50%-70%的内存，留下30%-50%的内存用于执 行bgsave命令和创建复制缓冲区

![img](https://cdn.nlark.com/yuque/0/2020/png/2396018/1599985757856-9f821705-f68d-4bf4-a7bb-b4db0ce971ab.png)

**数据同步阶段slave说明**

1. 为避免slave进行全量复制、部分复制时服务器响应阻塞或数据不同步，建议关闭此期间的对外服务

  slave-serve-stale-data yes|no

1. 数据同步阶段，master发送给slave信息可以理解master是slave的一个客户端，主动向slave发送命令
2. 多个slave同时对master请求数据同步，master发送的RDB文件增多，会对带宽造成巨大冲击，如果master带宽不足，因此数据同步需要根据业务需求，适量错峰

1. slave过多时，建议调整拓扑结构，由一主多从结构变为树状结构，中间的节点既是master，也是 slave。注意使用树状结构时，由于层级深度，导致深度越高的slave与最顶层master间数据同步延迟 较大，数据一致性变差，应谨慎选择

##### 2.2.1.3 阶段三：命令传播

- 当master数据库状态被修改后，导致主从服务器数据库状态不一致，此时需要让主从数据同步到一致的状态，同步的动作称为命令传播
- master将接收到的数据变更命令发送给slave，slave接收命令后执行命令

**命令传播阶段的部分复制**

命令传播阶段出现了断网现象：

网络闪断闪连：忽略

短时间网络中断：部分复制

长时间网络中断：全量复制

这里我们主要来看部分复制，部分复制的三个核心要素

1. 服务器的运行 id（run id）
2. 主服务器的复制积压缓冲区

1. 主从服务器的复制偏移量

**服务器运行ID（runid）**

```
概念：服务器运行ID是每一台服务器每次运行的身份识别码，一台服务器多次运行可以生成多个运行id

组成：运行id由40位字符组成，是一个随机的十六进制字符
例如：fdc9ff13b9bbaab28db42b3d50f852bb5e3fcdce

作用：运行id被用于在服务器间进行传输，识别身份
如果想两次操作均对同一台服务器进行，必须每次操作携带对应的运行id，用于对方识别

实现方式：运行id在每台服务器启动时自动生成的，master在首次连接slave时，会将自己的运行ID发送给slave，
slave保存此ID，通过info Server命令，可以查看节点的runid
```

**复制缓冲区**

```
概念：复制缓冲区，又名复制积压缓冲区，是一个先进先出（FIFO）的队列，用于存储服务器执行过的命令，每次传播命令，master都会将传播的命令记录下来，并存储在复制缓冲区
	复制缓冲区默认数据存储空间大小是1M
	当入队元素的数量大于队列长度时，最先入队的元素会被弹出，而新元素会被放入队列
作用：用于保存master收到的所有指令（仅影响数据变更的指令，例如set，select）

数据来源：当master接收到主客户端的指令时，除了将指令执行，会将该指令存储到缓冲区中
```