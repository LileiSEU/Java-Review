# Spring全家桶

## JDBC

JDBC： Java DataBase Connectivity(Java 语言连接数据库)；JDBC是SUN公司制定的一套接口(interface)，数据库厂家提供驱动；

```java
// JDBC 编程6步
//1.注册驱动
try{
     Driver driver = new com.mysql.cj.jdbc.Driver();
     DriverManager.registerDriver(driver);
}catch (SQLException e){
     e.printStackTrace();
}
//2.获取连接
// 表示JVM的进程与数据库进程之间的通道打开了，这属于进程间的通信，重量级的，使用完之后一定要关闭
String url = "jdbc:mysql://10.192.52.233:3306/bjpowernode";
String user = "root";
String password = "123";
Connection conn = DriverManager.getConnection(url,user,password);
//3.获取数据库操作对象(Statement专门执行SQL语句)
Statement stmt = conn.createStatement();
//4.执行SQL语句
String sql = "insert into dept(deptno, dname, loc) values(50,'人事部','北京')";
//专门执行DML语句（insert delete update）
//返回值是影响数据库的记录条数
//int executeUpdate(insert/delete/update);此函数只能用SQL的增删改语句
int count = stmt.executeUpdate(sql);
System.out.println(count == 1 ? "保存成功" : "保存失败");
//5.处理查询结果集
ResultSet rs = null;
rs = stmt.executeQuery(sql);//查询语句
while(rs.next()){
       String deptno = rs.getString("deptno");//查询输入是列名
       String dname = rs.getString("dname");
       String loc = rs.getString("loc");
       System.out.println(deptno + ", " + dname + " ," + loc);
}
//6.释放资源
//为了保证资源一定释放，在finally语句块中关闭资源
//并且要遵循从小到大一次关闭
//分别对其进行try...catch...
try{
    if(rs != null)
         rs.close();
     }catch(SQLException e){
         e.printStackTrace();
    }

    try{
        if(stmt != null)
                    stmt.close();
    }catch(SQLException e){
        e.printStackTrace();
    }
    try{
        if(conn != null)
                    conn.close();
    }catch(SQLException e){
         e.printStackTrace();
    }
}
```

### SQL注入

根本原因：用户输入的信息中含有SQL语句的关键字，并且这些关键字参与SQL的编译过程，导致SQL语句的原意被扭曲，进而达到SQL注入；

解决SQL注入：

​		只要用户提供的信息不参与SQL语句的编译过程，问题就解决了；

​		即使用户提供的信息中含有SQL语句的关键字，但是没有参与编译，不起作用；

​		要想用户信息不参与SQL语句的编译，使用**java.sql.PreparedStatement**

​		PreparedStatement接口继承了java.sql.Statement，属于预编译的数据库操作对象；

​		**PreparedStatement的原理是：预先对SQL的框架进行编译，然后再给SQL语句传“值”；**

```java
// 解决SQL注入代码：（以后更常用）
PreparedStatement ps = null;//这里使用PreparedStatement（预编译的数据库操作对象）
//注册驱动；获取连接对象；获取操作对象；
Class.forName("com.mysql.cj.jdbc.Driver");
conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/bjpowernode","root","123");
//3.获取预编译的数据库操作对象
//其中一个？表示一个占位符，一个？将来接收一个“值”。注意：占位符不能使用单引号括起来
String sql = "SELECT loginName,loginPwd,realName FROM t_user WHERE loginName = ? and loginPwd = ?;";//SQL语句框架
//程序执行到此处，会发送sql语句给DBMS，然后DBMS进行sql语句的预编译，？会被''代替；
ps = conn.prepareStatement(sql);
//给占位符？传值（第一个？下标是1，第二个？下标是2，JDBC中所有下标从1开始）
ps.setString(1,userName);
ps.setString(2,passWord);
//4.执行sql语句
rs = ps.executeQuery();
if(rs.next()){
   loginSuccess = true;
   String loginName = rs.getString("loginName");
   String pwd = rs.getString("loginPwd");
   String realName = rs.getString("realName");
   System.out.println("登录名：" + loginName + " , 登陆密码： " + pwd + " ，真实姓名为： " + realName);
}
```

解决sql注入的关键：用户提供的信息即使含有sql语句的关键字，但是这些关键字并没有参与编译，不起作用；

### Statement 和 PreparedStatement 对比

--Statement存在sql注入问题，PreparedStatement解决了sql注入问题；

--Statement是编译一次执行一次，PreparedStatement是编译一次，执行多次（数据库若命令相同，则不需要编译），所以PreparedStatement的效率高一点；

--PreparedStatement会在编译阶段做类型的安全检查；

综上所述：PreparedStatement使用较多，只有极少数的情况下需要使用Statement;

Statement支持sql注入，凡是业务方面要求是需要进行sql语句拼接的，必须使用Statement。



### JDBC事务自动提交机制

JDBC中的事务是自动提交的；只要执行任意一条DML语句，则自动提交一次；这是JDBC默认的事务行为；

但是在实际业务中，通常都是N条DML语句共同联合才能完成的，必须保证他们这些DML语句在同一个事务中同时成功或失败；

```java
// connection接口的三个方法：
void setAutoCommit(boolean autoCommit) throws SQLException;//事务自动提交机制的开启与否
void commit() throws SQLException;//使所有上一次提交/回滚后进行的更改成为持久更改，并释放该Connection对象当前持有的所有数据库锁；
void rollback() throws SQLException;//取消在当前事务中进行的所有更改，并释放此Connection对象当前持有的所有数据库锁；
// 实际执行代码
conn.setAutoCommit(false);
conn.commit();
conn.rollback();
```



## JavaWeb

### Servlet

Servlet是Sun公司开发动态Web的一门技术，Sun公司在这些API中提供一个接口：Servlet；开发Servlet程序，完成两个小步骤：

1）编写一个类，实现Servlet接口；2）把开发好的Java类部署到Web服务器中；

把实现了Servlet接口的Java程序，叫做Servlet；Servlet接口被定义用来处理客户端发来的请求，又针对HTTP协议提供了子类HttpServlet处理HTTP请求。也就是说Servlet在顶层设计上可以处理各种协议的请求，只不过目前仅仅应用在HTTP请求上；

HTTPServletRequest代表客户端的请求，用户通过HTTP协议访问服务器，HTTP请求中的所有信息都会被封装到HTTPServletRequest，通过这个HTTPServletRequest的方法，获得客户端的所有信息；

HttpServletRequest **request**是请求对象，内部封装了客户端请求的各种信息，主要有请求路径、请求参数、请求头、cookie等；

HttpServletResponse **response**是响应对象，专门用来生成响应；



### Cookie、Session

**基于传统的Session认证**：

1.认证方式

- http协议本身是一种无状态的协议，而这就意味着如果用户向我们的应用提供了用户名和密码来进行用户认证，那么下一次请求时，用户还要在一次进行用户认证才行，因为根据http协议，服务端并不能知道是哪个用户发出的请求，所以为了让应用能识别是哪个用户发出的请求，在服务器存储一份用户登录的信息，这份登录信息会在响应时传递给浏览器，告诉其保存为cookie，以便下次请求时发送给应用，这样应用就可以识别请求来自哪个用户了，这就是传统的基于session认证。
- Session是服务器对象，服务器响应的时候将sessionid存到cookie中，以后每次浏览器访问携带cookie即sessionid；
- 第一次登陆成功后，服务端会响应一个 JSESSIONID 到 cookie 中，以后每次访问都携带cookie即携带了 JSESSIONID，通过JSESSIONID 找服务端的 session 对象；

3.暴露问题

- 1.每个用户经过应用认证后，应用都要在服务端做一次记录，以方便用户下次请求的鉴别，通常而言session都是保存在内存中(例如Redis存储），而随着认证用户的增多，服务端的开销会明显增大；
- 2.用户认证之后，服务端做认证记录，如果认证的记录被保存在内存中的话，这意味着用户下次请求还必须要请求在这次服务器上，这样才能拿到授权的资源，这样在分布式的应用上，相应的限制了负载均衡器的能力，这也以为着限制了应用的扩展能力(基于Redis集群的session共享)；
- 3.因为这是基于cookie来进行用户识别的，cookie如果被截获，用户就容易受到跨站请求伪造的攻击；
- 4.增加了前后端分离系统的复杂性：
	前后端分离应用在应用解耦后增加了部署的复杂性。通常用户一次请求就要转发多次。如果用session每次携带	sessionid到服务器，服务器还要查询用户信息。同时如果用户很多，这些信息存储在服务器内存中，给服务器增加负担。另外，sessionid是一个特征值，表达的信息不够





**会话**：用户打开一个浏览器，点击了很多超链接，访问多个web资源，关闭浏览器，这个过程可以称之为会话；

**有状态会话**：服务端记录客户端曾经来过，称之为有状态会话；

​	1、服务端给客户端一个信件，客户端下次访问服务器带上信件就可以了；cookie

​	2、服务器登记客户来过，下次来的时候匹配客户；session

**保存会话的两种技术**：

- cookie：客户端技术(响应、请求)
- session：服务器技术，利用服务器技术，可以保存用户的会话信息，可以把信息或数组放在Session中；

Session和Cookie的区别

​	1、cookie是把用户的数据写给用户的浏览器，浏览器保存；(可以保存多个)

​	2、Session把用户的数据写到用户独占Session中，服务器端保存；(保存重要的信息，减少服务器资源的浪费)

​	3、Session对象由服务器创建；

​	4、public void setAttribute(String name, Object value)，session创建键值对，String+Object；public Cookie(String name, String value) {}，cookie的构造只能是String+String；





### jwt

JWT简称 JSON Web Token，也就是通过JSON 形式作为 Web 应用中的令牌，用于在各方之间安全地将信息作为JSON对象传输，在数据传输过程中还可以完成数据加密、签名等相关处理；

**JWT 能做什么？**

1.授权

- 这是使用JWT的最常见方案。一旦用户登录，每个后续请求将包括JWT，从而允许用户访问该令牌允许的路由，服务和资源。单点登录时当今使用JWT的一项功能，因为它的开销很小并且可以在不同的域中轻松使用。

2.信息交换

- JSON Web Token是在各方之间安全地传输信息的好方法。因为可以对JWT进行签名，所以可以确保发件人是他们所说的人。此外，由于签名是使用标头和有效负载计算的，因此还可以验证内容是否遭到篡改。

![image-20210105211223279](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210105211223279.png)

```markdown
# 1.认证流程
- 首先，前端通过Web表单将自己的用户名和密码发送到后端的接口，这一过程一般是一个 HTTP POST请求，建议的方式是通过SSL加密的传输(http协议)，从而避免敏感信息被嗅探；
- 后端核对用户名和密码成功后，将用户的ID等其他信息作为JWT Payload(负载)，将其与头部分别进行Base64编码拼接后签名，形成一个JWT(token)，形成的JWT就是一个形同lll.zzz.xxx的字符串。Token: head.payload.signature;
- 后端将JWT字符串作为登录成功的返回结果返回给前端。前端可以将返回的结果保存在localStorage(本地缓存)或sessionStorage上，退出登录时前端删除保存的JWT即可；
- 前端在每次请求时将JWT放入HTTP Header中的Authorization位(授权位)；
- 后端检查是否存在，如存在验证JWT额有效性；
- 验证通过后后端使用JWT中包含的用户信息进行其他逻辑操作，并返回相应结果；

# 2.jwt优势
- 简洁(Compact)：可以通过URL，POST参数或者在HTTP header发送，因为数据量小，传输速度也很快；
- 自包含(Self-contained)：负载中包含了所有用户所需要的信息，避免了多次查询数据库；
- 因为Token是以JSON加密的形式保存在客户端的，所以JWT是跨语言的，原则上任何web形式都支持；
- 不需要在服务端保存会话信息，特别适用于分布式微服务；
- 应用场景：一次性验证

# 3.jwt问题
- 无法满足注销场景：传统的 session+cookie 方案用户点击注销，服务端清空 session 即可，因为状态保存在服务端。但 jwt 的方案就比较难办了，因为 jwt 是无状态的，服务端通过计算来校验有效性。没有存储起来，所以即使客户端删除了 jwt，但是该 jwt 还是在有效期内，只不过处于一个游离状态。
- 无法满足修改密码场景：修改密码则略微有些不同，假设号被盗了，修改密码（是用户密码，不是 jwt 的 secret）之后，盗号者在原 jwt 有效期之内依旧可以继续访问系统，所以仅仅清空 cookie 自然是不够的，这时，需要强制性的修改 secret。
```













## 互联网持久框架 - Mybatis

当配置了XML文件或者提供代码后，MyBatis会读取配置文件，通过**Configuration 类对象构建整个Mybatis的上下文**。

### MyBatis 核心组件

Mybatis 的核心组件分为4个部分：

SqlSessionFactoryBuilder(构造器)：根据配置或者代码来生成SqlSessionFactory，采用的是分步构建的Builder模式。**生命周期：**只能存在于创建SqlSessionFactory的方法中；

SqlSessionFactory(工厂接口)：每个基于Mybatis的应用都是以一个SqlSessionFactory的实例为中心的，而SqlSessionFactory的唯一作用就是生产Mybatis的核心接口对象SqlSession，所以采用**单例模式**处理它。**生命周期：**MyBatis的应用周期，SqlSessionFactory作为一个单例在应用中共享。

SqlSession(会话)：在MyBatis中SqlSession是其核心接口。在MyBatis中有两个实现类，DefaultSqlSession和SqlSessionManager。DefaultSqlSession 是单线程使用的，而SqlSessionManager在多线程环境下使用；作用有3个：获取Mapper接口、发送SQL给数据库、控制数据库事务。**生命周期：**存活在一个业务请求中。

SQL  Mapper(映射器)：映射器由一个接口和对应的XML文件(或注解)组成，可以配置以下内容：描述映射规则、提供SQL语句，并可以配置SQL参数类型、返回类型、缓存刷新等信息、配置缓存、提供动态SQL。MyBatis运用动态代理技术为接口生成一个代理对象使得接口可以运行。**生命周期：**一个请求中完成相应逻辑后销毁。

### MyBatis 配置

MyBatis配置 - properties属性：配置properties属性的三种方式并按优先级排序：使用程序传递  -->  使用properties文件的方式  -->  使用property子元素的方式；

MyBatis配置 - settings 设置：常用settings配置项：关于缓存的cacheEnabled，关于级联的 lazyLoadingEnabled 和 aggressiveLazyLoading，关于自动映射的autoMappingBehavior 和 mapUnderscoreToCamelCase，关于执行器类型的 defaultExecutorType。

Mybatis配置 - typeAliases别名：系统定义别名和自定义别名；系统定义别名使用TypeAliasRegistry 的 registerAlias 方法注册别名，通过 Configuration 类获取 TypeAliasRegistry 类对象，其中的 getTypeAliasRegistry 方法可以获得别名，然后通过 registerAlias 方法对别名注册。

Mybatis配置 - typeHandler 类型转换器：在JDBC中，需要在 PreparedStatement 对象中设置已经预编译过的SQL语句的参数。执行SQL后，会通过ResultSet对象获取得到数据库的数据，MyBatis是根据数据的类型通过typeHandler来实现的。在typeHandler中，分为jdbcType和javaType，其中jdbcType用于定义数据库类型，而javaType 用于定义java类型，则typeHandler的作用就是承担jdbcType和javaType之间的相互转换；还可以自定义typeHander去处理类型之间的相互转换问题，自定义的typeHandler通过配置或扫描来注册。

```java
// 定义类型转换器需要实现 TypeHandler 接口
public interface TypeHandler<T> {
    void setParameter();
    T getResult();
}
// 系统定义的类型转换器继承了 BaseTypeHandler
public abstract class BaseTypeHandler<T> extends TypeReference<T> implements TypeHandler<T>{
}
```

MyBatis配置 - ObjectFactory(对象工厂)：当创建结果集时，MyBatis会使用一个对象工厂来完成创建这个结果集实例。

MyBatis配置 - environments(运行环境)：运行环境主要的作用是配置数据库信息，可配置的元素：事务管理器(transactionManager)、数据源(dataSource)。

```shell
# 事务管理器(transactionManager)
	transactionManager 需要实现 Transaction接口，该接口的主要方法有：提交、回滚、关闭数据库事务；
	<transactionManager type="JDBC">: 使用JdbcTransactionFactory生成的JdbcTransaction对象实现，以JDBC的方式对数据库的提交和回滚进行操作；
	<transactionManager type="MANAGED">: 使用 ManagedTransactionFactory生成的ManagedTransaction对象实现，提交和回滚方法不用任何操作，而是把事务交给容器处理。

# 数据源(dataSource)
	<dataSource tyep="UNPOLLED">：UnpooledDataSourceFactory --> UnpooledDataSource类对象，非数据库池的管理方式，每次请求都会打开一个新的数据库连接，创建比较慢；
	<dataSource tyep="POLLED">：PooledDataSourceFactory --> PooledDataSource类对象，类似于线程池的概念，利用池将JDBC的Connection对象组织起来，开始时会有一些空置并且已经连接好的数据库连接，请求时无须建立和验证，省去了创建新的连接实例所必须的初始化和认证时间。
	<dataSource tyep="JNDI">：数据源JNDI的实现是为了能在如EJB或应用服务器这类容器中使用，容器可以集中或在外部配置数据源，然后放置一个JNDI上下文的引用。
```



### 映射器

```shell
# 参数传递的四种方法
	1. 使用map传递参数导致了业务可读性的丧失，导致后序扩展和维护的困难，在实际应用中不使用该方法；
	2. 使用@Param注解传递多个参数，n <= 5 时，最佳的传参方法；
	3. 参数多于 5 个时，使用 Java Bean 方式；
	4. 对于使用混合参数的，要明确参数的合理性。
```

**级联**：MyBatis的级联分为3种：级联的好处是获取关联数据便捷，但是级联过多会增加系统的复杂度，同时降低系统的性能，所以当级联的层级超过3层时，就不要考虑使用级联了，因为会造成多个对象的关联导致系统的耦合、复杂。

- 鉴别器(discriminator)：根据某些条件决定采用具体实现类级联的方案；
- 一对一(association)
- 一对多(collection)

**缓存**：Mybatis分为一级缓存和二级缓存，同时也可以配置关于缓存的设置。

- 一级缓存：在SqlSession上的缓存(默认开启)，当一个SqlSession第一次通过SQL和参数获取对象后，就会将其缓存起来，如果下次的SQL和参数都没有发生变化并且缓存没有超时或者声明需要刷新缓存时，就可以从缓存中获取数据。一级缓存不需要POJO对象可序列化(实现java.io.Serializable接口)；
- 二级缓存：在SqlSessionFactory上的缓存，可以提供给各个SqlSession使用，需要实现可序列化接口。

**resultMap**：结果映射，指定列名和java对象属性的对应关系；自定义列值赋给属性；列名和属性名不一致时，使用resultMap指定对应关系。

```sehll
//Mapper文件中定义resultMap和在select语句中使用resultMap
    <!--使用resultMap,
        定义resultMap,
        id:自定义名称，表示定义的resultMap；
        type：java类型的全限定名称；
    -->
<resultMap id="studentMap" type="cn.seu.domain.Student">
   <!--  列名和java属性的关系  -->
   <!--  主键列，使用id标签
                column:列名
                property：java类型的属性名
   -->
   		<id column="id" property="id"/>
   <!--  非主键列，使用result  -->
        <result column="name" property="name"/>
        <result column="email" property="email"/>
        <result column="age" property="age"/>
</resultMap>
<select id="selectAllStudents" resultMap="studentMap">
    select id,name,email,age from student;
</select>
```



**动态SQL**

动态SQL：sql的内容是变化的，可以根据条件获取到不同的SQL语句，主要是where部分发生变化；

动态SQL的实现，使用的是MyBatis提供的标签，<if>,<where>,<foreach>;

动态SQL使用java对象作为参数对象；where标签里面是多个if；foreach：循环数组，list集合；



### MyBatis 的解析和运行原理

Mybatis的运行过程分为两大步：第1步，读取配置文件缓存到Configuration 对象，用以创建 SqlSessionFactory；第2步：SqlSession 的执行过程。

**构建SqlSessionFactory过程**：

```shell
# SqlSessionFactory 的创建过程：
# 参考博客：https://www.cnblogs.com/a-small-lyf/p/12678248.html
	1. 创建 SqlSessionFactoryBuilder 对象，简称sfb；
	2. sfb 中创建 XMLConfigBuilder 对象解析配置的XML文件，读出所配置的参数，XmlConfignBuilder创建一个默认配置的Configuration对象，XMLConfigBuilder继承了BaseBuilder，里面有Configuration对象的引用；
	3. 通过 XMLConfigBuilder 对象进行配置文件的解析，把配置信息存入Configuration对象中。Configuration 采用的是单例模式。
	4. 最后得到configuration对象后，sfb通过build方法创建SqlSessionFactory对象；SqlSessionFactory 是一个接口，提供了默认的实现类DefaultSqlSessionFactory。
	public SqlSessionFactory build(Configuration config) {return new DefaultSqlSessionFactory(config);}
```

```java
public class SqlSessionFactoryBuilder {
    public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());
}
// XMLConfigBuilder -> Configuraion
public class XMLConfigBuilder extends BaseBuilder{
    public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
}
```

在构建SqlSessionFactory中Configuration是最重要的，它的作用是：

- 读入配置文件，包括基础配置的XML和映射器XML；
- 初始化一些基础配置，比如 MyBatis的别名等，一些重要的类对象（比如插件、映射器、Object工厂、typeHandler对象等）；
- 提供单例，为后续创建SessionFactory服务，提供配置的参数；
- 执行一些重要对象的初始化方法；

**SqlSession运行过程**：

映射器Mapper的动态代理：Mapper映射是通过动态代理来实现的，生成动态代理对象；映射器就是一个动态代理对象进入到了MapperMethod的execute方法，然后通过评断进入SqlSession的 delete、update、insert、select等方法；

```java
// Mybatis 中代码
RoleMapper roleMapper = sqlSession.getMapper(RoleMapper.class);
// 方法调用过程如下：
// MapperMethod.execute()方法把SqlSession和当前运行的参数传递进去，执行；
// SqlSession 是接口，对应的默认实现类是： DefaultSqlSession,Executor用于执行SQL语句；
public class DefaultSqlSession implements SqlSession {
    private final Configuration configuration;
  	private final Executor executor;
}
```

![image-20210224161024562](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210224161024562.png)

SqlSession 下的四大对象：

- Executor 代表执行器，由它调度 StatementHandler、ParameterHandler、ResultSetHandler等来执行对应的SQL；其中 Statement 是最重要的；
- StatementHandler 的作用是使数据库的Statement(PreparedStatement)执行操作，许多重要的插件都是拦截它来实现的；
- ParameterHandler是用来处理SQL参数的；
- ResultSetHandler是进行数据集的封装返回处理的；

**插件原理**：MyBatis插件构建一层层的动态代理对象，在调度真实的Executor方法之前执行配置插件的代码，这就是插件的原理；层层代理；

一条查询SQL的执行过程：Executor先调用StatementHandler的prepare()方法预编译SQL，同时设置一些基本的运行参数。然后用parameterize()方法启用ParameterHandler设置参数，完成预编译执行查询；查询的话MyBatis会使用ResultSetHandler封装返回结果给调用者。最终创建结果集时，MyBatis会使用一个对象工厂来创建结果集实例，默认对象工厂：DefaultObjectFactory。



## Spring

Spring使用简单的POJO(Plain Old Java  Object，即无任何限制的普通Java对象)来进行企业级开发。每一个被Spring管理的Java对象都称之为Bean；而 Spring 提供了一个IOC容器用来初始化对象，解决对象间的依赖管理和对象的使用。

Spring 框架本身有四大原则：

- 使用POJO进行轻量级和最小侵入式开发；
- 通过依赖注入和基于接口编程实现松耦合；
- 通过AOP和默认习惯进行声明式编程；
- 使用AOP和模板(template)减少模式化代码。

Scope 描述的是 Spring 容器如何新建 Bean 的实例的，Spring 的 Scope 有以下几种，通过 @Scope 注解来实现：

（1）Singleton：一个Spring容器中只有一个Bean的实例，此为Spring的默认配置，全容器共享一个实例；

（2）Prototype：每次调用新建一个Bean的实例；

（3）Request：Web项目中，给每一个http request新建一个Bean实例；

（4）Session：Web项目中，给每一个http Session新建一个Bean实例；

```java
// Scope使用
@Service
@Scope("singleton/prototyp")
```



### Spring  IoC 概述





### 面向切面编程

关于数据库事务的重要约定：

- 当方法标注为 @Transactional 时，则方法启用数据库事务功能；
- 在默认情况下，如果原有方法出现异常则回滚整个事务；如果没有发生异常，那么就提交事务，这样整个事务管理AOP就完成了整个流程；
- 最后关闭数据库资源，AOP框架完成；

AOP 通过动态代理，带来管控各个对象操作的切面环境；Spring  AOP 是基于方法拦截的AOP；



### Spring 和 数据库编程

在 Spring 中数据库事务是通过 PlatformTransactionManager(接口) 进行管理的，能够支持事务的是 org.springframework.transactin.support.TransactionTemplate 模板，它是 Spring 所提供的事务管理器的模板；

最常用的数据库事务管理器：DataSourceTransactionManager，声明式事务 @Transactional；

@Transactional 注解可以配置事务的隔离级别和传播行为

- @Transactional 的底层实现是Spring AOP 技术，而Spring AOP 技术使用的是动态代理，即对于静态(static)方法和非public方法@Transactional注解是失效的；

- 隔离级别的默认值为： Isolation.DEFAULT，其含义是默认的，随数据库默认值的变化而变化；
- 传播行为的默认值为：REQUIRED；

事务的四种隔离级别：读未提交、读已提交、可重复读、序列化读；指两个事务的隔离程度；

事务的七种传播行为：传播行为是指**方法之间的调用事务策略**的问题。

| 传播行为        | 含义                                                         | 备注                                                         |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| required(*)     | 当方法调用时，如果不存在事务，那么就创建事务；<br />如果之前的方法已经存在事务了，那么就沿用之前的事务； | 这是Spring默认的传播行为                                     |
| supports        | 当方法调用时，如果不存在当前事务，那么不启用事务；<br />如果存在当前事务，那么就沿用当前事务； |                                                              |
| mandaroty       | 方法必须在事务内运行                                         | 如果不存在当前事务，那么就抛出异常                           |
| requires_new(*) | 无论是否存在当前事务，方法都会在新的事务中运行               | 事务管理器会打开新的事务运行该方法                           |
| not_supported   | 不支持事务，如果不存在当前事务也不会创建事务；如果存在当前事务，则挂起它，直至该方法结束后才恢复当前事务 | 适用于那些不需要事务的SQL                                    |
| never           | 不支持事务，只有在没有事务的环境中才能运行它                 | 如果方法存在当前事务，则抛出异常                             |
| nested(*)       | 嵌套事务，也就是调用方法如果抛出异常只回滚自己内部执行的SQL，而不回滚主方法的SQL | 它的实现存在两种清理，如果当前数据库支持保存点，就会在当前事务上使用保存点技术；如果发生异常则将方法内执行的SQL回滚到保存点上，而不是全部回滚。否则就等于requires_new 创建新的事务运行方法代码 |



## Spring  MVC

MVC:  Model  +  View  +  Controller

三层架构：Presentation  tier  +  Application  tier  +  Data tier (展现层 + 应用层 + 数据访问层)

实际上MVC只存在三层架构的展现层，M实际上是数据模型，是包含数据的对象。在Spring MVC里，有一个专门的类叫Model，用来和V之间的数据交互、传值；

三层架构是整个应用的架构，是由Spring框架负责管理的。一般项目结构中都有Service层、DAO层，这两个反馈在应用层和数据访问层。

![SpringMVC执行流程](D:\JAVA\Typora笔记\SpringMVC.assets\image-20201019223446532.png)

简要分析SpringMVC执行流程：

1、DispatcherServlt表示前置控制器，是整个SpringMVC的控制中心，用户发出请求，DispatcherServlet接收请求并拦截请求；

```shell
假设请求的URL为：http://localhost:8080/SpringMVC/hello
如上URL拆分为三部分：
http://localhost:8080服务器域名
SpringMVC部署在服务器上的web站点
hello表示控制器
通过分析，如上URL表示为：请求位于服务器localhost:8080上的SpringMVC站点的hello控制器
```

2、HandlerMapping为处理器映射。DispatcherServlet调用HandlerMapping，根据请求URL查找Handler；HandlerExecution表示具体的Handler，其主要作用是根据URL查找控制器，将解析后的信息传递给DispatcherServlet；

3、HandlerAdapter表示处理器适配器，其按特定的规则去执行Handler，Handler让具体的Controller执行(实现Controller接口的类)，Controller将具体的执行信息返回给HandlerAdapter，如ModelAndView；HandlerAdapter将视图逻辑名或模型传递给DispatcherServlet；(Controller中的代码已经执行完毕)

4、DispatcherServlet调用视图解析器(ViewResolver)来解析HandlerAdapter传递的逻辑视图名；视图解析器将解析的逻辑视图名传给DispatcherServlet，DispatcherServlet根据视图解析器解析的视图结果，调用具体的视图，最终视图呈现给用户；



### JSON 详解

JSON(JavaScript  Object  Notation, JS对象标记)是一种轻量级的数据交换格式，采用完全独立于编程语言的**文本格式**来存储和表示数据；JSON键值对是用来保存JavaScript对象的一种方式。



### Ajax技术

AJAX(Asynchronous JavaScript and XML，异步的JavaScript和XML)，AJAX是一种在无需重新加载整个网页的情况下，能够更新部分网页的技术；

Ajax是一种用于创建更好更快以及交互性更强的Web应用程序的技术；

**传统的网页**：想要更新内容或者提交一个表单，都需要重新加载整个网页；

**使用Ajax技术的网页**：通过在后台服务器进行少量的数据交换，就可以实现异步局部更新，使用Ajax用户可以创建接近本地桌面应用的直接、高可用、更丰富、更动态的Web用户界面；

```shell
jQuery.ajax(...)
	部分参数：
		url：请求地址
		type：请求方式，GET、POST(1.9.0之后用method)
		data：要发送的数据；格式：data:{"key","value"},K-V键值对
		success：成功之后执行的回调函数(全局)
		error：失败之后执行的回调函数(全局)
```



### 请求转发和重定向的区别

页面跳转的两种实现方式：请求转发ahead重定向；

**请求转发**：

客户端首先发送一个请求到服务器端，服务器端发现匹配的Servlet并制定它去执行，当这个servlet执行完之后，调用getRequestDispatcher() 方法，把请求转发给指定的student_list.jsp，整个流程都是在服务器端完成的，而且是在同一个请求里面完成的，因此servlet和jsp共享的是同一个request；整个过程是一个请求一个相应。

**重定向**：

客户发送一个请求到服务器，服务器匹配servlet，servlet处理完之后调用了sendRedirect()方法，立即向客户端返回这个响应，响应行告诉客户端你必须要再发送一个请求，去访问student_list.jsp，紧接着客户端收到这个请求后，立刻发出一个新的请求，去请求student_list.jsp,这里两个请求互不干扰，相互独立，在前面request里面setAttribute()的任何东西，在后面的request里面都获得不了。可见，在sendRedirect()里面是两个请求，两个响应。（服务器向浏览器发送一个302状态码以及一个location消息头，浏览器收到请求后会向再次根据重定向地址发出请求）

**区别**：

1、请求次数：重定向是浏览器向服务器发送一个请求并收到响应后再次向一个新地址发出请求，转发是服务器收到请求后为了完成响应跳转到一个新的地址；重定向至少请求两次，转发请求一次；

2、地址栏不同：重定向地址栏会发生变化，转发地址栏不会发生变化；

3、是否共享数据：重定向两次请求不共享数据，转发一次请求共享数据（在request级别使用信息共享，使用重定向必然出错）；

4、跳转限制：重定向可以跳转到任意URL，转发只能跳转本站点资源；

5、发生行为不同：重定向是客户端行为，转发是服务器端行为；



## SpringBoot

Spring  Boot 具有以下特征：

- 遵循“习惯优于配置”原则，使用Spring Boot 只需很少的配置，大部分时候可以使用默认配置；
- 项目快速搭建，可无配置整合第三方框架；
- 可完全不使用xml配置，只使用自动配置和 Java Config；
- 内嵌Servlet容器，应用可jar包运行；
- 运行中应用状态的监控；



### SpringBoot 自动配置原理

![image-20210227100936881](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210227100936881.png)



**Thymeleaf模板引擎**：是一个xml/xhtml/html5的模板引擎，可以作为MVC的Web应用的View层。

**SpringBoot内置servlet容器**：Tomcat；



## Spring Cloud

**微服务**：微服务是**系统架构上的一种设计风格**，它的主旨是将一个原本独立的系统拆分成多个小型服务，这些小型服务都在各自独立的进程中运行，服务之间通过基于HTTP的RESTFul  API进行通信协作。被拆分的每一个小型服务都围绕着系统中的某一项或一些耦合度较高的业务进行构建，并且每个服务都维护着自身的数据存储、业务开发、自动化测试案例以及独立部署机制。由于有了轻量级的通信协作基础，所以这些微服务可以使用不同的语言来编写。



### 服务治理：Spring Cloud Eureka

服务治理主要用来实现各个微服务实例的自动化注册与发现。

Eureka 的通信机制使用了HTTP的REST接口实现；

Eureka Server 的高可用：将自己作为服务向其他注册中心注册自己，形成一组互相注册的服务中心，实现服务清单的互相同步(请求转发实现)，达到高可用的效果。

Eureka 服务治理基础框架的三个核心要素：

- 服务注册中心：Eureka提供的服务端，提供服务注册与发现的功能；
- 服务提供者：提供服务的应用；
- 服务消费者：消费者应用从服务中心获取服务列表，从而消费者可以知道去何处调用其所需的服务。



### 客户端负载均衡：Spring  Cloud  Ribbon

Ribbon 是一个基于HTTP和TCP的客户端负载均衡工具，将面向服务的REST模板请求自动转换成客户端负载均衡的服务调用。

```shell
# 服务端负载均衡与客户端负载均衡
	# 服务端负载均衡
	服务端负载均衡分为硬件负载均衡和软件负载均衡。硬件负载均衡主要通过在服务节点之间安装专门用于负载均衡的设备如F5等；而软件负载均衡则是通过在服务器上安装一些具有均衡负载功能或模块的软件来完成请求分发工作。
	负载均衡的设备或软件模块维护一个下挂可用的服务端清单，通过心跳检测来剔除故障的服务端节点；当客户端发送请求到负载均衡设备时，设备按照某种算法从维护的可用服务清单中取出一台服务端的地址然后进行转发。
	# 客户端负载均衡
	客户端节点维护自己要访问的服务端清单，而服务端清单来自于服务注册中心。
```

Ribbon 通过 **RestTemplate**实现客户端负载均衡的；





### 服务容错保护：Spring  Cloud  Hystrix

在分布式架构中，当某个服务单元发生故障之后，通过断路器的故障监向调用方返回一个错误响应，而不是长时间的等待。这样就不会使得线程因调用故障服务被长时间占用不释放，避免了在分布式系统中的蔓延。

Hystrix 具备服务降级、服务熔断、线程和信号隔离、请求缓存、请求合并以及服务监控等强大功能。

当命令执行失败的时候，Hystrix 会进入 fallback 尝试回退处理，通常称该操作为“服务降级”。

Hystrix 实现线程隔离并加入熔断机制，以避免在微服务架构中因个别服务出现异常而引起级联故障蔓延。



### 声明式服务调用：Spring Cloud  Feign

Feign 整合了 Spring Cloud Ribbon 和 Spring  Cloud Hystrix；还提供了一种声明式的Web服务客户端定义方式。



### API网关服务：Spring Cloud Zuul

API网关是一个更为智能的应用服务器，它是整个微服务架构系统的门面，所有的外部客户端访问都要经过它来进行调度和过滤。它除了要实现请求路由、负载均衡、校验过滤等功能外，还需要服务治理框架的结合、请求转发时的熔断机制、服务的聚合等一系列高级功能。

Zuul：

- 对于路由规则与服务实例的维护问题，Zuul 将自身注册为Eureka服务治理下的应用，从Eureka中获得了所有其他微服务的实例信息。
- 对于签名校验、登录校验在微服务架构中的冗余问题，可以使用Zuul来创建各种校验过滤器。



## 分布式服务跟踪：Spring  Cloud  Sleuth

**与Zipkin整合**

Zipkin可以用来收集各个服务器上请求链路的跟踪数据，并通过它提供的REST API 接口来辅助查询跟踪数据以实现对分布式系统的监控程序，从而及时发现系统中出现的延迟高问题并找出系统性能瓶颈的根源。















## 高并发业务

**系统设计**：

高并发系统往往需要分布式的系统分摊请求的压力，这就需要使用负载均衡服务了，它进行建议判断后就会分发到具体Web服务器。

水平分法：服务器按业务划分，需要RPC调用服务。

垂直分法：将一个很大的请求量，不按子系统分，而是将它们按照互不相干的几个同样的系统分摊下去。

CDN(Content  Delivery Network，即内容分发网络)技术，允许企业将自己的静态数据缓存到网络CDN的节点中。



















