+++
title = "SpringBoot JPA快速上手"
date = 2020-03-30 19:31:41

[taxonomies]
tags = ["JPA", "Spring Boot"]
categories = ["Spring Boot"]
+++

现在SpringBoot上比较主流的持久层ORM框架应该就是MyBatis和JPA了。JPA对于简单查询（特别是作业）非常方便，可以说开箱即用，但是面对复杂查询就要稍微多学一点；而MyBatis写起来虽然没那么简洁，但在复杂查询的时候直接上SQL语句就完事了

<!-- more -->

## 引入JPA与MySQL依赖

在pom.xml中添加如下两条

```xml
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
  <scope>compile</scope>
</dependency>
```

## 配置application.yml

提醒一下，URL中的数据库要提前建好，比如下面示例中的数据库是library

然后用户名和密码也在这里配置

```yml
spring:
    datasource:
        url: jdbc:mysql://localhost:3306/library?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
        username: root
        password: root
        driver-class-name: com.mysql.cj.jdbc.Driver
    jpa:
        properties:
            hibernate:
                hbm2ddl:
                    auto: update
        database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
        show-sql: true
```

## Hello World

JPA每一个Entity就是一张表，在Entity里的每一个成员变量就是表里的一列，另外JPA还能够自动建表，十分省事

```java
@Entity
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    String name;

}
```

注意：对于实体，本文都省略了Getter、Setter和构造函数

然后对于一个Entity，需要写一个继承自JpaRepository的接口供逻辑层调用，不需要写实现类

```java
public interface BookRepo extends JpaRepository<Book, Long> {
}
```

最后我们可以跑一下单元测试

```java
@SpringBootTest
class BookRepoTests {

    @Autowired BookRepo repo;

    @Test
    void test() {

    // 增
    Book book1 = repo.save(new Book(null, "Book 1"));
    Book book2 = repo.save(new Book(null, "Book 2"));

    // 删
    repo.deleteById(book1.getId());

    // 改
    book2.setName("Book 2x");
    repo.save(book2);

    // 查
    Book book3 = repo.findById(book2.getId()).orElse(null);
    assertEquals("Book 2x", book3.getName());
    }

}
```

需要注意的是JPA的save方法，如果实体的ID为空或零，他会INSERT，否则UPDATE  
并且save方法的返回值才能作为最终有效的实体，作为参数传入的实体JPA不保证有效

## 自定义查询

根据语法提示我们发现JpaRepository提供了跟ID有关的很多访问数据库的方法，但是对于其他查询就要自己写，比如使用ID之外的字段来查询记录

直接在方法名中写查询语句就行了，不过要注意字段名的大小写

我们修改BookRepo如下

```java
public interface BookRepo extends JpaRepository<Book, Long> {

    // 根据name获取所有的书
    List<Book> findByName(String name);

    // 根据name获取匹配的书的数量
    Long countByName(String name);
}
```

## 多表关联

我们可能会遇到两张表相关联的情况，比如上面的例子就可能若干本书关联同一个作者

我们先写一个Author的实体，或者说表

```java
@Entity
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    String name;

}
```

并添加一个Repo

```java
public interface AuthorRepo extends JpaRepository<Author, Long> {
}
```

然后修改Book如下

这里的ManyToOne表示多个Book对应同一个Author，此外还有OneToMany、ManyToMany等

```java
@Entity
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    String name;

    @ManyToOne
    @JoinColumn
    Author author;

}
```

接下来写个单元测试跑一跑

```java
@SpringBootTest
class BookRepoTests {

    @Autowired
    AuthorRepo authorRepo;

    @Autowired
    BookRepo bookRepo;

    @Test
    void test() {

    Author author = new Author(null, "a1");
    author = authorRepo.save(author);

    Book book1 = new Book(null, "b1", author);
    Book book2 = new Book(null, "b2", author);
    book1 = bookRepo.save(book1);
    book2 = bookRepo.save(book2);

    assertEquals(book1.getAuthor().getId(), book2.getAuthor().getId());
    }
}
```
