---
layout: post
title: SpringBoot + Spring Data JPA金融项目（一）
category: project
tags: [SpringBoot,Spring Data JPA,project]
---

### 想起来也是有一段时间没有更新博客了，惭愧啊！！！ 

#### 这个月跟高中同学见面啥的花了点时间，不过目前跟进的两个项目都差不多了，今天先贴出一个项目！ 

#### 项目已上传github：https://github.com/tihomcode/tihom-finance

---

###  模块化开发

* 业务层次    dao，service
* 功能划分    管理端和销售端
* 重复使用    单独划分

### 使用技术：

* SpringBoot
* Spring Data JPA
* Swagger2
* MySql
* Maven
* Junit
* JsonRpc
* 缓存
* RSA签名、TYK、HTTPS实现节流限速及安全

使用的开发工具是IDEA 2018.2 

数据库管理工具是DataGrip

测试接口工具是Postman

### 数据库表设计

* 产品表（t_product）
  * 编号   id varchar(50) 主键
  * 名称   name  varchar(50)
  * 收益率   reward_rate decimal(5,3)
  * 锁定期（七天、一个月...）  lock_term smallint
  * 状态（审核中、销售、暂停销售、已结束）  status
  * 起投金额   threshold_amount decimal(15,3)
  * 投资步长   step_amount decimal(15,3)
  * 备注   memo varchar(200)
  * 创建时间   create_at datetime
  * 创建者   create_user varchar(20)
  * 更新时间   update_at datetime
  * 更新者   update_user varchar(20)
* 订单表（t_order）
  * 订单编号  order_id  varchar(50) 主键
  * 渠道编号（公司编号）  chan_id  varchar(50) 
  * 产品编号  product_id  varchar(50)
  * 用户编号  chan_user_id  varchar(50)
  * 外部订单编号（公司的订单编号）  outer_order_id  varchar(50)
  * 类型（申购、赎回）  order_type  varchar(50)
  * 状态（处理中、处理成功、处理失败） order_status  varchar(50)
  * 金额  amount  decimal(15,3)
  * 备注  memo  varchar(200)
  * 创建时间  create_at  datetime
  * 更新时间  update_at  datetime

### 实现增删改查

| 具体操作     | Rest风格 | url            | JPA接口                  |
| ------------ | -------- | -------------- | ------------------------ |
| 添加产品     | POST     | /products      | JpaRepository            |
| 查询单个产品 | GET      | /products/{id} | JpaRepository            |
| 条件查询产品 | GET      | /products      | JpaSpecificationExecutor |

---

### 经验

* 写实体类的时候，记住要重写toString方法，因为要经常打印在日志中，重写后可以比较直观的反映内容
* 数据对象的属性不能直接设定默认值，不能使用原始类型
* 在业务处理的时候记得创建类后就添加日志对象，在service日志级别为debug，在controller中使用info级别

对于实体类默认值的设定，不提倡在实体类中直接赋值，且推荐数据类型用包装后的对象而不使用原始类型（如Integer和int）

---

### 常见错误

* Error creating bean with name 'productController': Unsatisfied dependency expressed through field 'productService'; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'productService': Unsatisfied dependency expressed through field 'repository'; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'productRepository': Invocation of init method failed; nested exception is java.lang.IllegalArgumentException: Not a managed type: class com.tihom.entity.Product

> 对于这种出现bean创建错误的，证明spring没有扫描到它的注解，所以在Springboot启动类上添加
>
> ```@EntityScan(basePackages = {"com.tihom.entity"})```

* jdbc错误，证明没有加mysq连接包，如添加只需添加在manager模块中即可

---

### 测试接口

使用postman测试方法是否有误

![](http://pbfnr73y8.bkt.clouddn.com/1533190241%281%29.jpg)

![](http://pbfnr73y8.bkt.clouddn.com/1533190257.jpg)

---

### 错误处理

* 用户友好的错误说明（返回前台时展示的内容）
* 统一处理，简化业务代码
* 异常标准化

##### 查看[SpringBoot官方文档](https://docs.spring.io/spring-boot/docs/1.5.15.RELEASE/reference/htmlsingle/#boot-features-error-handling)寻找答案，找到27.1.9处

* 第一种，直接自定义错误静态页面（/error）

* 第二种，自定义MyErrorController，继承BasicErrorController，BasicErrorController在ErrorMvcAutoConfiguration中注册到spring中，自定义要返回前端的内容中要包含的参数，一般只需要返回错误码和错误信息即可，无需返回时间戳等等参数。

  ```
  /**
   * 自定义错误异常Controller
   * @author TiHom
   * create at 2018/8/2 0002.
   */
  public class MyErrorController extends BasicErrorController {
  
      public MyErrorController(ErrorAttributes errorAttributes, ErrorProperties errorProperties, List<ErrorViewResolver> errorViewResolvers) {
          super(errorAttributes, errorProperties, errorViewResolvers);
      }
  
      @Override
      protected Map<String, Object> getErrorAttributes(HttpServletRequest request, boolean includeStackTrace) {
          Map<String,Object> attrs = super.getErrorAttributes(request,includeStackTrace);
          /** 这里面只需要message和返回前端的判断码需要,其他删除就好
           * {
           *     "timestamp": "2018-08-02 13:02:30",
           *     "status": 500,
           *     "error": "Internal Server Error",
           *     "exception": "java.lang.IllegalArgumentException",
           *     "message": "编号不可为空",
           *     "path": "/manager/products"
           *     + code
           *     + canRetry
           * }
           */
          attrs.remove("timestamp");
          attrs.remove("error");
          attrs.remove("exception");
          attrs.remove("path");
          attrs.remove("status");
          //通过返回的message拿到错误对象
          String errorCode = (String) attrs.get("message");
          ErrorEnum errorEnum = ErrorEnum.getByCode(errorCode);
          //放入返回参数中
          attrs.put("message",errorEnum.getMessage());
          attrs.put("code",errorEnum.getCode());
          attrs.put("canRetry",errorEnum.isCanRetry());
          return attrs;
      }
  }
  ```

  

  自定义错误枚举类，在里面写上错误的种类定义三个属性（错误码、错误信息、是否可重试。。。可以自己扩展）

  ```
  /**
   * 错误种类
   * @author TiHom
   * create at 2018/8/2 0002.
   */
  public enum  ErrorEnum {
  
      ID_NOT_NULL("F001","编号不可为空",false),
      //...
      UNKOWN("999","未知异常",false);
  
      private String code;
      private String message;
      private boolean canRetry; //是否可重试
  
      ErrorEnum(String code, String message, boolean canRetry) {
          this.code = code;
          this.message = message;
          this.canRetry = canRetry;
      }
  
      public static ErrorEnum getByCode(String code){
          for (ErrorEnum errorEnum : ErrorEnum.values()) {
              if(errorEnum.code.equals(code)){
                  return errorEnum;
              }
          }
          return UNKOWN;
      }
  
      public boolean isCanRetry() {
          return canRetry;
      }
  
      public void setCanRetry(boolean canRetry) {
          this.canRetry = canRetry;
      }
  
      public String getCode() {
  
          return code;
      }
  
      public void setCode(String code) {
          this.code = code;
      }
  
      public String getMessage() {
          return message;
      }
  
      public void setMessage(String message) {
          this.message = message;
      }
  }
  ```

  

  错误处理相关配置类，用来模仿ErrorMvcAutoConfiguration的处理来注册MyErrorController到Spring中

  ```
  /**
   * 错误处理相关配置
   * @author TiHom
   * create at 2018/8/2 0002.
   */
  
  @Configuration
  public class ErrorConfiguration {
  
      //ErrorMvcAutoConfiguration中的处理方式
      @Bean
      public MyErrorController basicErrorController(ErrorAttributes errorAttributes,ServerProperties serverProperties,
                  ObjectProvider<List<ErrorViewResolver>> errorViewResolversProvider) {
          return new MyErrorController(errorAttributes, serverProperties.getError(),
                  errorViewResolversProvider.getIfAvailable());
      }
  }
  
  ```

  

  然后进行之前错误信息返回代码的重构，在ProductService中修改

  修改之前：

  ```
  Assert.notNull(product.getId(),"编号不可为空");
  ```

  修改之后

  ```
  Assert.notNull(product.getId(),ErrorEnum.ID_NOT_NULL.getCode());
  ```

* 第三种，ControllerAdvice，这是controller的增强，同样的就可用于错误处理，在controller外面包一层ControllerAdvice，出错误了就到ControllerAdvice中，如果ControllerAdvice再出现异常，就到MyErrorController中处理。流程如下：

![](http://pbfnr73y8.bkt.clouddn.com/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6.png)

##### 	注意：两种方式同时存在或只存在一种都是可以的



#### 对于返回前段的timestamp参数时间显示格式问题：（默认显示为long型）

这主要是返回的json格式化的问题，查看[官方文档](https://docs.spring.io/spring-boot/docs/1.5.15.RELEASE/reference/htmlsingle/#boot-features-error-handling)中Part X. Appendices，找到Jackson的配置。

在application.yml中spring下添加如下内容

```
jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
```

---

### 自动化测试

使用Junit

* 测试覆盖率（正确和异常情况都要测试），边界条件
* 以功能测试来做测试
* 执行顺序问题，@FixMethodOrder
* 条件查询

---

### 接口文档

* 前后端分离
* 第三方合作

#### 优化:

* 选择性的显示接口
* 详细的注释说明
* 中文显示

#### 常用注解:

* @ApiModelProperty
* @EnableSwagger2
* @ApiOperation

查看路径发现swagger-ui包下的swagger-ui.html在META-INF/resources下，在swagger模块中创建相同路径的文件

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Swagger UI</title>
    <link rel="icon" type="image/png" href="webjars/springfox-swagger-ui/images/favicon-32x32.png" sizes="32x32"/>
    <link rel="icon" type="image/png" href="webjars/springfox-swagger-ui/images/favicon-16x16.png" sizes="16x16"/>
    <link href='webjars/springfox-swagger-ui/css/typography.css' media='screen' rel='stylesheet' type='text/css'/>
    <link href='webjars/springfox-swagger-ui/css/reset.css' media='screen' rel='stylesheet' type='text/css'/>
    <link href='webjars/springfox-swagger-ui/css/screen.css' media='screen' rel='stylesheet' type='text/css'/>
    <link href='webjars/springfox-swagger-ui/css/reset.css' media='print' rel='stylesheet' type='text/css'/>
    <link href='webjars/springfox-swagger-ui/css/print.css' media='print' rel='stylesheet' type='text/css'/>

    <script src='webjars/springfox-swagger-ui/lib/object-assign-pollyfill.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/jquery-1.8.0.min.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/jquery.slideto.min.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/jquery.wiggle.min.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/jquery.ba-bbq.min.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/handlebars-4.0.5.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/lodash.min.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/backbone-min.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/swagger-ui.min.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/highlight.9.1.0.pack.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/highlight.9.1.0.pack_extended.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/jsoneditor.min.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/marked.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lib/swagger-oauth.js' type='text/javascript'></script>

    <!-- 引入国际化的两个js文件 -->
    <script src='webjars/springfox-swagger-ui/lang/translator.js' type='text/javascript'></script>
    <script src='webjars/springfox-swagger-ui/lang/zh-cn.js' type='text/javascript'></script>

    <script src='webjars/springfox-swagger-ui/springfox.js' type='text/javascript'></script>
</head>

<body class="swagger-section">
<div id='header'>
    <div class="swagger-ui-wrap">
        <a id="logo" href="http://swagger.io"><img class="logo__img" alt="swagger" height="30" width="30" src="webjars/springfox-swagger-ui/images/logo_small.png" /><span class="logo__title">swagger</span></a>
        <form id='api_selector'>
            <div class='input'>
                <select id="select_baseUrl" name="select_baseUrl"/>
            </div>
            <div class='input'><input placeholder="http://example.com/api" id="input_baseUrl" name="baseUrl" type="text"/></div>
            <div id='auth_container'></div>
            <div class='input'><a id="explore" class="header__btn" href="#" data-sw-translate>Explore</a></div>
        </form>
    </div>
</div>

<div id="message-bar" class="swagger-ui-wrap" data-sw-translate>&nbsp;</div>
<div id="swagger-ui-container" class="swagger-ui-wrap"></div>
</body>
</html>

```

引入两个js文件实现国际化，使ui界面显示为中文

#### Swagger模块

* 使用配置文件配置
* 组合注解
* @Enable*原理

#### 如何用manager模块使用swagger模块的swagger配置 呢？

* 第一，在Springboot启动类上加上

  ```
  @Import(SwaggerConfiguration.class)
  ```

* 第二，自定义注解EnableMySwagger

  ```
  @Retention(RetentionPolicy.RUNTIME)
  @Target({ElementType.TYPE})
  @Documented 
  @Import(SwaggerConfiguration.class)  //导入自定义的swagger配置类
  @EnableSwagger2  //组合注解
  public @interface EnableMySwagger {
  }
  ```

  组合注解就是在自定义的注解上注入多个注解组合成一个多功能的新注解

* 第三，连注解都不需要写，直接在swagger模块下的resources路径下的META-INF包下定义一个spring.factories，加上

  ```
  org.springframework.boot.autoconfigure.EnableAutoConfiguration =com.tihom.swagger.SwaggerConfiguration
  ```

  SpringBoot在启动的时候会自动扫描META-INF包下的spring.factories来自动配置

  开启在SwaggerConfiguration上的@EnableSwagger2注解，在manager模块中无需任何配置，只需依赖swagger模块即可



#### 自定义配置的使用

为了使swagger-ui界面上的显示内容能够在配置类中直接修改而无需修改源代码 

在SwaggerInfo类上加上```@ConfigurationProperties(prefix = "swagger")```

可能会出现如下错误：```Spring Boot Annotion processor not found in classpath```

解决方案：直接忽视提示，在application.yml中添加上

```
swagger:
  groupName: manager
  basePackage: com.tihom.manager.controller
```

发现ui界面显示也有改变，证明修改成功



#### 将swagger部署在服务器上，这里部署在工程中

github上选择2.x的版本https://github.com/swagger-api/swagger-ui/archive/2.x.zip或者直接按这个链接下载

解压后将dist文件夹下的所有文件拷贝到manager模块中的resources/static/下















