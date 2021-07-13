## redisson+springboot 实现分布式锁

在一些场景时，需要保证数据的不重复，以及数据的准确性，特别是特定下，某些数据的准确性显得尤为重要，所以这个时候要保证某个方法同一时刻只能有一个线程执行。在单机情况下可以用jdk的乐观锁进行保证数据的准确性。而在分布式系统中，这种jdk的锁就无法满足这种场景。

所以需要使用redssion实现分布式锁，它不仅可以实现分布式锁，也可以在某些情况下保证不重复提交，保证接口的幂等性。

redisson是基于redis实现的分布式锁，因为redis执行命令操作时是单线程，所以可以保证线程安全。当然还有其他实现分布式锁的方案，例如zk，MongoDB等。

#### 简单来聊一下各自优缺点

| 方案              | 实现原理                                                     | 优点                                                         |                                                              |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| MongoDB           | 1.加锁：执行findAndModify原子命令查找document，若不存在则新增<br />2.解锁：删除document | 实现较为简单                                                 | 1.大部分公司数据库用MySQL，可能缺乏相应的MongoDB运维、开发人员<br/>2.锁无超时自动失效机制 |
| ZooKeepe          | 1.加锁：在/lock目录下创建临时有序节点，判断创建的节点序号是否最小。若是，则表示获取到锁；否，则则watch /lock目录下序号比自身小的前一个节点<br/>2.解锁：删除节点 | 1.由zk保障系统高可用<br/>2.Curator框架已原生支持系列分布式锁命令，使用简单 | 需单独维护一套zk集群，维保成本高                             |
| redis             | 1. 加锁：执行setnx，若成功再执行expire添加过期时间<br/>2. 解锁：执行delete命令 | 实现简单，相比数据库和分布式系统的实现，该方案最轻，性能最好 | 1.setnx和expire分2步执行，非原子操作；若setnx执行成功，但expire执行失败，就可能出现死锁<br/>2.delete命令存在误删除非当前线程持有的锁的可能<br/>3.不支持阻塞等待、不可重入 |
| redis Lua脚本能力 | 1. 加锁：执行SET lock_name random_value EX seconds NX 命令 <br />2. 解锁：执行Lua脚本，释放锁时验证random_value -- ARGV[1]为random_value, KEYS[1]为lock_name | 同上；实现逻辑上也更严谨，除了单点问题，生产环境采用用这种方案，问题也不大。 | 不支持锁重入，不支持阻塞等待                                 |
| redisson          | redisson这个框架重度依赖了Lua脚本和Netty，加锁、解锁Lua脚本是redisson分布式锁 |                                                              |                                                              |



#### 分布式锁需满足四个条件

首先，为了确保分布式锁可用，我们至少要确保锁的实现同时满足以下四个条件：

1. 互斥性。在任意时刻，只有一个客户端能持有锁。
2. 不会发生死锁。即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁。
3. 解铃还须系铃人。加锁和解锁必须是同一个客户端，客户端自己不能把别人加的锁给解了，即不能误解锁。
4. 具有容错性。只要大多数Redis节点正常运行，客户端就能够获取和释放锁

#### redisson实现分布式锁案例

##### 1、导入依赖

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.3.1</version>
        </dependency>
        <!--<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>-->
        <!-- https://mvnrepository.com/artifact/org.redisson/redisson -->
       <!-- <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson</artifactId>
            <version>3.12.0</version>
        </dependency>-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.16</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.22</version>
        </dependency>
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.9.0</version>
        </dependency>

    </dependencies>
```



##### 2、配置redisson-single(单机)

```yaml
#单机
singleServerConfig:
  idleConnectionTimeout: 10000
  pingTimeout: 1000
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  reconnectionTimeout: 3000
  failedAttempts: 3
  subscriptionsPerConnection: 5
  clientName: null
  address: "redis://localhost:6379"
  subscriptionConnectionMinimumIdleSize: 1
  subscriptionConnectionPoolSize: 50
  connectionMinimumIdleSize: 32
  connectionPoolSize: 64
  database: 0
  #在最新版本中dns的检查操作会直接报错 所以我直接注释掉了
  #dnsMonitoring: false
  dnsMonitoringInterval: 5000
threads: 0
nettyThreads: 0
codec: !<org.redisson.codec.JsonJacksonCodec> {}
transportMode : "NIO"
```

##### 3、配置application

```yaml
server:
  port: 8080
spring: 
  redis:
    host: localhost
    port: 6379
    database: 3
    timeout: 2000
```

##### 4、编写redisson配置类

```java
@Configuration
public class RedissonConfig {


    @Bean(destroyMethod="shutdown")
    public RedissonClient redisson() throws IOException {
        return Redisson.create(
                Config.fromYAML(new ClassPathResource("redisson-single.yml").getInputStream()));
    }

}
```

##### 5、具体业务实现

```java
@Slf4j
@Service
public class GoodsServiceImpl extends ServiceImpl<GoodsMapper, Goods> implements GoodsService {

    @Autowired
    private RedissonClient redissonClient;

    public static final String LOCK_KEY = "lock";


    /**
     * 库存递减
     *
     * @param id  id
     * @param num 数量
     * @return
     */
    @Override
    public boolean killGoods(Long id, Integer num) {
        String key = LOCK_KEY + id;
        RLock lock = redissonClient.getLock(key);
        try {
            //上锁
            lock.lock();
            Goods goods = this.getById(id);
            if (goods.getQuantity()<=0){
                return false;
            }
            log.info("库存数量======"+goods.getQuantity());
            //将库存减操作
            goods.setQuantity(goods.getQuantity()-1);
            this.updateById(goods);

        } catch (Exception e) {
            return false;
        } finally {
            //解锁
            lock.unlock();
        }
        return true;
    }
```

##### 6、接口实现

```java
@RequestMapping
@RestController
public class GoodsController {
    @Resource
    private GoodsService goodsService;

    @GetMapping("test")
    public String createOrderTest() {
        if (!goodsService.killGoods(1405065181720055809L, 1)) {
            return "库存不足";
        }
        return "创建订单成功";
    }

}
```

##### 7、测试，用ab测试工具

模拟200个并发测试

```shell
D:\develop\Apache24\bin>ab  -n 200 -c 200 "http://localhost:8080/test"
```

结果：

![1623922545310](.\img\1623922545310.png)

![1623922677743](.\img\1623922677743.png)

没有库存变成负数的情况，说明分布式锁已生效