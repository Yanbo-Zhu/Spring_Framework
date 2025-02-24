
在 **Spring Boot（Spring Data JPA）** 中，`@Table(name = "table_name")` 是 **JPA（Jakarta Persistence API** 提供的注解之一，主要用于**指定实体类对应的数据库表名称**。


# 1 @Table and @Entity 注解


|属性名|作用|
|---|---|
|`name`|指定表名（默认使用类名作为表名）|
|`schema`|指定数据库 Schema|
|`catalog`|指定数据库 Catalog|
|`uniqueConstraints`|指定表级唯一约束|
|`indexes`|指定索引|


```java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

@Entity
@Table(
    name = "users", 
    schema = "public", // 这将映射到 **`public.users`** 表。
    uniqueConstraints = { @UniqueConstraint(columnNames = "email") } // 如果希望某个字段是唯一的（不使用 `@Column(unique = true)`），可以在 `@Table` 中定义：
    indexes = { @Index(name = "idx_email", columnList = "email") } // 可以使用 `indexes` 在数据库表上创建索引： 这样，数据库会为 `email` 字段创建一个索引 `idx_email`，提高查询性能。
)
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true) 
    private String email;
}

```


添加唯一约束






