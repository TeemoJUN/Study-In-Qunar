# QConfig

````
<Context debug="0" docBase="D:\demo1\web" path="/demo1" privileged="true" reloadable="true"/>
````
```

<parent>
    <groupId>qunar.common</groupId>
    <artifactId>qunar-supom-generic</artifactId>
    <version>1.3.2</version>
</parent>
 
 
<!--在父pom中配置版本子pom中依赖的信息-->

<properties>
  <qunar.common.version>8.2.7</qunar.common.version>
  <qconfig.version>1.1.4</qconfig.version>
</properties>

```

```
<dependency>
    <groupId>qunar.common</groupId>
    <artifactId>common-core</artifactId>
</dependency>
<dependency>
    <groupId>qunar.tc.qconfig</groupId>
    <artifactId>qconfig-client</artifactId>
</dependency>
```



### 四种加载方式
(1)

````
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:qconfig="http://www.qunar.com/schema/qconfig"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.qunar.com/schema/qconfig http://www.qunar.com/schema/qconfig.xsd
         http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
````

````
<!-- 这里可以写多个文件，用逗号分隔 -->
<qconfig:config files="zookeeper.properties,mysql.properties"/>
 
<!-- 如果想写多个 qconfig:config，不是写在一行，请注意这里的ignore-unresolvable  -->
 <qconfig:config files="config.properties" ignore-unresolvable="true" />
 <qconfig:config files="configt.properties" ignore-unresolvable="true" />
 
<!-- 如果还需要引入spring原生的properties文件呢？ -->
<context:property-placeholder location="classpath:xxxx.properties" ignore-unresolvable="true"/>
 
<!-- 其他地方的使用方式与properties文件的方式一致 -->
<bean id="orderService" class="com.qunar.hotel.OrderService">
   <property name="zkAddress" value="${zkAddress}" />
</bean>
````

注意：当配置文件不是写在一行，而使用了多个<qconfig:config files="filename" ignore-unresolvable="true" />来配置多个配置文件，
或<qconfig:config files="config.properties" ignore-unresolvable="true" />标签与<context:property-placeholder location="classpath:xxxx.properties" ignore-unresolvable="true"/>标签混用时，
都需要加上ignore-unresolvable="true"属性，否则会报unresolved异常。

或者前面的都用true最后一个用false

[context:property-placeholder](http://www.iteye.com/topic/1131688)

1、在PropertyPlaceholderBeanDefinitionParser的父类中shouldGenerateId返回true，即默认会为每一个bean生成一个唯一的名字； 如果使用了两个<context:property-placeholder则注册了两个PropertySourcesPlaceholderConfigurer Bean；所以不是覆盖（而且bean如果同名是后边的bean定义覆盖前边的）； 

2、PropertySourcesPlaceholderConfigurer本质是一个BeanFactoryPostProcessor，spring实施时如果发现这个bean实现了Ordered，则按照顺序执行；默认无序； 

3、此时如果给<context:property-placeholder加order属性，则会反应出顺序，值越小优先级越高即越早执行； 
比如 
   <context:property-placeholder order="2" location="classpath*:conf/conf_a.properties"/>  
   <context:property-placeholder order="1" location="classpath*:conf/conf_b.properties"/> 

此时会先扫描order='1' 的，如果没有扫描order='2'的 


4、默认情况下ignore-unresolvable；即如果没找到的情况是否抛出异常。默认false：即抛出异常； 
<context:property-placeholder location="classpath*:conf/conf_a.properties" ignore-unresolvable="false"/> 



ignore-unresolvable：是否忽略解析不到的属性，如果不忽略，找不到将抛出异常;默认false：即抛出异常



（2）使用@value注入:使用Spring的 @Value 没有自动更新功能
````
<context:annotation-config />

@Value("${test}")
private String city;
````


（3）使用@QConfig注解的方式

````
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:qconfig="http://www.qunar.com/schema/qconfig"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.qunar.com/schema/qconfig http://www.qunar.com/schema/qconfig.xsd
         http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
  <context:annotation-config />
 
  <qconfig:annotation-driven />
````
````
//这个类需要配置成Spring的bean
public class YourClass{
 
  @QConfig("config2.properties")
  private Properties config;
 
  //以下config都是动态变化的，config.properties发生变更后，每次都去config获取值都是最新值
 
  //支持Map<String,String>
  @QConfig("config1.properties")
  private Map<String, String> config;
 
  //还支持String，config里的内容是config.properties里完整内容，业务可以自行解析
  @QConfig("config3.properties")
  private String config;
}
````
（4）使用原生API：使用原生api获取文件名为filename的文件时并不需要在spring配置中使用<qconfig:config files="filename" />，只有在需要配合spring进行属性注入时才需要使用qconfig:config标签

1）map型listener

 
````
MapConfig config = MapConfig.get("config.properties");
//这个map是动态变化的
Map<String, String> map = config.asMap();
//注册监听
config.addListener(new Configuration.ConfigListener<Map<String, String>>() {
  @Override //文件加载成功或者有新版本触发
  public void onLoad(Map<String, String> conf) {
     for (Map.Entry<String, String> entry : conf.entrySet()) {
         logger.info(entry.getKey() + "=>" + entry.getValue());
     }
  }
});

````
2）PropertiesChangeListener
与map型listener不同，PropertiesChangeListener只有在properties文件内容有变化时才触发。如果文件里面的property为空，那么添加listener的操作并不会触发调用。
````
MapConfig config = MapConfig.get("config.properties");
config.asMap();
config.addPropertiesListener(new MapConfig.PropertiesChangeListener() {
  @Override //配置发生变更的时候会触发，没有变更不触发
  public void onChange(PropertiesChange change) {
     // ...
  }
});
````

### 自定义类型

1. 自定义格式文件需要自行解析
````
TypedConfig<MyConfig> config = TypedConfig.get("config.cus", new TypedConfig.Parser<MyConfig>(){
   @Override
   public MyConfig parse(String data) throws IOException{
       //自行解析配置
   }
});
 
//MyConfig是你自己定义的一个类
MyConfig config = config.current();
 
//或订阅变更
config.addListener(new ConfigListener<MyConfig>(){
  @Override //配置发生变更的时候会触发
  public void onLoad(MyConfig conf) {
     //
  }
});

````

[QConfig原理解析](http://wiki.corp.qunar.com/confluence/pages/viewpage.action?pageId=94011409)


![](d.jpg)


#### QConfigAnnotationProcessor.postProcessBeforeInitialization
   1.解析字段： parseFields(bean, bean.getClass().getDeclaredFields());
                 第一次获取：  MapConfig.get(entry.getKey(), entry.getValue());
                 添加事件监听
   2.解析方法：parseMethods(bean, bean.getClass().getDeclaredMethods());
   
   3.MapConfig.get(filename) 调用 AbstractDataLoader.load()
   
#### 定时任务加载  AbstractDataLoader.setupPoller()
      AbstractDataLoader初始化时，调用静态初始化块static{setupPoller()}，
      setupPoller()方法中使用s定时器，周期性地调用reLoading()方法
      
#### PushRequestHandler.handle()没有找到

Spring的BeanPostProcessor接口，该接口作用是：如果我们需要在Spring容器完成Bean的实例化，配置和其他的初始化后添加一些自己的逻辑处理，
就可以定义一个或者多个BeanPostProcessor接口的实现。



+ parseFields()中拉取配置文件的代码在于这一句:
MapConfig config = MapConfig.get(fileName,Feature);
//返回出配置文件内容后，赋值给字段
setField(field, bean, config.asMap());


+ MapConfig.get()中：[一般没有多套配置文件时为null，server根据client端address自动判断属于哪个环节]
return (MapConfig) AbstractDataLoader.load(groupName, fileName, Feature, DataLoader.Generator);

+ public class AbstractData
private static final ConcurrentMap<String, FileStore> configs = Maps.newConcurrentMap();//静态存储，每轮到一个文件，在这个集合里记录一次，最后记录了需要的配置文件对应的存储工具类
private static final ConcurrentMap<Meta, Version> versions = new ConcurrentHashMap<Meta, Version>();//最后记录了所有需要的配置文件的版本信息
FileStore store = configs.get(key);  //拿到被注释指定了名称的那个配置文件的存储工具类
String key = Meta.createKey(groupName, fileName);  //创建文件标识


VersionProfile version = VersionProfile.ABSENT;//代表无效版本
Version ver = versions.get(store.getMeta());//拿到指定配置文件的版本信息
if (ver != null) {      
//版本信息不为空，说明在本地缓存有该文件的旧版本，进行版本更新
    final Version nVer = ver;
    nVer.addListener(new Runnable() {
        @Override
        public void run() {
            VersionProfile newVer = temp.initLoad(nVer.updated.get(), client);//使用httpClient建立连接，首次拉取最新配置文件
            nVer.setUpdated(newVer);//把该文件的最新版本信息跟新到versions Map中
        }
    }, executor);
}

+ 如果版本信息不为空
VersionProfile newVer = temp.initLoad(nVer.updated.get(), client);//获取最新版本
versions.putIfAbsent(store.getMeta(), new Version(newVer));//把该文件的最新版本信息更新到versions Map中
ver = versions.get(store.getMeta());
ver.setUpdated(newVer);

+ 将fileStore拉取到的数据以用户自定方式存在config中，比如MapConfig,然后返回赋给指定字段
bstractConfiguration config = store.getConfig();
if (!config.getClass().equals(conf.getClass())) {
            throw new RuntimeException("同一个配置文件只能以一种方式读取: MapConfig或TypedConfig或TableConfig");
        }
return config;



开发人员在本地开发测试的时候可以不依赖配置中心，只需要在源程序的resources资源文件夹下创建一个qconfig_test目录，
将需要在本地覆盖的配置文件放置在应用名目录(应用名即在应用中心中申请的名称)里面即可。比如应用名称叫qtest，那么在qconfig_test目录下再建立一个qtest目录。qconfig_test会被bds过滤，不会发布到线上。举例：假设你的应用名为qtask，
有一个配置文件mysql.properties，你在本地测试的时候想使用这个文件，而不使用配置中心中的配置，
则将mysql.properties这个文件放在/resources/qconfig_test/qtask/mysql.properties即可。
