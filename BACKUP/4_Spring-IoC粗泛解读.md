# [Spring-IoC粗泛解读](https://github.com/Winniekun/article/issues/4)

# 序

## IoC

IoC控制反转，意思就是将创建对象的控制权从我们自己硬编码来new一个对象反转到了第三方身上。IoC的主要实现方式是依赖注入，Spring中的依赖注入方式有：`构造方法注入`、`settter注入`、`接口注入`。

- **控制反转**是一种思想
- **依赖注入**是一种设计模式

> IoC是一种可以帮助我们解耦各业务对象间依赖关系的对象绑定方式

## IoC Service Provider

虽然业务对象可以通过IoC方式声明相应的依赖，但是最终仍然需要通过某个角色或者服务将这些依赖的对象绑定到一起。而IoC Service Provider就对应IoC场景中的这一角色。IoC Service Provider在这里是一个抽象出来的概念，它可以指代任何将IoC场景中的业务对象绑定到一起的实现方式。它可以是一段代码，也可以是一组相关的类，甚至可以是比较通用的IoC框架或 者IoC容器实现。

### 职责

1. 业务对象的构建管理

   IoC中，业务对象无需关心所依赖注入的对象**如何构建如何取得**，但是这部分的工作还是需要有人来做的。所以`IoC Service Provider`需要将对象的构建逻辑从*客户端*对象那里抽离出来，一面这部分逻辑污染业务对象的实现。

   > **客户端**：代使用某个对象或者某种服务的对象。如果对象A需要引用对象B，那么A就是B的客户端对象，而不管A 处于Service层还是数据访问层。

2. 业务对象之间的依赖绑定

   最艰巨也是最重要的，这 是它的最终使命之所在。如果不能完成这个职责，那么，无论业务对象如何的“呼喊”，也不 会得到依赖对象的任何响应(最常见的倒是会收到一个NullPointerException)。`IoC Service Provider`通过结合之前构建和管理的所有业务对象，以及各个业务对象间可以识别的依赖关系，将这些对象所依赖的对象注入绑定，从而保证每个业务对象在使用的时候，可以处于就绪状 态。

## Spring IoC容器

![image-20210520140010820](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20210520140010820.png)

Spring的IoC容器就是一个IoC Provider，但是其相比于IoC Provider的功能更加的丰富，除了业务对象的构建、依赖关系的绑定这种IoC Provider的基本功能，还新增了业务对象声明周期的管理，AOP支持等。同时Spring提供了两种容器类型:BeanFactory和ApplicationContext。

- BeanFactory

  基础类型IoC容器，提供完整的IoC服务支持。如果没有特殊指定，==默认采用延迟初始化策略(lazy-load)==。只有当客户端对象需要访问容器中的某个受管对象的时候，才对该受管对象进行初始化以及依赖注入操作。所以，相对来说，容器启动初期速度较快，所需要的资源有限。对于资源有限，并且功能要求不是很严格的场景，BeanFactory是比较合适的 IoC容器选择。

- ApplicationContext

  在BeanFactory的基础上构建，是相对比较高 级的容器实现，除了拥有BeanFactory的所有支持，ApplicationContext还提供了其他高级特性，比如事件发布、国际化信息支持等。ApplicationContext所管理 的对象，在该类型容器启动之后，==默认全部初始化并绑定完成==。所以，相对于BeanFactory来 说，ApplicationContext要求更多的系统资源，同时，因为在启动时就完成所有初始化，容器启动时间较之BeanFactory也会长一些。在那些系统资源充足，并且要求更多功能的场景中， ApplicationContext类型的容器是比较合适的选择。

### 简单的式例



### Spring IoC容器的一些特性简介

#### autowire

自动注入，通过autowire，我们可以略去很多人工显示的配置bean的操作，被依赖的发现和注入都交给了Spring，同时其有几个类型可选择，比如byName、byType等。

#### FactoryBean

BeanFactory和FactoryBean傻傻分不清？虽然两者类似，但是功能不同。FactoryBean是一种工厂Bean，和普通的Bean不一样，FactoryBean是可以生产Bean的Bean。

#### depends-on

当一个 bean 直接依赖另一个 bean，可以使用 `<ref/>` 标签进行配置。不过如某个 bean 并不直接依赖于其他 bean，但又需要其他 bean 先实例化好，这个时候就需要使用 depends-on 特性了。depends-on 特性比较简单，就不演示了。仅贴一下配置文件的内容，如下：

这里有两个简单的类，其中 Hello 需要 World 在其之前完成实例化。相关配置如下：

```xml
<bean id="hello" class="xyz.coolblog.depnedson.Hello" depends-on="world"/>
<bean id="world" class="xyz.coolblog.depnedson.World" />
```

#### BeanPostProcessor

BeanPostProcessor 是 bean 实例化时的后置处理器，包含两个方法，其源码如下：

```java
public interface BeanPostProcessor {
    // bean 初始化前的回调方法
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    // bean 初始化后的回调方法    
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```

BeanPostProcessor是Spring框架的一个扩展点，可以通过BeanPostProcessor接口，我们能够插手Bean的实例化过程，比如AOP就是在bean实例将切面逻辑织入bean实例中，AOP也是正是通过BeanPostProcessor和IoC容器建立了联系

##### 示例

```java
@Slf4j
public class LoggerBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        log.info("Before  " + beanName + " Initialization");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        log.info("After  " + beanName + " Initialization");
        return bean;
    }
}


```

##### **配置如下**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean class="com.titizz.simulation.toyspring.learninng.LoggerBeanPostProcessor"/>

    <bean id="car" class="com.titizz.simulation.toyspring.Car">

    </bean>
</beans>
```

##### **测试如下**

```java
@Test
public void testBeanPostProcessor() {
    String xmlPath = "test-spring.xml";
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext(xmlPath);
}
```

![image-20210502181644978](https://i.loli.net/2021/05/02/iUgIOGvfmPo1ws5.png)

### 核心接口

#### Resource

以 `Resource` 接口为核心发散出的几个类，都是用于解决 IoC 容器中的内容从哪里来的问题，也就是 **配置文件从哪里读取、配置文件如何读取** 的问题。

| 类名             | 说明                                                         |
| :--------------- | :----------------------------------------------------------- |
| `Resource`       | 接口，标识一个外部资源。通过 `getInputStream()` 方法 **获取资源的输入流** 。 |
| `UrlResource`    | 实现 `Resource` 接口的资源类，通过 URL 获取资源。            |
| `ResourceLoader` | 资源加载类。通过 `getResource(String)` 方法获取一个 `Resouce` 对象，是 **获取 `Resouce` 的主要途径** 。 |

#### BeanDefinition

**功能：**用于解决Bean的定义问题，包括Bean的名字，类型以及它的属性赋予了什么值或者引用。说白了也就是**解决了在IoC容器中定义一个Bean，使得IoC容器可以根据这个定义来生成实例。**

| 类名                           | 说明                                                         |
| :----------------------------- | :----------------------------------------------------------- |
| `BeanDefinition`               | 该类保存了 `Bean` 定义。包括 `Bean` 的 **名字** `String beanClassName`、**类型** `Class beanClass`、**属性** `PropertyValues propertyValues`。根据其 **类型** 可以生成一个类实例，然后可以把 **属性** 注入进去。`propertyValues` 里面包含了一个个 `PropertyValue` 条目，每个条目都是键值对 `String` - `Object`，分别对应要生成实例的属性的名字与类型。在 Spring 的 XML 中的 `property` 中，键是 `key` ，值是 `value` 或者 `ref`。对于 `value` 只要直接注入属性就行了，但是 `ref` 要先进行解析。`Object` 如果是 `BeanReference` 类型，则说明其是一个引用，其中保存了引用的名字，需要用先进行解析，转化为对应的实际 `Object`。 |
| `BeanDefinitionReader`         | 解析 `BeanDefinition` 的接口。通过 `loadBeanDefinitions(String)` 来从一个地址加载类定义。 |
| `AbstractBeanDefinitionReader` | 实现 `BeanDefinitionReader` 接口的抽象类（未具体实现 `loadBeanDefinitions`，而是规范了 `BeanDefinitionReader` 的基本结构）。内置一个 `HashMap rigistry`，用于保存 `String` - `beanDefinition` 的键值对。内置一个 `ResourceLoader resourceLoader`，用于保存类加载器。用意在于，使用时，只需要向其 `loadBeanDefinitions()` 传入一个资源地址，就可以自动调用其类加载器，并把解析到的 `BeanDefinition` 保存到 `registry` 中去。 |
| `XmlBeanDefinitionReader`      | 具体实现了 `loadBeanDefinitions()` 方法，从 XML 文件中读取类定义。 |

#### BeanFactory :question:

以 `BeanFactory` 接口为核心发散出的几个类，都是用于解决 IoC 容器在 **已经获取 `Bean` 的定义的情况下，如何装配、获取 `Bean` 实例** 

的问题。

| 类名                         | 说明                                                         |
| :--------------------------- | :----------------------------------------------------------- |
| `BeanFactory`                | 接口，标识一个 IoC 容器。通过 `getBean(String)` 方法来 **获取一个对象** |
| `AbstractBeanFactory`        | `BeanFactory` 的一种抽象类实现，规范了 IoC 容器的基本结构，但是把生成 `Bean` 的具体实现方式留给子类实现。IoC 容器的结构：`AbstractBeanFactory` 维护一个 `beanDefinitionMap` 哈希表用于保存类的定义信息（`BeanDefinition`）。获取 `Bean` 时，如果 `Bean` 已经存在于容器中，则返回之，否则则调用 `doCreateBean` 方法装配一个 `Bean`。（所谓存在于容器中，是指容器可以通过 `beanDefinitionMap` 获取 `BeanDefinition` 进而通过其 `getBean()` 方法获取 `Bean`。） |
| `AutowireCapableBeanFactory` | 可以实现自动装配的 `BeanFactory`。在这个工厂中，实现了 `doCreateBean` 方法，该方法分三步：1，通过 `BeanDefinition` 中保存的类信息实例化一个对象；2，把对象保存在 `BeanDefinition` 中，以备下次获取；3，为其装配属性。装配属性时，通过 `BeanDefinition` 中维护的 `PropertyValues` 集合类，把 `String` - `Value` 键值对注入到 `Bean` 的属性中去。如果 `Value` 的类型是 `BeanReference` 则说明其是一个引用（对应于 XML 中的 `ref`），通过 `getBean` 对其进行获取，然后注入到属性中。 |

#### ApplicationContext

以 `ApplicationContext` 接口为核心发散出的几个类，主要是对前面 `Resouce` 、 `BeanFactory`、`BeanDefinition` 进行了功能的封装，解决 **根据地址获取 IoC 容器并使用** 的问题。

| 类名                           | 说明                                                         |
| :----------------------------- | :----------------------------------------------------------- |
| `ApplicationContext`           | 标记接口，继承了 `BeanFactory`。通常，要实现一个 IoC 容器时，需要先通过 `ResourceLoader` 获取一个 `Resource`，其中包括了容器的配置、`Bean` 的定义信息。接着，使用 `BeanDefinitionReader` 读取该 `Resource` 中的 `BeanDefinition` 信息。最后，把 `BeanDefinition` 保存在 `BeanFactory` 中，容器配置完毕可以使用。注意到 `BeanFactory` 只实现了 `Bean` 的 **装配、获取**，并未说明 `Bean` 的 **来源** 也就是 `BeanDefinition` 是如何 **加载** 的。该接口把 `BeanFactory` 和 `BeanDefinitionReader` 结合在了一起。 |
| `AbstractApplicationContext`   | `ApplicationContext` 的抽象实现，内部包含一个 `BeanFactory` 类。主要方法有 `getBean()` 和 `refresh()` 方法。`getBean()` 直接调用了内置 `BeanFactory` 的 `getBean()` 方法，`refresh()` 则用于实现 `BeanFactory` 的刷新，也就是告诉 `BeanFactory` 该使用哪个资源（`Resource`）加载类定义（`BeanDefinition`）信息，该方法留给子类实现，用以实现 **从不同来源的不同类型的资源加载类定义** 的效果。 |
| ClassPathXmlApplicationContext | 从类路径加载资源的具体实现类。内部通过 `XmlBeanDefinitionReader` 解析 `UrlResourceLoader` 读取到的 `Resource`，获取 `BeanDefinition` 信息，然后将其保存到内置的 `BeanFactory` 中。 |

> 对 `ApplicatinoContext` 的分层更为细致。`AbstractApplicationContext` 中为了实现 **不同来源** 的 **不同类型** 的资源加载类定义，把这两步分层实现。以“从类路径读取 XML 定义”为例，首先使用 `AbstractXmlApplicationContext` 来实现 **不同类型** 的资源解析，接着，通过 `ClassPathXmlApplicationContext` 来实现 **不同来源** 的资源解析。



# 战略观望

<img src="https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20210520110056368.png" alt="官网示例" style="zoom:50%;" />



![Spring容器功能实现的各个阶段](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20210520105808650.png)

## 容器启动阶段

使用Resource相关接口标识我们配置文件从哪里获取，然后使用BeanDefinitionReader对标识的文件进行解析，将分析后的信息编组为相应的BeanDefinition。最后将保存了每个Bean定义信息的BeanDefinition，注册到相应的BeanDefinitionRegistry，这样容器的启动就完成了。

![启动阶段任务](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20210520110900263.png)

## Bean实例化阶段

通过容器的启动阶段，现在所有的Bean定义的信息，都通过BeanDefiniton的方式注册到了BeanDefinitionRegistry中，当某个请求方通过容器的getBean方法明确的调用某个对象的时候，或者因为依赖关系容器需要隐式的调用getBean方法，就会出发第二阶段的活动。

![bean的实例化过程](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20210520150106541.png)

Spring容器将对其所管理的对象全部给予统一的生命周期管理，这些被管理的对象完全摆脱了原 来那种“new完后被使用，脱离作用域后即被回收”的命运

# 详细阐述

## 最简单的IoC容器

```java
public class BeanFactory {
  // 说白了就是一个map
	private Map<String, Object> beanMap = new HashMap<>();

	public void registerBean(String name, Object bean) {
		beanMap.put(name, bean);
	}

	public Object getBean(String name) {
		return beanMap.get(name);
	}
}

public class SimpleBeanContainerTest {
	@Test
	public void testGetBean() throws Exception {
    // 需要我们去手动new依赖对象，然后放入
		BeanFactory beanFactory = new BeanFactory();
		beanFactory.registerBean("helloService", new HelloService());
		HelloService helloService = (HelloService) beanFactory.getBean("helloService");
		assertThat(helloService).isNotNull();
		assertThat(helloService.sayHello()).isEqualTo("hello");
	}
	class HelloService {
		public String sayHello() {
			System.out.println("hello");
			return "hello";
		}
	}
}
```

## BeanFactoryPostProcess和BeanPostProcessor

BeanFactoryPostProcess和BeanPostProcessor是spring框架中具有重量级地位的两个接口，理解了这两个接口的作用，基本就理解spring的核心原理了

1. BeanFactoryPostProcessor是spring提供的容器拓展机制，允许我们在**bean实例化之前修改bean的定义信息（BeanDedinition）**，其重要的实现类PropertyPlaceholderConfigurer和CustomEditorConfigurer
   - PropertyPlaceholderConfigurer的作用是用properties文件的配置值替换xml文件中的占位符
   - CustomEditorConfigurer的作用是实现类型转换。
2. BeanPostProcessor也是spring提供的容器拓展机制，不同于BeanFactoryPostProcessor，BeanPostProcesssor在**Bean实例化之后修改Bean或替换Bean**。也是实现AOP的关键。

等等等。。。



## 整体流程图

![整体流程图](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/iShot2021-05-21%2000.08.58.png)

1. BeanDefinitionReader读取Bean的相关配置信息，并将读取到的信息使用BeanDefinition表示，同时注册到BeanDefinitionRegistry中。
2. 通过实现了BeanFactoryPostProcessor的类，自定义修改BeanDefinition中的信息
3. Bean的实例化
   1. 采用策略模式，策略化Bean的实例，共包含两种方式：cglib、反射
   2. 获取Bean的实例之后，根据BeanDefinition中的信息，填充Bean的属性、依赖
4. 检测各种检测各种Aware接口，并放入Bean中
5. 调用BeanPostProcessor接口的前置处理方法，处理符合要求的Bean实例
6. 如果实现了InitializingBean接口，则处理对应的afterPropertiesSet()方法
7. 如果定义init-method方法，处理对应的初始化方法
8. 调用BeanPostProcessor接口的后置处理方法，处理符合要求的Bean实例
9. 使用
10. 判断Bean的Scope，如果是prototype，不再管理
11. 如果是单例，如果实现了DisposableBean接口，则执行对应的destroy方法
12. 如果定义了destory-method，则执行自定义的销毁方法



# 结束

整理的还是很乱，虽然看的时候是都看了。。。

后面慢慢优化吧（涉及到以后==永远不会）:expressionless:



# references
* spring揭秘
* [tiny-spring](https://github.com/code4craft/tiny-spring)
* [mini-spring](https://github.com/DerekYRC/mini-spring/blob/main/changelog.md#Bean%E5%AE%9E%E4%BE%8B%E5%8C%96%E7%AD%96%E7%95%A5InstantiationStrategy)
* [tiny-spring分析](https://www.zybuluo.com/dugu9sword/note/382745#beanfactory)
* [Spring IoC容器分析](https://javadoop.com/post/spring-ioc)
* [Spring IOC 容器源码分析系列](http://www.tianxiaobo.com/2018/05/30/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%AF%BC%E8%AF%BB/#451-%E5%AE%9E%E7%8E%B0-applicationcontextaware-%E6%8E%A5%E5%8F%A3)