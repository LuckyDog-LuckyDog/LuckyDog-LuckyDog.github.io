---
title: mvc请求结果封装与全局异常封装
date: 2020-10-10 15:44:53
tags:
- spring mvc
- 请求结果自动封装
- 全局异常
categories:
- Java
---

对web请求的结果进行自动封装 , 对于一些异常,可以用全局异常进行捕获;

<!--more-->

# 1 响应结果自动封装

通常我们业务返回结果处理的时候 , 会进行人为手动对请求的结果的封装成特定的格式 , 这样可以自定义错误码和错误信息 , 但是对于不变的错误码我们要进行多次人为的封装 , 为此可以用下面的操作,下面是Demo , 供学习

## 1.1 设置返回结果实体与返回枚举结果
### 1.1.1 设置结果实体类-Result

```java
package xiaoliu.demo;

/**
 * Description
 *
 * @author:xiaoLiu
 */
public class Result {
    private Integer code;
    private String message;
    private Object data;

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }
    public void setResultCode(ResultCode resultCode){
        this.code = resultCode.getCode();
        this.message = resultCode.getMessage();
    }

    public static Result success(){
        Result result = new Result();
        result.setResultCode(ResultCode.SUCCESS);
        return result;
    }
    public static Result success(Object data){
        Result result = new Result();
        result.setData(data);
        result.setResultCode(ResultCode.SUCCESS);
        return result;
    }
    public static Result failure(ResultCode resultCode){
        Result result = new Result();
        result.setResultCode(resultCode);
        return result;
    }
    public static Result failure(ResultCode resultCode,Object data){
        Result result = new Result();
        result.setData(data);
        result.setResultCode(resultCode);
        return result;
    }
}

```
###1.1.2 设置枚举类-ResultCode
```java
public enum ResultCode {
    SUCCESS(1,"成功"),
    fail(0,"失败"),
    fail2(500,"全局的失败");
    
    private Integer code;
    private String message;

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    ResultCode(Integer code, String message) {
        this.code = code;
        this.message = message;
    }
}
```
## 1.2 设置注解和mvc拦截器
### 1.2.1 设置注解
这里的注解作用是为了判断类上或者方法上是否需要进行请求结果的包装
```java
/**
 * Description
 * @author:xiaoLiu
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE,ElementType.METHOD})
@Documented
public @interface ResponseResult {
}
```
### 1.2.2 设置mvc拦截器
为什么这里使用拦截器 , 不用Filter?  因为Interceptor可以拿到spring框架中的自定义属性 , 而filter不能.

```java
/**
 * Description
 * @author:xiaoLiu
 */
@Component
public class ResultIncepter implements HandlerInterceptor {
	//这是一个请求参数的标识
    public static final String RESPONSE_RESULT_ANN = "RESPONSE_RESULT_ANN ";
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        Log log = Log.get();
        log.info("开始进入拦截器..........");
        if (handler instanceof HandlerMethod) {
            HandlerMethod handlerMethod =(HandlerMethod)handler;
            Method method = handlerMethod.getMethod();
            Class<?> clazz = handlerMethod.getBeanType();
            //类上注解?
            if(clazz.isAnnotationPresent(ResponseResult.class)){
                request.setAttribute(RESPONSE_RESULT_ANN,clazz.getAnnotation(ResponseResult.class));//方法上注解?
            }else if(method.isAnnotationPresent(ResponseResult.class)){
 request.setAttribute(RESPONSE_RESULT_ANN,method.getAnnotation(ResponseResult.class));
            }
        }
        return true;
    }
}

```
## 1.3 将拦截器注入spring
mvc提供了一个接口WebMvcConfigurer 或者使用EnableWebMvc , WebMvcConfigurer配置类其实是Spring内部的一种配置方式，采用JavaBean的形式来代替传统的xml配置文件形式进行针对框架个性化定制，可以自定义一些Handler，Interceptor，ViewResolver，MessageConverter。基于java-based方式的spring mvc配置，需要创建一个配置类并实现WebMvcConfigurer 接口；
在Spring Boot 1.5版本都是靠重写WebMvcConfigurerAdapter的方法来添加自定义拦截器，消息转换器等。SpringBoot 2.0 后，该类被标记为@Deprecated（弃用）。官方推荐直接实现WebMvcConfigurer或者直接继承WebMvcConfigurationSupport，方式一实现WebMvcConfigurer接口（推荐），方式二继承WebMvcConfigurationSupport类，具体实现可看这篇文章。https://blog.csdn.net/fmwind/article/details/82832758

```java
/**
 * Description
 * @author:xiaoLiu
 */
@Component
public class ResultConfig implements WebMvcConfigurer {
    @Autowired
    private ResultIncepter testIncepter;
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(testIncepter);
    }
}
```

## 1.4 设置响应结果包装
ResponseBodyAdvice 源码中注释
```
 * Allows customizing the response after the execution of an {@code @ResponseBody} or a {@code ResponseEntity} controller method but before the body is written with an {@code HttpMessageConverter}. <p>Implementations may be registered directly with {@code RequestMappingHandlerAdapter} and {@code ExceptionHandlerExceptionResolver} or more likely annotated with {@code @ControllerAdvice} in which case they will be auto-detected by both.
```
译文
```
允许在执行{@code@ResponseBody}或{@code ResponseEntity}控制器方法之后，但在使用{@code HttpMessageConverter}编写正文之前，自定义响应。<p>实现可以直接用{@code RequestMappingHandlerAdapter}和{@code ExceptionHandlerExceptionResolver}注册，或者更可能用{@code@ControllerAdvice}进行注释，在这种情况下，它们将由这两种方法自动检测。
```
为此使用@ControllerAdvice注解加上ResponseBodyAdvice< T >实现对响应结果的封装

```java
/**
 * Description
 * @author:xiaoLiu
 */
@RestControllerAdvice
public class ResponseResultHandler implements ResponseBodyAdvice<Object> {
    public static final String RESPONSE_RESULT_ANN = "RESPONSE_RESULT_ANN ";
    public final Log log = Log.get();

    /**
     * 是否对请求结果进行包装
     *
     * @param methodParameter
     * @param aClass
     * @return
     */
    @Override
    public boolean supports(MethodParameter methodParameter, Class<? extends HttpMessageConverter<?>> aClass) {
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = requestAttributes.getRequest();
        ResponseResult result = (ResponseResult) request.getAttribute(RESPONSE_RESULT_ANN);
        if (Objects.nonNull(result)) {
            log.info("请求结果需要进行包装");
            return true;
        }
        log.info("请求结果不需要包装");
        return false;
    }

    @Override
    public Object beforeBodyWrite(Object o, MethodParameter methodParameter, MediaType mediaType, Class<? extends HttpMessageConverter<?>> aClass, ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse) {
        //如果没有全局异常,可以用这个对响应结果进行包装
        if (o instanceof Exception) {
            log.info("异常包装");
            Throwable exception = (Throwable) o;
            StackTraceElement traceElement = ((Exception) o).getStackTrace()[0];
            String tip =traceElement.toString()+"/exception:" + exception.toString();
            return Result.failure(ResultCode.fail, tip);
        }
        
        //已经被全局异常的捕获
        if(o instanceof Result){
            return o;
        }
        
        //注意这里 , 如果业务的返回类型是String类型 , 源码中HttpMessageConverter 会将结果解析为字符串解析器 , 而结果不转为字符串 , 就会出现类型转化异常
        if (o instanceof String) {
            log.info("将响应格式转换为json字符串");
            return JSON.toJSONString(Result.success(o));
        }
        return Result.success(o);
    }
}
```
**注意**: 如果业务的返回类型是String类型 , 源码中HttpMessageConverter 会将结果解析为字符串解析器 , 而结果不转为字符串 , 就会出现类型转化异常

## 1.5 全局异常拦截

```java
/**
 * Description
 *
 * @author:xiaoLiu
 */
@ControllerAdvice
@Order(1)
public class GlobalExceptionHandler{
    public final Log log = Log.get();
	
	//这个后面写,属于validator校验异常
    @ResponseBody
    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public Result bindExceptionHandler( MethodArgumentNotValidException ex) {
        log.info("[bindExceptionHandler]:{}", ex.getBindingResult());
//         拼接错误
        StringBuilder detailMessage = new StringBuilder();
        for (ObjectError objectError : ex.getBindingResult().getAllErrors()) {
            // 使用 ; 分隔多个错误
            if (detailMessage.length() > 0) {
                detailMessage.append(";");
            }
            // 拼接内容到其中
            detailMessage.append(objectError.getDefaultMessage());
        }
        // 包装 CommonResult 结果
        return Result.failure(ResultCode.fail3,ex.getBindingResult());
//        return Result.failure(ResultCode.fail3,new TestRImpl());
    }

    @ExceptionHandler(RuntimeException.class)
    @ResponseBody
    public Result allException(Throwable throwable){
        log.info("被全局异常给捕获了.........");
        String tip = throwable.getStackTrace()[0].toString()+"--->"+throwable.toString();
        return Result.failure(ResultCode.fail2,tip);
    }

}
```
# 2 Demo
```java
/**
 * Description
 *
 * @author:xiaoLiu
 */
@RestController
@RequestMapping("/test")
//自定义包装注解
@ResponseResult
public class ResultController {
	
	//实体
    @RequestMapping("/entity")
    public People test(){
        People people = new People("xiaoliu");
        return people;
    }
    
    //字符串
    @RequestMapping("/string")
    public String test2(@RequestParam(value = "str") String str){
        return str;
    }
	
	//异常
    @RequestMapping("/exception")
    public Exception test3(){
        try {
            Integer num =  1/0 ;
            return null;
        } catch (Exception e) {
            return e;
        }
    }
	
	//全局异常
    @RequestMapping("/global")
    public String test4(){
            Integer num =  1/0 ;
            return "全局哦";
    }
}
```

# 3 参考资料
1. https://www.cnblogs.com/lvbinbin2yujie/p/10574812.html