### 入门

要和Spring一起使用Mybatis，需要在Spring应用的上下文中定义至少两样东西：一个SqlSessionFactory和至少一个数据映射器类

配置工厂bean

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
</bean>
```

### SqlSessionFactoryBean

配置Mapper路径有两种方法，一种是手动在MyBatis的XML配置文件中使用<mappers>部分来指定类路径，第二是使用工厂bean的mapperLocation属性。

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
  <property name="mapperLocations" value="classpath*:sample/config/mappers/**/*.xml" />
</bean>
```

### 事务

要开启Spring的事务处理，在Spring的XML配置文件中创建一个DataSourceTransactionManager对象：

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <property name="dataSource" ref="dataSource" />
</bean>
```

### 使用SqlSession

使用MyBatis-Spring之后，你不需要直接使用SqlSessionFactory了，因为你的bean可以通过一个线程安全的SqlSession来注入，基于Spring的事务配置来自动提交，回滚，关闭session。

```java
public class UserDaoImpl implements UserDao {

  private SqlSession sqlSession;

  public void setSqlSession(SqlSession sqlSession) {
    this.sqlSession = sqlSession;
  }

  public User getUser(String userId) {
    return (User) sqlSession.selectOne("org.mybatis.spring.sample.mapper.UserMapper.getUser", userId);
  }
}
```

### 注入映射器

为了替代手工使用SqlSessionDaoSupport或SqlSessionTemplate编写Dao代码，My-Batis提供了一个动态代理实现：MapperFactoryBean，这个类可以让你直接注入数据映射器接口到你的service层bean中，当使用映射器的时候，你仅仅调用你的DAO一样调用它们就可以了，但是你不用编写任何DAO实现的代码，因为MyBatis-Spring会自动为你创建代理。

MapperFactoryBean

```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
  <property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

在service中以和注入任意Spring bean的相同方式直接注入映射器

```xml
<bean id="fooService" class="org.mybatis.spring.sample.mapper.FooServiceImpl">
  <property name="userMapper" ref="userMapper" />
</bean>
```

这个bean可以直接在service中使用

```Java
public class FooServiceImpl implements FooService {

  private UserMapper userMapper;

  public void setUserMapper(UserMapper userMapper) {
    this.userMapper = userMapper;
  }

  public User doSomeBusinessStuff(String userId) {
    return this.userMapper.getUser(userId);
  }
}
```

MapperScannerConfigurer

没必要在Spring的XML配置中注册所有的映射器，相反，你可以使用MapperScannerConfigurer，他会将在查找类路径下的映射器并自动将它们创建成MapperFactoryBean

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
  <property name="basePackage" value="org.mybatis.spring.sample.mapper" />
</bean>
```

