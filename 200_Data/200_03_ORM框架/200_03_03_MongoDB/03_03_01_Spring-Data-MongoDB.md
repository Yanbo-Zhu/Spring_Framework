
文章是关于Java中使用SpringData-MongoDB框架接入MongoDB的全面指南，包括入门、进阶用法、MongoTemplate的使用、配置详解、库内分表方案等内容。介绍了实体类注解、自定义方法、分页查询、自定义语句、事务机制、MongoTemplate的方法分类与对象详解、聚合管道对象、匹配器对象、配置参数，还讲解了副本集和分片集群配置、多数据源整合以及按时间和数据量的库内分表方案，最后总结了四种CRUD模式。

https://juejin.cn/post/7276408879366111292#heading-1

---



我们都是高级语言的开发者，就如同`MySQL`一样，在不同语言中，都会有各自的驱动、`ORM`框架，`MongoDB`亦是如此，而在`Java`中如何使用`MongoDB`呢？总共有两种方案：

- ①`MongoDB-Driver`：官方提供的数据库驱动，可以基于此执行各种`MongoDB`支持的操作；
- ②`Spring-Data-MongoDB`：`Spring`对原生驱动的封装，提供了更简易的`API`。

通常来说，我们一般不会使用第一种方式操作`MongoDB`，类比`MySQL`，第一种方案就相当于原生的`JDBC`，而第二种方案就相当于`MyBatis/Plus、JPA、Hibernate`这种持久层框架。两者相比较，显然后者对开发者更加友好，也能在极大程度上提升开发效率，从而满足`Boss`的“快速开发”理念。

  
# 1 pom and application.yml 


pom 
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

这里使用的是spring-boot-starter，所以无需指定版本号，默认会跟SpirngBoot版本保持一致，我这里SpringBoot版本为2.7.15，


application.yml
```yaml
spring:
  data:
    mongodb:
      host: 192.168.229.136
      port: 27017
      database: zhuzi
      username: zhuzi
      password: 123456
      # uri: mongodb://zhuzi:123456@192.168.229.137:27017/zhuzi
      authentication-database: zhuzi
      auto-index-creation: on
    
```

下面解释一下每个参数的作用：

- `host`：部署`MongoDB`服务的机器`IP`；
- `port`：`MongoDB`服务的端口号，默认为`27017`；
- `database`：当前项目连接的数据库；
- `username`：连接`MongoDB`的账号；
- `password``MongoDB`账号的密码；
- `uri`：前五项的合集；
- `authentication-database`：认证的数据库（连接用的账号，不在连接的库下时使用）；
- `auto-index-creation`：是否自动创建索引的配置；

  

# 2 mvc 结构 



## 2.1 beispiel 1
1 entity
这里没啥太值得注意的，和MyBatisPlus也很相似，由于目前实体类名和集合名不一样，因此使用@Document显示指定一下集合名词，其次用@Id注解显示指定出主键。接着来看food属性，因为这是一个嵌入文档，所以咱们又定义了另外一个类Food，并将其作为了Panda类的一个属性，SpringData在映射时，会拿着food属性名+Food类里的属性名拼接，形成food.name这样的格式，正好和MongoDB嵌入文档的使用方式相同。
```java
@Data
@Document(collection = "xiong_mao")
public class Panda implements Serializable {
    private static final long serialVersionUID = 1L;
    @Id
    private Integer id;
    private String name;
    private Integer age;
    private String color;
    private Food food;
}

@Data
public class Food implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private String grade;
}

```


2 持久层
如你所见，持久层仅仅定义了一个接口，而后继承了SpringData提供的MongoRepository接口，并传递了两个泛型，前者代表与当前Repository绑定的实体类（集合），后者表示集合内的主键类型。除此之外，咱们什么都不用写，因为SpringData会通过动态代理的方式，帮我们生成基础的CRUD方法
```java 
@Repository
public interface PandaRepository extends MongoRepository<Panda, Integer> {}
```




3 业务层
接着来编写Service层，遵循传统的项目编码风格，先定义接口，再撰写实现类，接口定义如下：
```java
public interface PandaService {
    void save(Panda panda);
    void deleteById(Integer id);
    void update(Panda panda);
    Panda findById(Integer id);
    List<Panda> findAll();
}
```


其中定义了五个基本的增删改查方法，注释也没写，毕竟一眼就能看懂的代码，接着看看实现类：
```java
@Service
public class PandaServiceImpl implements PandaService {
    @Autowired
    private PandaRepository pandaRepository;

    @Override
    public void save(Panda panda) {
        pandaRepository.save(panda);
    }

    @Override
    public void deleteById(Integer id) {
        pandaRepository.deleteById(id);
    }

    @Override
    public void update(Panda panda) {
        pandaRepository.save(panda);
    }

    @Override
    public Panda findById(Integer id) {
        return pandaRepository.findById(id).get();
    }

    @Override
    public List<Panda> findAll() {
        return pandaRepository.findAll();
    }
}
```

又是一眼能看懂的代码，这里将前面定义的PandaRepository注入了进来，而后实现了PandaService接口的每个方法，每个方法中没有包含任何业务逻辑，只是单纯调了pandaRepository的默认方法，不过这里注意：update修改方法，调用的也是pandaRepository.save()方法实现，这是为什么呢？大家可以点进save()方法看看：
此时不难发现，这些方法最终会调用到CrudRepository接口提供的基本方法，而这个CRUD接口中，并没有提供update()系列的方法，而save()插入相同_id的数据时，会直接覆盖上一次的数据，为此，我们可以通过该方法间接实现修改功能，只不过每次修改之前，需要先find一次将原数据查询出来。

![](images/17d7023ce3004f0fa7d52dd3878e39e6~tplv-k3u1fbpfcp-jj-mark_3024_0_0_0_q75.webp)


4 测试案例 
该测试类中，将咱们定义的PandaService注入了进来，而后为每一个service方法写了用例，具体的执行结果就不贴了，诸位可以自己执行一下看看结果，其实和操作MySQL没有太大差异。

```java
@SpringBootTest
public class PandaServiceTest {
    @Autowired
    private PandaService pandaService;

    @Test
    void testSave() {
        Panda panda = new Panda();
        panda.setId(111);
        panda.setName("黑黑");
        panda.setColor("黑色");
        panda.setAge(8);
        Food food = new Food();
        food.setName("黑晶竹");
        food.setGrade("A");
        panda.setFood(food);
        pandaService.save(panda);
    }

    @Test
    void testDelete() {
        pandaService.deleteById(111);
    }

    @Test
    void testFindById() {
        Panda panda = pandaService.findById(111);
        System.out.println(panda);
    }

    @Test
    void TestUpdate() {
        Panda panda = pandaService.findById(111);
        panda.setAge(9);
        pandaService.update(panda);
    }

    @Test
    void testFindAll(){
        List<Panda> pandas = pandaService.findAll();
        pandas.forEach(System.out::println);
    }
}

```



## 2.2 Beispiel 2

https://juejin.cn/post/6844903554776317960?from=search-suggest

若MongoDB的安装配置采用默认端口，那么在自动配置的情况下，我们不需要做任何参数配置，就能马上连接上本地的MongoDB。下面直接使用spring-data-mongodb来尝试对mongodb的存取操作。（记得mongod启动您的mongodb）

创建要存储的User实体，包含属性：id、username、age

```java
import org.springframework.data.annotation.Id
import org.springframework.data.mongodb.core.mapping.Document

@Document
data class User(@Id var id: Long? = -1, var username: String = "", val age: Int? = 0)
```


---

实现User的数据访问对象：UserRepository

```java
import name.quanke.kotlin.chaper11_6_4.entity.User
import org.springframework.data.mongodb.repository.MongoRepository
import org.springframework.stereotype.Repository

/**
 * Created by http://quanke.name on 2018/1/11.
 */
@Repository
interface UserRepository : MongoRepository<User, Long> {
    fun findByUsername(username: String): User

}
```

通过上面的例子，我们可以轻而易举的对MongoDB进行访问，但是实战中，应用服务器与MongoDB通常不会部署于同一台设备之上，这样就无法使用自动化的本地配置来进行使用。这个时候，我们也可以方便的配置来完成支持，只需要在application.properties中加入mongodb服务端的相关配置，具体示例如下：



----

参数配置

在`application.yml`文件中增加

```java
spring:
  data:
    mongodb:
      uri: mongodb://192.168.2.53:27017/test
```

在尝试此配置时，记得在mongo中对test库创建具备读写权限的用户（用户名为name，密码为pass），不同版本的用户创建语句不同，注意查看文档做好准备工作

----

单元测试

```java

import name.quanke.kotlin.chaper11_6_4.entity.User
import name.quanke.kotlin.chaper11_6_4.repository.UserRepository
import org.apache.commons.logging.LogFactory
import org.junit.Test
import org.junit.runner.RunWith
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.junit4.SpringRunner


/**
 * Created by http://quanke.name on 2018/1/9.
 */
@RunWith(SpringRunner::class)
@SpringBootTest
class ApplicationTests {

    val log = LogFactory.getLog(ApplicationTests::class.java)!!

    @Autowired
    lateinit var userRepository: UserRepository

    @Test
    fun `redis string test"`() {
        userRepository.save(User(1L, "quanke", 30))
        userRepository.save(User(2L, "quanke.name", 40))

        log.info("数据总条数：${userRepository.count()}")

        log.info("用户名字为 quanke : ${userRepository.findByUsername("quanke")}")
    }

}
```





## 2.3 Beispiel 3

https://www.cnblogs.com/vipstone/p/9752623.html

Spring Boot项目集成MyBatis前两篇文章已经做了详细的介绍，这里就不做过多的介绍，本文重点来介绍MongoDB的集成。



Setp1：添加依赖

在pom.xml添加如下依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```


---

Setp2：配置MongoDB连接

在application.properties添加如下配置：

```bash
spring.data.mongodb.uri=mongodb://username:pwd@172.16.10.79:27019/testdb
```


---

Setp3：创建实体类

```java
import java.io.Serializable;

public class User implements Serializable {
    private Long id;
    private String name;
    private int age;
    private String pwd;
	//...略set、get
}

```


---



Setp4：创建Dao类

```java
import com.hello.springboot.entity.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Component;
import java.util.List;

@Component
public class UserDao {
    @Autowired
    private MongoTemplate mongoTemplate;
    /**
     * 添加用户
     * @param user User Object
     */
    public void insert(User user) {
        mongoTemplate.save(user);
    }

    /**
     * 查询所有用户
     * @return
     */
    public List<User> findAll() {
        return mongoTemplate.findAll(User.class);
    }

    /**
     * 根据id 查询
     * @param id
     * @return
     */
    public User findById(Long id) {
        Query query = new Query(Criteria.where("id").is(id));
        User user = mongoTemplate.findOne(query, User.class);
        return user;
    }

    /**
     * 更新
     * @param user
     */
    public void updateUser(User user) {
        Query query = new Query(Criteria.where("id").is(user.getId()));
        Update update = new Update().set("name", user.getName()).set("pwd", user.getPwd());
        mongoTemplate.updateFirst(query, update, User.class);
    }

    /**
     * 删除对象
     * @param id
     */
    public void deleteUserById(Long id) {
        Query query = new Query(Criteria.where("id").is(id));
        mongoTemplate.remove(query, User.class);
    }

}

```

---

Setp4：创建Controller

```java
import com.hello.springboot.dao.IndexBuilderDao;
import com.hello.springboot.dao.UserDao;
import com.hello.springboot.entity.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.ModelAndView;

@RestController
@RequestMapping("/")
public class UserController {
    @Autowired
    private UserDao userDao;

    @RequestMapping("/")
    public ModelAndView index() {
        User user = new User();
        user.setId(new Long(1));
        user.setAge(18);
        user.setName("Adam");
        user.setPwd("123456");
        userDao.insert(user);

        ModelAndView modelAndView = new ModelAndView("/index");
        modelAndView.addObject("count", userDao.findAll().size());
        return modelAndView;
    }
}

```


---


Setp5：创建页面代码

```html
<html>
<head>
    <title>王磊的博客</title>
</head>
<body>
Hello ${count}
</body>
</html>
```

到此为止已经完成了MongoDB的集成，启动项目，输入“[http://localhost:8080/”去数据库查看插入的数据吧。](http://localhost:8080/%E2%80%9D%E5%8E%BB%E6%95%B0%E6%8D%AE%E5%BA%93%E6%9F%A5%E7%9C%8B%E6%8F%92%E5%85%A5%E7%9A%84%E6%95%B0%E6%8D%AE%E5%90%A7%E3%80%82)

正常插入数据库如下图：



### 2.3.1 MongoDB主键自增

细心的用户可能会发现，虽然MongoDB已经集成完了，但插入数据库的时候user的id是手动set的值，接下来我们来看怎么实现MongoDB中的id自增。



MongoDB 实现id自增和Spring Boot JPA类似，是在数据库创建一张表，来记录表的“自增id”，只需要保证每次都增加的id和返回的id的原子性，就能保证id实现“自增”的功能。



创建实体类

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "IndexBuilder")
public class IndexBuilder {
    @Id
    private String id;
    private Long seq;
	//..省略get、set方法
}
```

其中collection = "IndexBuilder"是指数据库的集合名称，对应关系型数据库的表名。


 
 创建Dao类

```java
import com.hello.springboot.entity.IndexBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.MongoOperations;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Component;
import static org.springframework.data.mongodb.core.FindAndModifyOptions.options;
import static org.springframework.data.mongodb.core.query.Criteria.where;
import static org.springframework.data.mongodb.core.query.Query.query;

@Component
public class IndexBuilderDao {
    @Autowired
    private MongoOperations mongo;
    /**
     * 查询下一个id
     * @param collectionName 集合名
     * @return
     */
    public Long getNextSequence(String collectionName) {
        IndexBuilder counter = mongo.findAndModify(
                query(where("_id").is(collectionName)),
                new Update().inc("seq", 1),
                options().returnNew(true).upsert(true),
                IndexBuilder.class);
        return counter.getSeq();
    }
}
```



使用“自增”的id

```java
User user = new User();
user.setId(indexBuilderDao.getNextSequence("user"));
//...其他设置
```

核心代码：indexBuilderDao.getNextSequence("user") 使用“自增”的id，实现id自增。

到此为止，已经完成了MongoDB的自增功能，如果使用正常，数据库应该是这样的：




## 2.4 Beispiel 4 MongoPlus 

https://blog.csdn.net/lovexinxin_/article/details/132529486

pom 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.6.13</version>
        <relativePath/>
    </parent>

    <groupId>com.mongoplus.bolg</groupId>
    <artifactId>mongo-plus-bolg</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.gitee.anwena</groupId>
            <artifactId>mongo-plus-boot-starter</artifactId>
            <!-- 使用最新版 -->
            <version>2.0.6.2</version>
        </dependency>
    </dependencies>

</project>

```


yml配置文件 ↓↓↓↓↓↓↓
```
mongo-plus:
  data:
    mongodb:
      host: 127.0.0.1
      port: 27017
      database: blog
      #账号密码没有的话就不用写
	  #username: 
	  #password:
      connectTimeoutMS: 50000
      waitQueueTimeoutMS: 50000
  log: true
```


然后！你要有一个实体类！
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User {

	//使用ID注解，标识这是_id字段，执行type为ASSIGN_ID，使用雪花算法生成id
    @ID(type = IdTypeEnum.ASSIGN_ID)
    private String id;

    private String userName;

    private Integer userStatus;

    private Integer age;

    private Role role;

}
```



之后！你要有你的Service！并且将Service继承Iservice，实现类则继承ServiceImpl
```java
public interface UserService extends IService<User> {

}
```

```java
@Service
public class UserServiceImpl extends ServiceImpl<User> implements UserService {

}
```


MongoPlusMapMapper，就是为了Mongo集合结构比较复杂时使用的，会返回一个map结构，直接注入即可使用
接下来！开始测试~~~

首先编写一个测试类，进行save测试！！！ ↓↓↓↓↓↓↓

```java
@SpringBootTest
public class BlogTest {

	//注入我的service
    @Resource
    private UserService userService;

    @Test
    public void insertTest(){
    	//创建user对象
        User user = new User();
        user.setUserName("张三");
        user.setUserStatus(1);
        user.setAge(18);

        Role role = new Role();
        role.setRoleName("超级管理员");
        role.setRoleIntroduce("拥有最高权限");
        user.setRole(role);
        //直接使用继承IService提供的save方法，和mp一样！
        Boolean save = userService.save(user);
        System.out.println(save?"添加成功":"添加失败");      
    }
}
```



## 2.5 Beispiel 5

https://github.com/aalansehaiyang/technology-talk/blob/master/docs/md/spring/springboot/MongoDB.md

定义实体对象
@Id 主键 不可重复，自带索引，可以在定义的列名上标注，需要自己生成并维护不重复的约束。如果自己不设置@Id主键，mongo会自动生成一个唯一主键，并且插入时效率远高于自己设置主键。在实际业务中不建议自己设置主键，应交给mongo自己生成，自己可以设置一个业务id，如int型字段，用自己设置的业务id来维护相关联的表。
@Field 代表一个字段，可以不加，不加的话默认以参数名为列名。
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class User {
    @Id
    private Long id;
    private String name;
    private Integer age;
    @Field("locs")
    private Collection<Address> addresses = new LinkedHashSet<Address>();
}
```


数据访问层 UserRepository
UserRepository 接口继承 MongoRepository ，默认提供很多实现，比如 CRUD 和列表查询参数相关的实现。MongoRepository 接口默认实现如下：
```
public interface UserRepository extends MongoRepository<User, String> {

    User findByName(String name);

    /**
     * 小于：$lt
     * 小于或等于：$lte
     * 大于：$gt
     * 大于或等于：$gte
     * 不等于：$ne
     * 属于：$in
     */
    @Query("{'age': { '$lt' : ?0}}")
    List<User> withQueryFindByAge(Integer age);
}

```


## 2.6 Beispiel6 
https://blog.csdn.net/weixin_36755535/article/details/134530333

实体类
```java
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
 
@Document(collection = "users")
public class User {
    
    @Id
    private String id;
    private String username;
    private String email;
 
    // 省略构造函数、getter和setter等

```


```java
import org.springframework.data.mongodb.repository.MongoRepository;
 
public interface UserRepository extends MongoRepository<User, String> {
    // 可以在这里定义一些自定义的查询方法
}
```

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
 
@Service
public class UserService {
 
    private final UserRepository userRepository;
 
    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
 
    public void saveUser(User user) {
        userRepository.save(user);
    }
 
    public User getUserById(String id) {
        return userRepository.findById(id).orElse(null);
    }
 
    // 其他操作...
}
```

# 3 SpringData-MongoDB进阶


## 3.1 实体类注解

前面在定义实体类时，简单介绍了`@Id、@Document`两个注解，这里先全面介绍一些常用注解。

- `@Document`：作用于类上面，被该注解修饰的类，会和`MongoDB`中的集合相映射，如果类名和集合名不一致，可以通过`collection`参数来指定。
- `@Id`：标识一个字段为主键，可以加在任意字段上，但如果该字段不为`_id`，每次插入需要自己生成全局唯一的主键；如果不设置`@Id`主键，`MongoDB`会默认插入一个`_id`值来作为主键。
- `@Transient`：被该注解修饰的属性，在`CRUD`操作发生时，`SpringData`会自动将其忽略，不会被传递给`MongoDB`。
- `@Field`：作用于普通属性上，如果`Java`属性名和`MongoDB`字段名不一致，可以通过该注解来做别名映射。
- `@DBRef`：一般用来修饰“嵌套文档”字段，主要用于关联另一个文档。
- `@Indexed`：可作用于任意属性上，被该注解修饰的属性，如果`MongoDB`中还未创建索引，在第一次插入时，`SpringData`会默认为其创建一个普通索引。
- `@CompoundIndex`：作用于类上，表示创建复合索引，可以通过`name`参数指定索引名，`def`参数指定组成索引的字段及排序方式。
- `@GeoSpatialIndexed、@TextIndexed`：和上面的`@Indexed`注解作用相同，前者代表空间索引，后者代表全文索引。

好了，上面这些便是`SpringData-MongoDB`提供用于修饰实体类的注解，其实除此之外，还有些由`SpingData`提供的通用注解，但这里不再例举，毕竟上述这些已经够用了。

  
## 3.2 自定义方法

经过前面的案例，会发现`SpringData-MongoDB`提供了默认的`CRUD`方法，那如果现在有个需求，没有对应的默认方法怎么办？这时可以自定义方法，但`SpringData`是以方法命名来决定方法怎么执行的，规范如下：

- `findBy<fieldName>`：根据指定的单个条件进行等值查询；
- `findBy<fieldName>And<fieldName>And<...>`：根据指定的多条件进行`and`查询；
- `findBy<fieldName>Or<fieldName>Or<...>`：根据指定的多条件进行`or`查询；
- `findBy<fieldName>Equals`：根据指定的单个条件进行等值查询；
- `findBy<fieldName>In`：对指定的单个字段进行`in`查询，入参为一个列表；
- `findBy<fieldName>Like`：对指定的单个字段进行`like`模糊查询；
- `findBy<fieldName>NotNull`：查询指定字段不为空的数据；
- `findBy<fieldName>GreaterThan`：对指定的单个字段进行`>`范围查询；
- `findBy<fieldName>GreaterThanEqual`：对指定的单个字段进行`>=`范围查询；
- `findBy<fieldName>LessThan`：对指定的单个字段进行`<`范围查询；
- `findBy<fieldName>LessThanEqual`：对指定的单个字段进行`<=`范围查询；
- `Page<...> findBy<...>`：根据指定的条件进行分页查询；
- `countBy<fieldName>`：根据指定的条件字段进行计数统计；
- `findTop<n>By<fieldName>`：根据指定字段做等值查询，并返回前`n`条数据；
- `findBy<fieldName>Between`：根据指定字段进行`between`范围查询；
- `findDistinctBy<fieldName>`：根据指定的单个条件进行去重查询；
- `findFirstBy<fieldName>`：根据指定的单个条件进行等值查询（只返回满足条件的第一个数据）；
- `findBy<fieldName1>OrderBy<fieldName2>`：根据第一个字段做等值查询，并根据第二个字段做排序；
- `……`：

大家看下来不难发现，`SpringData-MongoDB`其实提供了一组命名约束，结合自定义的实体类字段，可以让`SpringData`帮我们生成各类执行语句，这里总体归类一下，在`Repository`中自定义方法，不同单词开头的含义如下：

- 以`get、find、read、query、stream`开头，代表是查询数据的方法；
- 以`count`开头，代表是计数统计的方法；
- 以`delete、remove`开头，代表是删除数据的方法；
- 以`exists`开头，代表是判断是否存在的方法；
- 以`search`开头，代表是全文搜索的方法；
- 以`update`开头，代表是修改数据的方法；

上面这些便是`SpringData-MongoDB`所有支持的方法类型，这些方法后面可以跟关键字，以`find`开头的方法为例：

- `By`：表示当前方法生成的查询语句，会根据`By`后面的逻辑来组成；
- `FirstBy`：表示当前方法生成的语句，只会返回符合条件的第一条数据；
- `DistinctBy`：表示当前方法生成的语句，会对符合条件的数据去重；
- `TopBy`：表示当前方法生成的语句，只会返回符合条件的前`N`条数据；
- `<实体类名称>By`：表示当前方法生成的语句，只会返回一条数据；
- `<实体类名称>sBy`：表示当前方法生成的语句，会返回多条数据；
- `AllBy`：表示当前方法生成的语句，会返回多条或所有数据；
- `DistinctFirstBy`：表示当前方法生成的语句，只会返回去重后的第一条数据；
- `DistinctTopBy`：表示当前方法生成的语句，只会返回去重后的前`N`条数据；

OK，在这些关键字之后，都是跟具体的字段名（实体类的属性名），字段名称后面可以接的关键字如下（同样以`find`为例）：

- `Or`：表示当前查询方法有多个条件，多个条件之间为“或者”关系；
- `And`：表示当前查询方法有多个条件，多个条件之间为“并且”关系；
- `OrderBy`：表示当前查询会涉及到排序，后面需要跟一个排序字段；
- `Between`：表示当前方法为`between`范围查询；
- `GreaterThan`：表示当前方法为`>`查询；
- `GreaterThanEqual`：表示当前方法为`>=`查询；
- `LessThan`：表示当前方法为`<`查询；
- `LessThanEqual`：表示当前方法为`<=`查询；
- `After`：和`GreaterThan`差不多，相当于查询指定数值之后的数据；
- `Before`：和`LessThan`差不多，查询指定条件之前的数据；
- `Containing`：查询某字段中包含指定字符的数据；
- `Empty`：表示当前方法会查询指定字段为空的数据，与之含义类似的还有`Null、Exists`；
- `Equals`：表示当前方法会根据指定字段做等值查询；
- `Is`：和`Equals`差不多；
- `In`：表示当前方法为`in`多值匹配查询；
- `Like`：表示当前方法为`like`模糊查询；
- `Not`：可以和上述大多数关键字组合，带有`Not`的则含义相反，如`NotEmpty`表示不为空；
- `……`

上面列出了大多数常用的连接关键字，看着是不是尤其的多？但其实不需要你死记，因为当你在`IDEA`工具编写方法时，它也会自动提示你！这里随便来定义几个方法，帮助诸位加深理解：

  
```java
@Repository
public interface PandaRepository extends MongoRepository<Panda, Integer> {
    // 查询指定年龄的熊猫数量
    Integer countByAge(Integer age);

    // 对熊猫姓名进行全模糊查询
    List<Panda> findByNameLike(String keyword);

    // 查询同时满足年龄、姓名两个条件的熊猫
    Panda findPandaByAgeAndName(int age, String name);

    // 查询满足颜色、年龄其中任一条件的所有熊猫
    List<Panda> findByColorOrAge(String color, int age);

    // 查询第一个带有颜色的熊猫
    Panda findFirstByColorNotNull();

    // 查询年龄大于等于指定岁数的熊猫
    List<Panda> findByAgeGreaterThanEqual(int age);

    // 对熊猫ID进行多值查询
    List<Panda> findByIdIn(List<Integer> ids);

    // 查询指定颜色的熊猫，并按照年龄降序返回
    List<Panda> findByColorOrderByAgeDesc(String color);

    // 查询年龄小于指定岁数的前三条数据
    List<Panda> findTop3ByAgeLessThan(int age);
}

```


## 3.3 分页查询

前面了解了`SpringDataMongoDB`中的自定义方法，接着来看看经典的“分页查询”，在这里该怎么玩呢？

首先需要定义一个`Repository`方法，如下：

```java
// 对年龄不为空的熊猫数据进行分页
Page<Panda> findByAgeNotNull(Pageable pageable);
```

命名约束完全相同，区别就在于返回类型，以及入参类型不同，这里的返回类型声明成了`Page<Panda>`，而入参列表中需要新增一个`Pageable`，接着继续封装一下`Service`层：

```java
// 在PandaService接口中定义分页方法
Page<Panda> pageByAgeNotNull(int pageNumber, int pageSize);

// 在PandaServiceImpl实现类中实现具体逻辑
@Override
public Page<Panda> pageByAgeNotNull(int pageNumber, int pageSize) {
    // 使用SpringData中内置的分页请求对象，直接将分页参数转换为请求对象
    PageRequest pageReq = PageRequest.of(pageNumber - 1, pageSize);
    // 调用Repository的分页方法
    return pandaRepository.findByAgeNotNull(pageReq);
}
```



写好`Service`层后，为了测试分页功能是否正常，这里再编写一个单元测试：

```java
@Test
void testPage(){
    Page<Panda> pandas = pandaService.pageByAgeNotNull(1, 3);
    int totalPages = pandas.getTotalPages();
    long totalElements = pandas.getTotalElements();
    System.out.println("总页数：" + totalPages);
    System.out.println("总行数：" + totalElements);
    System.out.println("第一页的数据为：");
    List<Panda> data = pandas.getContent();
    // 也可以直接遍历pandas这个Page对象
    data.forEach(System.out::println);
}
```

这个用例很简单，无非就是调用了一下`service`方法，而后传递了页码、条数，接着从`Page`对象中，拿到了总页数、总行数，以及具体的分页数据，输出结果如下：

```java
总页数：6
总行数：16
第一页的数据为：
Panda(id=1, name=肥肥, age=3, color=黑白色, food=null)
Panda(id=4, name=黑熊, age=3, color=黑白色, food=Food(name=黄金竹, grade=S))
Panda(id=5, name=白熊, age=4, color=null, food=Food(name=翠绿竹, grade=B))
```

效果很明显，总共有`16`条数据，每页三条总共`6`页，这里获取了第一页的数据，拿到的分页结果也符合`age`字段不为空的特性，当然，大家也可以自行改改页码测试，这里不再重复贴结果，毕竟分页使用起来尤为简单。


## 3.4 自定义MongoDB语句

前面掌握了遵循`SpringData`的命名规范，通过自定义`Repository`方法，完成了对`MongoDB`数据的增删改查操作，但这种方式有个缺陷，就是必须遵守规范去给方法命名，使用时难免会有种约束感，能不能自由一点呢？当然可以。

以前操作传统关系型数据库时，无论是在任何`ORM`框架中，都支持通过注解来编写简单`SQL`，例如`MyBatis`：

```java
@Select("select * from zhu_zi where id = #{id}")
ZhuZi selectById(@Param("id") Integer id);
```

那在`SpringDataMongoDB`中，支不支持这种注解形式定义语句呢？答案是当然可以，来看个例子：

```java
// Repository方法定义
@Query("{'name': ?0}")
List<Panda> queryXxx(String name);

// Service接口定义
List<Panda> queryXxx(String name);

// ServiceImpl接口实现
@Override
public List<Panda> queryXxx(String name) {
    return pandaRepository.queryXxx(name);
}
```

这个代码特别简单，重点看`repository`里定义的接口方法：`queryXxx()`，在方法上面有一个`@Query`注解，这用于自定义查询语句，其中声明根据`name`字段进行查询，`?0`表示方法参数的索引（占位符），此处的`0`表示第一个参数`name`，下面看看测试结果：

```java
// Junit测试用例
@Test
void testQueryXxx() {
    List<Panda> pandas = pandaService.queryXxx("肥肥");
    pandas.forEach(System.out::println);
}

/* 执行结果：
    Panda(id=1, name=肥肥, age=3, color=粉色, food=null)
*/
```

这里并没有遵循`SpringData`的命名规范，但仍然根据`name`字段查询到了数据，效果同预期中的一样，除此之外，还有另外几个注解，分别对应其他操作：

- `@Update`：用于自定义更新语句的注解；
- `@DeleteQuery`：用于自定义删除语句的注解；
- `@CountQuery`：用于自定义统计语句的注解；
- `@ExistsQuery`：用于自定义查询语句，但执行后只返回是否存在满足条件的数据，并不返回具体的文档；
- `@Aggregation`：用于自定义聚合管道语句的注解；
- `……`

除开上述列出的外，其实还有一些注解，具体大家可以去看看`org.springframework.data.mongodb.repository`这个包，关于这些注解的用法就不再过多说明，毕竟用注解写原生语句的机会并不多，实际开发中，要么根据命名规范自定义方法，要么使用`MongoTemplate`写复杂操作（后续会细说）。

## 3.5 SpingDataMongoDB事务机制

在之前的篇章中提到过，`MongoDB`本身支持事务，那`SpingDataMongoDB`中该如何使用事务呢？如下：

```java
// Service层定义接口
void mongoTransaction();

// ServiceImpl具体实现
@Override
public void mongoTransaction() {
    // 1.先通过mongoClient开启一个session会话
    ClientSession session = mongoClient.startSession();
    
    try{
        // 2.通过session开启事务
        session.startTransaction();
        
        // 3.创建一个实体对象
        Panda panda = new Panda();
        panda.setId(222);
        panda.setName("白白");
        panda.setColor("白色");
        panda.setAge(1);
        
        // 4.通过mongoClient获取集合对象
        MongoCollection<Document> collection = mongoClient
                .getDatabase("zhuzi")
                .getCollection("xiong_mao");
        // 5.通过集合对象提供的insert方法插入数据
        collection.insertOne(session, Document.parse(JSONObject.toJSONString(panda)));
        
        // 6.模拟执行异常
        int n = 100 / 0;
        
        // 7.如果执行到这里，说明执行没报错，提交事务
        session.commitTransaction();
    } catch (Exception e) {
        // 8.如果进入了catch，说明出现异常，回滚事务
        session.abortTransaction();
        e.printStackTrace();
    }
    // 9.关闭前面开启的session会话
    session.close();
}
```

OK，对于每一步代码的含义，这里已经在上面写出了注释，为此不再重复说明，整个流程很简单，上面是一个向`MongoDB`插入单个文档的方法，如果执行成功，提交事务；如果执行出现异常，回滚事务，但要注意几点：

- ①事务依赖于`ClientSession`开启，所以需要先打开`ClientSession`会话；
- ②想要让当前写操作的事务生效，必须调用`MongoCollection`的写方法；
- ③前两个对象的获取，都依赖于`MongoClient`对象，所以要记得注入该对象，如下：

```java
@Autowired
private MongoClient mongoClient;
```

接着来写个测试用例，看看事务到底会不会生效呢？如下：

```java
@Test
void testMongoTransaction() {
    pandaService.mongoTransaction();
}
```

按理来说，由于咱们手动加了一行`100/0`，因此肯定会报错而后回滚，但是来看看实际执行结果：

```java
Caused by: com.mongodb.MongoCommandException: 
    Command failed with error 20 (IllegalOperation): 
        'Transaction numbers are only allowed on a replica set member 
        or mongos' on server 192.168.229.136:27017. The full response is 
        {"ok": 0.0, "errmsg": "Transaction numbers are only allowed on a 
        replica set member or mongos", "code": 20, "codeName": "IllegalOperation"}
```

这个报错很明显，并不是取模零的异常，而是告诉咱们，事务只能在`replicaSet`副本集，或者`mongos`分片集群下生效，在单机的`MongoDB`上，事务无法正常开启，为此，在使用单机`MongoDB`时，事务并不能正常生效（如果你搭建了集群环境，上述代码是能正常执行、然后回滚数据的）。



  
# 4 MongoTemplate


前面的内容中，简单过了一下`SpringData`自定义方法、如何定义分页查询，以及如何通过注解自定义原生语句。不过更多时候，咱们都是在遵循着`SpringData`的命名规范编写，语句的生成、执行、数据聚合，全由`SpringData`自动完成。但，如果有一个特殊的需求，无法通过前面的方式查询到数据怎么办？这里就得用到`MongoTemplate`，可以用它来实现更加灵活的增删改查操作。

`MongoTemplate`中提供了不同的方法，用于执行不同类型的操作，首先来看看


## 4.1 插入方法

```java
// 插入一个文档（自动根据对象类型解析目标集合）
T insert(T objectToSave);

// 向指定的集合插入一个文档
T insert(T objectToSave, String collectionName);

// 批量插入多个文档
Collection<T> insertAll(Collection<? extends T> objectsToSave);

// 向指定的集合里批量插入多个文档
Collection<T> insert(Collection<? extends T> batchToSave, String collectionName);
Collection<T> insert(Collection<? extends T> batchToSave, Class<?> entityClass);

// 批量插入多个文档，并设置写关注级别
Collection<T> doInsertBatch(String collectionName, Collection<? extends T> batchToSave,
                            MongoWriter<T> writer);

// 这两个方法等价于前面的insert方法
T save(T objectToSave);
T save(T objectToSave, String collectionName);
```

观察下来会发现，这些方法对应着`MongoDB`自身提供的`API`，即`insertOne、insertMany`，如果入参`T`已经与一个集合映射，则无需额外指定类型、集合名，`MongoTemplate`会自动解析。但如果你想将一个字段完全不同的文档插入到某个集合，则可以显式指定集合名称。


## 4.2 删除方法

接着来看看删除方法：
删除方法命名中都带remove，可以直接传入一个对象进行删除，也可以根据条件进行删除，没啥好讲的，
```java
// 删除一个文档（根据入参类型自动解析目标集合）
DeleteResult remove(Object object);

// 从指定集合中删除一个文档
DeleteResult remove(Object object, String collectionName);

// 根据条件从指定集合中删除一个文档
DeleteResult remove(Query query, Class<?> entityClass);
DeleteResult remove(Query query, Class<?> entityClass, String collectionName);
DeleteResult remove(Query query, String collectionName);

// 根据指定条件删除文档，并返回删除前的文档（默认只对第一个满足条件的文档生效）
<T> T findAndRemove(Query query, Class<T> entityClass);
```


## 4.3 修改方法

修改方法有点多，所以这里省略了一些同名不同参的方法，其中只有updateMulti方法会更新所有满足条件的文档，其余方法都只会更新“满足条件”的第一个文档。为此，在使用时要额外注意：当你的操作涉及到多个文档修改时，请记住调用updateMulti()方法，而不是其他修改方法。


```java
// 更新指定的文档
ExecutableUpdate<T> update(Class<T> domainType);

// 更新满足条件的第一个文档（如果未找到满足条件的文档，则将当前数据插入到集合）
UpdateResult upsert(Query query, UpdateDefinition update, Class<?> entityClass);

// 更新满足条件的第一个文档
UpdateResult updateFirst(Query query, UpdateDefinition update, String collectionName);

// 更新满足条件的所有文档
UpdateResult updateMulti(Query query, UpdateDefinition update, Class<?> entityClass);

// 根据指定条件删除文档，并返回删除前的文档（默认只对第一个满足条件的文档生效）
T findAndModify(Query query, UpdateDefinition update, Class<T> entityClass);

// 根据指定条件替换文档，并返回删除前的文档（默认只对第一个满足条件的文档生效）
<S, T> T findAndReplace(Query query, S replacement, FindAndReplaceOptions options, 
        Class<S> entityType, String collectionName, Class<T> resultType);

```


## 4.4 查询方法


上面同样省略了一些同名的重载方法，以及某些不常用的方法，诸位可以简单瞟几眼。

```java
// 查询指定集合里是否存在满足条件的数据
boolean exists(Query query, String collectionName);

// 根据ID查询数据
T findById(Object id, Class<T> entityClass);

// 返回满足查询条件的单条数据
T findOne(Query query, Class<T> entityClass);

// 返回满足查询条件的多条数据
List<T> find(Query query, Class<T> entityClass);

// 查询指定集合中的所有文档
List<T> findAll(Class<T> entityClass, String collectionName);

// 返回满足条件的所有数据（并根据指定字段去重）
List<T> findDistinct(Query query, String field, Class<?> entityClass, Class<T> resultClass);

// 统计指定文档中，满足查询条件的文档数量
long count(Query query, String collectionName);

// 使用聚合管道查询数据
AggregationResults<O> aggregate(TypedAggregation<?> aggregation, Class<O> outputType);

```



## 4.5 聚合管道命令

这里同样省略了一些重载方法，不过这并不重要，执行聚合管道操作的方法分为两大类，
一类是同步执行的aggregate方法，即命令发给MongoDB服务端后，需要阻塞等待至MongoDB返回结果为止；
而另一类是支持响应式编程的aggregateStream方法，可以通过调用subscribe()方法或其他响应式编程操作来处理聚合结果（后面再细说）。

前面对MongoTemplate的常用方法有了一定认识后，下面来说说如何使用MongoTemplate。


```java
// 在指定集合中，执行aggregate聚合管道操作
AggregationResults<O> aggregate(Aggregation aggregation, 
                        String collectionName, Class<O> outputType)

// 支持异步Flux流（响应式编程）的聚合管道操作
CloseableIterator<O> aggregateStream(Aggregation aggregation, 
                        String collectionName, Class<O> outputType)

// 执行给定的aggregate聚合管道操作（这种方式无需传入集合名，而是采用Java映射类的形式）
AggregationResults<O> aggregate(TypedAggregation<?> aggregation, Class<O> outputType);

```


## 4.6 `Query` 和 `Criteria`

在上一节中，大家会看到许多入参类型为`Query`的方法，这到底是个啥东东？

用过`MyBatisPlus`框架的小伙伴一定不陌生，在`MP`中查询、修改、删除数据时，如果不想编写`xml`文件，咱们可以通过`Warpper`条件构造器，以`Java`代码的形式指定`where`条件，从而快捷、方便的实现条件修改、查询、删除功能。

而`SpringData-MongoDB`，`Query、Criteria`对象的作用亦是如此，通过`Java`对象的方式，完全取代掉`MongoDB`原生的`shell`语法，比如你想实现一个条件查询，可以这样写：

```java
// Servcie接口方法定义
List<Panda> findByColor(String color);

// ServiceImpl中的具体实现
public List<Panda> findByColor(String color) {
    Query query = new Query();
    query.addCriteria(Criteria.where("color").is(color));
    return mongoTemplate.find(query, Panda.class);
}
```

这样就实现了按照颜色字段来查询数据的效果，当然，上面的查询条件还可以简写为：

```java
Query query = Query.query(Criteria.where("color").is(color));
```

这样指定的条件，效果和前面`new`的方式相同，来看看效果：

```java
@Test
void testFindByColor(){
    List<Panda> pandas = pandaService.findByColor("黑白色");
    pandas.forEach(System.out::println);
}

/* 输出结果如下：
    Panda(id=1, name=肥肥, age=3, color=黑白色, food=null)
    Panda(id=2, name=花花, age=null, color=黑白色, food=null)
    Panda(id=4, name=黑熊, age=3, color=黑白色, food=Food(name=黄金竹, grade=S))
*/
```

从结果中可以看出，和我们预期中的效果相同，的确实现了按颜色查询集合文档的效果。

  
### 4.6.1 Query查询对象详解

经过上述小案例的学习后，各位应该大致对`SpringData`中的条件构造器有了基本认知，接着来详细说说`Query`对象，各方法的作用如下：

```java
// 定义条件查询对象
Query.query(CriteriaDefinition criteriaDefinition);

// 在现有的Query对象上，新增一个条件对象
addCriteria(CriteriaDefinition criteriaDefinition);

// 在现有查询对象上，新增一个Sort排序对象
with(Sort sort);

// 在现有查询对象上，新增一个Pageable分页对象
with(Pageable pageable);

// 查询结束后，跳过指定的前skip条数据
skip(long skip);

// 查询结束后，限制返回的行数
limit(int limit);

// 查询结束后，限制返回的字段（投影查询）
Field fields();
```

好了，`Query`类中实则定义了几十个方法，但较为常用的则是上面列出的这几个，通常咱们理解上面这几个方法的作用即可，下面结合前面的案例，简单过一下这些方法：

```java
// 一、定义Service接口方法
List<Panda> findByColorAndAge(String color, int age);

// 二、编写Impl子类实现
@Override
public List<Panda> findByColorAndAge(String color, int age) {
    // 1. 通过query()方法快速定义带条件的查询对象
    Query query = Query.query(Criteria.where("color").is(color));
    // 2. 通过addCriteria()方法，在现有的查询对象上新增一个条件
    query.addCriteria(Criteria.where("age").is(age));
    // 3. 通过with()方法添加一个排序对象（基于_id降序）
    query.with(Sort.by(Sort.Direction.DESC, "_id"));
    // 4. 通过skip()方法跳过结果集中的第一条数据
    query.skip(1);
    // 5. 通过limit()方法限制返回的行数为一条
    query.limit(1);
    // 6. 通过fields()方法指定返回的字段
    query.fields().include("_id", "name", "age");
    return mongoTemplate.find(query, Panda.class);
}

// 三、撰写单元测试用例
@Test
void testFindByColorAndAge(){
    List<Panda> pandas = pandaService.findByColorAndAge("黑白色", 3);
    pandas.forEach(System.out::println);
}
```

这个案例中，几乎将前面列出的方法都用上了，接着来看看结果：

```java
/* 集合内符合查询条件的数据如下：
    Panda(id=1, name=肥肥, age=3, color=黑白色, food=null)
    Panda(id=4, name=黑熊, age=3, color=黑白色, food=Food(name=黄金竹, grade=S))
*/

/* 输出结果如下：
    Panda(id=1, name=肥肥, age=3, color=null, food=null)
*/
```

来根据代码推导一下过程，首先基于给定的条件，会找到`id=1、4`这两条数据；其次会基于`id`降序，结果集变为`4、1`这个顺序；然后回通过`skip`跳过第一条数据，即`id=4`这条数据；接着通过`limit`限制返回一条数据，拿到`id=1`这条数据；最后通过`fields`指定返回的字段，因为没有要求返回`color、food`，所以最终结果集相应字段为`null`。


---


好了，这个结果和咱们预期中的完全相同，不过上面这些案例中，又涉及到了一些新对象，如`Sort、Field`等，这些对象也会有相应`API`，这里简单过一下，先来看看`Sort`类：

```java
// 仅有的构造器，使用private关键字修饰
private Sort(Direction direction, List<String> properties);

// 对外提供by()方法来创建实例对象
public static Sort by(String... properties);
public static Sort by(List<Order> orders);
public static Sort by(Order... orders);
public static Sort by(Direction direction, String... properties);

// 快速创建升序、降序对象的方法
public Sort descending();
public Sort ascending();

// 默认的排序方式（升序）
public static final Direction DEFAULT_DIRECTION;
static {
    DEFAULT_DIRECTION = Sort.Direction.ASC;
}
```

这里咱们重点关注`by()`方法即可，排序主要有两个入参，一个是排序方式，另一个是排序字段，如果调用不指定排序方式的`by`方法，默认使用`DEFAULT_DIRECTION`升序方式。再者，我们可以直接指定这两个参数，也可以封装成`Order`对象，`Order`内部类如下：

```java
public static class Order implements Serializable {
    // 排序方式
    private final Direction direction;
    // 排序字段
    private final String property;
    
    // 使用默认的升序方式，为传入的字段构建Order对象
    public static Order by(String property) {
        return new Order(Sort.DEFAULT_DIRECTION, property);
    }
    // 使用升序方式为传入的字段构建Order对象
    public static Order asc(String property) {
        return new Order(Sort.Direction.ASC, property, DEFAULT_NULL_HANDLING);
    }
    // 使用降序方式为传入的字段构建Order对象
    public static Order desc(String property) {
        return new Order(Sort.Direction.DESC, property, DEFAULT_NULL_HANDLING);
    }
    // 判断一个Order对象是否为升序
    public boolean isAscending() {
        return this.direction.isAscending();
    }
    // 判断一个Order对象是否为降序
    public boolean isDescending() {
        return this.direction.isDescending();
    }
    
    // 省略其他成员与方法……
}
```

`Order`类并不难懂，无非是对排序方式、排序字段的封装，用起来也差不多，只不过在使用时，需要先构建一个`Order`对象，而后再传入到`Sort`对象中，如下：

```java
query.with(Sort.by(Sort.Order.desc("_id")));
```


好了，另一个`Direction`类没啥好讲的，就是一个枚举类，里面就定义了`ASC、DESC`两个枚举，下

---


面来看看`Filed`类，该类主要用于投影查询，即调用`Query.fileds()`方法时产生的对象，如下：

```java
// Query.fileds()方法的定义
public Field fields();

public class Field {
    // 指定本次查询需要返回的字段
    public Field include(String... fields);
    // 指定本次查询需要排除的字段
    public Field exclude(String... fields);
    // 用于数组型字段，offset代表切片起始位置，size表示要获取的元素数量
    public Field slice(String field, int offset, int size);
    // 返回数组字段中，指定下标的元素值
    public Field position(String field, int value);
    // 用于再次过滤数组/嵌套文档中的数据
    public Field elemMatch(String field, Criteria elemMatchCriteria);
    // 省略其他不常用方法......
}
```

上面列出了`Filed`类中的常用方法，这里主要关心前面两个方法，一个是指定需要返回的字段列表，另一个是指定需要排除的字段列表，如下：

```java
// 指定返回name、age字段，并强制将_id字段排除
query.fields().include("name", "age").exclude("_id");
```

而对于后面的几个方法，都是用于数组/嵌套类型的字段，一般情况下使用较少，为此，这里不再举例说明，下面重点来看看`Criteria`条件构造器对象。


### 4.6.2 Criteria条件对象详解

在`MongoTemplate`中提供的大部分增删改方法，其中都会有一个`Query`类型的入参，而`Query`对象无法指定过滤条件，因此需要结合`Criteria`对象来指定条件，大家可以把`Criteria`理解成`MyBatisPlus`框架的`Warpper`对象，当然，它们两者在使用上也相差不大。

在前面的案例中，咱们曾写过这样的代码：

```java
Query query = Query.query(Criteria.where("color").is(color));
query.addCriteria(Criteria.where("age").is(age));
```

其实这两者都是在指定过滤条件，最终都会转变成一个`Criteria`对象传入到`Query`对象中，下面来看看其中的方法：

```java
// 过滤条件的起始方法（类似于SQL里的where关键字）
public static Criteria where(String key);

// 用于连接多个条件（and并且关系）
public Criteria and(String key);

// 匹配所有与指定值相等的文档（=）
public Criteria is(Object value);

// 匹配所有指定字段为空的文档（is null）
public Criteria isNull();

// 匹配所有不等于指定值的文档（!=）
public Criteria ne(@Nullable Object value);

// 匹配所有小于指定值的文档（<）
public Criteria lt(Object value);

// 匹配所有小于、等于指定值的文档（<=）
public Criteria lte(Object value);

// 匹配所有大于指定值的文档（>）
public Criteria gt(Object value);

// 匹配所有大于、等于指定值的文档（>=）
public Criteria lt(Object value);

// 匹配所有与指定的多个值、其中一个相等的文档（in）
public Criteria in(Object... values);

// 匹配所有与指定的多个值、不相等的文档（not in）
public Criteria nin(Object... values);

// 匹配所有与指定的多个值、完全相等的文档（用于数组字段）
public Criteria all(Object... values);

// 匹配指定字段是否存在的所有文档（true：存在，false：不存在）
public Criteria exists(boolean value);

// 可以与上面大多数方法配合使用，含义为取反
public Criteria not();

// 匹配所有符合正则表达式的文档（可用于实现like查询）
public Criteria regex(String regex);

// 连接多个条件的方法（or或者关系）
public Criteria orOperator(Criteria... criteria);

// 连接多个条件的方法（and并且关系）
public Criteria andOperator(Criteria... criteria);

// 连接多个条件的方法（nor非或关系）
public Criteria norOperator(Criteria... criteria);

// 省略其他方法.......
```

上面列出了`Criteria`类中大多数常用方法，其他一些要么属于操作嵌套文档、地理空间的方法，要么就比较小众冷门的方法，所以不再详细罗列。其实大家观察这些方法的命令，应该也能摸清楚大致含义，无非就是原生语法，转变成了`Java`代码形式，前面自定义`Repository`方法也是相同的含义，将各种条件表达式，以“方法命名规范”的形式来编写。

毕竟这里提供的方法，和前面的命名规范有很多共通之处，因此就随便来个例子练习一下：

```java
// 查询所有age>=3，并且food.grade=S、B的文档
Criteria criteria = Criteria.where("age")
                            .gte(3)
                            .and("food.grade")
                            .in("S", "B");
Query query = Query.query(criteria);
List<Panda> pandas = mongoTemplate.find(query, Panda.class);
pandas.forEach(System.out::println);

/* 输出结果如下：
    Panda(id=5, name=白熊, age=4, color=null, food=Food(name=翠绿竹, grade=B))
    Panda(id=4, name=黑熊, age=3, color=黑白色, food=Food(name=黄金竹, grade=S))
    Panda(id=12, name=金熊, age=4, color=null, food=Food(name=黄金竹, grade=S))
* */
```

这个结果和代码没啥好说的，一眼就能看懂。不过这里注意：之前咱们写的例子，几乎每一步都是分开写的，但要注意，`SpringDataMongoDB`中的大多数方法，都支持函数式编程的链式调用风格，即可以一直`.`下去，不需要每行代码之间用`;`分割。




## 4.7 Update修改对象

前面讲完了`Query、Criteria`两个类之后，接着来详细说说`Update`对象，大家对这玩意儿或许有点陌生，毕竟前面都没出现过，对不？其实该对象主要作用于“修改”方法，可仔细观察`MongoTemplate`提供的修改方法，貌似入参都为`UpdateDefinition`类型呀！这里大家可以仔细去看看`Update`类的定义：


```java
public class Update implements UpdateDefinition {}
```

从类头能明显看出，`UpdateDefinition`是个接口，而`Update`则是具体的实现类，所以到这里就清楚了，调用`MongoTemplate`提供的修改方法时，其实我们需要传递的是`Update`对象，这里先来个案例简单使用一下：

```java
// Service接口中定义：根据ID修改年龄的方法
UpdateResult updateColorById(Integer id, String color);

// Impl类中的具体实现
@Override
public UpdateResult updateColorById(Integer id, String color) {
    Query query = new Query(Criteria.where("id").is(id));
    Update update = new Update().set("color", color);
    return mongoTemplate.updateMulti(query, update, Panda.class);
}

// 测试用例
@Test
void testUpdateAgeById() {
    Panda oldPanda = pandaService.findById(1);
    System.out.println("修改前的数据：" + oldPanda);
    
    UpdateResult result = pandaService.updateColorById(1, "粉色");
    System.out.println("受影响的行数：" + result.getModifiedCount());

    Panda newPanda = pandaService.findById(1);
    System.out.println("修改前的数据：" + newPanda);
}
```
  
上面这个案例，代码也特别容易看明白，上面总共有两个对象：`Query、Update`，其中`Query`对象用于承载过滤条件，而`Update`对象则用来承载要修改的数据，来看看输出结果：

```java
/*
    修改前的数据：Panda(id=1, name=肥肥, age=3, color=黑白色, food=null)
    受影响的行数：1
    修改前的数据：Panda(id=1, name=肥肥, age=3, color=粉色, food=null)
*/
```


从修改前后的数据对比来看，本次修改的确生效了，而且最关键的一点是：**本次修改并没有触发全量替换，而是动态修改，即只变更了修改字段的值，其他字段没有变为`Null`**！这是因为`MongoTemplate`的`update`系列方法，底层使用了`$set`操作符。同时注意，`Update`系方法的返回值，都为`UpdateResult`对象，该对象主要有四个常用方法：

- `getMatchedCount()`：获取满足条件的文档数；
- `getModifiedCount()`：获取本次修改操作影响的行数；
- `getUpsertedId()`：如果本次修改操作为`upsert`操作，返回未匹配到数据、插入数据后生成的`ID`；
- `wasAcknowledged()`：判断本次`MongoDB`服务端执行完成后，其返回值是否正确



---


好了，大概认识了`Update`对象的用法后，接着来看看其中提供的方法：


```java
// 将指定字段的值，修改为给定值
public static Update update(String key, @Nullable Object value);

// 将指定字段的值，修改为给定值
public Update set(String key, @Nullable Object value);

// 将指定字段的值，修改为给定值（只有在指定字段为null的情况下生效）
public Update setOnInsert(String key, @Nullable Object value);

// 删除指定字段的值
public Update unset(String key);

// 对指定字段的值进行自增（前者代表+1，后者代表+inc）
public void inc(String key);
public Update inc(String key, Number inc);

// 向指定的字段，插入当前时间
public Update currentDate(String key);

// 向指定的字段，插入当前时间戳
public Update currentTimestamp(String key);

// 向指定的数组字段末尾，追加单个元素（可以插入重复值）
public Update push(String key, @Nullable Object value);

// 向指定的数组字段末尾，追加多个元素（可以插入重复值）
public Update pushAll(String key, Object[] values);

// 从指定的数组字段中，移除指定下标的元素
public Update pop(String key, Position pos);

// 从指定的数组字段中，移除单个数组元素
public Update pull(String key, @Nullable Object value);

// 从指定的数组字段中，移除多个数组元素
public Update pullAll(String key, Object[] values);

// 向数组中添加一个值，如果该值在数组中已存在，则忽略本次插入动作
public Update addToSet(String key, @Nullable Object value);

//  向数组中添加多个值，如果插入的某个值在数组中已存在，自动忽略相应值
public AddToSetBuilder addToSet(String key);
// 上述方法必须结合AddToSetBuilder.each方法使用，如addToSet("names").each(...)
public Update each(Object... values);

// 根据条件过滤数组内的元素（结合set("filed.$[element]", "...")方法一起使用）
public Update filterArray(CriteriaDefinition criteria);

// 将指定字段的名称，修改为新的字段名
public Update rename(String oldName, String newName);
```

OK，上面同样未曾将所有方法列出，仅仅只写了一些常用方法，其实如果你的集合中，不涉及到数组类型的字段，那只需要关注前半部分方法即可。当然，这里只说明了每个`API`的作用，想练习的小伙伴可以自行去玩一玩，不会使用直接问`ChatGPT`即可。


## 4.8 Aggregation聚合管道对象

在[《MongoDB入门》](https://juejin.cn/post/7272730422735929396#heading-11 "https://juejin.cn/post/7272730422735929396#heading-11")这一篇文章中，曾经提到过一点，如果你想实现较为复杂的操作，几乎都需要通过聚合管道来完成，`MongoDB`提供的聚合管道十分强大，几乎可以满足任何场景下的业务操作，而在`Java`中如何使用聚合管道呢？答案是`Aggregation`对象。


先从之前的篇章，`cpoy`过来一个原生的聚合管道命令，如下：

```JavaScript
/* 按年龄进行分组，并统计各组的数量（没有age字段的数据统计到一组） */
db.xiong_mao.aggregate([
    // 1：通过$group基于age分组，通过$sum实现对各组+1的操作
    {$group: {_id:"$age", count: {$sum:1}}},
    // 2：基于前面的_id（原age字段）进行排序，1代表正序
    {$sort: {_id:1}}
]);
```



这个案例比较简单，先基于年龄分组，再基于`_id`字段排序，如何转换成`Java`代码实现呢？如下：

```Java
@Test
void testAggregation() {
    Aggregation aggregation = Aggregation.newAggregation(
            // 基于年龄字段分组，接着统计每组梳理，并为统计字段取别名
            Aggregation.group("age").count().as("count"),
            // 基于分组后的_id字段（原age）字段排序（升序）
            Aggregation.sort(Sort.Direction.ASC, "_id")
    );
    AggregationResults<Map> results = mongoTemplate.aggregate(aggregation, "xiong_mao", Map.class);
    List<Map> maps = results.getMappedResults();
    maps.forEach(System.out::println);
}
/* 执行结果：
    {_id=null, count=3}
    {_id=1.0, count=3}
    {_id=2.0, count=4}
    {_id=3.0, count=6}
    {_id=4.0, count=2}
    {_id=9, count=1}
*/
```


这里为了节省篇幅，就不再写`service`层代码了，直接在`Test`方法中写逻辑，上面的代码大家参考注释即可，这里重点说说语法。认真阅读过[《入门篇》](https://juejin.cn/post/7272730422735929396#heading-11 "https://juejin.cn/post/7272730422735929396#heading-11")的小伙伴应该还记得，`MongoDB`的聚合管道，可以指定多个的阶段，在每个阶段中进行不同的处理，而上述代码中亦是如此，不过该如何指定不同的阶段呢？大家看到中间的`,`了嘛？`newAggregation()`方法可以传入任意个`Aggregation`对象，每个对象则代表一个阶段。

同时，`MongoDB`原生的语法中，并没有提供`$count`统计操作符，想要实现计数，只能以`$sum:1`这种方式实现，而`SpringData`则封装了一个`count()`方法提供给咱们使用，可谓是十分贴切~

  
接着看看返回结果：`AggregationResults`对象，这个对象需要指定一个泛型，该泛型意味着返回结果的类型，我这里传入了`Map`，这是因为我懒得再次封装一个对象出来，如果大家以后有些聚合操作要经常执行，并且返回结果的字段已经确定，最好的做法还是封装一个`entity`类出来。这点先放一遍，先说说`AggregationResults`里的常用方法：

- `getMappedResults()`：返回映射的聚合结果集，即得到指定类型的集合（如案例中的`List<Map>`）；
- `getRawResults()`：返回未做映射处理的原始结果集，则获取`MongoDB`返回的`BSON`对象；
- `getUniqueMappedResult()`：返回映射后的一个结果对象，在明确清楚聚合结果只有一条的情况使用；
- `iterator()`：返回用于遍历聚合结果的迭代器，可以用于逐个处理聚合结果中的数据；

对于`AggregationResults`对象而言，咱们只要明白这几个方法的作用即可，至于其他没列出的并不重要。


### 4.8.1 Aggregation案例练习


好了，上阶段对`SpringData`聚合管道有了简单认知后，那聚合管道的其他方法呢？怎么使用？其实把`MongoDB`原生的聚合管道命令掌握后，在`Java`里的意思也是相同的，毕竟方法名字都没改，和操作符的名称一模一样，下面来些案例玩玩：

1 
这个案例不难，相信大家一定能看懂，这里顺便演示了一下获取原生`BSON`对象，而后转换为`JSON`字符串输出的过程。

```java
// 案例一：基于颜色分组，并求出每组中年龄最大值，最后按年龄降序排列
@Test
void testAggregation1() {
    Aggregation aggregation = Aggregation.newAggregation(
            Aggregation.group("color").max("age").as("max_age"),
            Aggregation.sort(Sort.Direction.DESC, "max_age")
    );
    AggregationResults<Map> results = mongoTemplate.aggregate(aggregation, "xiong_mao", Map.class);
    Document bson = results.getRawResults();
    System.out.println(bson.toJson());
}
/* 执行结果：
    {
    	"results": [{
    		"_id": "黑色", "max_age": 9
    	}, {
    		"_id": null, "max_age": 4.0
    	}, {
    		"_id": "粉色", "max_age": 3.0
    	}, {
    		"_id": "黑白色", "max_age": 3.0
    	}],
    	"ok": 1.0
    }
*/
```

2 

这个案例需求是我瞎编的，没有任何逻辑可言，目的在于帮大家熟悉复杂场景下的`API`用法，代码的含义可参考注释，重点来说说执行结果，这个结果有些小伙伴可能看的有点懵，其实特别简单。`_id`代表原本的`food.grade`食物等级，而`index`则是原文档在各分组中的下标，因为原文档按顺序被压入了`pandas`数组，为啥会有重复值呢？

很简单，因为这个下标是以“分组”来区分的，上述`B`分组只有一个文档，所以`index=0`，`A`分组中有三个文档，所以`index=0、1、2`，其他分组以此类推……，这样解释后，相信大家一定能看懂这个执行结果。

到这里，关于`SpringData`中如何使用聚合管道的案例，就暂且告一段落了，如果大家感觉还没有尽兴，可以自行回到[《入门篇-聚合管道查询》](https://juejin.cn/post/7272730422735929396#heading-11 "https://juejin.cn/post/7272730422735929396#heading-11")阶段，将其中每个案例用`Java`代码实现一遍~


```java
/* 案例二：过滤掉food为空，以及age小于3岁的数据，接着按food.grade分组，
           并求出每组的平均年龄、以及输出每组第一个、最后一个、所有熊猫姓名，
           最后按照平均年龄升序排列 */
@Test
void testAggregation2() {
    // 1.组装过滤条件，food字段必须存在，并且age大于等于3
    Criteria criteria = Criteria.where("food").exists(true).and("age").gte(3);
    Aggregation aggregation = Aggregation.newAggregation(
            // 2.过滤阶段：传入构建好的条件对象
            Aggregation.match(criteria),
            // 3.分组阶段：按food.grade食物等级分组
            Aggregation.group("food.grade")
                    // 3.1. 求出age字段的平均值
                    .avg("age").as("avg_age")
                    // 3.2.拿到每组的第一个熊猫姓名
                    .first("name").as("first_name")
                    // 3.3.拿到每组的最后一个熊猫姓名
                    .last("name").as("last_name")
                    // 3.4.将所有熊猫姓名push到names数组
                    .push("name").as("names"),
            // 4.排序阶段：将分组后的数据，按平均年龄升序排列
            Aggregation.sort(Sort.Direction.ASC, "avg_age")
    );
    AggregationResults<Map> results = mongoTemplate.aggregate(aggregation, "xiong_mao", Map.class);
    List<Map> maps = results.getMappedResults();
    maps.forEach(System.out::println);
}
/* 执行结果：
    {_id=C, avg_age=3.0, first_name=粉熊, last_name=粉熊, names=[粉熊]}
    {_id=S, avg_age=3.5, first_name=黑熊, last_name=金熊, names=[黑熊, 金熊]}
    {_id=B, avg_age=4.0, first_name=白熊, last_name=白熊, names=[白熊]}
    {_id=A, avg_age=5.0, first_name=棕熊, last_name=黑黑, names=[棕熊, 紫熊, 黑黑]}
*/

```



## 4.9 ExampleMatcher匹配器对象

前面讲完了聚合管道，接着来说说匹配器，这是`SpringData`家族都有的对象，用于在匹配查询条件时对查询对象进行更细粒度的匹配配置，我们可以通过它自定义匹配的行为，好比排除某些字段、只匹配某些字段等。

可能按上面这么说，估计大家也不太明白这玩意儿的作用，那么先上个简单例子，大家看完自然而然就明白了，如下：

```java
@Test
void testExampleMatcher() {
    Panda panda = new Panda();
    panda.setName("黑黑");
    ExampleMatcher matcher = ExampleMatcher.matching()
            .withIgnoreCase()
            .withMatcher("name", ExampleMatcher.GenericPropertyMatchers.contains());
    Example<Panda> example = Example.of(panda, matcher);
    Criteria criteria = new Criteria().alike(example);
    List<Panda> pandas = mongoTemplate.find(Query.query(criteria), Panda.class);
    pandas.forEach(System.out::println);
}
/* 执行结果：
    Panda(id=111, name=黑黑, age=9, color=黑色, food=Food(name=黑晶竹, grade=A))
*/

```


上面是个使用`ExampleMatcher`的案例，从结果来看，无非就是基于`name`字段做查询，最终查到了“黑黑”这条数据，重点来看看匹配器的定义：


```java
ExampleMatcher matcher = ExampleMatcher.matching()
        .withIgnoreCase()
        .withMatcher("name", ExampleMatcher.GenericPropertyMatchers.contains());

```


这是在定义匹配规则，`withIgnoreCase()`表示忽略大小写，`withMatcher()`表示具体的匹配逻辑，对于`name`字段的条件，会以`contains()`包含规则去匹配，大家可以尝试把案例中的“黑黑”替换为“黑”，执行时照样能匹配到“黑黑”这条数据，因为“黑黑”包含了“黑”。


---


经过上述解释，相信诸位应该大致清楚`ExampleMatcher`的作用了吧？它可以针对每个字段，自定义细粒度的匹配规则，`ExampleMatcher`提供了三个方法用于创建匹配器：

```java
  // 创建一个默认的ExampleMatcher实例（底层调用了matchingAll()方法）
static ExampleMatcher matching();
// 创建一个匹配所有属性（字段）的ExampleMatcher实例
static ExampleMatcher matchingAll();
// 创建一个匹配任意属性（字段）的ExampleMatcher实例
static ExampleMatcher matchingAny();

```

----

下面再来看看ExampleMatcher中，自定义匹配规则的方法：
```java
// 为指定字段设置自定义的匹配规则
ExampleMatcher withMatcher(String propertyPath, GenericPropertyMatcher genericPropertyMatcher);
// 为字符串设置自定义的匹配规则
ExampleMatcher withStringMatcher(StringMatcher defaultStringMatcher);

// 设置匹配时忽略大小写
ExampleMatcher withIgnoreCase();
// 设置匹配时包含空值字段
ExampleMatcher withIncludeNullValues();
// 设置匹配时忽略空值字段
ExampleMatcher withIgnoreNullValues();
// 为指定字段设置“字段值转换器”
ExampleMatcher withTransformer(String propertyPath, PropertyValueTransformer
                                propertyValueTransformer);
// 设置匹配时要排除的字段值
ExampleMatcher withIgnorePaths(String... ignoredPaths);

```

---

OK，上面这些方法简单了解一下，看`withMatcher()`方法的入参，是一个`GenericPropertyMatcher`类型，而该类中提供了一些内置规则方法，比如前面的`contains()`包含规则，其他方法如下：

```java
// 设置匹配时，指定字段的值，必须包含给定值
public GenericPropertyMatcher contains();
// 设置匹配时，指定字段的值，必须以给定值开头
public GenericPropertyMatcher startsWith();
// 设置匹配时，指定字段的值，必须以给定值结尾
public GenericPropertyMatcher endsWith();
// 设置匹配时，指定字段的值，必须与给定值完全匹配
public GenericPropertyMatcher exact();
// 设置匹配时，指定字段的值，必须符合给定的正则表达式
public GenericPropertyMatcher regex();
// 设置匹配时，指定字段的值会区分大小写
public GenericPropertyMatcher caseSensitive();
// 设置匹配时，指定字段的值不区分大小写
public GenericPropertyMatcher ignoreCase();
```

加上这些注释后，大家是不是就能很好理解了？其实整体很简单，就是一种“另类”的条件查询，可以针对每个字段指定多条匹配规则。

好了，关于这玩意儿就此打住，其实`ExampleMatcher`用的也不是特别多，大家在这里了解即可，因为前面讲到的其他查询方式，足以满足日常开发中的所有需求，匹配器只有在某些特殊的业务场景下，才会使用。


# 5 MongoTemplate例子

https://juejin.cn/post/7073273393528700964#heading-43
## 5.1 集合的应用

1 
`mongoTemplate.createCollection("production");`
如果这个集合原本就存在，会报错，提示 Collection already exists. NS: test.production 的错误；

```java
// 创建集合
@Test
public void Times() {
    if (!mongoTemplate.collectionExists("production")) {
        mongoTemplate.createCollection("production");
    } else {
        System.out.println("集合已存在;");
    }
}

```


2 
mongoTemplate.dropCollection("production");

在MongoDB中删除不同于创建，这个语句可以不进行是否存在的判断，可以重复删除多次；（如果不存在也可以删除！）

### 5.1.1 相关注解

- `@Document`
    - 修饰范围：用在类上；
    - 作用：用来映射这个类的一个对象作为MongoDB中一条数据；
    - 属性：（`value`, `collection`）用来指定操作的集合名称；
- `@ID`
    - 修饰范围：用在成员变量、方法上；
    - 作用：用来将成员变量的值映射为 `_id` 的值；
- `@Field`
    - 修饰范围：用在成员变量、方法上；
    - 作用：用来将成员变量以及值映射为文档中的一个键值对；
    - 属性：（`name`, `value`）用来指定在文档中 key 的名称，**默认为成员变量名**；
- `@Transient`
    - 修饰范围：用在成员变量、方法上；
    - 作用：用来指定改成员变量，不参与文档的序列化；

  
## 5.2 文档

### 5.2.1 文档的添加
```java
    // 文档的添加
    @Test
    public void addDocument() {
        User user = new User(888, "UrbaneH", 88888, new Date());
//        mongoTemplate.save(user); // _id存在时会把旧数据进行覆盖；
        mongoTemplate.insert(user); // _id存在时会提示主键重复的异常；
    }

```

save 与 insert都是保存操作，都可以进行保存，但是不完全相同；
⚠️注意:
插入重复数据时：insert 会报DuplicateKeyException 提示主键重复。而 save会将重复的数据进行覆盖；
批量处理操作时：insert可以一次性插入整个数据，效率很高。而 save需要遍历整个数据，一次性插入或更新，效率较低（只能用for循环一条一条插入）；

```java
List<User> users = Arrays.asList(
		new User(1, "hahaha", 7768, new Date()),
		new User(2, "今天吃什么", 77777, new Date()),
		new User(3, "好饿", 12345, new Date())
	);
mongoTemplate.insert(users, User.class);

```

在插入批量数据时，需要两个参数，第一个参数是需要插入的对象，第二个参数是集合的名称，之所以写User.class是因为在这个文件内，已经写名了 @Document("production") 声明了集合；


### 5.2.2 文档的查询

查询所有 
在findAll中有两个参数，第一个参数是必选的，是返回的类的对象，第二个参数是可选的，是需要获取数据的集合名称。其实在类中已经定义了；

```java
// 文档的查询
@Test
public void queryDocument() {
		List<User> all = mongoTemplate.findAll(User.class);
		List<User> all = mongoTemplate.findAll(User.class, "production");
		all.forEach(System.out::println);
	}

```


基于ID查询
查询id为888；
```java
User byId = mongoTemplate.findById(888, User.class);
System.out.println(byId);
```

条件查询
如果传入的是一个空的Query对象则返回此集合中所有的数据；
```java
List<User> users = mongoTemplate.find(new Query(), User.class);
users.forEach(System.out::println);
```



等值查询
这一部分又个很智能的地方，如果之前使用 @Field修改过成员对象的名字，它也会根据注解来自动转换，无需特别修改；
比方说：
之前修改name的别名为username，那么在where内无论是 name 还是 username都是有效的操作，结果相同；
```java
	@Test
	public void queryDocument() {
		Query query = Query.query(Criteria.where("name").is("UrbaneH"));
		List<User> users = mongoTemplate.find(query, User.class);
		users.forEach(System.out::println);
	}

```


AND查询
⬇️ 查询name的值为UrbaneH且id为888；
    只需在where后面添加 and即可；
    ⚠️注意:    一定要注意字段的类型；

```java
		@Test
    public void queryDocument() {
        Query query = Query.query(Criteria.where("name").is("UrbaneH").and("_id").is(888));
        List<User> users = mongoTemplate.find(query, User.class);
        users.forEach(System.out::println);

    }

```



OR查询
⬇️ 查询name的值为好饿或者今天吃什么；

⚠️注意:
使用OR不同于使用AND的方法一样
1⃣️ 需要先实例化一个 Criteria 对象；
2⃣️ 直接接一个 orOperator	 方法；
3⃣️ 在 orOperator 内添加多个 Criteria ，每一个都属于一个条件（也可在每个条件的where后面直接加 and 进行 and操作）；

```java
		@Test
    public void queryDocument() {
        Criteria criteria = new Criteria();
        criteria.orOperator(
                Criteria.where("name").is("好饿"),
                Criteria.where("name").is("今天吃什么")
        );
        Query query = Query.query(criteria);
        List<User> users = mongoTemplate.find(query, User.class);
        users.forEach(System.out::println);
    }

---
---
---
User(id=2, name=今天吃什么, salary=77777.0, birthday=Sun Mar 06 18:05:12 CST 2022)
User(id=3, name=好饿, salary=12345.0, birthday=Sun Mar 06 18:05:12 CST 2022)

```


#### 5.2.2.1 排序

⬇️ 根据salary进行升序排列；

1⃣️ 创建一个Query对象；
2⃣️ 使用 with 连接，调用Sort.by；
3⃣️ 在Sort.by中调用 Sort.Order.asc("字段名");

⚠️注意:
asc：表示升序排列；
desc：表示降序排列；

```java
// 文档的查询
    @Test
    public void queryDocument() {
        Query query = new Query().with(Sort.by(Sort.Order.asc("salary")));
        List<User> users = mongoTemplate.find(query, User.class);
        users.forEach(System.out::println);
    }
--- 
---
---
User(id=1, name=hahaha, salary=7768.0, birthday=Sun Mar 06 18:05:12 CST 2022)
User(id=3, name=好饿, salary=12345.0, birthday=Sun Mar 06 18:05:12 CST 2022)
User(id=2, name=今天吃什么, salary=77777.0, birthday=Sun Mar 06 18:05:12 CST 2022)
User(id=888, name=UrbaneH, salary=88888.0, birthday=Sun Mar 06 17:38:33 CST 2022)
User(id=8888, name=UrbaneH, salary=88888.0, birthday=Sun Mar 06 17:41:51 CST 2022)

```


#### 5.2.2.2 分页

⬇️ 每页分为三个，查询第一页（从0开始，第一页即为0）；

```java
@Test
    public void queryDocument() {
        Query query = new Query().with(Sort.by(Sort.Order.asc("salary")));
        query.limit(3).skip(0);
        List<User> users = mongoTemplate.find(query, User.class);
        users.forEach(System.out::println);
    }
---
---
---
User(id=1, name=hahaha, salary=7768.0, birthday=Sun Mar 06 18:05:12 CST 2022)
User(id=3, name=好饿, salary=12345.0, birthday=Sun Mar 06 18:05:12 CST 2022)
User(id=2, name=今天吃什么, salary=77777.0, birthday=Sun Mar 06 18:05:12 CST 2022)
```

> 只是在之前排序的基础上使用 `limit` 和 `skip` 即可；
> `limit` ：表示每页分为多少条数据；
> `skip`：表示当前展示第几页；


#### 5.2.2.3 查询总条数

⬇️ 查询总条数，如果有条件可以在Query内添加需要的条件；

```java
System.out.println(mongoTemplate.count(new Query(), User.class));
```

#### 5.2.2.4 去重

⬇️ 去除指定条件的重复数据，条件如空则为去除所有；

```java
mongoTemplate.findDistinct(new Query(), "name", User.class, String.class);
```

> ✒️可以填写四个参数：
> 第一个参数是指定条件，就是需要去重的数据需要满足的条件，如果为空则是全部数据；
> 第二个参数是字段名，需要给哪一个字段去重；
> 第三个参数是文档的对象；
> 第四个参数是去重字段的数据类型，比如字符串就是String.class、整型就是Integer.class；


### 5.2.3 使用原生写法

**如果更加适应原生写发，mongoTemplate也提供了内置的方法；**

⬇️ 是查询name为UrbaneH的数据；

```java
		@Test
    public void queryDocument() {
        Query query = new BasicQuery("{name: 'UrbaneH'}");
        mongoTemplate.find(query, User.class).forEach(System.out::println);
    }
```

> 只需要使用Query的子类BasicQuery在其中写入原生命令即可；



如果需要返回指定字段，可以直接下在一个参数写入相应的原声命令即可

```java
new BasicQuery("{name: 'UrbaneH'}"， "{name: 1}");
```

### 5.2.4 文档的更新

1 更新第一条数据

⬇️ 更新用户UrbaneH的工资为1后面好多0和生日为现在时间；

```java
    @Test
    public void updateDemo() {
        // 更新第一条；
        Update update = new Update();
        update.set("salary", 10000000000L)
                .set("birthday", new Date());
        System.out.println(mongoTemplate.updateFirst(new Query(
                Criteria.where("name").is("UrbaneH")
        ), update, User.class));
    }
```

> 1⃣️ 使用 `mongoTemplate.updateFirst` ；
> 2⃣️ 确定需要更新的数据的条件；
> 3⃣️ 选择更新的数据 `Update` 对象，需要使用set传入一个键值对进行对数据的修改，可以连接多个set；
> 4⃣️ 传入实体类对象；



2  更新多条数据
**使用方法与上面的一直，只不过把调用的方法换成了 `mongoTemplate.updateMulti`；**


3  插入更新
⬇️ 更新name为我吃饱了的工资，如果不存在则插入数据；

```java
    @Test
    public void updateDemo() {
        mongoTemplate.upsert(
                new Query(
                        Criteria.where("name").is("我吃饱了")
                ),
                new Update()
                        .set("salary", 7758521),
                User.class
        );
    }
```

如果需要更新的数据查询不到，则会插入此条数据；

> **⚠️注意:**
> 插入的数据只会有查询的条件以及更新的条件，例如上面所举的🌰，只会插入带有默认id以及name和salary的数据；
> 但是如果通过SpringBoot读取后，还是会生成指定的对象，但是没有数据的值为 `null`；
> 如果要指定id可以使用 `.setOnInsert("_id", 具体id)`

  
### 5.2.5 文档的删除

```java
    @Test
    public void updateDemo() {
        mongoTemplate.remove(new Query(
                Criteria.where("name").is("我吃饱了")
        ), User.class).sout;
    }
```

> 用法与查询基本相同
> 如果查询条件为空则为删除所有的数据；
> 如果有条件则删除指定条件的数据；






# 6 配置 


## 6.1 副本集配置

```yml
mongodb://username:password@[host:port1],[host:port2],[....]/database?options

uri: mongodb://zhuzi:123456@192.168.229.137:27018,192.168.229.137:27019,192.168.229.137:27020/zhuzi?connect=replicaSet&slaveOk=true&replicaSet=zhuzi

```

其中的参数说明如下：

- `connect`：连接模式，指定为`replicaSet`代表连接副本集群；
- `slaveOk`：从节点是否可读，为`true`表示可读，执行语句时，读操作自动发往从节点；
- `replicaSet`：副本集群的名称，这里为`zhuzi`；

将`uri`换成上述这个格式后，就配置好了副本集连接参数，并且`SpringData`会自动根据执行的操作，将请求分发到不同类型的节点，从而实现读写分离的效果（大家可以自行去执行原本的操作，来检测配置是否生效）。

## 6.2 分片集群配置

在上篇中还提到了分片集群，那`SpringData`中如何连接分片集群呢？这里语法类似，但不需要跟`option`参数，只需要配置所有`mongos`所在的`IP`、端口即可，如下：

```yml
uri: mongodb://zhuzi:123456@192.168.229.137:27024,192.168.229.137:27025/zhuzi
```

上篇中，`27024、27025`被咱们做成了路由节点，为此，这里配置这两个节点即可，配置完成后，`SpringData`会自动去将请求交给`mongos`节点，再由`mongos`根据配置的分片键、分片算法，将请求分发到具体的分片节点（具体测试大家可以自行去玩玩）。



## 6.3 多数据源整合

不知大家是否还记得，之前曾说过：**虽然`MongoDB`支持事务，但其事务功能存在一定缺陷**，那假设咱们实际开发过程中，某些业务又特别依赖事务怎么办？这是最好的做法是：**将这些事务依赖项强的数据，还是存储在传统的关系型数据库里**！

啥意思呢？就比如`MySQL、MongoDB`一起使用，数据量较大、事务要求不高的数据，放入到`MongoDB`中；而事务要求较高、增量不算太快的业务数据，则依旧放在`MySQL`里面，怎么整合呢？其实很简单，把`MySQL`的驱动、连接池依赖引入进来，而后和之前一样配置即可，

而后就可以实现多数据源操作啦，要存到MySQL的数据，就用之前的MyBatis、JPA、Hibernate等框架，实体类、Dao层按之前的逻辑编写。而要存到MongoDB中的数据，就按本文的方式编写实体类、repository层即可。
  
```xml
<!-- MySQL驱动 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.28</version>
</dependency>
<!-- 连接池依赖 -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>4.0.3</version>
</dependency>

```


```yml
spring:
  datasource:
    url: jdbc:mysql://.....
    username: zhuzi
    password: zhuzi
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.zaxxer.hikari.HikariDataSource
    hikari:
      minimum-idle: 0
      maximum-pool-size: 20
      idle-timeout: 10000
      auto-commit: true
      connection-test-query: SELECT 1

```


# 7 库内分表方案


好了，上一阶段几乎都是简单的配置，没有任何难点，因此没有刻意展开讲述，下面来看一个特殊场景，假设现在采用`MongoDB`存储客服的聊天记录，其他数据全部存到`MySQL`，假设这里每天有`10W`个客户会话，每个客户平均的消息对话数量为`50`条，稍微算算每日数据的增量：

> `100000 * 50 = 5000000`（五百万条）

单日暴增`500W`数据，虽然`MongoDB`号称大数据存储组件，但目前仅有一个业务的数据存在里面，直接上分片集群显然大材小用了，该怎么存储？在之前的`MySQL`专栏，其实出过一章：[《库内分表篇》](https://juejin.cn/post/7169098754719481864 "https://juejin.cn/post/7169098754719481864")，用来讲述该如何解决这类问题，同样的思想也可以用到`MongoDB`上，即：**按照一定维度对数据进行分表（分集合）**。


## 7.1 数据拆分思路

常规的做法有两种，一种是按时间分，另一种是按数据量来划分，想要将数据分发到同一个库、不同集合中，最关键的在于如何计算出：**一条数据究竟该落入哪个集合？后续操作时又能精准拿到**？这个问题解决后，其他就不再是问题，下面展开讲讲。

相较于`MySQL`做库内分表，`MongoDB`实现起来的难度低许多许多，`Why`？大家回想一下前面`MongoTemplate`提供的方法：

```java
insert(T objectToSave, String collectionName);
remove(Query query, String collectionName);
updateMulti(Query query, UpdateDefinition update, String collectionName);
find(Query query, String collectionName);
findAll(Query query, String collectionName);
......
```

大家观察这些方法的最后一个参数，会明显发现，增删改查方法都支持指定集合名称！也就意味着，我们只需要计算出集合名词，就能实现“库内分表”。同时，`MongoDB`由于其无模式特性，咱们也不需要提前创建表结构，每当数据量增长到一定程度时，就能自动使用一个新集合来存储。

  
当然，如果之前没阅读过[《库内分表篇》](https://juejin.cn/post/7169098754719481864 "https://juejin.cn/post/7169098754719481864")的小伙伴，看起来或许会有点懵懂，因此建议大家可以去看看前面那篇文章，下面开始实操，这里先给出一个公用的实体类：

```java
@Data
public class ZhuZi {
    private Long id;
    private String name;
}
```



## 7.2 按照时间划分


按时间维度来分表（专业称呼叫分集合，但分表比较顺口，后续都以分表来称呼），首先得以时间字段作为分片键，这里也可以为时间戳，所以咱们首先来编写一个`ID`生成器，主要用于生成时间戳类型的`ID`：

```java
public class IdGenerator {
    /*
     * 获取时间戳ID（生产换成雪花ID，雪花ID包含时间戳，具备全局唯一性）
     * */
    public static long getTimeStampId() {
        return System.currentTimeMillis();
    }
}
```

这个主键生成器尤为简单，就是获取了一下当前的系统时间戳，但这样做并不专业，我仅仅是为了演示，图个方便，更好的做法是使用雪花`ID`算法，生成出来的`ID`会包含时间戳，并且可以保证全局唯一性（大家不想自己写，可以直接去拷贝`MyBatisPlus`里的雪花`ID`生成类）。



好了，有了上述这个简陋的`ID`生成器后，接着再来写动态分表的策略类，首先定义一个接口：

```java
// 动态分表策略接口
public interface DynamicCollection<T> {
    /*
    * 获取目标集合名的方法
    * */
    String getTargetCollectionName(T criteria);
}
```

这个接口实则非常简单，里面就定义了一个“获取目标集合（表）名”的方法，为啥要定义这个接口呢？因为可能会存在多种分片策略，为此，先抽象出一个公共的接口来定义分片行为。再来看方法的入参，它是一个泛型，这里主要用于指定分片键的类型，下面来看具体的实现：


```java
// 根据日期时间分表的实现类
public class DateTimeDynamicCollection implements DynamicCollection<Long> {
    // 日期格式化对象
    private static final ThreadLocal<SimpleDateFormat> monthTL =
            ThreadLocal.withInitial(() ->
                    new SimpleDateFormat("yyyyMMdd"));
    // 集合名的前缀
    private static final String collectionPrefix = "zhuzi_";

    /**
     *  根据时间戳获取目标集合名（每天的数据归档到一个集合）
     * @param timeStamp 时间戳
     * @return 集合名
     */
    @Override
    public String getTargetCollectionName(Long timeStamp) {
        Date date = new Date(timeStamp);
        return collectionPrefix + (monthTL.get().format(date));
    }
}
```

这是一个根据日期时间分表的实现类，入参为一个时间戳，接着内部会将时间戳转为`yyyyMMdd`格式的字符串，最后拼接上对应的前缀名，然后对外返回出“目标表名”，相当于按“天”分表，是不是特别简单？答案是`Yes`，毕竟这只是个`Demo`，旨在讲清如何实现动态分表。

  
接着来编写测试用例，如下：

```java
@Autowired
private MongoTemplate mongoTemplate;

@Test
void testDynamicCollectionByDateTime(){
    ZhuZi zhuZi = new ZhuZi();
    long id = IdGenerator.getTimeStampId();
    zhuZi.setId(id);
    zhuZi.setName("竹子一号");
    
    // 这里最好做成工具类或者单例Bean，使用时不需要new，而是直接用
    DynamicCollection<Long> dynamic = new DateTimeDynamicCollection();
    mongoTemplate.insert(zhuZi, dynamic.getTargetCollectionName(id));
}
```

这个用例特简单，先获取了一下当前时间戳作为主键，接着再通过生成的`ID`，得到了要插入的目标表名，最后通过`mongoTemplate.insert()`方法，将这条数据插入到了对应的集合中，下面去`MongoDB`看看：

![时间分表-1](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f34135f2458416bbde98e3e8c0b195a~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1616&h=646&s=55642&e=png&b=010101)

如上图所示，`MongoDB`中的确多了一个`zhuzi_20230918`的集合，并且“竹子一号”这条数据，也成功插入到了该集合中，

  
---

接着再来换一个时间戳试试看，如下：

```java
@Test
void testDynamicCollectionByDateTime() throws ParseException {
    ZhuZi zhuZi = new ZhuZi();
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    long timeStamp = sdf.parse("2023-11-11 11:11:11").getTime();
    zhuZi.setId(timeStamp);
    zhuZi.setName("竹子二号");

    DynamicCollection<Long> dynamic = new DateTimeDynamicCollection();
    mongoTemplate.insert(zhuZi, dynamic.getTargetCollectionName(zhuZi.getId()));
}
```

这里咱们手动创建了一个时间为`2023-11-11 11:11:11`的`Date`对象，并将其转变为了时间戳，接着依旧和之前代码相差不大，直接来看结果：

![时间分表-2](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a484b3bee344216a8cec0fe2ae8a823~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1529&h=656&s=54557&e=png&b=010101)

效果十分明显，此时又多出了一个`zhuzi_20231111`集合，并且“竹子二号”也被插入到了其中，此时动态分表的目的已然达成，

---


那其他操作又该怎么办呢？比如查询、修改、删除……，很简单，基于目前的分片键：`id`，去对数据进行操作即可，例如查询：

```java
// 这里复制了之前插入的时间戳
long id = 1699672271000L;
DynamicCollection<Long> dynamic = new DateTimeDynamicCollection();
String target = dynamic.getTargetCollectionName(id);
Query query = Query.query(Criteria.where("_id").is(id));
ZhuZi zhuzi = mongoTemplate.findOne(query, ZhuZi.class, target);
System.out.println(zhuzi);

/* 执行结果：
    ZhuZi(id=1699672271000, name=竹子二号)
*/
```

效果很明显，查询到了“竹子二号”这条数据，好了，对于修改、删除操作就不再演示，过程都类似，记住这里的核心思想：**每次操作必须传入分片键，因为需要通过分片键来计算表名，从而才能操作对应的表，如果没有分片键字段，就只能对所有集合全部执行一次相应操作**。

以上述案例来说，如果要基于`name`字段查询数据，就无法解析出“目标集合名”，只能对所有集合发起查询，而后再对数据进行汇总，如果集合数量过多，假设有`100`个，此时的执行效率堪忧，而这样的问题，在`Sharding-Sphere`这样的框架中，为了提升性能，底层会采用多线程去并发查询，大家也可以借鉴这点思想，个人不再继续完善。



## 7.3 按数据量划分

前面讲了按时间分表，现在再来说说业内另一种常见的分片策略，即按数据量划分，例如一千万条数据为一张表，怎么做呢？首先你的数据中，必须存在一个记录当前行数的值，这里可以选用自增`ID`来实现，毕竟自增`ID`天生具备这个功能，为此，这里再在之前的`ID`生成器中新增一个方法：

```java
// 原子长整型自增器
private static final AtomicLong ATOMIC_LONG = new AtomicLong(1);

/*
* 获取自增ID（生产换成特定的原子累加器，如MySQL自增、Redis自增）
* */
public static long getIncrId() {
    return ATOMIC_LONG.incrementAndGet();
}
```

这里依旧是`Demo`模式，专业做法并不能依靠`AtomicLong`来维护自增序列，因为服务一旦重启，这个计数器会清零重新开始，就会和之前自增生成的序列产生冲突，所以最好选用第三方中间件来维护自增序列，如数据库、`Redis……`啥的都行。

咱们目前不考虑这么多，能出效果就行，这里再写一个按数量量分片的策略实现类：

```java
// 根据数据量分表的实现类
public class DataVolumeDynamicCollection implements DynamicCollection<Long> {
    // 集合名的前缀
    private static final String collectionPrefix = "zhuzi_";

    /**
     * 根据自增ID获取目标集合名（每一千万条数据归档到一个集合）
     * @param incrValue 自增ID
     * @return 集合名
     */
    @Override
    public String getTargetCollectionName(Long incrValue) {
        return collectionPrefix + (incrValue / 10000000);
    }
}
```

又是一个特别简单的实现过程，这里会通过外部生成的自增`ID`，对`1000W`求商（相除），因为`Java`的`/`运算，并不会得到余数，所以这里会返回一个整数，例如：

```java
1 / 10000000 = 0;
10000001 / 10000000 = 1;
20000001 / 10000000 = 2;
```

而后直接与定义好的前缀拼接，从而得到的目标表名。当然，这里的表前缀名是完全一样的，最好的做法是将这个重复的定义，抽象为一个常量类，或者枚举类，方便于后续维护，并且能减少冗余代码，感兴趣的小伙伴可以自己去改造一下~

OK，下面直接来写单元测试，看看效果：

  
代码不解释了，和之前近乎一模一样，唯一区别就在于换了实现类，直接上效果图：

![数据量分表-1](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96475364166d4f94a981cff838ea1a2c~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1675&h=895&s=42146&e=png&b=010101)

可以看到，这里又多出了一个`zhuzi_0`集合，并且“一号竹子”也被插入到了其中，至于其他测试就不再重复叙述了，大家可以试着手动分配一个值为`10000001L`的`ID`，同样能观察到`zhuzi_1`集合的出现……。与前面的时间分片策略类似，基于数据量分片后，无论任何操作都需要传入分片键字段，如果不传递，则会导致无法计算出目标表名……。

> PS：这也是数据分片存在的通病，不管是传统的分库分表，还是例如`TiDB`这样的分布式数据库，又或者上篇所提到的`MongoDB`分片集群，都存在这样的缺陷，一旦数据被分片存储，操作数据时就必须传入分片键，否则只能对所有分片节点都执行一遍操作。

  






