---
title: springboot 整合mongodb实现CRUD
categories: java
tags:
  - spring boot
  - mongodb
  - spring data jpa
  - querydsl
abbrlink: 634f3968
date: 2019-11-27 13:43:02
---

## MongoDB 简介

MongoDB（来自于英文单词“Humongous”，中文含义为“庞大”）是可以应用于各种规模的企业、各个行业以及各类应用程序的开源数据库。基于分布式文件存储的数据库。由C++语言编写。旨在为 WEB 应用提供可扩展的高性能数据存储解决方案。MongoDB 是一个高性能，开源，无模式的文档型数据库，是当前 NoSql 数据库中比较热门的一种。

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。他支持的数据结构非常松散，是类似 json 的 bjson 格式，因此可以存储比较复杂的数据类型。MongoDB 最大的特点是他支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引。

传统的关系数据库一般由数据库（database）、表（table）、记录（record）三个层次概念组成，MongoDB 是由数据库（database）、集合（collection）、文档对象（document）三个层次组成。MongoDB 对于关系型数据库里的表，但是集合中没有列、行和关系概念，这体现了模式自由的特点。

MongoDB 中的一条记录就是一个文档，是一个数据结构，由字段和值对组成。MongoDB 文档与 JSON 对象类似。字段的值有可能包括其它文档、数组以及文档数组。MongoDB 支持 OS X、Linux 及 Windows 等操作系统，并提供了 Python，PHP，Ruby，Java及 C++ 语言的驱动程序，社区中也提供了对 Erlang 及 .NET 等平台的驱动程序。

MongoDB 的适合对大量或者无固定格式的数据进行存储，比如：日志、缓存等。对事物支持较弱，不适用复杂的多文档（多表）的级联查询。



## 配置

### maven依赖配置

在pom文件的dependencies节点中增加依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

### application配置

在项目的application.yml配置文件中加入mongodb服务地址，集群模式配置将地址用逗号隔开即可，举例如下：

```yaml
# mongodb
spring:
  data:
    mongodb:
      uri: mongodb://username:password@ip:port,ip2:port,ip3:port3/databaseName
```

### spring boot 启动类配置

在spring boot项目启动类上增加注解__@EnableMongoRepositories__，basePackages指定需要扫描的包路径：

```java
@SpringBootApplication()
@EnableMongoRepositories(basePackages = {"com.xxx.xxx.xx.xxxx.repository"})
public class Application {
     public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.addListeners(new AppStartedListener());
        app.run(args);
    }
}
```



## MongoDB 增删改查

### 直接继承MongoRepository实现增删改查

首先创建实体类：

```java
@Document(collection = "SysOperateRecord")
@Data
public class SysOperateRecord implements Serializable {
    @Id
    private String id;

    /**
     * 操作类型:参数管理，仓库管理，子仓管理，库区管理，货位管理，包裹管理，波次管理，
     */
    private String operateCode;

    /**
     * 操作类型名称
     */
    private String operateName;

    /**
     * 操作的记录ID
     */
    @Indexed
    private Integer operateId;

    /**
     * 操作内容
     */
    private String operateContent;

    /**
     * 操作人
     */
    private String operateUser;

    /**
     * 备注
     */
    private String remark;

    /**
     * 是否删除标记0 未删除 1 已删除
     */
    private Boolean isDelete;

    /**
     * 创建时间
     */
    private Date createTime;

    /**
     * 更新时间
     */
    private Date lastUpdateTime;

    /**
     * 操作前
     */
    private String operateBefore;

    private static final long serialVersionUID = 1L;

}
```

其中

@Document：标识该类为mongodb文档类，设置的collection名会直接映射到mongodb中

@id：标识字段为主键id

@Indexed：以该字段创建索引

更多注解请参考[官方文档](https://docs.spring.io/spring-data/mongodb/docs/2.2.2.RELEASE/reference/html/#mapping-usage-annotations)

新建xxxRepository类继承MongoRepository：

```java
@Repository
public interface SysOperateRecordRepository extends MongoRepository<SysOperateRecord, String>{
    
}
```

在MongoRepository接口中提供了一系列的增删改查接口如下：

```java
@NoRepositoryBean
public interface MongoRepository<T, ID extends Serializable>
		extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.CrudRepository#save(java.lang.Iterable)
	 */
	@Override
	<S extends T> List<S> save(Iterable<S> entites);

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.CrudRepository#findAll()
	 */
	@Override
	List<T> findAll();

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.PagingAndSortingRepository#findAll(org.springframework.data.domain.Sort)
	 */
	@Override
	List<T> findAll(Sort sort);

	/**
	 * Inserts the given entity. Assumes the instance to be new to be able to apply insertion optimizations. Use
	 * the returned instance for further operations as the save operation might have changed the entity instance
	 * completely. Prefer using {@link #save(Object)} instead to avoid the usage of store-specific API.
	 *
	 * @param entity must not be {@literal null}.
	 * @return the saved entity
	 * @since 1.7
	 */
	<S extends T> S insert(S entity);

	/**
	 * Inserts the given entities. Assumes the given entities to have not been persisted yet and thus will optimize the
	 * insert over a call to {@link #save(Iterable)}. Prefer using {@link #save(Iterable)} to avoid the usage of store
	 * specific API.
	 *
	 * @param entities must not be {@literal null}.
	 * @return the saved entities
	 * @since 1.7
	 */
	<S extends T> List<S> insert(Iterable<S> entities);

	/* 
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.query.QueryByExampleExecutor#findAll(org.springframework.data.domain.Example)
	 */
	@Override
	<S extends T> List<S> findAll(Example<S> example);

	/* 
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.query.QueryByExampleExecutor#findAll(org.springframework.data.domain.Example, org.springframework.data.domain.Sort)
	 */
	@Override
	<S extends T> List<S> findAll(Example<S> example, Sort sort);
}
```

通过调用save方法，可以实现批量新增和修改记录。当记录存在时会自动更新记录，不存在则新增记录。

通过调用insert方法，可以实现新增或者批量新增记录。

通过调用findAll方法，可以实现查询记录。

同时，MongoRepository还继承了PagingAndSortingRepository和QueryByExampleExecutor，分别提供了分页查询方法和基于Example查询的方法。

使用举例：

```java
//调用insert方法新增记录
SysOperateRecord sysOperateRecord = new SysOperateRecord();
sysOperateRecord.setOperateCode(operateType.getCode());
sysOperateRecord.setOperateName(operateType.getDescription());
sysOperateRecord.setOperateId(i);
sysOperateRecord.setOperateContent(operateType.getDescription());
sysOperateRecord.setOperateUser("root");
sysOperateRecord.setRemark(operateType.getDescription());
sysOperateRecordRepository.insert(sysOperateRecords); //insert支持传入list批量新增

//调用save方法新增记录
SysOperateRecord sysOperateRecord = new SysOperateRecord();
sysOperateRecord.setOperateCode(operateType.getCode());
sysOperateRecord.setOperateName(operateType.getDescription());
sysOperateRecord.setOperateId(i);
sysOperateRecord.setOperateContent(operateType.getDescription());
sysOperateRecord.setOperateUser("root");
sysOperateRecord.setRemark(operateType.getDescription());
sysOperateRecordRepository.save(sysOperateRecords); //save支持传入list批量新增

//调用save方法更新记录
SysOperateRecord sysOperateRecord = new SysOperateRecord();
sysOperateRecord.setId("xxxxx"); //id在mongodb collection中已经存在
sysOperateRecord.setOperateCode(operateType.getCode());
sysOperateRecord.setOperateName(operateType.getDescription());
sysOperateRecord.setOperateId(i);
sysOperateRecord.setOperateContent(operateType.getDescription());
sysOperateRecord.setOperateUser("root");
sysOperateRecord.setRemark(operateType.getDescription());
sysOperateRecordRepository.save(sysOperateRecords); //save支持传入list批量更新

//使用delete方法删除记录
sysOperateRecordRepository.delete("sss");//删除id为sss的记录

//调用findAll查询记录
SysOperateRecord sysOperateRecord = new SysOperateRecord();
sysOperateRecord.setOperateCode("AAA");
sysOperateRecord.setOperateId(1);
sysOperateRecordRepository.findAll(Example.of(sysOperateRecord), new Sort(Sort.Direction.DESC, "createTime")); //查询OperateCode为AAA且OperateId为1的记录，根据createTime倒序排序

```

此外在xxxRepository中，我们可以自定义方法进行查询：

```java
 /**
     * 根据操作id查询记录
     *
     * @param operateId
     * @return
     */ 
    List<SysOperateRecord> findByOperateId(Integer operateId);
```

更多用法请参考[spring data jpa](https://docs.spring.io/spring-data/jpa/docs/2.2.2.RELEASE/reference/html/#reference) 

### 使用@Query注解

在上一节中，我们直接继承MongoRepository，使用框架内置的查询方法可以进行一些简单查询。奈何在实际使用时，我们会使用像IN这种操作，如果查询限制条件非常多的话，可能我们会自定义如下方法：

```java
List<SysOperateRecord> findByOperateCodeAndOperateBeforeAndOperateContentAndOperateIdIn(String operateCode,String operateBefore, String operateContent, List<Integer> operateIds);
```

方法名会变的特别长，很难看也难以维护。

此时我们可以使用**@Query**注解

```java
@Query(value = "{operateCode: ?0, operateBefore:?1,operateContent:?2,operateId:{$in: ?3}, isDelete:false}")
    List<SysOperateRecord> find(String operateCode,String operateBefore, String operateContent, List<Integer> operateIds);
```

注解value字段直接写入json形式的字符串，占位符？0表方法第一个参数，以此类推。

### 使用Querydsl

上节我们使用**@Query**注解解决了方法名过长的问题，有心的小伙伴会发现，使用注解还是会存在俩个问题：

1. 我们必须对manogo原生查询非常熟悉才能很快写出json字符串
2. 当查询的条件很多时，方法参数也会变的很多，易读性差，使用性也差

那么有没有更优雅的方式来进行复杂条件查询呢，of course。

#### Querydsl简介

[官方](http://www.querydsl.com/)介绍如下：

```markdown
Querydsl is a framework which enables the construction of type-safe SQL-like queries for multiple backends including JPA, MongoDB and SQL in Java.

Instead of writing queries as inline strings or externalizing them into XML files they are constructed via a fluent API.
```

大意就是使用这个类库，可以让我像写sql一样的去写java代码。免去了我们手动在字符串中或者xml中写原生sql的麻烦。听起来好像还不错，那么如何在spring项目中使用该类库呢？

很简单，sping data对Querydsl进行了集成。咱们需要额外引入的是Querydsl对mongodb提供支持的类库。

#### pom文件配置

##### 1. 在pom文件的依赖中添加：

```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-mongodb</artifactId>
</dependency>
```

##### 2. 添加apt插件

```xml
<plugin>
    <groupId>com.mysema.maven</groupId>
    <artifactId>apt-maven-plugin</artifactId>
    <version>1.1.3</version>
    <dependencies>
        <dependency>
            <groupId>com.querydsl</groupId>
            <artifactId>querydsl-apt</artifactId>
            <version>${querydsl.version}</version>
        </dependency>
    </dependencies>
    <executions>
        <execution>
            <phase>generate-sources</phase>
            <goals>
                <goal>process</goal>
            </goals>
            <configuration>
                <outputDirectory>target/generated-sources/queries</outputDirectory>
                <processor>org.springframework.data.mongodb.repository.support.MongoAnnotationProcessor</processor>
                <logOnlyOnError>true</logOnlyOnError>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### 执行mave编译命令

执行maven命令：

```shell
mvn clean compile
```

执行完毕后，在target/generated-sources/queries包下会生成__Q__开头的实体类。本文中即为**QSysOperateRecord**，该类是由maven插件自动生成的。插件会为带@Document注解的类自动生成QXXX。

```java
/**
 * QSysOperateRecord is a Querydsl query type for SysOperateRecord
 */
@Generated("com.querydsl.codegen.EntitySerializer")
public class QSysOperateRecord extends EntityPathBase<SysOperateRecord> {

    private static final long serialVersionUID = -643412538L;

    public static final QSysOperateRecord sysOperateRecord = new QSysOperateRecord("sysOperateRecord");

    public final DateTimePath<java.util.Date> createTime = createDateTime("createTime", java.util.Date.class);

    public final StringPath id = createString("id");

    public final BooleanPath isDelete = createBoolean("isDelete");

    public final DateTimePath<java.util.Date> lastUpdateTime = createDateTime("lastUpdateTime", java.util.Date.class);

    public final StringPath operateBefore = createString("operateBefore");

    public final StringPath operateCode = createString("operateCode");

    public final StringPath operateContent = createString("operateContent");

    public final NumberPath<Integer> operateId = createNumber("operateId", Integer.class);

    public final StringPath operateName = createString("operateName");

    public final StringPath operateUser = createString("operateUser");

    public final StringPath remark = createString("remark");

    public QSysOperateRecord(String variable) {
        super(SysOperateRecord.class, forVariable(variable));
    }

    public QSysOperateRecord(Path<? extends SysOperateRecord> path) {
        super(path.getType(), path.getMetadata());
    }

    public QSysOperateRecord(PathMetadata metadata) {
        super(SysOperateRecord.class, metadata);
    }

}
```

#### 使用方式

##### 1. xxxRepository继承QueryDslPredicateExecutor

```java
@Repository
public interface SysOperateRecordRepository extends MongoRepository<SysOperateRecord, String>, QueryDslPredicateExecutor<SysOperateRecord> {
}
```

QueryDslPredicateExecutor中提供了对Querydsl的支持。

##### 2. 使用举例

查询举例：

```java
public List<SysOperateRecord> queryBy(String operateCode, List<Integer> operateIds) {
    QSysOperateRecord qSysOperateRecord = QSysOperateRecord
        .sysOperateRecord;
    //通过链式的调用方式，组装查询条件
    BooleanExpression condition = qSysOperateRecord.operateCode.eq(operateCode)
        .and(qSysOperateRecord.operateId.in(operateIds))
        .and(qSysOperateRecord.isDelete.eq(false));
   //调用findAll方法，传入查询条件和排序方式，排序方式支持多字段排序
    Iterable<SysOperateRecord> records = sysOperateRecordRepository
        .findAll(condition, new QSort(qSysOperateRecord.createTime.desc()));
    return Lists.newArrayList(records);
}
```

分页查询举例：

```java
public List<SysOperateRecord> queryPageBy(String operateCode, List<Integer> operateIds, PageReq pageReq) {
    QSysOperateRecord qSysOperateRecord = QSysOperateRecord
        .sysOperateRecord;
    BooleanExpression condition = qSysOperateRecord.operateCode.eq(operateCode)
        .and(qSysOperateRecord.operateId.in(operateIds))
        .and(qSysOperateRecord.isDelete.eq(false));
    Iterable<SysOperateRecord> records = sysOperateRecordRepository
        .findAll(condition,
                 new QPageRequest(
                     pageReq.getPageNum(),
                     pageReq.getPageSize(),
                     new QSort(qSysOperateRecord.createTime.desc())));
    return Lists.newArrayList(records);
}
```



## 结语

本文介绍了spring项目如何集成mongodb，并简要分析了几种查询方式的优缺点。基本结论如下：

* 简单查询直接使用repository内置api即可
* 复杂查询推荐使用Querydsl