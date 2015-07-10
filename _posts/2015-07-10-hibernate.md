---
layout: post
title: Hibernate使用小札
category: WEB开发
tags: java
date: 2015-07-10
---
{% include JB/setup %}


* 目录
{:toc}

### 前言

Hibernate作为Java中最为流行的O/R映射框架，同时已经完全遵照JPA规范并作为其实现的一个超集，它能够帮助我们快速进行开发，从繁重的持久化层实现中脱离出来。本文将由浅入深为您带来一个粗略的Hibernate使用体验。

### Hibernate配置

Hinernate配置文件如下：

~~~XML
<!--标准的XML文件的起始行，version='1.0'表明XML的版本，encoding='gb2312'表 明XML文件的编码方式-->  
<!--表明解析本XML文件的DTD文档位置，DTD是DocumentType Definition 的缩写,  即文档类型的定义,
XML解析器使用DTD文档来检查XML文件的合法性。 /hibernate.sourceforge.net/hibernate-configuration-3.0dtd
可以在Hibernate3.1.3软件包中的src\org\hibernate目录中找到此文件-->
<?xml version='1.0' encoding='UTF-8'?>
<!DOCTYPE hibernate-configuration PUBLIC
          "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
          "http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">
<!--声明Hibernate配置文件的开始-->  
<hibernate-configuration>
<!--表明以下的配置是针对session-factory配置的，SessionFactory是 Hibernate中的一个类，
这个类主要负责保存HIbernate的配置信息，以及对Session的操作-->
    <session-factory>
        <!-- hibernate.dialect 只是Hibernate使用的数据库方言,就是要用Hibernate连接那种类型的数据库服务器 -->
        <property name="dialect">
            org.hibernate.dialect.MySQLDialect
        </property>
        
        <!-- 设置数据库的连接url:jdbc:mysql://localhost/hibernate,
        其中localhost表示mysql服务器名称，此处为本机，hibernate是数据库名 -->
        <property name="connection.url">
            jdbc:mysql://127.0.0.1:3306/HibernateTest
        </property>
        <!-- 连接数据库是用户名 -->
        <property name="connection.username">root</property>
        <!-- 连接数据库是密码 -->
        <property name="connection.password">123456</property>
        
        <!-- 配置数据库的驱动程序，Hibernate在连接数据库时，需要用到数据库的驱动程序 -->
        <property name="connection.driver_class">
            com.mysql.jdbc.Driver
        </property>
        <!-- jdbc.batch_size是指Hibernate批量插入,删除和更新时每次操作的记录数。BatchSize越大，
        批量操作的向数据库发送Sql的次数越少，速度就越快，同样耗用内存就越大 -->
        <property name="jdbc.batch_size">30</property>
        <!-- jdbc.use_scrollable_resultset是否允许Hibernate用JDBC的可滚动的结果集。
        对分页的结果集。对分页时的设置非常有帮助 -->
        <property name="jdbc.use_scrollable_resultset">false</property>
        <!-- jdbc.fetch_size是指Hibernate每次从数据库中取出并放到JDBC的Statement中的记录条数。
        FetchSize设的越大，读数据库的次数越少，速度越快，Fetch Size越小，读数据库的次数越多，速度越慢 -->
        <property name="jdbc.fetch_size">50</property>
        <!-- connection.useUnicode连接数据库时是否使用Unicode编码 -->
        <property name="connection.useUnicode">true</property>
        <!-- connection.characterEncoding连接数据库时数据的传输字符集编码方式，最好设置为gbk，用gb2312有的字符不全 -->
        <property name="connection.characterEncoding">gbk</property>
        <!-- 是否自动创建数据库表  他主要有一下几个值:validate:当sessionFactory创建时，
        自动验证或者schema定义导入数据库。  create:每次启动都drop掉原来的schema，创建新的。  create-drop:
        当sessionFactory明确关闭时，drop掉schema。update(常用):如果没有schema就创建，有就更新。 -->
        <property name="hbm2ddl.auto">create</property>
        <!-- 数据库连接池的大小 -->
        <property name="connection.pool_size">1</property>
        <!-- 配置此处 sessionFactory.getCurrentSession()可以完成一系列的工作，当调用时，  
        hibernate将session绑定到当前线程，事务结束后，hibernate将session从当前线程中释放，
        并且关闭session。当再次调用getCurrentSession()时，将得到一个新的session，并重新开始这一系列工作。 -->
        <property name="current_session_context_class">thread</property>
        <property name="cache.provider_class">org.hibernate.cache.NoCacheProvider</property>
        <property name="myeclipse.connection.profile">mysql</property>
        <!-- 是否在后台显示Hibernate用到的SQL语句，开发时设置为true，便于差错，程序运行时可以在Eclipse
        的控制台显示Hibernate的执行Sql语句。项目部署后可以设置为false，提高运行效率 -->
        <property name="show_sql">true</property>
        <!-- 指定映射文件为“om/cn/entity/User.hbm.xml -->
        <mapping resource="com/cn/entity/User.hbm.xml" />

    </session-factory>

</hibernate-configuration>
~~~

与Spring集成的时候，你也可以选择集成到ApplicationContext.xml中：

~~~XML
<bean id="sessionFactory"
		class="org.springframework.orm.hibernate4.LocalSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<property name="packagesToScan" value="sbeat.model*"></property>
		<property name="hibernateProperties">
			<props>
				<prop key="hibernate.dialect"> org.hibernate.dialect.MySQLDialect </prop>
				<prop key="hibernate.show_sql"> true </prop>
				<prop key="hibernate.hbm2ddl.auto">update</prop>
			</props>
		</property>
	</bean>
	<tx:annotation-driven /> 
	<bean id="transactionManager "
		class="org.springframework.orm.hibernate4.HibernateTransactionManager">
		<property name="sessionFactory" ref="sessionFactory"></property>
		<property name="dataSource" ref="dataSource" />
	</bean>
~~~

这里面有一个比较难以发现的坑，那就是Hibernate4中dispatcherServlet的配置竟然对Hibernate有影响，举例如下

~~~XML
<mvc:default-servlet-handler />
	<context:component-scan base-package="sbeat.controller" />
<mvc:annotation-driven />
~~~

上面是dispatcher-servlet配置的一部分，但是如果你把base-package改成sbeat，就会爆一个奇怪的错误：Hibernate Could not obtain transaction-synchronized Session for current thread.

你在网上搜得，大部分人会告诉你在Service上加@Transactional注解，或者让你在ApplicationContext中指定<tx:annotation-driven transactionManager="transactionManager"/> 。然而并没有什么卵用，至于为什么servlet配置会导致这个问题，小生也没有弄清楚。


### Hibernate 映射关系

首先让我们给出一个映射的例子

~~~java
@Entity
@Table(name = "Resume")
public class Resume {
	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	@Column(name = "id", nullable = false)
	private Long id;

	@OneToOne(cascade = CascadeType.MERGE, fetch = FetchType.LAZY)
	@JoinColumn
	private User user;//对应用户，仅同步更新
	
	@Column(name="created",columnDefinition="DATETIME default '2015-07-10 22:07:00'",nullable=false)
	private Date created= new Date();//创建日期
	
	@ElementCollection
	private List<String> tags;//雇员技能标签

	@Enumerated(EnumType.STRING)
	@Column(name = "sex", nullable = false, columnDefinition = "VARCHAR(10) default 'UNKNOWN' ")
	private Sex sex=Sex.UNKNOWN;

	@Column(name = "description", length = 30)
	private String description;

	@Column(name = "degree", nullable = false, columnDefinition = "int default 0")
	private Integer degree=0;//学历，由低到高

	@Column(name = "education")
	private String education;//教育经历

	@Column(name = "experience")
	private String experience;//工作经历

	@Column(name = "hobbies")
	private String hobbies;//兴趣爱好
~~~

上面省略了getter/setter，下面主要讲几个重点。

#### 外键

~~~java
@ManyToOne(targetEntity = Bar.class, fetch = FetchType.LAZY)
@JoinColumns( {
    @JoinColumn(name = "column_1", referencedColumnName = "column_1"),
    @JoinColumn(name = "column_2", referencedColumnName = "column_2"),
    @JoinColumn(name = "column_3", referencedColumnName = "column_3"),
    @JoinColumn(name = "column_4", referencedColumnName = "column_4")
})
~~~

#### Collection
Hibernate对于Collection型的属性，会默认采用一对多映射，采用另一张表来记录，其查询也按照一对多映射的规律

#### 级联方式

级联方式有以下几种

- PERSIST 插入时帮你插入
- MERGE 更新时帮你更新
- REFRESH 刷新时帮你刷新
- REMOVE 删除时顺带帮你删了
- DETACH 无视外键依赖直接就可以删除，比如上文如果设置了此选项，可以不管Resume删掉User
- ALL 以上所有


#### 动态插入和动态更新

Hibernate默认的插入和更新是讲所有属性值刷一遍，所以数据库中的default值不会起作用，变为null，用SQL来说明，就是所有属性都出现在SQL语句中。

想要使用数据库的默认值，可以使用注解，如下

~~~java
@Entity
@Table(name = "Resume")
@DynamicInsert(true)
@DynamicUpdate(true)
~~~

但经本人测试好像这两个注解并不起作用，我的版本是4.3.10.Final，另外动态更新的效果是与merge一致，只写有变化的属性，如果你的属性为null，更新后依然会变成null。



### Hibernate 操作

#### merge，save，update，persist

- merge在存之前先会做一次查找，最后只修改有变化的属性
- update就把所有属性都写了一遍，会造成锁的繁忙
- persist是讲缓存中游离的对象持久化，对于已经persist的对象即使再次persist也不会有两行
- save 对应数据库的一次插入行为，对于同一个引用对象，persist后再save，会生成两行，persist是考虑到缓存中的引用，同一个引用只有一行。


#### 查询

##### 按Id查询

load/get(Class<T>, id)

- load 和缓存有关，未找到抛出异常
- get 直接从数据库中读取，未找到返回null

##### 交集查询

举例如下

~~~java
Criteria cr=getCurrentSession().createCriteria(clazz);
cr.add(Restrictions.eq(key,val));
cr.add(Restrictions.sizeGe(key,val));
return cr.list();
~~~

##### 多条件并集查询

举例如下

~~~
Criteria cr=getCurrentSession().createCriteria(clazz);
Disjunction disjunction=Restrictions.disjunction();
disjunction.add(Restrictions.like("nickName", keyword));
disjunction.add(Restrictions.like("description", keyword));
cr.add(disjunction);
~~~

##### 外键查询

对于外键本身

~~~java
public Resume findByUserId(Long id) {
	Criteria cr=getCurrentSession().createCriteria(Resume.class);
	cr.add(Restrictions.eq("user.id",id));
	return (Resume) cr.list().get(0);
}
~~~

对于外键对象的属性

~~~java
public Resume findByAttr(Attr attr) {
	Criteria cr=getCurrentSession().createCriteria(Resume.class);
	cr.createAlias("user","u");
	cr.add(Restrictions.eq("u.attr",attr));
	return cr.list();
}
~~~

##### Collection查询

同外键查询一样，举例如下

~~~java
cr.createAlias("tags", "t");
cr.add(Restrictions.eq("t", tag);
~~~


##### 分页

举例如下

~~~java
cr.setFirstResult((pageNum-1)*SbeatConfig.PAGE_SIZE);
cr.setMaxResults(SbeatConfig.PAGE_SIZE);
~~~

##### 排序

~~~java
cr.addOrder(Order.desc("created"));
~~~

#### 刷新

由于Hibernate借助缓存，所以对象在数据库中改变后并不会立即在取的时候改变，这时就需要刷新操作。

~~~java
session.refresh(t);
~~~


#### HQL 语句

~~~java
String hql = "UPDATE Employee set salary = :salary "  + 
             "WHERE id = :employee_id";
Query query = session.createQuery(hql);
query.setParameter("salary", 1000);
query.setParameter("employee_id", 10);
int result = query.executeUpdate();
~~~

### Hibernate CRUD

CRUD操作是十分常用的操作，我们自然不会为每个DAO实现一遍，所以是时候采用泛型和反射了，下面给出一种实现方案。

~~~java

//首先给出泛化的接口
public interface GenericDAO<T> {
	public void insert(T t);
	
	public void update(T t);
	
	public void simpleUpdate(Map<Object, Object> delta);
	
	public void delete(T t);
	
	public T load(Long id);
	
	public void refresh(T t);
	
	public List<T> list(int pageNum);
	
	public List<T> findByCondition(Map<Object,Object> condition);

}

//下面是给出上述接口的实现

@SuppressWarnings("unchecked")
public class GenericDAOImpl<T> implements GenericDAO<T> {

	private Class<T> clazz;
	
	@Autowired
	protected SessionFactory sessionFactory;
	
	protected Session getCurrentSession() {
		return sessionFactory.getCurrentSession();
	}
	
	public GenericDAOImpl(){
		//取得T的类型变量
		ParameterizedType type = (ParameterizedType) this.getClass().getGenericSuperclass();
        clazz = (Class<T>) type.getActualTypeArguments()[0];
	}
	
	public void insert(T t) {
		getCurrentSession().persist(t);
	}

	public void update(T t) {
		getCurrentSession().update(t);
	}

	//该方法以单域作为选择的基准，使用Map作增量更新
	public void simpleUpdate(Map<Object, Object> delta) {
		Field[] fields=clazz.getDeclaredFields();
		String hql="UPDATE "+clazz.getName()+" set";
		
		String keyName="id";//默认选择基准为Id
		
		//调整选择域
		if(delta.containsKey("keyName")) keyName=(String) delta.get("keyName");
		
		for(Field field:fields){
			String fieldName=field.getName();
			if(fieldName==keyName) continue;
			if(delta.containsKey(fieldName)){
				hql=hql+" "+fieldName+"=:"+fieldName+", ";
			}
		}
		
		
		hql=hql.substring(0,hql.lastIndexOf(","))+" where "+keyName+"=:"+keyName;
		Query query=getCurrentSession().createQuery(hql);
		
		for(Field field:fields){
			String fieldName=field.getName();
			if(delta.containsKey(fieldName)){
				query.setParameter(fieldName, delta.get(fieldName));
			}
		}
		
		query.executeUpdate();
	}
	
	public void delete(T t) {
		getCurrentSession().delete(t);
	}
	
	public T load(Long id) {
		return (T) getCurrentSession().get(clazz, id);
	}
	
	
	public List<T> list(int pageNum) {
		//分页查找所有
		Session session=getCurrentSession();
		Query query=session.createQuery("from "+clazz.getName()+" order by created desc");
		if(pageNum>0){
			query.setFirstResult((pageNum-1)*SbeatConfig.PAGE_SIZE);
			query.setMaxResults(SbeatConfig.PAGE_SIZE);
		}
		return query.list();
	}
	
	public List<T> findByCondition( Map<Object, Object> condition) {
		//交集分页查找并排序
		Field[] fields=clazz.getDeclaredFields();
		Criteria cr=getCurrentSession().createCriteria(clazz);
		for(Field field:fields){
			String fieldName=field.getName();
			if(condition.containsKey(fieldName)){
				cr.add(Restrictions.eq(fieldName, condition.get(fieldName)));
			}
		}
		//以插入数据库的倒序来排列
		cr.addOrder(Order.desc("created"));
		Integer pageNum=0;
		if(condition.containsKey(pageNum)){
			pageNum=(Integer) condition.get("pageNum");
		}
		if(pageNum>0){
			cr.setFirstResult((pageNum-1)*SbeatConfig.PAGE_SIZE);
			cr.setMaxResults(SbeatConfig.PAGE_SIZE);
		}
		return cr.list();
	}

	public void refresh(T t) {
		getCurrentSession().refresh(t);
	}

}

~~~

