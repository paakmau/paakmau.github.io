+++
title = "封装 Spring Boot 库并使用 GitHub Packages 发布"
date = 2022-01-31 21:33:14
slug = "202201312133"

[taxonomies]
tags = ["GitHub Packages", "Java", "Gradle", "Maven"]
+++

后端写多了一些常用的东西可以封装成一个库，最好还能发布到什么地方直接加个仓库就能加依赖。
因为我的项目大部分是托管在 GitHub 上的，于是可以考虑直接使用 GitHub Packages 进行发布。

<!-- more -->

这样的好处在于，源码与发布都集中在同一个地方，方便管理。
~~更重要的是还不用担心自己的屎山污染中央仓库从而受到他人的谴责。~~

需要注意一点是，如果库需要依赖 Spring Boot，那么打包的和发布都会跟普通的 Java 库有一点区别。

另外，本文使用的构建工具是 Gradle。

## 参考资料

[Creating a Multi Module Project](https://spring.io/guides/gs/multi-module/)

[Guide to building Spring Boot library](https://piotrminkowski.com/2020/08/04/guide-to-building-spring-boot-library/)

[Creating Your Own Auto-configuration](https://docs.spring.io/spring-boot/docs/2.6.3/reference/html/features.html#features.developing-auto-configuration)

[Publishing Java packages with Gradle](https://docs.github.com/en/actions/publishing-packages/publishing-java-packages-with-gradle)

## 封装动机

考虑这样一个典中典场景：假设我是开外包公司的，我们要做好几个换壳应用，但是对于不同的甲方，这些应用会有一些功能上的小修改，而且根据报价和需求还会有功能上的删减。
~~于是显而易见每次外包就直接 fork 下来摁改就完事了。~~
合理的做法是把核心封装成若干个高度内聚的库，根据甲方的需要引相应库的依赖就好了。
接下来我们又发现，普通的 Maven 库不够爽，每次我还是要在 Spring Boot 应用里面手动配置一遍 Service 啥的再把库里的东西拉出来搞七搞八。
那么我们就考虑再开一个库，这个库依赖那个普通的 Maven 库，里面直接配置好 Spring Bean。
只要引入这个库的依赖，我们就可以在应用里面直接自动注入这些 Bean 了。
这个基于 Spring Boot 的库我们就称之为启动器（starter）。

## 基于 Spring Boot 的库

一个基于 Spring Boot 的库会有一些坑。
我们需要依赖 Spring Boot 的 Gradle 插件，但是这个插件默认执行 `bootJar` 任务，这个东西要求我们提供一个 `main` 函数，显然一个库不应当有 `main` 函数。
于是我们需要把 Spring Boot 的 Gradle 插件禁用掉，但是仍然保留它的引入，因为我们需要它的依赖管理功能。

最后大致的 `build.gradle` 就会长这样。

```gradle
plugins {
    id 'org.springframework.boot' version '2.6.3' apply false
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

repositories {
    mavenCentral()
}

dependencyManagement {
    imports {
        mavenBom org.springframework.boot.gradle.plugin.SpringBootPlugin.BOM_COORDINATES
    }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

至于其他人说的禁用 `bootJar` 任务，启用 `jar` 任务应该也是可行的，但是在官方文档中被改成了现在这个版本。

## 启动器

根据 Spring 的官方文档，一个自定义的启动器可以包含两个东西：自动配置模块和启动器模块。

自动配置模块其实就是把需要集成到 Spring 上的 Maven 库提供的接口封装进 Spring Bean 里。
当然一般还需要定义一些可选的配置键。
此外，还可以定义可选依赖，比如有时候对于同一个功能我们会封装一个适配器，可以在若干个外部依赖中选择一个作为具体实现。

而启动器模块只是引入了自动配置模块，当然还有普通的 Maven 库，此外还引入了自动配置模块提供的可选依赖中一些最常用的依赖库。
简单地说就是启动器模块就只是做默认的依赖引入，从而实现开箱即用。
用户在使用的时候也可以不选择这个启动器模块，而是自行引入自动配置模块以及其他的可选依赖，也就是自定义的启动器模块。

不过这两个模块的划分并不是必要的，因为有些时候我们发现我们的库可能根本没那么复杂，也不需要什么可选依赖，那么官方的推荐做法就是把自动配置模块直接合并到启动器模块中。

## 命名

对于需要集成到 Spring 上的 Maven 库，自动配置模块我们命名为 `xxx-spring-boot-autoconfigure`，启动器模块我们命名为 `xxx-spring-boot-starter`。

## 配置键

在使用这个库的时候我们可能希望进行一些自定义配置，比如对于一些网络服务我们可能会想要修改默认的监听端口。
这个时候我们就要自定义一些配置键，这样我们的 Spring Bean 就能够读取到它们。
我们可以使用 `@ConfigurationProperties` 来定义配置键。
需要注意的是一定要给每个属性添加 javadoc 以保证文档化。
另外根据实践经验，配置键的前缀应当使用自定义的命名空间，不要跟 Spring Boot 用的命名空间揉在一起。

比如在下面这个例子中，我们添加了一个 `lib.port` 的整型配置键，默认值为 8082。
于是我们就可以使用 `@Autowired` 在其他 Bean 中自动注入这个东西获取配置键的实际值。

```java
@ConfigurationProperties("lib")
public class LibProperties {

    /**
     * The listen port.
     */
    private int port = 8082;

    // getters/setters ...

}
```

另外需要注意的是，我们需要生成一些元数据供 IDE 识别，这样 IDE 才能在 `application.properties` 文件中对这些自定义的配置键提供自动补齐等支持。
于是我们需要引入 `spring-boot-configuration-processor` 依赖。
使用下面的例子引入即可，具体的解释可以参考 [Generating Your Own Metadata by Using the Annotation Processor](https://docs.spring.io/spring-boot/docs/2.6.3/reference/html/configuration-metadata.html#configuration-metadata.annotation-processor)。

```gradle
dependencies {
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
}

compileJava.dependsOn(processResources)
```

然而有一个遗憾的事情是，如果你在 VS Code 上用 Gradle，这个操作是无效的。
因为 VS Code 的 Java 插件用 Eclipse JDT 来提供 Gradle 支持，然后 Gradle 不对 Eclipse 提供注解处理配置的支持。
可以参考这个 [issue](https://github.com/spring-projects/sts4/issues/585)。
不过也有好消息，如果用 Maven 的话就没有这个问题。

## 发布与引入

一般依赖包发布的地方跟代码托管的地方保持一致会令人觉得很爽。
好消息是 GitHub 和 GitLab 都集成了免费的 Maven 仓库。
于是这里我们以发布到 GitHub Packages 为例。

首先看一下构建脚本，也就是 `build.gradle`，大概长这样。

```gradle
plugins {
  ...
  id 'maven-publish'
}

publishing {
  ...

  repositories {
    maven {
      name = "GitHubPackages"
      url = "https://maven.pkg.github.com/<owner>/lib-spring-boot-starter"
      credentials {
        username = System.getenv("GITHUB_ACTOR")
        password = System.getenv("GITHUB_TOKEN")
      }
    }
  }
}
```

注意到这个 `url` 参数前缀是固定的，然后 `<owner>` 可以是个人的用户名也可以是组织名，后面跟着就是项目名。

然后写一个持续集成脚本就行了，命名为 `publish.yml`。

```yml
# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.

name: Publish package to GitHub Packages
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
      - name: Validate Gradle wrapper
        uses: gradle/wrapper-validation-action@e6e38bacfdf1a337459f332974bb2327a31aaf4b
      - name: Publish package
        uses: gradle/gradle-build-action@4137be6a8bf7d7133955359dbd952c0ca73b1021
        with:
          arguments: publish
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

这个东西的触发条件是有满足 `v[0-9]+.[0-9]+.[0-9]+` 正则表达式的 tag 的 push。

最后我们就可以在其他项目中引入这个依赖了。

```gradle
repositories {
    maven {
        url = uri("https://maven.pkg.github.com/<owner>/lib-spring-boot-starter")
        credentials {
            username = "<your-name>"
            password = "<your-token>"
        }
    }
}

dependencies {
    implementation 'com.example:lib-spring-boot-starter:0.0.1-SNAPSHOT'
}
```

首先就是添加 GitHub Packages 提供的 Maven 仓库。
同样注意到这个 `url` 参数跟我们发布时是相同的。
但是根据 GitHub 员工所述，项目名并不重要，直接用 `<owner>/*` 即可，注意 `*` 是必要的。
另外 `password` 参数不能使用真实的密码，应当在用户的设置界面申请一个具有 `read:packages` 权限的 token，并使用这个 token。
尽管这个 token 只具有访问依赖包的权限，但是最好还是使用 actions secrets 避免明文挂在仓库中。
最后正常引入依赖就行了。

于是问题来了，我的仓库是开源的，发布的依赖包也是公开的，为什么访问一个公开的依赖包也需要身份认证？
这个时候，请移步[讨论区](https://github.community/t/download-from-github-package-registry-without-authentication/14407/106)。
可以看到因为这个问题 GitHub 已经被吐槽了两年多了。
其中有一个 GitHub 员工提出的比较好的解决方案是配置一个机器人账号并直接明文使用它的 `read:packages` 权限的 token。
不过也有邪恶的人指出可以使用 GitHub Actions 发布到 GitLab 上，因为 GitLab 对公开的依赖包的访问不需要授权，并且还给出了详细的示例。

## 完整实例

<https://github.com/paakmau/spring-boot-starter-template>
