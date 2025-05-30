
# Spring 核心面试知识点笔记


## 一、Bean 的生命周期


### 1. 完整生命周期阶段

Spring Bean 的生命周期是从实例化到销毁的完整过程，核心阶段如下（按执行顺序）：

| 阶段                | 说明                                                                 | 关键接口/注解                          |
|---------------------|----------------------------------------------------------------------|----------------------------------------|
| **实例化**          | 通过构造函数创建对象（或工厂方法）                                   | `InstantiationAwareBeanPostProcessor`  |
| **属性注入**        | 依赖注入（如 `@Autowired`、`@Resource`）                             | `AutowiredAnnotationBeanPostProcessor` |
| **Aware 接口回调**  | 感知 Spring 容器信息                                                 | `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware` |
| **BeanPostProcessor 前置处理** | 容器对 Bean 进行自定义修改（如 AOP 代理）       | `postProcessBeforeInitialization`      |
| **初始化**          | 完成业务初始化逻辑                                                   | `InitializingBean`（`afterPropertiesSet`）、`@PostConstruct`、`init-method` |
| **BeanPostProcessor 后置处理** | 初始化后的最终修改                                                   | `postProcessAfterInitialization`       |
| **使用**            | Bean 被业务代码调用                                                 | -                                      |
| **销毁**            | 容器关闭时执行清理逻辑                                               | `DisposableBean`（`destroy`）、`@PreDestroy`、`destroy-method` |


### 2. 生命周期流程图（简化版）

```mermaid
graph TD
    A[实例化] --> B[属性注入]
    B --> C[Aware接口回调]
    C --> D[BeanPostProcessor前置处理]
    D --> E[初始化（@PostConstruct/afterPropertiesSet/init-method）]
    E --> F[BeanPostProcessor后置处理]
    F --> G[使用]
    G --> H[销毁（@PreDestroy/destroy/destroy-method）]
```


### 3. 代码示例：生命周期阶段验证

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

@Component
public class LifecycleBean implements 
    BeanNameAware, BeanFactoryAware, ApplicationContextAware, InitializingBean, DisposableBean {

    // 1. 实例化阶段（构造函数）
    public LifecycleBean() {
        System.out.println("[1] 构造函数调用（实例化）");
    }

    // 2. 属性注入（假设存在依赖注入）
    // @Autowired 会在此阶段执行

    // 3. Aware接口回调
    @Override
    public void setBeanName(String name) {
        System.out.println("[2] BeanNameAware.setBeanName() 调用，Bean名称：" + name);
    }

    @Override
    public void setBeanFactory(BeanFactory factory) throws BeansException {
        System.out.println("[3] BeanFactoryAware.setBeanFactory() 调用");
    }

    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        System.out.println("[4] ApplicationContextAware.setApplicationContext() 调用");
    }

    // 4. BeanPostProcessor 前置处理（由容器自动触发，示例中省略具体实现）

    // 5. 初始化阶段
    @PostConstruct
    public void postConstruct() {
        System.out.println("[5] @PostConstruct 注解方法调用");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("[6] InitializingBean.afterPropertiesSet() 调用");
    }

    // 自定义初始化方法（需在配置中声明）
    public void customInit() {
        System.out.println("[7] 自定义初始化方法 customInit() 调用");
    }

    // 6. BeanPostProcessor 后置处理（由容器自动触发）

    // 7. 使用阶段（业务方法）
    public void doWork() {
        System.out.println("[8] Bean 正在被使用");
    }

    // 8. 销毁阶段
    @PreDestroy
    public void preDestroy() {
        System.out.println("[9] @PreDestroy 注解方法调用");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("[10] DisposableBean.destroy() 调用");
    }

    // 自定义销毁方法（需在配置中声明）
    public void customDestroy() {
        System.out.println("[11] 自定义销毁方法 customDestroy() 调用");
    }
}
```

**配置类声明初始化/销毁方法**：
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class LifecycleConfig {
    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    public LifecycleBean lifecycleBean() {
        return new LifecycleBean();
    }
}
```


## 二、事务及其传播机制


### 1. 事务的 ACID 特性

| 特性         | 说明                                                                 |
|--------------|----------------------------------------------------------------------|
| 原子性（Atomicity） | 事务内操作要么全部成功，要么全部回滚                                 |
| 一致性（Consistency） | 事务执行前后数据状态符合业务规则（如转账后总金额不变）               |
| 隔离性（Isolation） | 多事务并发时，彼此互不干扰（通过隔离级别控制）                       |
| 持久性（Durability） | 事务提交后，数据修改永久保存（依赖数据库持久化机制）                 |


### 2. Spring 事务管理方式

| 方式         | 实现原理               | 优点                                   | 缺点                                   |
|--------------|------------------------|----------------------------------------|----------------------------------------|
| 声明式事务   | 基于 AOP 自动代理       | 无侵入性，代码简洁（通过 `@Transactional` 注解） | 依赖 AOP 环境，无法细粒度控制事务边界   |
| 编程式事务   | 手动调用 `TransactionTemplate` | 灵活控制事务边界（如条件提交/回滚）     | 代码冗余，侵入业务逻辑                 |


### 3. 事务传播机制（7种）

传播机制定义了**多个事务方法嵌套调用时**，事务如何合并或独立执行。Spring 提供以下 7 种传播行为：

| 传播行为          | 符号常量                | 说明                                                                 | 典型场景                     |
|-------------------|-------------------------|----------------------------------------------------------------------|------------------------------|
| `REQUIRED`        | `TransactionDefinition.PROPAGATION_REQUIRED` | **默认**：当前有事务则加入，无则新建事务 | 大多数业务场景（如用户注册+日志记录） |
| `REQUIRES_NEW`    | `TransactionDefinition.PROPAGATION_REQUIRES_NEW` | 新建独立事务，挂起当前事务（若存在） | 需要隔离失败的子操作（如支付回调）   |
| `NESTED`          | `TransactionDefinition.PROPAGATION_NESTED` | 在当前事务的嵌套事务中执行（支持部分回滚） | 可补偿的子操作（如订单分阶段处理）   |
| `SUPPORTS`        | `TransactionDefinition.PROPAGATION_SUPPORTS` | 当前有事务则加入，无则非事务执行       | 读操作（如查询详情）           |
| `MANDATORY`       | `TransactionDefinition.PROPAGATION_MANDATORY` | 必须存在当前事务，否则抛异常           | 敏感操作（如账户扣款）         |
| `NOT_SUPPORTED`   | `TransactionDefinition.PROPAGATION_NOT_SUPPORTED` | 非事务执行，挂起当前事务（若存在）     | 无需事务的耗时操作（如发送通知）   |
| `NEVER`           | `TransactionDefinition.PROPAGATION_NEVER` | 必须非事务执行，否则抛异常             | 禁止事务的场景（如纯查询）     |


### 4. 代码示例：传播机制演示

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private LogService logService;

    // 默认传播行为：REQUIRED（主事务）
    @Transactional(rollbackFor = Exception.class)
    public void createOrder(String productId) throws Exception {
        // 1. 扣减库存（加入当前事务）
        inventoryService.deductInventory(productId, 1);
        
        // 2. 记录操作日志（加入当前事务）
        logService.recordLog("创建订单：" + productId);
        
        // 3. 模拟异常（触发主事务回滚）
        throw new RuntimeException("订单创建失败");
    }

    // 传播行为：REQUIRES_NEW（独立事务）
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void retryDeductInventory(String productId) {
        inventoryService.deductInventory(productId, 1);
    }

    // 传播行为：NESTED（嵌套事务）
    @Transactional(propagation = Propagation.NESTED, rollbackFor = Exception.class)
    public void partialUpdate(String productId) {
        inventoryService.updateInventoryStatus(productId, "LOCKED");
    }
}

@Service
class InventoryService {
    // 假设操作数据库的方法（省略 DAO 细节）
    @Transactional
    public void deductInventory(String productId, int count) {
        // 扣减库存 SQL：UPDATE inventory SET count = count - #{count} WHERE product_id = #{productId}
    }
}

@Service
class LogService {
    @Transactional
    public void recordLog(String content) {
        // 记录日志 SQL：INSERT INTO operation_log (content) VALUES (#{content})
    }
}
```


## 三、Spring 中常见设计模式及应用


### 1. 单例模式（Singleton）

**定义**：确保一个类仅有一个实例，并提供全局访问点。  
**Spring 应用**：Bean 默认作用域为 `singleton`（单例），由容器管理实例创建。

```java
// Spring 中默认单例（无需额外配置）
@Component
public class AppConfig {
    // 私有构造防止外部实例化
    private AppConfig() {}

    public void printConfig() {
        System.out.println("单例配置对象");
    }
}

// 验证单例：两次获取同一 Bean 是同一个实例
public class Test {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        AppConfig bean1 = context.getBean(AppConfig.class);
        AppConfig bean2 = context.getBean(AppConfig.class);
        System.out.println(bean1 == bean2); // 输出：true
    }
}
```


### 2. 工厂模式（Factory）

**定义**：通过工厂类创建对象，解耦对象创建逻辑。  
**Spring 应用**：`BeanFactory` 是核心工厂接口，负责创建和管理所有 Bean。

```java
// 工厂接口（Spring 内置）
public interface BeanFactory {
    Object getBean(String name) throws BeansException;
    // ... 其他方法
}

// 使用示例：通过工厂获取 Bean
public class FactoryDemo {
    public static void main(String[] args) {
        // ClassPathXmlApplicationContext 是 BeanFactory 的子接口实现
        BeanFactory factory = new ClassPathXmlApplicationContext("applicationContext.xml");
        UserService userService = factory.getBean("userService", UserService.class);
        userService.createUser();
    }
}
```


### 3. 代理模式（Proxy）

**定义**：通过代理对象控制对原对象的访问（如增强功能）。  
**Spring 应用**：AOP（面向切面编程）通过动态代理（JDK/CGLIB）实现方法拦截。

```java
// 业务接口
public interface UserService {
    void createUser(String username);
}

// 真实对象
@Service
public class UserServiceImpl implements UserService {
    @Override
    public void createUser(String username) {
        System.out.println("创建用户：" + username);
    }
}

// 切面（代理逻辑）
@Aspect
@Component
public class LogAspect {
    // 定义切入点：匹配 UserService 的所有方法
    @Pointcut("execution(* com.example.service.UserService.*(..))")
    public void userServicePointcut() {}

    // 前置通知：方法执行前打印日志
    @Before("userServicePointcut()")
    public void beforeCreateUser(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        System.out.println("准备执行方法：" + methodName + "，参数：" + Arrays.toString(args));
    }
}

// 效果：调用 userService.createUser("张三") 时，会先执行 LogAspect 的前置通知
```


### 4. 观察者模式（Observer）

**定义**：对象间一对多依赖，目标对象状态变更时通知所有观察者。  
**Spring 应用**：事件驱动模型（`ApplicationEvent` + `ApplicationListener`）。

```java
// 1. 自定义事件（继承 ApplicationEvent）
public class UserRegisterEvent extends ApplicationEvent {
    private final String username;

    public UserRegisterEvent(Object source, String username) {
        super(source);
        this.username = username;
    }

    public String getUsername() {
        return username;
    }
}

// 2. 事件发布者（触发事件）
@Service
public class UserService {
    @Autowired
    private ApplicationEventPublisher eventPublisher;

    public void registerUser(String username) {
        System.out.println("用户注册成功：" + username);
        // 发布事件，通知所有监听器
        eventPublisher.publishEvent(new UserRegisterEvent(this, username));
    }
}

// 3. 事件监听器（方式1：实现 ApplicationListener）
@Component
public class SmsListener implements ApplicationListener<UserRegisterEvent> {
    @Override
    public void onApplicationEvent(UserRegisterEvent event) {
        System.out.println("发送短信通知：用户 " + event.getUsername() + " 注册成功");
    }
}

// 3. 事件监听器（方式2：使用 @EventListener 注解）
@Component
public class LogListener {
    @EventListener
    public void handleUserRegister(UserRegisterEvent event) {
        System.out.println("记录日志：用户 " + event.getUsername() + " 注册时间：" + new Date());
    }
}
```


### 5. 策略模式（Strategy）

**定义**：定义算法族，使其可相互替换，客户端动态选择。  
**Spring 应用**：资源加载策略（`ResourceLoader`）、消息转换器（`HttpMessageConverter`）。

```java
// 策略接口（Spring 内置的资源加载策略）
public interface ResourceLoader {
    Resource getResource(String location);
}

// 具体策略1：ClassPathResourceLoader（从类路径加载资源）
public class ClassPathResourceLoader implements ResourceLoader {
    @Override
    public Resource getResource(String location) {
        return new ClassPathResource(location);
    }
}

// 具体策略2：FileSystemResourceLoader（从文件系统加载资源）
public class FileSystemResourceLoader implements ResourceLoader {
    @Override
    public Resource getResource(String location) {
        return new FileSystemResource(location);
    }
}

// 客户端（动态选择策略）
@Service
public class ResourceService {
    @Autowired
    private ResourceLoader resourceLoader; // 由 Spring 注入具体策略（默认是 ClassPathResourceLoader）

    public void loadResource(String path) {
        Resource resource = resourceLoader.getResource(path);
        // 处理资源...
    }
}
```


## 总结

本文覆盖了 Spring 面试中高频考察的三个核心模块：  
- **Bean 生命周期**：需掌握各阶段顺序及关键接口/注解的作用；  
- **事务传播机制**：重点理解 `REQUIRED`、`REQUIRES_NEW`、`NESTED` 的区别及适用场景；  
- **设计模式应用**：结合 Spring 源码（如 AOP、事件机制）理解模式如何解决实际问题。