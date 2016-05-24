title: 设计模式汇总
tags:
  - 设计模式
  - java
categories:
  - 设计模式
date: 2016-05-16 15:16:18
---

1. 单例模式 
    - [http://www.hollischuang.com/archives/1373](http://www.hollischuang.com/archives/1373)
    - JDK中的单例 [http://www.hollischuang.com/archives/1383](http://www.hollischuang.com/archives/1383)

<!--more-->

2. 简单工厂模式 [http://www.hollischuang.com/archives/1391](http://www.hollischuang.com/archives/1391)
3. 工厂方法模式 
    - [http://www.hollischuang.com/archives/1401](http://www.hollischuang.com/archives/1401)
    - JDK中的工厂方法 [http://www.hollischuang.com/archives/1408](http://www.hollischuang.com/archives/1408)
4. 抽象工厂模式 [http://www.hollischuang.com/archives/1420](http://www.hollischuang.com/archives/1420)
5. 工厂模式总结 [http://www.hollischuang.com/archives/1430](http://www.hollischuang.com/archives/1430)
6. 建造者模式：[http://www.hollischuang.com/archives/1477](http://www.hollischuang.com/archives/1477)
    - 实例 [http://www.hollischuang.com/archives/1533](http://www.hollischuang.com/archives/1533)
    - 工厂模式是将对象的全部创建过程封装在工厂类中，由工厂类向客户端提供最终的产品；而建造者模式中，建造者类一般只提供产品类中各个组件的建造，而将具体建造过程交付给导演类。由导演类负责将各个组件按照特定的规则组建为产品，然后将组建好的产品交付给客户端。
    - 建造者模式更灵活地定制不同属性的组合、属性构建的顺序等，这些都可以通过新建导演类来实现。
7. 适配器模式 [http://www.hollischuang.com/archives/1524](http://www.hollischuang.com/archives/1524)
    - 活性和扩展性都非常好，通过使用配置文件，可以很方便地更换适配器，也可以在不修改原有代码的基础上增加新的适配器类


