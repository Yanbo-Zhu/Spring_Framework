

- Swagger是一个规范和完整的框架，用于生成、描述、调用和可视化RESTfull风 格的Web服务，是非常流行的APl表达工具。
- Swaggeri能够自动生成完善的RESTful API文档，，同时并根据后台代码的修改 同步更新，同时提供完整的测试页面来调试API。

WebMvcConfigurationSupport，则在yml中配置的相关内容会失效。 需要重新指定静态资源
添加配置之后还是启动不了的需要手动在WebMvcConfigurer实现类中添加资源配置


# 1 配置

1 
在Spring BootI项目中集成Swaggerl同样非常简单，只需在项目中引入 springfox-swagger2 和 springfox-swagger-ui 依赖即可。

pom.xml 中加入 
```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
```

2 
Spring Boot2.6.X后与Swaggert有版本冲突问题，需要在 application.properties中加入以下配置：

```
spring.mvc.pathmatch.matching-strategy=ant_path_matcher
```


3  
配置 Swagger 
在config 包下 添加一个新的文件 SwaggerConfig.java 

```java
@Configuration//告诉Spring容器，这个类是一个配置类
@EnableSwagger2//启Swagger2功能
public class Swagger2Config {
    
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
            .apiInfo(apiInfo()) // 这里使用 apiInfo 是在下面定义的 
            .select()
            .apis(RequestHandlerSelectors.basePackage("com")) //com包下所有API都交给Swagger2管理, 会自动取读取 com 下面的所有的 controller 
            .paths(PathSelectors.any()).build();
    }
    
    //API文档页面显示信息
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
            .tite("演示项目API")   //标题
            .description("学习Swagger2的演示项目")   //描述
            .build();
    }
    
}

```




启动项目 访问 `http://127.0.0.1:8080/swaggger-ui.html` 就可以打开自动生成的可视化测试页面 
![](images/Pasted%20image%2020250222120422.png)

# 2 加入注解的语法

Swagger 提供了一系列的注解来添加接口信息, 包括接口说明, 请求方法, 请求参数 , 返回信息等等 

![](images/Pasted%20image%2020250222120614.png)


填入下面的@ApiOperation 这个注释 ,
![](images/Pasted%20image%2020250222120913.png)

在下面就会出现 获取用户 这几个字样 
![](images/Pasted%20image%2020250222120928.png)


