> 第六部分 附录

# 附录一 乱码问题解决

- Post请求乱码，web.xml 中加入过滤器

  ```xml
  <!--springmvc 提供的针对 post 请求的编码过滤器-->
  <filter>
      <filter-name>encoding</filter-name>
      <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
      <init-param>
          <param-name>encoding</param-name>
          <param-value>utf-8</param-value>
      </init-param>
  </filter>
  <filter-mapping>
      <filter-name>encoding</filter-name>
      <url-pattern>/*</url-pattern>
  </filter-mapping>
  ```

- Get 请求乱码（Get 请求乱码需要修改 tomcat 下 server.xml 的配置）

  ```xml
  <Connector URIEncoding="utf-8" connectionTimeout="20000" port="8080" protocol="HTTP/1.1" redirectPort="8443"/>
  ```



# 附录二  玩转 SpringMVC 必备设计模式

## 2.1 策略模式

> 策略模式（Strategy），就是一个问题有多种解决方案，选择其中一种使用。这种情况下，我们使用策略模式来实现灵活地选择，也能够方便地增加新的解决方法。比如做数学题，一个问题的解决方法可能有多种；

### 2.1.1 结构

- **策略（Strategy）**

  定义所有支持算法的公共接口。Context 使用这个接口调用某 ConcreteStrategy 定义的算法。

- **策略实现（ConcreteStrategy）**

  实现了 Strategy 接口的具体算法

- **上下文（Context）**

  维护一个 Strategy 对象的引用

  用一个 ConcreteStrategy对象来装配

  可定义一个接口方法让 Strategy 访问它的数据

### 2.1.2 示例

假如现在有一个商场优惠活动，有的商品原价售卖，有的商品打 8.5 折，有的商品打 6 折，有的返现 5 元。

```java
package com.turbo.strategy;

import java.text.MessageFormat;

public class BuyGoods {

    private String goods;
    private double price;
    private double finalPrice;
    private String desc;

    public BuyGoods(String goods, double price) {
        this.goods = goods;
        this.price = price;
    }

    public double calculate(String discountType){
        switch (discountType){
            case "discount85":
                finalPrice = price * 0.85;
                desc = "该商品可享受8.5折优惠";
                break;
            case "discount6":
                finalPrice = price * 0.6;
                desc = "该商品可享受6折优惠";
                break;
            case "return5":
                finalPrice = price >5?price-5:0;
                desc = "该商品可返现5元";
                break;
            default:
                finalPrice = price;
                desc = "对不起，该商品不参与优惠";
                break;
        }
        System.out.println(MessageFormat.format("您购买的商品为：{0}，原价为： {1}，{2}，最终售卖价格为：{3}",
                goods,price,desc,finalPrice));
        return finalPrice;
    }
}
```

测试：

```java
package com.turbo.strategy;

public class Test {

    public static void main(String[] args) {
        BuyGoods buyGoods1 = new BuyGoods("Java编程思想", 99.00);
        buyGoods1.calculate("discount85");

        BuyGoods buyGoods2 = new BuyGoods("雷柏", 199.00);
        buyGoods2.calculate("discount6");

        BuyGoods buyGoods3 = new BuyGoods("Mac Book Air", 9900.00);
        buyGoods3.calculate("return5");

        BuyGoods buyGoods4 = new BuyGoods("富士相机", 5999.00);
        buyGoods4.calculate("null");
    }
}
```

上述代码可以解决问题，但是从代码设计的角度还是存在问题。

- 增加或者修改打折方案时必须修改 BuyGoods 类源代码，违反了面向对象设计的 **开闭原则**，代码的灵活性和扩展性较差。
- 打折方案代码聚合在一起，如果其他项目需要重用某个打折方案的代码，只能复制粘贴对应代码，无法以类组件的方式进行重用，代码的复用性差。
- BuyGoods 类的 calculate() 方法随着优惠方案的增多会非常庞大，代码中会出现跟多 if 分支，可维护性差。

此时，我们可以使用策略模式对 BuyGoods 类进行重构，将打折方案逻辑（算法）的定义和使用分离。

抽象策略类 AbstractDiscount，它是所有具体打折方案（算法）的父类，定义了一个 `discount` 抽象方法。

```java
package com.turbo.strategy.now.discount;

public abstract class AbstractDiscount {

    protected double finalPrice;
    protected String desc;

    public double getFinalPrice() {
        return finalPrice;
    }

    public void setFinalPrice(double finalPrice) {
        this.finalPrice = finalPrice;
    }

    public String getDesc() {
        return desc;
    }

    public void setDesc(String desc) {
        this.desc = desc;
    }

    public AbstractDiscount(String desc) {
        this.desc = desc;
    }

    public abstract double discount(double price);
}
```

四种具体策略类，继承自抽象策略类 AbstractDiscount，并在 `discount` 方法中实现具体的打折方案（算法）

```java
package com.turbo.strategy.now.discount.impl;

import com.turbo.strategy.now.discount.AbstractDiscount;

public class Discount85 extends AbstractDiscount {

    public Discount85() {
        super("该商品可享受8.5折优惠");
    }

    @Override
    public double discount(double price) {
        finalPrice = price * 0.85;
        return finalPrice;
    }
}
```

```java
public class Discount6 extends AbstractDiscount {
    public Discount6() {
        super("该商品可享受6折优惠");
    }

    @Override
    public double discount(double price) {
        finalPrice = price * 0.6;
        return finalPrice;
    }
}
```

```java
public class Return5 extends AbstractDiscount {
    public Return5() {
        super("该商品可返现5元");
    }

    @Override
    public double discount(double price) {
        finalPrice = price >= 5 ? price -5 :0;
        return finalPrice;
    }
}
```

```java
public class NoDiscount extends AbstractDiscount {
    public NoDiscount() {
        super("该商品不参与优惠活动");
    }

    @Override
    public double discount(double price) {
        finalPrice = price;
        return finalPrice;
    }
}
```

BuyGoods 类，维护了一个 AbstractDiscount 引用：

```java
package com.turbo.strategy.now;

import com.turbo.strategy.now.discount.AbstractDiscount;

import java.text.MessageFormat;

public class BuyGoods {

    private String goods;
    private double price;
    private AbstractDiscount abstractDiscount;

    public BuyGoods(String goods, double price, AbstractDiscount abstractDiscount) {
        this.goods = goods;
        this.price = price;
        this.abstractDiscount = abstractDiscount;
    }

    public double calculate(){
        double finalPrice = abstractDiscount.discount(this.price);
        String desc = abstractDiscount.getDesc();
        System.out.println(MessageFormat.format("您购买的商品为：{0}，原价为： {1}，{2}，最终售卖价格为：{3}",
                this.goods,this.price,desc,finalPrice));
        return finalPrice;
    }
}
```

测试：

```java
package com.turbo.strategy.now;

import com.turbo.strategy.now.discount.impl.*;

public class Test {

    public static void main(String[] args) {
        BuyGoods buyGoods1 = new BuyGoods("Java编程思想", 99.00,new Discount85());
        buyGoods1.calculate();

        BuyGoods buyGoods2 = new BuyGoods("雷柏", 199.00,new Discount6());
        buyGoods2.calculate();

        BuyGoods buyGoods3 = new BuyGoods("Mac Book Air", 9900.00,new Return5());
        buyGoods3.calculate();

        BuyGoods buyGoods4 = new BuyGoods("富士相机", 5999.00,new NoDiscount());
        buyGoods4.calculate();
    }
}
```

重构后：

- 增加新的优惠方案时只需要继承抽象策略类即可，修改优惠方案时不需要修改 BuyGoods 类源码。
- 代码复用变得简单，直接复用某一个具体策略类即可。
- BuyGoods 类的calculate 变得简洁，没有了原本的 if 分支。

## 2.2 模板方法模式

模板方法模式是指定义一个算法的骨架，并允许子类为一个挥着多个步骤提供实现。模板方法模式使得子类在不改变算法结构的情况下，重新定义算法的某些步骤，属于行为型设计模式。

采用模板方法模式的核心思路是处理某个流程的代码已经具备，但其中某些节点的代码暂时不能确定。此时可以使用模板方法。

### 2.2.1 示例

```java
package com.turbo.template;

/**
 * 面试流程
 */
public abstract class Interview {

    private final void register(){
        System.out.println("面试登记");
    }

    protected abstract void communicate();

    private final void notifyResult(){
        System.out.println("HR通知面试结果");
    }

    protected final void process(){
        this.register();
        this.communicate();
        this.notifyResult();
    }
}
```

Java面试

```java
package com.turbo.template;

public class Interviewee1 extends Interview {
    @Override
    protected void communicate() {
        System.out.println("我是面试人员1，来面试Java工程师，我们聊的是Java相关内容");
    }
}
```

前端面试：

```java
package com.turbo.template;

public class Interviewee2 extends Interview {
    @Override
    protected void communicate() {
        System.out.println("我是面试人员2，来面试前端工程师，我们聊的是前端相关内容");
    }
}
```

测试：

```java
package com.turbo.template;

public class InterviewTest {

    public static void main(String[] args) {
        // 面试 java
        Interviewee1 interviewee1 = new Interviewee1();
        interviewee1.process();

        // 面试前端
        Interviewee2 interviewee2 = new Interviewee2();
        interviewee2.process();
    }
}
```



## 2.3 适配器模式

