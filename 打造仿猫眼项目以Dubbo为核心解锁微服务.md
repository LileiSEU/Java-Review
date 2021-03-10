# 仿猫眼项目---以Dubbo为核心解锁微服务

```shell
# Dubbo 中文官网
https://dubbo.gitbooks.io/dubbo-user-book/content/demos/async-call.html
dubbo 两个注解：@Service @Reference
```

- 后续了解的技术栈：**Dubbo、MyBatisPlus、jwt、Hystrix、SpringAOP**
- 后续粗浅了解：kubernetes ，Openresty，Nginx

## 第一章：微服务入门

### 1.1  课程导学

```shell
# 为什么要使用Dubbo？
# Dubbo 是基于RPC通讯协议，速度更快，二进制传输；
# Dubbo 的多中心配置更灵活
# Dubbo 可以按需集成其他组件，完成微服务生态环境构建
# 包括阿里、小米、京东等多家互联网公司都有使用
```

```shell
# 课程学习目标
# 构建业务完整的商业化项目
# 掌握以 Dubbo 为底的各项微服务套件的应用
# 掌握基于 Dubbo 的微服务常见面试问题，项目中应用
```

![image-20201227115121849](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201227115121849.png)

![image-20201227115552655](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201227115552655.png)



### 1.2  传统应用带来的问题

- 1.单一业务开发和迭代困难；
- 2.扩容困难
- 3.部署和回滚困难



### 1.3  微服务概述

面向服务开发 - SOA

微服务概述

- 微服务是一种将业务系统进一步拆分的架构风格；
- 微服务强调每个单一业务都独立运行，每一个业务独立占用一个JVM，资源独立、业务独立；
- 每个单一服务都应该使用更轻量的机制保持通信；
- 服务不强调环境，可以不同语言或数据源；

微服务选择：

- Dubbo
- Spring   Cloud
- Zero ICE



## 第二章：演示环境构建

微服务基本概念：

- Provider：服务提供者，提供服务实现；
- Consumer：服务调用者，调用Provider提供的服务实现；

```shell
# 注解形式Bean的默认名称：类名首字母小写；
```

```shell
# 直连提供者
# 消费端知道服务提供者的地址，直接进行连接
# 该种方式一般只在测试环境中使用；直连提供者限制了分布式的易扩展性；
```

![image-20201227154340064](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201227154340064.png)

- Provider：暴露服务的服务提供方；
- Consumer：调用远程服务的服务消费方；
- Registry：服务注册与发现的注册中心，通常使用ZooKeeper；
- Monitor：统计服务的调用次数与调用时间的监控中心；
- Container：服务运行容器；

```xml
<!-- Dubbo依赖 -->
<dependency>
    <groupId>com.alibaba.spring.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.0.0</version>
</dependency>
```



### 2.2  Springboot集成注册中心

```xml
<!-- ZooKeeper注册中心依赖 -->
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.10</version>
</dependency>
```



## 第三章：业务基础环境构建

### 3.1  章节概要

- 构建基于Guns +  Springboot  +  Dubbo 的框架
- 学会抽离业务接口
- 学会API网关变形应用



### 3.2 API 网关介绍

```shell
# API 网关
# API 网关有点类似于设计模式中的 Facade 模式；
# API 网关一般都是微服务系统中的门面；
# API 网关是微服务的重要组成部分；

# API 网关的常见作用
# 1. 身份验证和安全
# 2. 审查和检测
# 3. 动态路由
# 4. 压力测试 --> 双十一、双十二电商平台压测
# 5. 负载均衡，API 网关负责负载均衡
# 6. 静态相应处理

# API 网关：服务聚合、熔断降级、身份安全
```

![image-20201227165943156](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201227165943156.png)



### 3.3 guns 环境构建

```yml
# guns URL
url: jdbc:mysql://127.0.0.1:3306/guns_rest?autoReconnect=true&useUnicode=true&characterEncoding=utf8&serverTimezone=GMT%2B8
```

```shell
# jwt auth 路径访问：
http://localhost/auth?userName=admin&password=admin
```

![image-20201228210015238](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201228210015238.png)



### 3.4  API网关模块构建

![image-20201228210058232](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201228210058232.png)

```yml
# dubbo 配置
spring:
  application:
    name: meeting-gateway
  dubbo:
    server: true
    registry: zookeeper://localhost:2181
```

```java
// 接口及接口实现类
// 接口实现类：@Service(com.alibaba.dubbo.config.annotation.Service)
public interface UserAPI {
    boolean login(String username, String password);
}

@Component
@Service(interfaceClass = UserAPI.class)
public class UserImpl implements UserAPI {
    @Override
    public boolean login(String username, String password) {
        return true;
    }
}
```

```java
// 项目启动类， @EnableDubboConfiguration注解
@SpringBootApplication(scanBasePackages = {"com.stylefeng.guns"})
@EnableDubboConfiguration
public class GunsRestApplication {

    public static void main(String[] args) {
        SpringApplication.run(GunsRestApplication.class, args);
    }
}
```



### 3.5  抽离业务API

```shell
# 问题：每个业务模块一个实现类就会有一个接口，微服务架构中各个服务都需要有接口
# 解决方法：单独建立一个工程，承载业务接口以及各个接口的实体类，类似于引入依赖，接口写一遍所有服务都可以使用
```

```shell
# 拷贝 guns-core 命名为 guns-api
```



## 第四章：用户模块开发

### 4.1  章节概要

![image-20201228225842380](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201228225842380.png)

```shell
# 用户模块开发 ---  修改Guns中的JWT模块
    # 增加忽略验证URL配置
    # 修改返回内容匹配业务
    # 增加ThreadLocal的用户信息保存
# 用户模块开发 --- 业务功能开发
	# 增加用户服务并提供接口
	# 初步了解API网关与服务之间交互的过程
	# 根据接口文档开发用户接口
```



### 4.2  用户服务于网关交互

```shell
# 用户通过浏览器输入账号密码，网关API调用User服务，将User中需要的Username 和 password，传给user服务并在控制台输出；
```

![image-20201229204300838](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201229204300838.png)

```yml
# 基于 SpringBoot 配置 jwt 的忽略列表
# jwt:
  header: Authorization   #http请求头所需要的字段
  secret: mySecret        #jwt秘钥
  expiration: 604800      #7天 单位:秒
  auth-path: auth         #认证请求的路径
  md5-key: randomKey      #md5加密混淆key
  ignore-url: /user/,/film  #忽略列表

# 在 Jwtproperties.class 类中配置变量 ignoreUrl， 并提供 set 和 get 方法
# private String ignoreUrl = "";

# 在 AuthFilter.class 类中配置忽略列表
// 配置忽略列表
String ignoreUrl = jwtProperties.getIgnoreUrl();
String[] ignoreUrls = ignoreUrl.split(",");
for(int i = 0; i < ignoreUrls.length; i ++){
    if(request.getServletPath().equals(ignoreUrls[i])){
          chain.doFilter(request, response);
          return;
    }
}
```



### 4.3  用户模块开发(1)

- UserModel 和 UserInfoModel 两个实体类，用于跨模块(跨Guns-gateway/Guns-user模块)交互

```java
// 创建UserModel实体类，用于用户登录，注册信息
public class UserModel {
    private String username;
    private String password;
    private String email;
    private String phone;
    private String address;
}
// 创建 UserInfoModel 实体类，用于存储用户所有信息，所有用到用户信息的都用UserInfoModel
public class UserInfoModel implements Serializable {
    private Integer uuid;
    private String username;
    private String nickname;
    private String email;
    private String phone;
    private int sex;
    private String birthday;
    private String lifeState;
    private String biography;
    private String address;
    private String headAddress;
    private long beginTime;
    private long updateTime;
}
```



### 4.4 修改Guns中的JWT模块

- 增加忽略验证URL配置；
- 修改返回内容匹配业务；
- 增加Threadlocal的用户信息保存；



**修改返回内容匹配业务**

```java
// 返回实体， 单例设计模式
public class ResponseVO<M> {
    // 返回状态【0-成功，1-业务失败，999-表示系统异常】
    private int status;
    // 返回信息
    private String msg;
    // 返回数据实体;
    private M data;

    // 单例设计模式
    private ResponseVO(){}

    // 成功
    public static<M> ResponseVO success(M m){
        ResponseVO responseVO = new ResponseVO();
        responseVO.setStatus(0);
        responseVO.setData(m);

        return responseVO;
    }
    public static<M> ResponseVO success(String msg){
        ResponseVO responseVO = new ResponseVO();
        responseVO.setStatus(0);
        responseVO.setMsg(msg);

        return responseVO;
    }
    // 业务异常
    public static<M> ResponseVO serviceFail(String msg){
        ResponseVO responseVO = new ResponseVO();
        responseVO.setStatus(1);
        responseVO.setMsg(msg);

        return responseVO;
    }
    // 系统异常
    public static<M> ResponseVO appFail(String msg){
        ResponseVO responseVO = new ResponseVO();
        responseVO.setStatus(999);
        responseVO.setMsg(msg);

        return responseVO;
    }
	// 实体的set和get方法
   
}
```



**ThreadLocal保存用户信息**

```java
// com/stylefeng/guns/rest/common/CurrentUser.java
// Threadlocal 有OOM(Out Of Memory) 的危险，所以不能存过大的对象
public class CurrentUser {
    // 线程绑定的存储空间，将userId 放入存储空间
    // Threadlocal 中内容不能过大，影响JVM性能
    private static final ThreadLocal<String> threadLocal = new ThreadLocal<>();
    public static void saveUserId(String userId){
        threadLocal.set(userId);
    }
    public static String getCurrentUser(){
        return threadLocal.get();
    }
}
```

```java
// AuthFilter 类通过Token获取userId，并存入ThreadLocal，以便后续业务调用
String userId = jwtTokenUtil.getUsernameFromToken(authToken);
    if(userId == null){
          return;
    } else {
          CurrentUser.saveUserId(userId); // UserID 放入ThreadLocal中
}
```



### 4.5 用户模块开发(2)

业务功能开发流程

- 使用代码生成器生成数据项
- 实现相应的接口功能



**Guns代码生成器**

```shell
# 生成 DAO 层
# Guns 的代码生成器在每个模块的 /test/java/com/stylefeng/guns/generator/EntityGenerator
# MoocUserT 与数据库对应，包含数据库的所有字段；
# 模块之间调用用户信息使用 UserInfoModel
```

![image-20201230084813446](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201230084813446.png)



- 数据库密码存储：明文密码 + MD5混合加密 + 随机盐值(Salt)

**用户模块**

```java
// 用户模块功能实现
// 注册、登录、用户名验证、查询用户信息、修改用户信息
@Component
@Service(interfaceClass = UserAPI.class)
public class UserServiceImpl implements UserAPI {
    @Autowired
    private MoocUserTMapper moocUserTMapper;

    @Override
    public boolean register(UserModel userModel) {
        // 将注册信息实体转换为数据实体[mooc_user_t]
        MoocUserT moocUserT = new MoocUserT();
        moocUserT.setUserName(userModel.getUsername());
        moocUserT.setEmail(userModel.getEmail());
        moocUserT.setAddress(userModel.getAddress());
        moocUserT.setUserPhone(userModel.getPhone());
        // 创建时间和修改时间 -> current_timestamp
        // 数据加密 【MD5混淆加密 + 盐值 -> Shiro加密】
        String md5Password = MD5Util.encrypt(userModel.getPassword());
        moocUserT.setUserPwd(md5Password); // 注意

        // 将数据实体存入数据库
        Integer insert = moocUserTMapper.insert(moocUserT);
        if(insert>0){
            return true;
        }else{
            return false;
        }
    }

    @Override
    public int login(String username, String password) {
        // 根据登陆账号获取数据库信息
        MoocUserT moocUserT = new MoocUserT();
        moocUserT.setUserName(username);

        MoocUserT result = moocUserTMapper.selectOne(moocUserT);

        // 获取到的结果，然后与加密以后的密码做匹配
        if(result!=null && result.getUuid()>0){
            String md5Password = MD5Util.encrypt(password);
            if(result.getUserPwd().equals(md5Password)){
                return result.getUuid();
            }
        }
        return 0;
    }

    @Override
    public boolean checkUsername(String username) {
        EntityWrapper<MoocUserT> entityWrapper = new EntityWrapper<>();
        entityWrapper.eq("user_name",username);
        Integer result = moocUserTMapper.selectCount(entityWrapper);
        if(result!=null && result>0){
            return false;
        }else{
            return true;
        }
    }

    private UserInfoModel do2UserInfo(MoocUserT moocUserT){
        UserInfoModel userInfoModel = new UserInfoModel();

//        userInfoModel.setUuid(moocUserT.getUuid());
        userInfoModel.setHeadAddress(moocUserT.getHeadUrl());
        userInfoModel.setPhone(moocUserT.getUserPhone());
        userInfoModel.setUpdateTime(moocUserT.getUpdateTime().getTime());
        userInfoModel.setEmail(moocUserT.getEmail());
        userInfoModel.setUsername(moocUserT.getUserName());
        userInfoModel.setNickname(moocUserT.getNickName());
        userInfoModel.setLifeState(""+moocUserT.getLifeState());
        userInfoModel.setBirthday(moocUserT.getBirthday());
        userInfoModel.setAddress(moocUserT.getAddress());
        userInfoModel.setSex(moocUserT.getUserSex());
        userInfoModel.setBeginTime(moocUserT.getBeginTime().getTime());
        userInfoModel.setBiography(moocUserT.getBiography());

        return userInfoModel;
    }

    @Override
    public UserInfoModel getUserInfo(int uuid) {
        // 根据主键查询用户信息 [MoocUserT]
        MoocUserT moocUserT = moocUserTMapper.selectById(uuid);
        // 将MoocUserT转换UserInfoModel
        UserInfoModel userInfoModel = do2UserInfo(moocUserT);
        // 返回UserInfoModel
        return userInfoModel;
    }

    private Date time2Date(long time){
        Date date = new Date(time);
        return date;
    }

    @Override
    public UserInfoModel updateUserInfo(UserInfoModel userInfoModel) {

        // 将传入的参数转换为DO 【MoocUserT】
        MoocUserT moocUserT = new MoocUserT();
        moocUserT.setUuid(userInfoModel.getUuid());
        moocUserT.setNickName(userInfoModel.getNickname());
        moocUserT.setLifeState(Integer.parseInt(userInfoModel.getLifeState()));
        moocUserT.setBirthday(userInfoModel.getBirthday());
        moocUserT.setBiography(userInfoModel.getBiography());
        moocUserT.setBeginTime(time2Date(userInfoModel.getBeginTime()));
        moocUserT.setHeadUrl(userInfoModel.getHeadAddress());
        moocUserT.setEmail(userInfoModel.getEmail());
        moocUserT.setAddress(userInfoModel.getAddress());
        moocUserT.setUserPhone(userInfoModel.getPhone());
        moocUserT.setUserSex(userInfoModel.getSex());
        moocUserT.setUpdateTime(time2Date(System.currentTimeMillis()));

        // DO存入数据库
        Integer isSuccess = moocUserTMapper.updateById(moocUserT);
        if(isSuccess>0){
            // 将数据从数据库中读取出来
            UserInfoModel userInfo = getUserInfo(moocUserT.getUuid());
            // 将结果返回给前端
            return userInfo;
        }else{
            return null;
        }
    }
    
}
```

**网关模块**

```java
@RequestMapping("/user/")
@RestController
public class UserController {

    @Reference(interfaceClass = UserAPI.class,check = false)
    private UserAPI userAPI;

    // 登录
    @RequestMapping(value="register",method = RequestMethod.POST)
    public ResponseVO register(UserModel userModel){
        if(userModel.getUsername() == null || userModel.getUsername().trim().length()==0){
            return ResponseVO.serviceFail("用户名不能为空");
        }
        if(userModel.getPassword() == null || userModel.getPassword().trim().length()==0){
            return ResponseVO.serviceFail("密码不能为空");
        }

        boolean isSuccess = userAPI.register(userModel);
        if(isSuccess){
            return ResponseVO.success("注册成功");
        }else{
            return ResponseVO.serviceFail("注册失败");
        }
    }

    @RequestMapping(value="check",method = RequestMethod.POST)
    public ResponseVO check(String username){
        if(username!=null && username.trim().length()>0){
            // 当返回true的时候，表示用户名可用
            boolean notExists = userAPI.checkUsername(username);
            if (notExists){
                return ResponseVO.success("用户名不存在");
            }else{
                return ResponseVO.serviceFail("用户名已存在");
            }

        }else{
            return ResponseVO.serviceFail("用户名不能为空");
        }
    }

    @RequestMapping(value="logout",method = RequestMethod.GET)
    public ResponseVO logout(){
        /*
            应用：
                1、前端存储JWT 【七天】 ： JWT的刷新
                2、服务器端会存储活动用户信息【30分钟】
                3、JWT里的userId为key，查找活跃用户，30分钟外查不到活跃用户
            退出（正式业务中）：
                1、前端删除掉JWT
                2、后端服务器删除活跃用户缓存
            现状(仿猫眼项目中)：
                1、前端删除掉JWT
         */
        return ResponseVO.success("用户退出成功");
    }

    @RequestMapping(value="getUserInfo",method = RequestMethod.GET)
    public ResponseVO getUserInfo(){
        // 获取当前登陆用户
        String userId = CurrentUser.getCurrentUser();
        if(userId != null && userId.trim().length()>0){
            // 将用户ID传入后端进行查询
            int uuid = Integer.parseInt(userId);
            UserInfoModel userInfo = userAPI.getUserInfo(uuid);
            if(userInfo!=null){
                return ResponseVO.success(userInfo);
            }else{
                return ResponseVO.appFail("用户信息查询失败");
            }
        }else{
            return ResponseVO.serviceFail("用户未登陆");
        }
    }

    @RequestMapping(value="updateUserInfo",method = RequestMethod.POST)
    public ResponseVO updateUserInfo(UserInfoModel userInfoModel){
        // 获取当前登陆用户
        String userId = CurrentUser.getCurrentUser();
        if(userId != null && userId.trim().length()>0){
            // 将用户ID传入后端进行查询
            int uuid = Integer.parseInt(userId);
            // 判断当前登陆人员的ID与修改的结果ID是否一致
            if(uuid != userInfoModel.getUuid()){
                return ResponseVO.serviceFail("请修改您个人的信息");
            }
            
            UserInfoModel userInfo = userAPI.updateUserInfo(userInfoModel);
            if(userInfo!=null){
                return ResponseVO.success(userInfo);
            }else{
                return ResponseVO.appFail("用户信息修改失败");
            }
        }else{
            return ResponseVO.serviceFail("用户未登陆");
        }
    }

}
```



**业务总结：**

- 验证忽略URL列表
- 申请JWT
- 使用JWT访问其他权限功能

- **所有需要交互的Model必须实现Serializable接口，可序列化之后才可以传输**



### 4.6  Dubbo特性

存在的问题：

- 必须先启动服务提供者，否则会报错；
- 如果我们将用户模块部署多态，消费者会如何访问
- Dubbo 的 Protocol(协议是什么)



**启动检查**

- 服务启动过程中验证服务提供者的可用性；
- 验证过程出现问题，则阻止整个Spring容器初始化；
- 服务启动检查可以尽可能早的发现服务问题；
- 绝大多数场景不建议配置 checked = false；
- 特殊情况下两个服务互相依赖，就需要去掉启动检查

```java
// 解决：@Referecnce 注解中添加 check = false
@Reference(interfaceClass = UserAPI.class,check = false)
```



**负载均衡**

```shell
# Dubbo 负载均衡策略
# Random LoadBalance
#     随机，按权重设置随机概率。
#     在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。
# RoundRobin LoadBalance
#   轮循，按公约后的权重设置轮循比率。
#    存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。
# LeastActive LoadBalance
#    最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。
#    使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。
# ConsistentHash LoadBalance
#    一致性 Hash，相同参数的请求总是发到同一提供者。
#    当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
```

![image-20201230112048923](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201230112048923.png)

**负载均衡配置**

- 可以配置在Service端或客户端；
- 一般配置在Service端；

```shell
# 服务端服务级别
<dubbo:service interface="..." loadbalance="roundrobin" />
# 客户端服务级别
<dubbo:reference interface="..." loadbalance="roundrobin" />
# 服务端方法级别
<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:service>
# 客户端方法级别
<dubbo:reference interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:reference>

# 配置在客户端
@Component
@Service(interfaceClass = UserAPI.class, loadbalance = "roundrobin")
public class UserServiceImpl implements UserAPI {}
```



**protocol**

```yml
# Dubbo protocol协议配置
spring:
  application:
    name: meeting-user
  dubbo:
    server: true
    registry: zookeeper://localhost:2181
    protocol:
      name: dubbo		# 配置Dubbo框架的通信协议，一般为 dubbo/rmi
      port: 20881		# 协议通信端口
```

- protocol 配置服务之间的通信协议，HTTP协议、TCP协议、UTP协议；

- Dubbo 支持多种协议，最常见的是dubbo(注意：是Dubbo框架中的dubbo通信协议)
- dubbo协议封装了TCP协议；

![image-20201230113758357](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201230113758357.png)

- 长连接：服务消费者和服务提供者建立管道，较少握手和连接时间；
- dubbo协议的数据包较小，一般100k左右；多个消费者的对应一个服务者，消费者远大于服务者的情况，多个gaetway对应一个user；



### 4.7  业务总结

学习的内容：

- API 网关(gateway服务消费者)与服务模块(user服务提供者)调用方式；
- Dubbo特性：负载均衡、启动检查、Dubbo协议；
- JWT 的业务应用；



## 第五章：影片模块开发

### 章节概要

- 掌握 API 网关服务聚合功能实现
  - 服务聚合就是将多个服务调用封装
  - 服务聚合可以简化前端调用方式
  - 服务聚合提供更好的安全性、可扩展性
- 掌握 Mybatis - plus 自定义 SQL 实现(复杂查询的拼接)；
- 掌握 Dubbo 异步调用(几个服务并行查询之后做结合)；



### 5.1  初始API网关特性-功能聚合

```shell
#  API 网关
#  1、功能聚合
#	好处：1.六个接口，一次请求，同一时刻节省了五次HTTP请求
#		 2.同一个接口对外暴露，降低了前后端分离开发的难度和复杂度
#	坏处：
#		 1.一次获取数据过多，容易出现问题
```

```java
// 页面对象创建
@Data
public class FilmIndexVO {

    private List<BannerVO> banners;
    private FilmVO hotFilms;
    private FilmVO soonFilms;
    private List<FilmInfo> boxRanking;
    private List<FilmInfo> expectRanking;
    private List<FilmInfo> top100;
    
}
```

- API 设计：状态表示在数据层，不能将数据库表示在前端(getHotFilm  和 getSoonFilm)；



### 5.2  首页实现

### 5.3  条件列表实现

- 判断集合是否存在catId，如果存在，则将对应的实体变成active状态； 如果不存在则默认全部变为Active状态；

### 5.4  影片查询功能实现

- 入参Model给请求传递对象，FilmRequestVO类；

### 5.5  影片详情查询

```shell
# 联合查询，SQL 语句，注意 MySQL 函数
SELECT
   film.uuid AS filmId,
   film.film_name AS filmName,
   info.`film_en_name` AS filmEnName,
   film.`img_address` AS imgAddress,
   info.`film_score` AS score,
   info.`film_score_num` AS scoreNum,
   film.`film_box_office` AS totalBox,
			(SELECT GROUP_CONCAT(show_name SEPARATOR ',') FROM mooc_cat_dict_t t
              WHERE FIND_IN_SET (t.uuid,
                (SELECT REPLACE(TRIM(BOTH '#' FROM film_cats),'#',',') FROM mooc_film_t t WHERE t.uuid=film.uuid))) AS info01,
    CONCAT((SELECT show_name FROM mooc_source_dict_t t WHERE t.uuid=film.uuid),' / ',info.`film_length`,'分钟') info02,
    CONCAT(film.`film_time`,(SELECT show_name FROM mooc_source_dict_t t WHERE t.uuid=film.uuid),'上映') info03
FROM mooc_film_t film,mooc_film_info_t info
WHERE film.`UUID` = info.`film_id`
AND film.`UUID` = #{uuid}
```

```sql
# 字符串拼接函数 CONCAT
CONCAT((SELECT show_name FROM mooc_source_dict_t t WHERE t.uuid=film.uuid),' / ',info.`film_length`,'分钟') info02,
CONCAT(film.`film_time`,(SELECT show_name FROM mooc_source_dict_t t WHERE t.uuid=film.uuid),'上映') info03
```

```sql
# 将数据库中的数据格式：#2#4#22#，转换为 爱情/喜剧/动画
# 思路： #2#4#22# ---> 2,4,22  ---> mooc_cat_dict_t 中查找
# 第一步：
SELECT REPLACE(TRIM(BOTH '#' FROM film_cats),'#',',') FROM mooc_film_t
# 第二步：
SELECT GROUP_CONCAT(show_name SEPARATOR ',') FROM mooc_cat_dict_t t
WHERE FIND_IN_SET (t.uuid,(SELECT REPLACE(TRIM(BOTH '#' FROM film_cats),'#',',') FROM mooc_film_t t WHERE t.uuid=film.uuid))
```

**电影表和演员表是一对多的关系，使用映射表 mooc_film_actor_t**



### 5.6  Dubbo异步调用

**存在的问题**

- 影片详情查询接口存在多个服务访问；

```java
// 查询影片的详细信息 -> Dubbo的异步调用，现在是同步调用多个接口，一个调用接口出结果后，调下一个接口
// 获取影片描述信息
FilmDescVO filmDescVO = filmServiceApi.getFilmDesc(filmId);
// 获取图片信息
ImgVO imgVO = filmServiceApi.getImgs(filmId);
// 获取导演信息
ActorVO directorVO = filmServiceApi.getDectInfo(filmId);
// 获取演员信息
List<ActorVO> actors = filmServiceApi.getActors(filmId);
```

![image-20210102114715327](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210102114715327.png)

- Java 异步调用：@Reference(interfaceClass = ,asyc = true)，表示接口中的所有方法都会被异步调用，若方法中没有Future对象就会报错，所以这也是Dubbo异步调用的缺点；Dubbo 注解只有 @Service 和 @Reference，无法在Method方法上加异步调用注解；
- 异步调用：UserThread 获取 Future 对象；

```java
// 第一步：创建异步调用接口： FilmAsyncServiceApi
public interface FilmAsyncServiceApi {
    // 获取影片描述信息
    FilmDescVO getFilmDesc(String filmId);

    // 获取图片信息
    ImgVO getImgs(String filmId);

    // 获取导演信息
    ActorVO getDectInfo(String filmId);

    // 获取演员信息
    List<ActorVO> getActors(String filmId);
}
```

```java
// 第二步：异步接口实现类  DefaultFilmAsyncServiceImpl
@Component
@Service(interfaceClass = FilmAsyncServiceApi.class)
public class DefaultFilmAsyncServiceImpl implements FilmAsyncServiceApi {

    @Autowired
    private MoocFilmInfoTMapper moocFilmInfoTMapper;
    @Autowired
    private MoocActorTMapper moocActorTMapper;

    private MoocFilmInfoT getFilmInfo(String filmId){}
    @Override
    public FilmDescVO getFilmDesc(String filmId) {}
    @Override
    public ImgVO getImgs(String filmId) {}
    @Override
    public ActorVO getDectInfo(String filmId) {}
    @Override
    public List<ActorVO> getActors(String filmId) {}
}
```

```java
// 第三步：控制器中引入 异步调用接口：FilmAsyncServiceApi
@RestController
@RequestMapping("/film/")
public class FilmController {
    
    @Reference(interfaceClass = FilmServiceApi.class)
    private FilmServiceApi filmServiceApi;

    @Reference(interfaceClass = FilmAsyncServiceApi.class,async = true)
    private FilmAsyncServiceApi filmAsyncServiceApi;
    
    @RequestMapping(value = "films/{searchParam}",method = RequestMethod.GET)
    public ResponseVO films(@PathVariable("searchParam")String searchParam,
                            int searchType) throws ExecutionException, InterruptedException {
        // 根据searchType，判断查询类型
        FilmDetailVO filmDetail = filmServiceApi.getFilmDetail(searchType, searchParam);

        if(filmDetail==null){
            return ResponseVO.serviceFail("没有可查询的影片");
        }else if(filmDetail.getFilmId()==null || filmDetail.getFilmId().trim().length()==0){
            return ResponseVO.serviceFail("没有可查询的影片");
        }

        String filmId = filmDetail.getFilmId();
        // 查询影片的详细信息 -> Dubbo的异步调用
        // 获取影片描述信息
//        FilmDescVO filmDescVO = filmAsyncServiceApi.getFilmDesc(filmId);
        filmAsyncServiceApi.getFilmDesc(filmId);
        Future<FilmDescVO> filmDescVOFuture = RpcContext.getContext().getFuture();
        // 获取图片信息
        filmAsyncServiceApi.getImgs(filmId);
        Future<ImgVO> imgVOFuture = RpcContext.getContext().getFuture();
        // 获取导演信息
        filmAsyncServiceApi.getDectInfo(filmId);
        Future<ActorVO> actorVOFuture = RpcContext.getContext().getFuture();
        // 获取演员信息
        filmAsyncServiceApi.getActors(filmId);
        Future<List<ActorVO>> actorsVOFutrue = RpcContext.getContext().getFuture();

        // 组织info对象
        InfoRequstVO infoRequstVO = new InfoRequstVO();

        // 组织Actor属性
        ActorRequestVO actorRequestVO = new ActorRequestVO();
        actorRequestVO.setActors(actorsVOFutrue.get());
        actorRequestVO.setDirector(actorVOFuture.get());

        // 组织info对象
        infoRequstVO.setActors(actorRequestVO);
        infoRequstVO.setBiography(filmDescVOFuture.get().getBiography());
        infoRequstVO.setFilmId(filmId);
        infoRequstVO.setImgVO(imgVOFuture.get());

        // 组织成返回值
        filmDetail.setInfo04(infoRequstVO);

        return ResponseVO.success("http://img.meetingshop.cn/",filmDetail);
    }
    
}
```

```java
// 第四步：SpringBoot 启动类添加注解 @EnableAsync
@SpringBootApplication(scanBasePackages = {"com.stylefeng.guns"})
@EnableAsync
@EnableDubboConfiguration
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

![image-20210102121401368](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210102121401368.png)





### 5.7  业务总结

**学习的内容**

- API 网关的服务聚合，注入依赖；
- API网关功能：路由转发(Dubbo使用不多,Dubbo 依靠注入依赖) ，服务聚合；
- Dubbo 特性：异步调用Interface级别，缺点：无法实现 Method级别的异步调用；
- Mybatis - plus 自定义SQL实现



## 第六章： 影院模块开发

### 章节概要

- 完成影院模块业务开发
- 修改全局异常返回
- 学习 Dubbo 特性：并发控制，结果缓存

![image-20210102150943059](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210102150943059.png)



**查询某影院下所有电影和场次**

- 一个电影对应多个场次，一个SQL表示出一对多；
- 一对多的查询(Mybatis)，关联查询；
- 注意：中文字符存储在数据库中很耗资源，一般尽量减少数据库对中文的存储，所以不会把很多中文信息存储在同一张表中；
- <resultMap> 标签，屏蔽实体对象和表不一致的问题；

```xml
<!-- Mybatis 一对多配置<collection>标签及 SQL 语句 -->
<!-- <resultMap> 标签，屏蔽实体对象和表不一致的问题 -->
<!-- 一对多的查询 -->
    <resultMap id="getFilmInfoMap" type="com.stylefeng.guns.api.cinema.vo.FilmInfoVO">
        <result column="film_id" property="filmId"></result>
        <result column="film_name" property="filmName"></result>
        <result column="film_length" property="filmLength"></result>
        <result column="film_language" property="filmType"></result>
        <result column="film_cats" property="filmCats"></result>
        <result column="actors" property="actors"></result>
        <result column="img_address" property="imgAddress"></result>
        <collection property="filmFields" ofType="com.stylefeng.guns.api.cinema.vo.FilmFieldVO">
            <result column="UUID" property="fieldId"></result>
            <result column="begin_time" property="beginTime"></result>
            <result column="end_time" property="endTime"></result>
            <result column="film_language" property="language"></result>
            <result column="hall_name" property="hallName"></result>
            <result column="price" property="price"></result>
        </collection>
    </resultMap>

<select id="getFilmInfos" parameterType="java.lang.Integer" resultMap="getFilmInfoMap">
        SELECT
          info.film_id,
          info.`film_name`,
          info.`film_length`,
          info.`film_language`,
          info.`film_cats`,
          info.`actors`,
          info.`img_address`,
          f.`UUID`,
          f.`begin_time`,
          f.`end_time`,
          f.`hall_name`,
          f.`price`
        FROM
          mooc_hall_film_info_t info
        LEFT JOIN
          mooc_field_t f
        ON f.`film_id` = info.`film_id`
        AND f.`cinema_id` = ${cinemaId}
</select>
```



### 6.1  Dubbo特性：结果缓存(cache = "lru")

**结果缓存：用于加速热门数据的访问速度，Dubbo提供声明式缓存，以减少用户加缓存的工作量。**

- Dubbo可以通过注解对热点数据进行缓存
- 热点数据：访问频率高且不怎么变化的数据，小数据量可以考虑放在Dubbo缓存，大数据量放在Redis缓存中；
- 了解Dubbo结果缓存与Redis等的区别

**缓存类型**

- lru ，基于最近最少使用原则删除多于内存，保持最热的数据被缓存；
- threadlocal，当前线程缓存比如一个页面渲染，用到很多portal，每个portal 都要去查用户信息，通过线程缓存可以减少这种多余访问；
- jcache，与JSR107集成，可以桥接各种缓存实现（使用较少）；

```java
@Reference(interfaceClass = CinemaServiceAPI.class, cache = "lru", check = false)
private CinemaServiceAPI cinemaServiceAPI;
```



### 6.2  Dubbo特性：并发与连接控制

- dubbo协议是长连接；
- Dubbo可以对链接和并发数量进行控制；
- 超出部分以错误形式返回；
- 服务雪崩：服务瞬间访问量很大，冲垮一个服务，后面的服务马上会跟随着崩掉；

**Service 和 Reference 配置区别**

- 配置在Service上，控制Reference的连接数为10个，超出10个的连接直接返回；

![image-20210102223310613](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210102223310613.png)

- 配置在Reference上，最多有10个Reference连接，Service端连接是不受限制的；

![image-20210102223421278](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210102223421278.png)

```shell
# 并发与控制连接配置：
# guns-cinema 模块的 application.yml 文件配置，accepts: 10
spring:
  application:
    name: meeting-user
  dubbo:
    server: true
    registry: zookeeper://localhost:2181
    protocol:
      name: dubbo
      port: 20885
      accepts: 10
      
# 服务提供者 DefaultCinemaServiceImpl，配置 executes = 10
@Service(interfaceClass = CinemaServiceAPI.class, executes = 10)
public class DefaultCinemaServiceImpl implements CinemaServiceAPI

# Reference端CinemaController，@Reference注解配置 connections = 10
@Reference(interfaceClass = CinemaServiceAPI.class,
                connections = 10,cache = "lru", check = false)
    private CinemaServiceAPI cinemaServiceAPI;
```



## 第七章： 订单模块开发*

### 章节概要

- 完成订单模块业务开发；
- 完成限流和熔断、降级相关内容；（完成高可用功能）
- Dubbo 特性之分组聚合和版本控制；



```shell
# 创建 Windows ftp服务器，
# 用户名：ftp
# 密码：ftp
# 设置 ftp 站点：192.168.1.103 端口：2100		ipconfig命令查询本机IP地址
# 访问 ftp：ftp://192.168.1.103:2100/
# ftp 服务器文件位置：D:/JAVA/ftp
```

![image-20210103093714520](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210103093714520.png)

![image-20210103094040019](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210103094040019.png)



### 7.1  订单模块分析

```shell
# 购票逻辑
# 1.验证售出的票是否为真
# 2.验证已经销售的座位里，有没有这些座位
# 3.创建订单

# 获取订单信息
# 1.获取当前登录人的信息
# 2.使用当前登录人获取已经购买的订单

# 影院模块
# 影院模块要显示已经销售的座位
```



### 7.2 SpringBoot集成ftp

```xml
<!-- ftp依赖 -->
<dependency>
    <groupId>commons-net</groupId>
    <artifactId>commons-net</artifactId>
    <version>3.6</version>
</dependency>
```

```java
// 编辑 FTPUtil 类
@Slf4j
@Data
@Configuration
@ConfigurationProperties(prefix = "ftp")
public class FTPUtil {

    // 地址 端口 用户名 密码
    private String hostName="192.168.1.103";
    private Integer port=2100;
    private String userName="ftp";
    private String password="ftp";

    private FTPClient ftpClient = null;

    private void initFTPClient(){
        try{
            ftpClient = new FTPClient();
            ftpClient.setControlEncoding("utf-8");
            ftpClient.connect(hostName,port);
            ftpClient.login(userName,password);
        }catch (Exception e){
            log.error("初始化FTP失败",e);
        }
    }

    // 输入一个路径，然后将路径里的文件转换成字符串返回给我
    public String getFileStrByAddress(String fileAddress){
        BufferedReader bufferedReader = null;
        try{
            initFTPClient();
            bufferedReader = new BufferedReader(
                    new InputStreamReader(
                            ftpClient.retrieveFileStream(fileAddress))
            );

            StringBuffer stringBuffer = new StringBuffer();
            while(true){
                String lineStr = bufferedReader.readLine();
                if(lineStr == null){
                    break;
                }
                stringBuffer.append(lineStr);
            }

            ftpClient.logout();
            return stringBuffer.toString();
        }catch (Exception e){
            log.error("获取文件信息失败",e);
        }finally {
            try {
                bufferedReader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return null;
    }
}
```

```yml
# application.yml 配置文件中的 ftp 配置
ftp:
  host-name: 192.168.1.103
  port: 2100
  user-name: ftp
  password: ftp
```



### 7.3 订单模块

- 微服务之间相互调用，DefaultOrderServiceImpl 类中调用 CinemaServiceAPI；
- orderTimestamp : 时间戳，订单生成到支付的有效时间(ex:14min);



```sql
SQL语句1：
<select id="getOrderInfoById" parameterType="java.lang.String" resultType="com.stylefeng.guns.api.order.vo.OrderVO">
        SELECT
          o.`UUID` AS orderId,
          h.`film_name` AS filmName,
          CONCAT(DATE_FORMAT(o.`order_time`,'%y年%m月%d日'),' ',f.`begin_time`) AS fieldTime,
          c.`cinema_name` AS cinemaName,
          o.`seats_name` AS seatsName,
          o.`order_price` AS orderPrice,
          UNIX_TIMESTAMP(o.`order_time`) AS orderTimestamp
        FROM
          mooc_order_t o,
          mooc_field_t f,
          mooc_hall_film_info_t h,
          mooc_cinema_t c
        WHERE o.`cinema_id` = c.`UUID`
          AND o.`field_id` = f.`UUID`
          AND o.`film_id` = h.`film_id`
          AND o.`UUID` = #{orderId}
</select>
```

**订单业务之后的问题总结**

- 订单模块的横向和纵向拆表解决；
- 服务限流如何处理；
- 服务熔断和降级，防止业务系统雪崩；
- 如何保证多版本的蓝绿上线；



### 7.4  Dubbo特性分组聚合、版本控制

订单表的横向拆分和纵向拆分：

- 订单表拆分后的问题：
  - 业务复杂度变高：例如查询所有订单，需要将纵向拆分的2017年订单和2018年订单合并到一起；
  - 业务人员避免查询过多的表，拆分表之后不会修改表结构即SQL语句还可以通用；

![image-20210103215750146](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210103215750146.png)



```shell
# 服务分组：总是只调一个可用组的实现。
# 当一个接口有多种实现时，可以用 group 区分；
# 分组：比如查询的一个影片信息在缓存和数据库中都有，可以用分组来配置查询的位置：缓存或数据库；
# 任意组：
<dubbo:reference id="barService" interface="com.foo.BarService" group="*" />
```

```shell
# 分组聚合
# 按组合并返回结果，比如菜单服务，接口一样但有多种实现，用 group 区分，现在消费方需从每种 group 中调用一次返回结果，合并结果返回，这样就可以实现聚合菜单项；
# 搜索所有分组：
<dubbo:reference interface="com.xxx.MenuService" group="*" merger="true" />
```

```shell
# 多版本：版本控制，类似于版本隔离
# 当一个接口实现，出现不兼容升级时，可以用版本号过滤，版本号不同的服务相互间不引用；
# 可以按照以下的步骤进行版本迁移：
#	1.在低压力时间段，先升级一般提供者为新版本；
#	2.再将所有消费者升级为新版本；
#	3.然后将剩下的一半提供者升级为新版本；
新版本服务提供者配置：
<dubbo:reference interface="com.xxx.MenuService" group="*" merger="true" />
新版本服务消费者配置：
<dubbo:reference id="barService" interface="com.foo.BarService" version="2.0.0" />
如果不需要区分版本，可以按照以下的方式配置：
<dubbo:reference id="barService" interface="com.foo.BarService" version="*" />
```

```java
// 将2017年订单和2018年订单查询结果组合
// getOrderInfo 方法查询用户所有订单，2017和2018的订单总数合并
// 控制器中引入 OrderSerivceAPI 接口的两个实现组：order2018、order2017；

@Service(interfaceClass = OrderServiceAPI.class,group = "order2017")
public class OrderServiceImpl2017 implements OrderServiceAPI

@Service(interfaceClass = OrderServiceAPI.class,group = "order2018")
public class OrderServiceImpl2018 implements OrderServiceAPI

@Slf4j
@RestController
@RequestMapping(value = "/order/")
public class OrderController {

    @Reference(interfaceClass = OrderServiceAPI.class,
            check = false,
            group = "order2018")
    private OrderServiceAPI orderServiceAPI;

    @Reference(interfaceClass = OrderServiceAPI.class,
            check = false,
            group = "order2017")
    private OrderServiceAPI orderServiceAPI2017;
        
    @RequestMapping(value = "getOrderInfo",method = RequestMethod.POST)
    public ResponseVO getOrderInfo(
            @RequestParam(name = "nowPage",required = false,defaultValue = "1")Integer nowPage,
            @RequestParam(name = "pageSize",required = false,defaultValue = "5")Integer pageSize){

        // 获取当前登陆人的信息
        String userId = CurrentUser.getCurrentUser();

        // 使用当前登陆人获取已经购买的订单
        Page<OrderVO> page = new Page<>(nowPage,pageSize);

        if(userId != null && userId.trim().length() > 0){

            Page<OrderVO> result = orderServiceAPI.getOrderByUserId(Integer.parseInt(userId), page);
            Page<OrderVO> result2017 = orderServiceAPI2017.getOrderByUserId(Integer.parseInt(userId), page);
            // 合并结果
            int totalPages = (int)(result.getPages() + result2017.getPages());
            // 2017和2018的订单总数合并
            List<OrderVO> orderVOList = new ArrayList<>();
            orderVOList.addAll(result.getRecords());
            orderVOList.addAll(result2017.getRecords());

            return ResponseVO.success(nowPage,totalPages,"",orderVOList);
        }else{
            return ResponseVO.serviceFail("用户未登陆");
        }
    }
}
```



### 7.7  服务限流

**限流思路**

- 限流措施是系统高可用的一种手段；
- 使用并发与连接控制限流，但不常用；
- 使用漏桶法和令牌桶算法进行限流；

```shell
# 漏桶法和令牌桶法
# 令牌桶法：系统中有一个守护线程创建令牌，请求到来时拿令牌处理，若没有令牌则请求等待或者返回；
# 漏桶法处理请求的频率是固定的，令牌桶法对于业务峰值有一定的承载能力，比如令牌桶中一段时间后有1000个令牌，突然来1000个请求就都可以处理；
```

![image-20210103230717189](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210103230717189.png)

![image-20210103230735973](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210103230735973.png)

```java
// 令牌桶算法的实现
// 因为令牌桶对业务有一定的容忍度
public class TokenBucket {

    private int bucketNums=100;  // 桶的容量
    private int rate=1;          // 流入速度
    private int nowTokens;      //  当前令牌数量
    private long timestamp=getNowTime();     //  时间

    private long getNowTime(){
        return System.currentTimeMillis();
    }

    private int min(int tokens){
        if(bucketNums > tokens){
            return tokens;
        }else{
            return bucketNums;
        }
    }

    public boolean getToken(){
        // 记录来拿令牌的时间
        long nowTime = getNowTime();
        // 添加令牌【判断该有多少个令牌】
        nowTokens = nowTokens + (int)((nowTime - timestamp)*rate);
        // 添加以后的令牌数量与桶的容量那个小
        nowTokens = min(nowTokens);
        System.out.println("当前令牌数量"+nowTokens);
        // 修改拿令牌的时间
        timestamp = nowTime;
        // 判断令牌是否足够
        if(nowTokens < 1){
            return false;
        }else{
            nowTokens -= 1;
            return true;
        }
    }
}
```



### 7.8  Hystrix熔断降级

- 执行失败或者执行超时都Fallback()，出现业务异常则服务降级返回；
- 服务熔断主要是判断服务没有正常进行，然后执行fallback方法的过程；
- 服务熔断后会采取折中的方法返回，就是服务降级；
- Hystrix：信号量隔离        线程池隔离        线程切换

![image-20210103232124288](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210103232124288.png)

```xml
<!-- Hystrix添加依赖包 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
	<version>2.0.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
	<version>2.0.0.RELEASE</version>
</dependency>
```

```java
// 启动类上添加开启注解@EnableHystrixDashboard   @EnableCircuitBreaker   @EnableHystrix
@SpringBootApplication(scanBasePackages = {"com.stylefeng.guns"})
@EnableAsync
@EnableDubboConfiguration
@EnableHystrixDashboard
@EnableCircuitBreaker
@EnableHystrix
public class GatewayApplication {
    public static void main(String[] args) {

        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

```java
// 需要熔断的方法上添加：
@HystrixCommand(fallbackMethod = "error", commandProperties = {
@HystrixProperty(name="execution.isolation.strategy", value = "THREAD"),
@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value
= "4000"),
@HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
@HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = ”50")
}, threadPoolProperties = {
@HystrixProperty(name = "coreSize", value = "1"),
@HystrixProperty(name = "maxQueueSize", value = "10"),
@HystrixProperty(name = "keepAliveTimeMinutes", value = "1000"),
@HystrixProperty(name = "queueSizeRejectionThreshold", value = "8"),
@HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "12"),
@HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "1500")
})
@RequestMapping(value = "buyTickets",method = RequestMethod.POST)
public ResponseVO buyTickets(Integer fieldId, String soldSeats, String seatsName)
```

```java
// 创建方法，返回值和方法参数与需要熔断的方法相同；
// 业务降级的方法，熔断后提供的方法 fallback()
public ResponseVO error(Integer fieldId,String soldSeats,String seatsName){
        return ResponseVO.serviceFail("抱歉，下单的人太多了，请稍后重试");
}
// 熔断的方法
@RequestMapping(value = "buyTickets",method = RequestMethod.POST)
public ResponseVO buyTickets(Integer fieldId, String soldSeats, String seatsName)
```



### 7.9  业务总结

**学习的内容**

- 学习了限流措施以及实现方法，令牌桶法实际系统按照字节数给令牌个数根据业务字节数给相应数量的令牌；
- 掌握了 Dubbo 的分组聚合、版本控制，Mycat？？？中间件
- 留下一个思考，如何处理分布式事务？事务保证 ACID，分布式系统保证ACID ??



## 第八章： 支付模块开发*

### 章节概要

- 完成支付模块业务开发 -> 支付宝当面付
- Dubbo 特性学习：隐式参数、参数验证等
- 支付流程：订单获取支付二维码 -> 等待支付宝回调 -> 修改订单状态 -> 定期对账

![image-20210104091002164](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210104091002164.png)

![image-20210104093750715](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210104093750715.png)



### 8.1  Dubbo特性：本地存根

- 本地存根类似于 Dubbo 的静态代理；
- Dubbo 使用本地代理会在客户端生成一个代理，处理部分业务；
- Stub(本地存根) 必须有可传入 Proxy 的构造函数；

```shell
# 远程服务后，客户端通常值剩下接口，而实现全在服务端，但提供方有些时候想在客户端也执行部分逻辑，比如：做 ThreadLocal 缓存，提前验证参数，调用失败后伪造容错数据(服务降级)等等，此时就需要在 API中带上 Stub(本地存根)，客户端生成 Proxy 实例，会把 Proxy 通过构造函数传给 Stub，然后把 Stub 暴露给用户， Stub 可以决定要不要去掉 Proxy。
```

![image-20210104163944715](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210104163944715.png)

```java
public class MyStub implements ServiceAPI{
    // 注入 Proxy 的构造函数
    private final SerivceAPI serviceAPI;
    public Mystub(ServiceAPI serviceAPI){
        this.serviceAPI = serviceAPI;
    }    
}
```

**使用场景：**

```shell
举例：
客户端：
	guns-gateway
		-> AliPayServiceAPI
		用户登录验证、orderId的合法性、orderId不能为空
		ServiceAPIStub 在客户端验证，若不满足以上条件可以直接对请求处理，减少一次传输
服务端：
	guns-alipay
		-> AlipayServiceImpl
```



### 8.2 Dubbo特性：本地伪装

- 本地伪装是本地存根的一个子集；
- 通常会使用本地伪装处理服务降级，原因： Hystrix服务降级SpringBoot项目，Spring项目无法使用Hystrix就可以使用本地伪装降级；
- 针对接口的业务降级：`public class AliPayServiceMock implements AliPayServiceAPI`
- 本地伪装是本地存根Stub的一个子集，只能捕获  RPCException 后处理；

```java
/*
    业务降级方法
 */
public class AliPayServiceMock implements AliPayServiceAPI {
    @Override
    public AliPayInfoVO getQRCode(String orderId) {
        return null;
    }

    @Override
    public AliPayResultVO getOrderStatus(String orderId) {
        AliPayResultVO aliPayResultVO = new AliPayResultVO();
        aliPayResultVO.setOrderId(orderId);
        aliPayResultVO.setOrderStatus(0);
        aliPayResultVO.setOrderMsg("尚未支付成功");

        return aliPayResultVO;
    }
}

// 服务类配置
@Service(interfaceClass = AliPayServiceAPI.class,
        mock = "com.stylefeng.guns.api.alipay.AliPayServiceMock")
public class DefaultAlipayServiceImpl implements AliPayServiceAPI;
```



### 8.3  隐式参数传递

**隐式参数**

- Dubbo提供了参数的隐式传递
- Dubbo 的隐式参数仅单次调用可用

- 注意隐式参数的保留字段
- 分布式事务中RequestId可以作为隐式参数，在业务中一致存在；

![image-20210104172010820](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210104172010820.png)

```shell
# 可以通过 RpcContext 上的 setAttachment 和 getAttachment 在服务消费方和提供方之间进行参数的隐式传递;
# 在服务消费方端设置隐式参数
# setAttachment 设置的 KV 对，在完成下面一次远程调用会被清空，即多次远程调用要多次设置
# RpcContext.getContext().setAttachment("index", "1"); // 隐式传参，后面的远程调用都会隐式将这些参数发送到服务器端，类似cookie，用于框架集成，不建议常规业务使用
# xxxService.xxx(); // 远程调用
```



### 8.4  业务总结

**学习的内容**

- Dubbo 特性之隐式参数；
- Dubbo 特性之本地存根和本地伪装；
- Dubbo 特性本地伪装只能处理 RpcException，其余异常需要try catch 捕获；

- 支付宝当面付功能对接，支付宝回调服务器接口；



## 第九章： 分布式事务*

### 章节概要

- 事务简介
- 分布式事务的前世今生
- 分布式事务解决方案
- 主流分布式事务框架介绍



### 9.1  事务简介

**事务是用来保证一组数据操作的完整性和一致性**

- 事务必须满足 ACID 的四大特性
- 事务具有四种隔离级别
- 事务具有七种传播行为

```shell
# 事务属性
	原子性（Atomicity）：事务中不管有多少个操作，对于我们来说事务都是一个完整的不可分割的整体，所以对我们来说事务内部所有的数据操作都是一个原子，不可再被分割；
	一致性（Consistency）：事务要么全成功，要么全失败；
	隔离性（Isolation）：事务与事务之间隔离，不相见；
	持久性（Durability）：事务一旦被提交，对数据库中数据的改变就是持久性的，即便在数据库系统遇到故障的情况下也不会丢失提交事务的操作；
```



### 9.2  分布式事务

- **分布式事务就是将多个节点的事务看成一个整体处理**

- 分布式事务由事务参与者、资源服务器、事务管理器等组成
- 常见的分布式事务的例子：支付、下订单



分布式事务实现思路：

- 两段式事务：2PC

- 三段式事务：3PC

- 基于XA的分布式事务

- 基于消息的最终一致性方案
- TCC 编程式补偿性事务



#### 9.2.1  两段式和三段式事务介绍

- 三段式事务，事务管理器在预备和提交状态中间加了一个预状态；

![image-20210104195652925](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210104195652925.png)



#### 9.2.2  基于XA的分布式事务介绍

- x/open 机构
- 基于XA的分布式事务：MySQL，Oracle 数据库的事务模型近 XA 事务；

![image-20210104200416806](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210104200416806.png)



#### 9.2.3  基于消息的最终一致性方案介绍

目前的业务环境：下单支付，钱已经到支付宝，如果支付成功修改订单状态失败，没办法处理；

- 好处：强一致性方案；
- 缺点：支付业务会暂停，一直等待，造成时间和内存的浪费；

![image-20210104202520260](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210104202520260.png)



#### 9.2.4  TCC柔性补偿式事务(Try/Confirm/Cancel, 尝试执行/确认操作/取消操作)

- 事务协调器调用 Confirm / Cancel 接口；

![image-20210104202952102](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210104202952102.png)

```shell
# 基于消息最终一致性与 TCC 补偿性事务；
     基于消息事务是强一致事务，会存在资源浪费；
     TCC 事务是柔性事务，在 try 阶段要对资源做预留；
     TCC 事务在确认或取消阶段释放资源；
     与基于消息事务对比，TCC 的时效性更好；
CAP理论
```

**主流分布式事务框架介绍**

- 全局事务服务（Global Transaction  Service，简称 GTS）
- 蚂蚁金服分布式事务（Distributed Transaction-eXtended，简称DTX）
- 开源TCC框架（TCC-Transaction）



### 9.3  TCC-Transaction Dubbo案例部署解析

```shell
1、需要提供分布式事务支持的接口上添加@Compensable
2、在对应的接口实现上添加@Compensable，注解的方法是 Try 方法
3、在接口实现上添加confirmMethod、cancelMethod、transactionContextEditor(Dubbo隐式参数传递)
@Compensable(confirmMethod="", cancelMethod="", transactionContextEditor="")
4、实现对应的confirmMethod、cancelMethod
	注意： confirm方法和cancel方法必须与try方法在同一个类中(反射需要确保在一个类中)
5、主事务的业务都已经实现的差不多的时候才调用子事务
	子事务的调用在主事务的主方法调用的最后

注意：
	1、分布式事务里，不要轻易在业务层捕获所有异常	
		因为在不抛出异常的情况下，分布式事务无法确认事务失败，认为事务成功继续执行；
	2、使用TCC-Transaction时，confirm和cancel的幂等性需要自己代码支持

思考： 
	为什么要在confirm、cancel里检查订单状态，而不直接修改为结束状态
	因为confirm确认的就是刚刚try方法里新增的一个订单。
	
	-》 为了保证服务的幂等性
	
幂等性：使用相同参数对同一资源重复调用某个接口的结果与调用一次的结果相同(update/delete)
```

```shell
分布式事务的数据与业务的数据不要放在一个库中；
```

```java
// 仿猫眼系统中的订单业务背景
public interface ServiceAPI{
    
    @Compensable
    String sendMessage(String message);
    
    /*
    	背景：传入购票数量、传入购买座位、影厅编号
    	业务：
    		1.判断传入的座位是否存在
    		2.查询过往订单，判断座位是否已售
    		3.新增订单(主事务)
    	逻辑：
    		1.新增一条订单
    		2.判断座位是否存在 & 是否已售
    		3.任意一条为假，则修改订单为无效状态
    */
       
}
```



### 9.4  TCC-Transaction 框架

- 需要分布式事务的方法会被拦截器拦截，事务拦截器通过处理交给事务管理器，事务管理器将事务存入事务存储器，事务处理JOB对事务存储器中的事务进行处理；
- GLOBAL_TX_ID：总事务编号，全局事务编号；  BRANCH_qUALIFIER：分支事务编号，DOMAIN：事务标志、域(确定是哪个域)；VERSION：用来做乐观锁；
- 事务的相关信息（事务存储器）：全局事务编号、乐观锁版本等要持久化存储；
- 资源：TCC【Try-Confirm-Cancel】,try 核心点：预留业务资源，把事务数据资源存入库中；
- 注解一般有两种方式使用：通过反射读取、通过AOP拦截获取；
- Java里获取对应的唯一方式是全限定名  ->  packageName + className + methodName；
- 序列化一般有两种：JdkSerializationSerializer(JDK提供的序列化方法)、KryoPoolSerializer(esotericsoftware提供的序列化方法)

![image-20210105083605229](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105083605229.png)

![image-20210105092454985](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105092454985.png)

```shell
# 事务存储器(repository)，JdbcTransactionRepository，存储到数据库中；数据的存储和查询
# 事务的相关信息【全局事务编号、乐观锁版本等要持久化存储】
```

```shell
# 拦截器，CompensableTransactionInterceptor 和 ResourceCoordinatorInterceptor
	TCC【Try-Confirm-Cancel】
		try 核心点：预留业务资源，把事务数据资源存入库中；
		confirm：释放资源
		cancel：回滚资源
# CompensableTransactionInterceptor
	AOP配置：  CompensableTransactionAspect(@Aspect)
		      @Pointcut("@annotation(org.mengyun.tcctransaction.api.Compensable)")
		      @Around("compensableService()")	//调用前后都有方法执行
通过 AOP 有 @Compensable 注解的方法被封装为 ProceedingJoinPoint对象；
当前是否有一个事务存在，隔离性；
事务存放在ThreadLocal中，获取当前事务：private static final ThreadLocal<Deque<Transaction>> CURRENT = new ThreadLocal<Deque<Transaction>>();
判断当前事务角色， ROOT-> 主事务   Provider->事务参与者(分支事务)，MethodType对象；
	# ROOT 主事务执行流程：
		开启一个全新的事务：1、持久化事务形态 -> 全局事务编号；2、注册一个事务【Threadlocal】
		执行目标方法
		清除队列中的事务：使用ThreadLocal当前线程结束后，手动clear防止事务信息错乱；
		In拦截器：
            1、开启全局事务
            2、持久化全局事务
            3、注册全局事务
            4、判断是应该Confirm还是cancel，只调用本身的Confirm方法
            5、清除事务
        OUT拦截器：
    # provider 分支事务执行流程：
    	判断状态：TRYING初始化一份事务参与者的数据进入到当前服务中，注册事务；

# ResourceCoordinatorInterceptor
	使用事务管理器(TransactionManager)，传递一些事务相关的信息
	获取配置的CC方法名：compensable.confirmMethod()和compensable.cancelMethod()
	方法执行流程：反射机制获取目标对象、confirm方法的执行上下文对象，为了后续执行的人获取到对应的信息、cancel方法的执行上下文对象、事务对象、将所有的资源信息，交给了事务管理器
```

```shell
Participant 事务对象
事务处理器 -》 通过反射调用我们预先设置好的失败方法
事务处理器 -》 通过反射调用我们预先设置好的确认方法
transactionManager：
1、修改数据库状态；2、执行rollback方法、confirm方法（在CompensableTransactionInterceptor主方法的捕获异常之后的执行）；3、如果执行成功，则删除事务资源数据
# 不要忘了一点：分布式事务就是在捕获异常之后去执行的方法 Confirm/Cancel；
```

```shell
# 事务JOB：
按配置进行事务重试：数据库中 CONTEXT 的内容；
删除事务存储器中的内容；
```



### 9.4 总结

```shell
分布式事务：
	1、重点不是讲事务 -> 慕课网的其他课程【JDBC事务，Spring事务】
	2、事务如何做分布式


1、tcc-transaction-dubbo
	1.1 字节码代理 -> 创建接口的代理对象
	1.2 DubboTransactionContextEditor -> TRANSACTION_CONTEXT[标识事务状态]
				利用Dubbo的隐式参数来传递关键的非业务数据
				
				
2、tcc-transaction-spring
	封装了一些关键的Spring组件
	
3、问题：
	1、什么时候生成的TRANSACTION_CONTEXT隐式参数
	2、如何判断一个大的事务下，都有哪些小的事务
	3、为什么要有@Compensable注解
	4、两个拦截器都没有处理Confirm和Cancel

4、基础概念：
	主事务和分支事务【事务参与者】
	事务的相关信息【全局事务编号、乐观锁版本等要持久化存储】
	
5、事务拦截器作用：[Spring AOP的基本概念要熟练掌握]
	5.1 CompensableTransactionInterceptor
			5.1.1 将事务区分为Root事务和分支事务
			5.1.2 不断的修改数据库内的状态【初始化事务，修改事务状态】
			5.1.3 注册和清除事务管理器中队列内容
		
	5.2 ResourceCoordinatorInterceptor
			5.2.1 主要处理try阶段的事情
			5.2.2 在try阶段，就将所有的"资源"封装完成并交给事务管理器
			5.2.3 资源 -》 事务资源
							事务的参与者
									1、Confirm上下文
									2、Cancel上下文
									3、分支事务信息
			5.2.4 事务管理器修改数据库状态	
	5.3 调用目标对象 -> order red cap
	
6、小结：
	6.1 事务的相关信息【全局事务编号，乐观锁版本等要持久化存储】
	6.2 资源：*
				TCC 【try-confirm-cancel】
					try核心点： 预留业务资源
										  把事务数据资源存入库中
	6.3 流程：
			6.3.1 注册和初始化事务 -> 组织事务参与者 -> 执行目标try方法 -> 执行confirm和cancel方法
```

**重点内容**

- 熟悉TCC-Transaction的分布式事务处理流程
- TCC-Transaction 不能保证幂等性，@Transaction && 订单状态是草稿状态；
- TCC 分布式事务的核心是资源
- 资源：业务资源(Try阶段预留业务资源)、事务资源（分布式事务跨库保证分布式，Transaction理论上分布式系统的数据不能和业务的数据放在一起）；



## 第十章： 服务监控

### 章节概要

- 了解 Dubbo 监控相关内容；
- 熟练掌握 Dubbo-admin 使用；
- 熟练掌握链路监控，完成业务系统部署；



### 10.1 Dubbo特性：路由规则/配置规则

```shell
# 路由规则
# 路由规则决定一次 dubbo 服务调用的目标服务器，分为条件路由规则和脚本路由规则，并且支持可扩展；

# 写入路由规则
# 向注册中心写入路由规则的操作通常由监控中心或治理中心的页面完成
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
registry.register(URL.valueOf("condition://0.0.0.0/com.foo.BarService?category=routers&dynamic=false&rule=" + URL.encode("host = 10.20.153.10 => host = 10.20.153.11") + "));
	condition:// 表示路由规则的类型，支持条件路由规则和脚本路由规则，可扩展，
	0.0.0.0 表示对所有 IP 地址生效，如果只想对某个 IP 的生效，请填入具体 IP
	com.foo.BarService 表示只对指定服务生效
	rule=URL.encode("host = 10.20.153.10 => host = 10.20.153.11") 表示路由规则的内容
```

```shell
# 条件路由规则
基于表达式的路由规则：
	`=>` 之前的为消费者匹配条件，所有参数和消费者的URL进行对比，当消费者满足匹配条件时，对该消费者执行后面的过滤规则；
	`=>` 之后为提供者地址列表的过滤条件，所有参数和提供者的URL进行对比，消费者最终只拿到过滤之后的地址列表；
	若匹配条件为空，则表示对所有消费方应用；
	若过滤条件为空，表示禁止访问；
	注意：一个服务只能有一条白名单规则，否则两条规则交叉，就都被筛选掉了 
	举例：读写分离：
	 method = find*,list*,get*,is* => host = 172.22.3.94,172.22.3.95,172.22.3.96(读方法)
 	 method != find*,list*,get*,is* => host = 172.22.3.97,172.22.3.98(写方法)
```

```shell
# 配置规则
向注册中心写入动态配置覆盖规则。 该功能通常由监控中心或治理中心的页面完成。
```



### 10.2  链路监控

![image-20210105152949238](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105152949238.png)



#### 10.2.1  ZIPKIN(Docker部署)



![image-20210105153056847](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105153056847.png)

| 构成           | 作用                       |
| -------------- | -------------------------- |
| TraceID        | 全局唯一表示，单次请求唯一 |
| SpanID         | 调用编号，每次远程调用唯一 |
| ParentID       | 父请求 编号，上一级SpanID  |
| Client Start   | cs，表示客户端发起请求     |
| Server Receive | sr，表示服务端收到请求     |
| Server Send    | ss，表示服务端完成处理     |
| Client Receive | cr，表示客户端收到响应     |

**Transport：openzipkin/brave（GitHub）**

```shell
# Docker 部署zipkin
# Note: this is mirrored as ghcr.io/openzipkin/zipkin
docker run -d -p 9411:9411 openzipkin/zipkin
```



#### 10.2.2  Dubbo特性Filter

- Dubbo 支持 Filter 机制
- Dubbo 诸多工作都是通过 Filter 实现
- 常见自定义 Filter：日志记录、trace 功能等；



### 10.3  业务系统集成 Zipkin

- 业务最长链路： gateway   ->  alipay  ->   order  ->   cinema

```xml
<!-- SpringBoot 依赖 -->
<dependencyManagement>
    <dependcies>
        <dependency>
            <groupId>io.zipkin.brave</groupId>
            <artifactId>brave</artifactId>
            <version>5.10.1</version>
        </dependency>
       <dependency>
            <groupId>io.zipkin.brave</groupId>
            <artifactId>brave-bom</artifactId>
            <version>5.5.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>io.zipkin.reporter2</groupId>
            <artifactId>zipkin-reporter</artifactId>
            <version>2.7.9</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependcies>
</dependencyManagement>

<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-instrumentation-dubbo-rpc</artifactId>
    <version>5.9.5</version>
</dependency>
<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-spring-beans</artifactId>
    <version>5.13.1</version>
</dependency>
<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-context-slf4j</artifactId>
    <version>5.13.1</version>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-sender-okhttp3</artifactId>
    <version>2.16.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>4.0.1</version>
</dependency>
```





### 10.4  OpenResty

```shell
# OpenResty又被称为ngx_openresty，是基于Nginx的核心Web应用程序服务器。
# OpenResty是基于Nginx和Lua的高性能Web平台，OpenResty通过汇聚各种设计精良的Nginx模块，从而将Nginx有效地变成一个强大的通用Web应用平台。
# OpenResty的目标是让Web服务直接运行在Nginx服务内部，充分利用Nginx的非堵塞I/O模型，不仅对HTTP客户端请求，甚至对远程后端DB都进行一系列的高性能响应。
# OpenResty借助于Nginx的事件驱动模型和非堵塞IO，以实现高性能的Web应用程序。
# OpenResty使我们可以借助于Nginx的异步非阻塞达到使用Lua异步并发访问后端DB等服务。
# OpenResty使用ngx.location.capture_multi极大地减少浏览器的HTTP连接数量，可以异步并发的访问后台接口。


# OpenResty® 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。
# OpenResty® 通过汇聚各种设计精良的 Nginx 模块（主要由 OpenResty 团队自主开发），从而将 Nginx 有效地变成一个强大的通用 Web 应用平台。这样，Web 开发人员和系统工程师可以使用 Lua 脚本语言调动 Nginx 支持的各种 C 以及 Lua 模块，快速构造出足以胜任 10K 乃至 1000K 以上单机并发连接的高性能 Web 应用系统。
# OpenResty® 的目标是让你的Web服务直接跑在 Nginx 服务内部，充分利用 Nginx 的非阻塞 I/O 模型，不仅仅对 HTTP 客户端请求,甚至于对远程后端诸如 MySQL、PostgreSQL、Memcached 以及 Redis 等都进行一致的高性能响应。
```

- Docker部署zipkin，OpenResty反向代理，ZooKeeper注册中心；





## 第十一章： 微服务面试总结

### 章节概要

- 了解掌握Dubbo相关常见面试题
- 了解掌握微服务相关常见面试题
- Openresty：应用防火墙；
- 应用层服务结构，微服务：

![image-20210105180137789](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105180137789.png)

![image-20210105180538652](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105180538652.png)



### 11.1  Dubbo

- Dubbo 主要基于 Netty 和 Mina 构成，主要是 Netty
- Dubbo 中的 dubbo 协议是一种长连接形式(TCP,Sockt 编程)，
- dubbo 协议的底层是 TCP 协议
- dubbo 协议配置连接池参数是配置长连接数量





### 11.2  服务治理

- 服务治理是一个从 SOA 时代就诞生的大的概念；
- 微服务
- 服务治理贯穿于微服务体系的整个生命周期
- 服务治理包括但不限于服务注册、服务发现、服务监控、熔断降级等
- 服务总线：类似于注册中心，但是服务的注册和发现需要设置在服务总线上，服务总线太重；

```shell
Zookeeper 注册信息
2021-01-05 18:19:29,312 [myid:] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /0:0:0:0:0:0:0:1:7465
2021-01-05 18:19:29,329 [myid:] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@942] - Client attempting to establish new session at /0:0:0:0:0:0:0:1:7465
2021-01-05 18:19:29,332 [myid:] - INFO  [SyncThread:0:FileTxnLog@203] - Creating new log file: log.61e
2021-01-05 18:19:29,350 [myid:] - INFO  [SyncThread:0:ZooKeeperServer@687] - Established session 0x176d20f059d0000 with negotiated timeout 30000 for client /0:0:0:0:0:0:0:1:7465
2021-01-05 18:19:29,404 [myid:] - INFO  [ProcessThread(sid:0 cport:2181)::PrepRequestProcessor@648] - Got user-level KeeperException when processing sessionid:0x176d20f059d0000 type:create cxid:0x4 zxid:0x620 txntype:-1 reqpath:n/a Error Path:/dubbo/com.stylefeng.guns.api.user.UserAPI/configurators Error:KeeperErrorCode = NodeExists for /dubbo/com.stylefeng.guns.api.user.UserAPI/configurators
```



### 11.3  服务网关

- 并没有实现网关的所有功能，补网关功能：服务聚合、静态响应；
- dubbo服务网关路由、负载均衡？

![image-20210105182143165](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105182143165.png)



### 11.4 分布式事务

- 常见分布式事务有三种：基于消息，TCC 和 2PC/3PC；
- 分布式事务强调的是如何解决分布式
- TCC柔性补偿效率相对较高，完全异步过程，没有等待过程；
- TCC 事务的核心思想是：资源，try 阶段预留业务资源占用库存 confirm/cancel；
- 通过状态实现幂等性；
- 如果对资源的掌控力度不够，则不应该使用TCC事务；





### 11.5 服务幂等性

- 幂等性：多次重复请求的结果始终如一
- 幂等性问题主要产生于增加和修改操作上
- 幂等性的常见解决方案：分布式锁和状态表示
- 分布式锁：唯一性表示记录，可以用 Redis 操作锁；
- 服务尽量无状态
- 分布式锁处理幂等性问题：

![image-20210105194119989](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105194119989.png)



### 11.6  限流方案

- 限流算法：令牌桶法、漏桶法
- 令牌桶法：Token 一般表示流量，不同业务的Token一般不同；
- 分布式系统中限流一般都是多层次的；
- 限流措施距离请求入口越近效果越好；
- 掌握令牌桶和漏桶法的区别，一段时间没人访问令牌桶会存一些令牌，可以接受突然的大流量(峰值)；漏桶法限流的速度时一样的；





### 11.7  自动化运维与部署

- 微服务对自动化的要求度非常的高；
- 推荐套装：Docker + kubernetes + 镜像仓库
- 监控工具推荐：Zipkin、falcon(小米运维推出)



**微服务的优劣势**

- 运维成本增加；





## 项目学到的知识点

- **所有需要交互的Model必须实现Serializable接口，可序列化之后才可以传输**
- **字典表**
- **公认的API网关的功能：路由转发和服务聚合**
- **一对多的查询(Mybatis)**
- **开源TCC框架（TCC-Transaction）**
- **分布式事务的资源：业务资源(Try阶段预留业务资源)、事务资源（分布式事务跨库保证分布式，Transaction理论上分布式系统的数据不能和业务的数据放在一起）**





## 项目可能的面试问题

- Dubbo框架的底层协议：TCP/HTTP协议等的实现都有；
- dubbo协议是TCP，SpringCloud底层是HTTP协议，所以Dubbo比Cloud快-> TCP比HTTP快；

![image-20201230113942101](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201230113942101.png)









