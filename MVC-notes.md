# 1. Spring MVC Notes

Spring MVC 的核心在于 **DispatcherServlet**, `DispatcherServlet` 实际上就是一个 **HttpServlet** 内部引入了 **ApplicationContext** 等 bean 提供了依赖注入的能力。如果我们有个自己写的 IOC Container， 例如，`CustomBeanRegistry`， 我们完全可以 embed 一个 ServletContainer， 例如 tomcat。然后我们通过继承 **HttpServlet** 写一个自己的 Servlet, 内部包含我们写的 IOC Container 的引用，在我们处理 **init**, **service**  等方法时，我们就可以动态的寻找或注入一些 bean.

在 **DispatcherServlet** 中，核心的组件有以下几个组成

- HandlerMapping
- HandlerExecutionChain
- HandlerAdaptor

## 1.1 HandlerMapping

**HandlerMapping** 用于获取 **HandlerExecutionChain**, 传统意义上来说， ServletContainer 通过 URL pattern 将请求 map 到指定的 Servlet 上， 由于 Spring 为了提供依赖注入和 bean 管理的功能， Spring 容器内部提供了一套自己写的类似的 mapping 方式。也就是说，我们使用 Spring 的时候，我们不再以 ServletContainer (如 tomcat) 的方式去进行 Servlet mapping, 某种意义上来说，我们只有一个 Servlet， 也就是 **DispatcherServlet**, 然后我们通过 Spring 的方式去将请求 map 到我们的处理方法中。

对应每一个 URL pattern, 我们一般使用 **@RequestMapping** 的方式来将处理方法 map 到请求上, 在 Spring MVC 中，负责处理请求的对象是 **HandlerMethod**， 实际上就是包装了带有 **@RequestMapping** 注释的方法的应用。还句话来说，**HandlerMapping** 的工作就是，给予一个 request (带有 url 的信息), 找到对应的 **HandlerMethod** 然后进行某种数据的反序列化，调用 **HandlerMethod** 方法，真正的处理请求。

回到 **HandlerExecutionChain** 上，**HandlerExecutionChain** 实际上就是多个 **HandlerInterceptor** + **HandlerMethod**, 它只是多带了一些 interceptor, 这些 **HandlerInterceptor** 的工作方式与 Filter 相似 (与 Spring Security的Filter实现方式不同). 

关于 **HandlerMapping** 如何根据请求找到对应的 **HandlerMethod**, 它内部带有一个 registry, 在 **AbstractHandlerMethodMapping.mappingRegistry** 中， 这个 **MappingRegistry** 如下面的 snippet 展示, 也是通过路径，名称来进行匹配的。

```
class MappingRegistry {

    private final Map<T, MappingRegistration<T>> registry = new HashMap<>();

    private final MultiValueMap<String, T> pathLookup = new LinkedMultiValueMap<>();

    private final Map<String, List<HandlerMethod>> nameLookup = new ConcurrentHashMap<>();

    private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();

    private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();

    // ...
}
```

具体处理的方式如下，在 **DispatcherServlet.doDispatch(...)** 有：

```
HandlerExecutionChain mappedHandler;

...

// Determine handler for the current request.
mappedHandler = getHandler(processedRequest);
if (mappedHandler == null) {
    noHandlerFound(processedRequest, response);
    return;
}
```

**getHandler(...)** 方法如下： 

```
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

## 1.2 HandlerExecutionChain

在上面已经提到了, **HandlerExecutionChain** 本质上就是 **HandlerInterceptor** + **HandlerMethod**. 它包装了 **HandlerMethod** 的同时还负责调用这些 **HandlerInterceptor**, 如下：

在真正处理请求之前：

```
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    for (int i = 0; i < this.interceptorList.size(); i++) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        if (!interceptor.preHandle(request, response, this.handler)) {
            triggerAfterCompletion(request, response, null);
            return false;
        }
        this.interceptorIndex = i;
    }
    return true;
}
```

在真正处理请求之后：

```
/**
* Apply postHandle methods of registered interceptors.
*/
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, 
            @Nullable ModelAndView mv) throws Exception {

    for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        interceptor.postHandle(request, response, this.handler, mv);
    }
}
```

在处理结束后 (处理可能失败，但 preHandle 必须成功, 因为如果 preHandle 不成功就是请求被拦截了):
 
```
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, 
            @Nullable Exception ex) {

    for (int i = this.interceptorIndex; i >= 0; i--) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        try {
            interceptor.afterCompletion(request, response, this.handler, ex);
        }
        catch (Throwable ex2) {
            logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
        }
    }
}
```

## 1.3 HandlerAdapter

**HandlerAdapter** 是 **HandlerMethod** 与 **HttpServletRequest**, **HttpServletResponse** 之间的 adapter, 负责调用 **HandlerMethod.handle(...)** 方法，用于将调用 handler 的细节隐藏起来，因为开发人员可能写自己的 handler 和 自己的 **HandlerAdapter** 用于调用自己写的 handler. **HandlerAdapter** 是通过我们用 **HandlerMapping** 获得的 handler 来获得的，具体代码如下：

```
HandlerExecutionChain mappedHandler = null;

...

mappedHandler = getHandler(processedRequest);

...

// Determine handler adapter for the current request.
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

**getHandlerAdapter(...)** 代码如下：

```
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler +
            "]: The DispatcherServlet configuration needs to include" +  
            "a HandlerAdapter that supports this handler");
}
```

使用 **HandlerAdaptor** 处理请求的代码如下： 

```
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

...

// Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

**HttpRequestHandlerAdapter** 中 **handle** 方法：

```
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, 
            Object handler) throws Exception {

    ((HttpRequestHandler) handler).handleRequest(request, response);
    return null;
}
```

## 1.4 大致交互

```
+----------------+    +-----------------------+
|                |    |                       |
| HandlerMapping +----> HandlerExecutionChain |
|                |    |                       |
+----------------+    +-----------------------+

+------------------------------+        +----------------+
|                              |        |                |
|HandlerExecutionChain.handler +--------> HandlerAdapter |
|                              |        |                |
+------------------------------+        +----------------+

+-------------------------+          +-------------------------------+
|                         |          |                               |
| HandlerAdapter.handle() +----------> HandlerExecutionChain.handler |
|                         |          |                               |
+-------------------------+          +-------------------------------+
```

**HandlerInterceptor** 生命周期

```
              +-----------------+
              |                 |
              | preHandle       |
              |                 |
              +-------+---------+
---------->           |          <---- handle()  -------
              +-------v---------+
              |                 |
              | postHandle      |
              |                 |
              +-------+---------+
                      |
              +-------v---------+
              |                 |
              | afterCompletion |
              |                 |
              +-----------------+
```

## 1.5 文字描述

**DispatcherServlet#doService()** 调用 **DispatcherServlet.doDispatch(...)** 调用 **DispatcherServlet.getHandler(...)**, 首先内部已经有注册了的 **HandlerMapping**, 轮询 **HandlerMapping** list, 每个 **HandlerMapping.getHandler(...)** 调用一次，不为 null 则为找到了, 实际上 **AbstractHandlerMapping** 中调用了 **#getHandlerInternal()** 方法由子类实现，其中的一个子类 **AbstractHandlerMethodMapping** 实现该方法的方式为在 **MappingRegistry** 中找到对应的 **HandlerMethod**.

获得了这个 handler (实际上是 **HandlerExecutionChain**) 之后， 拿真正的 handler 通过 **#getHandlerAdapter(handlerExecutionChain.handler)** 去匹配 **HandlerAdapter**, 在 **DispatcherServlet** 里已经有注册了的多个 **HandlerAdapter**, 所谓的匹配也就是调用 **handlerAdapter.supports(handler)** 来看是否支持。拿到 adapter 之后，使用 **HandlerExecutionChain.applyPreHandle()** 来调用 handler 对应的 **HandlerInterceptor** 的 **preHandle** 方法。如果这些 preHandle 调用里面有失败的 (`return false`), 这意味着请求被拦截了，结束处理 (但还是会调用 afterCompletion). 

调用完 preHandle 以后开始进行真正的请求处理，调用 **handlerAdapter.handle(request, response, handler)**, 返回的结果是 **ModelAndView**, 这个对象只是 model 和 view 的放在了一起，仅此而已. 处理完以后，再次过一边 **HandlerInterceptor** 的 postHandle, 通过调用 **HandlerExecutionChain.applyPostHandle()**. 

如果在 handle 的过程中发生了异常, 则会用该 exception 通过内部注册的 **HandlerExceptionResolver** 的 **#resolveException(...)** 方法j来找到对应的处理结果. 返回的结果是 **ModelAndView**, 这基本上就是 **ControllerAdvice** 的用法, 更具体的来说，是类 **ExceptionHandlerExceptionResolver**, **@ExceptionHandler** 是注释在方法上的，所以工作方式跟 **MethodHandler** 基本一直，匹配哪个方法用的是 Exception 的类型, 这也是包含在 **@ExceptionHandler** 中的信息。 

最后统一对 **ModelAndView** 对象处理，也就是把数据返回给客户端。请求处理完毕后，调用 **HandlerInterceptor.afterCompletion** 方法.  

 