# Spring问题解决

---
tags: [问题解决, Spring, SpringBoot, SpringCloud, 调试, 性能优化]
created: 2026-02-21
updated: 2026-02-21
status: 持续更新
importance: ⭐⭐⭐⭐
---

## 🚨 高频问题速查

### 问题1：循环依赖问题 `#循环依赖`
**现象**：应用启动时报错 "The dependencies of some of the beans in the application context form a cycle"
**原因**：两个或多个Bean相互依赖，形成循环引用
**解决**：
1. 使用@Lazy注解延迟加载其中一个Bean
2. 使用@PostConstruct在初始化后设置依赖
3. 重构代码，消除循环依赖关系
4. 使用ApplicationContextAware获取Bean

**相关原理**：[[Spring核心原理#循环依赖解决机制]]

---

### 问题2：Bean创建失败 `#Bean创建`
**现象**：NoSuchBeanDefinitionException或BeanCreationException
**原因**：Bean定义不正确、依赖注入失败、构造函数参数问题
**解决**：
1. 检查@Component、@Service等注解是否正确
2. 确认包扫描路径包含目标类
3. 检查构造函数参数和依赖注入
4. 使用@Qualifier指定具体的Bean

**相关原理**：[[Spring核心原理#Bean生命周期]]

---

### 问题3：事务不生效 `#事务问题`
**现象**：数据库操作没有回滚，事务注解不起作用
**原因**：方法不是public、内部调用、异常被捕获、事务传播设置错误
**解决**：
1. 确保方法是public的
2. 避免同类内部方法调用，使用AopContext.currentProxy()
3. 检查异常类型，默认只回滚RuntimeException
4. 正确设置rollbackFor属性

**相关原理**：[[Spring核心原理#Spring事务管理]]

---

### 问题4：自动装配失败 `#自动装配`
**现象**：SpringBoot启动时某些配置类没有生效
**原因**：条件注解不满足、配置文件错误、依赖缺失
**解决**：
1. 检查@ConditionalOnClass等条件注解
2. 确认spring.factories文件配置正确
3. 检查依赖是否正确引入
4. 使用@EnableAutoConfiguration(exclude={})排除冲突

**相关原理**：[[SpringBoot实战#自动装配原理]]

---

### 问题5：配置文件不生效 `#配置问题`
**现象**：application.yml或application.properties中的配置没有生效
**原因**：配置文件位置错误、属性名错误、环境配置冲突
**解决**：
1. 确认配置文件在classpath根目录或config目录下
2. 检查属性名拼写和层级结构
3. 使用@ConfigurationProperties绑定配置
4. 检查profile环境配置

**相关原理**：[[SpringBoot实战#配置管理]]

---

### 问题6：内存泄漏问题 `#内存问题`
**现象**：应用运行一段时间后内存持续增长，最终OOM
**原因**：Bean作用域错误、事件监听器未清理、缓存未清理
**解决**：
1. 检查Bean的作用域设置
2. 及时移除事件监听器
3. 定期清理缓存
4. 使用弱引用或软引用

**相关原理**：[[Spring核心原理#Bean生命周期]]

---

### 问题7：启动速度慢 `#启动优化`
**现象**：SpringBoot应用启动时间过长
**原因**：自动配置过多、Bean创建耗时、类路径扫描范围过大
**解决**：
1. 排除不需要的自动配置类
2. 使用@Lazy延迟初始化
3. 缩小包扫描范围
4. 优化数据库连接池配置

**相关原理**：[[SpringBoot实战#性能优化]]

---

### 问题8：微服务调用失败 `#服务调用`
**现象**：Feign调用报错，服务间通信失败
**原因**：服务未注册、负载均衡配置错误、网络问题、超时设置
**解决**：
1. 检查服务注册中心状态
2. 确认服务名称和端口正确
3. 配置合适的超时时间
4. 添加重试机制和熔断降级

**相关原理**：[[SpringCloud微服务#跨进程通讯 RPC]]

---

### 问题9：网关路由不生效 `#网关问题`
**现象**：Gateway路由配置不生效，请求无法转发
**原因**：路由配置错误、断言条件不匹配、过滤器异常
**解决**：
1. 检查路由配置的path和uri
2. 验证断言条件是否正确
3. 检查过滤器逻辑
4. 查看Gateway日志排查问题

**相关原理**：[[SpringCloud微服务#API网关 - Gateway]]

---

### 问题10：分布式事务回滚失败 `#分布式事务`
**现象**：Seata分布式事务没有正确回滚
**原因**：TC连接失败、RM注册失败、数据源配置错误
**解决**：
1. 检查Seata Server连接状态
2. 确认数据源代理配置正确
3. 检查undo_log表是否存在
4. 验证全局事务注解配置

**相关原理**：[[SpringCloud微服务#分布式事务 - Seata]]

## 🔧 调试技巧

### 常用调试方法

#### 1. 日志调试
```yaml
logging:
  level:
    org.springframework: DEBUG
    com.yourpackage: DEBUG
    org.springframework.web: DEBUG
    org.springframework.security: DEBUG
```

#### 2. Actuator监控
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always
```

#### 3. 条件评估报告
```bash
# 启动时添加参数查看自动配置报告
java -jar app.jar --debug
```

#### 4. Bean信息查看
```java
@RestController
public class DebugController {

    @Autowired
    private ApplicationContext applicationContext;

    @GetMapping("/beans")
    public String[] getBeans() {
        return applicationContext.getBeanDefinitionNames();
    }

    @GetMapping("/bean/{name}")
    public Object getBean(@PathVariable String name) {
        return applicationContext.getBean(name);
    }
}
```

### 性能分析工具

#### 1. JProfiler
- 内存分析
- CPU性能分析
- 线程分析

#### 2. Arthas
```bash
# 下载并启动Arthas
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar

# 常用命令
dashboard  # 系统概览
thread     # 线程分析
jvm        # JVM信息
sc         # 查看类信息
sm         # 查看方法信息
```

#### 3. Spring Boot Admin
```yaml
# 客户端配置
spring:
  boot:
    admin:
      client:
        url: http://localhost:8080
        instance:
          prefer-ip: true
```

### 日志分析技巧

#### 1. 结构化日志
```yaml
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%logger{50}] - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%logger{50}] - %msg%n"
```

#### 2. 链路追踪
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

#### 3. 自定义日志
```java
@Slf4j
@Component
public class RequestLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        long startTime = System.currentTimeMillis();

        try {
            chain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("Request: {} {} - Duration: {}ms",
                    httpRequest.getMethod(),
                    httpRequest.getRequestURI(),
                    duration);
        }
    }
}
```

## 🚀 性能优化技巧

### 1. 启动优化
```java
@SpringBootApplication
@EnableAutoConfiguration(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class Application {
    public static void main(String[] args) {
        System.setProperty("spring.devtools.restart.enabled", "false");
        SpringApplication app = new SpringApplication(Application.class);
        app.setLazyInitialization(true);
        app.run(args);
    }
}
```

### 2. 内存优化
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true

server:
  tomcat:
    max-threads: 200
    min-spare-threads: 10
```

### 3. 数据库优化
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### 4. 缓存优化
```java
@EnableCaching
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        RedisCacheManager.Builder builder = RedisCacheManager
                .RedisCacheManagerBuilder
                .fromConnectionFactory(redisConnectionFactory())
                .cacheDefaults(cacheConfiguration(Duration.ofMinutes(10)));
        return builder.build();
    }
}
```

## 🔍 常用排查命令

### 1. JVM相关
```bash
# 查看JVM参数
jinfo -flags <pid>

# 查看堆内存使用情况
jmap -heap <pid>

# 生成堆转储文件
jmap -dump:format=b,file=heap.hprof <pid>

# 查看GC情况
jstat -gc <pid> 1000
```

### 2. 线程相关
```bash
# 查看线程栈
jstack <pid>

# 查看线程CPU使用情况
top -H -p <pid>
```

### 3. 网络相关
```bash
# 查看端口占用
netstat -tlnp | grep :8080

# 查看网络连接
ss -tulpn | grep :8080
```

## 🔗 相关文档
- **技术原理**：[[Spring核心原理]]
- **实战应用**：[[SpringBoot实战]]
- **微服务架构**：[[SpringCloud微服务]]
- **数据库问题**：[[../../04-mysql|MySQL问题解决]]

## 🏷️ 标签
#问题解决 #Spring #SpringBoot #SpringCloud #调试 #性能优化 #故障排查 #监控