

# 1 spring-boot-devtools,

Springboot 提供 spring-boot-devtools, 使得不需要手动重启 springBoot 就可以重新 编译 


## 1.1 开发热部署 Hot Swap

![](images/Pasted%20image%2020250220145725.png)

1 
![](images/Pasted%20image%2020250220145836.png)

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>true</scope>
</dependency>
```

2 
![](images/Pasted%20image%2020250220150028.png)

```properties
# warm-startup the springboor project  
spring.devtools.restart.enabled=true  
  
# setup which directory to restart  
spring.devtools.restart.additional-paths=src/main/java  
  
# the files in the directory under classpath don't need to be restarted  
spring.devtools.restart.exclude=static/**
```


3 

![](images/Pasted%20image%2020250220150625.png)

Turn on the flag: File > Settings > Advanced Settings > **Allow auto-make to start even if developed application is currently running**

![](images/Pasted%20image%2020250220151932.png)

![](images/Pasted%20image%2020250220151310.png)


![](images/Pasted%20image%2020250220151250.png)





