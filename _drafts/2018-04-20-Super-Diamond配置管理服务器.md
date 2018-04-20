---
layout: page
title: Super-Diamond配置管理服务器
subtitle: Super-Diamond配置管理服务器
date: 2018-04-20 19：08：36
author: donaldhan
catalog: true
category:  Super-Diamond
categories:
    -  Super-Diamond
tags:
    -  Super-Diamond
---

# 引言

在我们配置服务器属性时候，一般常使用方法，将配置属性放在在一个配置文件中，当应用上线时，需要修改配置文件，这样容易导致手动修改配置文件出错；
自从maven的出现使我们可以，我们可以将不同环境的配置，写到不同的文件中，比如（dev，test ，exp，prod）等环境，在项目上线时，我们只需要根据
Profile属性，打包相应的属性文件，这样避免的手动修改配置引起的认为问题，但无法解决修改配置文件需要重新启动或重新打包的问题。历史的车轮，终是向前推动的。
一些配置管理服务顺应出现，比如Spring的[spring-cloud-config][], 淘宝的淘宝[diamond][],另外还有一种轻量级的配置管理服务器[ Super-Diamond][]。Spring家族的
spring-cloud-config的文档，分支版本管理比较规范，毕竟是专业滴。淘宝的diamond，不知现在淘宝现在，还在不在用，不过，在github上，搜不到淘宝的diamond，但是有个
diamond的分支，不过文档很少，这也是阿里开源产品通病。今天我们来看轻量级的配置管理服务器[Super-Diamond][]，从源码的版权声明来看是苏州科大国创信息技术有限公司的产品。



[spring-cloud-config]:https://github.com/Donaldhan/spring-cloud-config "spring-cloud-config"  
[diamond]:https://github.com/takeseem/diamond "diamond"  
[Super-Diamond]:https://github.com/Donaldhan/super-diamond "Super-Diamond"

# 目录
* [Super-Diamond架构设计](super-diamond架构设计)
    * [Super-Diamond数据库设计](#super-diamond数据库设计)
    * [Super-Diamond服务端](#super-diamond服务端)
    * [Super-Diamond客户端](#super-diamond客户端)
* [总结](#总结)

# Super-Diamond架构设计

![Super-Diamond架构设计](/image/super-diamond/framework.png)

从上图中，我们可以看到Super-Diamond主要包括两部分，一个是配置中心服务端，一个配置中心客户端。在服务中心配置，有一个配置中心后台界面我们可以管理项目和项目配置项，同时有一个
配置监听服务器。当项目配置改变时，配置监听服务器将配置更改信息以Json字符串的形式，推送到客户端。客户端启动时，开启一个定时任务以一定的间隔从服务端拉去配置信息。
由于所有的配置信息，实际上是放在数据库中，下面我们来看Super-Diamond数据库设计。

## Super-Diamond数据库设计
Super-Diamond其实是将配置信息放在数据库表中，客户端拉去时，服务端从数据库中，以Json字符串的形式，发送给客户端。主要表结构项目表，项目配置表，项目模块表，用户表。
这里我们就不给出数据库表设计图了，只简单给不表创建语句。具体如下：

### 项目表

```sql
CREATE TABLE `CONF_PROJECT` (
  `ID` int(11) NOT NULL,
  `PROJ_CODE` varchar(32) DEFAULT NULL,
  `PROJ_NAME` varchar(32) DEFAULT NULL,
  `OWNER_ID` int(11) DEFAULT NULL COMMENT '拥有者id',
  `DEVELOPMENT_VERSION` INT(11) DEFAULT 0  NULL,
  `PRODUCTION_VERSION` INT(11) DEFAULT 0  NULL,
  `TEST_VERSION` INT(11) DEFAULT 0  NULL,
  `DELETE_FLAG` int(1) DEFAULT '0',
  `CREATE_TIME` datetime DEFAULT NULL,
  `UPDATE_TIME` datetime DEFAULT NULL,
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 项目配置表

```sql
CREATE TABLE `CONF_PROJECT_CONFIG` (
  `CONFIG_ID` INT(11) NOT NULL,
  `CONFIG_KEY` VARCHAR(64) NOT NULL,
  `CONFIG_VALUE` VARCHAR(256) NOT NULL,
  `CONFIG_DESC` VARCHAR(256) DEFAULT NULL,
  `PROJECT_ID` INT(11) NOT NULL,
  `MODULE_ID` INT(11) NOT NULL,
  `DELETE_FLAG` INT(1) DEFAULT '0',
  `OPT_USER` VARCHAR(32) DEFAULT NULL,
  `OPT_TIME` DATETIME DEFAULT NULL,
  `PRODUCTION_VALUE` VARCHAR(256) NOT NULL,
  `PRODUCTION_USER` VARCHAR(32) DEFAULT NULL,
  `PRODUCTION_TIME` DATETIME DEFAULT NULL,
  `TEST_VALUE` VARCHAR(256) NOT NULL,
  `TEST_USER` VARCHAR(32) DEFAULT NULL,
  `TEST_TIME` DATETIME DEFAULT NULL,
  `BUILD_VALUE` VARCHAR(256) NOT NULL,
  `BUILD_USER` VARCHAR(32) DEFAULT NULL,
  `BUILD_TIME` DATETIME DEFAULT NULL,
  PRIMARY KEY (`CONFIG_ID`)
) ENGINE=INNODB DEFAULT CHARSET=utf8;
```
从上面可以看出，项目环境主要有测试，BUILD，和生产环境。

### 项目模块表

```sql
CREATE TABLE `CONF_PROJECT_MODULE` (
  `MODULE_ID` int(11) NOT NULL,
  `PROJ_ID` int(11) NOT NULL,
  `MODULE_NAME` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`MODULE_ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 配置用户表

```sql
CREATE TABLE `CONF_USER` (
  `ID` int(11) NOT NULL,
  `USER_CODE` varchar(32) DEFAULT NULL,
  `USER_NAME` varchar(32) NOT NULL,
  `PASSWORD` varchar(32) NOT NULL,
  `DELETE_FLAG` int(1) DEFAULT '0',
  `CREATE_TIME` datetime DEFAULT NULL,
  `UPDATE_TIME` datetime DEFAULT NULL,
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 项目用户关系表，项目用户角色权限表
```sql

CREATE TABLE `CONF_PROJECT_USER` (
  `PROJ_ID` int(11) NOT NULL,
  `USER_ID` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`PROJ_ID`,`USER_ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `CONF_PROJECT_USER_ROLE` (
  `PROJ_ID` int(11) NOT NULL,
  `USER_ID` int(11) NOT NULL,
  `ROLE_CODE` varchar(32) NOT NULL,
  PRIMARY KEY (`PROJ_ID`,`USER_ID`,`ROLE_CODE`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
从数据库的表设计可以看出，配置管理中心，关键点主要是项目和项目配置（test，build，product)

## Super-Diamond服务端

服务端两中部署方式，一种放在tomcat中，一个是放到Jetty中。我们先来看Tomcat方式：
配置监听服务器主要由Spring的应用上下文来管理，在bean工厂初始化的时候，初始化配置监听服务器。
具体的配置在applicationContext.xml配文件中：

```xml
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate" >
    	<constructor-arg index="0" ref="dataSource" />
    </bean>

    <!-- 配置事务管理器 -->
	<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">   
  		<property name="dataSource" ref="dataSource" />
 	</bean>

	<bean id="transactionTemplate" class="org.springframework.transaction.support.TransactionTemplate">
 		<property name="transactionManager" ref="transactionManager" />
 	</bean>

 	<tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true" />

	<context:component-scan base-package="com.github.diamond.web"/>

	<bean id="diamondServerHandler" class="com.github.diamond.netty.DiamondServerHandler" />
	<bean class="com.github.diamond.netty.DiamondServer">
		<property name="port" value="${netty.server.port}" />
		<property name="serverHandler" ref="diamondServerHandler" />
	</bean>
```

从上面配置来看，主要配合中心服务端的Dao使用的是JdbcTemplate。


下面我们根据服务端的几个关键类，来看一下服务端：
![Super-Diamond服务端](/image/super-diamond/server.png)
从上图中，我们可以看到两个关键的类，ConfigService, ConfigController和 ,DiamondServerHandler。

先来看DiamondServerHandle：

```java
@Sharable
public class DiamondServerHandler extends SimpleChannelInboundHandler<String> {
	public static ConcurrentHashMap<ClientKey /*projcode+profile*/, List<ClientInfo> /*client address*/> clients =
			new ConcurrentHashMap<ClientKey, List<ClientInfo>>();
	private ConcurrentHashMap<String /*client address*/, ChannelHandlerContext> channels =
			new ConcurrentHashMap<String, ChannelHandlerContext>();
    private final Charset charset = Charset.forName("UTF-8");
    @Autowired
    private ConfigService configService;
}
```
 从上面可以看出，DiamondServerHandler保存着所有客户端的信息，并且是共享模式，以便项目配置有变动时，
 将配置信息推送给所有客户端。

下面我们在来看，推送信息给客户端

```java
/**
    * 向服务端推送配置数据。
    *
    * @param projCode
    * @param profile
    * @param config
    */
   public void pushConfig(String projCode, String profile, final String module) {
     for(ClientKey key : clients.keySet()) {
       if(key.getProjCode().equals(projCode) && key.getProfile().equals(profile)) {
       List<ClientInfo> addrs = clients.get(key);
         if(addrs != null) {
           for(ClientInfo client : addrs) {
             ChannelHandlerContext ctx = channels.get(client.getAddress());
             if(ctx != null) {
               if(key.moduleArr.length == 0) {
                 String config = configService.queryConfigs(projCode, profile, "");
                 sendMessage(ctx, config);
               } else if(ArrayUtils.contains(key.getModuleArr(), module)) {
                 String config = configService.queryConfigs(projCode, key.getModuleArr(), profile, "");
                   sendMessage(ctx, config);
               }
             }
           }
         }
       }
     }
   }
```

当接收到客户端拉去配置请求时，将项目配置信息发送给客户端

```java
/**
* 处理客户端的请求
* @param request
* @throws Exception
*/
  @SuppressWarnings("unchecked")
Override
  public void channelRead0(ChannelHandlerContext ctx, String request) throws Exception {
  	String config;
      if (request != null && request.startsWith("superdiamond=")) {
      	request = request.substring("superdiamond=".length());

      	Map<String, String> params = (Map<String, String>) JSONUtils.parse(request);
      	String projCode = params.get("projCode");
      	String modules = params.get("modules");
      	String[] moduleArr = StringUtils.split(modules, ",");
      	String profile = params.get("profile");
      	ClientKey key = new ClientKey();
      	key.setProjCode(projCode);
      	key.setProfile(profile);
      	key.setModuleArr(moduleArr);
      	//String version = params.get("version");

      	List<ClientInfo> addrs = clients.get(key);
      	if(addrs == null) {
      		addrs = new ArrayList<ClientInfo>();
      	}

      	String clientAddr = ctx.channel().remoteAddress().toString();
      	ClientInfo clientInfo = new ClientInfo(clientAddr, new Date());
      	addrs.add(clientInfo);
      	clients.put(key, addrs);
      	channels.put(clientAddr, ctx);
      	//查询配置
      	if(StringUtils.isNotBlank(modules)) {
              config = configService.queryConfigs(projCode, moduleArr, profile, "");
      	} else {
      		config = configService.queryConfigs(projCode, profile, "");
      	}
      } else {
      	config = "";
      }

      sendMessage(ctx, config);
  }
  /**
  	 * 发送项目配置信息给客户端
  	 * @param ctx
  	 * @param config
  	 */
      private void sendMessage(ChannelHandlerContext ctx, String config) {
      	byte[] bytes = config.getBytes(charset);
      	ByteBuf message = Unpooled.buffer(4 + bytes.length);
          message.writeInt(bytes.length);
          message.writeBytes(bytes);
          ctx.writeAndFlush(message);
}
```

当客户端断开来连接时，移除客户端信息

```java
/**
 * 客户端断开来连接时，移除客户端信息
 * @param ctx
 * @throws Exception
 */
  @Override
  public void channelInactive(ChannelHandlerContext ctx) throws Exception {
    super.channelInactive(ctx);
    String address = ctx.channel().remoteAddress().toString();
    channels.remove(address);
    for(List<ClientInfo> infos : clients.values()) {
      for(ClientInfo client : infos) {
        if(address.equals(client.getAddress())) {
          infos.remove(client);
          break;
        }
      }
    }

    logger.info(ctx.channel().remoteAddress() + " 断开连接。");
  }
```

在来看看一下，在服务器管理界面修改，触发的操作：

```java
@Controller
public class ConfigController extends BaseController {
	private static final Logger LOGGER = LoggerFactory.getLogger(ConfigController.class);

	@Autowired
	private ConfigService configService;
	@Autowired
	private ProjectService projectService;
	@Autowired
	private ModuleService moduleService;
	@Autowired
	private DiamondServerHandler diamondServerHandler;

	/**
	 *
	 * @param type profile的值
	 * @param configId
	 * @param configKey
	 * @param configValue
	 * @param configDesc
	 * @param projectId
	 * @param moduleId
	 * @param selModuleId
	 * @param page
	 * @param flag
	 * @return
	 */
	@RequestMapping("/config/save")
	public String saveConfig(String type, Long configId, String configKey, String configValue,
			String configDesc, Long projectId, Long moduleId, Long selModuleId, int page,
			@RequestParam(defaultValue="")String flag) {
		User user = (User) SessionHolder.getSession().getAttribute("sessionUser");
		if(configId == null) {
			configService.insertConfig(configKey, configValue, configDesc, projectId, moduleId, user.getUserCode());
		} else {
			configService.updateConfig(type, configId, configKey, configValue, configDesc, projectId, moduleId, user.getUserCode());
		}

		String projCode = (String)projectService.queryProject(projectId).get("PROJ_CODE");
		String moduleName = moduleService.findName(moduleId);
		//推送最新项目配置给所有客户端
		diamondServerHandler.pushConfig(projCode, type, moduleName);
		if(selModuleId != null)
			return "redirect:/profile/" + type + "/" + projectId + "?moduleId=" + selModuleId + "&flag=" + flag;
		else
			return "redirect:/profile/" + type + "/" + projectId + "?page=" + page + "&flag=" + flag;
	}

	/**
	 * @param type
	 * @param projectId
	 * @param moduleName
	 * @param id
	 * @return
	 */
	@RequestMapping("/config/delete/{id}")
	public String deleteConfig(String type, Long projectId, String moduleName, @PathVariable Long id) {
		configService.deleteConfig(id, projectId);
		String projCode = (String)projectService.queryProject(projectId).get("PROJ_CODE");
		//推送最新项目配置给所有客户端
		diamondServerHandler.pushConfig(projCode, type, moduleName);
		return "redirect:/profile/" + type + "/" + projectId;
	}
```
我们再来看一下会话处理Handler

```java
public class SessionHolder {
    private static ThreadLocal<HttpSession> sessionThreadLocal = new ThreadLocal<HttpSession>() {

        @Override
        protected HttpSession initialValue() {
            return null;
        }

    };
    public static void remove() {
        sessionThreadLocal.remove();
    }
    public static void setSession(HttpSession session) {
        sessionThreadLocal.set(session);
    }
    public static HttpSession getSession() {
        return sessionThreadLocal.get();
    }
}
```

会话handler用ThreadLocal来管理所有会话，我很疑问，这个为什么用TheadLocal，而不是ConcurrentHashMap。


不规范地方   

#### 日志

```java
public class ConfigController extends BaseController {
	private static final Logger LOGGER = LoggerFactory.getLogger(ConfigController.class);
}
```

```java
abstract public class BaseController {
	protected static final Logger logger = LoggerFactory.getLogger(BaseController.class);
}
```

从上面可以看出，日志命名不统一。

```java
/**
 * 打印工程版本信息
 *
 * @author libinsong1204@gmail.com
 * @date 2012-3-1 下午1:20:50
 */
@SuppressWarnings("serial")
public class PrintProjectVersionServlet extends GenericServlet {
	private static final Logger logger = LoggerFactory.getLogger(PrintProjectVersionServlet.class);
	@Override
	public void init(ServletConfig config) throws ServletException {
		StringBuilder sBuilder = new StringBuilder("\n");
		try {
            Enumeration<java.net.URL> urls;
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                urls = classLoader.getResources("META-INF/res/env.properties");
            } else {
                urls = ClassLoader.getSystemResources("META-INF/res/env.properties");
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL url = urls.nextElement();
                    try {
                        BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
                        Properties properties = new Properties();
                        properties.load(reader);

                        sBuilder.append("项目名称：").append(properties.getProperty("project.name")).append(", ");
                        sBuilder.append("项目版本：").append(properties.getProperty("build.version")).append(", ");
                        sBuilder.append("构建时间：").append(properties.getProperty("build.time")).append(".\n");
                    } catch (Throwable t) {
                        logger.error(t.getMessage(), t);
                    }
                }
            }
		} catch (Throwable t) {
            logger.error(t.getMessage(), t);
        }
		String info = sBuilder.toString();
		System.out.println("===================================");
		System.out.println(info);
		System.out.println("========================================");
	}
	@Override
	public void service(ServletRequest req, ServletResponse res)
			throws ServletException, IOException {
	}
	private static ClassLoader findClassLoader() {
		return  PrintProjectVersionServlet.class.getClassLoader();
    }
}
```

摒弃System，StringBuilder -》 StringBuffer


```java
@Controller
public class SecurityController extends BaseController {
    @Autowired
    private UserService userService;

    @RequestMapping(value="/login",method = RequestMethod.POST)
    public String login(HttpServletRequest request, String userCode, String password) {
    	Object result = userService.login(userCode, password);
        if (result instanceof User) {
        	request.getSession().removeAttribute("message");
            request.getSession().setAttribute("sessionUser", result);
            return "redirect:/index";
        } else {
        	request.getSession().setAttribute("userCode", userCode);
        	request.getSession().setAttribute("message", result);
            return "redirect:/";
        }
}
```

```java
@Service
public class UserService {

	private static final Logger LOGGER = LoggerFactory.getLogger(UserService.class);

	@Autowired
	private JdbcTemplate jdbcTemplate;
	public Object login(String userCode, String password) {
		String md5Passwd = MD5.getInstance().getMD5String(password);

		try {
			String sql = "SELECT ID, USER_NAME, PASSWORD, DELETE_FLAG " +
					"FROM CONF_USER WHERE USER_CODE = ?";
			User user = jdbcTemplate.query(sql, new UserResultSetExtractor(), userCode);

			if(md5Passwd.equals(user.getPassword())) {
				user.setUserCode(userCode);
				return user;
			} else if(user.getDeleteFlag() == 1)
				return "用户已经被注销";
			else
				return "登录失败，用户密码不正确";
		} catch(TransientDataAccessResourceException e) {
			return "登录失败，用户不存在";
		}
	}
}
```
从上面可以看出，用户登录密码，转到后台是没有加密的密码，这样，容易导致密码泄漏。


基于Jetty的部署方式：
```java
public class JettyServer {
	private static final Logger LOGGER = LoggerFactory.getLogger(JettyServer.class);
	private static int maxThreads;
	private static int minThreads;
	private static int serverPort;
	private static String serverHost;
	static {
		try {
			org.apache.commons.configuration.Configuration config =
					new PropertiesConfiguration("META-INF/res/jetty.properties");

			LOGGER.info("加载jetty.properties");

			maxThreads = config.getInt("thread.pool.max.size", 100);
			minThreads = config.getInt("thread.pool.min.size", 10);
			serverPort = config.getInt("server.port", 8080);
			serverHost = config.getString("server.host");
		} catch(Exception e) {
			LOGGER.error(e.getMessage(), e);
		}
	}
  //...
}
```


## Super-Diamond客户端

![Super-Diamond客户端](/image/super-diamond/client.png)

# 总结
