---
title: nacos面试总结 其二
date: 2023-09-17 09:39:00
tags: [Spring,Spring Cloud Alibaba,Nacos,nacos,面试]
categories: Java
---

# 为什么要有配置中心？

配置中心是分布式系统中的一个重要组件，它的存在有多个关键原因和优势：

1. **集中化配置管理：** 配置中心提供了一种集中化管理应用程序配置的方式。它允许您将所有配置信息集中存储在一个地方，而不是散布在各个应用程序中。这样可以更轻松地管理配置，确保一致性和可维护性。

2. **动态配置更新：** 配置中心允许在运行时动态更新应用程序的配置，而无需重新部署或重启应用程序。这意味着您可以在不中断服务的情况下修改配置，快速适应不同的环境和需求。

3. **版本管理：** 配置中心通常支持配置版本管理，允许您跟踪和恢复以前的配置版本。这对于诊断问题、回滚配置更改以及进行审计非常有用。

4. **安全性和访问控制：** 配置中心可以提供访问控制和安全性，以确保只有授权的用户或应用程序可以访问敏感配置信息。这有助于保护敏感数据和配置。

5. **跨环境配置：** 配置中心允许为不同的环境（例如开发、测试、生产）和不同的部署场景（例如云环境、本地环境）提供不同的配置。这可以确保应用程序在不同环境中具有正确的配置。

6. **分布式应用程序支持：** 对于分布式应用程序，配置管理变得更加复杂。配置中心可以提供一种统一的方式来管理分布式系统的配置，确保不同部分的应用程序使用相同的配置信息。

7. **易于维护和变更：** 通过将配置从应用程序代码中分离出来，配置中心使得配置更容易维护和更改。开发人员可以专注于应用程序逻辑，而不必担心配置的细节。

8. **减少硬编码：** 配置中心减少了硬编码配置的需要。相反，应用程序可以从配置中心获取所需的配置信息，从而降低了代码的耦合性。

总之，配置中心的存在有助于简化配置管理、提高配置的可维护性和灵活性，支持动态更新和版本控制，同时也增强了应用程序的安全性和可管理性。对于大型和分布式系统来说，配置中心是不可或缺的工具，有助于降低维护成本并提供更好的系统可靠性和可伸缩性。

# Nacos做配置中心时是AP还是CP？

`Nacos`在作为配置中心时，通常是CP（一致性和分区容错）的系统。CP系统强调在网络分区或节点故障的情况下，保持一致性。这对于配置中心非常重要，因为配置数据的一致性是系统的关键要素，而配置变更需要被正确地传播到所有相关的服务实例中。

`Nacos`使用了Raft一致性协议来确保配置数据的一致性和可靠性。Raft是一种分布式一致性算法，能够在节点故障或网络分区的情况下保持数据的一致性。这意味着无论发生了什么故障或中断，`Nacos`都会努力保持配置数据的一致性，并确保配置更改得到正确的传播。

总之，`Nacos`作为配置中心时倾向于CP系统，它注重配置数据的一致性和可靠性，以确保系统在面对故障或分区时仍然能够提供可靠的配置管理服务。这是配置中心的关键特性，因为配置数据对于应用程序的正确运行至关重要。

# Raft和Distro有何区别？

Raft 和 Distro 是两种不同的分布式一致性协议，它们解决了不同类型的分布式系统问题，有以下区别：

1. **一致性问题：**
   - Raft：Raft 协议主要用于解决分布式系统中的一致性问题，例如在复制日志系统中确保多个副本的一致性。Raft 强调领导者选举和日志复制，以确保数据的一致性。
   - Distro：Distro（或称为Distributed Consistency Protocol）是用于解决分布式数据存储系统中的数据一致性问题的协议。它确保分布式系统中的数据副本保持一致性，以提供高可用性和容错性。

2. **应用领域：**
   - Raft：Raft 协议通常用于分布式存储系统，如分布式数据库或分布式日志系统，以确保数据一致性和可用性。
   - Distro：Distro 协议主要用于分布式数据存储系统，例如键值存储、分布式文件系统等，以确保数据一致性和高可用性。

3. **设计目标：**
   - Raft：Raft 协议的设计目标是提供相对简单和容易理解的一致性算法，以便开发者更容易实现和维护分布式系统。
   - Distro：Distro 协议的设计目标是在复杂的分布式数据存储系统中提供高一致性、高可用性和高性能的数据复制和分发。

4. **性能和复杂性：**
   - Raft：Raft 协议相对较简单，容易理解和实现，但在一些性能和吞吐量方面可能会有一些限制。
   - Distro：Distro 协议通常更复杂，因为它需要处理更多的数据分发和一致性保证，但可以提供更高的性能和可用性。

总的来说，Raft 和 Distro 都是用于分布式系统的一致性协议，但它们关注的问题领域和设计目标不同。Raft 更适合解决一致性问题，而 Distro 更适合解决分布式数据存储系统的数据一致性和高可用性问题。选择使用哪个协议取决于您的系统需求和复杂性。

# Nacos作为配置中心宕机之后，是不是意味着依赖这个配置中心的服务也读取不了配置进而下线呢？

Nacos作为配置中心宕机不会立即导致依赖该配置中心的服务下线，但它可能会对这些服务的运行产生一些影响，具体取决于服务的配置和设计。

当Nacos配置中心宕机时，可能发生以下情况：

1. **已加载的配置数据仍然可用：** 如果服务已经从Nacos配置中心加载了配置数据，并将其缓存在本地，那么即使Nacos宕机，服务仍然可以使用本地缓存的配置数据。这意味着已加载的配置在一定时间内仍然可用。

2. **新配置无法获取：** 如果服务需要获取新的配置或在Nacos宕机后尝试刷新配置，那么它将无法从Nacos获取新的配置。这可能会影响服务对于动态配置更改的感知。

3. **服务启动和注册问题：** 如果服务依赖Nacos进行服务注册和发现，Nacos宕机可能会影响新服务的注册和发现。现有的服务可能会继续运行，但新的服务可能无法注册，因为它们无法找到Nacos实例。

4. **服务的健康检查和自我修复：** 如果服务实现了自我修复机制，它们可能会尝试重新连接Nacos或在它恢复运行时重新注册自己。这取决于服务的设计和实现。

总的来说，Nacos配置中心的宕机通常不会立即导致依赖它的服务下线，但可能会对新配置的获取、服务注册和发现以及配置更新等方面产生影响。为了提高系统的可靠性，可以考虑在设计中采用适当的容错策略和缓存机制，以应对Nacos配置中心的宕机情况。此外，定期监控Nacos的健康状态并实施恢复措施也是重要的。

# 服务会主动更新已经从配置中心加载了的配置吗？

一般情况下，服务不会主动更新已经从配置中心加载的配置。服务在启动时会从配置中心加载配置数据，然后使用这些数据来配置自身的行为。一旦配置数据被加载到服务中，它通常不会自动更新。

如果您希望服务能够获取最新的配置数据并动态更新自身的行为，您需要实现配置的热加载或定期轮询配置中心以检查是否有新的配置变更。这可以通过以下方式来实现：

1. **定期轮询：** 在服务中实现一个定时任务或线程，定期向配置中心查询是否有新的配置变更。如果检测到变更，服务可以重新加载配置并应用新的配置。

2. **使用配置中心的通知机制：** 许多配置中心提供了通知机制，允许服务订阅配置变更通知。当配置发生变化时，配置中心可以通知已注册的服务。服务在接收到通知后，可以主动获取新的配置并更新自身。

3. **使用回调机制：** 部分配置中心支持回调函数，允许您在配置发生变化时执行特定的回调操作。您可以将回调函数与配置绑定，以便在配置更改时触发相应的操作。

不同的配置中心提供不同的机制来支持配置的动态更新，具体取决于您使用的配置中心的特性和功能。无论采用哪种方法，都需要确保服务能够处理配置变更，并在需要时重新加载和应用新的配置，以实现配置的动态更新。这对于实现灵活性和可维护性的分布式系统非常重要。

# 针对Nacos,如何实现动态加载配置？

Nacos可以通过以下几种方式实现动态加载配置：

1. **监听配置变更：** Nacos提供了监听配置变更的机制。您可以注册一个监听器，当配置发生变化时，Nacos将通知您的应用程序。在Java客户端中，您可以使用Nacos SDK中的`Listener`接口来实现配置监听。以下是一个简单的示例：

```java
import com.alibaba.nacos.api.config.listener.Listener;
import com.alibaba.nacos.api.config.ConfigService;

public class ConfigListenerExample {
    public static void main(String[] args) throws Exception {
        String dataId = "your-data-id";
        String group = "your-group";
        String namespace = "your-namespace";
        
        ConfigService configService = NacosFactory.createConfigService(properties);
        
        configService.addListener(dataId, group, new Listener() {
            @Override
            public void receiveConfigInfo(String configInfo) {
                // 配置发生变化时的处理逻辑
                System.out.println("Received updated config: " + configInfo);
            }
        });
        
        // 保持应用程序运行，以便继续监听配置变更
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

2. **定时轮询：** 如果不使用监听机制，您还可以定期轮询Nacos以检查配置是否发生了变化。这是一种简单的方式，但效率可能较低，因为它需要定期发送请求来检查配置。

3. **使用Spring Cloud的@RefreshScope：** 如果您正在使用Spring Cloud，并且已将Nacos集成到Spring Cloud Config中，您可以使用`@RefreshScope`注解来标记需要动态刷新的bean。当配置发生变化时，通过调用Actuator的`/actuator/refresh`端点，可以刷新被`@RefreshScope`注解标记的bean。这将导致Spring重新加载配置并应用到相应的bean。

以上是几种常见的方式来实现Nacos配置的动态加载。您可以根据您的应用程序需求和技术栈选择最合适的方法。要注意，动态加载配置通常需要您的应用程序具备处理配置变更的逻辑，以确保新配置的正确应用。

# 如果此时动态加载配置，而配置中心此时是下线状态，会加载到空配置还是继续使用原来的已加载配置？

当配置中心（例如Nacos）处于下线状态时，动态加载配置的行为取决于具体的实现和配置。通常情况下，如果配置中心不可用，已加载的配置将保持不变，而不会被动态更新。

以下是不同情况下的可能行为：

1. **已加载配置的保留：** 大多数配置加载库和框架会保留已加载的配置，即使配置中心不可用。这意味着如果配置中心不可用，已加载的配置将继续被使用，而不会被替换为空配置。

2. **失败安全机制：** 一些配置加载库会实现失败安全机制，即使配置中心不可用，它们也会尝试保留已加载的配置，并在配置中心恢复可用时重新尝试获取新配置并应用。这可以提供一定程度的容错性。

3. **配置缓存：** 部分配置加载库和框架支持本地配置缓存，即使配置中心不可用，它们也可以从本地缓存中获取配置。这可以确保即使配置中心不可用，应用程序也可以继续使用缓存的配置。

请注意，上述行为可能会因使用的配置加载库或框架而有所不同。在使用特定的配置加载库时，建议查阅该库的文档，了解在配置中心不可用的情况下它的行为是如何的。

为了提高系统的可靠性，通常建议在配置中心恢复可用之前继续使用已加载的配置，以确保应用程序的正常运行。一旦配置中心再次可用，系统可以尝试获取最新的配置并动态更新。这种方式可以避免配置中心的临时故障对应用程序的影响。

# Spring Cloud中配置文件加载顺序是怎么样的？

1、开发者工具 `Devtools` 全局配置参数；

2、单元测试上的 `@TestPropertySource` 注解指定的参数；

3、单元测试上的 `@SpringBootTest` 注解指定的参数；

4、命令行指定的参数，如 `java -jar springboot.jar --name="Java技术栈"`；

5、命令行中的 `SPRING_APPLICATION_JSONJSON` 指定参数, 如 `java -Dspring.application.json='{"name":"Java技术栈"}' -jar springboot.jar`

6、`ServletConfig` 初始化参数；

7、`ServletContext` 初始化参数；

8、JNDI参数（如 `java:comp/env/spring.application.json`）；

9、Java系统参数（来源：`System.getProperties()`）；

10、操作系统环境变量参数；

11、`RandomValuePropertySource` 随机数，仅匹配：`ramdom.*`；

12、JAR包外面的配置文件参数（`application-{profile}.properties（YAML）`）

13、JAR包里面的配置文件参数（`application-{profile}.properties（YAML）`）

14、JAR包外面的配置文件参数（`application.properties（YAML）`）

15、JAR包里面的配置文件参数（`application.properties（YAML）`）

16、`@Configuration`配置文件上 `@PropertySource` 注解加载的参数；

17、默认参数（通过 `SpringApplication.setDefaultProperties` 指定）；

***在Nacos中：***

```
1. bootstrap.yaml
2. bootstrap.properties
3. bootstrap-{profile}.yaml
4. bootstrap-{profile}.properties
5. application.yaml
6. application.properties
7. application-{profile}.yaml
8. application-{profile}.properties
9. nacos配置中心共享配置（通过spring.cloud.nacos.config.shared-configs指定）
10. Nacos配置中心该服务配置（通过spring.cloud.nacos.config.prefix和spring.cloud.nacos.config.file-extension指定）
11. Nacos配置中心该服务-{profile}配置（通过spring.cloud.nacos.config.prefix和spring.cloud.nacos.config.file-extension、以及spring.profiles.active指定）

因此，配置生效覆盖关系：

对于key不同，则直接生效；

对于key相同的同名配置项，后加载会覆盖掉前加载，故而最终为后加载的配置项生效！
```

同时可以配置：

```yaml
spring:
  cloud:
    config:
      override-none: true
      allow-override: true
      override-system-properties: false  
```

开启本地配置优先

注意：**一定要在 nacos上配置不然不生效**

# 配置中心的地址不配置在`application.yaml`里面？

配置中心的地址通常是配置在`bootstrap.properties`（或`bootstrap.yml`）中的，而不是`application.properties`（或`application.yml`）中。这是因为`bootstrap`配置文件在Spring Boot应用程序启动时最先加载，它用于配置应用程序上下文之前的一些基础设置，包括与配置中心的连接信息。

比如数据库的配置配置在配置中心里面，如果把配置中心地址配置在`application.yaml`里面，当Spring加载到此处时，没有获取到注入`DataSource`所需的一些依赖，就会直接报错了。

# bootstrap和application配置文件之间有何区别？

在Spring Boot应用程序中，`bootstrap.properties`（或`bootstrap.yml`）和`application.properties`（或`application.yml`）是两个不同的配置文件，它们之间有以下区别：

1. **加载顺序：**
   - `bootstrap`配置文件在Spring Boot应用程序启动时最先加载。它用于配置Spring Boot的应用上下文之前的一些基础设置，通常包括与配置中心的连接和一些系统属性的设置。
   - `application`配置文件是主要的应用程序配置文件，用于配置应用程序的各个组件，例如数据库连接、日志级别、Web服务器端口等。它在`bootstrap`配置文件加载后加载，但在应用程序上下文创建之前。

2. **用途：**
   - `bootstrap`配置文件通常用于配置与应用程序启动相关的基础设置，例如配置中心的连接信息、加密算法、自定义属性解析等。它们在应用程序上下文创建之前加载，因此可以用于配置Spring Cloud Config Server、Nacos等配置中心的连接参数。
   - `application`配置文件用于配置应用程序的具体功能和组件。它包含了应用程序的业务逻辑所需的配置信息，例如数据库连接信息、缓存配置、日志设置等。

3. **覆盖关系：**
   - `bootstrap`配置文件中的属性可以覆盖`application`配置文件中的同名属性。这意味着如果在`bootstrap`配置文件和`application`配置文件中都定义了相同的属性，那么`bootstrap`配置文件中的属性将具有更高的优先级。
   - 通常，`bootstrap`配置文件用于配置一些全局的、与环境无关的基础设置，而`application`配置文件用于配置应用程序的具体行为。

4. **使用场景：**
   - `bootstrap`配置文件适用于配置与应用程序启动相关的内容，以及在应用程序上下文之前需要解析的配置。
   - `application`配置文件适用于配置应用程序的具体功能和组件，包括数据库、消息队列、Web服务器等。

总之，`bootstrap`配置文件和`application`配置文件在Spring Boot应用程序中具有不同的加载顺序和用途。了解它们之间的区别有助于您更好地组织和管理应用程序的配置。

# Nacos集群架构

![img](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/09/17_10_38_32_af431fa073fd560b873b2d261e9597a1.png)
