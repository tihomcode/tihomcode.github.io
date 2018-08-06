---
layout: post
title: SpringBoot + Spring Data JPA金融项目（四）
category: project
tags: [SpringBoot,Spring Data JPA,project]
---

#### 项目已上传github：https://github.com/tihomcode/tihom-finance

---

#### JPA多数据源

#### JPA多数据源运行原理及源码查看

* 主备、读写分离（对账就可能是在备份库和读库执行的，下单操作就是在主库上执行的）

* springboot自动配置过程

* [Spring Data JPA的文档](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.java-config) 查看 Annotation-based Configuration这个块的内容

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/180806%E4%B8%89%E4%B8%AA%E5%85%B3%E9%94%AEbean.jpg)

  发现代码中这三个关键的bean

  * 数据源
  * 实体管理工厂（找到代码的实体类与数据库的表对应起来）
  * 事务管理工厂

* [Spring Boot官方文档](https://docs.spring.io/spring-boot/docs/1.5.15.RELEASE/reference/htmlsingle/#howto-configure-a-datasource) 查看Data Access的内容，不过文档的内容相对比较简单很难看出如何实现

* 查看源码

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/%EF%BC%91%EF%BC%98%EF%BC%90%EF%BC%98%EF%BC%90%EF%BC%96%E5%A4%9A%E6%95%B0%E6%8D%AE%E6%BA%90%E9%85%8D%E7%BD%AE%E6%BA%90%E7%A0%81%EF%BC%90%EF%BC%91.jpg)

  按照这个浏览顺序查看

  ```
  @Configuration
  @ConditionalOnBean({DataSource.class})
  @ConditionalOnClass({JpaRepository.class})
  @ConditionalOnMissingBean({JpaRepositoryFactoryBean.class, JpaRepositoryConfigExtension.class})
  @ConditionalOnProperty(
      prefix = "spring.data.jpa.repositories",
      name = {"enabled"},
      havingValue = "true",
      matchIfMissing = true
  )
  @Import({JpaRepositoriesAutoConfigureRegistrar.class})  
  //这里表示配置要在HibernateJpaAutoConfiguration之后进行配置
  @AutoConfigureAfter({HibernateJpaAutoConfiguration.class})  
  public class JpaRepositoriesAutoConfiguration {
      public JpaRepositoriesAutoConfiguration() {
      }
  }
  ```

  进入JpaRepositoriesAutoConfigureRegistrar上的@EnableJpaRepositories注解

  ```
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Inherited
  @Import({JpaRepositoriesRegistrar.class})
  public @interface EnableJpaRepositories {
      String[] value() default {};
  
      String[] basePackages() default {};
  
      Class<?>[] basePackageClasses() default {};
  
      Filter[] includeFilters() default {};
  
      Filter[] excludeFilters() default {};
  
      String repositoryImplementationPostfix() default "Impl";
  
      String namedQueriesLocation() default "";
  
      Key queryLookupStrategy() default Key.CREATE_IF_NOT_FOUND;
  
      Class<?> repositoryFactoryBeanClass() default JpaRepositoryFactoryBean.class;
  
      Class<?> repositoryBaseClass() default DefaultRepositoryBaseClass.class;
  	//发现这两个关键的bean，然后我们查找这两个bean在哪里被注册
      String entityManagerFactoryRef() default "entityManagerFactory";
  
      String transactionManagerRef() default "transactionManager";
  
      boolean considerNestedRepositories() default false;
  
      boolean enableDefaultTransactions() default true;
  }
  ```

  回头看看HibernateJpaAutoConfiguration，发现代码也没有什么相关，查看父类JpaBaseConfiguration，发现了这里出现了我们要找的那两个关键的bean

  ```
      @Bean
      @ConditionalOnMissingBean({PlatformTransactionManager.class})
      public PlatformTransactionManager transactionManager() {
          JpaTransactionManager transactionManager = new JpaTransactionManager();
          if (this.transactionManagerCustomizers != null) {
              this.transactionManagerCustomizers.customize(transactionManager);
          }
  
          return transactionManager;
      }
  
      @Bean
      @ConditionalOnMissingBean
      public JpaVendorAdapter jpaVendorAdapter() {
          AbstractJpaVendorAdapter adapter = this.createJpaVendorAdapter();
          adapter.setShowSql(this.properties.isShowSql());
          adapter.setDatabase(this.properties.determineDatabase(this.dataSource));
          adapter.setDatabasePlatform(this.properties.getDatabasePlatform());
          adapter.setGenerateDdl(this.properties.isGenerateDdl());
          return adapter;
      }
  
  	//构建entityManagerFactory
      @Bean
      @ConditionalOnMissingBean
      public EntityManagerFactoryBuilder entityManagerFactoryBuilder(JpaVendorAdapter jpaVendorAdapter, ObjectProvider<PersistenceUnitManager> persistenceUnitManager) {
          EntityManagerFactoryBuilder builder = new EntityManagerFactoryBuilder(jpaVendorAdapter, this.properties.getProperties(), (PersistenceUnitManager)persistenceUnitManager.getIfAvailable());
          builder.setCallback(this.getVendorCallback());
          return builder;
      }
  
      @Bean
      @Primary
      @ConditionalOnMissingBean({LocalContainerEntityManagerFactoryBean.class, EntityManagerFactory.class})
      public LocalContainerEntityManagerFactoryBean entityManagerFactory(EntityManagerFactoryBuilder factoryBuilder) {
          Map<String, Object> vendorProperties = this.getVendorProperties();
          this.customizeVendorProperties(vendorProperties);
          return factoryBuilder.dataSource(this.dataSource).packages(this.getPackagesToScan()).properties(vendorProperties).jta(this.isJta()).build();
      }
  ```

  大致流程应该

  > Hibernate帮我们绑定数据源以及实体类、事务管理的Bean注册到容器中，然后自动配置使用这个bean

#### 实际操作配置多数据源

* 新建一个备份数据库seller_backup，然后把seller数据库中的表复制到其中，删除主库上对账的表，删除所有记录

* 数据库同步方面保持数据一致性的方法

  * mysql配置主从复制
  * 阿里开源的框架alibaba/otter

* 修改application.yml的内容，配置两个数据源

  ```
  spring:
    datasource:
      primary:
        url: jdbc:mysql://localhost:3306/seller?useUnicode=true&useSSL=false&characterEncoding=utf-8
        username: root
        password: xxxxxx(你的密码)
      backup:
        url: jdbc:mysql://localhost:3306/seller_backup?useUnicode=true&useSSL=false&characterEncoding=utf-8
        username: root
        password: xxxxxx
  ```

* 定义一个DataAccessConfiguration数据库相关操作配置类

  查看文档中的操作并且复制代码到类中进行修改

  ```
     /**
       * 主数据源
       * @return
       */
      @Bean
      @Primary    //限定注解
      @ConfigurationProperties("spring.datasource.primary")
      public DataSource primaryDataSource() {
          return DataSourceBuilder.create().build();
      }
  
      /**
       * 备份数据源
       * @return
       */
      @Bean
      @ConfigurationProperties("spring.datasource.backup")
      public DataSource backupDataSource() {
          return DataSourceBuilder.create().build();
      }
  
      /**
       * 实体管理
       * @param builder
       * @param dataSource
       * @return
       */
      @Bean
      @Primary
      public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
              EntityManagerFactoryBuilder builder,@Qualifier("primaryDataSource") DataSource dataSource) { //使用限定词注入
          return builder
                  .dataSource(dataSource) //这里类型注入的时候因为两个bean的类型都一致，所以会报错
                  .packages(Order.class)
                  
                  .persistenceUnit("primary")
                  .build();
      }
  
      /**
       * 实体管理
       * @param builder
       * @param dataSource
       * @return
       */
      @Bean
      public LocalContainerEntityManagerFactoryBean backupEntityManagerFactory(
              EntityManagerFactoryBuilder builder,@Qualifier("backupDataSource") DataSource dataSource) {
          return builder
                  .dataSource(dataSource)
                  .packages(Order.class)
                  
                  .persistenceUnit("backup")
                  .build();
      }
  ```

  查看源代码，发现这里还需要一个properties配置

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/180806entityManager%E6%BA%90%E7%A0%81.png)

  实现类方法如下![](http://pbfnr73y8.bkt.clouddn.com/csdn/180806getVendorProperties%E6%96%B9%E6%B3%95.png)

  然后复制到我们的自定义配置类中进行修改

  ```
  /**
   * 数据库相关操作配置
   * @author TiHom
   * create at 2018/8/5 0005.
   */
  
  @Configuration
  public class DataAccessConfiguration {
  
      @Autowired
      private JpaProperties properties;
  
      /**
       * 主数据源
       * @return
       */
      @Bean
      @Primary    //限定注解
      @ConfigurationProperties("spring.datasource.primary")
      public DataSource primaryDataSource() {
          return DataSourceBuilder.create().build();
      }
  
      /**
       * 备份数据源
       * @return
       */
      @Bean
      @ConfigurationProperties("spring.datasource.backup")
      public DataSource backupDataSource() {
          return DataSourceBuilder.create().build();
      }
  
      /**
       * 实体管理
       * @param builder
       * @param dataSource
       * @return
       */
      @Bean
      @Primary
      public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
              EntityManagerFactoryBuilder builder,@Qualifier("primaryDataSource") DataSource dataSource) { //使用限定词注入
          return builder
                  .dataSource(dataSource) //这里类型注入的时候因为两个bean的类型都一致，所以会报错
                  .packages(Order.class)
                  .properties(getVendorProperties(dataSource))
                  .persistenceUnit("primary")
                  .build();
      }
  
      /**
       * 实体管理
       * @param builder
       * @param dataSource
       * @return
       */
      @Bean
      public LocalContainerEntityManagerFactoryBean backupEntityManagerFactory(
              EntityManagerFactoryBuilder builder,@Qualifier("backupDataSource") DataSource dataSource) {
          return builder
                  .dataSource(dataSource)
                  .packages(Order.class)
                  .properties(getVendorProperties(dataSource))
                  .persistenceUnit("backup")
                  .build();
      }
  
  
      protected Map<String, Object> getVendorProperties(DataSource dataSource) {
          Map<String, Object> vendorProperties = new LinkedHashMap<String, Object>();
          vendorProperties.putAll(properties.getHibernateProperties(dataSource));
          return vendorProperties;
      }
  
      @Bean
      @Primary
      public PlatformTransactionManager primaryTransactionManager(@Qualifier("primaryEntityManagerFactory")
                           LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory) {
          JpaTransactionManager transactionManager = new JpaTransactionManager(primaryEntityManagerFactory.getObject());
          return transactionManager;
      }
  
      @Bean
      public PlatformTransactionManager backupTransactionManager(@Qualifier("backupEntityManagerFactory")
                                                                          LocalContainerEntityManagerFactoryBean backupEntityManagerFactory) {
          JpaTransactionManager transactionManager = new JpaTransactionManager(backupEntityManagerFactory.getObject());
          return transactionManager;
      }
  
      // repository 扫描的时候，并不确定哪个先扫描，查看源代码进行判断
      @EnableJpaRepositories(basePackageClasses = OrderRepository.class,
              entityManagerFactoryRef = "primaryEntityManagerFactory",transactionManagerRef = "primaryTransactionManager")
      public class PrimaryConfiguration {
      }
  
      @EnableJpaRepositories(basePackageClasses = VerifyRepository.class,
              entityManagerFactoryRef = "backupEntityManagerFactory",transactionManagerRef = "backupTransactionManager")
      public class backupConfiguration {
      }
  
  }
  ```

  下面解析这些类！

#### 解析JPA的三个关键bean

* EntityManagerFactoryBean
* DataSource
* TransactionManager

```
@EnableJpaRepositories(basePackageClasses = OrderRepository.class,
            entityManagerFactoryRef = "primaryEntityManagerFactory",transactionManagerRef = "primaryTransactionManager")
public class PrimaryConfiguration {
}
            
@EnableJpaRepositories(basePackageClasses = VerifyRepository.class,
            entityManagerFactoryRef = "backUpEntityManagerFactory",transactionManagerRef = "backUpTransactionManager")     
public class BackUpConfiguration {
}
```

这里主备需要实现的repository接口不同，但是basePackageClasses的扫描机制是扫描对应类所在的包然后找到那个包下面的所有类，这里OrderRepository和VerifyRepository是在同一个包下的，这两个配置扫描的包是同一个，所以会把配置配置两份，这就出现问题了，产生了冲突只会生效一个。还有我们不知道PrimaryConfiguration先配置还是BackUpConfiguration先配置，所以需要查看源码来判断。

结果发现，BackUpConfiguration先注入且先注入OrderRepository再注入VerifyRepository，然后PrimaryConfiguration再注入且先注入VerifyRepository后注入OrderRepository

官方也没有提供合适的解决方法，所以最简单的做法就是将主库与备份库操作的repository分别放在repository和repositorybackup中，这样就不会混淆了。

#### 一些问题的解释

* @Primary：如果我们一个容器当中添加了两个相同类型但是名称不相同的bean，我们如果想通过名称注入这些bean，那么这个@Primary无需使用。但是如果我们想直接通过类型，不用名称限定直接注入的话，spring容器就会报错，我们需要一个类型，但是这个类型对应两个bean，spring不知道哪个才是我们需要的，这个时候使用@Primary在一个bean上面让它成为主bean，spring就会自动识别
* MySql主从复制、alibaba/otter
* 不同数据源分包

#### 查看源码的步骤

* 先看官方文档
* 尝试按自己的思维去实现功能
* 暴力搜索Ctrl+Shift+F（注意会跟搜狗输入法中的快捷键冲突，两个软件二选一进行修改）

---

### JPA读写分离

#### 读写分离的作用（what？why？when？how？）

参考于https://blog.csdn.net/u013421629/article/details/78793966

#### 不同数据源相同repositories

* 添加额外接口，继承

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/180806%E8%AF%BB%E5%86%99%E5%88%86%E7%A6%BB.png)

  使用OrderRepository读出来的数据和backupOrderRepository读出来的数据是不一样的

  ```
  public interface backupOrderRepoistory extends OrderRepository {
  }
  ```

* 修改源码

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/%EF%BC%91%EF%BC%98%EF%BC%90%EF%BC%98%EF%BC%90%EF%BC%96%E5%A4%9A%E6%95%B0%E6%8D%AE%E6%BA%90%E9%85%8D%E7%BD%AE%E6%BA%90%E7%A0%81.png)

  找到注册过程的代码，找到这个类发现这里是生成bean名称的地方

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/180806bean%E5%90%8D%E7%A7%B0.png)

然后复制这个类的全部代码

![](http://pbfnr73y8.bkt.clouddn.com/csdn/180806%E6%BA%90%E7%A0%81%E6%8B%B7%E8%B4%9D.png)

这个 RepositoryConfigurationDelegate与源码完全相同，然后我们自己进行一些修改

为什么这样可行呢？因为当前应用下有对应类时不会使用依赖包内的，优先使用当前应用下包内的。这是修改源码的核心！！！

* 声明一个注解@RepositoryBeanNamePrefix

```
/**
 * repository bean 名称的前缀
 * @author TiHom
 * create at 2018/8/6 0006.
 */

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface RepositoryBeanNamePrefix {
    String value();
}
```

* 修改RepositoryConfigurationDelegate类

  ```
  while(var5.hasNext()) {
              RepositoryConfiguration<? extends RepositoryConfigurationSource> configuration = (RepositoryConfiguration)var5.next();
              BeanDefinitionBuilder definitionBuilder = builder.build(configuration);
              extension.postProcess(definitionBuilder, this.configurationSource);
              if (this.isXml) {
                  extension.postProcess(definitionBuilder, (XmlRepositoryConfigurationSource)this.configurationSource);
              } else {
                  extension.postProcess(definitionBuilder, (AnnotationRepositoryConfigurationSource)this.configurationSource);
              }
  
              AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
              String beanName = this.beanNameGenerator.generateBeanName(beanDefinition, registry);
  			---- 开始修改 ----
              AnnotationMetadata metadata = (AnnotationMetadata) configurationSource.getSource();
              //判断配置类是否使用primary进行了标注，如果有，就设为primary
              if(metadata.hasAnnotation(Primary.class.getName())){
                  beanDefinition.setPrimary(true);
              }else if(metadata.hasAnnotation(RepositoryBeanNamePrefix.class.getName())) {
                  //再判断是否使用了RepositoryBeanNamePrefix进行了标注，如果有，添加名称前缀
                  Map<String,Object> prefixData = metadata.getAnnotationAttributes(RepositoryBeanNamePrefix.class.getName());
                  String prefix = (String) prefixData.get("value");
                  beanName = prefix + beanName;
              }
  			---- 结束修改 ----
              if (LOGGER.isDebugEnabled()) {
                  LOGGER.debug("Spring Data {} - Registering repository: {} - Interface: {} - Factory: {}", new Object[]{extension.getModuleName(), beanName, configuration.getRepositoryInterface(), extension.getRepositoryFactoryClassName()});
              }
  
              beanDefinition.setAttribute("factoryBeanObjectType", configuration.getRepositoryInterface());
              registry.registerBeanDefinition(beanName, beanDefinition);
              definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
          }
  
  ```

  

* 修改DataAccessConfiguration类

```
@EnableJpaRepositories(basePackageClasses = OrderRepository.class,
            entityManagerFactoryRef = "primaryEntityManagerFactory",transactionManagerRef = "primaryTransactionManager")
    @Primary
    public class PrimaryConfiguration {
    }

    @EnableJpaRepositories(basePackageClasses = OrderRepository.class,
            entityManagerFactoryRef = "primaryEntityManagerFactory",transactionManagerRef = "backupEntityManagerFactory")
    @RepositoryBeanNamePrefix("read")
    public class ReadConfiguration {
    }

    @EnableJpaRepositories(basePackageClasses = VerifyRepository.class,
            entityManagerFactoryRef = "backupEntityManagerFactory",transactionManagerRef = "backupTransactionManager")
    public class BackupConfiguration {
    }
```

#### 总结

* 接口继承，在不同包下面，并且维护只需要维护被继承的类
* 同一个Repoistory，不需要创建多一个继承，直接拦截注册过程，创建两个同类型但是不同名称的bean到spring容器中，主库的加上@Primary注解，读库的使用自定义的@RepositoryBeanNamePrefix注解