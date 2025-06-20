---
title: "SpringBoot常用注解总结"
date: "2022-11-26"
tags: ['SpringBoot']
categories: ['编程知识']
---

## 简介

基于 SpringBoot 平台开发的项目数不胜数，与常规的基于Spring开发的项目最大的不同之处，SpringBoot 里面提供了大量的注解用于快速开发，而且非常简单，基本可以做到开箱即用！

## 注解总结

### SpringMVC 相关注解

- @Controller

通常用于修饰`controller`层的组件，由控制器负责将用户发来的URL请求转发到对应的服务接口，通常还需要配合注解`@RequestMapping`使用。

- @RequestMapping

提供路由信息，负责`URL`到`Controller`中具体函数的映射，当用于方法上时，可以指定请求协议，比如`GET`、`POST`、`PUT`、`DELETE`等等。

- @RequestBody

表示请求体的`Content-Type`必须为`application/json`格式的数据，接收到数据之后会自动将数据绑定到`Java`对象上去。

- @ResponseBody

表示该方法的返回结果直接写入`HTTP response body`中，返回数据的格式为`application/json`。

比如，请求参数为json格式，返回参数也为json格式，示例代码如下：
```java
@Controller
@RequestMapping("api")
public class LoginController {

    /**
     * 登录请求，post请求协议，请求参数数据格式为json
     * @param request
     */
    @RequestMapping(value = "login", method = RequestMethod.POST)
    @ResponseBody
    public ResponseEntity login(@RequestBody UserLoginDTO request){
        //...业务处理
        return new ResponseEntity(HttpStatus.OK);
    }
}
```

- @RestController

和`@Controller`一样，用于标注控制层组件，不同的地方在于：它是`@ResponseBody`和`@Controller`的合集，也就是说，在当`@RestController`用在类上时，表示当前类里面所有对外暴露的接口方法，返回数据的格式都为`application/json`。

- @RequestParam

用于接收请求参数为表单类型的数据，通常用在方法的参数前面，示范代码如下：

```java
@RequestMapping(value = "login", method = RequestMethod.POST)
@ResponseBody
public ResponseEntity login(@RequestParam(value = "userName",required = true) String userName,
                            @RequestParam(value = "userPwd",required = true) String userPwd){
    //...业务处理
    return new ResponseEntity(HttpStatus.OK);
}
```

- @PathVariable

用于获取请求路径中的参数，通常用于restful风格的api上，示范代码如下：

```java
@RequestMapping(value = "queryProduct/{id}", method = RequestMethod.POST)
@ResponseBody
public ResponseEntity queryProduct(@PathVariable("id") String id){
    //...业务处理
    return new ResponseEntity(HttpStatus.OK);
}
```

- @GetMapping

除了`@RequestMapping`可以指定请求方式之外，还有一些其他的注解，可以用于标注接口路径请求，比如`GetMapping`用在方法上时，表示只支持`get`请求方法，等价于`@RequestMapping(value="/get",method=RequestMethod.GET)`。

- @PostMapping

用在方法上，表示只支持`post`方式的请求。

- @PutMapping

用在方法上，表示只支持`put`方式的请求，通常表示更新某些资源的意思。

- @DeleteMapping

用在方法上，表示只支持`delete`方式的请求，通常表示删除某些资源的意思。

### bean 相关注解

- @Service

通常用于修饰service层的组件，声明一个对象，会将类对象实例化并注入到bean容器里面。

- @Component

泛指组件，当组件不好归类的时候，可以使用这个注解进行标注，功能类似于于`@Service`。

- @Repository

通常用于修饰`dao`层的组件，`@Repository`注解属于`Spring`里面最先引入的一批注解，它用于将数据访问层 (`DAO层`) 的类标识为`Spring Bean`，具体只需将该注解标注在DAO类上即可，示例代码如下：

```java
@Repository
public interface RoleRepository extends JpaRepository<Role,Long> {

    //具体的方法
}
```

为什么现在使用的很少呢？

主要是因为当我们配置服务启动自动扫描`dao`层包时，`Spring`会自动帮我们创建一个实现类，然后注入到`bean`容器里面。当某些类无法被扫描到时，我们可以显式的在数据持久类上标注`@Repository`注解，`Spring`会自动帮我们声明对象。

- @Bean

相当于 `xml` 中配置 `Bean`，意思是产生一个 `bean` 对象，并交给`spring`管理，示例代码如下：
```java
@Configuration
public class AppConfig {

    //相当于 xml 中配置 Bean
    @Bean
    public Uploader initFileUploader() {
        return new FileUploader();
    }

}
```

- @Autowired

自动导入依赖的`bean`对象，默认时按照`byType`方式导入对象，而且导入的对象必须存在，当需要导入的对象并不存在时，我们可以通过配置`required = false`来关闭强制验证。

- @Resource

也是自动导入依赖的`bean`对象，由`JDK`提供，默认是按照`byName`方式导入依赖的对象；而`@Autowired`默认时按照`byType`方式导入对象，当然`@Resource`还可以配置成通过`byType`方式导入对象。

```java

/**
 * 通过名称导入（默认通过名称导入依赖对象）
 */
@Resource(name = "deptService")
private DeptService deptService;

/**
 * 通过类型导入
 */
@Resource(type = RoleRepository.class)
private DeptService deptService;
```

- @Qualifier

当有多个同一类型的`bean`时，使用`@Autowired`导入会报错，提示当前对象并不是唯一，`Spring`不知道导入哪个依赖，这个时候，我们可以使用`@Qualifier`进行更细粒度的控制，选择其中一个候选者，一般于`@Autowired`搭配使用，示例如下：

```java
@Autowired
@Qualifier("deptService")
private DeptService deptService;
```

- @Scope

用于生命一个`spring bean`的作用域，作用的范围一共有以下几种：

1. singleton：唯一 bean 实例，Spring 中的 bean 默认都是单例的。
2. prototype：每次请求都会创建一个新的 bean 实例，对象多例。
3. request：每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
4. session：每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。

```java
/**
 * 单例对象
 */
@RestController
@Scope("singleton")
public class HelloController {

}
```

### 配置相关注解

- @Configuration

表示声明一个 `Java` 形式的配置类，`Spring Boot` 提倡基于 `Java` 的配置，相当于你之前在 `xml` 中配置 `bean`，比如声明一个配置类`AppConfig`，然后初始化一个`Uploader`对象。

```java
@Configuration
public class AppConfig {

    @Bean
    public Uploader initOSSUploader() {
        return new OSSUploader();
    }

}
```

- @EnableAutoConfiguration

`@EnableAutoConfiguration`可以帮助`Spring Boot`应用将所有符合条件的`@Configuration`配置类，全部都加载到当前`Spring Boot`里，并创建对应配置类的`Bean`，并把该`Bean`实体交给`IoC`容器进行管理。

某些场景下，如果我们想要避开某些配置类的扫描（包括避开一些第三方jar包下面的配置，可以这样处理。
```java
@Configuration
@EnableAutoConfiguration(exclude = { org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration.class})
public class AppConfig {

 //具有业务方法
}
```

- @ComponentScan

标注哪些路径下的类需要被`Spring`扫描，用于自动发现和装配一些`Bean`对象，默认配置是扫描当前文件夹下和子目录下的所有类，如果我们想指定扫描某些包路径，可以这样处理。
```java
@ComponentScan(basePackages = {"com.xxx.a", "com.xxx.b", "com.xxx.c"})
```

- @SpringBootApplication

等价于使用`@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan`这三个注解，通常用于全局启动类上，示例如下：
```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

把`@SpringBootApplication`换成`@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan`这三个注解，一样可以启动成功，`@SpringBootApplication`只是将这三个注解进行了简化！

- @EnableTransactionManagement

表示开启事务支持，等同于 `xml` 配置方式的`<tx:annotation-driven />`

- @Conditional

从 `Spring4` 开始，可以通过`@Conditional`注解实现按条件装载`bean`对象，目前 `Spring Boot` 源码中大量扩展了`@Condition`注解，用于实现智能的自动化配置，满足各种使用场景。下面我给大家列举几个常用的注解：

1. @ConditionalOnBean：当某个特定的Bean存在时，配置生效
2. @ConditionalOnMissingBean：当某个特定的Bean不存在时，配置生效
3. @ConditionalOnClass：当Classpath里存在指定的类，配置生效
4. @ConditionalOnMissingClass：当Classpath里不存在指定的类，配置生效
5. @ConditionalOnExpression：当给定的SpEL表达式计算结果为true，配置生效
6. @ConditionalOnProperty：当指定的配置属性有一个明确的值并匹配，配置生效

具体的应用案例如下：
```java
@Configuration
public class ConditionalConfig {

    /**
     * 当AppConfig对象存在时，创建一个A对象
     */
    @ConditionalOnBean(AppConfig.class)
    @Bean
    public A createA(){
        return new A();
    }

    /**
     * 当AppConfig对象不存在时，创建一个B对象
     */
    @ConditionalOnMissingBean(AppConfig.class)
    @Bean
    public B createB(){
        return new B();
    }

    /**
     * 当KafkaTemplate类存在时，创建一个C对象
     */
    @ConditionalOnClass(KafkaTemplate.class)
    @Bean
    public C createC(){
        return new C();
    }

    /**
     * 当KafkaTemplate类不存在时，创建一个D对象
     */
    @ConditionalOnMissingClass(KafkaTemplate.class)
    @Bean
    public D createD(){
        return new D();
    }

    /**
     * 当enableConfig的配置为true，创建一个E对象
     */
    @ConditionalOnExpression("${enableConfig:false}")
    @Bean
    public E createE(){
        return new E();
    }

    /**
     * 当filter.loginFilter的配置为true，创建一个F对象
     */
    @ConditionalOnProperty(prefix = "filter",name = "loginFilter",havingValue = "true")
    @Bean
    public F createF(){
        return new F();
    }
}
```

- @value

可以在任意 `Spring` 管理的 `Bean` 中通过这个注解获取任何来源配置的属性值，比如你在`application.properties`文件里，定义了一个参数变量！

`config.name=zhangsan`

在任意的`bean`容器里面，可以通过`@Value`注解注入参数，获取参数变量值。

```java
@RestController
public class HelloController {

    @Value("${config.name}")
    private String config;

    @GetMapping("config")
    public String config(){
        return JSON.toJSONString(config);
    }
}
```

- @ConfigurationProperties

上面`@Value`在每个类中获取属性配置值的做法，其实是不推荐的。

一般在企业项目开发中，不会使用那么杂乱无章的写法而且维护也麻烦，通常会一次性读取一个 `Java` 配置类，然后在需要使用的地方直接引用这个类就可以多次访问了，方便维护，示例如下：

首先，在`application.properties`文件里定义好参数变量。

```properties
config.name=demo_1
config.value=demo_value_1
```

然后，创建一个 `Java` 配置类，将参数变量注入即可！

```java
@Component
@ConfigurationProperties(prefix = "config")
public class Config {

    public String name;

    public String value;

    //...get、set
}
```

最后，在需要使用的地方，通过`IOC`注入`Config`对象即可！

- @PropertySource

这个注解是用来读取我们自定义的配置文件的，比如导入`test.properties`和`bussiness.properties`两个配置文件，用法如下：

```java
@SpringBootApplication
@PropertySource(value = {"test.properties","bussiness.properties"})
public class PropertyApplication {

    public static void main(String[] args) {
        SpringApplication.run(PropertyApplication.class, args);
    }
}
```

- @ImportResource

用来加载 `xml` 配置文件，比如导入自定义的`aaa.xml`文件，用法如下：

```java
@ImportResource(locations = "classpath:aaa.xml")
@SpringBootApplication
public class PropertyApplication {

    public static void main(String[] args) {
        SpringApplication.run(PropertyApplication.class, args);
    }
}
```

### 异常处理相关注解

- @ControllerAdvice 和 @ExceptionHandler

通常组合使用，用于处理全局异常，示例代码如下：

```java
@ControllerAdvice
@Configuration
@Slf4j
public class GlobalExceptionConfig {

    private static final Integer GLOBAL_ERROR_CODE = 500;

    @ExceptionHandler(value = Exception.class)
    @ResponseBody
    public void exceptionHandler(HttpServletRequest request, HttpServletResponse response, Exception e) throws Exception {
        log.error("【统一异常处理器】", e);
        ResultMsg<Object> resultMsg = new ResultMsg<>();
        resultMsg.setCode(GLOBAL_ERROR_CODE);
        if (e instanceof CommonException) {
           CommonException ex = (CommonException) e;
           if(ex.getErrCode() != 0) {
                resultMsg.setCode(ex.getErrCode());
            }
            resultMsg.setMsg(ex.getErrMsg());
        }else {
            resultMsg.setMsg(CommonErrorMsg.SYSTEM_ERROR.getMessage());
        }
        WebUtil.buildPrintWriter(response, resultMsg);
    }
}
```

### 测试相关注解

- @ActiveProfiles

一般作用于测试类上， 用于声明生效的 `Spring` 配置文件，比如指定`application-dev.properties`配置文件。

- @RunWith 和 @SpringBootTest

一般作用于测试类上， 用于单元测试用，示例如下：
```java
@ActiveProfiles("dev")
@RunWith(SpringRunner.class)
@SpringBootTest
public class TestJunit {

    @Test
    public void executeTask() {
        //测试...
    }
}
```