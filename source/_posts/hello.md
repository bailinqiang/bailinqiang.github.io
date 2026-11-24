---
title: hello
date: 2019-03-04
tags: []
---

### 背景

在使用springmvc的时候  用@ResponseBody注解 直接返回对象，但是用ajax查询返回的结果中 引号都被转义了，可以确实是springmvc的处理，在返回的过程中给转义了。那么当时想到的解决办法：

端上进行替换、网上有很多的解决办法是这样的，注意一点，js中要用正在表达式才能全部替换掉，replease只能替换第一个匹配到的
服务端返回成JSON String  也就是 先将返回去的对象序列化成jsonstr返回，这样前端接收到的是可以正常解析的

### 最终解决办法

第一种方案 被排除了，因为在客户端发起的ajax调用，当时没有封装一个统一的出口，所以修改的地方会很多，这个大家在写前端项目的时候可以注意下，封装一个 方法，来统一处理ajax调用 比如 promise(url, data, successFun, failFun) 这种统一方法。
第二种方案，1、简单一点 处理每个方法的返回值，但是这种办法很浪费时间，也不是咱程序员应该处理的，既然是springmvc项目，那么 对springmvc了解一些的同学应该知道，springmvc有统一的对返回值处理的地方，那既然这样的话，我首先去排查一下是哪个 HandlerMethodReturnValueHandler对 引号等进行的转义，
2、自己实现一个ReturnValueHandler去将返回对象转换成 JSONstr。
#### 找到转义的那个HandlerMethodReturnValueHandler

这个应该是比较简单的，我们可以从applicationContext中getBean(class)来获取到具体的values具体代码如下1
2
3
RequestMappingHandlerAdapter handlerAdapter = applicationContext.getBean(RequestMappingHandlerAdapter.class);
List<HandlerMethodReturnValueHandler> handlers = new ArrayList<HandlerMethodReturnValueHandler>();
List<HandlerMethodReturnValueHandler> returnValueHandlers = handlerAdapter.getReturnValueHandlers();

获取到的大概有一下几个:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
ReturnValueHandler  -- 自定义的
ModelAndViewMethodReturnValueHandler
ModelMethodProcessor
ViewMethodReturnValueHandler
ResponseBodyEmitterReturnValueHandler
StreamingResponseBodyReturnValueHandler
HttpEntityMethodProcessor
HttpHeadersReturnValueHandler
CallableMethodReturnValueHandler
DeferredResultMethodReturnValueHandler
AsyncTaskMethodReturnValueHandler
ModelAttributeMethodProcessor
XssRequestResponseBodyMethodProcessor
ViewNameMethodReturnValueHandler
MapMethodProcessor
ModelAttributeMethodProcessor

看下springmvc是如何定位具体使用哪个returnHandler的  代码如下：1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
// HandlerMethodReturnValueHandlerComposite
public void handleReturnValue(Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception &#123;

    HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
    if (handler == null) &#123;
        throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
    &#125;
    handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
&#125;
// 核心在 selectHandler  方法中
private HandlerMethodReturnValueHandler selectHandler(Object value, MethodParameter returnType) &#123;
    boolean isAsyncValue = isAsyncReturnValue(value, returnType);
    for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) &#123;
        if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) &#123;
            continue;
        &#125;
        if (handler.supportsReturnType(returnType)) &#123;
            return handler;
        &#125;
    &#125;
    return null;
&#125;

/*** 可以看出根据 supportReturnType来决定使用哪个handler的，那么我们可以在自定义的returnHandler中
    重写supportReturnType方法，来支持我们需要处理的那些方法，例如 含有@ResponseBody的方法 笔者代码如下：
**/
public boolean supportsReturnType(MethodParameter returnType) &#123;
    return returnType.getMethodAnnotation(ResponseBody.class) != null;
&#125;
// 因为笔者项目全部都是 异步调用返回对象而非 ModelAndView 或者View，所以这么写。

确定handler后 接下来就简单了，只需要在handlerReturnValue中，将Object对象 转换成JSONStr  然后 response.write出去即可 ，笔者代码如下1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
public void handleReturnValue(Object o, MethodParameter methodParameter, ModelAndViewContainer mavContainer, NativeWebRequest nativeWebRequest) throws Exception &#123;
    HttpServletResponse response = nativeWebRequest.getNativeResponse(HttpServletResponse.class);
    String result = JSON.toJSONString(o);
    mavContainer.setRequestHandled(true);
    if (o instanceof OpenWorkbenchResult || o instanceof ApiResult) &#123;
        result = result.replace("\t", "");
        result = result.replace("\n", "");
        result = result.replace("\r", "");
    &#125;
    response.setContentType("application/json;charset=UTF-8");
    response.setHeader("Pragma", "No-cache");
    response.setHeader("Cache-Control", "no-cache");
    response.setDateHeader("Expires", 0);
    response.getWriter().write(result);

&#125;
/***
    if 条件里的 大家尽可以不管，这个是因为笔者项目中  前端json解析服务端返回字段，遇到非法json字符报错，
    所以在这里统一做了处理，如果大家返回json中都是标准的，那么这一步可以不做。
**/
&#125;
