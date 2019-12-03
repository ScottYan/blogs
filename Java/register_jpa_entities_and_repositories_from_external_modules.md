# 从外部模块注入JPA Entity和Repositories

> 场景: <br />
> 一个项目使用了多个模块, 其中有一个common模块中定义了若干个Jpa的entity和repository, 另外两个业务模块(Module1和Module2)都需要使用这些entity和repository. 

主要有两种做法, 一种是在业务模块的application上面手动添加entityScan和enableJpaRepositories注解: 
```java
@SpringBootApplication
@EnableJpaRepositories(basePackages ={ "com.example.common.dao", "com.example.Module1.dao"})
@EntityScan(basePackages ={ "com.example.common.model", "com.example.Module1.model"})
public class Module1Application {
}
```
> 注意 <br />
> 当使用这种模式时, basePackages中必须包括当前项目的包名. 因为这个设置会覆盖默认设置, 如果只写了common模块的包名, 则Jpa就会只扫描common模块. 

这样手动去写, 显然让人觉得很不适. 所以应该改用第二种方法: 
1. 首先在common中启用自动配置: 
    在Common模块的resource/META-INF下面,新建一个spring.factories文件, 内容如下: 
    ```properties
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    com.example.common.config.CommonAutoConfig
    ```
2. 在CommonAutoConfig类上加上如下配置: 
    ```java
    @AutoConfigureAfter(JpaRepositoriesAutoConfiguration.class)
    @EnableJpaRepositories(basePackages = {"com.example.common.dao.*"})
    @Import({CommonEntityRegistrar.class})
    public class CommonEntityAutoConfig {
    }
    ```
3. 创建一个名为CommonEntityRegistrar的类, 代码如下: 
    ```java
    @Order(1)
    public class CommonEntityRegistrar implements ImportBeanDefinitionRegistrar {

        private static final String BEAN = EntityScanPackages.class.getName();
        private static final String COMMON_ENTITY_PACKAGE_NAME = MyCommonEntity.class.getPackageName();

        public CommonEntityRegistrar() {
        }

        @Override
        public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
            BeanFactory beanFactory = (BeanFactory) registry;
            //获取主项目所在的包路径
            List<String> basePackage = AutoConfigurationPackages.get(beanFactory);
            Set<String> securityDomainPackage = new HashSet<>(basePackage);
            //获取当前项目的主路径.
            securityDomainPackage.add(COMMON_ENTITY_PACKAGE_NAME);
            //注入这两个项目的Entity实体.
            register(registry, Collections.unmodifiableSet(securityDomainPackage));
        }

        private static void register(BeanDefinitionRegistry registry, Collection<String> packageNames) {
            Assert.notNull(registry, "Registry must not be null");
            Assert.notNull(packageNames, "PackageNames must not be null");
            if (registry.containsBeanDefinition(BEAN)) {
                BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
                ConstructorArgumentValues constructorArguments = beanDefinition.getConstructorArgumentValues();
                constructorArguments.addIndexedArgumentValue(0,
                        addPackageNames(constructorArguments, packageNames));
            }
            else {
                GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
                beanDefinition.setBeanClass(EntityScanPackages.class);
                beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0,
                        packageNames.toArray(new String[0]));
                beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
                registry.registerBeanDefinition(BEAN, beanDefinition);
            }
        }

        private static String[] addPackageNames(
                ConstructorArgumentValues constructorArguments,
                Collection<String> packageNames) {
            String[] existing = (String[]) Objects.requireNonNull(constructorArguments
                    .getIndexedArgumentValue(0, String[].class)).getValue();
            var merged = new LinkedHashSet<String>();
            merged.addAll(Arrays.asList(existing != null ? existing : new String[0]));
            merged.addAll(packageNames);
            return merged.toArray(new String[0]);
        }
    }
    ```

**Done**
