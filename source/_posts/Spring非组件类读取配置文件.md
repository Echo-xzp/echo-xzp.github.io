---
title: Spring非组件类读取配置文件
tags:
  - Java
  - Spring
categories: Java
abbrlink: cc267d2
date: 2023-08-16 15:28:13
---

# 需求分析

博主在做的项目的时候要求通过消息模板（实体类）来发送消息，消息模板有唯一的模板id，第三方接口正是通过这个模板id来判断模板类型并实现消息发送的。

退单消息模板：

```java
@Data
public class WechatRefundTemplate implements WechatInfoTemplate {

    private final String id = "退单消息模板id";

    // 下单人姓名以及手机号
    @JSONField(name = "thing2")
    String nameAndPhone;

    // 线路名
    @JSONField(name = "thing1")
    String lineName;

    // 预期时间
    @JSONField(name = "time3")
    String planTime;

    // 上车点名称
    @JSONField(name = "thing6")
    String upStationName;

    // 退票金额
    @JSONField(name = "character_string5")
    String refund;
    
    @Override
    public String getId(){
	    return id;
    }
}
```

下单消息模板：

```java
@Data
public class WechatOrderTemplate implements WechatInfoTemplate{

    private final String id = "下单消息模板id";

    // 下单手机号
    @JSONField(name = "phone_number10")
    String phoneNumber;

    // 乘客数
    @JSONField(name = "number4")
    Integer passageNum;

    // 预期时间
    @JSONField(name = "time3")
    String planTime;

    // 上车点名称
    @JSONField(name = "thing7")
    String upStationName;

    // 下车点名称
    @JSONField(name = "thing8")
    String downStationName;
    
    @Override
    public String getId(){
	    return id;
    }
}
```

理论上我每个实体类里面将模板id直接写死就能实现需求了，但是实际上模板id可能在不同环境下是不同的，每次变化都必须手动更改代码并再次编译，我更希望的是直接在外面的`application.yml`配置文件中直接配置，然后再在实体类上读取，这样就能便捷的动态配置属性了。于是自然而然的想到了`@value`注解，直接将配置注入。

示例：

```java
@Data
public class WechatOrderTemplate implements WechatInfoTemplate{
	
	@value("${template.id}")
    private String id;
    
    // the same code
}
```

想法很不错，但是实际运行的时候发现取得的id值为`null`，那就说明这种方案明显不行。

查阅资料后知道原因：

>@Value 注解是 Spring 框架提供的一种方式，用于在 Spring 管理的 Bean 中注入外部属性值。它可以用于将配置文件中的属性值、系统属性、环境变量等注入到 Bean 的属性中。
>
>下面是 @Value 注解的工作原理：
>属性值注入： @Value 注解可以用在 Bean 的属性上，用于标记需要注入属性值的字段或方法。
>属性值解析： Spring 容器在实例化 Bean 时，会解析 @Value 注解，并根据注解的表达式或值来确定需要注入的属性值。
>表达式语法： @Value 注解支持使用 SpEL（Spring 表达式语言）来指定属性值的来源。您可以在注解中使用 ${} 或 #{} 来引用配置文件属性、系统属性、环境变量等。
>配置源： 属性值可以来自于不同的配置源，例如 `application.properties` 或 `application.yml` 配置文件、系统属性、环境变量等。
>类型转换： Spring 会根据属性的类型自动进行类型转换，将字符串类型的属性值转换为目标类型。
>
>需要注意的是，**@Value 注解仅适用于 Spring 管理的 Bean，您需要确保您的类被 Spring 容器扫描并管理**。

也就是说非组件类是不能使用自动注入的。

# 问题解决

回到最初的问题，我需要注入值的类不是组件类，所以无法使用Spring的自动注入。那么把我的消息类添加`@Component`改成组件类不就行了。这样确实能解决问题，但是这样就相当于滥用`bean`了，可能引起其他很多问题。

那换个思路，能不能直接通过已有的组件类读取呢？

答案是可以的。

```java
@Data
@RequiredArgsConstructor
public class WechatOrderTemplate implements WechatInfoTemplate{
	
    private final String path = "配置文件路径";
    
    private final Environment environment;	 // Spring环境对象读取属性
   
    // the same code
    
    @Override
    public String getId(){
	    return environment.getProperty(path);	// 读取路径配置
    }
}
```

现在就实现了动态读取，只要指明属性名，就能通过`environment`组件动态读取配置文件中的属性。

# 拓展

上面说的`@value`注入属性，为什么只有组件类才有效呢？他底层又是如何实现的呢？

>当使用 `@Value("${my.property}")` 标记的属性时，Spring 容器会执行以下步骤将配置文件中的属性值注入到该属性：
>
>1. **扫描和解析注解：** Spring 容器在初始化时会扫描被管理的组件类，找到带有 `@Value` 注解的字段、方法参数或方法返回值。
>
>2. **解析属性表达式：** 对于 `@Value` 注解，Spring 会解析注解中的属性表达式，即 `${}` 或 `#{}` 中的内容。在 `${}` 中，Spring 将会查找配置文件中对应的属性值。
>
>3. **获取配置属性：** Spring 将根据属性表达式 `${my.property}`，在配置文件（如 `application.properties` 或 `application.yml`）中查找名为 `my.property` 的属性值。
>
>4. **类型转换：** 获取到属性值后，Spring 会根据被注解字段或方法参数的类型进行自动类型转换，将字符串类型的属性值转换为目标类型。
>
>5. **注入至目标位置：** 最终，获取到转换后的属性值，Spring 将会将属性值注入到被注解的字段、方法参数或方法返回值中。
>
>这个过程是由 Spring 容器自动完成的，开发者不需要手动干预。通过在 `@Value` 注解中指定属性表达式，Spring 能够自动找到配置文件中对应的属性值，并将其注入到目标位置。
>
>需要注意的是，这个过程中的属性表达式可以引用配置文件中的属性、系统属性、环境变量等。Spring 在解析属性表达式时会按照一定的优先级顺序查找这些属性值。如果找不到对应的属性值，将会抛出异常。

原因说的比较笼统，那再看看注入属性是如何一步一步实现的：

>`@Value` 注解的底层实现涉及到 Spring 框架的属性解析机制和属性编辑器。虽然详细的底层源码涉及复杂的逻辑，但是您可以大致了解它的实现原理，以便参照写一个类似的自定义注解。
>
>以下是 `@Value` 注解的简化底层实现过程：
>
>1. **属性解析器：** Spring 框架中有一个属性解析器，负责解析属性表达式和获取属性值。它可以解析 `${}` 和 `#{}` 中的内容，并根据配置文件、系统属性、环境变量等获取属性值。
>
>2. **属性编辑器：** Spring 使用属性编辑器来进行类型转换，将字符串类型的属性值转换为目标类型。属性编辑器可以自定义，用于处理不同类型的属性值转换。
>
>3. **BeanPostProcessor：** Spring 提供了一个 `BeanPostProcessor` 接口，允许开发者在 Bean 初始化的前后进行自定义操作。`@Value` 注解底层通过这个机制，在 Bean 初始化之前，使用属性解析器解析属性值并进行类型转换，然后将属性值注入到目标位置。
>
>4. **反射：** Spring 使用反射机制来操作目标位置，将解析后的属性值设置到被注解的字段、方法参数或方法返回值中。
>
>如果您想参照写一个类似的自定义注解，可以按照以下步骤：
>
>1. 创建一个类似的注解，如 `@MyValue`。
>2. 使用自定义注解时，在 `BeanPostProcessor` 的实现中解析注解中的属性表达式，获取属性值。
>3. 使用属性编辑器将属性值转换为目标类型。
>4. 使用反射将属性值设置到目标位置。
>
>请注意，这是一个高度简化的示例，实际上 Spring 的属性解析和注入机制更加复杂和灵活。如果您想深入了解和实现一个类似的自定义注解，建议查阅 Spring 框架的源码和相关文档，以便更好地理解底层实现和机制。

下面是一个简化的示例代码，展示了如何自定义一个类似于 `@Value` 的注解 `@MyValue`，并实现属性值的注入功能。请注意，这个示例是高度简化的，实际的 Spring 框架中涉及更多复杂的逻辑和机制。

首先，创建一个自定义注解 `@MyValue`：

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.PARAMETER})
public @interface MyValue {
    String value();
}
```

然后，创建一个自定义的 `BeanPostProcessor` 实现，用于解析 `@MyValue` 注解并进行属性值注入：

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.context.ApplicationContext;
import org.springframework.stereotype.Component;
import java.lang.reflect.Field;

@Component
public class MyValueBeanPostProcessor implements BeanPostProcessor {
    @Autowired
    private ApplicationContext applicationContext;

@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        Field[] fields = bean.getClass().getDeclaredFields();
        for (Field field : fields) {
            MyValue myValueAnnotation = field.getAnnotation(MyValue.class);
            if (myValueAnnotation != null) {
                String propertyName = myValueAnnotation.value();
                String propertyValue = applicationContext.getEnvironment().getProperty(propertyName);
                if (propertyValue != null) {
                    field.setAccessible(true);
                    try {
                        field.set(bean, propertyValue);
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
        return bean;
    }

@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

最后，在需要使用 `@MyValue` 注解的类中进行注入：

```java
import org.springframework.stereotype.Component;

@Component
public class MyBean {
    @MyValue("my.property")
    private String myProperty;

    public String getMyProperty() {
        return myProperty;
    }
}
```

在这个示例中，`@MyValue` 注解模拟了类似 `@Value` 的注解，`MyValueBeanPostProcessor` 实现了 `BeanPostProcessor` 接口，用于在 Bean 初始化过程中解析注解并进行属性值注入。

需要注意的是，这只是一个简化的示例，实际的 Spring 框架中有更多复杂的逻辑和机制来支持属性解析和注入。如果您想实现一个更完整和稳定的自定义注解，建议深入了解 Spring 框架的源码和文档。
