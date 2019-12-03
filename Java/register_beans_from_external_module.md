# 注入外部模块的bean

> 场景: 
> 项目使用多模块设计, 其中一个模块是通用模块(假如叫common), 里面定义了一些component, 在另外的两个业务模块(A和B)中, 期望用@Autowired 把定义在common中的component注进来. 

默认情况下, 虽然在A和B中添加了对Common模块的依赖, 也能在代码中访问到common的类, 但是自动注入是无法生效的. 因为spring不会去扫描common下面的包. 这时需要用到Spring的自动配置功能. 

做法: 
> 在Common模块的resource/META-INF下面,新建一个spring.factories文件, 内容如下: 
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.common.config.CommonAutoConfig
```

> 然后在Common模块下新建一个config包, 新建一个CommonAutoConfig类: 
```java
@ComponentScan(basePackages = {"com.example.common.*"})
public class CommonAutoConfig {
}
```

**Done!**