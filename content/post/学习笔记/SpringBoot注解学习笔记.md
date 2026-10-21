+++
date = '2023-05-08T10:00:00+08:00'
draft = true
title = 'SpringBoot注解学习笔记'
tags = ["SpringBoot","注解","学习笔记"]
series = ["使用教程"]
categories = ["Java", "SpringBoot"]
+++

# SpringBoot注解学习笔记

相关参考资料：
- SpringBoot注解：https://www.cnblogs.com/wylja/p/16837543.html#springboot_1
- 元注解：https://www.jianshu.com/p/8d33ebd176d8

## 1. 元注解介绍

元注解可以理解为注解的注解

```java
@Retention     // 注解的作用 被它所注解的注解保留多久    保留在源文件、class文件、长期存在
/*
* 使用的例子如下
*/
@Retention(RetentionPolicy.SOURCE)        // 保留在源文件
@Retention(RetentionPolicy.CLASS)        // 保留在class文件
@Retention(RetentionPolicy.RUNTIME)        // 长期存在 

@Target    // 注解的作用 修饰这个注解可以用在什么上，注解的作用目标
/*
* 使用的例子如下
*/
@Target(ElementType.TYPE)              // 允许被修饰的注解作用在类、接口和枚举上
@Target(ElementType.FIELD)            // 允许作用在属性字段上
@Target(ElementType.METHOD)            // 允许作用在方法上
@Target(ElementType.PARAMETER)        // 允许作用在方法参数上
@Target(ElementType.CONSTRUCTOR)    // 允许作用在构造器上
@Target(ElementType.LOCAL_VARIABLE)    // 允许作用在本地局部变量上
@Target(ElementType.ANNOTATION_TYPE)    // 允许作用在注解上
@Target(ElementType.PACKAGE)        // 允许作用在包上

@Documented    // 说明该注解将被包含在javadoc中 

@Inherited    // 说明子类可以继承父类中的该注解
```

## 2. SpringBoot的相关注解

### 2.1 springboot启动

#### 2.1.1 @SpringBootApplication

这是 Spring Boot 最核心的注解，用在Spring Boot 主类上，标识这是一个Spring Boot应用，用来开始Spring Boot 的各项能力。

@SpringBootApplication 这个注解是一个复合注解，可以由以下三个注解代替：

1. **@EnableAutoConfiguration** 启用 Spring Boot 的自动配置机制：是Spring Boot 实现自动化配置的核心注解，通过这个注解把Spring应用所需的bean注入容器中。

2. **@ComponentScan** 用于类或接口上主要是指定扫描路径，Spring会把指定路径下带有指定注解的类自动装配到bean容器中。会被自动装配的注解包括 @Controller、@Service、@Component、@Repository 等等。

3. **@SpringBootConfiguration** 允许在Spring 上下文中注册额外的 Bean 或导入其他配置类

### 2.2 springBean相关

- **@Autowired**：标注在属性上，默认是将属性名作为组件的id注入到容器中，自动导入对象到类中
- **@Qualifier**
- **@Resource**
- **@Component**
- **@Service**
- **@Controller**
- **@Repository**
- **@RestController**
- **@Scope**
- **@Configuration**
- **@Bean**
- **@ResponseBody**

### 2.3 REST风格 — 处理常见的HTTP请求类型

- **@GetMapping**
- **@PostMapping**
- **@PutMapping**
- **@DeleteMapping**
- **@RequestMapping**

### 2.4 前后端传值

- **@PathVariable**
- **@RequestParam**
- **@RequestBody**
- **@RequestHeader**
- **@CookieValue**
- **@RequestAttribute**
- **@MatrixVariable**

### 2.5 读取配置信息

- **@Value**
- **@ConfigurationProperties**
- **@PropertySource**

### 2.6 数据库实体类相关

- **@Table**
- **@Entity**
- **@Id**
- **@GeneratedValue**
- **@Param**

### 2.7 事务

- **@Transactional**

### 2.8 测试相关

- **@Test**
- **@ParameterizedTest**
- **@RepeatedTest**
- **@DisplayName**
- **@BeforeEach**
- **@AfterEach**
- **@BeforeAll**
- **@AfterAll**
- **@Tag**
- **@Disabled**
- **@Timeout**
- **@ExtendWith**

### 2.9 整合相关

- **@ComponentScan**
- **@Import**
- **@ImportResource**
- **@Conditional**

## 注解的积累

这个注解作用在实体的属性上，起到时间数据格式转换的功能：

```java
@JsonFormat(pattern = "yyyy-MM-dd")
```
