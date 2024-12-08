+++
title = "使用 MockMvc 对 SpringBoot 项目集成测试"
date = 2020-03-30 20:12:40
slug = "202003302012"

[taxonomies]
tags = ["MockMvc", "Spring Boot"]
+++

对后端进行集成测试的时候，我们一般都希望从发送 HTTP 请求开始对后端进行完整的测试，可能会考虑用 Postman 或者 Postwoman 这类东西，但是直觉告诉我们应该也可以把集成测试写在代码里

<!-- more -->

## 在 pom.xml 里引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.62</version>
</dependency>
```

## 准备随便一个项目用于测试

`BookVo` 如下

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class BookVo {

    Long id;
    String name;
}
```

这里使用了 Lombok 插件自动生成 Getter、Setter 和构造函数，并重写了 `equals` 方法

`BookController` 如下

```java
@RestController
@RequestMapping("/book")
public class BookController {

    @PostMapping
    public String postBook(@RequestParam String log, @RequestBody BookVo vo) {
        return log + vo.getName();
    }

    @DeleteMapping("/{id}")
    public Long deleteBook(@PathVariable Long id) {
        return id;
    }

    @PutMapping(value = "/{id}")
    public BookVo putBook(@PathVariable Long id, @RequestBody BookVo vo) {
        return vo;
    }

    @GetMapping("/{id}")
    public Long getBook(@PathVariable Long id) {
        return id;
    }
}
```

## 集成测试

我们可以使用 MockMvc 指定 HTTP 请求的类型、`PathVariable`、`RequestParam`、`RequestBody` 等，并能使用断言验证返回的状态码、结果等<br>
请求返回的 JSON 数据也能使用 Fastjson 方便的转换为对象

不多逼逼，看代码

```java
@SpringBootTest
@AutoConfigureMockMvc
class BookTests {

    @Autowired
    MockMvc mockMvc;

    @Test
    void test() throws Exception {

        BookVo vo = new BookVo(12L, "Book 1");

        // POST
        MvcResult res = mockMvc
            .perform(MockMvcRequestBuilders.post("/book").param("log", "Hello")
            .contentType(MediaType.APPLICATION_JSON).content(JSONObject.toJSONString(vo)))
            .andExpect(MockMvcResultMatchers.status().isOk()).andReturn();
        assertEquals("Hello Book 1", res.getResponse().getContentAsString());

        // DELETE
        res = mockMvc.perform(MockMvcRequestBuilders.delete("/book/{id}", "2"))
            .andExpect(MockMvcResultMatchers.status().isOk()).andReturn();
        assertEquals(2L, Long.valueOf(res.getResponse().getContentAsString()));

        // PUT
        res = mockMvc
            .perform(MockMvcRequestBuilders.put("/book/{id}", "12").contentType(MediaType.APPLICATION_JSON)
            .content(JSONObject.toJSONString(vo)))
            .andExpect(MockMvcResultMatchers.status().isOk()).andReturn();
            BookVo resVo = JSONObject.parseObject(res.getResponse().getContentAsString(), BookVo.class);
        assertEquals(vo, resVo);

        // 404
        mockMvc.perform(MockMvcRequestBuilders.get("/aook")).andExpect(MockMvcResultMatchers.status().isNotFound());
    }
}
```

需要注意的是使用 `RequestBody` 接收的参数需要使用 Fastjson 转换为字符串，并指定类型为 `APPLICATION_JSON`<br>
并且请求的返回值也可以使用 Fastjson 转换成简单的对象，比如上面代码就转成了 `BookVo`<br>
另外 `@Data` 注解会为 `BookVo` 重写 `equals` 方法，因此可以直接用 `assertEquals` 断言
