---
layout: post
title: 将springmvc配置成一个REST服务器的过程
date: 2018-01-21
categories: java
tags: java spring springmvc
---



现在比较流行的一个开发方式时把逻辑尽可能多地写到前端，后端只负责做数据交互，当前端需要数据时，使用REST风格的URL向后端请求，然后后端返回一个json串给前端。java后端现在似乎已经有很多REST框架，但是大多数java程序员比较熟悉的框架还是springmvc，既然springmvc现在也支持REST，何尝不试试呢？于是就折腾了一下午总算搞定了，这里记录一下以供后来参考。springmvc使用的版本是4.3.6.RELEASE。笔者将假设读者已经具有了springmvc的基本使用知识。


# RESTful风格的URL简介
---
这里仅说说我的最简单的理解，欢迎吐槽。

在开发传统web应用时，我们并没有完整的使用http协议所提供的所有方法，只使用了GET与POST方法，并当用GET方法传递参数时使用的是类似于`http://dqy.today/articles.action?articleId=3` 这样冗长的URL，参数是通过"?+键值对"的方式传递。
RESTful风格的URL则主张参数通过url路径本身来传递，比如上面给出的URL写成RESTful风格就是`http://dqy.today/articles/3` ，直接将参数写在了url的路径当中，这样的URL更加简端，而且含义也更加明确。
RESTful也主张将HTTP中的GET，POST，PUT，DELETE方法全部使用上，让同一个URL可以表达更多的含义，以`http://dqy.today/articles` 这个URL为例，这几种操作的含义分别是：

 - GET：获得全部文章（article）
 - POST：增加一篇文章
 - PUT：修改文章
 - DELETE：删除一篇文章

REST风格使得URL在变得简洁的同时也增强了URL的表现力，更加充分利用了HTTP协议。

# 让springmvc拦截所有请求
---
将springmvc核心Servlet的拦截路径改为 `/`，如下：
```html
<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```
注意这里只能写成`/` 不能写成`/*` ，否则会报奇怪的错误，具体原因我也不是很清楚。

接下来还需要在springmvc的配置文件中配置让框架不拦截静态资源的路径，使用如下的标签：
```html
<!-- 静态资源解析
	包括：js,css,img... -->
<mvc:resources location="/static/" mapping="/static/**"/>
<!--不拦截欢迎页-->
<mvc:resources location="/" mapping="/index.html"/>
```
mapping表示要映射的url，可以使用通配符,`*` 表示只能匹配一层子路径，而`**` 表示能够匹配所有子路径，比如`/static/*` 能够匹配url路径`/static/a`，但是无法匹配`/static/a/b`，而`/static/**` 能够匹配`/static/a/b`。location其在项目目录下的真实位置，框架会将mapping匹配到的子路径连接在location的后面，比如`/static/js/b.js`这样的url路径就不会被springmvc的核心Servlet拦截，而是直接去项目目录的`/static/js/b.js` 路径下寻找静态资源。

# 使用springmvc获得嵌入在URL目录中的参数
---
之前传统的web应用springmvc都是自动获取"?"后面的参数并绑定到方法参数中的，现在的参数在URL路径中，要怎么获取呢？其实springmvc也提供了几个方便的注解让我们获取，示例如下：

修改文章请求的处理方法：
```java
@RequestMapping(value = "articles/{articleId}", method = RequestMethod.PUT)
public void modifyArticle(@PathVariable("articleId") String id,Article article){
    ......
}
```

删除文章请求的处理方法：
```java
@RequestMapping(value = "articles/{articleId}", method = RequestMethod.DELETE)
public void delArticle(@PathVariable("articleId") String id,Article article){
    ......
}
```

这两个方法都可以获得客户端发起的`/article/*`这类请求（{articleId}就类似于一个通配符），然后框架会根据@RequestMapping的method 参数决定最终给哪个方法处理，如果是http的PUT方法则会交给modifyArticle方法处理，如果是DELETE方法，则交给delArticle方法处理。

通过@PathVariable注解将路径中的参数绑定到方法参数中，它会将{articleId}所匹配到的路径中的值绑定到方法参数id中。

# 使用springmvc收发json串
---
REST服务器比较流行用json进行收发，springmvc也提供了相关的便利，但是还需要添加额外的jackson的依赖，添加的jackson的版本需要与springmvc相匹配，否则会报错，经过笔者的验证，在4.3.6版本的springmvc下引入如下版本的jackson依赖：
```html
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.8.9</version>
</dependency>

<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-core</artifactId>
  <version>2.8.9</version>
</dependency>

<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-annotations</artifactId>
  <version>2.8.9</version>
</dependency>
```

然后通过如下的注解就可以让springmvc帮我们在收发json串自动将它们转换成对象：
```java
@RequestMapping(value = "/...")
public @ResponseBody Object1 (@RequestBody Object2 object2){
...
...
return object2;
}
```
Object1与Object2是两个自定义的对象，使用@RequestBody注解修饰方法参数object2，当该方法接受到请求时，springmvc会将json串自动转换成对象传入object2参数中（按照对象中Object2中的成员属性名匹配），经过笔者验证，@RequestBody最多只能修饰一个方法参数，如果同时修饰多个方法参数的话运行时会报错。@ResponseBody注解能够将方法返回的对象转换成json串传递给客户端。

# End
---
掌握这些知识后，就可以将springmvc改造成一个REST服务器了


