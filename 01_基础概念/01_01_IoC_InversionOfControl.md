
https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/beans.html

https://www.baeldung.com/inversion-control-and-dependency-injection-in-spring

> IoC enables a framework to take control of the flow of a program and make calls to our custom code

# 1 What Is Inversion of Control

[Inversion of Control](https://www.baeldung.com/cs/ioc) is a principle in software engineering which transfers the control of objects or portions of a program to a container or framework. We most often use it in the context of object-oriented programming.

In contrast with traditional programming, in which our custom code makes calls to a library, ==IoC enables a framework to take control of the flow of a program and make calls to our custom code==. To enable this, frameworks use abstractions with additional behavior built in. ==**If we want to add our own behavior, we need to extend the classes of the framework or plugin our own classes.**==

The advantages of this architecture are:
- decoupling the execution of a task from its implementation
- making it easier to switch between different implementations
- greater modularity of a program
- greater ease in testing a program by isolating a component or mocking its dependencies, and allowing components to communicate through contracts

具体能够实现 Inversion of Control  的 mechanisms 
We can achieve Inversion of Control through various mechanisms such as: Strategy design pattern, Service Locator pattern, Factory pattern, and Dependency Injection (DI).

We’re going to look at DI next.


# 2 What Is Dependency Injection

一个control 被植入 通过设立object’s dependencies
control 是干什么的: 添加 control,  add our own behavior add tp framework， 使得 这个 framework 可以  ake control of the flow of a program and make calls to our custom code

Dependency injection is a pattern we can use to implement IoC, where ==the control being inverted is setting an object’s dependencies.==

Connecting objects with other objects, or “injecting” objects into other objects, is done by an assembler rather than by the objects themselves.

Here’s how we would create an object dependency in traditional programming:

```java
public class Store {
    private Item item;
 
    public Store() {
        item = new ItemImpl1();    
    }
}
```

In the example above, we need to instantiate an implementation of the _Item_ interface within the _Store_ class itself.

By using DI, we can rewrite the example without specifying the implementation of the _Item_ that we want:
```java
public class Store {
    private Item item;
    public Store(Item item) {
        this.item = item;
    }
}
```

In the next sections, we’ll look at how we can provide the implementation of _Item_ through metadata.

Both IoC and DI are simple concepts, but they have deep implications in the way we structure our systems, so they’re well worth understanding fully.


# 3 Introduction to the Spring IoC container and beans

This chapter covers the Spring Framework implementation of the Inversion of Control (IoC) [[1]](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/beans.html#ftn.d5e1144)principle. IoC is also known as _dependency injection_ (DI). It is a process whereby objects define their dependencies, that is, the other objects they work with, only through constructor arguments, arguments to a factory method, or properties that are set on the object instance after it is constructed or returned from a factory method. The container then _injects_ those dependencies when it creates the bean. This process is fundamentally the inverse, hence the name _Inversion of Control_ (IoC), of the bean itself controlling the instantiation or location of its dependencies by using direct construction of classes, or a mechanism such as the _Service Locator_ pattern.

The `org.springframework.beans` and `org.springframework.context` packages are the basis for Spring Framework's IoC container. The `[BeanFactory](http://static.springsource.org/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/BeanFactory.html)` interface provides an advanced configuration mechanism capable of managing any type of object. `[ApplicationContext](http://static.springsource.org/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationContext.html)` is a sub-interface of `BeanFactory.` It adds easier integration with Spring's AOP features; message resource handling (for use in internationalization), event publication; and application-layer specific contexts such as the `WebApplicationContext` for use in web applications.

In short, the `BeanFactory` provides the configuration framework and basic functionality, and the `ApplicationContext` adds more enterprise-specific functionality. The `ApplicationContext` is a complete superset of the `BeanFactory`, and is used exclusively in this chapter in descriptions of Spring's IoC container. For more information on using the `BeanFactory` instead of the `ApplicationContext,` refer to [Section 5.15, “The BeanFactory”](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/beans.html#beans-beanfactory "5.15 The BeanFactory").

In Spring, the objects that form the backbone of your application and that are managed by the Spring IoC _container_ are called _beans_. A bean is an object that is instantiated, assembled, and otherwise managed by a Spring IoC container. Otherwise, a bean is simply one of many objects in your application. Beans, and the _dependencies_ among them, are reflected in the _configuration metadata_ used by a container.








