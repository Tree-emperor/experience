

---

# SpringBoot 学习笔记

## 一、IOC（控制反转）

### 1.1 含义
IOC（Inversion of Control，控制反转）是一种设计原则，将对象的创建权和依赖管理权从应用程序代码转移到外部容器（Spring容器）。

### 1.2 为什么有IOC
传统开发模式下，对象之间的依赖关系由程序内部代码直接创建和管理，导致：
- **高度耦合**：类之间相互依赖，修改一处可能影响全局
- **难以测试**：依赖对象无法替换，单元测试困难
- **代码复用性差**：对象创建逻辑散落在各处
- **扩展困难**：添加新功能需要修改大量现有代码

### 1.3 如何实现
Spring通过**依赖注入（Dependency Injection，DI）**实现IOC：

```java
// 传统方式：对象自己创建依赖
public class UserService {
    private UserDao userDao = new UserDaoImpl(); // 硬编码耦合
}

// IOC方式：由容器注入依赖
public class UserService {
    private UserDao userDao;
    
    public UserService(UserDao userDao) {
        this.userDao = userDao; // 依赖由外部注入
    }
}
```

注入方式：
- **构造器注入**：通过构造函数注入
- **Setter注入**：通过setter方法注入
- **字段注入**：通过反射直接注入字段

### 1.4 优势
- **松耦合**：对象间依赖关系由配置决定，便于替换实现
- **易于测试**：可以注入mock对象进行单元测试
- **单例模式**：容器管理的bean默认单例，提高性能
- **生命周期管理**：容器负责对象的创建、初始化、销毁

### 1.5 实例
```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService(userDao());
    }
    
    @Bean
    public UserDao userDao() {
        return new JdbcUserDao();
    }
}
```

### 1.6 实现机制

#### 1.6.1 Spring容器体系

```
BeanFactory（根接口）
    ↓
    ├── ApplicationContext（应用上下文）
    │       ├── ClassPathXmlApplicationContext（XML配置）
    │       ├── FileSystemXmlApplicationContext（XML配置）
    │       └── AnnotationConfigApplicationContext（注解配置）
    │
    └── ListableBeanFactory（扩展接口，可枚举bean）
```

- **BeanFactory**：Spring最底层接口，负责bean的创建和依赖注入，采用延迟加载（第一次getBean时才创建bean）
- **ApplicationContext**：BeanFactory的子接口，在启动时预加载所有单例bean，提供国际化、事件发布等高级特性

#### 1.6.2 Bean生命周期

Spring容器管理的bean从创建到销毁的完整过程：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Bean生命周期                                │
├─────────────────────────────────────────────────────────────────┤
│  1. 实例化（Instantiation）                                      │
│     └─ 调用构造函数创建bean实例                                   │
│                                                                  │
│  2. 属性填充（Populate）                                         │
│     └─ 通过setter方法注入依赖的bean                               │
│                                                                  │
│  3. 初始化（Initialization）                                     │
│     ├─ Aware接口回调（BeanNameAware、BeanFactoryAware等）         │
│     ├─ BeanPostProcessor.postProcessBeforeInitialization       │
│     ├─ @PostConstruct注解方法                                   │
│     ├─ InitializingBean.afterPropertiesSet()                   │
│     └─ 自定义init-method                                       │
│                                                                  │
│  4. 使用（In Use）                                               │
│     └─ bean处于就绪状态，可被应用程序使用                          │
│                                                                  │
│  5. 销毁（Destruction）                                          │
│     ├─ @PreDestroy注解方法                                      │
│     ├─ DisposableBean.destroy()                                │
│     └─ 自定义destroy-method                                     │
└─────────────────────────────────────────────────────────────────┘
```

#### 1.6.3 依赖注入实现原理

**构造器注入实现原理**：
```java
// Spring通过反射调用构造函数，并自动注入所需的依赖
public class UserService {
    private UserDao userDao;
    
    // Spring通过反射找到这个构造函数，并注入UserDao实例
    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

**Setter注入实现原理**：
```java
public class UserService {
    private UserDao userDao;
    
    // Spring通过反射调用setter方法注入依赖
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

**字段注入实现原理**：
```java
public class UserService {
    // Spring通过反射直接设置字段值（即使是private）
    @Autowired
    private UserDao userDao;
}
```

核心原理：**反射（Reflection）**。Spring利用Java的反射API，在运行时：
1. 获取类的构造函数、setter方法、字段信息
2. 通过`Constructor.newInstance()`创建实例
3. 通过`Method.invoke()`调用setter方法
4. 通过`Field.setAccessible(true)`后直接设置字段值

#### 1.6.4 容器启动与bean加载流程

```
ApplicationContext启动流程：

1. 扫描（Scanning）
   └─ @ComponentScan指定包路径，扫描所有@Component衍生注解

2. 注册（Registering）
   └─ 将扫描到的bean定义（BeanDefinition）注册到BeanFactory

3. 刷新（Refreshing）
   ├─ 创建单例bean实例
   ├─ 填充bean属性（依赖注入）
   ├─ 执行初始化回调
   └─ 发布容器刷新事件

4. 就绪（Ready）
   └─ 所有单例bean已创建完毕，容器可正常使用
```

#### 1.6.5 循环依赖处理

当两个bean互相依赖时，Spring通过**三级缓存**解决：

```java
// 三级缓存
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);      // 一级：成品bean
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);  // 二级：提前暴露的引用
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);     // 三级：bean工厂
```

处理流程：
1. 创建A时，发现依赖B，将A的工厂放入三级缓存
2. 创建B时，发现依赖A，从三级缓存获取A的工厂，创建A的引用放入二级缓存
3. B创建完成，填充到A中
4. A创建完成，从二级缓存获取完整A

**注意**：构造器注入的循环依赖无法解决，Spring会抛出`BeanCurrentlyInCreationException`。

---

## 二、AOP（面向切面编程）

### 2.1 含义
AOP（Aspect-Oriented Programming）是一种编程范式，旨在将**横切关注点**（如日志、事务、安全）与业务逻辑分离。

### 2.2 为什么有AOP
传统面向对象编程中，日志、事务、安全等功能会散布在多个模块中，造成：
- **代码重复**：相同逻辑在多处重复实现
- **业务逻辑不纯粹**：核心业务被通用逻辑污染
- **维护成本高**：修改一个功能需改多处代码
- **耦合度高**：业务代码与系统级代码强耦合

### 2.3 如何实现
Spring AOP基于**代理模式**实现：

```java
// 目标对象
public class UserService {
    public void addUser(User user) {
        System.out.println("添加用户业务逻辑");
    }
}

// 代理对象（事务管理器）
public class UserServiceProxy {
    private UserService target;
    
    public void addUser(User user) {
        // 前置通知
        beginTransaction();
        try {
            target.addUser(user);
            // 后置通知
            commit();
        } catch (Exception e) {
            rollback();
            throw e;
        }
    }
}
```

Spring AOP关键概念：
- **Join Point（连接点）**：程序执行的某个位置
- **Pointcut（切点）**：匹配连接点的表达式
- **Advice（通知）**：增强逻辑（前置、后置、环绕等）
- **Aspect（切面）**：切点 + 通知的组合
- **Weaving（织入）**：将切面应用到目标对象的过程

### 2.4 优势
- **分离关注点**：业务逻辑与系统级逻辑解耦
- **代码复用**：通知只定义一次，多处使用
- **维护性高**：修改通知逻辑只需改一处
- **业务代码纯净**：开发者专注业务逻辑

### 2.5 AOP的应用场景
- **日志记录**：方法入口出口记录日志
- **事务管理**：方法执行前后开启/提交事务
- **安全控制**：方法执行前校验权限
- **性能监控**：方法执行前后计时
- **异常处理**：统一异常捕获和处理

### 2.6 实例
```java
@Aspect
@Component
public class TransactionAspect {
    
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void servicePointcut() {}
    
    @Around("servicePointcut()")
    public Object around(ProceedingJoinPoint joinPoint) {
        Object result = null;
        try {
            beginTransaction();
            result = joinPoint.proceed();
            commit();
        } catch (Throwable e) {
            rollback();
            throw new RuntimeException(e);
        }
        return result;
    }
}
```

---

## 三、SpringBoot相比Spring的优势

| 特性 | Spring | SpringBoot |
|------|--------|------------|
| 配置 | 大量XML配置 | 自动配置（auto-configuration） |
| 依赖 | 手动管理版本和兼容 | starter自动依赖管理 |
| 内嵌服务器 | 需手动配置Tomcat | 内嵌Tomcat/Jetty |
| 监控 | 手动集成 | Actuator提供健康监控 |
| 开发效率 | 低 | 高（开箱即用） |

### 3.1 优势详解

**1. 自动配置（Auto-Configuration）**
```java
// Spring需要手动配置数据源
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <!-- 大量配置... -->
</bean>

// SpringBoot只需配置
spring.datasource.url=jdbc:mysql://localhost:3306/demo
spring.datasource.username=root
spring.datasource.password=123456
```

**2. Starter依赖**
```xml
<!-- 一个依赖搞定一切 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
会自动引入SpringMVC、Jackson、Tomcat等所有相关依赖。

**3. 内嵌服务器**
- 打成的jar包直接`java -jar`运行，无需部署到外部Tomcat
- 默认使用Tomcat（也可切换为Jetty或Undertow）

**4. 生产级特性**
- 健康检查、指标监控、远程管理
- 外部化配置（application.properties/yml）
- 零代码生成

---

## 四、动态代理

### 4.1 什么是动态代理
动态代理是在运行时动态生成代理类的代理机制，无需预先编写代理类代码。

### 4.2 JDK动态代理
基于接口的代理，需要目标类实现接口：

```java
public class JdkProxyFactory {
    public static Object createProxy(Object target) {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    System.out.println("方法执行前...");
                    Object result = method.invoke(target, args);
                    System.out.println("方法执行后...");
                    return result;
                }
            }
        );
    }
}

// 使用
UserService userService = (UserService) JdkProxyFactory.createProxy(new UserServiceImpl());
userService.addUser(user);
```

### 4.3 CGLIB动态代理
基于继承的代理，无需接口：

```java
public class CglibProxyFactory {
    public static Object createProxy(Object target) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(new MethodInterceptor() {
            @Override
            public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                System.out.println("方法执行前...");
                Object result = proxy.invokeSuper(obj, args);
                System.out.println("方法执行后...");
                return result;
            }
        });
        return enhancer.create();
    }
}
```

### 4.4 与AOP的关系
**AOP是动态代理的应用场景之一**：

```
AOP实现 → 动态代理（核心机制）
         ↓
    Spring AOP使用两种代理：
    1. JDK动态代理（目标对象实现接口）
    2. CGLIB代理（目标对象未实现接口或配置proxy-target-class=true）
```

Spring AOP的工作流程：
1. 当调用被切面匹配的方法时
2. 调用不会直接到目标对象
3. 而是通过代理对象（动态生成）
4. 代理对象执行通知逻辑
5. 再调用目标对象的原方法

**这就是AOP能够"无侵入"增强功能的根本原因**。

---

# DDD（领域驱动设计）与Cola架构

## 一、DDD含义
DDD（Domain-Driven Design，领域驱动设计）是一种软件开发方法论，通过**领域模型**（Domain Model）来捕获业务知识，将业务逻辑作为核心，使技术架构服务于业务复杂性。

## 二、核心概念

### 2.1 战略设计（Strategic Design）
- **领域（Domain）**：业务知识所属的领域范围
- **子域（Subdomain）**：将大领域拆分为多个小领域
  - 核心域（Core Domain）：业务核心竞争力，必须重点投入
  - 支撑域（Supporting Domain）：必要但非核心的功能
  - 通用域（Generic Domain）：可直接采购的通用解决方案
- **限界上下文（Bounded Context）**：明确领域的边界，避免概念歧义
- **上下文映射（Context Mapping）**：不同限界上下文之间的集成关系

### 2.2 战术设计（Tactical Design）
- **实体（Entity）**：具有唯一标识，且生命周期内标识不变的对象
- **值对象（Value Object）**：无唯一标识，通过属性值定义相等的对象
- **聚合（Aggregate）**：一组相关对象的组合，由聚合根统一对外暴露
- **仓储（Repository）**：封装聚合的持久化逻辑
- **领域服务（Domain Service）**：不属于任何实体的业务逻辑
- **领域事件（Domain Event）**：领域中发生的业务事件

**领域服务（Domain Service）详解**：

当某个业务逻辑不属于任何实体或值对象时，将其放在领域服务中。例如：转账操作涉及两个账户实体，计算转账金额是否合法、转账手续费等逻辑放在领域服务中。

```java
// 领域服务：不属于任何实体的跨实体业务逻辑
public class TransferDomainService {
    
    public void transfer(Account sourceAccount, Account targetAccount, Money amount) {
        // 1. 校验源账户是否足够余额
        if (sourceAccount.getBalance().lessThan(amount)) {
            throw new InsufficientBalanceException();
        }
        
        // 2. 校验转账金额是否在有效范围内
        if (!amount.isWithinTransferLimit()) {
            throw new TransferLimitExceededException();
        }
        
        // 3. 执行转账：扣款 + 入账
        sourceAccount.debit(amount);
        targetAccount.credit(amount);
        
        // 4. 生成领域事件
        sourceAccount.publishEvent(new AccountDebitedEvent(amount));
        targetAccount.publishEvent(new AccountCreditedEvent(amount));
    }
}
```

**领域事件（Domain Event）详解**：

领域中发生的业务事件，用于解耦服务间通信。当聚合状态发生变化时，发布领域事件，订阅者异步处理后续逻辑。


**领域事件的核心价值**：
1. **松耦合**：发布者不需要知道订阅者是谁，实现服务间解耦
2. **最终一致性**：通过事件驱动实现多个聚合间的状态同步
3. **可追溯性**：事件日志可记录完整的业务轨迹

## 三、Cola架构

Cola是阿里巴巴团队开源的一套**COLA（Cross-layer Optimization & Lattice Architecture）**架构，设计的核心思想是**将业务逻辑与技术细节解耦**，使系统具备更高的可维护性和扩展性。

### 3.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户层（User Layer）                       │
│              Web/API/UI 等外部入口                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ↑
┌─────────────────────────────────────────────────────────────────┐
│                    应用层（Application Layer）                    │
│         Command / Query / Event Handler（用例编排）               │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ↑
┌─────────────────────────────────────────────────────────────────┐
│                     领域层（Domain Layer）                        │
│    Domain（实体/值对象/领域服务）  │  Repo（仓储接口）             │
│    Event（领域事件）              │  DomainService               │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ↑
┌─────────────────────────────────────────────────────────────────┐
│                     基础设施层（Infrastructure Layer）            │
│        Repository实现 / 外部服务调用 / 配置 / 平台能力             │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 各层职责

| 层级 | 职责 | 典型组件 |
|------|------|---------|
| **用户层** | 处理用户请求适配，协议转换 | Controller、Filter、DTO转换 |
| **应用层** | 用例编排，事务控制，权限校验 | CommandHandler、QueryHandler、EventHandler |
| **领域层** | 业务规则校验，业务逻辑实现，领域模型 | Entity、ValueObject、Aggregate、DomainService |
| **基础设施层** | 技术实现，持久化，外部服务调用 | MyBATIS实现、RPC调用、缓存、消息队列 |

### 3.3 代码结构示例

```
com.example.order
├── user（用户限界上下文）
│   ├── controller（用户层）
│   │   └── UserController.java
│   ├── app（应用层）
│   │   ├── cmd（命令）
│   │   │   ├── CreateUserCmd.java
│   │   │   └── UpdateUserCmd.java
│   │   ├── query（查询）
│   │   │   └── UserByIdQuery.java
│   │   └── handler（处理器）
│   │       └── UserCommandHandler.java
│   ├── domain（领域层）
│   │   ├── model（领域模型）
│   │   │   ├── User.java（实体）
│   │   │   └── UserId.java（值对象）
│   │   ├── service（领域服务）
│   │   │   └── UserDomainService.java
│   │   ├── event（领域事件）
│   │   │   └── UserCreatedEvent.java
│   │   └── repo（仓储接口）
│   │       └── UserRepository.java
│   └── infrastructure（基础设施层）
│       ├── persistence（持久化）
│       │   └── MybatisUserRepository.java
│       └── external（外部服务）
│           └── UserRemoteService.java
```

## 四、为何使用Cola架构

### 4.1 解决的核心问题
- **业务逻辑散乱**：传统项目中业务逻辑分散在Controller、Service甚至Util中
- **领域边界模糊**：缺乏清晰限界上下文，领域概念互相渗透
- **技术耦合业务**：数据库、缓存、RPC等技术与业务代码高度耦合
- **难以测试**：业务逻辑依赖外部组件，单元测试困难
- **团队协作困难**：多人协作时代码冲突频繁，职责不清

### 4.2 Cola的核心价值
1. **职责分离**：每层只关注自己的职责，代码边界清晰
2. **领域内聚**：业务逻辑收拢到领域层，技术细节下沉到基础设施层
3. **可测试性**：领域层无外部依赖，可直接单元测试
4. **可扩展性**：新增功能只需在对应层添加代码，不影响其他层
5. **团队协作**：不同团队可独立开发不同限界上下文

## 五、Cola与DDD的关系（如何落地DDD）

Cola架构是DDD的**一种落地实现方式**，它将DDD的战术设计思想结构化为四层架构：

```
DDD概念          →    Cola实现
─────────────────────────────────
Entity/Aggregate  →    domain.model（领域模型）
Value Object      →    domain.model（值对象）
Domain Service    →    domain.service（领域服务）
Repository接口    →    domain.repo（仓储接口）
Repository实现    →    infrastructure.persistence
领域事件          →    domain.event + infrastructure.messaging
Application Service →  app.handler（应用处理器）
Command/Query     →    app.cmd / app.query（命令查询对象）
```

**Cola落地DDD的关键实践**：

1. **以领域为核心建模**
   - 通过战略设计划分限界上下文
   - 通过战术设计定义领域模型（实体、值对象、聚合）
   - 领域模型是**贫血模型**还是**充血模型**的选择：
     - 贫血模型：领域对象只有getter/setter，业务逻辑在DomainService
     - 充血模型：领域对象包含业务行为，更符合DDD思想

2. **依赖方向约束**
   - 外层依赖内层，内层不依赖外层
   - 领域层是核心，不依赖任何外部技术

3. **仓储模式隔离持久化**
   - 领域层定义仓储接口（纯粹的业务接口）
   - 基础设施层实现仓储（技术实现细节）

## 六、Cola与MVC对比

| 维度 | MVC架构 | Cola架构 |
|------|---------|----------|
| **设计思想** | 前后端分离 + 分层 | 领域建模 + 洋葱架构 |
| **复杂度适用** | 简单业务场景 | 复杂业务场景 |
| **业务逻辑位置** | Service层（容易膨胀） | 领域层（业务收拢） |
| **领域模型** | 贫血模型（只有数据） | 充血模型（数据+行为） |
| **耦合度** | 技术与业务耦合 | 技术与业务解耦 |
| **可测试性** | 较低（依赖Service实现） | 高（领域层可独立测试） |
| **事务边界** | 通常在Service层 | 应用层（更精准） |
| **适用场景** | 小型项目、CRUD为主 | 中大型项目、业务复杂 |

**MVC的典型问题**：
下面用一个具体例子来说明：“用户下单资格预检”。
这个业务不属于订单实体，也不属于用户实体，而是跨实体、跨数据源、跨外部服务的领域业务。它需要：

查订单库：用户今日已下单次数；

查用户库：用户等级、是否黑名单；

调用外部风控服务：得到风险评分；

根据以上信息执行规则：

- 今日下单次数 ≥ 10：拒绝；
- 用户是黑名单：拒绝；
- 普通用户风控分 < 80：拒绝；
- VIP 用户风控分 < 60：拒绝；
- 其余：允许下单。

传统 mvc 实现：


```java

@Service
public class PreOrderCheckService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private RiskFeignClient riskFeignClient;

    public PreOrderCheckVO check(Long userId) {
        // 查订单库
        int todayOrderCount = orderMapper.countTodayByUserId(userId);

        // 查用户库
        UserDO user = userMapper.selectById(userId);
        if (user == null) {
            throw new BizException("用户不存在");
        }

        // 调外部风控服务
        RiskDTO risk = riskFeignClient.assess(userId);
        if (risk == null || risk.getScore() == null) {
            throw new BizException("风控服务异常");
        }

        String result = "ALLOW";
        String reason = "";

        // 业务规则
        if (todayOrderCount >= 10) {
            result = "REJECT";
            reason = "今日下单次数过多";
        } else if (user.getStatus() == 1) {
            result = "REJECT";
            reason = "用户被拉黑";
        } else if (user.getLevel() == 0 && risk.getScore() < 80) {
            result = "REJECT";
            reason = "风控评分不足";
        } else if (user.getLevel() > 0 && risk.getScore() < 60) {
            result = "REJECT";
            reason = "风控评分不足";
        }

        PreOrderCheckVO vo = new PreOrderCheckVO();
        vo.setResult(result);
        vo.setReason(reason);
        vo.setScore(risk.getScore());
        vo.setTodayOrderCount(todayOrderCount);
        return vo;
    }
}

```

**MVC 的问题**
- 业务规则被技术代码淹没
这个 Service 里既有数据库访问，又有外部服务调用，还有业务规则。规则只是一堆 if-else，很难看出这是“下单资格预检领域规则”。

- Service 直接依赖 MyBatis Mapper 和 Feign Client
业务层直接知道了：
订单数据来自哪个表；
用户数据来自哪张表；
风控服务是 Feign 调用；
外部返回结构是 RiskDTO。
外部服务字段一改，核心业务 Service 就要改。

- 难以单元测试
要测试这段业务规则，必须启动 Spring 容器，或者 Mock 掉 OrderMapper、UserMapper、RiskFeignClient。
本来核心逻辑只是几个规则判断，但测试成本变得很高。

- 业务规则散落、重复
如果另一个业务也要判断“用户是否黑名单”，可能又复制一遍 user.getStatus() == 1。
时间一长，规则会散落到各个 Service、Controller、工具类里。

**COLA 优势**

核心领域服务的实现：


```java

public class PurchaseEligibilityEvaluator {

    private static final int MAX_ORDER_COUNT = 10;
    private static final int NORMAL_RISK_THRESHOLD = 80;
    private static final int VIP_RISK_THRESHOLD = 60;

    public EligibilityResult evaluate(UserProfile profile,
                                      TodayOrderCount orderCount,
                                      RiskScore riskScore) {
        if (orderCount.getValue() >= MAX_ORDER_COUNT) {
            return reject("今日下单次数过多");
        }

        if (profile.getStatus() == UserStatus.BLACKLISTED) {
            return reject("用户被拉黑");
        }

        int threshold = profile.getLevel() == UserLevel.VIP
                ? VIP_RISK_THRESHOLD
                : NORMAL_RISK_THRESHOLD;

        if (riskScore.getValue() < threshold) {
            return reject("风控评分不足");
        }

        return allow();
    }

    private EligibilityResult reject(String reason) {
        return new EligibilityResult("REJECT", reason);
    }

    private EligibilityResult allow() {
        return new EligibilityResult("ALLOW", "OK");
    }
}

```
可以看到，**业务规则完全独立、可读、可测试**。
它不关心数据来自哪个数据库，也不关心风控服务是 HTTP 还是 RPC。

应用层的实现：
应用层负责把数据取出来，交给领域服务计算。它不写业务规则，只做流程编排。

```java
@Service
public class PreOrderCheckAppService {

    private final UserProfileRepository userProfileRepository;
    private final OrderQueryRepository orderQueryRepository;
    private final RiskAssessmentPort riskAssessmentPort;
    private final PurchaseEligibilityEvaluator evaluator;

    public PreOrderCheckAppService(UserProfileRepository userProfileRepository,
                                   OrderQueryRepository orderQueryRepository,
                                   RiskAssessmentPort riskAssessmentPort,
                                   PurchaseEligibilityEvaluator evaluator) {
        this.userProfileRepository = userProfileRepository;
        this.orderQueryRepository = orderQueryRepository;
        this.riskAssessmentPort = riskAssessmentPort;
        this.evaluator = evaluator;
    }

    public EligibilityResult check(Long userId) {
        UserProfile profile = userProfileRepository.findByUserId(userId)
                .orElseThrow(() -> new BizException("用户不存在"));

        TodayOrderCount orderCount = orderQueryRepository.todayCountByUserId(userId);
        RiskScore riskScore = riskAssessmentPort.assess(userId);

        return evaluator.evaluate(profile, orderCount, riskScore);
    }
}

```

领域服务里没有 @Service、@Autowired、Mapper、Feign。
它只依赖领域模型和 Java 标准库，因此：
- 可以脱离 Spring 进行单元测试；
- 可以长期稳定，不随框架升级而修改。


**Cola优势体现**：
| 维度 | MVC | Cola |
|------|-----|------|
| 业务逻辑位置 | 散落在Controller和Service | 收拢在领域层 |
| 代码长度 | 单个方法数百行 | 每方法不超过20行 |
| 可测试性 | 难，需 mock 大量服务 | 领域层可直接单元测试 |
| 扩展性 | 修改一处影响全局 | 领域事件解耦，新增逻辑只需添加订阅者 |
| 事务边界 | 粗粒度，整个Controller方法 | 精准控制在领域操作 |

## 七、Cola与微服务的关系

### 7.1 天然契合
- Cola的限界上下文（bounded context）与微服务的**服务边界**高度一致
- 每个微服务可以独立采用Cola架构
- 服务间通过**上下文映射**（Context Mapping）集成，如：
  - 防腐层（ACL）：转换外部服务数据为己方模型
  - 开放主机服务（OHS）：暴露公开接口
  - 发布语言（PL）：定义服务间交互的数据格式

### 7.2 层级对应关系

```
微服务架构                      Cola架构
──────────────────────────────────────────
API Gateway                     用户层（Controller）
Service（服务）                  应用层 + 领域层
Data（持久化）                   基础设施层
```

### 7.3 落地微服务的最佳实践

1. **先DDD拆分，再Cola落地**
   - 通过DDD战略设计识别限界上下文
   - 每个限界上下文对应一个微服务
   - 服务内部采用Cola架构组织代码

2. **服务间解耦**
   - 通过领域事件（Domain Event）实现服务间通信
   - 避免同步调用导致的服务耦合
   - 事件驱动架构天然适配Cola的领域事件机制

3. **独立部署与演进**
   - 每个Cola服务可独立部署
   - 领域层稳定，技术层可灵活替换

### 7.4 架构演进路径

```
单体架构          →   微服务架构（Cola）
──────────────────────────────────────
所有模块在同一个代码库    按限界上下文拆分为独立服务
所有层混在一起            每服务内部采用Cola四层架构
单数据库                  每服务独立数据库
同步调用                  事件驱动/异步通信
```

## 八、Cola架构实践建议

1. **渐进式演进**
   - 不建议从零开始引入完整Cola架构
   - 可在现有项目中逐步拆分限界上下文，逐步沉淀领域层

2. **团队匹配**
   - 需要DDD培训，使团队理解领域建模思想
   - 建立清晰的代码规范，明确每层职责

3. **避免过度设计**
   - 简单业务场景不必强制使用Cola
   - 当业务复杂度增加时再考虑引入

4. **关键原则**
   - 领域层是核心，优先保证领域模型的业务完整性
   - 依赖方向必须遵守：外层依赖内层，内层不依赖外层
   - 仓储接口定义在领域层，实现放在基础设施层
