# 【优雅代码】10优雅和前端交互
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/front) 
* [上一篇](./09idea.md)idea调优
* [下一篇](./11stream.md)stream精选示例

## 1.背景
在目前开发中，前后端分离已然成为主流，而后端则有三件事要做，1.不完全信任前端的数据，2.减轻前后端压力3.将接口文档暴露给前端并确保其能看得懂，
## 2.注解边界值
### 2.1官方注解
1. 给接口添加@Validated注解(其可以使用分组更为优秀)
```java
@PostMapping("testFront")
public ResponseVO<FrontRepVO> testFront(@Validated @RequestBody FrontReqVO frontReqVO) {
    return new ResponseVO<>();
}
```
2. 使用javax.validation.constraints下的注解
```java
public class FrontReqVO {
    @NotNull
    private String name;
    @Size(min = 1)
    private String name2;
    // 如果加了其它分组要把default带上，不然会覆盖
    @TrimNotNull(groups = {Default.class, Insert.class})
    // 序列化
    @JsonSerialize(using = TrimNotNullJson.class)
    // 反序列化
    @JsonDeserialize(using = TrimNotNullDeJson.class)
    private String name3;
    @Pattern(regexp = "^[Y|N]$")
    private String yesOrNo;
    @JsonFormat(pattern = "yyyy-MM-dd", timezone = "GMT+8")
    private Date date;
    private ErrorCodeEnum errorCodeEnum;
    private Boolean boo;
}
```

### 2.2自定义注解
有些时候自定义注解不能完全实现自己的功能，这时候就需要自定义注解，比如TrimNotNull，该注解作为用为此字段不可为null，trim后不可为空
1. 添加注解,和官方原注解复制过来即可，关键改动@Constraint中要写该注解的实现类
```java
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(TrimNotNull.List.class)
@Documented
@Constraint(
        validatedBy = {TrimNotNullImpl.class}
)
public @interface TrimNotNull {
    String message() default "不能为null或空";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    @Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface List {
        TrimNotNull[] value();
    }
}
```
2. 自定义注解实现类
```java
public class TrimNotNullImpl implements ConstraintValidator<TrimNotNull, String> {
    @Override
    public void initialize(TrimNotNull constraintAnnotation) {

    }
    @Override
    public boolean isValid(String s, ConstraintValidatorContext constraintValidatorContext) {
        return StringUtils.isNoneBlank(s);
    }
}
```

## 3.统一规范错误
### 3.1错误码
阿里将00000定义为成功，A开头的错误为用户端错误，B开头的错误为服务器错误，C开头的错误为远程调用类错误，根据错误码可以初步定为是怎样的问题,源代码过长就不贴了,该部分参考阿里手册即可
```java
public enum ErrorCodeEnum {
    SUCCESS("00000", "成功", "Success"),
    CLIENT_ERROR("A0001", "用户端错误", "Client error"),
    USER_REGISTRATION_ERROR("A0100", "用户注册错误", "User registration error");
    public static final Map<String, ErrorCodeEnum> MAPS = Stream.of(ErrorCodeEnum.values()).collect(Collectors.toMap(ErrorCodeEnum::getCode, s -> s));

}
```
### 3.2全局异常
为避免将各种错误返回给用户，所以需要对其进行统一封装
* 校验类封装
```java
public class FieldValidError {
    private String field;
    private String msg;
}
```
* 全局异常
```java
@RestControllerAdvice
@Controller
public class GlobalExceptionHandler implements ErrorController {

    /**
     * 404
     *
     * @author seal 876651109@qq.com
     * @date 2020/6/4 1:29 AM
     */
    @RequestMapping(value = "/error")
    @ResponseBody
    public ResponseVO<String> error(HttpServletRequest request, HttpServletResponse response) {
        return ResponseVO.<String>builder().code(ErrorCodeEnum.ADDRESS_NOT_IN_SERVICE.getCode()).msg(ErrorCodeEnum.ADDRESS_NOT_IN_SERVICE.getZhCn()).build();
    }

    /**
     * 其它未定义错误
     *
     * @author seal 876651109@qq.com
     * @date 2020/6/4 12:57 AM
     */
    @ExceptionHandler(Exception.class)
    public ResponseVO<String> handleException(Exception e) {
        log.warn("handleException:" + e.getMessage(), e);
        return ResponseVO.<String>builder().code(ErrorCodeEnum.SYSTEM_EXECUTION_ERROR.getCode()).msg(ErrorCodeEnum.SYSTEM_EXECUTION_ERROR.getZhCn()).build();
    }

    /**
     * 请求方式异常例如get/post
     *
     * @author seal 876651109@qq.com
     * @date 2020/6/4 12:58 AM
     */
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResponseVO<String> HttpRequestMethodNotSupportedException(HttpRequestMethodNotSupportedException e) {
        return ResponseVO.<String>builder().code(ErrorCodeEnum.USER_API_REQUEST_VERSION_MISMATCH.getCode()).msg(ErrorCodeEnum.USER_API_REQUEST_VERSION_MISMATCH.getZhCn()).build();
    }

    /**
     * 重写校验异常，按照统一格式返回
     *
     * @param e
     * @return {@link ResponseVO< List< FieldValidError>>}
     * @author 876651109@qq.com
     * @date 2021/5/14 5:06 下午
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseVO<List<FieldValidError>> MethodArgumentNotValidException(MethodArgumentNotValidException e) {
        ResponseVO<List<FieldValidError>> vo = ResponseVO.<List<FieldValidError>>builder().build();
        List<FieldError> fieldErrors = e.getBindingResult().getFieldErrors();
        List<FieldValidError> list = new ArrayList<>(fieldErrors.size());
        vo.setData(list);
        for (FieldError error : fieldErrors) {
            FieldValidError build = FieldValidError.builder().field(error.getField()).msg(error.getDefaultMessage()).build();
            switch (error.getCode()) {
                case "Size":
                    build.setMsg(String.format("必须在%s到%s之间", error.getArguments()[2], error.getArguments()[1]));
                    break;
                default:
                    break;
            }
            list.add(build);
        }
        return vo;
    }


    @Override
    public String getErrorPath() {
        return null;
    }
}
```

## 4.扩展转换
在日常开发中难免会遇到string转各种奇怪的格式，或预处理，或后置处理，则对其进行扩展非常有必要
1. json格式
* 反序列化，外部数据转对象时
- 写法一
```java
/**
 * 实现去空格反序列化
 *
 * @author 876651109@qq.com
 * @date 2021/5/14 1:46 下午
 */
public class TrimNotNullDeJson extends JsonDeserializer<String> {

    @Override
    public String deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {
        return StringUtils.trim(p.getText());
    }
}
- 写法二
```java
/**
 * 在该注解下，反序列化时会调用该类的这个方法，注意方法要static
 *
 * @author 876651109@qq.com
 * @date 2021/5/14 1:46 下午
 */
@JsonCreator
public static ErrorCodeEnum get(String value) {
    if (StringUtils.isBlank(value)) {
        return null;
    }
    ErrorCodeEnum errorCodeEnum = MAPS.get(value);
    if (errorCodeEnum == null) {
        return ErrorCodeEnum.valueOf(value);
    } else {
        return errorCodeEnum;
    }
}
```

序列化，对象转string时
```java
/**
 * 自定义序列化
 *
 * @author 876651109@qq.com
 * @date 2021/5/14 1:46 下午
 */
public class TrimNotNullJson extends JsonSerializer<String> {
    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(StringUtils.trim(value));
    }
}
```
2. 表单或get请求格式
* 实现Converter，将code转成enum
```java
public class StringToCodeConvert implements Converter<String, ErrorCodeEnum> {
    @Override
    public ErrorCodeEnum convert(String source) {
        return ErrorCodeEnum.MAPS.get(source);
    }
}
```
* 注册
```java
@Configuration
public class SpringMvcConfig extends WebMvcConfigurationSupport {
    @Override
    protected void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToCodeConvert());
    }
}
```

## 5.aop
这部分功能旨在减少后端各种不规范的tryCath日志，可以将其相对统一规范，同时提供了打印入参、返回值、方法耗时等。核心功能为在方法出错时可以打印入参方便定为错误。
1. 创建注解
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface LogAnnotation {
    boolean totalConsume() default true;
    boolean parameter() default false;
    boolean result() default false;
    boolean exception() default true;
}
```
2. 增加aop配置
```java
@Aspect
@Component
@Order(1000)
@Slf4j
public class LogInfoAspect {
    /**
     * 通过@Pointcut注解声明频繁使用的切点表达式
     */
    @Pointcut("execution(* com.example.demo..*.*(..))")
    public void AspectController() {
    }

    @Pointcut("execution(* com.example.demo..*.*(..))")
    public void AspectController2() {
    }

    /**
     * 先执行、先退出
     *
     * @author seal 876651109@qq.com
     * @date 2020/6/3 2:34 PM
     */
    @Around("AspectController() || AspectController2()")
    public Object doAround(ProceedingJoinPoint pjp) throws Throwable {
        log.debug("环绕前");
        MethodSignature methodSignature = (MethodSignature) pjp.getSignature();
        pjp.getTarget();
        LogAnnotation logAnnotationClass = pjp.getTarget().getClass().getAnnotation(LogAnnotation.class);
        LogAnnotation logAnnotationMethod = methodSignature.getMethod().getAnnotation(LogAnnotation.class);
        if (logAnnotationClass == null && logAnnotationMethod == null) {
            return pjp.proceed();
        }
        LogAnnotation logAnnotation = ObjectUtils.defaultIfNull(logAnnotationMethod, logAnnotationClass);
        StopWatch sw = new StopWatch();
        String className = pjp.getTarget().getClass().getName();
        String methodName = methodSignature.getName();
        if (logAnnotation.parameter()) {
            log.info("{}:{}:parameter:{}", className, methodName, pjp.getArgs());
        }
        sw.start();
        Object result = null;
        try {
            result = pjp.proceed();
        } catch (Throwable e) {
            if (logAnnotation.exception()) {
                log.warn(e.getMessage(), e);
                log.info("{}:{}:parameter:{}", className, methodName, pjp.getArgs());
            }
            throw e;
        }
        if (logAnnotation.result()) {
            log.info("{}:{}:result:{}", className, methodName, result);
        }
        sw.stop();
        if (logAnnotation.totalConsume()) {
            log.info("{}:{}:totalConsume:{}s", className, methodName, sw.getTotalTimeSeconds());
        }
        log.debug("环绕后");
        return result;
    }
}
```

## 6.压缩返回值
在yml加入
```java
server:
  compression:
    enabled: true
    mime-types: application/javascript,text/css,application/json,application/xml,text/html,text/xml,text/plain
```
## 7.对接文档
1. swagger-ui
* [官网](https://swagger.io/)
这是一个相对完善的在线api生成器，在编写java的时候直接通过api注解可以辅助生成中文注释，当然没有的话它也会生成一份全部接口的文档，同时支持在线调试，功能十分强大。但对于持久化等显然不够友好，swagger-ui-layer则在此基础上补充了这一点，支持导出，更好看的ui。关于这一块的内容不过多赘述，东西比较简单，网上资料也比较多。
* 引入jar包
```java
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>3.0.0</version>
</dependency>
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-spring-ui</artifactId>
    <version>3.0.2</version>
</dependency>
```
* 加入配置
```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    //http://localhost:8081/swagger-ui.html
    //http://localhost:8081/doc.html

    @Value("${swagger.enable:true}")
    private boolean enable;

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("微服务接口调用文档")
                .enable(enable)
                .pathMapping("/")
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.example.demo"))
                .paths(PathSelectors.any())
                .build().apiInfo(new ApiInfoBuilder()
                        .title("SpringBoot整合Swagger")
                        .description("SpringBoot整合Swagger，详细信息......")
                        .version("9.0")
                        .license("The Apache License")
                        .licenseUrl("http://www.baidu.com")
                        .build());
    }
}
```
![swagger](http://seal_li.gitee.io/sealbook/pic/grace_10front_swagger.png)
2. yapi
* [github](https://github.com/YMFE/yapi)
与swagger不同的是，这个有权限控制，这对于整个大团队维护一份文档则显得尤为重要，其支持多种格式的导入，包括上面提到的swagger。
## 8.方法间使用valid