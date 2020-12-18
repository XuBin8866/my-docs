# 自实现简易SSM框架

源码已上传至GitHub和码云，如果觉得有帮助的话随手点颗Star呗，感谢各位。

GitHub：https://github.com/XuBin8866/simple-framework

码云：https://gitee.com/AlwaysXu/simple-framework

## 一、框架介绍

该框架基本实现SSM的功能，从IoC的容器和DI依赖注入到AOP，使用方式和Spring框架基本相同，IoC、DI、AOP和自实现的简易MyBatis框架通过MVC模块下的DispatcherServlet进行整合。作为一个简单框架，已经运用到我的一个JSP项目中，在业务开发的过程中对框架代码进行了多次优化，目前基本能满足开发需求。不过作为自实现框架，更多的还是为我们理解Spring提供一些帮助。

#### 1.目录结构

![simple-framework](https://photo-store8866.oss-cn-hangzhou.aliyuncs.com/img/simple-framework.png)

#### 2.框架分析

+ 2.1 aop

  使用CGLib动态代理创建代理对象，引入AsepctJ的切入点表达式和切入点解析器实现@Aspect注解的横切逻辑和切面定位，通过@Order注解规定代理方法的执行顺序

+ 2.2 core

  扫描指定包下的所有java类，通过反射创建Class对象，扫描对象上的注解来判断是否存入IoC容器中

+ 2.3 inject

  获取IoC容器中的Class对象，扫描对象属性上的@AutoWired注解，从IoC容器中获取属性的实例化对象，通过反射方式进行赋值，实现依赖注入

+ 2.4 mvc

  使用责任链模式对请求进行处理，再对请求结果进行渲染，在DispatcherServlet中对IoC、DI、AOP和自实现的MyBatis框架进行整合，使整个框架能够满足Web开发需求

+ 2.5 util

  提供反射创建Class对象、参数类型转换、日志、字符串处理和空值判断的工具类。

+ 2.6 simple-mybatis

  之前实现的简易MyBatis框架，地址：[自实现简易MyBatis框架](https://blog.csdn.net/weixin_44804750/article/details/105713236)

## 二、详细分析

分析代码不一定完整，建议结合源码进行阅读

#### 1.框架使用方式

和Spring框架的使用方式类似，如果直接将该框架作为Module引入则无需做任何修改，框架的配置文件名默认为application.properties,可在DispatcherServlet中进行对参数命名进行修改

```java
@WebServlet(name="DispatcherServlet" ,urlPatterns="/*",
initParams ={@WebInitParam(name="contextConfigLocation",value = "application.properties")},
 loadOnStartup = 1)
public class DispatcherServlet extends HttpServlet {}
```

如果想把该框架打成jar包引入，则需要在打包前先把上述的注解删除再进行打包，在引入该jar包的项目的web.xml中将上述注解的内容进行配置。

application.properties配置参考

```properties
##sspring配置

#spring扫描的包
scanPackage=com.xxbb.demo

##smybatis配置

####mapper接口所在的包
mapper.location=com.xxbb.demo.mapper
####与数据库表对应的po类所在的包
po.location=com.xxbb.demo.domain
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

log4j.properties日志配置参考：

```properties
log4j.rootLogger =DEBUG,CONSOLE,D,E


log4j.appender.CONSOLE = org.apache.log4j.ConsoleAppender
log4j.appender.CONSOLE.Target = System.out
log4j.appender.CONSOLE.layout = org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern = [%-5p] %d{yyyy-MM-dd HH:mm:ss,SSS} %l%m%n


log4j.appender.D = org.apache.log4j.DailyRollingFileAppender
log4j.appender.D.File = D://logs/log.log
log4j.appender.D.Append = true
log4j.appender.D.Threshold = DEBUG
log4j.appender.D.layout = org.apache.log4j.PatternLayout
log4j.appender.D.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n


log4j.appender.E = org.apache.log4j.DailyRollingFileAppender
log4j.appender.E.File=D://logs/error.log
log4j.appender.E.Append = true
log4j.appender.E.Threshold = ERROR
log4j.appender.E.layout = org.apache.log4j.PatternLayout
log4j.appender.E.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n
```

#### 2.代码分析

###### 2.1 core

core包下的BeanContainer即为IoC的容器，使用@Component、@Controller、@Service、@Respository和aspect中的@Aspect注解对需要存入IoC容器中的类进行标记

存储bean对象的容器如下

```java
/**
 * bean容器
 */
private final Map<Class<?>, Object> beanMap = new ConcurrentHashMap<>();
```

BeanContainer类的设计运用了单例设计模式，使用的是枚举类单例，想了解单例设计模式和枚举类单例的优势可以看看如下文章：

[gof23——单例设计模式详解](https://blog.csdn.net/weixin_44804750/article/details/105883244)

[单例设计模式——枚举类单例源码分析](https://blog.csdn.net/weixin_44804750/article/details/106124842)

BeanContainer的单例实现：

```java
	/**
 	* 通过内部枚举类来实现bean容器的单例，线程安全，不会被反射或者序列化破坏
 	*/
	private enum ContainerHolder {
    	/**
     	* 存储bean容器的枚举对象
    	*/
    	HOLDER;
    	private BeanContainer instance;

    	ContainerHolder() {
        	instance = new BeanContainer();
    	}
	}
 /**
     * 获取单例容器对象
     *
     * @return 容器对象
     */
    public static BeanContainer getInstance() {

        return ContainerHolder.HOLDER.instance;
    }
```

加载bean对象：

通过ClassUtil对指定包名进行扫描，将所有的类存储至Set集合中，再遍历Set集合找到将被注解标记的类和其实例化对象存入IoC容器中

```java
/**
 * 将bean对象加载进容器
 *
 * @param packageName 扫描包名
 */
public void loadBeans(String packageName) {
    if (isLoaded()) {
        LOGGER.warn("BeanContainer has been loaded");
        return;
    }
    Set<Class<?>> classSet = ClassUtil.extractPackageClass(packageName);
    if (ValidationUtil.isEmpty(classSet)) {
        LOGGER.warn("Extract nothing from packageName:" + packageName);
        return;
    }
    for (Class<?> clazz : classSet) {
        for (Class<? extends Annotation> annotation : BEAN_ANNOTATION) {
            //如果类对象中存在注解则加载进bean容器中
            if (clazz.isAnnotationPresent(annotation)) {
                LOGGER.debug("load bean: "+clazz.getName());
                beanMap.put(clazz, ClassUtil.newInstance(clazz, true));
            }
        }
    }
    loaded = true;
}
```

ClassUtil：

```java
public class ClassUtil {
    private final static String FILE_PROTOCOL="file";
    private final static String SUFFIX =".class";
    private final static Logger LOGGER=LogUtil.getLogger();

    /**
     * 反射设置bean对象的值
     * @param targetBean bean对象
     * @param field 属性
     * @param fieldValue 值
     * @param accessible 是否允许设置私有属性的值
     */
    public static void setField(Object targetBean , Field field, Object fieldValue,boolean accessible){
        field.setAccessible(accessible);
        try {
            field.set(targetBean,fieldValue);
        } catch (IllegalAccessException e) {
            LOGGER.error("setField error:"+e);
            throw new RuntimeException(e);
        }
    }
    /**
     * 实例化class对象
     * @param clazz bean对象的class
     * @param accessible 是否实例化私有构造方法的对象
     * @param <T> 泛型
     * @return bean
     */
    public static <T>T newInstance(Class<T> clazz,Boolean accessible){
        try {
            Constructor<T> declaredConstructor = clazz.getDeclaredConstructor();
            declaredConstructor.setAccessible(accessible);
            return declaredConstructor.newInstance();
        } catch (Exception e) {
            LOGGER.error("bean newInstance error:"+e);
            throw new RuntimeException(e);
        }
    }
    /**
     * 加载类资源
     * @param packageName 需加载类所在的包
     * @return 类资源的set
     */
    public static Set<Class<?>> extractPackageClass(String packageName){

        //获取类加载器
        ClassLoader classLoader=getClassLoader();
        //通过类加载器获取到加载的资源,replace是字符或字符串匹配替换，replaceAll是字符或正则替换
        URL url = classLoader.getResource(packageName.replaceAll("\\.", "/"));
        if(null==url){
            LOGGER.warn("unable to retrieve anything from package: "+packageName);
            return null;
        }
        //依据不同的资源类型，采用不同的方式获取资源的集合
        Set<Class<?>> classSet=null;
            //判断文件协议
        if(FILE_PROTOCOL.equalsIgnoreCase(url.getProtocol())){
            classSet=new HashSet<>();
            File packageDirectory=new File(url.getFile());
            extractClassFile(classSet,packageDirectory,packageName);
        }
        return classSet;
    }

    /**
     * 获取目标package中的所有class文件，包括子package中的文件
     * @param classSet 装载目标类的集合
     * @param fileSource 文件或目录
     * @param packageName 扫描包名
     */
    private static void extractClassFile(Set<Class<?>> classSet, File fileSource, String packageName) {
        //如果传入文件非文件夹
        if(!fileSource.isDirectory()){
            return;
        }
        //如果是文件夹，调用listFile方法获取文件夹下内容
        File[] files=fileSource.listFiles(new FileFilter() {
            @Override
            public boolean accept(File file) {
                //提取出文件夹
                if(file.isDirectory()){
                    return true;
                }else{
                    //非文件夹如果以.class结尾就直接载入set中
                    String absoluteFilePath=file.getAbsolutePath();
                    if(absoluteFilePath.endsWith(SUFFIX)){
                        addToClassSet(absoluteFilePath);
                    }
                }
                return false;
            }
            //获取类class对象载入set中
            private void addToClassSet(String absoluteFilePath) {
                absoluteFilePath=absoluteFilePath.replace(File.separator,".");
                String className=absoluteFilePath.substring(absoluteFilePath.indexOf(packageName))
                        .replace(SUFFIX,"");
                try {
                    Class<?> clazz=Class.forName(className);
                    classSet.add(clazz);
                } catch (ClassNotFoundException exception) {
                    LOGGER.error("Load class error:"+exception);
                    throw new RuntimeException(exception);
                }

            }
        });
        //递归文件夹
        //foreach遍历即使files为空也会进行，然后报空指针错误，所以这里提前判断
        if(files!=null){
            for(File file:files){
                extractClassFile(classSet,file,packageName);
            }
        }
    }

    /**
     * 获取当前线程的类加载器
     * @return 类加载器
     */
    public static ClassLoader getClassLoader(){
        return Thread.currentThread().getContextClassLoader();
    }


}
```

获取bean对象的方式有三种，通过class对象获取、通过注解获取、通过对象的接口或父类获取

```java
public Object getBean(Class<?> clazz) {
    return beanMap.get(clazz);
}
/**
 * 根据注解获取class对象的的集合
 * @param annotation 注解
 * @return class对象集合
 */
public Set<Class<?>> getClassesByAnnotation(Class<? extends Annotation> annotation) {
    //获取beanMap的所有class对象
    Set<Class<?>> keySet = getClasses();
    if (ValidationUtil.isEmpty(keySet)) {
        LOGGER.warn("nothing in beanMap");
        return null;
    }
    //通过注解筛选需要的class对象，并添加到classSet里
    Set<Class<?>> classSet = new HashSet<>();
    for (Class<?> clazz : keySet) {
        if (clazz.isAnnotationPresent(annotation)) {
            classSet.add(clazz);
        }
    }
    return classSet.size() > 0 ? classSet : null;
}

/**
 * 通过接口或者父类获取到对应的class
 * @param classOrInterface 接口或父类
 * @return 集合
 */
public Set<Class<?>> getClassesBySuper(Class<?> classOrInterface) {
    //获取beanMap的所有class对象
    Set<Class<?>> keySet = getClasses();
    if (ValidationUtil.isEmpty(keySet)) {
        LOGGER.warn("nothing in beanMap");
        return null;
    }
    //通过注解筛选需要的class对象，并添加到classSet里
    Set<Class<?>> classSet = new HashSet<>();
    for (Class<?> clazz : keySet) {
        //这里只想判断该类是否是作为classOrInterface的子类，而不想获取到自己本身
        if (classOrInterface.isAssignableFrom(clazz)&&!clazz.equals(classOrInterface)) {
            classSet.add(clazz);
        }
    }
    return classSet.size() > 0 ? classSet : null;
}
```

###### 2.2 inject

inject包下的DependencyInject实现依赖注入的功能，扫描Class对象中属性的注解@AutoWired进行依赖注入，通过反射的方式个成员变量赋值

```java
public class DependencyInject {
    private BeanContainer beanContainer;
    private static final Logger LOGGER= LogUtil.getLogger();
    public DependencyInject() {
        beanContainer = BeanContainer.getInstance();
    }

    /**
     * 依赖注入
     */
    public void doDependencyInject() {

        //遍历Bean容器中的所有class对象
        Set<Class<?>> classSet = beanContainer.getClasses();
        if (ValidationUtil.isEmpty(classSet)) {
            LOGGER.warn("There is an empty classSet");
        }
        for (Class<?> clazz : classSet) {
            //遍历class对象的所有成员变量
            Field[] fields = clazz.getDeclaredFields();
            if (ValidationUtil.isEmpty(fields)) {
                continue;
            }
            for (Field field : fields) {
                //找出被Autowired标记的成员变量
                if (field.isAnnotationPresent(Autowired.class)) {
                    String autowiredValue = field.getAnnotation(Autowired.class).value();
                    //获取成员变量类型
                    Class<?> fieldClass = field.getType();
                    //获取成员变量类型在容器中对应的实例
                    Object fieldValue = getFieldInstance(fieldClass, autowiredValue);
                    if (fieldValue == null) {
                        throw new RuntimeException("unable to inject relevant type, target fieldClass is " + fieldClass.getSimpleName());
                    }
                    //通过反射将对应的成员变量实例注入到成员变量中
                    Object targetBean = beanContainer.getBean(clazz);
                    LogUtil.getLogger().debug("Dependency inject for class: "+clazz.getName()+";injected value："+fieldClass.getName());
                    ClassUtil.setField(targetBean, field, fieldValue, true);

                }
            }
        }

    }

    /**
     * 获取属性的类型对应的实现类（如果一个接口有多个实现类如何处理？先找到实现类，在通过定义的名称找到具体类）
     *
     * @param fieldClass     属性的class类
     * @param autowiredValue 需要注入实例的类名称，约定首字母小写
     * @return 实例对象
     */
    private Object getFieldInstance(Class<?> fieldClass, String autowiredValue) {
        //传入类非接口或者父类，在容器中找得到实例
        Object fieldValue = beanContainer.getBean(fieldClass);
        if (null != fieldValue) {
            return fieldValue;
        } else {
            //获取接口的实现类
            Class<?> implementedClass = getImplementClass(fieldClass, autowiredValue);
            if (null != implementedClass) {
                return beanContainer.getBean(implementedClass);
            } else {
                return null;
            }
        }
    }

    /**
     * 获取接口的实现类class对象
     *
     * @param fieldClass     属性的class类
     * @param autowiredValue 需要注入实例的类名称，约定首字母小写
     * @return 实例对象
     */
    private Class<?> getImplementClass(Class<?> fieldClass, String autowiredValue) {
        Set<Class<?>> classSet = beanContainer.getClassesBySuper(fieldClass);
        if (!ValidationUtil.isEmpty(classSet)) {
            if (ValidationUtil.isEmpty(autowiredValue)) {
                if (classSet.size() == 1) {
                    return classSet.iterator().next();
                } else {
                    //如果该接口或父类多于一个实现类但用户未指定则抛出异常
                    throw new RuntimeException("multiple implemented classes for " + fieldClass.getName() + " please set @Autowired's value to pick one");
                }

            } else {
                //这里获取的是该接口的实现类，我们给autowired命名时需要指定实现类的类名（首字母小写）
                for (Class<?> clazz : classSet) {
                    String classSimpleName= StringUtil.firstCharToLowerCase(clazz.getSimpleName());
                    if(classSimpleName.equals(autowiredValue)){
                        return clazz;
                    }
                }
            }
        }
        return null;
    }
}
```

###### 2.3 aop

aop包下即是为被代理类提供切面方法，并将切面方法织入被代理类中，织入的具体实现方式是使用CGLib动态代理创建代理对象，取代原有的类存入IoC容器中

aop涉及两个注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Aspect {
    /**
     * 切入点表达式
     * @return 切入点表达式
     */
    String pointcut();
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Order {
    int value() default 1;
}
```

@Aspect用来标记切面类，IoC容器初始化时会将切面类也存入IoC容器中，@Order类用于当一个被代理类有多个切面类时，@Order的值决定不同切面类方法的执行顺序，值越小的切面类中的前置通知优先执行，也就是前置通知按@Order的值从小到大排序顺序执行，而后置、异常、最终通知倒序执行。



DefaultAspect作为所有切面类的抽象类使用，提供了前置、后置、异常、最终通知的基本方法。

AspectInfo封装了一个切面了的Order值，切面类本身、该切面的切入点定位器。



根据CGLib创建动态代理的机制，封装了一个创建动态代理的类和方法：

```java
public class ProxyCreator {
    /**
     * 创建动态代理对象并返回
     * @param targetClass 被代理的目标对象
     * @param methodInterceptor 代理的回调接口
     * @return 代理对象
     */
    public static Object createProxy(Class<?> targetClass, MethodInterceptor methodInterceptor){
      return Enhancer.create(targetClass,methodInterceptor);
    }
}
```

这里使用了aspectj的切入点表达式和表达式解析器：

```java
/**
 * 解析Aspect表达式并且定位被织入的目标
 * @author xxbb
 */
public class PointcutLocator {
    /**
     * Pointcut解析器，直接给它赋值上AspectJ的所有表达式，以便支持对众多表达式的解析
     * 目前只使用到EXECUTION和WITHIN
     */
    private PointcutParser pointcutParser=PointcutParser.getPointcutParserSupportingSpecifiedPrimitivesAndUsingContextClassloaderForResolution(
            PointcutParser.getAllSupportedPointcutPrimitives()
    );

    /**
     * 表达式解析器
     */
    private PointcutExpression pointcutExpression;

    public PointcutLocator(String expression){
        this.pointcutExpression=pointcutParser.parsePointcutExpression(expression);
    }

    /**
     * 判断传入的Class对象是否是Aspect类的目标代理类，即匹配Pointcut表达式（初筛）
     * @param targetClass 目标类
     * @return 是否匹配
     */
    public boolean roughMatches(Class<?> targetClass){
        //只能校验within语法，其他语法的表达式直接返回true
        //这意味着如果一个类有多个切面表达式准备织入，如果within表达式，则可以判断是否符合该类
        //如果是execution表达式，粗筛无法判断，即使表达式对应的不是该目标类，也会将该切面存入该类的切面数组中
        //需要依赖下一步精筛来排除那些不属于该类的切面方
        return pointcutExpression.couldMatchJoinPointsInType(targetClass);
    }

    /**
     * 判断传入的Method对象是否是Aspect的目标代理方法，即匹配Pointcut表达式（精确筛选）
     * @param method 方法
     * @return 是否匹配具体方法
     */
    public boolean accurateMatches(Method method){
        ShadowMatch shadowMatch=pointcutExpression.matchesMethodExecution(method);
        return shadowMatch.alwaysMatches();
    }
}
```

AOP织入功能的核心类AspectWeaver和AspectListExecutor

AspectWeaver实现切面的织入，AspectListExecutor为实现了MethodInterceptor的代理对象

aop核心代码：

织入流程：

+ 1.获取容器中所有的被@Aspect标记的类的class对象，存入set集合
+ 2.遍历set集合，读取每一个class对象的@Order值和@Aspect值，构建AspectInfo对象，存入一个List数组中
+ 3.再次遍历容器，获取非@Aspcet标记的类。
  + 3.1将该类的class对象和2中的List< AspectInfo >的每一个AspectInfo对象进行匹配，匹配上的AspectInfo对象存入新的List数组，表示通过该类粗筛后的AspectInfo对象集合（粗筛的原因是aspectj提供的粗筛方法只能识别within表达式，对于其他表示式如execution无法识别，会直接通过粗筛）
  + 3.2通过粗筛的List和目标对象的class对象准备进行织入
    + 3.2.1 将粗筛的List和目标对象的class对象传入AspectListExecutor对象中，进行实例化，在该对象的实例化过程中会完成对AspectInfo对象的排序
    + 3.2.2 AspectListExecutor在intercept方法中对粗筛的List进行精确筛选，将方法与切入点表达式进行匹配，删除List中不匹配的AspectInfo对象，剩余的对象按约定顺序实现切面方法。
  + 此时有了目标对象类，也有了MethodInterceptor的实现类，通过之前编写的创建代理对象的方法创建代理对象，将他存入IoC容器中，取代之前目标对象类的实现类。

```java
public void doAspectOrientedProgramming(){
    //1.获取所有的切面类
    Set<Class<?>> aspectSet = beanContainer.getClassesByAnnotation(Aspect.class);
    //没有切面类的情况
    if(ValidationUtil.isEmpty(aspectSet)){
        LogUtil.getLogger().warn("There is no aspect in  bean container");
        return;
    }
    //2.拼装AspectInfoList
    List<AspectInfo> aspectInfoList=packAspectInfoList(aspectSet);

    //3.遍历容器里的类
    Set<Class<?>> classSet = beanContainer.getClasses();
    for(Class<?> targetClass:classSet){
        //排除被Aspect注解的类本身
        if(targetClass.isAnnotationPresent(Aspect.class)){
            continue;
        }
        //4.粗筛符合条件的Aspect
        List<AspectInfo> roughMatchedAspectList=collectRoughMatchedAspectListForSpecificClass(aspectInfoList,targetClass);
        //5.尝试进行Aspect织入
        wrapIfNecessary(roughMatchedAspectList,targetClass);
    }

}
```

织入流程：

+ 1.获取容器中所有的被@Aspect标记的类的class对象，存入set集合
+ 2.遍历set集合，读取每一个class对象的@Order值和@Aspect值，构建AspectInfo对象，存入一个List数组中

```java
/**
 * 将所有的切面类的信息（Order、DefaultAspect、pointcut）封装成封装成一个Aspect数组
 * @param aspectSet 切面类set集合
 * @return 数组
 */
private List<AspectInfo> packAspectInfoList(Set<Class<?>> aspectSet) {
    List<AspectInfo> aspectInfoList=new ArrayList<>();
    for(Class<?> aspectClass: aspectSet){
        if(verifyAspect(aspectClass)){
            //获取该切面类的数据
            Order orderTag=aspectClass.getAnnotation(Order.class);
            Aspect aspectTag=aspectClass.getAnnotation(Aspect.class);
            DefaultAspect defaultAspect= (DefaultAspect) beanContainer.getBean(aspectClass);
            //初始化表达式定位器
            PointcutLocator pointcutLocator=new PointcutLocator(aspectTag.pointcut());
            AspectInfo aspectInfo=new AspectInfo(orderTag.value(),defaultAspect,pointcutLocator);
            aspectInfoList.add(aspectInfo);
        }else{
            LogUtil.getLogger().error("packAspectInfoList error!");
            throw new RuntimeException("@Aspect and @Order must be added to the Aspect class, and Aspect class must extend from DefaultAspect");
        }

    }
    return aspectInfoList;
}
```

+ 3.再次遍历容器，获取非@Aspcet标记的类。

  + 3.1 将该类的class对象和2中的List< AspectInfo >的每一个AspectInfo对象进行匹配，匹配上的AspectInfo对象存入新的List数组，表示通过该类粗筛后的AspectInfo对象集合（粗筛的原因是aspectj提供的粗筛方法只能识别within表达式，对于其他表示式如execution无法识别，会直接通过粗筛）

  ```java
  /**
   * 粗筛切面类
   * @param aspectInfoList 所有的切面类组成的数组
   * @param targetClass 需要进行织入操作的目标类
   * @return 目标类需要的切面类的数组
   */
  private List<AspectInfo> collectRoughMatchedAspectListForSpecificClass(List<AspectInfo> aspectInfoList, Class<?> targetClass) {
      List<AspectInfo> roughMatchedAspectList=new ArrayList<>();
      for(AspectInfo aspectInfo:aspectInfoList){
          if(aspectInfo.getPointcutLocator().roughMatches(targetClass)){
              roughMatchedAspectList.add(aspectInfo);
          }
      }
      return roughMatchedAspectList;
  }
  ```

  + 3.2 通过粗筛的List和目标对象的class对象准备进行织入

    + 3.2.1 将粗筛的List和目标对象的class对象传入AspectListExecutor对象中，进行实例化，在该对象的实例化过程中会完成对AspectInfo对象的排序

    ```java
    AspectListExecutor aspectListExecutor=new AspectListExecutor(targetClass,roughMatchedAspectList);
    ```

    AspectListExecutor:

    ```java
    /**
     * 被代理类
     */
    private Class<?> targetClass;
    /**
     * 排好序的切面列表
     */
    private List<AspectInfo> sortedAspectInfoList;
    
    public AspectListExecutor(Class<?> targetClass, List<AspectInfo> aspectInfoList) {
        this.targetClass = targetClass;
        this.sortedAspectInfoList = sortAspectInfoList(aspectInfoList);
    }
    ```

    + 3.2.2 AspectListExecutor在intercept方法中对粗筛的List进行精确筛选，将方法与切入点表达式进行匹配，删除List中不匹配的AspectInfo对象，剩余的对象按约定顺序实现切面方法。

    ```java
    /**
     * 根据当前被代理的目标方法精确筛选切面方法
     * @param method 被代理的目标方法
     */
    private void collectAccurateMatchedAspectList(Method method) {
        if(ValidationUtil.isEmpty(sortedAspectInfoList)){
            return;
        }
        //将不需要的切面方法去除
        sortedAspectInfoList.removeIf(aspectInfo -> !aspectInfo.getPointcutLocator().accurateMatches(method));
    
    }
    ```

    ```java
    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        Object returnValue=null;
        //对切面方法进行精确筛选
        collectAccurateMatchedAspectList(method);
        //没有切面对方法进行增强的情况
        if(ValidationUtil.isEmpty(sortedAspectInfoList)){
            returnValue=methodProxy.invokeSuper(o,args);
            return returnValue;
        }
        //1.按照order的顺序升序执行完所有的Aspect的before方法
        invokeBeforeAdvices(method,args);
    
        try {
            //2.执行被代理类的方法
            returnValue=methodProxy.invokeSuper(o, args);
            //3.如果代理方法返回正常，则安装order的顺序降序执行完所有的Aspect的afterRunning方法
            returnValue=invokeAfterReturningAdvices(method,args,returnValue);
        }catch (Exception e){
            //4.如果代理方法返回异常，则安装order的顺序降序执行完所有的Aspect的afterThrowing方法
            invokeAfterThrowingAdvices(method,args,e);
        }finally{
            //最终通知
            invokeAfterAdvices(method,args);
        }
    
        return returnValue;
    }
    ```

  + 3.3 此时有了目标对象类，也有了MethodInterceptor的实现类，通过之前编写的创建代理对象的方法创建代理对象，将他存入IoC容器中，取代之前目标对象类的实现类。

  ```java
  //创建目标类的动态代理对象，将切面方法进行织入
  AspectListExecutor aspectListExecutor=new AspectListExecutor(targetClass,roughMatchedAspectList);
  Object proxyBean=ProxyCreator.createProxy(targetClass,aspectListExecutor);
  LogUtil.getLogger().debug("wrapping for class: {}, proxyBean: {}",targetClass.getSimpleName(),proxyBean.getClass().getSimpleName());
  beanContainer.addBean(targetClass,proxyBean);
  ```

###### 2.4 mvc

mvc包下的类提供了web方面的支持，对前三项功能以及自实现MyBatis框架进行整合。

该模块的实现逻辑：

配置一个DispatcherServlet拦截所有请求，通过责任链模式对不同类型的请求进行处理，对于不同的请求结果会创建对应的Render进行渲染。

在DispatcherServlet的初始化方法中：

+ 1.获取自身的初始化参数作为需要读取的配置文件名
+ 2.创建BeanContainer容器并加载bean对象
+ 3.进行AOP织入
+ 4.实例化SqlSessionFactory对象并存入容器中
+ 5.依赖注入
+ 6.初始化请求处理责任链

```java
@Override
public void init(ServletConfig servletConfig) {
    //读取配置文件
    doLoadConfig(servletConfig.getInitParameter("contextConfigLocation"));
    //初始化容器
    BeanContainer beanContainer = BeanContainer.getInstance();
    beanContainer.loadBeans(contextCofig.getProperty("scanPackage"));
    System.out.println();
    //AOP织入
    new AspectWeaver().doAspectOrientedProgramming();
    //初始化简易mybatis框架，往IoC容器中注入SqlSessionFactory对象
    new SqlSessionFactoryBuilder().build(servletConfig.getInitParameter("contextConfigLocation"));
    //依赖注入
    new DependencyInject().doDependencyInject();


    //初始化请求处理器责任链
    PROCESSORS.add(new PreRequestProcessor());
    PROCESSORS.add(new StaticResourceRequestProcessor(servletConfig.getServletContext()));
    PROCESSORS.add(new JspRequestProcessor(servletConfig.getServletContext()));
    PROCESSORS.add(new ControllerRequestProcessor());
}
```

在DispatcherServlet的service方法中

+ 1.创建责任链对象实例
+ 2.通过责任链模式来调用请求处理器对请求进行处理
+ 3.对处理结果进行渲染

```java
@Override
protected void service(HttpServletRequest req, HttpServletResponse resp) {
    //1.创建责任链对象实例
    RequestProcessorChain requestProcessorChain = new RequestProcessorChain(PROCESSORS.iterator(), req, resp);
    //2.通过责任链模式来一次调用请求处理器对请求进行处理
    requestProcessorChain.doRequestProcessorChain();
    //3.对处理结果进行渲染
    requestProcessorChain.doRender();

}
```

以责任链模式按编码预处理（PreRequestProcessor）、静态资源处理（PreRequestProcessor）、jsp页面请求（JspRequestProcessor）、控制层请求处理（ControllerRequestProcessor）这种顺序依次处理请求。其中最重要的就是控制层请求处理（ControllerRequestProcessor），业务基本经过该请求实现。

该处理器中有两个重要容器

```java
/**
 * 请求和对应Controller方法的映射集合
 */
private Map<RequestPathInfo, ControllerMethod> requestPathInfoControllerMethodMap=new ConcurrentHashMap<>();
/**
 * 存储查询请求结果的缓存
 */
private final ResultCache<String, Object> resultCaches;
```

在DispatcherServlet初始化的过程中实例化了ControllerRequestProcessor对象，实例化的过程中会去IoC容器中获取所有被Controller标记的类，匹配类和方法上的RequestMapping注解，将注解的值封装成RequestPathInfo对象，将该类、该方法、该方法的参数封装成ControllerMethod，存入requestPathInfoControllerMethodMap中。当前端发送请求到Servlet时，到达ControllerRequestProcessor后，会将请求路径和请求方法封装成RequestPathInfo对象传入该map中获取对应的方法对象，通过反射调用方法。之后根据方法的返回值来创建对应的Render对象对结果进行渲染。



如果该方法对应的请求时查询请求时，会使用使用缓存接收请求结果，当下次有相同的请求时直接返回请求结果，如果请求时增删改等可能会影响查询结果的请求时，缓存刷新。

```java
 //2.解析请求参数，并传递给获取到的controllerMethod实例去执行
    Object result;
    //4种需要对缓存进行处理的请求类型
    String[] requestPrefix = new String[]{"query", "add", "modify", "remove"};
    if (path.contains(requestPrefix[0])) {
        //如果时查询请求则将结果存入缓存
        String cacheKey=path + "," + method+":"+requestParametersMapToString;
        log.debug("匹配到查询请求:{},使用缓存",cacheKey);
        Callable<Object> task = () -> invokeControllerMethod(controllerMethod, requestProcessorChain.getReq());
        resultCaches.setTask(task);
        result = resultCaches.get(cacheKey);
        log.debug("当前缓存数：{}",resultCaches.size());
    } else if (path.contains(requestPrefix[1]) || path.contains(requestPrefix[2]) || path.contains(requestPrefix[3])) {
        //如果有增删改请求则刷新缓存
        log.debug("匹配到修改请求：{}，刷新缓存",path);
        resultCaches.clear();
        result = invokeControllerMethod(controllerMethod, requestProcessorChain.getReq());
    } else {
        //其他情况则直接实现方法
        result = invokeControllerMethod(controllerMethod, requestProcessorChain.getReq());
    }
    //3.根据解析结果，选择对应的render进行渲染
    setResultRender(result, controllerMethod, requestProcessorChain);
    return true;
}
```

该缓存由双向Node节点+HashMap构成，采用LRU算法。HashMap用来存储Node节点，Node节点存储方法的请求结果，Node节点的的顺序来表示它使用的频繁程度。为了避免HashMap的扩容，HashMap的实际大小会比缓存的可用大小更大。

```java
/**
 * 双向节点链表
 */
static class Node<K, V> {
    K key;
    V value;
    Node<K, V> pre;
    Node<K, V> next;

    public Node(K key, V value) {
        this.key = key;
        this.value = value;
    }
}
 public ResultCache(int initCapacity) {
        if (initCapacity > 0) {
            this.currentSize = 0;
            this.capacity = initCapacity;
            this.realCapacity = (int) Math.ceil((capacity + 1) / DEFAULT_LOAD_FACTORY);
            this.caches = new HashMap<>(realCapacity);
        } else {
            throw new RuntimeException("init ResponseCaches failed: initCapacity<=0" + initCapacity);
        }

    }
```

为了实现有请求结果时直接返回请求结果，没请求结果时执行请求方法，这里使用了Callable接口的匿名内部类，重写Call方法，由于Call方法是带返回值的，符合我们的需求。每次处理查询请求时，先将请求方法封装到Callable内，再将他传入缓存中。如果缓存中没有该请求的结果，就会使用线程池执行该Callable线程，存储该Callable线程的结果并返回。

```java
/**
 * 获取缓存内容
 *
 * @param key key
 * @return value
 */
public V get(K key) {
    lock.lock();
    try {
        while (true) {
            Node<K, V> node = caches.get(key);

            if (node == null) {
                log.debug("该请求的响应缓存不存在，调用线程执行任务");
                try {
                    Future<V> future = pool.submit(task);
                    node = new Node<>(key, future.get());
                    put(key, node);
                } catch (ExecutionException | InterruptedException e) {
                    log.error(e.getMessage());
                    remove(key);
                    //出错一定要抛出异常，不然这里会陷入死循环
                    throw new RuntimeException(e);
                }
            } else {
                log.debug("该请求的响应缓存存在：{}", node.value);
            }

            moveToHead(node);
            //获取出错则删除缓存，避免污染
            try {
                return node.value;
            } catch (Exception e) {
                remove(key);
                log.error(e.getMessage());
            }
        }
    } finally {
        lock.unlock();
    }
}
```

可以看到使用缓存的get方法会上一把锁（ ReentrantLock），可重入锁，这样保证了获取请求结果时的线程安全，这也是为什么Map的容器使用的是HashMap而不是ConcurrentHahsMap。



获取到请求结果后，根据请求结果的类型创建对应的渲染器

```java
/**
 * 根据不同情况设置不同的渲染器
 * @param result 结果
 * @param controllerMethod controllerMethod
 * @param requestProcessorChain requestProcessorChain
 */
private void setResultRender(Object result, ControllerMethod controllerMethod, RequestProcessorChain requestProcessorChain) {
    if(null==result){
        log.warn("controller method's return result is null");
        return;
    }
    ResultRender resultRender;
    boolean isJson=controllerMethod.getInvokeMethod().isAnnotationPresent(ResponseBody.class);
    if(isJson){
        resultRender=new JsonResultRender(result);
    }else{
        resultRender=new ViewResultRender(result);
    }
    requestProcessorChain.setResultRender(resultRender);
}
```

一般请求的结果都是ModelAndView对象，封装了转发路径和转发携带的值，在ViewResultRender中解析ModelAndView对象进行转发

```java
/**
 * 将请求处理结果按照视图路径转发至对应视图进行展示
 * @param requestProcessorChain 请求处理器
 * @throws Exception 异常
 */
@Override
public void render(RequestProcessorChain requestProcessorChain) throws Exception {
    HttpServletRequest request=requestProcessorChain.getReq();
    HttpServletResponse response=requestProcessorChain.getResp();
    String path=modeAndView.getView();
    Map<String,Object> model=modeAndView.getModel();
    HttpSession session=request.getSession();
    if(!model.isEmpty()){
        for(Map.Entry<String,Object> entry:model.entrySet()){
            String key=entry.getKey();
            //如果参数期望是session参数时
            if(key.startsWith("session")){
                key=key.substring(8);
                session.setAttribute(key,entry.getValue());
            }

            request.setAttribute(key,entry.getValue());
        }
    }
    //跳转到jsp页面
    String directPath=(VIEW_PATH+path).replace("//","/");
    request.getRequestDispatcher(directPath).forward(request,response);
}
```

###### 2.5 simple-mybatis

相比于之前写的simple-mybatis，这里在SqlSessionFactoryBuilder类中添加了BeanContainer对象，在build()方法中实例化了SqlSessionFactory对象并将它存入了BeanContainer中，这样就能通过依赖注入获取到SqlSessionFactory并生成session对象实现对数据库的操作。

```java
public SqlSessionFactory build(InputStream inputStream) {
    try {
        Configuration.pros.load(inputStream);
    } catch (IOException e) {
        LogUtil.getLogger().error(e.getMessage());
    }
    SqlSessionFactory factory= DefaultSqlSessionFactory.getInstance(new Configuration());
    //将sqlFactory注入到IoC容器中
    LogUtil.getLogger().debug("load sqlSession Factory: "+factory.getClass().getName());
    BeanContainer.getInstance().addBean(factory.getClass(),factory);
    return factory;
}
```