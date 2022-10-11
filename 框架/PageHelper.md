# PageHelper

## 什么是分页

​		分页,是一种将所有数据分段展示给用户的技术.用户每次看到的不是全部数据,而是其中的一部分,如果在其中没有找到自习自己想要的内容,用户可以通过制定页码或是翻页的方式转换可见内容,直到找到自己想要的内容为止.其实这和我们阅读书籍很类似。可提高用户体验度，同时减少一次性加载，[内存](https://so.csdn.net/so/search?q=内存&spm=1001.2101.3001.7020)溢出风险。

## 分页的意义

​		分页确实有效,但它一定会加大系统的复杂度,但可否不分页呢?如果数据量少的话当然可以.但是对于企业信息系统来说数据量不会限制在一个小范围内.如果不顾一切的Select * from某个表,再将返回的数据一古脑的扔给客户,即使客户能够忍受成千上万足够让人眼花缭乱的表格式数据,繁忙的网络,紧张的服务器也会提出它们无声的抗议,甚至有时会以彻底的罢工作为终结。

## 分页的实现

​		用户发起请求，后端查询数据库返回所有的条数，并且返回用户所需要的数据，比方说用户请求的是第一页(page = 1)，用户设置的第一页的数据显示条数为50条(limit=50)，那么后台查询满足条件的数据，并且返回前50条给用户显示，此时用户看到的就是第一页的50条数据，加入用户请求的是第二页的数据，那么传给后台page为2，显示条数limit为50后台查询的就是符合条件的51-100条数据返回给用户。

## PageHelper中SQL语句怎么拼接

在调用PageHelper.startPage()方法后，会根据传入的分页参数生成一个Page对象，然后将Page对象设置到ThreadLocal变量中去。在组装SQL语句的时候，从ThrealLocal变量中获取分页参数，通过实现 Mybatis 的 Interceptor 接口完成对 query sql 的动态分页，	

> 简单的分页执行过程
>
> 1. 设置 page 参数
> 2. 执行 query 方法
> 3. Interceptor 接口 中校验 ThreadLocal 中是否存在有设置的 page 参数
> 4. 存在 page 参数，重新生成 `count sql` 和 `page sql`，并执行查询。不存在 page 参数，直接返回 查询结果
> 5. 执行 `LOCAL_PAGE.remove()`清除 page 参数

## PageHelper并发会出现什么问题

​		startPage方法把分页信息Pagination存放到了ThreadLocal里。ThreadLocal是线程保护的，作用域只限于当前线程。

​	  观察上述的执行过程，可以发现，如果在第 1 步和第 2 步 之间发生异常，那么 LOCAL_PAGE 中当前线程对应的 page 参数并不会 remove。

​        在不使用线程池的情况下，当前线程在执行完毕后会被销毁，这时 当前线程 中的 threadLocals 参数 将会被情况，也就清空 了 LOCAL_PAGE 中 当前线程的 page 参数。

​         但是如果使用了线程池，当前线程执行完毕，并不会被销毁，而是会将当前线程再次存放到池中，标记为空闲状态，以便后续使用。在后续使用这个线程的时候，由于 线程 的 threadLocals 依旧存在有值，尽管我们在第 1 步时未设置 page 参数，第 3 步 的也能获取到page参数，从而生成 count sql 和 page sql，从而影响我们的正常查询。SpringBoot 项目中会使用内置的 Tomcat 作为服务器，而Tomcat会默认使用线程池来处理请求，从而便引发了上述问题。

### 解决方案

​           因为Tomcat的线程是用来处理request请求，那么在请求完成时，清空当前线程的threadLocals 属性值，也就是执行LOCAL_PAGE.remove() 即可。

#### 实现方式：

1. 使用 aop，对所有 controller 进行处理
2. 实现 HandlerInterceptor 或者 WebRequestInterceptor 对 request 请求的拦截器接口，通过 afterCompletion 方法执行 LOCAL_PAGE.remove() 。

```java
public class PageLocalWebInterceptor implements HandlerInterceptor {
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

        // PageHelper.clearPage() 内部调用 LOCAL_PAGE.remove()
        PageHelper.clearPage();

    }
}

```

定义配置类，配置类需实现 WebMvcConfigurer 接口完成对于WebMvc的相关配置 ，注册 PageLocalWebInterceptor ：

```java
@Configuration
public class FrameworkAutoConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new PageLocalWebInterceptor());
    }
}

```



## 参考

1. [分页查询原理]: https://blog.csdn.net/qq_43386944/article/details/119635774

   

