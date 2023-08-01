---
title: 关于Mybatis四大核心组件的思考
date: 2023-08-01 14:42:32
tags: [Mybatis,Spring,Java,SqlSession]
categories: Java
---

# 问题产生

上两个月面试的时候被问到`Mybatis`的四大核心组件的是什么，当时就没了解过直接被拷打了，后面回来的时候大致了解了一下，但是缺乏有效的总结，最近在做`Mybatis`批量插入数据的时候又想到了这个问题，正好作次有效的总结。

# 四大组件

1. **Configuration（配置信息）**：
   - `Configuration` 是 MyBatis 框架的核心配置信息对象。
   - 它负责管理 MyBatis 的全局配置信息，包括数据库连接池、映射器注册、插件、类型处理器等。
   - `Configuration` 对象通常由 `SqlSessionFactoryBuilder` 使用 XML 配置文件来创建，并传递给 `SqlSessionFactory`。
   - 通过 `Configuration`，MyBatis 可以实现高度的灵活性和可配置性，满足不同项目的需求。
2. **SqlSessionFactory（Sql会话工厂）**：
   - `SqlSessionFactory` 是 MyBatis 的核心接口之一，用于创建 `SqlSession` 对象。
   - 它是应用程序与 MyBatis 框架之间的桥梁，负责加载 MyBatis 的配置信息，并创建 `SqlSession` 对象。
   - `SqlSessionFactory` 通常是在应用程序启动时创建的，因为它的创建成本较高，但是一旦创建后，可以重复使用，并且线程安全，因此通常是单例的。
3. **SqlSession（Sql会话）**：
   - `SqlSession` 是 MyBatis 与数据库之间交互的核心接口。
   - 它代表了一个与数据库的连接会话，用于执行 SQL 语句、获取映射器（Mapper）等数据库操作。
   - `SqlSession` 提供了多种数据库操作方法，包括查询、插入、更新、删除等。
   - 注意，`SqlSession` 是非线程安全的，因此每个线程通常都应该拥有自己的 `SqlSession` 实例。
4. **Mapper（映射器）**：
   - Mapper 是 MyBatis 中定义数据操作接口的组件，用于执行数据库操作。
   - 在 MyBatis 中，Mapper 接口是通过 XML 映射文件或注解来定义的，用于描述数据库操作的 SQL 语句和参数映射。
   - Mapper 接口与具体的数据库实现无关，它只是描述数据操作的规范，而实际的 SQL 语句和参数映射是在映射文件或注解中定义的。
   - MyBatis 提供了许多辅助类和注解，用于将 Mapper 接口与映射文件或注解关联起来，使其能够与数据库交互。

实际Spring开发中也就是上面四点的递进关系，逐步创建对象，示例：

```
SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build((Reader) null);	// build构建必须传入配置，可以是xml文件路径，或者直接是Configuration配置对象
SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
RpDriverLineIncomeMapper mapper = sqlSession.getMapper(RpDriverLineIncomeMapper.class);
```

现在获取的`Mapper`其实是个代理对象了：

- 动态代理：当你使用 `SqlSession.getMapper()` 方法获取 Mapper 接口的实例时，MyBatis 实际上并没有提供 Mapper 接口的具体实现类。而是在运行时使用 Java 的动态代理机制，动态地生成一个 Mapper 接口的实现类，并返回这个实现类的实例给你。
- 方法调用：当你通过 Mapper 接口的实例调用其中的方法时，实际上是调用了动态生成的实现类的对应方法。在这个实现类中，MyBatis 将会根据映射关系，解析相应的 SQL 语句，然后执行数据库操作。

另外值得一提的是用了`Spring Boot`,那么不就是没有了关于Mybatsi的xml配置文件了吗？那么又该如何构建`SqlSessionFactory`对象呢？其实`SqlSessionFactory`现在以及被自动注册成`bean`了，直接注入就可以了：

```
    @Autowired
    private  SqlSessionFactory sqlSessionFactory;
```

# 批量插入问题


MyBatis 提供了三种批量插入数据的方式：

1. 遍历数据，每次都调用一次插入一条数据的方法：

   ```
   list.forEach(YourMapper::insert)
   ```

   这种方法最不推荐，插入数据会开启多次连接，极为浪费资源。

2. 使用 `<foreach>` 标签： 这是最常见的批量插入数据的方式。你可以使用 MyBatis 的 `<foreach>` 标签来动态生成插入语句，将多个数据对象一次性插入到数据库。在 XML 映射文件中，你可以在插入语句中使用 `<foreach>` 标签，将一个 Java 集合中的数据作为参数传递给插入语句。

   示例代码：

   ```
   xmlCopy code<!-- YourMapper.xml -->
   <mapper namespace="com.example.YourMapper">
   
     <insert id="batchInsert" parameterType="java.util.List">
       INSERT INTO your_table (column1, column2)
       VALUES
       <foreach collection="list" item="item" separator=",">
         (#{item.column1}, #{item.column2})
       </foreach>
     </insert>
   
   </mapper>
   ```

   Java 代码：

   ```
   javaCopy code// YourMapper.java
   public interface YourMapper {
     void batchInsert(List<YourObject> dataList);
   }
   ```

3. 使用 BatchExecutor 类型的 SqlSession： 通过将 `SqlSession` 的执行类型设置为 `ExecutorType.BATCH`，你可以在 Java 代码中手动执行批量插入操作。这种方式不需要使用 `<foreach>` 标签，而是通过循环遍历数据列表，使用单条插入语句，然后在合适的时机一次性提交所有插入操作。

   示例代码：

   ```
   javaCopy codeimport org.apache.ibatis.session.SqlSession;
   import org.apache.ibatis.session.SqlSessionFactory;
   import org.apache.ibatis.session.ExecutorType;
   
   // Assuming you already have a SqlSessionFactory instance
   SqlSessionFactory sqlSessionFactory = getSqlSessionFactory();
   
   // Use BatchExecutor type of SqlSession
   SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
   
   try {
       YourMapper yourMapper = sqlSession.getMapper(YourMapper.class);
       List<YourObject> dataList = new ArrayList<>();
       // Add multiple YourObject instances to dataList
   
       // Perform batch insertion
       for (YourObject obj : dataList) {
           yourMapper.insert(obj);
       }
   
       // Commit the batch
       sqlSession.commit();
   } catch (Exception e) {
       sqlSession.rollback();
   } finally {
       sqlSession.close();
   }
   ```

值得说的是第二种和第三种插入，这两种方法在实际项目中才有使用价值，那么他们两种方法如何取舍呢？

使用 `<foreach>` 标签和 `BatchExecutor` 类型的 `SqlSession` 实现批量插入，取决于数据集的大小、数据库配置、性能需求和数据完整性等因素。每种方法都有其优势和权衡。

使用 `<foreach>` 标签：

- 优势：
  - 简化代码：`<foreach>` 标签允许你编写一个包含多个插入行的 SQL 语句，简化了代码并减少了应用程序与数据库之间的交互次数。
  - 可能更快：对于适度大小的数据集，使用 `<foreach>` 标签可能更快，因为它减少了数据库交互的次数，而数据库可以针对单个查询优化执行计划。
- 权衡：
  - SQL 长度限制：使用 `<foreach>` 标签的一个限制是如果数据集特别大，SQL语句就会异常的长，可能会达到数据库支持的 SQL 长度限制，从而导致潜在的数据库错误。

使用 `BatchExecutor` 类型的 `SqlSession`：

- 优势：
  - 适用大数据集：`BatchExecutor` 可以处理大数据集，通过将单个插入语句作为批处理执行。它不会受到 `<foreach>` 标签的 SQL 长度限制的影响。
  - 更多的事务控制：使用 `BatchExecutor`，你可以更多地控制事务。你可以显式地提交或回滚批处理，这在某些情况下可能有助于数据完整性。
- 权衡：
  - 更复杂的代码：使用 `BatchExecutor` 实现批量插入涉及编写额外的代码来循环遍历数据并执行单个插入语句。与 `<foreach>` 标签相比，可能增加了一些代码复杂性。

选择合适的方法取决于你的具体用例和数据库配置。如果你有一个大数据集，可能会超过 SQL 长度限制，或者需要更多的事务控制，那么使用 `BatchExecutor` 可能是一个更好的选择。另一方面，如果你的数据集大小适中，且希望代码简单，`<foreach>` 标签可能已经足够。

在实践中，通常根据情况使用这两种方法。对于较小的数据集或 SQL 长度不是问题的情况下，可以使用 `<foreach>` 标签。对于较大的数据集或需要事务控制的情况，可能更倾向于使用 `BatchExecutor` 方法。

#  SqlSession类型

`SqlSession`由`SqlSessionFactory`创立，在创立的时候可以选择开启的Session类型：

```
SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
```

`ExecutorType`枚举源码：

![image-20230801152116298](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2023/08/1_15_21_26_image-20230801152116298.png)

可见总共三种类型，那么他们之间有何区别呢？

1. **SIMPLE**： `ExecutorType.SIMPLE` 是最简单的执行方式，它每次执行 SQL 语句时，都会创建一个新的 Statement 对象，并立即执行 SQL 语句，然后关闭 Statement。这种方式适合简单的 SQL 操作。
2. **REUSE**： `ExecutorType.REUSE` 是复用方式，它在执行 SQL 语句时，会重用先前创建的 Statement 对象。如果 Statement 对象已经存在，就会使用它执行新的 SQL 语句，否则会创建一个新的 Statement。执行完成后，Statement 不会关闭，而是保存在缓存中，供下次复用。这种方式适合频繁执行相同 SQL 语句的情况，可以减少 Statement 的创建和销毁开销。
3. **BATCH**： `ExecutorType.BATCH` 是批处理方式，它用于执行批量操作，例如批量插入、更新或删除。在 `BATCH` 模式下，会积累多个 SQL 语句，并一次性提交到数据库执行，从而减少数据库交互次数，提高性能。

像之前的批量插入示例就是采用了`Batch`类型的SqlSeion,多次插入，一次提交，来达到批量插入的目的。

那又有个问题，为什么打开的是`SqlSession`，他不去叫~~SqlSession~~,而是叫**ExecutorType**呢？他们之间有何联系？当使用MyBatis时，`SqlSession` 和 `Executor` 是密切相关的组件，理解它们的关系对于理解MyBatis的内部工作机制非常重要。

1. `SqlSession`：

   - `SqlSession` 是MyBatis提供的高级接口，表示用于执行SQL语句和管理与数据库的事务会话。
   - 它提供了各种方法来执行数据库操作，比如查询、插入、更新和删除数据。
   - `SqlSession` 是对单个数据库连接的包装，用于在该连接上执行SQL语句。

2. `Executor`：

   - `Executor` 是MyBatis的低级组件，负责执行SQL语句。

   - 它负责准备和执行针对数据库的SQL语句。

   - ```
     Executor
     ```

      接口有两个主要的实现类：

     - `SimpleExecutor`：它是MyBatis用于非批量SQL操作的默认执行器。它直接执行SQL语句。
     - `BatchExecutor`：它用于批量操作，比如批量插入、更新和删除。它将多个SQL语句组合在一起，并作为批量执行。

`SqlSession` 和 `Executor` 之间的关系如下：

1. 当使用 `SqlSession` 提供的方法执行数据库操作（比如 `selectOne`、`insert`、`update`等），`SqlSession` 会将执行委托给合适的 `Executor`。
2. 对于非批量操作，`SqlSession` 使用 `SimpleExecutor` 来执行SQL语句。
3. 对于批量操作，`SqlSession` 使用 `BatchExecutor`，将多个SQL语句组合在一起，并作为批量执行，以提高性能。
4. `SqlSession` 充当高级接口，提供了更用户友好的方式来与数据库交互，而 `Executor` 是低级组件，负责实际执行SQL语句。

值得注意的是，`SqlSession` 也管理事务范围，根据你的配置和使用方式，控制事务的开始、提交和回滚。而 `Executor` 则专注于执行SQL语句，不直接涉及事务管理。

# 批量插入的拓展

之前所说的批量插入，由于原子性的存在，只要插入数据中有一条数据异常，就会造成整个批量插入的失败。但是在某些场景下，我是允许批量插入中的失败的，那么又该怎么写这种允许失败的批量插入方法呢？

先从`MySql`的层次来讲，`MySQL`是存在允许批量插入部分失败的：

1. 使用INSERT IGNORE语句：这个方法可以在插入数据时忽略已存在的唯一索引冲突或主键重复的记录，而不会抛出错误，允许插入操作继续进行。但是，注意这种方式只适用于唯一索引或主键的冲突。

示例：
```sql
INSERT IGNORE INTO your_table (column1, column2, ...) VALUES
(value1, value2, ...),
(value3, value4, ...),
...
```

2. 使用INSERT INTO... ON DUPLICATE KEY UPDATE语句：这种方法在插入数据时，如果发生唯一索引冲突或主键重复，可以选择更新已存在的记录而不会抛出错误，允许插入操作继续进行。

示例：
```sql
INSERT INTO your_table (column1, column2, ...)
VALUES (value1, value2, ...)
ON DUPLICATE KEY UPDATE column1 = VALUES(column1), column2 = VALUES(column2), ...;
```

3. 使用事务（TRANSACTION）：使用事务可以将多个INSERT语句包含在一个事务中，如果发生错误，可以回滚（ROLLBACK）整个事务，从而允许部分失败。

示例：
```sql
START TRANSACTION;

INSERT INTO your_table (column1, column2, ...) VALUES (value1, value2, ...);
INSERT INTO your_table (column1, column2, ...) VALUES (value3, value4, ...);
...

COMMIT;
```



在此基础上，便可以完成`Mybatis`中实现允许部分失败的批量插入方法：

1. 使用 INSERT IGNORE 语句： 对于 `INSERT IGNORE`，MyBatis 提供了 `insert` 标签，并且可以在 SQL 映射文件中编写相应的 SQL 查询。

   SQL 映射文件示例：

   ```
   xmlCopy code<insert id="batchInsert" parameterType="java.util.List">
     INSERT IGNORE INTO your_table (column1, column2, ...) VALUES
     <foreach collection="list" item="item" separator=",">
       (#{item.value1}, #{item.value2}, ...)
     </foreach>
   </insert>
   ```

2. 使用 INSERT INTO... ON DUPLICATE KEY UPDATE 语句： 对于 `INSERT INTO... ON DUPLICATE KEY UPDATE`，同样可以使用 `insert` 标签，并在 SQL 映射文件中编写相应的 SQL 查询。

   SQL 映射文件示例：

   ```
   xmlCopy code<insert id="batchInsert" parameterType="java.util.List">
     INSERT INTO your_table (column1, column2, ...)
     VALUES
     <foreach collection="list" item="item" separator=",">
       (#{item.value1}, #{item.value2}, ...)
     </foreach>
     ON DUPLICATE KEY UPDATE
     column1 = VALUES(column1), column2 = VALUES(column2), ...;
   </insert>
   ```

3. 使用事务（TRANSACTION）： MyBatis 支持事务管理，你可以在 Service 层或者在 XML 映射文件中配置事务。这样，如果插入数据过程中发生错误，可以回滚整个事务，从而实现部分失败。

   Service 层示例（基于 Spring 的声明式事务管理）：

   ```
   javaCopy code@Service
   @Transactional
   public class YourService {
       @Autowired
       private YourMapper yourMapper;
   
       public void batchInsert(List<YourObject> dataList) {
           yourMapper.batchInsert(dataList);
       }
   }
   ```

   SQL 映射文件示例：

   ```
   xmlCopy code<insert id="batchInsert" parameterType="java.util.List">
     <foreach collection="list" item="item" separator=";">
       INSERT INTO your_table (column1, column2, ...) VALUES (#{item.value1}, #{item.value2}, ...)
     </foreach>
   </insert>
   ```

现在又有个问题，前面说的两种方法都是在存在唯一索引冲突或主键重复才能使用，假如我就是一个普通的字段，那就只能用第三中方法，但是第三种方法从本质上又回到了之前说的了，插入多少数据就要给数据库发多少次请求，从本质上就不能算批量插入了。那么是否有什么合适的解决方法呢？博主苦思冥想半天，也没找到好方法，或许从数据库设计本身来说，插入数据就应该保持原子性，允许部分失败就是在破坏它的原子性，因而没有什么好方法解决这个问题吧。
