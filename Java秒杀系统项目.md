## Java高并发商城秒杀项目(2020.12.2-)

- public()方法首先要做参数校验；

## 课程介绍(Redis贯穿其中)

![课程介绍](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201129221617440.png)

![秒杀](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201129221755721.png)

- 如何应对大并发：如何利用缓存；如何使用异步；如何编写优雅的代码；

- 课程目标：秒杀核心技术；



## 1、项目框架搭建

```
1.Spring Boot环境搭建；项目使用SpringBoot版本：1.5.8.RELEASE
2.集成Thymeleaf， Result结果封装；
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
    <artifactId>spring-boot-starter-web</artifactId>
    <artifactId>lombok</artifactId>
3.集成MyBatis + Druid；
4.集成Jedis + Redis安装 + 通用缓存Key封装；
```

```
controller中的方法分为两类：
	1、rest api  json输出；
		接口输出结果的封装：
		{
			"code":500100;
			"msg":库存不足
			"data":{}[]
		}
	2、页面
```



### 1.1  RestFul类型的返回结果Result的封装

> 之所以需要对结果进行固定格式的封装，是为了让前端更好的接收和处理结果，对数据进行展示；

```java
//接口输出结果：字符串封装
//数据类型未知，用泛型T表示
@Data
public class Result<T> {

    private int code;
    private String msg;
    private T data;

    //构造方法私有
    private Result(T data) {
        this.code = 0;
        this.msg = "success";
        this.data = data;
    }

    private Result(CodeMsg cm){
        if(cm == null)
            return;
        this.code = cm.getCode();
        this.msg = cm.getMsg();
    }

    /**
     * 成功时候的调用
     * 成功的时候传入data
     * public static <T> Result<T> success(T data)---> static后面的<T>给方法指定泛型
     */
    public static <T> Result<T> success(T data){
        return new Result<>(data);
    }

    /**
     * 失败时候的调用，传入Code和msg，组装为CodeMsg对象
     */
    public static <T> Result<T> error(CodeMsg cm){
        return new Result<>(cm);
    }

}

//CodeMsg中定义错误码对应的含义，在controller发生错误时，将对应的信息传入Result类中；
//Result类构造方法中有传入CodeMsg类；
@Data
public class CodeMsg {
    private int code;
    private String msg;

    //在CodeMsg类中封装错误码对应的含义
    public static CodeMsg SUCCESS = new CodeMsg(0,"success");
    public static CodeMsg  SERVER_ERROR = new CodeMsg(500100,"服务端异常");
    //登录模块  5002XX

    //商品模块  5003XX

    //订单模块  5004XX

    //秒杀模块  5005XX

    private CodeMsg(int code, String msg){this.code = code;this.msg = msg;}

}
```



### 1.2  集成Mybatis

```
<!-- 引入依赖 -->
<dependency>
     <groupId>org.mybatis.spring.boot</groupId>
     <artifactId>mybatis-spring-boot-starter</artifactId>
     <version>2.1.1</version>
</dependency>
<dependency>
     <groupId>com.alibaba</groupId>
     <artifactId>druid-spring-boot-starter</artifactId>
     <version>1.1.10</version>
</dependency>

<!--   数据库驱动  -->
<dependency>
     <groupId>mysql</groupId>
     <artifactId>mysql-connector-java</artifactId>
     <version>8.0.20</version>
</dependency>
```



```java
//package: cn.seu.pojo
//实体类
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {

    private int id;
    private String name;

}

//package: cn.seu.dao
//dao层，@Mapper注解
@Mapper
@Repository
public interface UserDao {

    User getUserByID(@Param("id") int id);

    int insert(User user);

}

//package: cn.seu.service
//service层
@Service
public class UserService {

    @Autowired
    UserDao userDao;

    public User getUserByID(int id){
        return userDao.getUserByID(id);
    }

    //启动事务标签
    //@Transactional
    public boolean tx(){
        User u1 = new User(2,"2222");
        userDao.insert(u1);
        User u2 = new User(1,"11111");
        userDao.insert(u2);
        return true;
    }
}
```



```xml
<!-- Mybatis配置文件,
位置：resources/mybaits.mapper
名称：UserDdao.xml
数据库操作语句
-->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.seu.dao.UserDao">
    <select id="getUserByID" resultType="cn.seu.pojo.User">
        select * from user where id = #{id}
    </select>
    <insert id="insert" parameterType="cn.seu.pojo.User">
         insert into user(id,name) values (#{id}, #{name})
    </insert>
</mapper>
```



### 1.3 集成Redis

- 关键：Redis的incr()和decr()方法是原子操作；不可分割；

```xml
<!-- 引入依赖 -->
<dependency>
     <groupId>com.alibaba</groupId>
     <artifactId>fastjson</artifactId>
     <version>1.2.62</version>
</dependency>
<dependency>
     <groupId>redis.clients</groupId>
     <artifactId>jedis</artifactId>
     <version>3.3.0</version>
</dependency>
```

```
避免key被修改，给key设置前缀，前缀+key=Redis中真实的key；
每个模块的prefix(Redis 中key 的前缀)不同：每个模块对应不同的类，类名不同，
通用缓存Key封装(模板模式)：
	接 口(KeyPrefix)
	  |
	抽象类(BasePrefix)
	  |
	实现类(UserKey,OrderKey)
```



```java
//package:cn.seu.redis
//作用：读取application.properties文件中定义的Redis设置，存储到相应的变量中
@Data
@Component
@ConfigurationProperties(prefix = "spring.redis")
public class RedisConfig {

    private String host;
    private int port;
    private int timeout;//秒
    private String password;
    private int jedisPoolMaxActive;
    private int jedisPoolMaxIdle;
    private int jedisPoolMaxWait;//秒

}
```



```java
//package:cn.seu.redis
//作用：利用RedisConfig类对象@Autowired，创建JedisPool对象@Bean注解，
@Service
public class RedisPoolFactory {

    @Autowired
    RedisConfig redisConfig;

    @Bean
    public JedisPool JedisPoolFactory() {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxIdle(redisConfig.getJedisPoolMaxIdle());
        poolConfig.setMaxTotal(redisConfig.getJedisPoolMaxActive());
        poolConfig.setMaxWaitMillis(redisConfig.getJedisPoolMaxWait() * 1000);
        JedisPool jp = new JedisPool(poolConfig, redisConfig.getHost(), redisConfig.getPort(),
                redisConfig.getTimeout()*1000, redisConfig.getPassword(), 0);
        return jp;
    }

}
```



```java
//package:cn.seu.redis
//作用：真正的Redis服务，包括：set(),get(),exists(),incr(),decr()方法，对Redis数据库的操作
@Service
public class RedisService {

    @Autowired
    JedisPool jedisPool;

    //Redis对应的 get方法
    //获得对象
    public <T> T get(KeyPrefix prefix, String key, Class<T> clazz){
        Jedis jedis = null;
        try {
            jedis = jedisPool.getResource();
            //生成真正的key
            String realKey = prefix.getPrefix() + key;
            String str = jedis.get(realKey);
            T t = stringToBean(str, clazz);
            return t;
        }finally {
            returnToPool(jedis);
        }

    }

    //Redis对应的 set方法
    //设置对象
    public <T> boolean set(KeyPrefix prefix, String key, T value){
        Jedis jedis = null;
        try {
            jedis = jedisPool.getResource();
            String str = beanToString(value);
            if(str == null ||str.length() <= 0)
                return false;
            //生成真正的key
            String realKey = prefix.getPrefix() + key;
            int second = prefix.expireSeconds();
            //second<=0表示，永不过期
            if(second <= 0){
                jedis.set(realKey, str);
            }else{
                jedis.setex(realKey,second,str);
            }

            return true;
        }finally {
            returnToPool(jedis);
        }

    }

    //Redis对应的 exists方法
    //判断是否存在
    public <T> boolean exists(KeyPrefix prefix, String key, T value){
        Jedis jedis = null;
        try {
            jedis = jedisPool.getResource();
            String str = beanToString(value);
            if(str == null ||str.length() <= 0)
                return false;
            //生成真正的key
            String realKey = prefix.getPrefix() + key;
            jedis.exists(realKey);
            return true;
        }finally {
            returnToPool(jedis);
        }
    }

    //Redis对应的 key自增方法
    //key增加值
    public <T> Long incr(KeyPrefix prefix, String key){
        Jedis jedis = null;
        try {
            jedis = jedisPool.getResource();
            //生成真正的key
            String realKey = prefix.getPrefix() + key;
            return jedis.incr(realKey);
        }finally {
            returnToPool(jedis);
        }
    }

    //Redis对应的 key自减方法
    //key减少值
    public <T> Long decr(KeyPrefix prefix, String key){
        Jedis jedis = null;
        try {
            jedis = jedisPool.getResource();
            //生成真正的key
            String realKey = prefix.getPrefix() + key;
            return jedis.decr(realKey);
        }finally {
            returnToPool(jedis);
        }
    }


    private <T> String beanToString(T value) {
        if(value == null){
            return null;
        }
        Class<?> clazz = value.getClass();
        if(clazz == int.class || clazz == Integer.class){
            return "" + value;
        }else if(clazz == String.class){
            return (String)value;
        }else if(clazz == long.class ||clazz == Long.class){
            return "" + value;
        }else{
            return JSON.toJSONString(value);
        }
    }

    //字符串转换为bean对象
    private <T> T stringToBean(String str, Class<T> clazz) {
        if(str == null || str.length() <= 0 || clazz == null)
            return null;
        if(clazz == int.class || clazz == Integer.class){
            return (T)Integer.valueOf(str);
        }else if(clazz == String.class){
            return (T)str;
        }else if(clazz == long.class ||clazz == Long.class){
            return (T)Long.valueOf(str);
        }else{
            return JSON.toJavaObject(JSON.parseObject(str),clazz);
        }
    }

    private void returnToPool(Jedis jedis) {
        if(jedis != null){
            jedis.close();
        }
    }


}
```



```java
//package:cn.seu.redis
//作用：将user和Order在Redis中的key区分开，利用模板设计模式
/**
通用缓存Key封装(模板模式)：
	接 口(KeyPrefix)
	  |
	抽象类(BasePrefix)
	  |
	实现类(UserKey,OrderKey)
*/
public interface KeyPrefix {

    //过期时间
    public int expireSeconds();
    //获得前缀
    public String getPrefix();

}

public abstract class BasePrefix implements KeyPrefix {

    private int expireSecond;
    private String prefix;

    //0代表永不过期，默认key永不过期
    public BasePrefix(String prefix){
        this(0,prefix);
    }

    //构造器传值
    public BasePrefix(int expireSecond, String prefix){
        this.expireSecond = expireSecond;
        this.prefix = prefix;
    }

    //默认0代表永不过期
    @Override
    public int expireSeconds() {
        return expireSecond;
    }

    @Override
    public String getPrefix() {
        String className = getClass().getSimpleName();//获得实际类的类名, getClass() 返回Object 的运行时类
        return className+":"+prefix;
    }
}

public class UserKey extends BasePrefix {


    private UserKey(String prefix) {
        super(prefix);
    }

    public static UserKey getById = new UserKey("id");
    public static UserKey getByName = new UserKey("name");

}

public class OrderKey extends BasePrefix {

    public OrderKey(int expireSecond, String prefix) {
        super(expireSecond, prefix);
    }

}
```



### 1.4 SpringBoot配置文件(application.properties)

```properties
# mybatis
mybatis.type-aliases-package=cn.seu.pojo
#下划线转为驼峰规则设置为true
mybatis.configuration.map-underscore-to-camel-case=true 
mybatis.configuration.default-fetch-size=100
mybatis.configuration.default-statement-timeout=3000
mybatis.mapperLocations = classpath:mybatis/mapper/*.xml

# druid
spring.datasource.url=jdbc:mysql://localhost:3306/secondkill?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8
spring.datasource.username=root
spring.datasource.password=123
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.druid.filters=stat
spring.datasource.druid.max-active=2
spring.datasource.druid.initial-size=1
spring.datasource.druid.max-wait=60000
spring.datasource.druid.min-idle=1
spring.datasource.druid.time-between-eviction-runs-millis=60000
spring.datasource.druid.min-evictable-idle-time-millis=300000
spring.datasource.druid.validation-query=select 'x'
spring.datasource.druid.test-while-idle=true
spring.datasource.druid.test-on-borrow=false
spring.datasource.druid.test-on-return=false
spring.datasource.druid.pool-prepared-statements=true
spring.datasource.druid.max-open-prepared-statements=20

#redis
spring.redis.host=118.31.247.75
spring.redis.port=6380
spring.redis.timeout=3
spring.redis.password=SEU@REDIS#lileireids/123
#spring.redis.jedis.pool.max-active=10
#spring.redis.jedis.pool.max-idle=10
#spring.redis.jedis.pool.max-wait=3
spring.redis.jedisPoolMaxActive=10
spring.redis.jedisPoolMaxIdle=10
spring.redis.jedisPoolMaxWait=3
```



## 2、实现用户登录以及分布式session功能

### 2.1  数据库设计

```sql
CREATE TABLE miaosha_user(
id bigint(20) NOT NULL COMMENT'用户ID,手机号码',
nickname varchar(255) NOT NULL,
password varchar(32) DEFAULT NULL COMMENT 'MD5(MD5(pass明文+固定salt))',
salt varchar(10) DEFAULT NULL,
head varchar(128) DEFAULT NULL COMMENT '头像，云存储的ID',
register_date datetime DEFAULT NULL COMMENT '注册时间',
last_login_date datetime DEFAULT NULL COMMENT '上次登录时间',
login_count int(11) DEFAULT '0' COMMENT '登录次数',
PRIMARY KEY(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```



### 2.1 两次MD5

- 1.用户端：PASS = MD5( 明文 + 固定Salt)，添加盐是为了防止被破解密码
- 2.服务端：PASS = MD5(用户输入 + 随机Salt)
- private  static final String salt = "1a2b3c4d";

```xml
<!-- 引入依赖 -->
<!--   MD5依赖   -->
        <dependency>
            <groupId>commons-codec</groupId>
            <artifactId>commons-codec</artifactId>
<!--            <version>1.10</version>-->
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.6</version>
        </dependency>
```



### 2.2 登录功能实现

```
Thymeleaf引入静态文件的方式：@{/../....js}
/代表的路径为：resources/static/...

```

> MD5从前端页面传入时要做一次MD5加密，从前端页面传来的FormPassWord通过后台在做一次MD5加密，与最终在数据库中存储的密码做比较；
>
> 注意：在数据库中保存了MD5的盐；



### 2.3  jsr303参数校验

- SpringBoot中不添加依赖版本号默认使用SpringBoot自己定义的版本；
- 自定义校验器可以减少代码中对前端传来的参数是否符合规范做校验；

```java
<!-- 依赖，SpringBoot2.3.5.RELEASE默认的jsr303校验版本为2.3.5.RELEASE -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
    <version>2.2.8.RELEASE</version>
</dependency>
```

```java
//自定义验证器(注解)
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint(validatedBy = {IsMobileValidator.class})
public @interface IsMobile {

    boolean required() default true;

    String message() default "手机号码格式错误";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

}

public class IsMobileValidator implements ConstraintValidator<IsMobile, String> {

    private boolean required = false;

    //初始化方法
    @Override
    public void initialize(IsMobile constraintAnnotation) {
        required = constraintAnnotation.required();
    }
	//判断手机号是否有效
    @Override
	public boolean isValid(String s, ConstraintValidatorContext constraintValidatorContext) {
        if(required){
            return ValidatorUtil.isMobile(s);
        }else{
            if(StringUtils.isEmpty(s)){
                return true;
            }else{
                return ValidatorUtil.isMobile(s);
            }
        }
    }

}

//ValidatorUtil是定义的判断手机号码格式的类
//判断手机号码是否符合格式
public class ValidatorUtil {

    private static final Pattern mobile_pattern = Pattern.compile("1\\d{10}");

    public static boolean isMobile(String src){
        if(StringUtils.isEmpty(src)){
            return false;
        }
        Matcher m = mobile_pattern.matcher(src);
        return m.matches();
    }
    
}
```

- 自定义校验器@IsMobile：
- @Constraint(validatedBy = {IsMobileValidator.class})，自定义校验器IsMobile需要实现类IsMobileValidator定义验证方法及验证逻辑；
- IsMobileValidator需要实现ConstraintValidator接口，重写initialize()初始化方法和isValid()判断是否有效的方法；



### 2.4  全局异常处理(简化代码)

- 全局异常处理之后，代码中出现错误的地方直接全部抛出异常，可以减少在代码中对异常情况的处理；
- 定义全局异常处理器GlobalExceptionHandler，拦截所有的异常并进行处理；
- 定义全局异常GlobalException，对属于CodeMsg中出现的异常抛出；
- 在MiaoShaUserService类中方法出现的异常直接抛出，全局异常处理器接收异常并做处理；

```java
//全局的异常处理器
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler {
    //拦截所有异常
    @ExceptionHandler(value=Exception.class)
    public Result<String> exceptionHandler(HttpServletRequest request, Exception e){
        if(e instanceof GlobalException){
            GlobalException ex = (GlobalException)e;
            return Result.error(ex.getCm());
        }else if(e instanceof BindException){
            BindException ex = (BindException)e;
            List<ObjectError> errors = ex.getAllErrors();
            ObjectError error = errors.get(0);
            String msg = error.getDefaultMessage();
            return Result.error(CodeMsg.BIND_ERROR.fillArgs(msg));
        }else{
            return Result.error(CodeMsg.SERVER_ERROR);
        }
    }

}

public class GlobalException extends RuntimeException {
    private static final long serialVersionUID = 1L;
    private CodeMsg cm;

    public GlobalException(CodeMsg cm) {
        super(cm.toString());
        this.cm = cm;
    }

    public CodeMsg getCm() {return cm;}
}

@Service
public class MiaoshaUserService {
    
    public boolean login(LoginVo loginVo) {
        if(loginVo == null){
            throw new GlobalException(CodeMsg.SERVER_ERROR);
        }
        String mobile = loginVo.getMobile();
        String formPass = loginVo.getPassword();
        //判断手机号是否存在
        MiaoshaUser user = getById(Long.parseLong(mobile));
        if(user == null){
            throw new GlobalException(CodeMsg.MOBILE_NOT_EXIST);
        }
        //验证密码
        String dbPass = user.getPassword();//数据库中保存的密码
        String saltDB = user.getSalt();
        String calcPass = MD5Util.formPassToDBPass(formPass, saltDB);
        if(!dbPass.equals(calcPass)){
            throw new GlobalException(CodeMsg.PASSWORD_ERROR);
        }
        return true;
    }
    
}
```



### 2.5  分布式session-上(Redis管理Session)

- UUID 是 通用唯一识别码（Universally Unique Identifier）的缩写；

> 为什么要使用分布式session：
>
> 实际使用中，有多台服务器不会是一台服务器，这时候就涉及用户session的处理，如果只使用服务器提供的原生的session，当用户请求另一台服务器的时候，用户session信息就丢失了；
>
> 实现分布式session：
>
> 用户登录成功后给用户生成类似于sessionID的信息(实际编程中为token，使用Java提供的UUID创建随机token),写到cookie中传递给客户端，客户端在随后的访问中在cookie中上传token，服务端从token中取到用户信息；
>
> 用户信息写到Redis中；
>
> 手机客户端的token一般通过参数上传，所以在controller中设置了两种情况；



```java
@Service
public class MiaoshaUserService {
public boolean login(HttpServletResponse response,LoginVo loginVo) {
        if(loginVo == null){
            throw new GlobalException(CodeMsg.SERVER_ERROR);
        }
        String mobile = loginVo.getMobile();
        String formPass = loginVo.getPassword();
        //判断手机号是否存在
        MiaoshaUser user = getById(Long.parseLong(mobile));
        if(user == null){
            throw new GlobalException(CodeMsg.MOBILE_NOT_EXIST);
        }
        //验证密码
        String dbPass = user.getPassword();//数据库中保存的密码
        String saltDB = user.getSalt();
        String calcPass = MD5Util.formPassToDBPass(formPass, saltDB);
        if(!dbPass.equals(calcPass)){
            throw new GlobalException(CodeMsg.PASSWORD_ERROR);
        }
    	//登录成功后设置cookie
        //生成cookie,随机生成token存储到cookie中，作为用户唯一标识
        //用户信息写入Redis中
        String token = UUIDUtil.uuid();//随机生成token
        redisService.set(MiaoshaUserKey.token,token,user);
    	//public Cookie(String name, String value){},cookie构造方法
        Cookie cookie = new Cookie(COOKI_NAME_TOKEN, token);
        cookie.setMaxAge(MiaoshaUserKey.token.expireSeconds());//设置cookie的过期时间
        cookie.setPath("/");//根目录
        response.addCookie(cookie);//cookie写入客户端
        return true;
    }
}

//秒杀user在Redis中key的前缀
public class MiaoshaUserKey extends BasePrefix{
    public static final int TOKEN_EXPIRE = 3600*24*2;
    private MiaoshaUserKey(int expireSeconds, String prefix) {
        super(expireSeconds, prefix);
    }
    public static MiaoshaUserKey token = new MiaoshaUserKey(TOKEN_EXPIRE, "tk");
}

//Controller请求页面
@Controller
@RequestMapping("/goods")
public class GoodsController {

    @Autowired
    MiaoshaUserService userService;

    @Autowired
    RedisService redisService;

    @RequestMapping("/to_list")
    public String list(Model model,
@CookieValue(value = MiaoshaUserService.COOKI_NAME_TOKEN, required = false)String cookieToken,
@RequestParam(value=MiaoshaUserService.COOKI_NAME_TOKEN, required = false)String paramToken) {
        if(StringUtils.isEmpty(cookieToken) && StringUtils.isEmpty(paramToken)){
            return "login";
        }
        String token = StringUtils.isEmpty(paramToken)?cookieToken:paramToken;
        MiaoshaUser user = userService.getByToken(token);
        model.addAttribute("user", user);
        return "goods_list";
    }

}
```



![image-20201204164629075](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201204164629075.png)

![image-20201204164727741](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201204164727741.png)

![Redis的key](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201204171023062.png)



### 2.6 分布式session-下(简化代码)

- 延长有效期：获取user的时候，重新设置缓存，生成新的cookie即达到延长有效期的目的；

```java
@Service
public class MiaoshaUserService {
    public static final String COOKI_NAME_TOKEN = "token";

    @Autowired
    MiaoshaUserDao miaoshaUserDao;
    @Autowired
    RedisService redisService;

    public MiaoshaUser getById(long id) {return miaoshaUserDao.getById(id);}

    public MiaoshaUser getByToken(HttpServletResponse response, String token) {
        if(StringUtils.isEmpty(token)){
            return null;
        }
        MiaoshaUser user =  redisService.get(MiaoshaUserKey.token,token,MiaoshaUser.class);
        //延长有效期
        if(user != null){
            addCookie(response,token,user);
        }
        return user;
    }

    public boolean login(HttpServletResponse response,LoginVo loginVo) {
        if(loginVo == null){
            throw new GlobalException(CodeMsg.SERVER_ERROR);
        }
        String mobile = loginVo.getMobile();
        String formPass = loginVo.getPassword();
        //判断手机号是否存在
        MiaoshaUser user = getById(Long.parseLong(mobile));
        if(user == null){
            throw new GlobalException(CodeMsg.MOBILE_NOT_EXIST);
        }
        //验证密码
        String dbPass = user.getPassword();//数据库中保存的密码
        String saltDB = user.getSalt();
        String calcPass = MD5Util.formPassToDBPass(formPass, saltDB);
        if(!dbPass.equals(calcPass)){
            throw new GlobalException(CodeMsg.PASSWORD_ERROR);
        }
        //生成cookie
        String token = UUIDUtil.uuid();
        addCookie(response,token,user);
        return true;
    }

    private void addCookie(HttpServletResponse response,String token, MiaoshaUser user){
        //生成cookie,随机生成token存储到cookie中，作为用户唯一标识
        //用户信息写入Redis中
        redisService.set(MiaoshaUserKey.token,token,user);
        Cookie cookie = new Cookie(COOKI_NAME_TOKEN, token);
        cookie.setMaxAge(MiaoshaUserKey.token.expireSeconds());
        cookie.setPath("/");//根目录
        response.addCookie(cookie);//cookie写入客户端
    }

}
```

- 代码简化：在控制器中对于token需要进行判断即对用户是否存在进行判断；重写SpringMVC中的类；
- Springboot2.0以后用WebMvcConfigurationSupport代替WebMvcConfigurationAdapter；

> 简化原因：
>
> 在之前实现分布式session时，对于一个请求需要传入多个方法并进行判断；多个方法都需要用户信息时判断代码重复使用，所以要简化代码，直接注入user对象；
>
> @RequestMapping("/to_list")
>     public String list(HttpServletResponse response,Model model,
> @CookieValue(value = MiaoshaUserService.COOKI_NAME_TOKEN, required = false)String cookieToken,
> @RequestParam(value=MiaoshaUserService.COOKI_NAME_TOKEN, required = false)String paramToken){...}；
>
> 简化之后Controller中的代码：
>
> ```java
> @RequestMapping("/to_list")
> public String list(Model model, MiaoshaUser user) {
>         System.out.println(user);
>         return "goods_list";
> }
> ```

**简化过程：**

```
SpringMVC的控制器controller会回调WebMvcConfigurationSupport类中的addArgumentResolvers()方法为控制器中的参数赋值，简化思路：遍历方法的参数，是否存在MiaoshaUser类，如果存在则通过代码给MiaoshaUser赋对象；
在argumentResolvers中添加自定义的ArgumentResolver(参数解析器)，解析方法参数是否存在MiaoshaUser以及返回一个MiaoshaUser对象；只要方法参数中存在MiaoshaUser，则调用自定义ArgumentResolver中的resolveArgument()方法返回MIaoshaUser对象；
```

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {
	//自定义ArgumentResolver注册
    @Autowired
    UserArgumentResolver userArgumentResolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(userArgumentResolver);
    }

}

//自定义参数解析器
@Service
public class UserArgumentResolver implements HandlerMethodArgumentResolver {

    @Autowired
    MiaoshaUserService userService;
	
    //遇到MiaoshaUser.class，则做处理返回MiaoshaUser对象
    @Override
    public boolean supportsParameter(MethodParameter methodParameter) {
        Class<?> clazz = methodParameter.getParameterType();
        return clazz == MiaoshaUser.class;
    }
    
	//处理逻辑
    @Override
    public Object resolveArgument(MethodParameter methodParameter, ModelAndViewContainer modelAndViewContainer,
                                  NativeWebRequest nativeWebRequest, WebDataBinderFactory webDataBinderFactory) throws Exception {
        HttpServletRequest request = nativeWebRequest.getNativeRequest(HttpServletRequest.class);
        HttpServletResponse response = nativeWebRequest.getNativeResponse(HttpServletResponse.class);
        String paramToken = request.getParameter(MiaoshaUserService.COOKI_NAME_TOKEN);
        String cookieToken = getCookieValue(request, MiaoshaUserService.COOKI_NAME_TOKEN);
        if(StringUtils.isEmpty(cookieToken) && StringUtils.isEmpty(paramToken))
            return null;
        String token = StringUtils.isEmpty(paramToken)?cookieToken:paramToken;
        return userService.getByToken(response,token);
    }
    
    //遍历cookie数组，找到需要的cookie
    private String getCookieValue(HttpServletRequest request, String cookieNmae){
        Cookie[] cookies = request.getCookies();
        for(Cookie cookie: cookies){
            if(cookie.getName().equals(cookieNmae))
                return cookie.getValue();
        }
        return null;
    }

}
```



## 3、秒杀功能开发及管理后台

```
数据库设计：
    商品表 ---->   秒杀商品表
    订单表 ---->   秒杀订单表
商品列表页：
	查询所有商品信息需要查询商品信息 + 秒杀信息 --->  创建GoodsVo类，GoodsVo继承Goods类，同时包含秒杀商品的信息，将查询得到的GoodsVo类展示在页面上；
GoodsVo包含商品信息和商品秒杀信息；
页面倒计时需要在客户端做，不能在服务端，因为访问量很大，但是在客户端倒计时存在计时不精确的问题；
设置秒杀时间的时候，发现从数据库中读取出来的时间与本机时间不一致
	解决方法：在访问数据库的URL中设置时区为亚洲上海Asia/Shanghai
	spring.datasource.url=jdbc:mysql://localhost:3306/secondkill?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf-8
```



### 3.1  秒杀功能的实现

```
用户点击秒杀按钮，实际上是form表单的提交，传入goodsId参数；
MiaoshaController的逻辑：
	1.判断是否有库存，没有库存返回错误页面；
	2.判断是不是同一用户重复操作(userId && goodsId)，已经秒杀过返回错误信息；
	3.执行秒杀操作，事务：减库存、下订单、写入秒杀订单；
```

```java
//秒杀控制器：
//GoodsService通过传入的goodsId判断goods商品是否有库存；
//orderService通过userId和goodsId判断该用户是否已经秒杀成功；
//MiaoshaService执行秒杀操作(作为事务)
@Controller
@RequestMapping("/miaosha")
public class MiaoshaController {

    @Autowired
    GoodsService goodsService;

    @Autowired
    OrderService orderService;

    @Autowired
    MiaoshaService miaoshaService;

    @RequestMapping("/do_miaosha")
    public String list(Model model, MiaoshaUser user,
                       @RequestParam("goodsId")long goodsId){
        model.addAttribute("user",user);
        if(user == null)
            return "login";
        //1.判断库存
        GoodsVo goods = goodsService.getGoodsVoByGoodsId(goodsId);
        Integer stockCount = goods.getStockCount();
        if(stockCount <= 0){
            model.addAttribute("errmsg", CodeMsg.MIAO_SHA_OVER.getMsg());
            return "miaosha_fail";
        }
        //2.有库存，判断秒杀成功，防止一人秒杀多个商品，判断是否重复下单
        MiaoshaOrder order = orderService.getMiaoshaOrderByUserIdGoodsId(user.getId(),goodsId);
        if(order != null){
            model.addAttribute("errmsg", CodeMsg.REPEATE_MIAOSHA.getMsg());
            return "miaosha_fail";
        }
        //3. 有库存且用户没有秒杀过，开始秒杀操作(事务)
        //减库存 下订单 写入秒杀订单
        //传入参数：用户+商品
        OrderInfo orderInfo = miaoshaService.miaosha(user,goods);
        model.addAttribute("orderInfo", orderInfo);
        model.addAttribute("goods",goods);
        return "order_detail";
    }

}
```

- **判断库存的实现**：

```java
//1、控制器中的判断，得到GoodsVo类对象，判断库存stockCount；
public class MiaoshaController {
    @Autowired
    GoodsService goodsService;
    
    @RequestMapping("/do_miaosha")
    public String list(Model model, MiaoshaUser user,@RequestParam("goodsId")long goodsId){
      //判断库存
      GoodsVo goods = goodsService.getGoodsVoByGoodsId(goodsId);
      Integer stockCount = goods.getStockCount();
      if(stockCount <= 0){
          model.addAttribute("errmsg", CodeMsg.MIAO_SHA_OVER.getMsg());
      return "miaosha_fail";
   }
}

//2.GoodsService调用GoodsDao中对数据库的访问
@Service
public class GoodsService {

    @Autowired
    GoodsDao goodsDao;
    
     public GoodsVo getGoodsVoByGoodsId(long goodsId) {
        return goodsDao.getGoodsVoByGoodsId(goodsId);
    }    
}

//3.GoodsDao对数据库的访问操作
@Mapper
@Repository
public interface GoodsDao {
   GoodsVo getGoodsVoByGoodsId(@Param("goodsId") long goodsId);    
}
    
<select id="getGoodsVoByGoodsId" resultType="cn.seu.vo.GoodsVo">
        select g.*,mg.stock_count, mg.start_date, mg.end_date,mg.miaosha_price from miaosha_goods mg left join goods g on mg.goods_id = g.id where g.id = #{goodsId}
</select>    
```

- 判断该用户是否已经秒杀成功，防止一人秒杀多个商品；

```java
//1、控制器中的判断，得到MiaoshaOrder对象，判断用户是否已经秒杀；
public class MiaoshaController {
    @Autowired
    OrderService orderService;
    
    @RequestMapping("/do_miaosha")
    public String list(Model model, MiaoshaUser user,@RequestParam("goodsId")long goodsId){
      //判断秒杀成功，防止一人秒杀多个商品
      MiaoshaOrder order = orderService.getMiaoshaOrderByUserIdGoodsId(user.getId(),goodsId);
        if(order != null){
            model.addAttribute("errmsg", CodeMsg.REPEATE_MIAOSHA.getMsg());
            return "miaosha_fail";
        }
   }
}

//2.OrderService调用OrderDao中对数据库的访问
@Service
public class OrderService {

    @Autowired
    OrderDao orderDao;
	
    public MiaoshaOrder getMiaoshaOrderByUserIdGoodsId(long userId, long goodsId) {
        return orderDao.getMiaoshaOrderByUserIdGoodsId(userId, goodsId);
    }    
}

//3.OrderDao对数据库的操作
@Mapper
@Repository
public interface OrderDao {
    @Select("select * from miaosha_order where user_id = #{userId} and goods_id = #{goodsId}")
    MiaoshaOrder getMiaoshaOrderByUserIdGoodsId(@Param("userId") long userId, @Param("goodsId") long goodsId);
}    
```

- **执行秒杀操作：减库存 下订单 写入秒杀订单：**

```java
//1、控制器中调用秒杀方法
public class MiaoshaController {
    @Autowired
    MiaoshaService miaoshaService;
    
    @RequestMapping("/do_miaosha")
    public String list(Model model, MiaoshaUser user,@RequestParam("goodsId")long goodsId){
		//秒杀操作，调用MiaoshaService中的miaosha方法
        OrderInfo orderInfo = miaoshaService.miaosha(user,goods);
   }
}

//2.MiaoshaService中调用GoodsService减库存，OrderService下订单
//使用@Transactional作为整体的事务；
@Service
public class MiaoshaService {

    @Autowired
    GoodsService goodsService;
    @Autowired
    OrderService orderService;

    @Transactional
    public OrderInfo miaosha(MiaoshaUser user, GoodsVo goods) {
        //减库存
        goodsService.reduceStock(goods);
        //下订单，order_info  miaosha_order
        return orderService.createOrder(user,goods);

    }
}

//3.GoodsService调用GoodsDao减库存
@Service
public class GoodsService {

    @Autowired
    GoodsDao goodsDao;
    
    public void reduceStock(GoodsVo goods) {
        MiaoshaGoods g = new MiaoshaGoods();
        g.setGoodsId(goods.getId());
        goodsDao.reduceStock(g);
    }
    
}
//GoodsDao操作数据库减库存
@Mapper
@Repository
public interface GoodsDao {
    @Update("update miaosha_goods set stock_count = stock_count - 1 where goods_id = #{goodsId}")
    int reduceStock(MiaoshaGoods g);
}

//4.OrderService给OrderDao传入参数，下订单和秒杀订单
@Service
public class OrderService {

    @Transactional
    public OrderInfo createOrder(MiaoshaUser user, GoodsVo goods) {
        OrderInfo orderInfo = new OrderInfo();
        orderInfo.setCreateDate(new Date());
        orderInfo.setDeliveryAddrId(0L);
        orderInfo.setGoodsCount(1);
        orderInfo.setGoodsId(goods.getId());
        orderInfo.setGoodsName(goods.getGoodsName());
        orderInfo.setGoodsPrice(goods.getMiaoshaPrice());
        orderInfo.setOrderChannel(1);
        orderInfo.setStatus(0);
        orderInfo.setUserId(user.getId());
        long orderId = orderDao.insert(orderInfo);//订单ID
        MiaoshaOrder miaoshaOrder = new MiaoshaOrder();
        miaoshaOrder.setGoodsId(goods.getId());
        miaoshaOrder.setOrderId(orderId);
        miaoshaOrder.setUserId(user.getId());
        orderDao.insertMiaoshaOrder(miaoshaOrder);
        return orderInfo;
    }
}

//OrderDao下订单和秒杀订单，对数据库的操作；
//注意：long insert(OrderInfo orderInfo)返回订单号的操作，statement="select last_insert_id()
@Mapper
@Repository
public interface OrderDao {
    @Insert("insert into order_info(user_id, goods_id, goods_name, goods_count, goods_price, order_channel, status, create_date)values("
            + "#{userId}, #{goodsId}, #{goodsName}, #{goodsCount}, #{goodsPrice}, #{orderChannel},#{status},#{createDate} )")
    @SelectKey(keyColumn="id", keyProperty="id", resultType=long.class, before=false, statement="select last_insert_id()")
    long insert(OrderInfo orderInfo);//下订单，返回订单号
    @Insert("insert into miaosha_order (user_id, goods_id, order_id)values(#{userId}, #{goodsId}, #{orderId})")
    void insertMiaoshaOrder(MiaoshaOrder miaoshaOrder);//下秒杀订单
    
}
```

![秒杀成功跳转支付页面](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201205222331652.png)



## 4、JMeter压测

数据库配置到服务器上；

TPS：

QPS：



## 5、页面优化技术(页面静态化:HTML + AJAX请求)(后端接口传对象渲染，页面存储在客户端)

设置缓存后，记得在方法中先提取缓存，再做其他操作；从缓存中取模板渲染，缓存中存在对象和页面；

### 5.1  页面缓存

商品分页，但只缓存前几页；

所有人访问的页面都是一样的；在秒杀项目中即为：goods_list.html页面，所有人看到的页面都是一致的，相应的Redis中的Key为：GoodsKey:gl，唯一的；

```
Spring5 中将页面缓存放到WebContext类中；用WebContext类代替SpringWebContext类；
页面缓存一般有效期比较短；
```

### 注意：Spring2.x的改进：

```
SpringBoot 1.5.8 RELEASE版本中，手动渲染使用SpringWebContext类，但是经过查找，在Spring2.0.x中对手动渲染的SpringWebContext类替换为WebContext类，剔除了对ApplicationContext类的依赖：
老版本：SpringWebContext ctx = new SpringWebContext(request,response,
    			request.getServletContext(),request.getLocale(), model.asMap(), applicationContext );
新版本：WebContext ctx = new WebContext(request,response,
                request.getServletContext(),request.getLocale(), model.asMap());
```

```java
@Controller
@RequestMapping("/goods")
public class GoodsController {

    @Autowired
    MiaoshaUserService userService;

    @Autowired
    RedisService redisService;

    @Autowired
    GoodsService goodsService;

    @Autowired
    ThymeleafViewResolver thymeleafViewResolver;
	@RequestMapping(value="/to_list", produces="text/html")
    @ResponseBody
    public String list(HttpServletRequest request, HttpServletResponse response,
                       Model model,MiaoshaUser user) {
        model.addAttribute("user", user);
        //取缓存
        String html = redisService.get(GoodsKey.getGoodsList, "", String.class);
        if(!StringUtils.isEmpty(html)) {
            return html;
        }
        List<GoodsVo> goodsList = goodsService.listGoodsVo();
        model.addAttribute("goodsList", goodsList);
        WebContext ctx = new WebContext(request,response,
                request.getServletContext(),request.getLocale(), model.asMap());
        //手动渲染
        html = thymeleafViewResolver.getTemplateEngine().process("goods_list", ctx);
        if(!StringUtils.isEmpty(html)) {
            redisService.set(GoodsKey.getGoodsList, "", html);
        }
        return html;
    }
}
```

- 页面缓存手动渲染：将



### 5.2 URL缓存

不同的对象的缓存是不同的，在秒杀项目中，秒杀商品1和秒杀商品2对应的缓存是不同的，对应的Redis的缓存key为：GoodsKey:gd1 和 GoodsKey:gd2；称为URL缓存；







### 5.3 对象缓存

- Service对象调用Service对象，Service中可能有缓存但Dao中没有；
- 更新用户密码的方法：必须先更新数据库后更新缓存；若顺序相反，先更新缓存后更新数据库，则缓存中的内容是旧的；
- 动态数据从服务端接收，页面静态存储；











### 5.4  商品详情页面静态化(缓存主要方法)

- 页面存储在客户端；常用技术AngularJS, VueJS；前后端分离，
- html + ajax请求
- AJAX调用；
- 静态页面存放在static目录下，把后缀名改为  .htm；

get 和 post的区别：



### 5.5  解决卖超问题

```
问题一：当只有一个物品时，几个线程同时访问会导致库存为负，即卖超问题；
解决方法：
减库存SQL加库存判断：stock_count > 0
@Update("update miaosha_goods set stock_count = stock_count - 1 where goods_id = #{goodsId} and stock_count > 0")
int reduceStock(MiaoshaGoods g);

问题二：一个用户刷两个请求，有两个商品
解决方法：
在miaosha_order数据库中建立唯一索引(user_id 和 goods_id)，秒杀表中，实际中会让用户输验证码；
```

![image-20201208213101062](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201208213101062.png)







### 5.6  静态资源优化

- JS/CSS压缩，减少流量；
- 多个JS/CSS组合，减少连接数；
- CDN就近访问；



### 总结

并发量大瓶颈在数据库；

可以做缓存；不同粒度、层面的缓存，减少数据库的压力；



## 6、服务级高并发秒杀优化

- Mycat分库分表，阿里

### 6.1  秒杀接口优化(基于消息事务的最终一致性方案)

思路：减少数据库的访问

- 1.系统初始化，把商品库存数量加载到Redis；
- 2.收到请求，Redis预减库存，库存不足，直接返回，否则进入3
- 3.请求入队(消息队列)，立即返回排队中；(异步下单)
- 4.请求出队，生成订单，减少库存
- 5.客户端轮询，是否秒杀成功



### 6.2 秒杀接口优化实现

#### 6.2.1  系统初始化时库存加载到缓存

- MiaoshaController 实现 InitializingBean()接口，重写方法afterPropertiesSet()，就可以将秒杀商品库存放到Redis中；

```java
@Controller
@RequestMapping("/miaosha")
public class MiaoshaController implements InitializingBean {
	/**
     * 系统初始化时调用
     * 商品库存加载到Redis中
     */
    @Override
    public void afterPropertiesSet() throws Exception {
        List<GoodsVo> goodsList = goodsService.listGoodsVo();
        if(goodsList == null){
            return;
        }
        for(GoodsVo good:goodsList){
            redisService.set(GoodsKey.getMiaoshaGoodsStock,""+good.getId(), good.getStockCount());
            localOverMap.put(good.getId(),false);
        }

    }
}
```



#### 6.2.2  MiaoshaController方法中的逻辑

- MiaoshaController中的miaosha()方法逻辑：预减Redis中存储的库存，在Redis中判断是否已经秒杀过；若库存充足且该用户未秒杀过，则进入秒杀队列，即入队操作；
- MiaoshaMessage类中存有MiaoshaUser和秒杀商品ID；

```java
@Controller
@RequestMapping("/miaosha")
public class MiaoshaController implements InitializingBean {
//必须用post方法提交
    @RequestMapping(value = "/do_miaosha", method = RequestMethod.POST)
    @ResponseBody
    public Result<Integer> miaosha(Model model, MiaoshaUser user,
                          @RequestParam("goodsId")long goodsId){
        model.addAttribute("user",user);
        if(user == null)
            return Result.error(CodeMsg.SESSION_ERROR);
        //内存标记，减少Redis访问
        boolean over = localOverMap.get(goodsId);
        if(over){
            return Result.error(CodeMsg.MIAO_SHA_OVER);
        }

        //预减库存
        long stock = redisService.decr(GoodsKey.getMiaoshaGoodsStock,""+goodsId);
        if(stock < 0){
            localOverMap.put(goodsId,true);
            return Result.error(CodeMsg.MIAO_SHA_OVER);
        }
        //判断该用户是否秒杀成功
        MiaoshaOrder order = orderService.getMiaoshaOrderByUserIdGoodsId(user.getId(),goodsId);
        if(order != null){
            return Result.error(CodeMsg.REPEATE_MIAOSHA);
        }
        //入队
        MiaoshaMessage message = new MiaoshaMessage();
        message.setUser(user);
        message.setGoodsId(goodsId);
        sender.sendMiaoshaMessage(message);
        return Result.success(0);// 0 代表排队中
	}
}

public class MiaoshaMessage {

    private MiaoshaUser user;
    private long goodsId;

}
```



#### 6.2.3  RabbitMQ Receiver中的操作

- 秒杀信息入队之后等待处理，在Receiver中可以对数据库操作，因为通过过滤请求比较少；

```java
@Service
public class MQReceiver {
    @Autowired
    GoodsService goodsService;
    @Autowired
    OrderService orderService;
    @Autowired
    MiaoshaService miaoshaService;

    private static Logger log = LoggerFactory.getLogger(MQReceiver.class);

    @RabbitListener(queues = MQConfig.MIAOSHA_QUEUE)
    public void receive(String message){
        log.info("receive message:" + message);
        MiaoshaMessage bean = RedisService.stringToBean(message, MiaoshaMessage.class);
        MiaoshaUser user = bean.getUser();
        long goodsId = bean.getGoodsId();

        //访问数据库判断库存
        GoodsVo goods = goodsService.getGoodsVoByGoodsId(goodsId);
        Integer stockCount = goods.getStockCount();
        if(stockCount <= 0){
            return;
        }
        //判断该用户是否秒杀成功
        MiaoshaOrder order = orderService.getMiaoshaOrderByUserIdGoodsId(user.getId(),goodsId);
        if(order != null){
            return;
        }
        //减库存 下订单 写入秒杀订单
        miaoshaService.miaosha(user,goods);
    }

}
```



#### 6.2.2 内存标记减少Redis访问

- 使用Map<Integer, Boolean> 来存放秒杀商品是否已经秒杀完成，可以减少对Redis的访问；
- 系统初始化同时初始化Map，查看Redis中存储的库存为0时，将商品秒杀库存设置为false；

```java
@Controller
@RequestMapping("/miaosha")
public class MiaoshaController implements InitializingBean {
    //1、定义localOverMap，存储商品是否已经卖完
	//预存商品是否已经卖完
    private Map<Long, Boolean> localOverMap = new HashMap<>();
    /**
     * 系统初始化时调用
     * 商品库存加载到Redis中
     */
    @Override
    public void afterPropertiesSet() throws Exception {
        List<GoodsVo> goodsList = goodsService.listGoodsVo();
        if(goodsList == null){
            return;
        }
        for(GoodsVo good:goodsList){
            redisService.set(GoodsKey.getMiaoshaGoodsStock,""+good.getId(), good.getStockCount());
            //系统初始化时，设置boolean值为false
            localOverMap.put(good.getId(),false);
        }

    }
    
    //必须用post方法提交
    @RequestMapping(value = "/do_miaosha", method = RequestMethod.POST)
    @ResponseBody
    public Result<Integer> miaosha(Model model, MiaoshaUser user,
                          @RequestParam("goodsId")long goodsId){
		model.addAttribute("user",user);
        if(user == null)
            return Result.error(CodeMsg.SESSION_ERROR);
        //3、内存标记，减少Redis访问，
        boolean over = localOverMap.get(goodsId);
        if(over){
            return Result.error(CodeMsg.MIAO_SHA_OVER);
        }
		 //预减库存
        long stock = redisService.decr(GoodsKey.getMiaoshaGoodsStock,""+goodsId);
        if(stock < 0){
            //4、库存为0之后，设置库存为true，表示已经秒杀完
            localOverMap.put(goodsId,true);
            return Result.error(CodeMsg.MIAO_SHA_OVER);
        }
        ...
    }
}
```



## 7、安全优化

### 7.1 秒杀接口地址隐藏

```
1、接口改造，带上PathVariable参数；
2、添加生成地址的接口
3、秒杀收到请求，先验证PathVariable
```



### 7.2  数学公式验证码

- 防机器人
- 分散用户的请求

```
1、添加生成验证码的接口
2、在获取秒杀路径的时候，验证验证码
3、ScriptEngine使用
BufferImage类，内存中的图像；
图形验证码刷新，重新请求；
```



### 7.3 接口限流防刷(简单)

- 限制用户规定时间内的访问次数，使用Redis实现；

- Redis中key 有效时间的设置：Redis Setex 命令为指定的 key 设置值及其过期时间。如果 key 已经存在， SETEX 命令将会替换旧的值。

  ```
  //second<=0表示，永不过期
  if(second <= 0){
      jedis.set(realKey, str);
  }else{
      jedis.setex(realKey,second,str);
  }
  ```

- ThreadLocal()，放到当前线程中的数据；

```java
// Example : 实现5秒钟访问5次
public class AccessKey extends BasePrefix{
	//expireSeconds，有效时间5秒钟
	private AccessKey(int expireSeconds, String prefix){
		super(expireSeconds, prefix);
	}
	public static AccessKey access = new AccessKey(5, "access");
       
}

//查询访问的次数
Integer count = redisService.get(AccessKey.access,key,Integer.class);
if(count == null){
    redisService.set(AccessKey.access,key,1);
}else if(count < 5){//有效时间内访问5次
    redisService.incr((AccessKey.access,key);
}else{
    return Result.error(CodeMsg.ACCESS_LIMIT_REACHED);
}
```



### 7.4  接口防刷限流(通用)(拦截器、设置规定时间的访问次数)

#### 7.4.1  定义注解

```java
//1、定义注解
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface AccessLimit {

    int seconds();
    int maxCount();
    boolean needLogin() default true;

}
```



#### 7.4.2  SpringBoot定义拦截器

- 功能：使用拦截器，拦截用户登录、防刷限流；
- 获得登录的User后保存到ThreadLocal中，ThreadLocal是多线程中保证线程安全的方式，与当前线程绑定，数据在线程中单独存在且线程安全；
- 拦截器，配置到WebMvcConfigurationSupport() 的 addInterceptor()方法；

```java
@Service
public class AccessInterceptor implements HandlerInterceptor {

    @Autowired
    MiaoshaUserService userService;

    @Autowired
    RedisService redisService;

    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if(handler instanceof HandlerMethod){
            //获取登录的用户信息
            MiaoshaUser user = getUser(request, response);
            //用户存储在ThreadLocal中
            UserContext.setUser(user);

            HandlerMethod hm = (HandlerMethod)handler;
            AccessLimit accessLimit = hm.getMethodAnnotation(AccessLimit.class);
            if(accessLimit == null){
                return true;
            }
            int seconds = accessLimit.seconds();
            int maxCount = accessLimit.maxCount();
            boolean needLogin = accessLimit.needLogin();
            String key = request.getRequestURI();
            if(needLogin){
                if(user == null){
                    render(response, CodeMsg.SESSION_ERROR);
                    return false;
                }
                key += "_" + user.getId();
            }else{
                // do nothing
            }
            AccessKey ak = AccessKey.withExpire(seconds);
            Integer count = redisService.get(ak, key, Integer.class);
            if(count  == null) {
                redisService.set(ak, key, 1);
            }else if(count < maxCount) {
                redisService.incr(ak, key);
            }else {
                render(response, CodeMsg.ACCESS_LIMIT_REACHED);
                return false;
            }

        }
        return true;
    }
    
	//输出消息到浏览器
    private void render(HttpServletResponse response, CodeMsg sessionError) throws Exception{
        response.setContentType("application/json;charset=UTF-8");
        ServletOutputStream out = response.getOutputStream();
        String str = JSON.toJSONString(sessionError);
        out.write(str.getBytes());
        out.flush();
        out.close();
    }

}
```



#### 7.4.3  ThreadLocal存储线程对象

- 获得登录的User后保存到ThreadLocal中，ThreadLocal是多线程中保证线程安全的方式，与当前线程绑定，数据在线程中单独存在且线程安全；

```java
public class UserContext {

 private static ThreadLocal<MiaoshaUser> userHolder = new ThreadLocal<MiaoshaUser>();

 public static void setUser(MiaoshaUser user) {
    userHolder.set(user);
 }

 public static MiaoshaUser getUser() {
    return userHolder.get();
 }

}
```



#### 7.4.4  拦截器配置到WebMvcConfigurationSupport类中

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {

    @Autowired
    UserArgumentResolver userArgumentResolver;
    @Autowired
    AccessInterceptor accessInterceptor;
	//配置参数解析器
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(userArgumentResolver);
    }
	//配置拦截器
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(accessInterceptor);
    }
	//静态资源
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/**").
            addResourceLocations("classpath:/static/").
                addResourceLocations("classpath:/resources/").addResourceLocations("classpath:/public");
        super.addResourceHandlers(registry);
    }


}
```



## 8、总结

- SpringBoot环境搭建
- 集成Thymeleaf， Result结果封装**(编程技巧)**
- 集成Mybatis + Druid
- 集成Jedis+Redis安装+通用缓存Key封装
- 明文密码两次MD5处理
- JSR303参数校验+全局异常处理器，服务端校验可以防止恶意用户**(编程技巧)**
- 分布式Session**(重要)**
- 实现秒杀功能
- 应对并发最有效的方法：使用缓存，页面静态优化；CDN优化
- 接口优化：异步下单
- Redis预减库存，减少数据库访问
- 内存标记减少Redis访问
- RabbitMQ队列缓存，异步下单，增强用户体验
- 防刷限流---> 拦截器，拦截用户登录，规定时间访问次数





## 项目后期需要补充学习的点

### WebMvcConfigurationSupport类

- SpringBoot配置拦截器出现“No Mapping for GET ... ”静态资源的情况；在webConfig.java中添加代码：

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {

    @Autowired
    UserArgumentResolver userArgumentResolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(userArgumentResolver);
    }
	//解决无法加载静态资源的问题
    private static final String[] CLASSPATH_RESOURCE_LOCATIONS = {
            "classpath:/META-INF/resources/", "classpath:/resources/",
            "classpath:/static/", "classpath:/public/" };


    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        if (!registry.hasMappingForPattern("/webjars/**")) {
            registry.addResourceHandler("/webjars/**").addResourceLocations(
                    "classpath:/META-INF/resources/webjars/");
        }
        if (!registry.hasMappingForPattern("/**")) {
            registry.addResourceHandler("/**").addResourceLocations(
                    CLASSPATH_RESOURCE_LOCATIONS);
        }

    }

}
```



### 实际电商系统中的ID使用snowflake算法

- 雪花算法(snowflake)：分布式环境生成全局唯一的订单号；
- snowflake是Twitter开源的分布式ID生成算法，结果是一个long型的ID；



### 访问页面静态资源

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {

    @Autowired
    UserArgumentResolver userArgumentResolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(userArgumentResolver);
    }

    
    private static final String[] CLASSPATH_RESOURCE_LOCATIONS = {
            "classpath:/META-INF/resources/", "classpath:/resources/",
            "classpath:/static/", "classpath:/public/" };
	//配置资源
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/**").
                addResourceLocations("classpath:/static/").                addResourceLocations("classpath:/resources/").addResourceLocations("classpath:/public");
        super.addResourceHandlers(registry);
    }

}
```















