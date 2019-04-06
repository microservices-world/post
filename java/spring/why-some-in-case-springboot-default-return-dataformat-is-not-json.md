# springBoot 项目web api返回格式探析

### 1. 问题描述

为什么在某些情况下，spring boot项目web Api默认返回的是xml格式，而不是json格式

### 2. 问题分析

2.1 对于spring来说，web请求最终会由org.springframework.web.servlet.DispatcherServlet#doDispatch方法来处理。其中最重要的三个函数调用为：

```java
// 1. Determine handler for the current request.
mappedHandler = getHandler(processedRequest);
// 2. Determine handler adapter for the current request.
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
// 3.Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

重点关注第三步，它是真正处理请求的方法。

2.2 通过端点调试可知，在org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle方法中处理请求，并且获取返回值。代码如下：

```java
Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
```

我们需要关注的就是对于返回值的返回格式处理。

2.3 返回值格式最终由org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor#writeWithMessageConverters处理。代码如下：

```java
//1. 请求中获取支持的格式
List<MediaType> requestedMediaTypes = getAcceptableMediaTypes(request);
//2. messgeConverter中获取支持的格式
List<MediaType> producibleMediaTypes = getProducibleMediaTypes(request, valueType, declaredType);
```

从请求中获取格式，调用org.springframework.web.accept.HeaderContentNegotiationStrategy#resolveMediaTypes方法。代码如下：

```java
List<MediaType> mediaTypes = MediaType.parseMediaTypes(headerValues);
```

从messageConverters中获取格时，会先判断程序员是否指定了格式，例如：

```java
    @GetMapping(produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public object find(){
    		//do something
      return new Object;
    }
```

就指定此方法返回格式是json，回到最初的问题，spring boot默认就是返回json，如果不指定返回格式也应该是json的。从messageConverters中获取messageConverter支持的格式。

最后综合requestedMediaTypes与producibleMediaTypes获取最终要返回的格式类型。

2.4 debug发现，程序中注入了org.springframework.http.converter.xml.MappingJackson2XmlHttpMessageConverter此类导致程序最终选择了xml格式最为返回。spring boot采取默认注入的方式，具体实现在org.springframework.boot.autoconfigure.web.HttpMessageConvertersAutoConfiguration。由源码可知，MappingJackson2XmlHttpMessageConverter实例化依赖com.fasterxml.jackson.dataformat.xml.XmlMapper，对应jar包为：

```xml
<!-- Eureka deps that are now optional in eureka -->
 <dependency>
	<groupId>com.fasterxml.jackson.dataformat</groupId>
	<artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

最后是由于我们引用了jar包：

```xml
<dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-eureka-server</artifactId>
       <version>1.4.3.RELEASE</version>
</dependency>
```
由注释可知，spring cloud 1.4.3版本中引入了jackson-dataformat-xml

