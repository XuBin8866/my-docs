# 自实现MyBatis框架

此框架实现了读取mapper.xml文件通过接口方法的代理 进行增删改查操作和直接调用Session的方法传入封装好数据的实体对象进行增删改操作的功能。框架的包名和类名大多和MyBatis下的文件采用相同的命名，框架的执行流程和使用方法和MyBatis类似，算是我前一篇所写的简易MyBatis框架的完善版本。前一篇文章中有对简易框架的执行流程分析，有需要可以参考下：
[手写MyBatis框架——按执行流程编写](https://blog.csdn.net/weixin_44804750/article/details/105496683)

## 一、框架介绍
自实现的简易MyBatis框架，通过构建者读取配置文件完成对sqlSession工厂的创建，sqlSession类中提供所有的CRUD方法，而CRUD方法调用Executor类中的方法实现对数据库的操作，目前使用的数据库连接池为自定义连接池，目前还不支持由外部提供连接池。使用接口方法读取mapper.xml文件进行CRUD操作时会生成该接口的代理实现类，通过动态代理来调用sqlSession中的方法；也可以直接调用sqlSession的增删改方法，将需要传入的sql语句中的参数值封装到所操作表对应的实体类对象中，将该对象传入方法中，实现对数据库的操作。
#### 1.目录结构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200423183201442.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDgwNDc1MA==,size_16,color_FFFFFF,t_70#pic_center)
#### 2.框架分析
+ 2.1 bean
ColumnInfo和TableInfo是用于封装数据库的字段信息和表信息的实体类
+ 2.2 binding
MapperProxy代理类、 MapperProxyFactory代理工厂、MapperRegistry代理工厂注册器。代理工厂注册器在session工厂初始化时将读取指定包下的所有mapper.xml文件并为每一个xml文件注册其对应接口的代理工厂。
+ 2.3 callback
MyCallback回调接口，用于在执行数据库操作时提供模板方法以简化代码。
+ 2.4 constant
代码中需要使用到的一些静态常量，避免在代码中出现魔法值
+ 2.5 executor
和数据库操作有关的所有类，包括sql参数处理，sql语句处理，sql结果处理以及执行器
+ 2.6 pool
包含自实现的单例数据库连接池及其接口类
+ 2.7 session
SqlSessionFactoryBuilder构建者、SqlSessionFactory工厂类及其实现类。SqlSession及其实现类为进行进行数据库操作的入口，Configuration类存储了properties文件信息、数据库连接池对象、代理注册器、表与实体类的映射关系等所有接下来操作需要用到的信息。
+ 2.8 utils
包含XmlParseUtils解析xml文件、StringUtils字符串格式转换、ReflectUtils反射调用方法、CommonUtils非空判断的工具类
## 二、详细分析
分析代码只显示类中部分代码，具体代码请到GitHub查看，分析顺序按我编写代码的思路。
#### 1.框架的使用方式
###### 1.1 使用接口代理类
```java
 public void testMain() {
        //构建sql工厂
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build("conf.properties");
        SqlSession session = factory.openSession();
        UserMapper userMapper = session.getMapper(UserMapper.class);
        System.out.println(userMapper.getAll());
        System.out.println("update：" + userMapper.updateUser("xxbb", 1));
        System.out.println("insert:" + userMapper.insertUser(24, "zzxx", "123456", 1));
        System.out.println("delete: " + userMapper.deleteUser(24));
    }
```
###### 1.2 直接调用session的方法（增删改方法）
```java
public void testMain() {
	    SqlSessionFactory factory = new SqlSessionFactoryBuilder().build("conf.properties");
        SqlSession session = factory.openSession();
        User u = new User();
        u.setId(24);
        u.setUsername("zzxx");
        u.setPassword("123456");
        u.setIfFreeze(1);
        System.out.println("testInsert：" + session.insert(u));
        System.out.println("testUpdate：" + session.update(u));
        System.out.println("testDelete:" + session.delete(u));
    }
```

#### 2.代码分析
###### 2.1 Constant类
存储字符串常量，配置文件使用的时properties文件，配置文件内所有的key在该类中都有一个字符串常量来表示，在读取配置文件信息时也使用Constant类中的常量作为key来获取。对于一些逻辑代码需要用到的字符串常量也在此处定义
Constant.java:
```java
/**
 * 配置文件和标签的名称常量
 *
 * @author xxbb
 */
public interface Constant {
    /**
     * 编码格式
     */
    String CHARSET_UTF8 = "UTF-8";

    //读取数据库配置文件中的信息
    /**
     * mapper.xml所在的包
     */
    String MAPPER_LOCATION = "mapper.location";
    /**
     * 与数据库表对应的po类所在的包
     */
    String PO_LOCATION = "po.location";
    /**
     * 所访问的数据库名
     */
    String CATALOG = "catalog";
    /**
     * 数据库连接驱动
     */
    String JDBC_DRIVER = "jdbc.driver";
    /**
     * 数据库连接url
     */
    String JDBC_URL = "jdbc.url";
    /**
     * 登录用户名
     */
    String JDBC_USERNAME = "jdbc.username";
    /**
     * 密码
     */
    String JDBC_PASSWORD = "jdbc.password";
    /**
     * 初始化连接个数
     */
    String JDBC_INIT_COUNT = "jdbc.initCount";
    /**
     * 最小连接个数
     */
    String JDBC_MIN_COUNT = "jdbc.minCount";
    /**
     * 最大连接个数
     */
    String JDBC_MAX_COUNT = "jdbc.maxCount";
    /**
     * 连接增长步长
     */
    String JDBC_INCREASING_COUNT = "jdbc.increasingCount";
}
```
conf.properties:
```
####mapper接口所在的包
mapper.location=test.mapper
####与数据库表对应的po类所在的包
po.location=test.domain
####访问的数据库名
catalog=db_orm
####数据连接参数
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/db_orm?useSSL=false&serverTimezone=UTC&characterEncoding=utf-8&allowPublicKeyRetrieval=true
jdbc.username=root
jdbc.password=123456
#初始化连接池个数
jdbc.initCount=8
#最小连接池个数
jdbc.minCount=8
#最大连接池个数
jdbc.maxCount=20
#连接池增长步长
jdbc.increasingCount=2
```
如上代码中如果我将<code>jdbc.driver</code>替换成了<code>driver</code>，只需要在Constanct类中修改对应常量的值，实现了与逻辑代码的解耦。

###### 2.2 MappedStatement类
封装了mapper.xml中sql语句的相关消息
```java
/**
 * 封装mapper.xml映射文件中的信息，一个对象封装一条sql语句相关信息
 *
 * @author xxbb
 */
public class MappedStatement {
    /**
     * mapper文件的namespace命名空间
     */
    private String namespace;
    /**
     * sql标签的id，和namespace一起组成唯一的标记
     */
    private String id;
    /**
     * sql标签的的类型：select/insert/update/delete
     */
    private String sqlType;
    /**
     * 查询结果的返回值类型
     */
    private String returnType;
    /**
     * sql语句
     */
    private String sql;
}
```
###### 2.3 数据库连接池类
首先创建了一个连接池类接口，继承了jdk提供的DataSource,再实现该接口，使我自定义的连接池成为DataSource的子类。连接池接口类全部使用default实现DataSourcede方法，使连接池实现类的代码更加简洁。在我上一篇文章也是这样写的。
连接池实现类使用双重检测锁实现单例，获取连接和归还连接之间通过wati()和notify()相互唤醒。这里通过反射破坏它的单例模式，详细可见构造方法中的注释，该类无法进行序列化操作，在继承Serializable后进行序列化操作会报错
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200423231747157.png)
如果能进行序列化和反序列化的单例类，可以在类中添加如下方法，可保证序列化和反序列化取到的是同一个类
```java
/**
     * 提供readResolve()方法
     *  当JVM反序列化地恢复一个新对象时，
     *  系统会自动调用这个readResolve()方法返回指定好的对象，
     *  从而保证系统通过反序列化机制不会产生多个java对象
     * @return 单例对象
     * @throws ObjectStreamException 对象流异常
     */
    private Object readResolve()throws ObjectStreamException {
        return instance;
    }
```
MyDataSourceImpl.java:
```java
package com.xxbb.smybatis.pool;

import com.xxbb.smybatis.constants.Constant;
import com.xxbb.smybatis.session.Configuration;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.util.LinkedList;

/**
 * 数据库连接池
 *
 * @author xxbb
 */
public class MyDataSourceImpl implements MyDataSource {
    /**
     * 数据库连接属性
     */
    private static final String DRIVER;
    private static final String URL;
    private static final String USERNAME;
    private static final String PASSWORD;
    /**
     * 初始连接数量
     */
    private static int initCount = 5;
    /**
     * 最小连接数量
     */
    private static int minCount = 5;
    /**
     * 最大连接数量
     */
    private static int maxCount = 20;
    /**
     * 已创建的连接数量
     */
    private static int createdCount;
    /**
     * 连接数增长步长
     */
    private static int increasingCount = 2;
    /**
     * 存储配置文件信息
     */
    private static Configuration configuration;
    /**
     * 存储连接的集合
     */
    private LinkedList<Connection> conns = new LinkedList<>();
    /**
     * 用于获取连接和归还连接的同步锁对象
     */
    private static final Object MONITOR = new Object();

    //属性初始化
    static {
        DRIVER = Configuration.getProperty(Constant.JDBC_DRIVER);
        URL = Configuration.getProperty(Constant.JDBC_URL);
        USERNAME = Configuration.getProperty(Constant.JDBC_USERNAME);
        PASSWORD = Configuration.getProperty(Constant.JDBC_PASSWORD);

        try {
            initCount = Integer.parseInt(Configuration.getProperty(Constant.JDBC_INIT_COUNT));
        } catch (Exception e) {
            System.out.println("initCount使用默认值：" + initCount);
        }
        try {
            minCount = Integer.parseInt(Configuration.getProperty(Constant.JDBC_MIN_COUNT));
        } catch (Exception e) {
            System.out.println("minCount使用默认值：" + minCount);
        }
        try {
            maxCount = Integer.parseInt(Configuration.getProperty(Constant.JDBC_MAX_COUNT));
        } catch (Exception e) {
            System.out.println("maxCount使用默认值：" + maxCount);
        }
        try {
            increasingCount = Integer.parseInt(Configuration.getProperty(Constant.JDBC_INCREASING_COUNT));
        } catch (Exception e) {
            System.out.println("increasingCount使用默认值：" + increasingCount);
        }

    }

    /**
     * 连接池对象
     */
    private static volatile MyDataSourceImpl instance;

    private MyDataSourceImpl() {
        //防止反射破坏单例
        //防止反射通过反射实例化对象而跳过getInstance方法
        //只能在已通过getInstance方法创建好对象后起作用
        //如果一开始就使用反射创建对象的话，由于instance对象并没有被实例化，所以能够一直用反射创建对象
        //要想使用反射创建必须满足instance对象为空，Configuration类中已经加载了配置文件
        if (instance != null) {
            throw new RuntimeException("Object has been instanced,please do not create Object by Reflect!!!");
        }
        init();
    }

    public static MyDataSourceImpl getInstance() {
        //双重检测锁
        if (null == instance) {
            synchronized (MyDataSourceImpl.class) {
                if (null == instance) {
                    instance = new MyDataSourceImpl();
                }
            }
        }
        return instance;
    }

    /**
     * 初始化连接池
     */
    private void init() {
        //循环给集合中添加初始化连接
        for (int i = 0; i < initCount; i++) {
            boolean flag = conns.add(createConnection());
            if (flag) {
                createdCount++;
            }
        }
        System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "连接池连接初始化----->连接池对象：" + this);
        System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "连接池连接初始化----->连接池可用连接数量：" + createdCount);

    }

    /**
     * 构建数据库连接对象
     *
     * @return 连接
     */
    private Connection createConnection() {
        try {
            Class.forName(DRIVER);
            return DriverManager.getConnection(URL, USERNAME, PASSWORD);

        } catch (Exception e) {
            throw new RuntimeException("数据库连接创建失败：" + e.getMessage());
        }
    }

    /**
     * 连接自动增长
     */
    private synchronized void autoAdd() {
        //增长步长默认为2
        if (createdCount == maxCount) {
            throw new RuntimeException("连接池中连接已达最大数量,无法再次创建连接");
        }
        //临界时判断增长个数
        for (int i = 0; i < increasingCount; i++) {
            if (createdCount == maxCount) {
                break;
            }
            conns.add(createConnection());
            createdCount++;
        }

    }

    /**
     * 自动减少连接
     */
    private synchronized void autoReduce() {
        if (createdCount > minCount && conns.size() > 0) {
            //关闭池中空闲连接
            try {
                conns.removeFirst().close();
                createdCount--;
                System.out.print("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "已关闭多余空闲连接");
                System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "  当前已创建连接数：" + createdCount + "  当前空闲连接数：" + conns.size());
            } catch (SQLException e) {
                e.printStackTrace();
            }
        } else {
            System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "空闲连接保留在到连接池中或已被使用" + "  已创建连接数量：" + createdCount + "  空闲连接数" + conns.size());
        }
    }

    /**
     * 获取池中连接
     *
     * @return 连接
     */
    @Override
    public Connection getConnection() {
        //判断池中是否还有连接
        synchronized (MONITOR) {
            if (conns.size() > 0) {
                System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "获取到连接：" + conns.getFirst() + "  已创建连接数量：" + createdCount + "  空闲连接数" + (conns.size() - 1));
                return conns.removeFirst();
            }
            //如果没有空连接，则调用自动增长方法
            if (createdCount < maxCount) {
                autoAdd();
                return getConnection();
            }
            //如果连接池连接数量达到上限,则等待连接归还
            System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "连接池中连接已用尽，请等待连接归还");
            try {
                MONITOR.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return getConnection();
        }

    }

    /**
     * 归还连接
     *
     * @param conn 连接
     */

    @Override
    public void returnConnection(Connection conn) {

        synchronized (MONITOR) {
            System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "准备归还数据库连接" + conn);
            conns.add(conn);
            MONITOR.notify();
            autoReduce();
        }
    }

    /**
     * 获取当前空闲连接数
     *
     * @return 空闲连接数
     */
    @Override
    public int getIdleCount() {
        return conns.size();
    }

    /**
     * 返回已创建连接数量
     *
     * @return 已创建连接数量
     */
    @Override
    public int getCreatedCount() {
        return createdCount;
    }


}
```
###### 2.4 工具类
字符串工具类主要使用的是属性格式和表字段格式的驼峰与下划线之间的转换。
StringUtils.java:
```java
/**
 * 封装了字符串常用的操作
 *
 * @author xxbb
 */
public class StringUtils {
    /**
     * 正则表达式 用于匹配下划线
     */
    private static Pattern linePattern = Pattern.compile("_(\\w)");
    /**
     * 正则表达式 用于匹配大写字母
     */
    private static Pattern humpPattern = Pattern.compile("[A-Z]");


    /**
     * 将传入的表名去除t_字符，转化为类名
     *
     * @param str 传入字段
     * @return 取出t_的下划线转驼峰+首字母大写字段
     */
    public static String tableNameToClassName(String str) {
        if ("t_".equals(str.substring(0, 2))) {
            return firstCharToUpperCase(lineToHump(str.substring(2)));
        } else {
            return lineToHump(str);
        }

    }


    /**
     * 将数据库字段名转化为类的命名规则，即下划线改驼峰+首字母大写，例如要获取if_freeze字段的方法，方法为getIfFreeze(),
     *
     * @param str 传入字段
     * @return 下划线改驼峰+首字母大写
     */
    public static String columnNameToMethodName(String str) {
        return firstCharToUpperCase(lineToHump(str));
    }

    /**
     * 将传入字符串的首字母大写
     *
     * @param str 传入字符串
     * @return 首字母大写的字符串
     */
    public static String firstCharToUpperCase(String str) {
        return str.toUpperCase().substring(0, 1) + str.substring(1);
    }

    /**
     * 下划线转驼峰
     *
     * @param str 待转换字符串
     * @return 驼峰风格字符串
     */
    public static String lineToHump(String str) {
        //将小写转换
        String newStr = str.toLowerCase();
        Matcher matcher = linePattern.matcher(newStr);
        StringBuffer sb = new StringBuffer();
        while (matcher.find()) {
            matcher.appendReplacement(sb, matcher.group(1).toUpperCase());
        }
        matcher.appendTail(sb);
        return sb.toString();
    }

    /**
     * 驼峰转下划线
     *
     * @param str 待转换字符串
     * @return 下划线风格字符串
     */
    public static String humpToLine(String str) {
        //将首字母先进行小写转换
        String newStr = str.substring(0, 1).toLowerCase() + str.substring(1);
        //比对字符串中的大写字符
        Matcher matcher = humpPattern.matcher(newStr);
        StringBuffer sb = new StringBuffer();
        //匹配替换
        while (matcher.find()) {
            matcher.appendReplacement(sb, "_" + matcher.group(0).toLowerCase());
        }
        matcher.appendTail(sb);
        return sb.toString();
    }


    /**
     * string is not empty
     *
     * @param str 字符串
     * @return 结果
     */
    public static boolean isNotEmpty(String str) {
        return str != null && str.trim().length() > 0;
    }



    /**
     * 对字符串去空白符和换行符等
     *
     * @return 字符串
     */
    public static String stringTrim(String src) {
        return (null != src) ? src.trim() : null;
    }
}
```
ReflectUtils.java
```java
/**
 * 反射调用类的set，get方法
 *
 * @author xxbb
 */
public class ReflectUtils {
    /**
     * 调用Object对应属性的get方法
     *
     * @param object    类对象
     * @param fieldName 与类对象属性对应的数据库字段名
     * @return 类对象的属性
     */
    public static Object invokeGet(Object object, String fieldName) {
        Class<?> clazz = object.getClass();
        Method method;
        Object res;
        try {
            method = clazz.getDeclaredMethod("get" + StringUtils.firstCharToUpperCase(fieldName));
            res = method.invoke(object);
        } catch (Exception e) {
            throw new RuntimeException("[" + Thread.currentThread().getName() + "]" +
                    "com.xxbb.smybatis.utils.ReflectUtils" + "--->" +
                    e.getMessage());
        }
        return res;
    }

    /**
     * 调用Object对应属性的get方法
     *
     * @param obj        类对象
     * @param columnName 与类对象属性对应的数据库字段名
     * @param value      属性值
     */
    public static void invokeSet(Object obj, String columnName, Object value) {
        Class<?> clazz = obj.getClass();
        Method method;
        try {
            method = clazz.getDeclaredMethod("set" + StringUtils.columnNameToMethodName(columnName), value.getClass());

            method.invoke(obj, value);
        } catch (Exception e) {
            throw new RuntimeException("[" + Thread.currentThread().getName() + "]" +
                    "com.xxbb.smybatis.utils.ReflectUtils" + "--->" + "value=" +
                    value.getClass());
        }

    }
}
```
CommonUtils
```java
/**
 * 公共工具类
 *
 * @author xxbb
 */
public class CommonUtils {
    /**
     * list/set is not empty
     *
     * @param collection 集合
     * @return 结果
     */
    public static boolean isNotEmpty(Collection<?> collection) {
        return collection != null && !collection.isEmpty();
    }

    /**
     * map is not empty
     *
     * @param map map
     * @return 结果
     */
    public static boolean isNotEmpty(Map<?, ?> map) {
        return map != null && !map.isEmpty();
    }

    /**
     * 数组不为空
     *
     * @param objects 数组
     * @return 结果
     */
    public static boolean isNotEmpty(Object[] objects) {
        return objects != null && objects.length > 0;
    }

```
xml解析工具使用的时dom4j进行解析
XmlParseUtils.java
```java
/**
 * 解析配置了sql语句的mapper.xml文件
 *
 * @author xxbb
 */
public class XmlParseUtils {
    /**
     * 解析mapper.xml中的sql语句
     *
     * @param filename      mapper.xml的文件位置
     * @param configuration 配置文件
     */
    @SuppressWarnings("rawtypes")
    public static void mapperParser(File filename, Configuration configuration) {
        try {
            //创建读取器
            SAXReader saxReader = new SAXReader();
            saxReader.setEncoding(Constant.CHARSET_UTF8);

            //读取文件内容
            Document document = saxReader.read(filename);

            //获取xml中的根元素
            Element rootElement = document.getRootElement();

            //判断根元素是否正确
            if (!Constant.XML_ROOT_LABEL.equals(rootElement.getName())) {
                System.out.println("mapper.xml文件的元素错误");
                return;
            }
            //获取标签内容
            String namespace = rootElement.attributeValue(Constant.XML_NAMESPACE);
            //遍历根元素内的标签
            for (Iterator iterator = rootElement.elementIterator(); iterator.hasNext(); ) {
                //封装mappedStatement信息
                MappedStatement statement = new MappedStatement();
                //遍历的标签
                Element element = (Element) iterator.next();
                //标签名
                String elementName = element.getName();
                //标签id
                String elementId = element.attributeValue(Constant.XML_ELEMENT_ID);
                //标签返回值
                String returnType = element.attributeValue(Constant.XML_ELEMENT_RESULT_TYPE);
                //sql语句内容
                String sql = element.getStringValue();
                //设置sql的唯一Id
                String id = namespace + "." + elementId;
                //封装信息
                //判断标签名是否已定义，未定义则使用default
                if (!Constant.SqlType.SELECT.value().equals(elementName) &&
                        !Constant.SqlType.UPDATE.value().equals(elementName) &&
                        !Constant.SqlType.DELETE.value().equals(elementName) &&
                        !Constant.SqlType.INSERT.value().equals(elementName)) {
                    System.out.println("mapper.xml中存在未定义标签:" + elementName);
                    statement.setSqlType(Constant.SqlType.DEFAULT.value());
                } else {
                    statement.setSqlType(elementName);
                }
                //判断是否有返回值类型
                if (null != returnType && !"".equals(returnType)) {
                    statement.setReturnType(returnType);
                }

                statement.setId(id);
                statement.setNamespace(namespace);
                statement.setSql(StringUtils.stringTrim(sql));

                //封装进configuration对象
                configuration.addMappedStatement(id, statement);
                //注册一个该mapper对象接口类的代理工厂
                configuration.addMapper(Class.forName(namespace));
            }
            System.out.println();


        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```
###### 2.5 Configuration类
核心配置类
Properties对象负责封装和读取propeities配置文件信息，properties对象的定义为public static ，方便调用
```java
 /**
     * 数据库连接配置
     */
    public static Properties pros = new Properties();
    /**
     * 获取配置文件属性(属性不存再则返回空字符串"")
     *
     * @param key 键名
     * @return 键值
     */
    public static String getProperty(String key) {
        return getProperty(key, "");
    }

    /**
     * 获取配置文件属性(可指定属性不存在时的返回值)
     *
     * @param key          键名
     * @param defaultValue 属性不存在时的返回值
     * @return 键值
     */
    public static String getProperty(String key, String defaultValue) {
        return pros.containsKey(key) ? pros.getProperty(key) : defaultValue;
    }
```
 MapperRegistry 类 为代理工厂的注册器，会在DefaultSqlSessionFactory类中完成对所有代理工厂的注册。
 ```java
  /**
     * mapper代理注册器
     */
    protected final MapperRegistry mapperRegistry = new MapperRegistry();
      /**
     * 注册mapper接口类
     *
     * @param type mapper接口类
     * @param <T>  泛型
     */
    public <T> void addMapper(Class<T> type) {
        this.mapperRegistry.addMapper(type);
    }

    /**
     * 获取mapper代理对象
     *
     * @param type       mapper接口
     * @param sqlSession 当前使用的sqlSession对象
     * @param <T>        泛型
     * @return 代理对象
     */
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return this.mapperRegistry.getMapper(type, sqlSession);
    }
    
 ```
mappedStatementMap存放所有封装了sql信息的mappedStatement对象
```java
/**
     * mapper中的sql信息
     */
    protected final Map<String, MappedStatement> mappedStatementMap = new HashMap<>();
/**
     * 将sql信息对象存储进map中
     *
     * @param statement       标识一个封装sql信息对象的唯一id
     * @param mappedStatement 封装了sql信息的对象
     */
    public void addMappedStatement(String statement, MappedStatement mappedStatement) {
        this.mappedStatementMap.put(statement, mappedStatement);
    }

    /**
     * 根据sql唯一id从map中获取封装sql信息的对象
     *
     * @param statement 标识一个封装sql信息对象的唯一id
     * @return 封装了sql信息的对象
     */
    public MappedStatement getMappedStatement(String statement) {
        return this.mappedStatementMap.get(statement);
    }

```
MyDataSource连接池对象及其get方法
```java
 /**
     * 数据库连接池对象
     */
    protected MyDataSource dataSource = MyDataSourceImpl.getInstance();
    public MyDataSource getDataSource() {
        return dataSource;
    }
```
classToTableInfoMap存放所有po类与数据库表信息的对应关系，通过po类获取对应表结构信息。po类的属性和数据库表的字段一一对应，格式可以进行驼峰格式到下划线格式的对应转换。该框架可以根据传入po类自动生成sql语句，对于修改和删除的sql语句需要知道传入类中的哪个属性作为条件，这里我使用主键所对应的属性作为条件。
```java
 /**
     * 所连数据库所有表的信息与类的映射
     */
    protected Map<Class<?>, TableInfo> classToTableInfoMap = new HashMap<>();
     /**
     * 获取类表关系
     *
     * @return 所连数据库所有表的信息与类的映射
     */
    public Map<Class<?>, TableInfo> getClassToTableInfoMap() {
        return classToTableInfoMap;
    }
```
###### 2.6 数据库信息类
用于封装字段结构和表结构的类
ColumnInfo.java
```java
/**
 * 封装数据库表的一个字段的信息
 *
 * @author xxbb
 */
public class ColumnInfo {
    /**
     * 字段名
     */
    private String name;
    /**
     * 字段数据类型
     */
    private String dataType;
    /**
     * 字段的键类型(规定 0:普通键，1：主键，2：外键)
     */
    private int keyType;
}
```
TableInfo.java
```java
/**
 * 封装数据库表的结构信息
 *
 * @author xxbb
 */
public class TableInfo {
    /**
     * 表名
     */
    private String tableName;
    /**
     * 表内字段集合
     */
    private Map<String, ColumnInfo> columnInfoMap;
    /**
     * 主键
     */
    private List<ColumnInfo> primaryKeys;
    /**
     * 外键集合
     */
    private List<ColumnInfo> foreignKeys;
```
###### 2.7 代理类
MapperRegistry类，代理工厂注册器，其中声明了一个HashMap用来存储接口类和其代理工厂类之间的映射。
```java
 /**
     * 存储代理mapper工厂
     */
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

    /**
     * 注册代理工厂
     *
     * @param type 被代理的接口类
     * @param <T>  接口类泛型
     */
    public <T> void addMapper(Class<T> type) {
        this.knownMappers.put(type, new MapperProxyFactory<T>(type));
    }
```
提供一个getMapper方法来获取对应接口类的代理类。实际上是根据传入接口类对象从其代理工厂中获取代理类。逻辑代码直接和此方法进行交互，获取代理类，而不需去关注代理类如何创建的。
```java
  /**
     * 获取代理工厂实例
     *
     * @param type       被代理接口类的类型，作为key去代理工厂的map中寻找对应的工厂
     * @param sqlSession 当前的sqlSession对象
     * @param <T>        接口类型
     * @return 接口的代理
     */
    @SuppressWarnings("unchecked")
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) this.knownMappers.get(type);
        return mapperProxyFactory.newInstance(sqlSession);
    }
```
MapperProxyFactory代理工厂类,创建接口代理类需要传入<code>Class< T > mapperInterface</code>,<code>SqlSession sqlSession</code>连个参数，前者是使用jdk提供的Proxy类所必须的参数，后者是为了在接口代理类中获取Configuration类对象，得到需要的mappedStatement，在进行数据库操作使用。
```java
/**
 * mapper代理对象工厂
 *
 * @author xxbb
 */
public class MapperProxyFactory<T> {

    /**
     * 被代理的接口
     */
    private final Class<T> mapperInterface;


    /**
     * 构造方法
     *
     * @param mapperInterface 需要被代理的接口类
     */
    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    /**
     * 创建mapper代理对象
     *
     * @param sqlSession 当前的sqlSession对象
     * @return 代理实现类
     */
    public T newInstance(SqlSession sqlSession) {
        MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, this.mapperInterface);
        return newInstance(mapperProxy);
    }

    /**
     * 根据mapper代理的类型返回对应的代理实例
     *
     * @param mapperProxy 代理对象
     * @return 代理对象
     */
    @SuppressWarnings("unchecked")
    public T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(this.mapperInterface.getClassLoader(),
                new Class[]{this.mapperInterface},
                mapperProxy);
    }
```
 MapperProxy继承了InvocationHandler接口实现动态代理功能，拦截接口方法进行增强。根据接口的全限定名+方法名获取对应的mappedStatement对象，判断该对象的CRUD类型去调用SqlSession中的对应方法。

```java
  @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Exception {
        try {
            return this.execute(method, args);
        } catch (Exception e) {
            throw new RuntimeException("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + e.getMessage());
        }
    }

    private Object execute(Method method, Object[] args) {
        //sql语句的唯一id
        String statementId = this.mapperInterface.getName() + "." + method.getName();
        //获取sql信息
        MappedStatement mappedStatement = this.sqlSession.getConfiguration().getMappedStatement(statementId);
        Object result = null;
        //根据mappedStatement提供的方法选择对应的CRUD方法
        //查询
        if (Constant.SqlType.SELECT.value().equals(mappedStatement.getSqlType())) {
            Class<?> returnType = method.getReturnType();
            //返回值为集合类型
            if (Collection.class.isAssignableFrom(returnType)) {
                result = sqlSession.selectList(statementId, args);
            } else {
                result = sqlSession.selectOne(statementId, args);
            }

        }
        //更新
        if (Constant.SqlType.UPDATE.value().equals(mappedStatement.getSqlType())) {
            result = sqlSession.update(statementId, args);
        }
        //插入
        if (Constant.SqlType.INSERT.value().equals(mappedStatement.getSqlType())) {
            result = sqlSession.insert(statementId, args);
        }
        //删除
        if (Constant.SqlType.DELETE.value().equals(mappedStatement.getSqlType())) {
            result = sqlSession.delete(statementId, args);
        }


        return result;
    }
```
###### 2.8 执行器类
包含参数处理类、结果集处理类、sql语句处理类和执行器类，在执行器类中调用前三个类。

参数处理类是按传入顺序将参数设置到paramPreparedStatement对象中，这就需要传入参数顺序和sql语句所需要参数的顺序一一对应。
DefaultParameterHandler.java
```java
/**
 * @author xxbb
 */
public class DefaultParameterHandler implements ParameterHandler {
    /**
     * 传入参数
     */
    private final Object parameter;

    public DefaultParameterHandler(Object parameter) {
        this.parameter = parameter;
    }

    /**
     * 设置参数到预处理对象中
     *
     * @param paramPreparedStatement 预处理对象
     */
    @Override
    public void setParameters(PreparedStatement paramPreparedStatement) {
        try {
            if (null != parameter) {

                if (parameter.getClass().isArray()) {
                    System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + parameter.getClass());
                    Object[] params = (Object[]) parameter;
                    for (int i = 0; i < params.length; i++) {
                        paramPreparedStatement.setObject(i + 1, params[i]);
                    }
                } else {
                    paramPreparedStatement.setObject(1, parameter);
                }
            }
        } catch (SQLException throwable) {
            throwable.printStackTrace();
        }
    }
}
```
sql语句处理类包含将sql语句通过正则匹配将<code>#{}</code>替换成<code>?</code> 的方法，还有数据库查询和修改的方法
SimpleStatementHandler.java
```java
/**
 * @author xxbb
 */
public class SimpleStatementHandler implements StatementHandler {
    /**
     * 正则匹配语句中的#{}标签
     */
    private static final Pattern PARAM_PATTERN = Pattern.compile("#\\{([^{}]*)}");
    /**
     * 封装sql语句的对象
     */
    private final MappedStatement mappedStatement;


    public SimpleStatementHandler(MappedStatement mappedStatement) {
        this.mappedStatement = mappedStatement;
    }

    /**
     * 将预处理sql语句中的#{}替换成？
     *
     * @param connection 连接
     * @return 处理好的sql语句
     * @throws SQLException sql异常
     */
    @Override
    public PreparedStatement prepared(Connection connection) throws SQLException {
        String originSql = mappedStatement.getSql();
        if (StringUtils.isNotEmpty(originSql)) {
            //替换#{},预处理，防止sql注入
            return connection.prepareStatement(parseSymbol(originSql));
        } else {
            throw new RuntimeException("origin sql is null.");
        }
    }

    /**
     * 数据库的查询方法
     *
     * @param preparedStatement statement对象
     * @return 结果集
     * @throws SQLException sql异常
     */
    @Override
    public ResultSet query(PreparedStatement preparedStatement) throws SQLException {
        return preparedStatement.executeQuery();
    }

    @Override
    public int update(PreparedStatement preparedStatement) throws SQLException {
        return preparedStatement.executeUpdate();
    }

    /**
     * 将SQL语句中的#{}替换为？，源码中是在SqlSourceBuilder类中解析的
     *
     * @param originSql 原始的sql语句
     * @return 处理好的替换好？的sql语句
     */
    private static String parseSymbol(String originSql) {
        originSql = originSql.trim();
        Matcher matcher = PARAM_PATTERN.matcher(originSql);
        return matcher.replaceAll("?");
    }

```
结果集处理类获取mappedStatement对象的returnType属性，通过反射获取到该语句需要的返回值的类，从结果集中获取结果集元数据，根据元数据的列名反射获取对应类的set方法将结果集参数封装到返回值类中  再存入List集合返回
```java
public <E> List<E> handlerResultSet(ResultSet resultSet) {
        try {
            //封装结果集对象
            List<E> result = new ArrayList<>();
            if (null == resultSet) {
                return null;
            }
            //获取数据库元数据
            ResultSetMetaData resultSetMetaData = resultSet.getMetaData();
            //处理结果集
            while (resultSet.next()) {
                //实例化结果集对应的类
                E rowObject = (E) Class.forName(mappedStatement.getReturnType()).newInstance();
                //遍历每一行数据并封装进对象中
                for (int i = 0; i < resultSetMetaData.getColumnCount(); i++) {
                    //获取列名和该列的值，从1开始，这里获取的是该列的别名
                    //一般多表查询时意味着所查询的类也可能是由多个数据库对应类组合而成，那么
                    //该类的属性名也将不会和数据库列名一一对应，故需要用别名去对应属性名
                    String columnName = resultSetMetaData.getColumnLabel(i + 1);
                    Object columnValue = resultSet.getObject(i + 1);

                    //将值赋值给类对象
                    ReflectUtils.invokeSet(rowObject, columnName, columnValue);
                }
                result.add(rowObject);

            }
            System.out.println(result.get(0));
            return result;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
```
 执行器类提供查询和修改方法，由于查询和修改操作中都需要先执行获取连接，处理sql语句的操作，故将该操作封装成一个模板，使用回调接口来实现细分的查询和修改方法
SimpleExecutor.java
```java
/**
     * 抽取出执行sql操作前的获取连接，预处理语句，赋值参数的操作成一个模板
     *
     * @param mappedStatement  封装sql信息的对象
     * @param parameter        参数
     * @param executorCallback 会调接口
     * @return List结果集或受影响的行数
     */
    private Object executeTemplate(MappedStatement mappedStatement, Object parameter, MyCallback executorCallback) {
        Connection connection = null;
        try {
            //获取连接
            connection = dataSource.getConnection();
            //实例化StatementHandler对象
            StatementHandler statementHandler = new SimpleStatementHandler(mappedStatement);
            //对mapperStatement中的sql语句进行处理，去除头尾空格，将#{}替换成?,封装成preparedStatement对象
            PreparedStatement preparedStatement = statementHandler.prepared(connection);
            //给占位符?的参数赋值
            ParameterHandler parameterHandler = new DefaultParameterHandler(parameter);
            parameterHandler.setParameters(preparedStatement);
            System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + preparedStatement);
            return executorCallback.doExecutor(statementHandler, preparedStatement);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        } finally {
            dataSource.returnConnection(connection);
        }
    }
```
###### 2.8 session类
SqlSessionFactoryBuilder类构建工厂类对象
```java
/**
 * 构建者模式创建sqlSession工厂对象
 *
 * @author xxbb
 */
public class SqlSessionFactoryBuilder {
    /**
     * 读取配置文件构建SqlSessionFactory工厂
     * 将配置文件的解析成输入流的工作也放到该方法中
     *
     * @param fileName 配置文件名
     * @return 工厂对象
     */
    public SqlSessionFactory build(String fileName) {
        InputStream is = SqlSessionFactoryBuilder.class.getClassLoader().getResourceAsStream(fileName);
        return build(is);
    }

    public SqlSessionFactory build(InputStream inputStream) {
        try {
            Configuration.pros.load(inputStream);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return DefaultSqlSessionFactory.getInstance(new Configuration());
    }
}

```
工厂类接口只有一个实现类DefaultSqlSessionFactory，也是一个单例工厂，同样因为其中使用的某些类没有继承Serializable接口而无法进行序列化和反序列化操作，但反射可能会破坏其单例性。

该类初始化时会扫描指定位置的mapper.xml文件，在Configuration对象中注册mapper.xml的接口代理类工厂和存储所有的MappedStatement对象，注册和存储的具体操作由XmlParseUtils完成。
```java
/**
     * 解析mapper.xml中的信息封装进Configuration对象的mappedStatementMap中
     *
     * @param dirName mapper.xml所在的文件夹名
     */
    private void loadMappersInfo(String dirName) {
        String resource = Objects.requireNonNull
                (DefaultSqlSessionFactory.class.getClassLoader().getResource(dirName)).getPath();
        System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getName() + "--->" + "加载资源路径" + resource);
        File mapperDir = new File(resource);
        //判断该路径是否为文件夹
        if (mapperDir.isDirectory()) {
            //获取文件夹下所有文件
            File[] mappers = mapperDir.listFiles();
            //非空判断
            if (CommonUtils.isNotEmpty(mappers)) {
                for (File file : mappers) {
                    //如果还存在文件夹则继续获取
                    if (file.isDirectory()) {
                        loadMappersInfo(dirName + "/" + file.getName());
                    } else if (file.getName().endsWith(Constant.MAPPER_FILE_SUFFIX)) {
                        //获取以.xml为后缀的文件,存入Configuration对象的mappedStatementMap中
                        //并注册一个该mapper接口类的代理工厂
                        XmlParseUtils.mapperParser(file, this.configuration);
                    }
                }
            }

        }
    }
```
之后会读取所访问数据库的元数据，将所有的表结构类与po类之间的映射关系存入Configuration对象的classToTableInfoMap中，具体过程请查看源码，都有详细注释。

SqlSessionFactory接口提供的一个方法为获取SqlSession对象的openSession()方法，方法中直接创建一个SqlSession对象
```java
 @Override
    public SqlSession openSession() {
        return new DefaultSqlSession(this.configuration);
    }
```
这里我本来是想在SqlSession的实现类中使用原型模式，通过clone创建对象。但是经过效率测试后发现new新对象的效率更高
```java
public void createTest() {
        DefaultSqlSessionFactory factory = (DefaultSqlSessionFactory) new SqlSessionFactoryBuilder().build("conf.properties");
        long start1 = System.currentTimeMillis();
        for (int i = 0; i < 2100000000; i++) {
            factory.openSession();
        }
        long end1 = System.currentTimeMillis();
        System.out.println("new:" + (end1 - start1));
        long start2 = System.currentTimeMillis();
        for (int i = 0; i < 2100000000; i++) {
            //由于jvm对new方式进行了优化，且SqlSession对象的构造方法中逻辑也不复杂
            //使得clone方法效率更低，在DefaultSqlSession中已删除了Cloneable接口
            //factory.cloneSession();
        }
        long end2 = System.currentTimeMillis();
        System.out.println("clone:" + (end2 - start2));
    }
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200424110446146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDgwNDc1MA==,size_16,color_FFFFFF,t_70)
DefaultSqlSession继承于SqlSession，是使用该框架功能的入口，提供了增删改查的所有方法，其中增删改方法可以直接传入po配对象执行。每个session对象实例化时都会创建一个执行器，所以如果是使用clone方法创建session对象会导致每一个不同的session对象持有同一个执行器，故使用clone方法在这里是根本不可取的
```java
public DefaultSqlSession(Configuration configuration) {
        this.configuration = configuration;
        executor = new SimpleExecutor(configuration);
    }
```
session类中的自动生成sql语句进行数据库操作的方法也使用了模板方法回调接口方法只用于生成sql语句
```java
 private <T> Object generateSqlTemplate(T type, MyCallback callback) {
        //获取类对象
        Class<?> clazz = type.getClass();
        //获取类属性
        Field[] fields = clazz.getDeclaredFields();
        //获取表信息
        TableInfo tableInfo = this.configuration.getClassToTableInfoMap().get(clazz);
        //获取主键
        List<ColumnInfo> primaryKeys = tableInfo.getPrimaryKeys();
        //存储参数的集合
        List<Object> params = new ArrayList<>();
        //sql信息对象
        MappedStatement mappedStatement = new MappedStatement();
        //构建sql语句,利用主键作为修改的条件
        StringBuilder sql = new StringBuilder();
        //生成sql语句
        callback.generateSqlExecutor(fields, tableInfo, primaryKeys, sql, mappedStatement, params);
        //去除最后一个逗号
        sql.setCharAt(sql.length() - 1, ' ');
        //打印数据
        System.out.println("[" + Thread.currentThread().getName() + "]" +
                this.getClass().getName() + "--->" +
                sql.toString());
        System.out.println("[" + Thread.currentThread().getName() + "]" +
                this.getClass().getName() + "--->" + "params:" +
                params);
        //封装到MappedStatement对象中
        mappedStatement.setSql(sql.toString());
  		//要将List数组转化为Object[]数组，不然在ParameterHandler中会进行isArray()判断，返回false，
        //而不会去遍历参数，直接将这个list集合作为一个参数传入sql语句中，会报参数个数异常的错误
        return this.executor.doUpdate(mappedStatement, params.toArray());
    }
```
在生成sql语句时，将传入po类的非空属性都构建到sql语句中，这时候我们就需要用到TableInfo类来获取主键所对应的属性作为条件。
```java
/**
     * 传入po类对象自动生成sql语句并执行
     *
     * @param type 封装好数据的po类对象
     * @return 受影响的行数
     */
    @Override
    public <T> int update(T type) {
        return (int) generateSqlTemplate(type, new MyCallback() {
            /**
             *
             * @param fields 该类的属性
             * @param tableInfo 该类对应的表
             * @param primaryKeys 该表的主键
             * @param sql 空的sql语句
             * @param mappedStatement 封装sql信息的对象
             * @param params 该po类携带的参数
             */
            @Override
            public void generateSqlExecutor(Field[] fields, TableInfo tableInfo, List<ColumnInfo> primaryKeys, StringBuilder sql, MappedStatement mappedStatement, List<Object> params) {
                sql.append("update ").append(tableInfo.getTableName()).append(" set ");
                for (Field field : fields) {
                    String fieldName = field.getName();
                    Object fieldValue = ReflectUtils.invokeGet(type, fieldName);
                    if (null != fieldValue) {
                        //判断该属性对应的列是否为非主键
                        for (ColumnInfo columnInfo : primaryKeys) {
                            if (!fieldName.equals(columnInfo.getName())) {
                                sql.append(StringUtils.humpToLine(fieldName)).append("=?,");
                                params.add(fieldValue);
                            }
                        }
                    }
                }
                //去除当前最后一个逗号
                sql.setCharAt(sql.length() - 1, ' ');
                //添加更新条件
                sql.append("where ");
                for (ColumnInfo columnInfo : primaryKeys) {
                    sql.append(StringUtils.humpToLine(columnInfo.getName())).append("=?,");
                    params.add(ReflectUtils.invokeGet(type, columnInfo.getName()));
                }

            }
        });

    }
```
到此整个框架就差不多完成了，这里只是介绍了一下我编写这个框架的思路，具体内容请参考我GitHub上的源码，感谢阅读！

框架代码参考：[读完源码，手写一个mybatis框架](https://blog.csdn.net/kuailebuzhidao/article/details/88355236?depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-3&utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-3)
                      

