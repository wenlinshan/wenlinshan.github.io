在一些复杂的业务当中，涉及到某个业务有多个状态，按照传统写法无非就是`if else `，其实还是有一种比较优雅的实现方式，就是设计模式中的状态机模式。没有spring之前虽然也能实现状态机模式，但是并不优雅。下面来说一个用springboot来实现状态机模式的案例。

  	举一个例子，比如订单：订单当中涉及到多个状态的跳转，有的时候还需要对修改状态前的逻辑进行判断，这个时候用状态机模式就是很好的实现。

##### 1、引入部分依赖

```xml
<groupId>org.example</groupId>
    <artifactId>order_demo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <parent>
        <artifactId>spring-boot-parent</artifactId>
        <groupId>org.springframework.boot</groupId>
        <version>2.3.0.RELEASE</version>
    </parent>
    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.20</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```
##### 主要的包结构:

![](.\images\Snipaste_2021-06-09_22-35-07.png)

##### 2、订单类：

```java
/**
 * @version 1.0
 * @date 2021/6/9 21:33
 * 订单类
 */
@AllArgsConstructor
@NoArgsConstructor
@Data
public class Order implements Serializable {
    /**
     * id
     */
    private Long id;
    /**
     * 金额
     */
    private BigDecimal price;
    /**
     * 状态
     */
    private Integer status;
    /**
     * 创建时间
     */
    private Date createTime;
}
```

##### 3、创建订单状态处理超类

```java
package org.example.bll;

import org.example.pojo.Order;

/**
 * @version 1.0
 * @date 2021/6/9 21:43
 * 订单状态处理超类
 */
public interface OrderState {
    /**
     * 原来状态
     * @param newOrder 新订单
     * @param oldOrder 老订单
     */
    void handle(Order newOrder,Order oldOrder);
}
```

##### 4、各个状态处理类

```java
package org.example.bll;

import org.example.pojo.Order;

/**
 * @version 1.0
 * @date 2021/6/9 21:46
 * 订单各状态处理
 */
public class OrderStateCommon implements OrderState {
    @Override
    public void handle(Order newOrder, Order oldOrder) {

    }

    /**
     * 生成日志
     *
     * @param order 订单
     */
    public void buildOrderLog(Order order) {
        //保存日志
        System.out.println("保存日志订单状态: " + order.getStatus());
    }
}
```

```java
import org.example.pojo.Order;
import org.springframework.stereotype.Component;

import java.util.Objects;

/**
 * @version 1.0
 * @date 2021/6/9 21:52
 * 订单状态: 1
 */
@Component(OrderStateFactory.ORDER_STATUS + 1)
public class OrderStateFor1 extends OrderStateCommon {
    @Override
    public void handle(Order newOrder, Order oldOrder) {
        if (Objects.isNull(oldOrder)){
            System.out.println("不做操作");
        }
        System.out.println("订单状态:1 处理逻辑==>");
        //保存日志
        super.buildOrderLog(newOrder);
    }
}
```

```java
package org.example.bll;

import org.example.pojo.Order;
import org.springframework.stereotype.Component;

/**
 * @version 1.0
 * @date 2021/6/9 21:52
 * 订单状态: 2
 */
@Component(OrderStateFactory.ORDER_STATUS + 2)
public class OrderStateFor2 extends OrderStateCommon{
    @Override
    public void handle(Order newOrder, Order oldOrder) {
        System.out.println("订单状态:2 处理逻辑==>");
        //保存日志
        super.buildOrderLog(newOrder);
    }
}
```

```java
package org.example.bll;

import org.example.pojo.Order;
import org.springframework.stereotype.Component;

/**
 * @version 1.0
 * @date 2021/6/9 21:52
 * 订单状态: 3
 */
@Component(OrderStateFactory.ORDER_STATUS + 3)
public class OrderStateFor3 extends OrderStateCommon{
    @Override
    public void handle(Order newOrder, Order oldOrder) {
        System.out.println("订单状态:3 处理逻辑==>");
        //保存日志
        super.buildOrderLog(newOrder);
    }
}
```

##### 5、工厂类

```java
package org.example.bll;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @author wenlinshan
 * @version 1.0
 * @date 2021/6/9 21:56
 */
@Component
public class OrderStateFactory {
    /**
     * 自定义bean名字前缀
     */
    public static final String ORDER_STATUS = "orderStatus";
    /**
     * 注入到容器里面
     */
    @Autowired
    Map<String, OrderState> states = new ConcurrentHashMap<>(4);

    /**
     * 获取对应状态的执行类
     *
     * @param status 状态
     * @return 具体状态的执行类
     */
    public OrderState getState(Integer status) {
        if (states.containsKey(ORDER_STATUS + status)) {
            return states.get(ORDER_STATUS + status);
        }
        throw new RuntimeException("未找到执行状态的类");
    }
}
```

##### 6、主要的入口类代码

```java
package org.example.service.impl;

import org.example.bll.OrderState;
import org.example.bll.OrderStateFactory;
import org.example.pojo.Order;
import org.example.service.IOrderService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Objects;

/**
 * @version 1.0
 * @date 2021/6/9 21:37
 */
@Service
public class OrderServiceImpl implements IOrderService {
    @Autowired
    private OrderStateFactory factory;
    @Override
    public void saveOrUpdateOrder(Order order) {
        //查找出原来的订单
        Order oldOrder = this.getById(order.getId());
        if (Objects.isNull(oldOrder)) {
            //新增
            this.save(order);
            if (Objects.nonNull(order.getStatus())) {
                //调用状态机
                factory.getState(order.getStatus()).handle(order,null);
            }
            return;
        }
        //修改
        OrderState orderState = Objects.isNull(order.getStatus()) ? factory.getState(oldOrder.getStatus()) : factory.getState(order.getStatus());
        orderState.handle(order,oldOrder);

    }
```

至此，所有的代码都已实现，下面开始演示‘

##### 7、测试类：

```java
package org.example;

import org.example.pojo.Order;
import org.example.service.IOrderService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.math.BigDecimal;
import java.util.Date;

/**
 * @version 1.0
 * @date 2021/6/9 22:03
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class OrderTest {
    @Autowired
    private IOrderService orderService;

    @Test
    public void testOrder1() {
        Order order = new Order(1L, BigDecimal.ONE, 2, new Date());
        orderService.saveOrUpdateOrder(order);
    }
    
}
```

运行结果：

![](.\images\Snipaste_2021-06-09_22-44-33.png)