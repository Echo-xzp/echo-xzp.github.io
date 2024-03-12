---
title: Spring异步支持
tags:
  - Spring
  - Java
  - 异步
  - Async
  - CompletableFuture
categories: Java
abbrlink: 7e9e3df7
date: 2023-08-17 15:44:48
---

# 问题产生

博主在做项目时，要求远程调用其他服务的接口，为了不阻塞原有代码实现，同时由于只需调用服务而不关注他的返回值，于是博主自然而然的想到了用Java多线程来处理，*直接new一个Thread执行服务调用就够了*。

![5baaebe61babfbe07f5982312ec80a3](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/08/17_15_57_57_5baaebe61babfbe07f5982312ec80a3.png)

事实上运行起来也确实没问题，但是等我提交代码被review的时候，被提出不建议使用这种方法来处理，同时让我使用Spring的`@Async`注解来实现。虽然没说具体原因，但是我自己猜测还是性能上的问题吧。

>Spring 管理的线程池通常会进行性能优化，如线程的创建和销毁、线程的复用等。使用 new Thread 可能无法达到最优的性能效果。Spring 推荐使用其提供的异步编程机制，如 @Async 注解和 CompletableFuture，以及 Spring 管理的线程池来执行异步任务。这样可以确保线程的合理管理、上下文环境的正确传递、线程安全性的保证等。同时，这也有助于更好地集成到 Spring 框架的生态系统中，实现更高效、稳定和可维护的异步编程

其实还是随便`new`一个`Thread`出来执行任务，执行完之后就释放掉，在并发高的情况下容易造成线程浪费，并且可能还会引起`OOM`问题。为了解决这个问题还是最好使用Spring推荐的那几种~~异步编程方式~~。

# 解决方法

就按给我提的，用`@Async`注解实现吧。

`@Async` 是 Spring 框架提供的注解，用于实现异步方法调用。通过将该注解应用于方法上，Spring 将会将方法的执行封装为异步任务，使得方法可以在单独的线程中执行，而不会阻塞当前线程。这可以提高应用程序的并发性和性能。

以下是 `@Async` 注解的一些重要细节和用法：

1. **配置开启异步支持：** 在配置类上使用 `@EnableAsync` 注解，以启用 Spring 的异步支持。这将告诉 Spring 在执行带有 `@Async` 注解的方法时，将其放到异步任务中执行。

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;

@Configuration
@EnableAsync
public class AsyncConfig {
    // 配置其他异步相关的设置
}
```

2. **在方法上添加 `@Async` 注解：** 在需要异步执行的方法上标注 `@Async` 注解。这样，方法将会被异步地调用，而不会阻塞当前线程。

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class MyService {

    @Async
    public void asyncMethod() {
        // 异步执行的方法体
    }
}
```

3. **自定义异步线程池：** 默认情况下，Spring 使用一个基于线程池的执行器来处理异步任务。您也可以自定义异步线程池的配置，以适应不同的需求，例如线程池大小、队列容量等。Spring应用默认的线程池，指在`@Async`注解在使用时，不指定线程池的名称。查看源码，`@Async`的默认线程池为`SimpleAsyncTaskExecutor`，而它其实不是真的线程池，这个类不重用线程，默认每次调用都会创建一个新的线程。因此最好自己注入自定义的线程池bean，推荐使用`ThreadPoolTaskExecutor`。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        return executor;
    }
}
```

4. **返回 `Future` 或 `CompletableFuture`：** 如果需要异步方法返回结果，可以将方法的返回类型设置为 `Future` 或 `CompletableFuture`（推荐）。这样，调用方可以获取异步方法执行的结果。

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import java.util.concurrent.CompletableFuture;

@Service
public class MyService {

    @Async
    public CompletableFuture<String> asyncMethod() {
        // 异步执行的方法体
        String result = ...;
        return CompletableFuture.completedFuture(result);
    }
}
```

通过使用 `@Async` 注解，您可以实现应用程序中的异步执行，提高并发性能，降低响应时间，以及更好地利用系统资源。需要根据具体的业务需求和场景来选择使用异步方法。

# 总结

按照上面说的方法，其实`@Async`的本质也就是多线程，只不过使用了线程池，提高了线程使用的效率，我感觉从本质上来讲，它根本称不上是所谓的异步调用，本质上还是多线程开发，真正意义的多线程还得是`vert.x`，`webflux`这种基于事件循环和消息驱动的，不过虽然这么说，但是实际开发中用`@Async`这种方法确实也有它的可行之处。
