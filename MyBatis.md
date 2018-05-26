### 安装

使用XML构建SqlSessionFactory

每个基于MyBatis的应用都是以一个`SqlSessionFactory`的实例为中心的。

而`SqlSessionFactory`的实例可以通过`SqlSessionFactoryBuilder`获得。

SQLSessionFactoryBuilder则可以通过XML配置文件或者一个预先定制的Configuration的实例构建

* 使用XML文件

  ```java
  String resource = "org/mybatis/example/mybatis-config.xml";
  InputStream inputStream = Resources.getResourceAsStream(resource);
  SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
  ```

  XML配置文件

  内容：MyBatis系统的核心设置、获取数据库连接实例的数据源、决定事务作用域和控制方式的事务管理器。

  ```xml
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE configuration
    PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">
  <configuration>
    <environments default="development">
      <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
          <property name="driver" value="${driver}"/>
          <property name="url" value="${url}"/>
          <property name="username" value="${username}"/>
          <property name="password" value="${password}"/>
        </dataSource>
      </environment>
    </environments>
    <mappers>
      <mapper resource="org/mybatis/example/BlogMapper.xml"/>
    </mappers>
  </configuration>
  ```

* 使用Java构建SqlSessionFactory

  ```Java
  DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
  TransactionFactory transactionFactory = new JdbcTransactionFactory();
  Environment environment = new Environment("development", transactionFactory, dataSource);
  Configuration configuration = new Configuration(environment);
  configuration.addMapper(BlogMapper.class); 
  SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
  ```

  configuration添加了一个映射器类，包含SQL映射语句的注解从而避免XML文件的依赖。

* 获取SqlSession

  ```java
  SqlSession session = sqlSessionFactory.openSession();
  try {
    BlogMapper mapper = session.getMapper(BlogMapper.class);
    Blog blog = mapper.selectBlog(101);
  } finally {
    session.close();
  }
  ```

  XML定义：

  ```xml
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
  <mapper namespace="org.mybatis.example.BlogMapper">
    <select id="selectBlog" resultType="Blog">
      select * from Blog where id = #{id}
    </select>
  </mapper>
  ```

  在命名空间`org.mybatis.example.BlogMapper`中定义一个`selectBlog`的映射语句

  Java注解定义：

  ```Java
  package org.mybatis.example;
  public interface BlogMapper {
    @Select("SELECT * FROM blog WHERE id = #{id}")
    Blog selectBlog(int id);
  }
  ```

  > 对于简单的语句可以使用注解，复杂的语句最好还是使用XML映射语句

#### 作用域和生命周期

SqlSessionFactoryBuilder：创建使用后就不需要了，因此最佳是方法作用域

SqlSessionFactory：一旦创建后就在应用的运行期间一直存在。

SqlSession：每个线程都有一个SqlSession实例，在web中使用SqlSession：每次接收HTTP请求，可以打开一个SqlSession返回一个响应，就关闭它。

映射器实例：由于是从SqlSession中创建，最好与SqlSession保持一致

### Mapper XML 文件

#### select

```xml
<select id="selectPerson" parameterType="int" resultType="hashmap">
  SELECT * FROM PERSON WHERE ID = #{id}
</select>
```

接收一个int类型的参数，返回一个hashmap类型对象

#### sql

这个元素可以被用来定义可以重用的SQL代码段

#### Result  Map

由resultType属性指定

```xml
<!-- In mybatis-config.xml file -->
<typeAlias type="com.someapp.model.User" alias="User"/>

<!-- In SQL Mapping XML file -->
<select id="selectUsers" resultType="User">
  select id, username, hashedPassword
  from some_table
  where id = #{id}
</select>
```

MyBatis会在幕后自动创建一个ResultMap，再基于属性名来映射列到JavaBean的属性上，如果没有精确匹配，可以在sql语句中使用别名

方法1：

```xml
<select id="selectUsers" resultType="User">
  select
    user_id             as "id",
    user_name           as "userName",
    hashed_password     as "hashedPassword"
  from some_table
  where id = #{id}
</select>
```

方法2：

```xml
<resultMap id="userResultMap" type="User">
  <id property="id" column="user_id" />
  <result property="username" column="user_name"/>
  <result property="password" column="hashed_password"/>
</resultMap>
```

引用它的语句使用resultMap属性

```xml
<select id="selectUsers" resultMap="userResultMap">
  select user_id, user_name, hashed_password
  from some_table
  where id = #{id}
</select>
```

### 动态SQL

#### if

```xml
<select id="findActiveBlogWithTitleLike"
     resultType="Blog">
  SELECT * FROM BLOG 
  WHERE state = ‘ACTIVE’ 
  <if test="title != null">
    AND title like #{title}
  </if>
</select>
```

假设没有传入title，则会返回所有处于“ACTIVE”的BLOG，反之，则加上title的模糊搜索

```xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’ 
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
```

#### choose，when，otherwise

有时候我们会有类似于Java中的switch操作，if是多选，而switch是单选

```xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <choose>
    <when test="title != null">
      AND title like #{title}
    </when>
    <when test="author != null and author.name != null">
      AND author_name like #{author.name}
    </when>
    <otherwise>
      AND featured = 1
    </otherwise>
  </choose>
</select>
```

otherwise相当于default，即前面所有的条件都不满足的情况下执行。

#### trim，where，set

有时候假设在where中所有的条件都是动态的时候可能最后拼接出来的SQL语句为

```sql
SELECT * FROM BLOG 
WHERE 
```

这时候可以使用where闭合标签

```xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG 
  <where> 
    <if test="state != null">
         state = #{state}
    </if> 
    <if test="title != null">
        AND title like #{title}
    </if>
    <if test="author != null and author.name != null">
        AND author_name like #{author.name}
    </if>
  </where>
</select>	
```

where标签会智能检测何时插入where，而且若语句的开头为AND或OR，where元素也会将其去掉

trim为where的详细定义版

```xml
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  ...
</trim>
```

同样的在update中使用set的时候也会与类似的情况

```xml
<update id="updateAuthorIfNecessary">
  update Author
    <set>
      <if test="username != null">username=#{username},</if>
      <if test="password != null">password=#{password},</if>
      <if test="email != null">email=#{email},</if>
      <if test="bio != null">bio=#{bio}</if>
    </set>
  where id=#{id}
</update>
```

set会前置set关键字，并且删除无关的符号

#### foreach

```xml
<select id="selectPostIn" resultType="domain.blog.Post">
  SELECT *
  FROM POST P
  WHERE ID in
  <foreach item="item" index="index" collection="list"
      open="(" separator="," close=")">
        #{item}
  </foreach>
</select>
```

collection表示你将要迭代的集合，index代表元素索引，item代表每项元素，还可以自定义开闭和分隔符号。