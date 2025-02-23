
MyBatis是一款优秀的数据持久层ORM框架，被广泛地应用于应用系统。 
MyBatis能够非常灵活地实现动态SQL,可以使用XML或注解来配置和映射原 生信息，能够轻松地将Java的POJO(Plain Ordinary Java Object,普通的 Java对象)与数据库中的表和字段进行映射关联。
MyBatis--Plus是一个MyBatis的增强工具，在MyBatis的基础上做了增强， 简化了开发。

需要注意mybatis-plus 3.5.3 才支持 spring boot 3
springboot 3.0出bug，mybatis-plus-boot-starter版本改用3.5.3.1
使用spring-boot3的小伙伴注意了！ 依赖是mybatis-plus-spring-boot3-starter！！！


# 1 配置

添加依赖 
```xml
<!--MyBatisP1us依赖 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.2</version>
</dependency>

<!--mysq1驱动依赖-->
<dependency>
    <groupId>mysq1</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
</dependency>

<!-- 数据连接池druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.20</version>
</dependency>

```


配置数据库相关信息。
```
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.driver-class-name=com.mysq1.jdbc.Driver
spring.datasource.url=jdbc:mysq1://localhost:3306/mydb?usessL=false
spring.datasource.username=root
spring.datasource.password=root
mybatis-plus.configuration.log-impl=org.apache.ibatis.logging.stdout.StdoutImpl
```

添加@MapperScan注解
```java
@SpringBootApplication
@Mapperscan("com.xx.mapper")   // mapper 包在何处
public class MybatisplusDemoApplication
public static void main(string[]args){
SpringApplication.run (MybatisplusDemoApplication.class,args);
```


# 2 CRUD 注解

![](images/Pasted%20image%2020250222195752.png)

```java
@Mapper
public interface UserMapper{
    @Insert("insert into user values (#id},#fusername},#password},#birthday})")
    int add(Useruser);

    @Update("update user set username=#{username},password=#{password},birthday=#{birthday}where id=#{id}")
    int update(User user);

    @Delete("delete * from user where id=#fid]")
    int delete(int id);

    @select("select * from user where id=#{id}")
    User findById(int id);
    
    @Select("select * from user")
    List<User>getAll();
}    
```




