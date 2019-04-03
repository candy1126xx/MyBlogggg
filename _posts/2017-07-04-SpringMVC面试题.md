---
layout:     post                    # 使用的布局
title:      SpringMVC面试题               # 标题 
subtitle:   SpringMVC面试题           #副标题
date:       2017-07-04              # 时间
author:     candy1126xx                      # 作者
header-img: img/home-bg-o.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 面试题
---

#### 什么是Spring MVC ？简单介绍下你对springMVC的理解?
Spring MVC是一个基于Java的实现了MVC设计模式的请求驱动类型的轻量级Web框架，通过把Model，View，Controller分离，将web层进行职责解耦，把复杂的web应用分成逻辑清晰的几部分，简化开发，减少出错，方便组内开发人员之间的配合。

#### SpringMVC的流程？
1、用户发送请求至前端控制器DispatcherServlet；
2、DispatcherServlet收到请求后，调用HandlerMapping处理器映射器，请求获取Handle；
3、处理器映射器根据请求url找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet；
4、DispatcherServlet 调用 HandlerAdapter处理器适配器；
5、HandlerAdapter 经过适配调用 具体处理器(Handler，也叫后端控制器)；
6、Handler执行完成返回ModelAndView；
7、HandlerAdapter将Handler执行结果ModelAndView返回给DispatcherServlet；
8、DispatcherServlet将ModelAndView传给ViewResolver视图解析器进行解析；
9、ViewResolver解析后返回具体View；
10、DispatcherServlet对View进行渲染视图（即将模型数据填充至视图中）
11、DispatcherServlet响应用户。

#### Spring MVC的主要组件？
1、前端控制器 DispatcherServlet（不需要程序员开发）
作用：接收请求、响应结果，相当于转发器，有了DispatcherServlet 就减少了其它组件之间的耦合度。
2、处理器映射器HandlerMapping（不需要程序员开发）
作用：根据请求的URL来查找Handler
3、处理器适配器HandlerAdapter
注意：在编写Handler的时候要按照HandlerAdapter要求的规则去编写，这样适配器HandlerAdapter才可以正确的去执行Handler。
4、处理器Handler（需要程序员开发）
5、视图解析器 ViewResolver（不需要程序员开发）
作用：进行视图的解析，根据视图逻辑名解析成真正的视图（view）
6、视图View（需要程序员开发jsp）
View是一个接口， 它的实现类支持不同的视图类型（jsp，freemarker，pdf等等）

#### SpringMVC怎么样设定重定向和转发的？
转发：在返回值前面加"forward:"，譬如"forward:user.do?name=method4"
重定向：在返回值前面加"redirect:"，譬如"redirect:http://www.baidu.com"

#### 如何解决POST请求中文乱码问题？
在web.xml中配置一个CharacterEncodingFilter过滤器，设置成utf-8。
```
<filter>
    <filter-name>CharacterEncodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>CharacterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

#### 如何解决GET请求中文乱码问题？
1、修改tomcat配置文件添加编码与工程编码一致。
2、对参数进行重新编码。

#### SpringMvc的控制器是不是单例模式,如果是,有什么问题,怎么解决？
是单例模式,所以在多线程访问的时候有线程安全问题,不要用同步,会影响性能的,解决方案是在控制器里面不能写字段。

#### SpringMVC常用的注解有哪些？
@Controller：标识这个类是一个控制器。
@RequestParam：当表单参数和方法形参名字不一致时，做一个名字映射
@PathVarible：用于获取uri中的参数,比如user/1中1的值
@RequestMapping：用于处理请求 url 映射的注解，可用于类或方法上。用于类上，则表示类中的所有响应请求的方法都是以该地址作为父路径。
@RequestBody：注解实现接收http请求的json数据，将json转换为java对象。
@ResponseBody：注解实现将conreoller方法返回对象转化为json对象响应给客户。
@RestController：@Controller+ @ResponseBody
@GetMapping@DeleteMapping@PostMapping：@PutMapping
@SessionAttribute：声明将什么模型数据存入session
@CookieValue：获取cookie值
@ModelAttribute：将方法返回值存入model中
@HeaderValue：获取请求头中的值

#### 如果在拦截请求中，我想拦截get方式提交的方法,怎么配置？
可以在@RequestMapping注解里面加上method=RequestMethod.GET。

#### 怎样在方法里面得到Request,或者Session？
直接在方法的形参中声明request,SpringMvc就自动把request对象传入。

#### 如果想在拦截的方法里面得到从前台传入的参数,怎么得到？
直接在形参里面声明这个参数就可以,但必须名字和传过来的参数一样。

#### 如果前台有很多个参数传入,并且这些参数都是一个对象的,那么怎么样快速得到这个对象？
直接在方法中声明这个对象,SpringMvc就自动会把属性赋值到这个对象里面。

#### 如果在拦截请求中,我想拦截提交参数中包含"type=test"字符串,怎么配置？
可以在@RequestMapping注解里面加上params="type=test"

#### SpringMvc中函数的返回值是什么？
返回值可以有很多类型,有String, ModelAndView。ModelAndView类把视图和数据都合并的一起的，但一般用String比较好。

#### SpringMvc怎么处理返回值的？
SpringMvc根据配置文件中InternalResourceViewResolver的前缀和后缀,用前缀+返回值+后缀组成完整的返回值

#### SpringMvc用什么对象从后台向前台传递数据的？
通过ModelMap对象,可以在这个对象里面调用put方法,把对象加到里面,前台就可以通过el表达式拿到。

#### 怎么样把ModelMap里面的数据放入Session里面？
可以在类上面加上@SessionAttributes注解,里面包含的字符串就是要放入session里面的key。

#### SpringMvc里面拦截器是怎么写的？
1、实现HandlerInterceptor接口。
2、继承适配器类，接着在接口方法当中，实现处理逻辑；然后在SpringMvc的配置文件中配置拦截器即可。
```
<mvc:interceptors>
    <bean id="myInterceptor" class="com.zwp.action.MyHandlerInterceptor"></bean>  // 拦截所有请求
    <mvc:interceptor>  // 拦截指定请求
        <mvc:mapping path="/modelMap.do" />
        <bean class="com.zwp.action.MyHandlerInterceptorAdapter" />
    </mvc:interceptor>
</mvc:interceptors>
```


