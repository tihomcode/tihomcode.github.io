---
layout: post
title: SpringBoot + Spring Data JPA金融项目（三）
category: project
tags: [SpringBoot,Spring Data JPA,project]
---

#### 项目已上传github：https://github.com/tihomcode/tihom-finance

---

### RSA签名加密

#### 原理介绍

使用私钥将明文进行签名生成全密文串与明文一起传输，对方接受数据偶使用公钥对明文和密文进行验签。如果验签通过就说明

* 数据没有被修改过
* 这些数据一定是持有私钥的人发送的，因为私钥只有自己持有，这就起到了防抵赖的作用

![](http://pbfnr73y8.bkt.clouddn.com/csdn/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6.png)

#### 使用

* 工具类
* 生成密钥对（在线网站http://web.chacuo.net/netrsakeypair）

#### 下单功能实现

* 定义OrderRepository

  ```
  public interface OrderRepository extends JpaRepository<Order,String>,JpaSpecificationExecutor<Order> {
  }
  ```

* 定义OrderService订单服务

  ```
  /**
   * 订单服务
   * @author TiHom
   * create at 2018/8/4 0004.
   */
  
  @Service
  public class OrderService {
  
      private static Logger LOG = LoggerFactory.getLogger(OrderService.class);
  
      @Autowired
      private OrderRepository orderRepository;
      @Autowired
      private ProductRpcService productRpcService;
  
      /**
       * 申购订单
       * @param order
       * @return
       */
      public Order apply(Order order){
          //数据校验
          checkOrder(order);
          //完善订单数据
          completeOrder(order);
          //saveAndFlush会立即刷新到数据库中(高并发使用好)
          order = orderRepository.saveAndFlush(order);
          return order;
      }
  
      /**
       * 完善订单数据
       * @param order
       */
      private void completeOrder(Order order) {
          order.setOrderId(UUID.randomUUID().toString().replaceAll("-",""));
          order.setOrderType(OrderType.APPLY.name());
          order.setOrderStatus(OrderStatus.SUCCESS.name());
          order.setUpdateAt(new Date());
      }
  
      /**
       * 校验数据
       * @param order
       */
      private void checkOrder(Order order) {
          //必填字段
          Assert.notNull(order.getOuterOrderId(),"需要外部订单编号");
          Assert.notNull(order.getChanId(),"需要渠道编号");
          Assert.notNull(order.getChanUserId(),"需要用户编号");
          Assert.notNull(order.getProductId(),"需要产品编号");
          Assert.notNull(order.getAmount(),"需要购买金额");
          Assert.notNull(order.getCreateAt(),"需要订单时间");
          //产品是否存在及金额是否符合要求
          Product product = productRpcService.findOne(order.getProductId());
          Assert.notNull(product,"产品不存在");
          //金额要满足如果有起投金额时，要大于等于起投金额，如果有投资步长时，超过起投金额的部分要是投资步长的整数倍
  //        Assert.isTrue(order.getAmount().compareTo(product.getThresholdAmount())>0,"购买金额要大于起投金额");
  //        BigDecimal[] bigDecimals = (order.getAmount().subtract(product.getThresholdAmount())).divideAndRemainder(product.getStepAmount());
  //        Assert.isTrue(bigDecimals[1].equals(BigDecimal.ZERO),"超过起投金额的部分要是投资步长的整数倍");
      }
  ```

* 定义OrderController

  ```
  /**
   * 订单相关
   * @author TiHom
   * create at 2018/8/4 0004.
   */
  
  @RestController
  @RequestMapping("/order")
  public class OrderController {
  
      private static Logger LOG = LoggerFactory.getLogger(OrderController.class);
  
      @Autowired
      private OrderService orderService;
  
      /**
       * 下单
       * @param order
       * @return
       */
      @PostMapping("/apply")
      public Order apply(@RequestBody Order order){
          LOG.info("申购请求:{}",order);
          order = orderService.apply(order);
          LOG.info("申购结果:{}",order);
          return order;
      }
  
  }
  ```

  #### 下单功能添加RSA加签验签

  修改Controller的apply方法

  ```
  /**
       * 下单
       * @param param
       * @return
       */
      @PostMapping("/apply")
      public Order apply(@RequestHeader String authId,@RequestHeader String sign, @RequestBody OrderParam param){  //注意这里一开始是Order order->OrderParam orderParam
          LOG.info("申购请求:{}",param);
          Order order = new Order();
          //深拷贝
          BeanUtils.copyProperties(param,order);
          order = orderService.apply(order);
          LOG.info("申购结果:{}",order);
          return order;
      }
  ```

  ##### 这里的变动是为什么呢？如果我们传递的时候是个Order对象，对象加密该怎么做呢？

  一般是根据对象中的非空属性根据字典排序之后的Json字符串进行加密，因为不排序的话，前后的排序不同就会使验签不通过，不排除非空的话空格和空字符串会使验签不通过，所以我们最好新建一个接口SignText

  ```
  /**
   * 签名明文
   * @author TiHom
   * create at 2018/8/4 0004.
   */
  
  @JsonInclude(JsonInclude.Include.NON_NULL) //json序列化时包含非空的字段
  @JsonPropertyOrder(alphabetic = true)   //字典排序为true
  public interface SignText {
  	//Java8的新特性
      default String toText(){
          return JsonUtil.toJson(this);
      }
  }
  ```

  对应的对象参数应该实现这个接口，新建一个OrderParam类（这里为什么不在原先的Order上直接实现接口呢，主要是不想让这个entity类太过复杂）

  ```
  /**
   * @author TiHom
   * create at 2018/8/4 0004.
   */
  public class OrderParam implements SignText {
  
      //渠道id
      private String chanId;
     
      private String chanUserId;
  
      private String productId;
  
      private BigDecimal amount;
  
      private String outerOrderId;
  
      private String memo;
  
      @JsonFormat(pattern = "YYYY-MM-DD HH:mm:ss")
      private Date createAt;
  
  	....
      省略get、set
  }
  ```

  ##### 编写AOP

  定义一个验签的AOP方法SignAop

  ```
  /**
   * 验签AOP
   * @author TiHom
   * create at 2018/8/4 0004.
   */
  @Component
  @Aspect
  public class SignAop {
  
      @Autowired
      private SignService signService;
      
  	//在执行具体的方法之前先进行拦截所以用before
      //拦截com.tihom.seller.controller包下的所有类的所有方法，但是限制参数类型为包含这三个类型authId,sign,text的参数
      @Before(value = "execution(* com.tihom.seller.controller.*.*(..)) && args(authId,sign,text,..)")
      public void verify(String authId,String sign,SignText text){
          String publicKey = signService.getPublicKey(authId);
          Assert.isTrue(RSAUtil.verify(text.toText(),sign,publicKey),"验签失败");
          //成功就执行业务方法
      }
  }
  ```

  ##### 编写签名服务

  ```
  /**
   * 签名服务
   * @author TiHom
   * create at 2018/8/4 0004.
   */
  
  @Service
  public class SignService {
  
      //一般实际开发这里的公钥应该放在数据库中或者缓存中
      private static Map<String,String> PUBLIC_KEYS = new HashMap<>();
      static {
          PUBLIC_KEYS.put("1000","MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCsYmTLuYEvuTduISqEdqlXSRvj\\n\" +\n" +
                  "            \"GCHK1Puicr5W75xI025i5AsJ3D4LQU5T36yDCQJ/A/wAU2GN5wkYXwADfU/goKxs\\n\" +\n" +
                  "            \"EiSx1dW+ufxZbl+b2QJph9Fc/rMS4cI7znOOcsMEi4p1/IJCRQAL7gCOC1DWKXzj\\n\" +\n" +
                  "            \"VNd830n9rTw5yt9sTQIDAQAB");
      }
  
      /**
       * 根据授权编号获取公钥
       * @param authId
       * @return
       */
      public String getPublicKey(String authId){
          return PUBLIC_KEYS.get(authId);
      }
  }
  ```

  #### 测试功能

  运行管理端和销售端，打开swagger-ui界面

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/1533393787.jpg)

  

按图中这样填写，然后点击试一下，会得到这种结果

![](http://pbfnr73y8.bkt.clouddn.com/csdn/1533393787%281%29.jpg)

我们返回idea控制台，找到错误的日志打印

![](http://pbfnr73y8.bkt.clouddn.com/csdn/1533393787%282%29.jpg)

将所选的{}内包括{}的内容复制到RSAUtil中的text上运行，得到对应的sign

![](http://pbfnr73y8.bkt.clouddn.com/csdn/1533393849%281%29.jpg)

将打印结果复制（不包含true，也就是复制第一行）到swagger-ui界面的sign中，再试一次得到200

![](http://pbfnr73y8.bkt.clouddn.com/csdn/1533393787%283%29.jpg)

这样就证明成功了！

![](http://pbfnr73y8.bkt.clouddn.com/csdn/1533394300%281%29.jpg)

---

### 对账

#### 对账过程

银行监管套壳公司和金融公司的账户。

小明在套壳公司申购100元，套壳公司记录下小明买了100元，然后套壳公司去银行把小明的账户 -100 元，套壳公司账户 +100 元，返回响应结果给套壳公司，然后套壳公司向金融公司发起一笔申购申请，金融公司就会记录下来小明买了100元，然后响应给套壳公司，套壳公司响应给小明，小明就知道自己已经申购了100元的金融产品（确认中）。

此时资金的状态：小明-100，套壳公司+100，金融公司未收账（因为还处在确认中的状态），直到在交易日确认之后（周末的话移到星期一，工作日的话就是第二天）套壳公司将钱转账给金融公司（具体金额由这两家公司协商的利益占比来具体实现），金融公司就确认小明已经拥有这100元的金融产品了，响应给套壳公司，套壳公司修改状态返回给用户，小明就看到产品“已确认”。

如果小明想拿出钱来，发起申请到套壳公司表示要收回钱，假设小明要赎回110元（带利润），套壳公司记录小明要赎回110元，然后把请求提交到金融公司，金融公司记录小明赎回110元，第二天，金融公司将钱转给套壳公司，套壳公司再根据自己公司的工作日内将钱转回给小明账户。

![](http://pbfnr73y8.bkt.clouddn.com/csdn/%E5%AF%B9%E8%B4%A6%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

##### 这里有几个名词

* 对账：如图所示，套壳公司记录了小明买了100元，金融公司也记录了小明买了100元，对账就是把套壳公司和金融公司的记录进行比对，因为这都是小明的请求，所以要确保两个公司的记录是一致的，才能确定小明的购买请求没有出差错，因为记录就是金钱转移的信息流。![](http://pbfnr73y8.bkt.clouddn.com/csdn/%E5%AF%B9%E8%B4%A6.png)
* 轧差：套壳公司和金融公司之间有申购有赎回，也有可能是小明进行申购小红进行赎回，每天会有非常多的用户进行申购和赎回，每个用户的请求都要进行资金转账，这样会导致两公司之间的转账记录会非常多，并且需要对账才能转账。进行简化后就是把套壳公司的所有申购的金额请求和赎回的金额请求做差，如果申购的多，就需要转账给金融公司，如果赎回的多，金融公司就需要转账给套壳公司，这样就只需要转一笔账。

* 轧账和平账：各个参与方的记录进行比对，发现差异进行处理，处理差异的过程叫平账。

* 长款和漏单：一般是时间问题导致的，比如我在23:59:59发送的申购请求，在套壳公司方则为今天，但是请求发送到金融公司响应回来时则为明天了，所以导致这个请求前后时间不一致，这时候对今天账的时候就没有把记录返回来，导致长款和漏单。这种解决方案就是以某一方的时间为基准。

#### 总体对账流程

![](http://pbfnr73y8.bkt.clouddn.com/csdn/1533399069%281%29.jpg)

#### 对账

##### 对账文件注意

* 只对order_id,chan_id,product_id,chan_user_id,order_type,outer_order_id,amount,create_at的内容
* yyyy-MM-dd HH:mm:ss 日期格式统一
* order_status=success 只对状态是成功的
* 各个字段以'|'作为分隔，放在txt文件中，文件地址为根目录下的/opt/verification/{yyyy-MM-dd-chanId}.txt
* native sql方式执行
* 解析对方的对账文件verification_order入库（verification_order表）

##### 基本流程

* 先建entity对象VerificationOrder（具体参考源码）

* 再建VerifyRepository操作数据库（这些基本知识我就不细讲了）

* 然后创建VerifyService对账服务，添加makeVerificationFile和saveChanOrders方法

* 在VerifyRepository中创建自定义查询

  ```
  /**
   * 对账相关
   * @author TiHom
   * create at 2018/8/5 0005.
   */
  public interface VerifyRepository extends JpaRepository<VerificationOrder,String>,JpaSpecificationExecutor<VerificationOrder> {
  
      /**
       * 查询某段时间[start,end)内的某个渠道chanId的对账数据
       * @param chanId
       * @param start
       * @param end
       * @return 对账数据列表
       */
      @Query(value = "select CONCAT_WS('|',order_id,outer_order_id,chan_id,chan_user_id,product_id,order_type,amount,DATE_FORMAT(create_at,'%Y-%m-%d %H:%i:%s'))\n" +
              "from t_order where order_status='success' and chan_id=?1 and create_at>=?2 and create_at<?3",nativeQuery = true)
      List<String> queryVerificationOrders(String chanId, Date start,Date end);
  }
  ```

* 创建对账测试VerifyTest和渠道配置信息

  ```
  /**
   * 对账测试
   * @author TiHom
   * create at 2018/8/5 0005.
   */
  @RunWith(SpringRunner.class)
  @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT) //使用SpringBoot环境随机端口
  @FixMethodOrder(MethodSorters.NAME_ASCENDING)
  public class VerifyTest {
  
      @Autowired
      private VerifyService verifyService;
      @Autowired
      private OrderRepository orderRepository;
      @Autowired
      private OrderRepository backupOrderRepoistory;
  
      //生成对账文件的功能
      @Test
      public void makeVerificationTest(){
          //创建日历
          Date day = new GregorianCalendar(2018,0,1).getTime();
          //获取对账文件存取路径
          File file = verifyService.makeVerificationFile("111",day);
          //打印绝对路径
          System.out.println(file.getAbsolutePath());
      }
  
      //解析对方的文件并存入数据库
      @Test
      public void saveVerificationOrders(){
          Date day = new GregorianCalendar(2018,0,1).getTime();
          verifyService.saveChanOrders("111",day);
      }
  
  }
  
  ```

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/%EF%BC%91%EF%BC%98%EF%BC%90%EF%BC%98%EF%BC%90%EF%BC%96%E7%94%9F%E6%88%90%E6%9C%AC%E5%9C%B0%E5%9C%B0%E5%9D%80.png)

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/%EF%BC%91%EF%BC%98%EF%BC%90%EF%BC%98%EF%BC%90%EF%BC%96%E5%AF%B9%E8%B4%A6%E6%96%87%E4%BB%B6.png)

  具体测试时测试对账可以先makeVerificationTest（order表中有数据）创建对账文件到本地，然后拷贝一份到ABC中，再saveVerificationOrders解析并保存到数据库

  ![](http://pbfnr73y8.bkt.clouddn.com/csdn/%EF%BC%91%EF%BC%98%EF%BC%90%EF%BC%98%EF%BC%90%EF%BC%96%E6%9C%AC%E5%9C%B0%E5%9C%B0%E5%9D%80%E5%8F%8A%E7%9B%AE%E5%BD%95.jpg)

##### 针对三种情况对账（具体测试比较复杂就不细讲了，这里看我成功的代码）

* 长款
* 漏单
* 不一致 

在VerifyRepository中添加三个自定义查询

```
   /**
     * 查询长款数据
     * @param chanId
     * @param start
     * @param end
     * @return
     */
    @Query(value = "select t.order_id from t_order t left join verification_order v on t.chan_id=?1 and t.outer_order_id=v.order_id where v.order_id is null\n" +
            "and t.create_at>=?2 and t.create_at<?3",nativeQuery = true)
    List<String> queryExcessOrders(String chanId, Date start,Date end);

    /**
     * 查询漏单数据
     * @param chanId
     * @param start
     * @param end
     * @return
     */
    @Query(value = "select v.order_id from verification_order v left join t_order t on t.chan_id=?1 and v.outer_order_id=t.order_id where t.order_id is null\n" +
            "and v.create_at>=?2 and v.create_at<?3",nativeQuery = true)
    List<String> queryMissOrders(String chanId, Date start,Date end);

    /**
     * 查询不一致的数据
     * @param chanId
     * @param start
     * @param end
     * @return
     */
    @Query(value = "select t.order_id from t_order t join verification_order v on t.chan_id=?1 and t.outer_order_id=v.order_id\n" +
            "where CONCAT_WS('|',t.chan_id,t.chan_user_id,t.product_id,t.order_type,t.amount,DATE_FORMAT(t.create_at,'%Y-%m-%d %H:%i:%s')) !=\n" +
            "CONCAT_WS('|',v.chan_id,v.chan_user_id,v.product_id,v.order_type,v.amount,DATE_FORMAT(v.create_at,'%Y-%m-%d %H:%i:%s'))\n" +
            "and t.create_at>=?2 and t.create_at<?3",nativeQuery = true)
    List<String> queryDifferentOrders(String chanId, Date start,Date end);
```

在VerifyService对账服务中添加verifyOrder方法

```
   /**
     * 校验订单
     * @param chanId
     * @param day
     * @return
     */
    public List<String> verifyOrder(String chanId,Date day){
        //创建一个用于存储错误信息的列表
        List<String> errors = new ArrayList<>();
        Date start = getStartOfDay(day);
        //获取start的毫秒时间加上一天的毫秒数就是结束时间
        Date end = add24Hours(start);
        List<String> excessOrders = verifyRepository.queryExcessOrders(chanId,start,end);
        List<String> missOrders = verifyRepository.queryMissOrders(chanId,start,end);
        List<String> differentOrders = verifyRepository.queryDifferentOrders(chanId,start,end);

        //join方法是帮助列表拼接字符串
        errors.add("长款订单号："+String.join(",",excessOrders));
        errors.add("漏单订单号："+String.join(",",missOrders));
        errors.add("不一致订单号："+String.join(",",differentOrders));
        //应该把结果存入数据库，这里直接打印了
        return errors;
    }
```

在VerifyTest中添加verifyTest方法

```
    @Test
    public void verifyTest(){
        Date day = new GregorianCalendar(2018,0,1).getTime();
        System.out.println(String.join(";", verifyService.verifyOrder("111", day)));
    }
```

#### 大数据量的优化（待完善）

* 生成、解析文件分批次（按时间段查询或者按渠道查询，避免一次性查询全部）
* 在备份库或者读库执行（sql语句不要在主库或写库执行）
* Java程序、nosql（不用使用sql方式对账，使用java内读取两方文件比较或者mongodb等等进行对账）

#### 平账（待完善，后期可能会加上去）

* 收到异常对账结果
* 通知人工 邮件或短信
* 轧差

#### 定时对账

* 固定时间自动执行
* @EnableScheduling、@Scheduled（Spring Task提供的）
* cron表达式（在线生成网站：http://www.pppet.net/）

添加VerifyTask类实现定时对账任务

```
/**
 * 定时对账任务
 * @author TiHom
 * create at 2018/8/5 0005.
 */

@Component
public class VerifyTask {

    @Autowired
    private VerifyService verifyService;

    @Scheduled(cron = "0 0 1,3,5 * * ? ") //每天凌晨的1、3、5点执行
    public void makeVerificationFile(){
        Date yesterday = new Date(System.currentTimeMillis()-24*60*60*1000);
        for (ChanEnum chanEnum : ChanEnum.values()) {
            verifyService.makeVerificationFile(chanEnum.getChanId(),yesterday);
        }
    }

    @Scheduled(cron = "0 0 2,4,6 * * ? ")  //每天的2、4、6点执行
    public void verify(){
        Date yesterday = new Date(System.currentTimeMillis()-24*60*60*1000);
        for (ChanEnum chanEnum : ChanEnum.values()) {
            verifyService.verifyOrder(chanEnum.getChanId(),yesterday);
        }
    }
}

```

在SellApp上添加```@EnableScheduling```注解即可

##### 使用Spring提供的定时任务存在的问题

* 多实例部署（定时任务会重复）
* 应用不可用（应用挂掉了，任务就不会再执行了）
* 因此需要自己集成**quartz（待完善，后期完成）**

