---
title: Dagger lerning
date: 2016-09-02 18:48:36
tags: Dagger
---

# 依赖注入
依赖是我们代码中两个模块之间的耦合，通常是一个模块需要用到另外一个模块。
```java
public interface Led{
    void on();
    void off();
}

public class A implements Led{
     void on() {
         System.out.println("A on");
     }
    void off(){
         System.out.println("A off");
     }
}

public class B implements Led{
     void on() {
         System.out.println("B on");
     }
    void off(){
         System.out.println("B off");
     }
}

public class User {
    ...
    Led led;
    ...
    public user() {
        led = new B();
    }
}

```
上面的代码中Class User 就依赖Class B 。

上面的实现明显有问题，类User中不应该出现Led的子类。明显应该把类A或者类B的实例作为参数传。
可以如下实现。
```java
public class User {
    ...
    Led led;
    ...
    public user(Led led) {
        this.led = led;
    }
}
```

修改之后的代码耦合性明显降低，这就叫依赖注入。 依赖注入有两个好处：

- 解耦  耦合度降低
- 方便测试

**依赖注入的核心思想就是，被依赖者不要在依赖者里面创建，而是别的地方创建，然后通过注入器注入到依赖者里面。**

# 依赖注入器Dagger
这里的dagger是指square出品的原始 [dependency injector](https://github.com/square/dagger)。
到目前为止最新版本1.2.5。
>compile 'com.squareup.dagger:dagger:1.2.5'
compile 'com.squareup.dagger:dagger-compiler:1.2.5'


IDEA 需要Settings > Compiler > Annotation Processors > "Enable annotation processing"  否则不会自动生成需要的代码。


有时候可能只是用用@Injection 那么只要引入下面这个包即可。
>compile  'javax.inject:javax.inject:1'

https://mvnrepository.com/artifact/javax.inject/javax.inject/1   这个链接是maven的仓库，列举了不同构建工具的依赖指令。



有人利用java手动实现依赖注入
http://blog.evercoding.net/2016/04/12/implement-dependency-injection-by-annotation

 
# Dagger 使用例子

1. 新建工程
![](/imgs/Dagger-lerning/dagger1.jpg)

2. 引入dagger库
![](/imgs/Dagger-lerning/dagger2.jpg)

3. 简单例子
``` java
import dagger.Module;
import dagger.ObjectGraph;

import javax.inject.Inject;

/**
 * Created by Peter on 2016/9/2.
 */
public class Test2 {
    @Inject
    FloorLamp lamp;

    public static void main(String args[]) {
        System.out.println("Test2 ++");
        ObjectGraph o = ObjectGraph.create(Test2M.class);
        Test2 t = o.get(Test2.class);
        t.lamp.on();
        t.lamp.off();

    }

    @Module(injects = Test2.class)
    static class Test2M {
    }

    static class FloorLamp implements Lamp {
        @Inject
        FloorLamp() {
            System.out.println("Floor lamp instance");
        }
        public void on() {
            System.out.println("Floor lamp on");
        }

        public void off() {
            System.out.println("Floor lamp off");
        }
    }

}
```
在类Test2中没有直接实例化FloorLamp。 

- Test2总要有个地方实例化
- Test2依赖的FloorLamp也要实例化

**@Module(injects = Test2.class)** 指明了要注入的地方。**ObjectGraph o = ObjectGraph.create(Test2M.class);**执行之后，Dagger就知道了依赖关系。

**@Inject FloorLamp() **告诉Dagger用这个构造函数去实例化FloorLamp，实例化之后赋值个Test2实例的lamp。
这些事情发生在**Test2 t = o.get(Test2.class);** 语句中。当这个语句返回，FloorLamp和Test2就已经实例化完成并实现了注入。

# 链接

http://square.github.io/dagger/
