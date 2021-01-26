---
layout: post
title: 定时任务调度实战2
category: springboot
tags: [springboot]
keywords: springboot
excerpt: springboot + quartz + mysql 实现持久化分布式调度，springboot整合xxl-job分布式调度任务平台
lock: noneed
---

## 前言

前面在定时任务调度实战1中介绍了 Quartz 的单体应用实践，如果只在单体环境中应用，Quartz 未必是最好的选择，例如`Spring Scheduled`一样也可以实现任务调度，并且与`SpringBoot`无缝集成，支持注解配置，非常简单，但是它有个缺点就是在集群环境下，会导致任务被重复调度！

而与之对应的 Quartz 提供了极为广用的特性，如任务持久化、集群部署和分布式调度任务等等，正因如此，基于 Quartz 任务调度功能在系统开发中应用极为广泛！

## 1、Quartz 集群架构

在集群环境下，Quartz 集群中的每个节点是一个独立的 Quartz 应用，没有负责集中管理的节点，而是**通过数据库表来感知另一个应用，利用数据库锁**的方式来实现集群环境下进行并发控制，每个任务当前运行的有效节点有且只有一个！

![](\assets\images\2021\juc\quartz-cluster.jpg)

特别需要注意的是：分布式部署时需要保证各个节点的系统时间一致！

下面我们一起来看看具体的应用实践！

### 数据表初始化

**数据库表结构**官网已经提供，我们可以直接访问`Quartz`对应的官方网站，找到对应的版本，然后将其下载！小编我选择的是`quartz-2.3.0-distribution.tar.gz`，下载完成之后将其解压，在文件中搜索`sql`，在里面选择适合当前环境的数据库脚本文件，然后将其初始化到数据库中即可

![](\assets\images\2021\juc\quartz-distribute-table-sql.jpg)

例如，我使用的数据库是`mysql-5.7`，因此我选择的是`tables_mysql_innodb.sql`脚本，具体内容如下：

```sql
DROP TABLE IF EXISTS QRTZ_FIRED_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_PAUSED_TRIGGER_GRPS;
DROP TABLE IF EXISTS QRTZ_SCHEDULER_STATE;
DROP TABLE IF EXISTS QRTZ_LOCKS;
DROP TABLE IF EXISTS QRTZ_SIMPLE_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_SIMPROP_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_CRON_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_BLOB_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_TRIGGERS;
DROP TABLE IF EXISTS QRTZ_JOB_DETAILS;
DROP TABLE IF EXISTS QRTZ_CALENDARS;

CREATE TABLE QRTZ_JOB_DETAILS(
SCHED_NAME VARCHAR(120) NOT NULL,
JOB_NAME VARCHAR(190) NOT NULL,
JOB_GROUP VARCHAR(190) NOT NULL,
DESCRIPTION VARCHAR(250) NULL,
JOB_CLASS_NAME VARCHAR(250) NOT NULL,
IS_DURABLE VARCHAR(1) NOT NULL,
IS_NONCONCURRENT VARCHAR(1) NOT NULL,
IS_UPDATE_DATA VARCHAR(1) NOT NULL,
REQUESTS_RECOVERY VARCHAR(1) NOT NULL,
JOB_DATA BLOB NULL,
PRIMARY KEY (SCHED_NAME,JOB_NAME,JOB_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_NAME VARCHAR(190) NOT NULL,
TRIGGER_GROUP VARCHAR(190) NOT NULL,
JOB_NAME VARCHAR(190) NOT NULL,
JOB_GROUP VARCHAR(190) NOT NULL,
DESCRIPTION VARCHAR(250) NULL,
NEXT_FIRE_TIME BIGINT(13) NULL,
PREV_FIRE_TIME BIGINT(13) NULL,
PRIORITY INTEGER NULL,
TRIGGER_STATE VARCHAR(16) NOT NULL,
TRIGGER_TYPE VARCHAR(8) NOT NULL,
START_TIME BIGINT(13) NOT NULL,
END_TIME BIGINT(13) NULL,
CALENDAR_NAME VARCHAR(190) NULL,
MISFIRE_INSTR SMALLINT(2) NULL,
JOB_DATA BLOB NULL,
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
FOREIGN KEY (SCHED_NAME,JOB_NAME,JOB_GROUP)
REFERENCES QRTZ_JOB_DETAILS(SCHED_NAME,JOB_NAME,JOB_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_SIMPLE_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_NAME VARCHAR(190) NOT NULL,
TRIGGER_GROUP VARCHAR(190) NOT NULL,
REPEAT_COUNT BIGINT(7) NOT NULL,
REPEAT_INTERVAL BIGINT(12) NOT NULL,
TIMES_TRIGGERED BIGINT(10) NOT NULL,
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_CRON_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_NAME VARCHAR(190) NOT NULL,
TRIGGER_GROUP VARCHAR(190) NOT NULL,
CRON_EXPRESSION VARCHAR(120) NOT NULL,
TIME_ZONE_ID VARCHAR(80),
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_SIMPROP_TRIGGERS
  (
    SCHED_NAME VARCHAR(120) NOT NULL,
    TRIGGER_NAME VARCHAR(190) NOT NULL,
    TRIGGER_GROUP VARCHAR(190) NOT NULL,
    STR_PROP_1 VARCHAR(512) NULL,
    STR_PROP_2 VARCHAR(512) NULL,
    STR_PROP_3 VARCHAR(512) NULL,
    INT_PROP_1 INT NULL,
    INT_PROP_2 INT NULL,
    LONG_PROP_1 BIGINT NULL,
    LONG_PROP_2 BIGINT NULL,
    DEC_PROP_1 NUMERIC(13,4) NULL,
    DEC_PROP_2 NUMERIC(13,4) NULL,
    BOOL_PROP_1 VARCHAR(1) NULL,
    BOOL_PROP_2 VARCHAR(1) NULL,
    PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
    FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
    REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_BLOB_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_NAME VARCHAR(190) NOT NULL,
TRIGGER_GROUP VARCHAR(190) NOT NULL,
BLOB_DATA BLOB NULL,
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),
INDEX (SCHED_NAME,TRIGGER_NAME, TRIGGER_GROUP),
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_CALENDARS (
SCHED_NAME VARCHAR(120) NOT NULL,
CALENDAR_NAME VARCHAR(190) NOT NULL,
CALENDAR BLOB NOT NULL,
PRIMARY KEY (SCHED_NAME,CALENDAR_NAME))
ENGINE=InnoDB;

CREATE TABLE QRTZ_PAUSED_TRIGGER_GRPS (
SCHED_NAME VARCHAR(120) NOT NULL,
TRIGGER_GROUP VARCHAR(190) NOT NULL,
PRIMARY KEY (SCHED_NAME,TRIGGER_GROUP))
ENGINE=InnoDB;

CREATE TABLE QRTZ_FIRED_TRIGGERS (
SCHED_NAME VARCHAR(120) NOT NULL,
ENTRY_ID VARCHAR(95) NOT NULL,
TRIGGER_NAME VARCHAR(190) NOT NULL,
TRIGGER_GROUP VARCHAR(190) NOT NULL,
INSTANCE_NAME VARCHAR(190) NOT NULL,
FIRED_TIME BIGINT(13) NOT NULL,
SCHED_TIME BIGINT(13) NOT NULL,
PRIORITY INTEGER NOT NULL,
STATE VARCHAR(16) NOT NULL,
JOB_NAME VARCHAR(190) NULL,
JOB_GROUP VARCHAR(190) NULL,
IS_NONCONCURRENT VARCHAR(1) NULL,
REQUESTS_RECOVERY VARCHAR(1) NULL,
PRIMARY KEY (SCHED_NAME,ENTRY_ID))
ENGINE=InnoDB;

CREATE TABLE QRTZ_SCHEDULER_STATE (
SCHED_NAME VARCHAR(120) NOT NULL,
INSTANCE_NAME VARCHAR(190) NOT NULL,
LAST_CHECKIN_TIME BIGINT(13) NOT NULL,
CHECKIN_INTERVAL BIGINT(13) NOT NULL,
PRIMARY KEY (SCHED_NAME,INSTANCE_NAME))
ENGINE=InnoDB;

CREATE TABLE QRTZ_LOCKS (
SCHED_NAME VARCHAR(120) NOT NULL,
LOCK_NAME VARCHAR(40) NOT NULL,
PRIMARY KEY (SCHED_NAME,LOCK_NAME))
ENGINE=InnoDB;

CREATE INDEX IDX_QRTZ_J_REQ_RECOVERY ON QRTZ_JOB_DETAILS(SCHED_NAME,REQUESTS_RECOVERY);
CREATE INDEX IDX_QRTZ_J_GRP ON QRTZ_JOB_DETAILS(SCHED_NAME,JOB_GROUP);

CREATE INDEX IDX_QRTZ_T_J ON QRTZ_TRIGGERS(SCHED_NAME,JOB_NAME,JOB_GROUP);
CREATE INDEX IDX_QRTZ_T_JG ON QRTZ_TRIGGERS(SCHED_NAME,JOB_GROUP);
CREATE INDEX IDX_QRTZ_T_C ON QRTZ_TRIGGERS(SCHED_NAME,CALENDAR_NAME);
CREATE INDEX IDX_QRTZ_T_G ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_GROUP);
CREATE INDEX IDX_QRTZ_T_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_STATE);
CREATE INDEX IDX_QRTZ_T_N_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP,TRIGGER_STATE);
CREATE INDEX IDX_QRTZ_T_N_G_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_GROUP,TRIGGER_STATE);
CREATE INDEX IDX_QRTZ_T_NEXT_FIRE_TIME ON QRTZ_TRIGGERS(SCHED_NAME,NEXT_FIRE_TIME);
CREATE INDEX IDX_QRTZ_T_NFT_ST ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_STATE,NEXT_FIRE_TIME);
CREATE INDEX IDX_QRTZ_T_NFT_MISFIRE ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME);
CREATE INDEX IDX_QRTZ_T_NFT_ST_MISFIRE ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME,TRIGGER_STATE);
CREATE INDEX IDX_QRTZ_T_NFT_ST_MISFIRE_GRP ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME,TRIGGER_GROUP,TRIGGER_STATE);

CREATE INDEX IDX_QRTZ_FT_TRIG_INST_NAME ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,INSTANCE_NAME);
CREATE INDEX IDX_QRTZ_FT_INST_JOB_REQ_RCVRY ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,INSTANCE_NAME,REQUESTS_RECOVERY);
CREATE INDEX IDX_QRTZ_FT_J_G ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,JOB_NAME,JOB_GROUP);
CREATE INDEX IDX_QRTZ_FT_JG ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,JOB_GROUP);
CREATE INDEX IDX_QRTZ_FT_T_G ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP);
CREATE INDEX IDX_QRTZ_FT_TG ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,TRIGGER_GROUP);

commit;
```

|           表名           |                             描述                             |
| :----------------------: | :----------------------------------------------------------: |
|    QRTZ_BLOG_TRIGGERS    |                   Trigger作为Blob类型存储                    |
|      QRTZ_CALENDARS      |                   存储Quartz的Calendar信息                   |
|    QRTZ_CRON_TRIGGERS    |          存储CronTrigger，包括Cron表达式和时区信息           |
|   QRTZ_FIRED_TRIGGERS    |  存储与已触发的Trigger相关的状态信息，以及相联Job的执行信息  |
|     QRTZ_JOB_DETAILS     |               存储每一个已配置的Job的详细信息                |
|      **QRTZ_LOCKS**      |                  **存储程序的悲观锁的信息**                  |
| QRTZ_PAUSED_TRIGGER_GRPS |                 存储已暂停的Trigger组的信息                  |
|   QRTZ_SCHEDULER_STATE   |    存储少量的有关Scheduler的状态信息，和别的Scheduler实例    |
|   QRTZ_SIMPLE_TRIGGERS   |    存储简单的Trigger，包括重复次数、间隔、以及已触的次数     |
|  QRTZ_SIMPROP_TRIGGERS   | 存储CalendarIntervalTrigger和DailyTimeIntervalTrigger两种类型的触发器 |
|      QRTZ_TRIGGERS       |                  存储已配置的Trigger的信息                   |

其中，QRTZ_LOCKS 就是 Quartz 集群实现同步机制的行锁表！

### 项目实践

创建一个SpringBoot项目

1、pom.xml导入依赖

```xml
<!--引入boot父类-->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.0.RELEASE</version>
</parent>

<!--引入相关包-->
<dependencies>
    <!--spring boot核心-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <!--spring boot 测试-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <!--springmvc web-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!--开发环境调试-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>
    <!--jpa 支持-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <!--mysql 数据源-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <!--druid 数据连接池-->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.1.17</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-quartz</artifactId>
    </dependency>
    <!--Alibaba Json处理包 -->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.46</version>
    </dependency>
  <dependency>
		  <groupId>org.projectlombok</groupId>
		  <artifactId>lombok</artifactId>
		  <version>1.18.12</version>
	  </dependency>
</dependencies>
```

2、application.properties配置文件

```properties
spring.application.name=springboot-quartz-001
server.port=8080

#引入数据源
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

创建quartz.properties

```properties
#调度配置
#调度器实例名称
org.quartz.scheduler.instanceName=SsmScheduler
#调度器实例编号自动生成
org.quartz.scheduler.instanceId=AUTO
#是否在Quartz执行一个job前使用UserTransaction
org.quartz.scheduler.wrapJobExecutionInUserTransaction=false

#线程池配置
#线程池的实现类
org.quartz.threadPool.class=org.quartz.simpl.SimpleThreadPool
#线程池中的线程数量
org.quartz.threadPool.threadCount=10
#线程优先级
org.quartz.threadPool.threadPriority=5
#配置是否启动自动加载数据库内的定时任务，默认true
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread=true
#是否设置为守护线程，设置后任务将不会执行
#org.quartz.threadPool.makeThreadsDaemons=true

#持久化方式配置
#JobDataMaps是否都为String类型
org.quartz.jobStore.useProperties=true
#数据表的前缀，默认QRTZ_
org.quartz.jobStore.tablePrefix=QRTZ_
#最大能忍受的触发超时时间
org.quartz.jobStore.misfireThreshold=60000
#是否以集群方式运行
org.quartz.jobStore.isClustered=true
#调度实例失效的检查时间间隔，单位毫秒
org.quartz.jobStore.clusterCheckinInterval=2000
#数据保存方式为数据库持久化
org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX
#数据库代理类，一般org.quartz.impl.jdbcjobstore.StdJDBCDelegate可以满足大部分数据库
org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.StdJDBCDelegate
#数据库别名 随便取
org.quartz.jobStore.dataSource=qzDS

#数据库连接池，将其设置为druid
org.quartz.dataSource.qzDS.connectionProvider.class=com.example.cluster.quartz.config.DruidConnectionProvider
#数据库引擎
org.quartz.dataSource.qzDS.driver=com.mysql.cj.jdbc.Driver
#数据库连接
org.quartz.dataSource.qzDS.URL=jdbc:mysql://127.0.0.1:3306/test-quartz?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
#数据库用户
org.quartz.dataSource.qzDS.user=root
#数据库密码
org.quartz.dataSource.qzDS.password=123456
#允许最大连接
org.quartz.dataSource.qzDS.maxConnection=5
#验证查询sql,可以不设置
org.quartz.dataSource.qzDS.validationQuery=select 0 from dual
```

3、注册启动插件Quartz任务工厂

```java
@Component
public class QuartzJobFactory extends AdaptableJobFactory {

    @Autowired
    private AutowireCapableBeanFactory capableBeanFactory;

    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
        //调用父类的方法
        Object jobInstance = super.createJobInstance(bundle);
        //进行注入
        capableBeanFactory.autowireBean(jobInstance);
        return jobInstance;
    }
}
```

4、quartz配置类 QuartzConfig.java

```java
@Configuration
public class QuartzConfig {
    @Autowired
    private QuartzJobFactory jobFactory;

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean() throws IOException {
        //获取配置属性
        PropertiesFactoryBean propertiesFactoryBean = new PropertiesFactoryBean();
        propertiesFactoryBean.setLocation(new ClassPathResource("/quartz.properties"));
        //在quartz.properties中的属性被读取并注入后再初始化对象
        propertiesFactoryBean.afterPropertiesSet();
        //创建SchedulerFactoryBean
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setQuartzProperties(propertiesFactoryBean.getObject());
        factory.setJobFactory(jobFactory);//支持在JOB实例中注入其他的业务对象
        factory.setApplicationContextSchedulerContextKey("applicationContextKey");
        factory.setWaitForJobsToCompleteOnShutdown(true);//这样当spring关闭时，会等待所有已经启动的quartz job结束后spring才能完全shutdown。
        factory.setOverwriteExistingJobs(false);//是否覆盖己存在的Job
        factory.setStartupDelay(10);//QuartzScheduler 延时启动，应用启动完后 QuartzScheduler 再启动

        return factory;
    }

    /**
     * 通过SchedulerFactoryBean获取Scheduler的实例
     * @return
     * @throws IOException
     * @throws SchedulerException
     */
    @Bean(name = "scheduler")
    public Scheduler scheduler() throws IOException, SchedulerException {
        Scheduler scheduler = schedulerFactoryBean().getScheduler();
        return scheduler;
    }
}
```

5、修改数据库连接

默认 Quartz 的数据连接池是 **c3p0**，由于性能不太稳定，不推荐使用，因此我们将其改成`driud`数据连接池

```java
public class DruidConnectionProvider implements ConnectionProvider {

    /**
     * 常量配置，与quartz.properties文件的key保持一致(去掉前缀)，同时提供set方法，Quartz框架自动注入值。
     * @return
     * @throws SQLException
     */

    //JDBC驱动
    public String driver;
    //JDBC连接串
    public String URL;
    //数据库用户名
    public String user;
    //数据库用户密码
    public String password;
    //数据库最大连接数
    public int maxConnection;
    //数据库SQL查询每次连接返回执行到连接池，以确保它仍然是有效的。
    public String validationQuery;

    private boolean validateOnCheckout;

    private int idleConnectionValidationSeconds;

    public String maxCachedStatementsPerConnection;

    private String discardIdleConnectionsSeconds;

    public static final int DEFAULT_DB_MAX_CONNECTIONS = 10;

    public static final int DEFAULT_DB_MAX_CACHED_STATEMENTS_PER_CONNECTION = 120;

    //Druid连接池
    private DruidDataSource datasource;

    @Override
    public Connection getConnection() throws SQLException {
        return datasource.getConnection();
    }

    @Override
    public void shutdown() throws SQLException {
        datasource.close();
    }

    @Override
    public void initialize() throws SQLException {
        if (this.URL == null) {
            throw new SQLException("DBPool could not be created: DB URL cannot be null");
        }

        if (this.driver == null) {
            throw new SQLException("DBPool driver could not be created: DB driver class name cannot be null!");
        }

        if (this.maxConnection < 0) {
            throw new SQLException("DBPool maxConnectins could not be created: Max connections must be greater than zero!");
        }

        datasource = new DruidDataSource();
        try{
            datasource.setDriverClassName(this.driver);
        } catch (Exception e) {
            try {
                throw new SchedulerException("Problem setting driver class name on datasource: " + e.getMessage(), e);
            } catch (SchedulerException e1) {
            }
        }

        datasource.setUrl(this.URL);
        datasource.setUsername(this.user);
        datasource.setPassword(this.password);
        datasource.setMaxActive(this.maxConnection);
        datasource.setMinIdle(1);
        datasource.setMaxWait(0);
        datasource.setMaxPoolPreparedStatementPerConnectionSize(DEFAULT_DB_MAX_CONNECTIONS);

        if (this.validationQuery != null) {
            datasource.setValidationQuery(this.validationQuery);
            if(!this.validateOnCheckout)
                datasource.setTestOnReturn(true);
            else
                datasource.setTestOnBorrow(true);
            datasource.setValidationQueryTimeout(this.idleConnectionValidationSeconds);
        }
    }

    public String getDriver() {
        return driver;
    }

    public void setDriver(String driver) {
        this.driver = driver;
    }

    public String getURL() {
        return URL;
    }

    public void setURL(String URL) {
        this.URL = URL;
    }

    public String getUser() {
        return user;
    }

    public void setUser(String user) {
        this.user = user;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public int getMaxConnection() {
        return maxConnection;
    }

    public void setMaxConnection(int maxConnection) {
        this.maxConnection = maxConnection;
    }

    public String getValidationQuery() {
        return validationQuery;
    }

    public void setValidationQuery(String validationQuery) {
        this.validationQuery = validationQuery;
    }

    public boolean isValidateOnCheckout() {
        return validateOnCheckout;
    }

    public void setValidateOnCheckout(boolean validateOnCheckout) {
        this.validateOnCheckout = validateOnCheckout;
    }

    public int getIdleConnectionValidationSeconds() {
        return idleConnectionValidationSeconds;
    }

    public void setIdleConnectionValidationSeconds(int idleConnectionValidationSeconds) {
        this.idleConnectionValidationSeconds = idleConnectionValidationSeconds;
    }

    public DruidDataSource getDatasource() {
        return datasource;
    }

    public void setDatasource(DruidDataSource datasource) {
        this.datasource = datasource;
    }

    public String getDiscardIdleConnectionsSeconds() {
        return discardIdleConnectionsSeconds;
    }

    public void setDiscardIdleConnectionsSeconds(String discardIdleConnectionsSeconds) {
        this.discardIdleConnectionsSeconds = discardIdleConnectionsSeconds;
    }
}
```

创建完成之后，还需要在`quartz.properties`配置文件中设置一下

```properties
#数据库连接池，将其设置为druid
org.quartz.dataSource.qzDS.connectionProvider.class=com.example.cluster.quartz.config.DruidConnectionProvider
```

6、编写Job具体任务类

```java
public class TfCommandJob implements Job {

    private static final Logger log = LoggerFactory.getLogger(TfCommandJob.class);

    @Override
    public void execute(JobExecutionContext context) {
        try {
            System.out.println(context.getScheduler().getSchedulerInstanceId() + "--" + new SimpleDateFormat("YYYY-MM-dd HH:mm:ss").format(new Date()));
        } catch (SchedulerException e) {
            log.error("任务执行失败",e);
        }
    }
}
```

编写服务层接口QuartzJobService.java

```java
public interface QuartzJobService {
    /**
     * 添加任务可以传参数
     * @param clazzName
     * @param jobName
     * @param groupName
     * @param cronExp
     * @param param
     */
    void addJob(String clazzName, String jobName, String groupName, String cronExp, Map<String, Object> param);

    /**
     * 暂停任务
     * @param jobName
     * @param groupName
     */
    void pauseJob(String jobName, String groupName);

    /**
     * 恢复任务
     * @param jobName
     * @param groupName
     */
    void resumeJob(String jobName, String groupName);

    /**
     * 立即运行一次定时任务
     * @param jobName
     * @param groupName
     */
    void runOnce(String jobName, String groupName);

    /**
     * 更新任务
     * @param jobName
     * @param groupName
     * @param cronExp
     * @param param
     */
    void updateJob(String jobName, String groupName, String cronExp, Map<String, Object> param);

    /**
     * 删除任务
     * @param jobName
     * @param groupName
     */
    void deleteJob(String jobName, String groupName);

    /**
     * 启动所有任务
     */
    void startAllJobs();

    /**
     * 暂停所有任务
     */
    void pauseAllJobs();

    /**
     * 恢复所有任务
     */
    void resumeAllJobs();

    /**
     * 关闭所有任务
     */
    void shutdownAllJobs();
}
```

实现类QuartzJobServiceImpl.java

```java
@Service
public class QuartzJobServiceImpl implements QuartzJobService {

    private static final Logger log = LoggerFactory.getLogger(QuartzJobServiceImpl.class);

    @Autowired
    private Scheduler scheduler;

    @Override
    public void addJob(String clazzName, String jobName, String groupName, String cronExp, Map<String, Object> param) {
        try {
            // 启动调度器，默认初始化的时候已经启动
//            scheduler.start();
            //构建job信息
            Class<? extends Job> jobClass = (Class<? extends Job>) Class.forName(clazzName);
            JobDetail jobDetail = JobBuilder.newJob(jobClass).withIdentity(jobName, groupName).build();
            //表达式调度构建器(即任务执行的时间)
            CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(cronExp);
            //按新的cronExpression表达式构建一个新的trigger
            CronTrigger trigger = TriggerBuilder.newTrigger().withIdentity(jobName, groupName).withSchedule(scheduleBuilder).build();
            //获得JobDataMap，写入数据
            if (param != null) {
                trigger.getJobDataMap().putAll(param);
            }
            scheduler.scheduleJob(jobDetail, trigger);
        } catch (Exception e) {
            log.error("创建任务失败", e);
        }
    }

    @Override
    public void pauseJob(String jobName, String groupName) {
        try {
            scheduler.pauseJob(JobKey.jobKey(jobName, groupName));
        } catch (SchedulerException e) {
            log.error("暂停任务失败", e);
        }
    }

    @Override
    public void resumeJob(String jobName, String groupName) {
        try {
            scheduler.resumeJob(JobKey.jobKey(jobName, groupName));
        } catch (SchedulerException e) {
            log.error("恢复任务失败", e);
        }
    }

    @Override
    public void runOnce(String jobName, String groupName) {
        try {
            scheduler.triggerJob(JobKey.jobKey(jobName, groupName));
        } catch (SchedulerException e) {
            log.error("立即运行一次定时任务失败", e);
        }
    }

    @Override
    public void updateJob(String jobName, String groupName, String cronExp, Map<String, Object> param) {
        try {
            TriggerKey triggerKey = TriggerKey.triggerKey(jobName, groupName);
            CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
            if (cronExp != null) {
                // 表达式调度构建器
                CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(cronExp);
                // 按新的cronExpression表达式重新构建trigger
                trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).build();
            }
            //修改map
            if (param != null) {
                trigger.getJobDataMap().putAll(param);
            }
            // 按新的trigger重新设置job执行
            scheduler.rescheduleJob(triggerKey, trigger);
        } catch (Exception e) {
            log.error("更新任务失败", e);
        }
    }

    @Override
    public void deleteJob(String jobName, String groupName) {
        try {
            //暂停、移除、删除
            scheduler.pauseTrigger(TriggerKey.triggerKey(jobName, groupName));
            scheduler.unscheduleJob(TriggerKey.triggerKey(jobName, groupName));
            scheduler.deleteJob(JobKey.jobKey(jobName, groupName));
        } catch (Exception e) {
            log.error("删除任务失败", e);
        }
    }

    @Override
    public void startAllJobs() {
        try {
            scheduler.start();
        } catch (Exception e) {
            log.error("开启所有的任务失败", e);
        }
    }

    @Override
    public void pauseAllJobs() {
        try {
            scheduler.pauseAll();
        } catch (Exception e) {
            log.error("暂停所有任务失败", e);
        }
    }

    @Override
    public void resumeAllJobs() {
        try {
            scheduler.resumeAll();
        } catch (Exception e) {
            log.error("恢复所有任务失败", e);
        }
    }

    @Override
    public void shutdownAllJobs() {
        try {

            if (!scheduler.isShutdown()) {
                // 需谨慎操作关闭scheduler容器
                // scheduler生命周期结束，无法再 start() 启动scheduler
                scheduler.shutdown(true);
            }
        } catch (Exception e) {
            log.error("关闭所有的任务失败", e);
        }
    }
}
```

编程控制层controller层接口

先创建一个请求实体类

```java
@Data
public class QuartzConfigDTO implements Serializable {
    private static final long serialVersionUID = 1L;
    /**
     * 任务名称
     */
    private String jobName;

    /**
     * 任务所属组
     */
    private String groupName;

    /**
     * 任务执行类
     */
    private String jobClass;

    /**
     * 任务调度时间表达式
     */
    private String cronExpression;

    /**
     * 附加参数
     */
    private Map<String, Object> param;
}
```

请求的接口方法

```java
@RestController
@RequestMapping("/test")
public class TestController {
    private static final Logger log = LoggerFactory.getLogger(TestController.class);

    @Autowired
    private QuartzJobService quartzJobService;

    /**
     * 添加新任务
     * @param configDTO
     * @return
     */
    @RequestMapping("/addJob")
    public Object addJob(@RequestBody QuartzConfigDTO configDTO) {
        quartzJobService.addJob(configDTO.getJobClass(), configDTO.getJobName(), configDTO.getGroupName(), configDTO.getCronExpression(), configDTO.getParam());
        return HttpStatus.OK;
    }

    /**
     * 暂停任务
     * @param configDTO
     * @return
     */
    @RequestMapping("/pauseJob")
    public Object pauseJob(@RequestBody QuartzConfigDTO configDTO) {
        quartzJobService.pauseJob(configDTO.getJobName(), configDTO.getGroupName());
        return HttpStatus.OK;
    }

    /**
     * 恢复任务
     * @param configDTO
     * @return
     */
    @RequestMapping("/resumeJob")
    public Object resumeJob(@RequestBody QuartzConfigDTO configDTO) {
        quartzJobService.resumeJob(configDTO.getJobName(), configDTO.getGroupName());
        return HttpStatus.OK;
    }

    /**
     * 立即运行一次定时任务
     * @param configDTO
     * @return
     */
    @RequestMapping("/runOnce")
    public Object runOnce(@RequestBody QuartzConfigDTO configDTO) {
        quartzJobService.runOnce(configDTO.getJobName(), configDTO.getGroupName());
        return HttpStatus.OK;
    }

    /**
     * 更新任务
     * @param configDTO
     * @return
     */
    @RequestMapping("/updateJob")
    public Object updateJob(@RequestBody QuartzConfigDTO configDTO) {
        quartzJobService.updateJob(configDTO.getJobName(), configDTO.getGroupName(), configDTO.getCronExpression(), configDTO.getParam());
        return HttpStatus.OK;
    }

    /**
     * 删除任务
     * @param configDTO
     * @return
     */
    @RequestMapping("/deleteJob")
    public Object deleteJob(@RequestBody QuartzConfigDTO configDTO) {
        quartzJobService.deleteJob(configDTO.getJobName(), configDTO.getGroupName());
        return HttpStatus.OK;
    }

    /**
     * 启动所有任务
     * @return
     */
    @RequestMapping("/startAllJobs")
    public Object startAllJobs() {
        quartzJobService.startAllJobs();
        return HttpStatus.OK;
    }

    /**
     * 暂停所有任务
     * @return
     */
    @RequestMapping("/pauseAllJobs")
    public Object pauseAllJobs() {
        quartzJobService.pauseAllJobs();
        return HttpStatus.OK;
    }

    /**
     * 恢复所有任务
     * @return
     */
    @RequestMapping("/resumeAllJobs")
    public Object resumeAllJobs() {
        quartzJobService.resumeAllJobs();
        return HttpStatus.OK;
    }

    /**
     * 关闭所有任务
     * @return
     */
    @RequestMapping("/shutdownAllJobs")
    public Object shutdownAllJobs() {
        quartzJobService.shutdownAllJobs();
        return HttpStatus.OK;
    }
}
```

7、测试

运行 SpringBoot 的`Application`类，启动服务！

![](\assets\images\2021\juc\springboot-quartz-1.jpg)

请求接口，创建一个每5秒钟执行一次的定时任务

![](\assets\images\2021\juc\springboot-quartz-2.jpg)

可以看到服务正常运行！

![](\assets\images\2021\juc\springboot-quartz-3.jpg)

> 注册监听器

当然，如果你想在 SpringBoot 里面集成 Quartz 的监听器，操作也很简单！分别继承三个监听器，并注册到scheduled调度器

- 创建任务调度监听器

  ```java
  @Component
  public class SimpleSchedulerListener extends SchedulerListenerSupport {
      @Override
      public void jobScheduled(Trigger trigger) {
          System.out.println("任务被部署时被执行");
      }
  
      @Override
      public void jobUnscheduled(TriggerKey triggerKey) {
          System.out.println("任务被卸载时被执行");
      }
  
      @Override
      public void triggerFinalized(Trigger trigger) {
          System.out.println("任务完成了它的使命，光荣退休时被执行");
      }
  
      @Override
      public void triggerPaused(TriggerKey triggerKey) {
          System.out.println(triggerKey + "（一个触发器）被暂停时被执行");
      }
  
      @Override
      public void triggersPaused(String triggerGroup) {
          System.out.println(triggerGroup + "所在组的全部触发器被停止时被执行");
      }
  
      @Override
      public void triggerResumed(TriggerKey triggerKey) {
          System.out.println(triggerKey + "（一个触发器）被恢复时被执行");
      }
  
      @Override
      public void triggersResumed(String triggerGroup) {
          System.out.println(triggerGroup + "所在组的全部触发器被回复时被执行");
      }
  
      @Override
      public void jobAdded(JobDetail jobDetail) {
          System.out.println("一个JobDetail被动态添加进来");
      }
  
      @Override
      public void jobDeleted(JobKey jobKey) {
          System.out.println(jobKey + "被删除时被执行");
      }
  
      @Override
      public void jobPaused(JobKey jobKey) {
          System.out.println(jobKey + "被暂停时被执行");
      }
  
      @Override
      public void jobsPaused(String jobGroup) {
          System.out.println(jobGroup + "(一组任务）被暂停时被执行");
      }
  
      @Override
      public void jobResumed(JobKey jobKey) {
          System.out.println(jobKey + "被恢复时被执行");
      }
  
      @Override
      public void jobsResumed(String jobGroup) {
          System.out.println(jobGroup + "(一组任务）被恢复时被执行");
      }
  
      @Override
      public void schedulerError(String msg, SchedulerException cause) {
          System.out.println("出现异常" + msg + "时被执行");
          cause.printStackTrace();
      }
  
      @Override
      public void schedulerInStandbyMode() {
          System.out.println("scheduler被设为standBy等候模式时被执行");
      }
  
      @Override
      public void schedulerStarted() {
          System.out.println("scheduler启动时被执行");
      }
  
      @Override
      public void schedulerStarting() {
          System.out.println("scheduler正在启动时被执行");
      }
  
      @Override
      public void schedulerShutdown() {
          System.out.println("scheduler关闭时被执行");
      }
  
      @Override
      public void schedulerShuttingdown() {
          System.out.println("scheduler正在关闭时被执行");
      }
  
      @Override
      public void schedulingDataCleared() {
          System.out.println("scheduler中所有数据包括jobs, triggers和calendars都被清空时被执行");
      }
  }
  ```

- 创建任务触发监听器

  ```java
  @Component
  public class SimpleTriggerListener extends TriggerListenerSupport {
      /**
       * Trigger监听器的名称
       * @return
       */
      @Override
      public String getName() {
          return "mySimpleTriggerListener";
      }
  
      /**
       * Trigger被激发 它关联的job即将被运行
       * @param trigger
       * @param context
       */
      @Override
      public void triggerFired(Trigger trigger, JobExecutionContext context) {
          System.out.println("myTriggerListener.triggerFired()");
      }
  
      /**
       * Trigger被激发 它关联的job即将被运行, TriggerListener 给了一个选择去否决 Job 的执行,如果返回TRUE 那么任务job会被终止
       * @param trigger
       * @param context
       * @return
       */
      @Override
      public boolean vetoJobExecution(Trigger trigger, JobExecutionContext context) {
          System.out.println("myTriggerListener.vetoJobExecution()");
          return false;
      }
  
      /**
       * 当Trigger错过被激发时执行,比如当前时间有很多触发器都需要执行，但是线程池中的有效线程都在工作，
       * 那么有的触发器就有可能超时，错过这一轮的触发。
       * @param trigger
       */
      @Override
      public void triggerMisfired(Trigger trigger) {
          System.out.println("myTriggerListener.triggerMisfired()");
      }
  
      /**
       * 任务完成时触发
       * @param trigger
       * @param context
       * @param triggerInstructionCode
       */
      @Override
      public void triggerComplete(Trigger trigger, JobExecutionContext context, Trigger.CompletedExecutionInstruction triggerInstructionCode) {
          System.out.println("myTriggerListener.triggerComplete()");
      }
  }
  ```

- 创建任务执行监听器

  ```java
  @Component
  public class SimpleJobListener extends JobListenerSupport {
      /**
       * job监听器名称
       * @return
       */
      @Override
      public String getName() {
          return "mySimpleJobListener";
      }
  
      /**
       * 任务被调度前
       * @param context
       */
      @Override
      public void jobToBeExecuted(JobExecutionContext context) {
          System.out.println("simpleJobListener监听器，准备执行："+context.getJobDetail().getKey());
      }
  
      /**
       * 任务调度被拒了
       * @param context
       */
      @Override
      public void jobExecutionVetoed(JobExecutionContext context) {
          System.out.println("simpleJobListener监听器，取消执行："+context.getJobDetail().getKey());
      }
  
      /**
       * 任务被调度后
       * @param context
       * @param jobException
       */
      @Override
      public void jobWasExecuted(JobExecutionContext context, JobExecutionException jobException) {
          System.out.println("simpleJobListener监听器，执行结束："+context.getJobDetail().getKey());
      }
  }
  ```

- 监听器注册到`Scheduler`，修改配置类 QuartzConfig.java

  ```java
  @Configuration
  public class QuartzConfig {
  ...
    @Autowired
    private SimpleSchedulerListener simpleSchedulerListener;
  
    @Autowired
    private SimpleJobListener simpleJobListener;
  
    @Autowired
    private SimpleTriggerListener simpleTriggerListener;
  
    @Bean(name = "scheduler")
    public Scheduler scheduler() throws IOException, SchedulerException {
      Scheduler scheduler = schedulerFactoryBean().getScheduler();
      //全局添加监听器
      //添加SchedulerListener监听器
      scheduler.getListenerManager().addSchedulerListener(simpleSchedulerListener);
  
      // 添加JobListener, 支持带条件匹配监听器
      scheduler.getListenerManager().addJobListener(simpleJobListener, KeyMatcher.keyEquals(JobKey.jobKey("myJob", "myGroup")));
  
      // 添加triggerListener，设置全局监听
      scheduler.getListenerManager().addTriggerListener(simpleTriggerListener, EverythingMatcher.allTriggers());
      return scheduler;
    }
  }
  ```

> 采用项目数据源

在上面的 Quartz 数据源配置中，**我们使用了自定义的数据源，目的是和项目中的数据源实现解耦**，当然有的同学不想单独建库，想和项目中数据源保持一致，配置也很简单！

1、在`quartz.properties`配置文件中，去掉`org.quartz.jobStore.dataSource`配置

```properties
#注释掉quartz的数据源配置
#org.quartz.jobStore.dataSource=qzDS
```

2、修改配置类 QuartzConfig.java，加入`dataSource`数据源，并将其注入到`quartz`中

```java
@Autowired
private DataSource dataSource;

@Bean
public SchedulerFactoryBean schedulerFactoryBean() throws IOException {
    //...

    SchedulerFactoryBean factory = new SchedulerFactoryBean();
    factory.setQuartzProperties(propertiesFactoryBean.getObject());
    //使用数据源，自定义数据源
    factory.setDataSource(dataSource);
    
    //...
    return factory;
}
```

> 多任务调度测试

在实际的部署中，项目都是集群进行部署，因此为了和正式环境一致，我们再新建两个相同的项目来测试一下**在集群环境下 quartz 是否可以实现分布式调度，保证任何一个定时任务只有一台机器在运行**

在idea，用不同端口启动同一项目的多个实例quartz-001`、`quartz-002`、`quartz-003，然后新增3个调度任务

<mark>注意</mark>

QUARTZ集群模式调度依赖于各个服务实例的时钟，时钟不同步将会导致调度失败，官方给出的解决办法是将各个不同服务器的时钟进行同步

## 2、其它分布式调度框架

- xxl-job

- elastic-job

### xxl-job

一个开源的分布式任务调度框架

- gitee地址：[https://gitee.com/xuxueli0323/xxl-job](https://gitee.com/xuxueli0323/xxl-job)

- 官方API文档:[https://www.xuxueli.com/xxl-job/](https://www.xuxueli.com/xxl-job/)

下面内容转载自 [https://www.fangzhipeng.com/architecture/2020/06/13/xxljob-test.html](https://www.fangzhipeng.com/architecture/2020/06/13/xxljob-test.html)，主要介绍springboot怎么快速的整合xxl-job，在xxl-job中，有2个角色：

- xxl-job-admin，调度任务管理系统，官方代码已经写好，直接启动即可
- xxl-job-excutor，通常是我们业务系统，比如本案例的springboot业务系统，需要配置xxl-job-admin的地址，主动向xxl-job-admin注册，并建立netty连接，然后admin就可以对excutor进行任务分发。在xxl-job-excutor中需要实现excutor的业务代码。

![](\assets\images\2021\juc\xxljob01.png)

> xxl-job-admin

在官网中下载最新的release代码，比如本文中的v2.2.0版本，下载地址为https://github.com/xuxueli/xxl-job/releases。

提前准备Mysql数据库，导入代码工程中的doc/db目录下的sql文件

![](\assets\images\2021\juc\xxljob-2.jpg)

修改xxl-job-admin工程中的resources中的application.properties的数据库配置，如下：

```properties
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

启动XxlJobAdminApplication的main函数，xxl-job-admin启动成功。在浏览器上访问http://localhost:8081/xxl-job-admin/ ，登陆用户名为admin，密码为123456。登陆成功后，显示的界面如下：

![](\assets\images\2021\juc\xxljob-2.png)

> xxl-job-excutor

新建一个springboot工程，

1、在pom.xml导入依赖

```xml
<dependency>
    <groupId>com.xuxueli</groupId>
    <artifactId>xxl-job-core</artifactId>
    <version>2.2.0</version>
</dependency>
```

2、application.properties中配置xxl.job.admin.addresses的地址，

```properties
# web port
server.port=8082
# no web
#spring.main.web-environment=false

# log config
logging.config=classpath:logback.xml

### xxl-job admin address list, such as "http://address" or "http://address01,http://address02"
xxl.job.admin.addresses=http://127.0.0.1:8081/xxl-job-admin

### xxl-job, access token
xxl.job.accessToken=

### xxl-job executor appname
xxl.job.executor.appname=xxl-job-executor-sample
### xxl-job executor registry-address: default use address to registry , otherwise use ip:port if address is null
xxl.job.executor.address=
### xxl-job executor server-info
xxl.job.executor.ip=
xxl.job.executor.port=9999
### xxl-job executor log-path
xxl.job.executor.logpath=../applogs/xxl-job/jobhandler
### xxl-job executor log-retention-days
xxl.job.executor.logretentiondays=30
```

3、初始化一个XxlJobSpringExecutor，该类用于处理xxl-job-admin和xxl-job-excutor之间的通讯以及任务的处理

```java
@Configuration
public class XxlJobConfig {
    private Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);

    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;
  
    @Value("${xxl.job.accessToken}")
    private String accessToken;

    @Value("${xxl.job.executor.appname}")
    private String appname;

    @Value("${xxl.job.executor.address}")
    private String address;

    @Value("${xxl.job.executor.ip}")
    private String ip;

    @Value("${xxl.job.executor.port}")
    private int port;

    @Value("${xxl.job.executor.logpath}")
    private String logPath;

    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;

    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppname(appname);
        xxlJobSpringExecutor.setAddress(address);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

        return xxlJobSpringExecutor;
    }
}
```

4、注册一个任务，任务名为demoJobHandler。

```java
@Component
public class SampleXxlJob {
    private static Logger logger = LoggerFactory.getLogger(SampleXxlJob.class);

    /**
     * 1、简单任务示例（Bean模式）
     */
    @XxlJob("demoJobHandler")
    public ReturnT<String> demoJobHandler(String param) throws Exception {
        XxlJobLogger.log("XXL-JOB, Hello World.");
        logger.info("XXL-JOB, Hello World. params:"+param);
        for (int i = 0; i < 5; i++) {
            XxlJobLogger.log("beat at:" + i);
            TimeUnit.SECONDS.sleep(2);
        }
        return ReturnT.SUCCESS;
    }
 }
```

启动工程xxl-job-excutor，在xxl-job-admin中可以看到demoJobHandler的配置，在控制台启动任务。

![img](\assets\images\2021\juc\xxljob-3.png)

启动任务后，可以看到执行任务的日志。同时在xxl-job-excutor中可以看到任务执行的业务日志。

![](\assets\images\2021\juc\xxljob-4.png)
