---
title: Flink 中的依赖注入
tags:
  - flink
  - guice
  - java
categories:
  - Big Data
date: 2024-02-07 13:28:41
alt_en: /2024/02/07/dependency-injection-in-flink/
---

*本文由 Qwen3 Max 翻译*

## TL;DR

使用 [Guice][1] 构建依赖图：

```java
public class DatabaseModule extends AbstractModule {
  @Provides @Singleton
  public DataSource provideDataSource() {
    return new HikariDataSource();
  }

  @Provides @Singleton
  public UserRepository provideUserRepository(DataSource dataSource) {
    return new UserRepositoryImpl(dataSource);
  }
}
```

创建单例注入器：

```java
public class AppInjector {
  private static class Holder {
    static final Injector INJECTOR = Guice.createInjector(new DatabaseModule());
  }

  private AppInjector() {}

  public static void injectMembers(Object instance) {
    Holder.INJECTOR.injectMembers(instance);
  }
}
```

在 Flink 函数中使用：

```java
public class UserMapper extends RichMapFunction<Long, User> {
  @Inject
  transient UserRepository userRepository;

  @Override
  public void open(Configuration parameters) throws Exception {
    AppInjector.injectMembers(this);
  }

  @Override
  public User map(Long userId) throws Exception {
    Objects.requireNonNull(userId, "用户 ID 为空");
    return userRepository.getById(userId).orElseThrow(() -> new RuntimeException("未找到用户"));
  }
}
```


## 动机

依赖注入（Dependency Injection，简称 DI）是 Java 编程中的常见实践，尤其当你有 Spring 背景时更是如此。最直接的好处是可测试性，即你可以用测试桩（stub）替换类的实现。其他好处还包括关注点分离、更好的类层次结构、控制反转等。组件通过类构造函数或带注解的成员声明其依赖，而 DI 框架则创建一个容器（或上下文）来正确地连接这些组件。该上下文通常在应用启动时创建，并贯穿整个应用生命周期。一些例子包括 Spring 的 `ApplicationContext` 和 Guice 的 `Injector`。

Flink 是一个分布式计算框架，通过依赖注入将业务逻辑与框架解耦是有利的。然而，Flink 应用由函数式类组成，这些类在驱动类（即 `main` 方法）中实例化、序列化后发送到分布式任务管理器。除非我们所有的组件都是可序列化的，否则无法将依赖注入到这些类中。幸运的是，Flink 提供了一个生命周期钩子 `open`，它在作业启动时被调用。结合另一种常见模式——单例模式，我们可以让 DI 框架与 Flink 良好协作。

<!-- more -->


## Guice 快速入门

我选择的依赖注入框架是 Guice，因为它简单、轻量且高效。通常我们通过构造函数声明类依赖，将所有组件添加到模块中，然后让 Guice 完成其余工作。


### 声明依赖

有三种方式为类声明依赖。推荐使用构造函数方式。

```java
import com.google.inject.Inject;
// 或 import jakarta.inject.Inject;

// 1. 构造函数
public class UserRepositoryImpl implements UserRepository {
  private DataSource dataSource;

  @Inject
  public UserRepositoryImpl(DataSource dataSource) {
    this.dataSource = dataSource;
  }
}

// 2. 成员变量
class UserRepositoryImpl implements UserRepository {
  @Inject
  private DataSource dataSource;
}

// 3. Setter 方法
public class UserRepositoryImpl implements UserRepository {
  private DataSource dataSource;

  @Inject
  public void setDataSource(DataSource dataSource) {
    this.dataSource = dataSource;
  }
}
```


### 将组件添加到模块

模块是 Guice 用于配置组件的机制。例如如何初始化组件、哪个具体类实现接口、当存在多个实现时如何处理等。组件被分组到模块中，模块本身也可以组合在一起。这里涉及很多主题，可参考其[官方文档][2]，我将介绍一些基本用法。

首先，只要 Guice 能仅凭类类型和注解推断出依赖图，就可以隐式地创建组件。例如：

```java
@ImplementedBy(UserRepositoryImpl.class)
public interface UserRepository {}

public class UserRepositoryImpl implements UserRespository {
  @Inject
  private HikariDataSource dataSource;
}

var injector = Guice.createInjector();
injector.getInstance(UserRepository.class);
```

`dataSource` 的类型是 `HikariDataSource`，这是一个具体类，因此 Guice 知道如何创建它。如果类型是 `DataSource`，Guice 将抛出缺少实现的错误。但对于 `UserRepository`，由于我们使用了 `ImplementedBy` 注解声明了实现类，Guice 知道其具体实现。否则，我们需要在模块中声明这种关系：

```java
import com.google.inject.AbstractModule;
import com.google.inject.Provides;

// 1. 添加绑定
public class DatabaseModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(UserRepository.class).to(UserRepositoryImpl.class);
  }
}

// 2. 使用提供方法
public class DatabaseModule extends AbstractModule {
  @Provides
  public UserRepository provideUserRepository(UserRepositoryImpl impl) {
    return impl;
  }
}

var injector = Guice.createInjector(new DatabaseModule());
injector.getInstance(UserRepository.class);
```

这两种方法是等价的。第二种方法可以这样理解：

* 用户请求一个 `UserRepository` 实例。
* Guice 发现带有 `@Provides` 注解且返回类型匹配的 `provideUserRepository` 方法。
* 该方法需要一个 `UserRepositoryImpl` 参数。
* Guice 隐式创建该实现类的实例，因为它是具体类。
* 方法获取该实例，可能对其进行修改，然后返回给用户。

第二种方法与我们之前使用的略有不同：之前方法的参数是 `DataSource`，我们手动创建 `UserRepositoryImpl`：

```java
@Provides
public UserRepository provideUserRepository(DataSource dataSource) {
  return new UserRepositoryImpl(dataSource);
}
```

在这种情况下，`UserRepositoryImpl` 中的 `@Inject` 注解可以省略，因为 Guice 不负责创建该实例，除非你显式地从 Guice 请求一个 `UserRepositoryImpl` 实例。

在提供方法中，我们可以配置返回的实例：

```java
@Provides @Singleton
public DataSource provideDataSource() {
  var config = new HikariConfig();
  config.setJdbcUrl("jdbc:mysql://localhost:3306/flink_di");
  config.setUsername("root");
  config.setPassword("");
  return new HikariDataSource(config);
}
```

最后，模块可以组合在一起：

```java
public class EtlModule extends AbstractModule {
  @Override
  protected void configure() {
    install(new ConfigModule());
    install(new DatabaseModule());
    install(new RedisModule());
  }
}

var injector = Guice.createInjector(new EtlModule());
```


### 命名与作用域组件

当同一类型存在多个不同配置的实例时，可使用 `@Named` 注解加以区分。也可以创建[自定义注解][3]，或在 `AbstractModule#configure` 中使用绑定，而非提供方法。

```java
public class DatabaseModule extends AbstractModule {
  @Provides @Named("customer") @Singleton
  public DataSource provideCustomerDataSource() {
    return new HikariDataSource();
  }

  @Provides @Named("product") @Singleton
  public DataSource provideProductDataSource() {
    return new HikariDataSource();
  }
}

@Singleton
public class UserRepositoryImpl extends UserRepository {
  @Inject @Named("customer")
  private DataSource dataSource;
}
```

数据源和实现类实例都标注了 `@Singleton`，表示 Guice 在每次请求时都会返回同一个实例。否则，其行为类似于 Spring 中的[原型作用域（prototype scope）][4]。


## Flink 流水线序列化

考虑以下简单流水线：将 ID 流转换为用户模型并打印到控制台。

```java
var env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStreamSource<Long> source = env.fromElements(1L);
DataStream<User> users = source.map(new UserMapper());
users.print();

env.execute();
```

在底层，Flink 会将该流水线构建成作业图，进行序列化，并发送到远程任务管理器。`map` 算子接收一个 `MapFunction` 实现，在本例中是一个 `UserMapper` 实例。该实例被包装在 `SimpleUdfStreamOperatorFactory` 中，并通过 Java 对象序列化机制进行序列化。

```java
// org.apache.flink.util.InstantiationUtil
public static byte[] serializeObject(Object o) throws IOException {
  try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
      ObjectOutputStream oos = new ObjectOutputStream(baos)) {
    oos.writeObject(o);
    oos.flush();
    return baos.toByteArray();
  }
}
```

流水线算子最终变成一系列配置的哈希映射，并通过远程调用发送给作业管理器。

```
org.apache.flink.configuration.Configuration {
  operatorName=Map,
  serializedUdfClassName=org.apache.flink.streaming.api.operators.SimpleUdfStreamOperatorFactory,
  serializedUDF=[B@6c67e137,
}
```

为了让 `ObjectOutputStream` 正常工作，流水线中的每个类及其成员字段都必须实现 `Serializable` 接口。对于 `UserMapper`，它继承了实现 `Serializable` 接口的 `RichMapFunction`。然而，如果我们添加一个不可序列化的依赖对象，就会发生错误：

```java
public class UserMapper extends RichMapFunction<Long, User> {
  @Inject
  UserRepository userRepository;
}

// main
var injector = Guice.createInjector(new DatabaseModule());
var userMapper = injector.getInstance(UserMapper.class);
DataStream<User> users = source.map(userMapper);
// java.io.NotSerializableException: com.zaxxer.hikari.pool.HikariPool$PoolEntryCreator
```

这是因为 `HikariDataSource` 不可序列化。因此，无法通过序列化携带 `userRepository`，而应在 `UserMapper` 被恢复并调用 `open` 方法后设置它，如本文开头所示。我们添加 `transient` 关键字，告知 Java 在序列化时不包含该字段。

```java
public class UserMapper extends RichMapFunction<Long, User> {
  @Inject
  transient UserRepository userRepository;

  @Override
  public void open(Configuration parameters) throws Exception {
    AppInjector.injectMembers(this);
  }
}
```

在 `AppInjector` 中，我们使用单例模式确保只有一个 Guice 注入器，而 Guice 本身是线程安全的，因此连接池等重量级资源可以在不同的用户自定义函数之间共享。


## 单元测试

如前所述，依赖注入提高了可测试性。要测试 `UserMapper`，我们可以模拟其依赖项，并像普通函数一样进行测试。其他测试技术可参见[官方文档][5]。

```java
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

public class UserMapperTest {
  @Test
  public void testMap() throws Exception {
    var userRepository = mock(UserRepository.class);
    when(userRepository.getById(1L))
        .thenReturn(Optional.of(new User(1L, "jizhang", new Date())));

    var userMapper = new UserMapper();
    userMapper.userRepository = userRepository;
    assertEquals("jizhang", userMapper.map(1L).getUsername());
  }
}
```


## 参考资料
* https://github.com/google/guice/wiki/GettingStarted
* https://getindata.com/blog/writing-flink-jobs-using-spring-dependency-injection-framework/
* https://medium.com/airteldigital/designing-and-developing-a-real-time-streaming-platform-with-flink-and-google-guice-213b40e063de


[1]: https://github.com/google/guice
[2]: https://github.com/google/guice/wiki/Bindings
[3]: https://github.com/google/guice/wiki/BindingAnnotations
[4]: https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html#beans-factory-scopes-prototype
[5]: https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/datastream/testing/
