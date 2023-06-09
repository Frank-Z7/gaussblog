+++

title = "【我与openGauss的故事】SSM+MySQL替换探索 opengauss3对比postgresql12"

date = "2022-10-8" 

tags = ["openGauss技术文章征集"]

archives = "2022-10"

author = "大数据模型"

summary = "开发人员 习惯用SSM+mysql，现在有一个选择，可以使用SSM+openGauss"

+++


## SSM介绍


SSM（Spring+SpringMVC+MyBatis）框架集由Spring、MyBatis两个开源框架整合而成（SpringMVC是Spring中的部分内容），常作为数据源较简单的web项目的框架。

### Spring

Spring就像是整个项目中装配bean的大工厂，在配置文件中可以指定使用特定的参数去调用实体类的构造方法来实例化对象。也可以称之为项目中的粘合剂。
Spring的核心思想是IoC（控制反转），即不再需要程序员去显式地`new`一个对象，而是让Spring框架帮你来完成这一切。

### SpringMVC

SpringMVC在项目中拦截用户请求，它的核心Servlet即DispatcherServlet承担中介或是前台这样的职责，将用户请求通过HandlerMapping去匹配Controller，Controller就是具体对应请求所执行的操作。SpringMVC相当于SSH框架中struts。

### mybatis

mybatis是对jdbc的封装，它让数据库底层操作变的透明。mybatis的操作都是围绕一个sqlSessionFactory实例展开的。mybatis通过配置文件关联到各实体类的Mapper文件，Mapper文件中配置了每个类对数据库所需进行的sql语句映射。在每次与数据库交互时，通过sqlSessionFactory拿到一个sqlSession，再执行sql命令。
页面发送请求给控制器，控制器调用业务层处理逻辑，逻辑层向持久层发送请求，持久层与数据库交互，后将结果返回给业务层，业务层将处理逻辑发送给控制器，控制器再调用视图展现数据。 


## SSM搭配的数据库

无庸置疑，SSM经常搭配的数据库是MySQL或者是Postgresql，而国内使用MySQL的人比 Postgresql的人多，  所以本文主要内容关于SSM  DEMO已经搭配用上MySQL，系统本身可以注册写入数据入库，可以从库中读取数据进行登录。


## SSM  demo代码

经过测试对比，SSM DEMO代码基于三个不同的数据库，除了JDBC连接串不同，如下。
![image.png](images/20220930-7b410e59-bc08-4396-a799-9b3b06defc4b.png)

另外最大的不同就是数据库的ID自增机制不同，ID自增机制是数据库的一项基本功能，我们非常重视这一点，这个不同会不会导致DEMO代码要做相关的适配，好听叫做适配，不好听叫做业务侵入。

我们现在开发的 SSM  DEMO代码是在MySQL的基础上开发，优先满足了MySQL。


### Postgresql  

如果把MySQL改换成Postgresql  ，运行程序的会报错，如下

```bash
org.springframework.web.util.NestedServletException: Request processing failed; nested exception is 
org.springframework.jdbc.UncategorizedSQLException: 
### Error updating database.  Cause: org.postgresql.util.PSQLException: Returning autogenerated keys is 
only supported for 8.2 and later servers.
### SQL: insert into user1 (username, password, age)     values (?, ?, ?)
### Cause: org.postgresql.util.PSQLException: Returning autogenerated keys
 is only supported for 8.2 and later servers.

```


笔者用的是postgresql12，经过网上查阅 ，mybatis的定义配置必须要更改。

```properties
  <insert id="insert" useGeneratedKeys="true" keyProperty="id" parameterType="com.cn.uuu.pojo.User1" >

```

==useGeneratedKeys="true" keyProperty="id"摘掉==改换成

```properties
  <insert id="insert"  parameterType="com.cn.uuu.pojo.User1" >

```



**重新在idea运行tomcat9**


![输入图片说明](images/20220930-8253a3d9-0fb9-4b92-a568-a958855d0dc9.png)


如下，发现数据能够写入了，但是写入的数据没有id列为空值。

```bash
mytest=# select * from user1;
 id | username  | password  | age 
----+-----------+-----------+-----
  1 | user1     | password1 |  18
    | hexin.xue | cxc       |  25

```



**这是由于postgresql没有自动增加id机制的功能，所以数值一直为空**。 如果要实现自增ID,它是通过sequence去实现增加ID的。

两个方法是给相关表增加sequence，一种是已建表的基础上增加sequence实现增加ID

```sql
create sequence public.userid_seq start with 1 increment by 1 no minvalue no maxvalue cache 1;
alter sequence public.userid_seq owner to henley;
alter table user1 alter column id set default nextval('public.userid_seq');
insert into user1 (username, password, age)     values ('username1', 'password1', 100);

```


另外一种重建表，建表就实现

```sql
CREATE SEQUENCE sq_user_id  START 1   INCREMENT 1  CACHE 20;
create table  user1(id int NOT NULL DEFAULT nextval('sq_user_id') ,
username  varchar(50),password  varchar(50),age  int);
```



建表后如果还有问题 ，下面再补刀

```sql
alter sequence public.sq_user_id owner to henley;
alter table user1 alter column id set default nextval('public.sq_user_id');

```

增加sequence后，写入数据后ID例有值了。


### OpenGauss

依然是同样代码，把jdbc的连接串改成OpenGauss，   我们启动tomcat，加载服务，我们惊喜的发现没有报错。
![image.png](images/20220930-84c4baf5-d181-426e-a889-57b889a2ece8.png)

```properties
  <insert id="insert" useGeneratedKeys="true" keyProperty="id" parameterType="com.cn.uuu.pojo.User1" > 
```

此处在postgresql12需要更改，在opengauss3不需要任何更改，spring指向的应用层没有报错。


但是数据库底层ID列的数据仍然空值，OpenGauss也和Postgresql一样，都是用sequence 去实现ID的自增长。

```sql
create sequence public.userid_seq start with 1 increment by 1
 no minvalue no maxvalue cache 1;
alter sequence public.userid_seq owner to henley;
alter table user1 alter column id set default nextval('public.userid_seq');

```





## 最后总结 

这是SSM开发中比较低端的DEMO代码，但是从中可见openGauss用心的包容,在接口层适配了mybatis,遇到自增ID提高了容错性，相对于postgresql多了一道方便。


## 体验源代码

https://gitee.com/angryart/ssm-opengauss