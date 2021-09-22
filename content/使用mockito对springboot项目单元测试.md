+++
title = "使用Mockito对SpringBoot项目单元测试"
date = 2020-03-30 20:30:55

[taxonomies]
tags = ["Mockito", "Spring Boot"]
categories = ["Spring Boot"]
+++

使用Mockito可以用来给模块打桩，实现单元测试

<!-- more -->

## 在pom.xml里引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <exclusions>
        <exclusion>
            <groupId>org.junit.vintage</groupId>
            <artifactId>junit-vintage-engine</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## 准备待测模块

```java
@Service
public class BookService {

    @Autowired
    BookRepo repo;
    
    public Long addBook(String name) {
        Book po = repo.save(new Book(null, name));
        return po.getId();
    }
}
```

其中Book实体大概长这样

```java
public class Book {

    Long id;
    String name;
}
```

## 打桩并测试

其中MockBean表示把Spring注入的Bean改为注入由Mockito生成的桩

然后对save方法打桩，对于一个输入设置对应的输出  
并断言该方法会被调用及调用时传入的参数

```java
@SpringBootTest
class BookServiceTests {
    @MockBean
    BookRepo repo;
    @Autowired
    BookService service;

    @Test
    void test() {
        Book bookToSave = new Book(null, "Hello");
        Mockito.when(repo.save(bookToSave)).thenReturn(new Book(12L, "Hello"));
        Long res = service.addBook("Hello");
        Mockito.verify(repo).save(bookToSave);
        assertEquals(12L, res);
    }
}
```
