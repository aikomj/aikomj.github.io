---
layout: post
title: 定时任务调度实战2
category: springboot
tags: [springboot]
keywords: springboot
excerpt: springboot + quartz + mysql 实现持久化分布式调度，springboot整合xxl-job分布式调度任务平台，springboot整合elastic-job分布式任务调度平台
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

2、其它分布式调度框架

- xxl-job

- elastic-job

## 2、xxl-job分布式任务调度

一个开源的分布式任务调度框架

- gitee地址：[https://gitee.com/xuxueli0323/xxl-job](https://gitee.com/xuxueli0323/xxl-job)

- 官方API文档:[https://www.xuxueli.com/xxl-job/](https://www.xuxueli.com/xxl-job/)

下面内容转载自 [https://www.fangzhipeng.com/architecture/2020/06/13/xxljob-test.html](https://www.fangzhipeng.com/architecture/2020/06/13/xxljob-test.html)，主要介绍springboot怎么快速的整合xxl-job，在xxl-job中，有2个角色：

- xxl-job-admin，调度任务管理系统，官方代码已经写好，直接启动即可
- xxl-job-excutor，通常是我们业务系统，比如本案例的springboot业务系统，需要配置xxl-job-admin的地址，主动向xxl-job-admin注册，并建立netty连接，然后admin就可以对excutor进行任务分发。在xxl-job-excutor中需要实现excutor的业务代码。

![](\assets\images\2021\juc\xxljob01.png)

### xxl-job-admin

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

### xxl-job-excutor

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

# 调度中心部署跟地址 [选填]：如调度中心集群部署存在多个地址则用逗号分隔。执行器将会使用该地址进行"执行器心跳注册"和"任务结果回调"；为空则关闭自动注册；
xxl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin
# 执行器通讯TOKEN [选填]：非空时启用；
xxl.job.accessToken=
# 执行器AppName [选填]：执行器心跳注册分组依据；为空则关闭自动注册
xxl.job.executor.appname=xxl-job-executor-sample
# 执行器注册 [选填]：优先使用该配置作为注册地址，为空时使用内嵌服务 ”IP:PORT“ 作为注册地址。从而更灵活的支持容器类型执行器动态IP和动态映射端口问题。
xxl.job.executor.address=
# 执行器IP [选填]：默认为空表示自动获取IP，多网卡时可手动设置指定IP，该IP不会绑定Host仅作为通讯实用；地址信息用于 "执行器注册" 和 "调度中心请求并触发任务"；
xxl.job.executor.ip=
# 执行器端口号 [选填]：小于等于0则自动获取；默认端口为9999，单机部署多个执行器时，注意要配置不同执行器端口；
xxl.job.executor.port=9999
### 执行器运行日志文件存储磁盘路径 [选填] ：需要对该路径拥有读写权限；为空则使用默认路径；
xxl.job.executor.logpath=/data/applogs/xxl-job/jobhandler
### 执行器日志文件保存天数 [选填] ： 过期日志自动清理, 限制值大于等于3时生效; 否则, 如-1, 关闭自动清理功能；
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

### 配置属性说明

打开xxl-job的官方文档 : [https://www.xuxueli.com/xxl-job/](https://www.xuxueli.com/xxl-job/)

找到配置属性详细说明，在我们配置任务的时候，需要选择路由策略，调度过期策略，阻塞处理策略

![](\assets\images\2021\springcloud\xxl-job-jobhandler.png)

```sh
高级配置：
    - 路由策略：当执行器集群部署时，提供丰富的路由策略，包括；
        FIRST（第一个）：固定选择第一个机器；
        LAST（最后一个）：固定选择最后一个机器；
        ROUND（轮询）：；
        RANDOM（随机）：随机选择在线的机器；
        CONSISTENT_HASH（一致性HASH）：每个任务按照Hash算法固定选择某一台机器，且所有任务均匀散列在不同机器上。
        LEAST_FREQUENTLY_USED（最不经常使用）：使用频率最低的机器优先被选举；
        LEAST_RECENTLY_USED（最近最久未使用）：最久未使用的机器优先被选举；
        FAILOVER（故障转移）：按照顺序依次进行心跳检测，第一个心跳检测成功的机器选定为目标执行器并发起调度；
        BUSYOVER（忙碌转移）：按照顺序依次进行空闲检测，第一个空闲检测成功的机器选定为目标执行器并发起调度；
        SHARDING_BROADCAST(分片广播)：广播触发对应集群中所有机器执行一次任务，同时系统自动传递分片参数；可根据分片参数开发分片任务；
    - 子任务：每个任务都拥有一个唯一的任务ID(任务ID可以从任务列表获取)，当本任务执行结束并且执行成功时，将会触发子任务ID所对应的任务的一次主动调度。
    - 调度过期策略：
        - 忽略：调度过期后，忽略过期的任务，从当前时间开始重新计算下次触发时间；
        - 立即执行一次：调度过期后，立即执行一次，并从当前时间开始重新计算下次触发时间；
    - 阻塞处理策略：调度过于密集执行器来不及处理时的处理策略；
        单机串行（默认）：调度请求进入单机执行器后，调度请求进入FIFO队列并以串行方式运行；
        丢弃后续调度：调度请求进入单机执行器后，发现执行器存在运行的调度任务，本次请求将会被丢弃并标记为失败；
        覆盖之前调度：调度请求进入单机执行器后，发现执行器存在运行的调度任务，将会终止运行中的调度任务并清空队列，然后运行本地调度任务；
    - 任务超时时间：支持自定义任务超时时间，任务运行超时将会主动中断任务；
    - 失败重试次数；支持自定义任务失败重试次数，当任务失败时将会按照预设的失败重试次数主动进行重试；
```



### 任务分片实战

场景：有k个地方，每个地市有x个订单需要执行，总共有kx个订单，下面初始化订单任务

```java
@Component
@Slf4j
public class AhOrdersXxlJob {
  //城市编号
  private static final List<Integer> CITY_ID_LIST = Arrays.asList(550, 551, 552, 553, 554, 555, 556, 557, 558, 559, 561, 562, 563, 564, 566);
  //每个城市的任务数
  private static final int PER_LATN_TASK_NUM = 30; 
  // 任务数据库
  private static final Map<Integer, List<String>> singleMachineMultiTasks; 

  static {
    singleMachineMultiTasks = new HashMap<>();
    CITY_ID_LIST.forEach(city -> {
      List<String> tasks = new ArrayList<>(PER_LATN_TASK_NUM);
      IntStream.rangeClosed(1, PER_LATN_TASK_NUM).forEach(index -> {
        String orderInfo = city + "------NO." + index ;
        tasks.add(orderInfo);
      });

      singleMachineMultiTasks.put(city, tasks);
    });
  }
}
```

> 单实例多任务分片

在xxl-job-admin配置两个任务

![](\assets\images\2021\springcloud\xxl-job-1.png)

每个任务指定不同的参数

![](\assets\images\2021\springcloud\xxl-job-2.png)

任务处理实现方法

```java
@XxlJob(value = "singleMachineMultiTasks", init = "init", destroy = "destroy")
public ReturnT<String> singleMachineMultiTasks(String cities) throws Exception {
  if (StringUtils.isEmpty(cities)) {
    return new ReturnT(FAIL_CODE, "latnIds不能为空");
  }

  Arrays.stream(cities.split(",")).map(String::trim).filter(StringUtils::isNotBlank).map(Integer::parseInt).forEach(latnId -> {
    List<String> tasks = singleMachineMultiTasks.get(latnId);
    Optional.ofNullable(tasks).ifPresent(todoTasks -> {
      todoTasks.forEach(task -> {
        XxlJobLogger.log("【{}】执行【{}】，任务内容为：{}", Thread.currentThread().getName(), latnId, task);
      });
    });
  });
  return ReturnT.SUCCESS;
}

 public void init() {
     log.info("init");
 }

 public void destroy() {
     log.info("destory");
 }
```

在xxl-job-admin上启动配置的两个任务，在xxl-job-excutor 执行实例上是分两个线程执行任务的

![](\assets\images\2021\springcloud\xxl-job-3.png)

![](\assets\images\2021\springcloud\xxl-job-4.png)

> 多实例单任务分片

启动多个excutor执行器实例

![](\assets\images\2021\springcloud\xxl-job-5.png)

在xxl-job-admin上可以看到多个excutor

![](\assets\images\2021\springcloud\xxl-job-6.png)

xxl-job-admin上配置任务

![](\assets\images\2021\springcloud\xxl-job-7.png)

这里要使用<mark>取模</mark>的方式进行数据分片，理解一下下面的方案，xxl-job封装了工具类`ShardingUtil`

```java
// 获取分片
ShardingUtil.ShardingVO shardingVO = ShardingUtil.getShardingVo();
//执行器数量
int number = shardingVO.getTotal();
//当前分片
int index = shardingVO.getIndex();

// sql每次从表中取100条数据：表id是自增的，用id对执行器数量（分片总数）取模，那么每个分片获取的数据就不会重复，避免重复消费数据。
SELECT id,name,password
FROM t_push
WHERE `status` = 0
AND mod(id,#{number}) = #{index}  //number 分片总数，index当前分片数 mod函数取余
order by id desc
LIMIT 100;
```

任务处理实现方法

```java
@XxlJob(value = "multiMachineMultiTasks", init = "init", destroy = "destroy")
public ReturnT<String> multiMachineMultiTasks(String params) throws Exception {
  // xxl-job封装的工具类
  ShardingUtil.ShardingVO shardingVO = ShardingUtil.getShardingVo();
  int n = shardingVO.getTotal(); // n 个实例  
  int i = shardingVO.getIndex(); // 当前为第i个

  IntStream.range(0, CITY_ID_LIST.size()).forEach(cityIndex -> {
    if (cityIndex % n == i) { // 取模后等于当前分片索引的，则处理
      int city = CITY_ID_LIST.get(cityIndex);
      List<String> tasks = singleMachineMultiTasks.get(city);
      Optional.ofNullable(tasks).ifPresent(todoTasks -> {
        todoTasks.forEach(task -> {
          XxlJobLogger.log("实例【{}】执行【{}】，任务内容为：{}", i, city, task);
        });

      });
    }
  });
  return ReturnT.SUCCESS;
}
```

ShardingUtil的源码

![](\assets\images\2021\springcloud\xxl-job-shardingutil.png)

现在我们分别启动3个执行器和xxl-job-admin上配置的任务，看每个执行器上的输出

![](\assets\images\2021\springcloud\xxl-job-sharding-1.png)

![](\assets\images\2021\springcloud\xxl-job-sharding-2.png)

![](\assets\images\2021\springcloud\xxl-job-sharding-3.png)

## 3、Elastic Job分布式任务调度

### 诞生

听名字跟ElasticSearch 分布式搜索引擎有关吗，Elastic Job分布式任务调度。Quartz 也可以通过集群方式来保证服务高可用，但是它也有一个的弊端，**那就是服务节点数量的增加，并不能提升任务的执行效率，即不能实现水平扩展**！

因为 Quartz 在分布式集群环境下是通过数据库锁方式来实现有且只有一个有效的服务节点来运行服务，从而保证服务在集群环境下定时任务不会被重复调用。

如果需要运行的定时任务很少的话，使用 Quartz 不会有太大的问题，但是如果现在有这么一个需求，例如理财产品，每天6点系统需要计算每个账户昨天的收益，假如这个理财产品，有几个亿的用户，如果都在一个服务实例上跑，可能第二天都无法处理完这项任务！

类似这样场景还有很多很多，**很显然 Quartz 很难满足我们这种大批量、任务执行周期长的任务调度**！当当网基于 Quartz 开发了一套适合在分布式环境下能高效率的使用服务器资源的 Elastic-Job 定时任务框架，在2015年开源，！没想到是当当网。

> 亮点：支持弹性扩容，任务分片

怎么实现的？思想上跟elsaticsearch的分片相似，在Java的角度，又想到了forkjoin将任务分支处理再合并结果。

比如现在有个任务要执行，如果将任务进行分片成10个，那么可以同时在10个服务实例上并行执行，互相不影响，从而大大的提升了任务执行效率，并且充分的利用服务器资源！对于上面的理财产品，如果这个任务需要处理1个亿用户，那么我们可以通过水平扩展，比如对任务进行分片为500，让500个服务实例同时运行，每个服务实例处理20万条数据，不出意外的话，1 - 2个小时可以全部跑完，如果时间还是很长，还可以继续水平扩张，添加服务实例来运行

### 项目架构介绍

Elastic-Job 最开始只有一个 elastic-job-core 的项目，定位轻量级、无中心化，最核心的服务就是<mark>支持弹性扩容和数据分片</mark>！从 2.X 版本以后，主要分为 <mark>Elastic-Job-Lite 和 Elastic-Job-Cloud</mark> 两个子项目。

-  Elastic-Job-Lite 定位为轻量级 无 中 心 化 解 决 方 案 ， 使 用`jar` 包 的 形 式 提 供 分 布 式 任 务 的 协 调 服 务 。
- Elastic-Job-Cloud 使用 Mesos + Docker 的解决方案，额外提供资源治理、应用分发以及进程隔离等服务（跟 Lite 的区别只是部署方式不同，他们使用相同的 API，只要开发一次）

`Elastic-Job-Lite`，最主要的功能特性如下：

- 分布式调度协调：采用 zookeeper 实现注册中心，进行统一调度。
- 支持任务分片：将需要执行的任务进行分片，实现并行调度。
- 支持弹性扩容缩容：将任务拆分为 n 个任务项后，各个服务器分别执行各自分配到的任务项。一旦有新的服务器加入集群，或现有服务器下线，elastic-job 将在保留本次任务执行不变的情况下，下次任务开始前触发任务重分片。

还**有失效转移、错过执行作业重触发**等等功能，大家可以访问官网文档了解更多[http://shardingsphere.apache.org/elasticjob/index_zh.html](http://shardingsphere.apache.org/elasticjob/index_zh.html)

![](\assets\images\2021\springcloud\elastic-job.png)

目前稳定版本是3.X,在企业中也得到广泛使用，下面是它的架构图

![elastic-job-arch](\assets\images\2021\springcloud\elastic-job-arch.png)

### 应用实践

> 1、安装zookeeper

`elastic-job-lite`，是直接依赖 zookeeper 的，因此在开发之前我们需要先准备好对应的 zookeeper 环境，关于 zookeeper 的安装过程，就不多说了，非常简单，网上都有教程！之前做kafka集群和Mycat集群时都有搭建zookeeper集群。

[http://139.199.13.139/blog/icoding-edu/2020/06/27/icoding-note-052.html](http://139.199.13.139/blog/icoding-edu/2020/06/27/icoding-note-052.html)

> 2、安装elastic-job-lite-console

elastic-job-lite-console就是控制台，一个任务作业可视化界面管理系统。可以单独部署，与平台不关，通过配置注册中心和数据源来抓取数据可视化。

这点跟阿里的流量卫兵控制台相似，通过java -jar启动的，参考文章 [http://139.199.13.139/blog/icoding-edu/2020/07/04/icoding-note-055.html](http://139.199.13.139/blog/icoding-edu/2020/07/04/icoding-note-055.html)

Github：[https://github.com/apache/shardingsphere-elasticjob](https://github.com/apache/shardingsphere-elasticjob)

Gitee: [https://gitee.com/elasticjob/elastic-job](https://gitee.com/elasticjob/elastic-job)

登录github查看版本 [https://github.com/apache/shardingsphere-elasticjob/releases](https://github.com/apache/shardingsphere-elasticjob/releases)

选择2.1.5版本下载

![](\assets\images\2021\springcloud\elastic-job-lite-console-215.png)

在idea执行`mvn clean install`进行打包，通过java -jar的方式启动服务后，在浏览器访问`http://127.0.0.1:8899`，输入账户、密码（都是`root`）即可进入控制台页面

![](\assets\images\2021\springcloud\elastic-job-lite-console-browser.png)

进入之后，将上文所在的 zookeeper 注册中心进行配置，包括数据库 mysql 的数据源也配置一下！

> 3、创建springboot工程

导入依赖

```xml
<!-- 引入elastic-job-lite核心模块 -->
<dependency>
    <groupId>com.dangdang</groupId>
    <artifactId>elastic-job-lite-core</artifactId>
    <version>2.1.5</version>
</dependency>

<!-- 使用springframework自定义命名空间时引入 -->
<dependency>
    <groupId>com.dangdang</groupId>
    <artifactId>elastic-job-lite-spring</artifactId>
    <version>2.1.5</version>
</dependency>
```

配置文件`application.properties`中提前配置好 zookeeper 注册中心

```properties
zookeeper.serverList=127.0.0.1:2181
zookeeper.namespace=example-elastic-job-test
```

新建 ZookeeperConfig 配置类

```java
@Configuration
@ConditionalOnExpression("'${zookeeper.serverList}'.length() > 0")
public class ZookeeperConfig {
    /**
     * zookeeper 配置了注册中心地址这个组件才生效
     * @return
     */
    @Bean(initMethod = "init")
    public ZookeeperRegistryCenter zookeeperRegistryCenter(@Value("${zookeeper.serverList}") String serverList, 
                                                           @Value("${zookeeper.namespace}") String namespace){
        return new ZookeeperRegistryCenter(new ZookeeperConfiguration(serverList,namespace));
    }
}
```

elastic-job支持三种类型的作业任务处理

- <mark>Simple 类型作业</mark>：Simple 类型用于一般任务的处理，只需实现`SimpleJob`接口。该接口仅提供单一方法用于覆盖，此方法将定时执行，与Quartz原生接口相似。
- <mark>Dataflow 类型作业</mark>：Dataflow 类型用于处理数据流，需实现`DataflowJob`接口。该接口提供2个方法可供覆盖，分别用于抓取(`fetchData`)和处理(`processData`)数据。
- <mark>Script 类型作业</mark>：Script 类型作业意为脚本类型作业，支持 shell，python，perl等所有类型脚本。只需通过控制台或代码配置 scriptCommandLine 即可，无需编码。执行脚本路径可包含参数，参数传递完毕后，作业框架会自动追加最后一个参数为作业运行时信息。

### 新建Simple类型作业

1、创建任务实现类`MySimpleJob`继承接口`SimpleJob`，当前工作主要是打印一条日志

```java
@Slf4j
public class MySimpleJob implements SimpleJob {
    @Override
    public void execute(ShardingContext shardingContext) {
        log.info(String.format("Thread ID: %s, 作业分片总数: %s, " +
                        "当前分片项: %s.当前参数: %s," +
                        "作业名称: %s.作业自定义参数: %s"
                ,
                Thread.currentThread().getId(),
                shardingContext.getShardingTotalCount(),
                shardingContext.getShardingItem(),
                shardingContext.getShardingParameter(),
                shardingContext.getJobName(),
                shardingContext.getJobParameter()
        ));
    }
}
```

2、创建任务监听器`MyElasticJobListener`实现接口`ElasticJobListener`，用于监听`MySimpleJob`的任务执行情况。

```java
@Slf4j
public class MyElasticJobListener implements ElasticJobListener {
    private long beginTime = 0;

    @Override
    public void beforeJobExecuted(ShardingContexts shardingContexts) {
        beginTime = System.currentTimeMillis();
        log.info("===>{} MyElasticJobListener BEGIN TIME: {} <===",shardingContexts.getJobName(),  DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
    }

    @Override
    public void afterJobExecuted(ShardingContexts shardingContexts) {
        long endTime = System.currentTimeMillis();
        log.info("===>{} MyElasticJobListener END TIME: {},TOTAL CAST: {} <===",shardingContexts.getJobName(), DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"), endTime - beginTime);
    }
}
```

3、创建配置类`MySimpleJobConfig`，将`MySimpleJob`注入到zookeeper

```java
@Configuration
public class MySimpleJobConfig {
    /**
     * 任务名称
     */
    @Value("${simpleJob.mySimpleJob.name}")
    private String mySimpleJobName;

    /**
     * cron表达式
     */
    @Value("${simpleJob.mySimpleJob.cron}")
    private String mySimpleJobCron;

    /**
     * 作业分片总数
     */
    @Value("${simpleJob.mySimpleJob.shardingTotalCount}")
    private int mySimpleJobShardingTotalCount;

    /**
     * 作业分片参数
     */
    @Value("${simpleJob.mySimpleJob.shardingItemParameters}")
    private String mySimpleJobShardingItemParameters;

    /**
     * 自定义参数
     */
    @Value("${simpleJob.mySimpleJob.jobParameters}")
    private String mySimpleJobParameters;

    @Autowired
    private ZookeeperRegistryCenter registryCenter;

    @Bean
    public MySimpleJob mySimpleJob() {
        return new MySimpleJob();
    }

    @Bean(initMethod = "init")
    public JobScheduler simpleJobScheduler(final MySimpleJob mySimpleJob) {
  //配置任务监听器
   MyElasticJobListener elasticJobListener = new MyElasticJobListener();
        return new SpringJobScheduler(mySimpleJob, registryCenter, getLiteJobConfiguration(), elasticJobListener);
    }

    private LiteJobConfiguration getLiteJobConfiguration() {
        // 定义作业核心配置
        JobCoreConfiguration simpleCoreConfig = JobCoreConfiguration.newBuilder(mySimpleJobName, mySimpleJobCron, mySimpleJobShardingTotalCount).
                shardingItemParameters(mySimpleJobShardingItemParameters).jobParameter(mySimpleJobParameters).build();
        // 定义SIMPLE类型配置
        SimpleJobConfiguration simpleJobConfig = new SimpleJobConfiguration(simpleCoreConfig, MySimpleJob.class.getCanonicalName());
        // 定义Lite作业根配置
        LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).overwrite(true).build();
        return simpleJobRootConfig;
    }
}
```

配置文件application.properties中配置好对应的mySimpleJob参数

```properties
#simpleJob类型的job
simpleJob.mySimpleJob.name=mySimpleJob
simpleJob.mySimpleJob.cron=0/15 * * * * ?
simpleJob.mySimpleJob.shardingTotalCount=3
simpleJob.mySimpleJob.shardingItemParameters=0=a,1=b,2=c
simpleJob.mySimpleJob.jobParameters=helloWorld
```

4、启动项目，发现任务执行了3次

![](\assets\images\2021\springcloud\elastic-job-simple-job.jpg)

登录一下console 控制台，查看任务配置

![](\assets\images\2021\springcloud\elastic-job-simple-job-2.jpg)

因为配置的分片数为3，这个时候会有3个线程进行同时执行任务，因为只有一个服务实例，这个任务被执行来3次，下面修改一下端口配置，创建三个相同的服务实例，看看效果如下：

![](\assets\images\2021\springcloud\elastic-job-simple-job-3.jpg)

很清晰的看到任务被执行一次！

### 新建DataFlowJob类型作业

1、创建任务实现类`MyDataFlowJob`实现接口`DataflowJob`

```java
@Slf4j
public class MyDataFlowJob implements DataflowJob<String> {
    private boolean flag = false;

    @Override
    public List<String> fetchData(ShardingContext shardingContext) {
        log.info("开始获取数据");
        if (flag) {
            return null;
        }
        return Arrays.asList("qingshan", "jack", "seven");
    }

    @Override
    public void processData(ShardingContext shardingContext, List<String> data) {
        for (String val : data) {
            // 处理完数据要移除掉，不然就会一直跑,处理可以在上面的方法里执行。这里采用 flag
            log.info("开始处理数据：" + val);
        }
        flag = true;
    }
}
```

2、创建配置类`MyDataFlowJobConfig `，将`MyDataFlowJob`注入到zookeeper

```java
@Configuration
public class MyDataFlowJobConfig {
    /**
     * 任务名称
     */
    @Value("${dataflowJob.myDataflowJob.name}")
    private String jobName;

    /**
     * cron表达式
     */
    @Value("${dataflowJob.myDataflowJob.cron}")
    private String jobCron;

    /**
     * 作业分片总数
     */
    @Value("${dataflowJob.myDataflowJob.shardingTotalCount}")
    private int jobShardingTotalCount;

    /**
     * 作业分片参数
     */
    @Value("${dataflowJob.myDataflowJob.shardingItemParameters}")
    private String jobShardingItemParameters;

    /**
     * 自定义参数
     */
    @Value("${dataflowJob.myDataflowJob.jobParameters}")
    private String jobParameters;

    @Autowired
    private ZookeeperRegistryCenter registryCenter;

    @Bean
    public MyDataFlowJob myDataFlowJob() {
        return new MyDataFlowJob();
    }

    @Bean(initMethod = "init")
    public JobScheduler dataFlowJobScheduler(final MyDataFlowJob myDataFlowJob) {
        MyElasticJobListener elasticJobListener = new MyElasticJobListener();
        return new SpringJobScheduler(myDataFlowJob, registryCenter, getLiteJobConfiguration(), elasticJobListener);
    }

    private LiteJobConfiguration getLiteJobConfiguration() {
        // 定义作业核心配置
        JobCoreConfiguration dataflowCoreConfig = JobCoreConfiguration.newBuilder(jobName, jobCron, jobShardingTotalCount).
                shardingItemParameters(jobShardingItemParameters).jobParameter(jobParameters).build();
        // 定义DATAFLOW类型配置
        DataflowJobConfiguration dataflowJobConfig = new DataflowJobConfiguration(dataflowCoreConfig, MyDataFlowJob.class.getCanonicalName(), false);
        // 定义Lite作业根配置
        LiteJobConfiguration dataflowJobRootConfig = LiteJobConfiguration.newBuilder(dataflowJobConfig).overwrite(true).build();
        return dataflowJobRootConfig;
    }
}
```

配置文件application.properties中配置好对应的myDataflowJob参数

```properties
#dataflow类型的job
dataflowJob.myDataflowJob.name=myDataflowJob
dataflowJob.myDataflowJob.cron=0/15 * * * * ?
dataflowJob.myDataflowJob.shardingTotalCount=1
dataflowJob.myDataflowJob.shardingItemParameters=0=a,1=b,2=c
dataflowJob.myDataflowJob.jobParameters=myDataflowJobParamter
```

3、启动项目，看看运行效果

![](\assets\images\2021\springcloud\elastic-job-dataflow-job.png)

注意：

- 流式处理类型任务，它会不停的**拉取数据、处理数据**，在拉取的时候，如果返回为空，就不会处理数据！

- 非流式处理类型任务，和上面介绍的`simpleJob`类型处理一样！

### 新建ScriptJob类型作业

ScriptJob类型有些不同，主要是用于定时执行某个脚本，一般用的比较少，因为目标是脚本，没有执行的任务，所以无需编写任务作业类型！

1、创建配置类`MyScriptJobConfig `

```java
@Configuration
public class MyScriptJobConfig {

    /**
     * 任务名称
     */
    @Value("${scriptJob.myScriptJob.name}")
    private String jobName;

    /**
     * cron表达式
     */
    @Value("${scriptJob.myScriptJob.cron}")
    private String jobCron;

    /**
     * 作业分片总数
     */
    @Value("${scriptJob.myScriptJob.shardingTotalCount}")
    private int jobShardingTotalCount;

    /**
     * 作业分片参数
     */
    @Value("${scriptJob.myScriptJob.shardingItemParameters}")
    private String jobShardingItemParameters;

    /**
     * 自定义参数
     */
    @Value("${scriptJob.myScriptJob.jobParameters}")
    private String jobParameters;

    @Autowired
    private ZookeeperRegistryCenter registryCenter;


    @Bean(initMethod = "init")
    public JobScheduler scriptJobScheduler() {
        MyElasticJobListener elasticJobListener = new MyElasticJobListener();
        return new JobScheduler(registryCenter, getLiteJobConfiguration(), elasticJobListener);
    }

    private LiteJobConfiguration getLiteJobConfiguration() {
        // 定义作业核心配置
        JobCoreConfiguration scriptCoreConfig = JobCoreConfiguration.newBuilder(jobName, jobCron, jobShardingTotalCount).
                shardingItemParameters(jobShardingItemParameters).jobParameter(jobParameters).build();
        // 定义SCRIPT类型配置
        ScriptJobConfiguration scriptJobConfig = new ScriptJobConfiguration(scriptCoreConfig, "echo 'Hello World !'");
        // 定义Lite作业根配置
        LiteJobConfiguration scriptJobRootConfig = LiteJobConfiguration.newBuilder(scriptJobConfig).overwrite(true).build();
        return scriptJobRootConfig;
    }
}
```

配置文件`application.properties`中配置好对应的`myScriptJob`参数

```properties
#script类型的job
scriptJob.myScriptJob.name=myScriptJob
scriptJob.myScriptJob.cron=0/15 * * * * ?
scriptJob.myScriptJob.shardingTotalCount=3
scriptJob.myScriptJob.shardingItemParameters=0=a,1=b,2=c
scriptJob.myScriptJob.jobParameters=myScriptJobParamter
```

2、启动项目，看看运行效果

![](\assets\images\2021\springcloud\elastic-job-script-job.png)

因为配置的分片数为3，这个时候会有3个线程进行同时执行任务，因为只有一个服务实例，这个任务被执行来3次

### 将任务状态持久化到数据库

`elastic-job`是如何存储数据的，用`ZooInspector`客户端链接`zookeeper`注册中心，你发现对应的任务配置被存储到相应的树根上！

![](\assets\images\2021\springcloud\elastic-job-zookeeper-config.png)

这个跟Mycat 做集群化一样，将配置信息存储到zookeeper上，同步每个Mycat节点的配置信息。

但是具体作业任务执行轨迹和状态结果是不会存储到`zookeeper`，需要我们在项目中通过数据源方式进行持久化！

将任务状态持久化到数据库配置过程也很简单，只需要在对应的配置类上注入数据源即可，以`MySimpleJobConfig`为例，代码如下：

1、配置类`MySimpleJobConfig`添加事件数据源配置

```java
@Configuration
public class MySimpleJobConfig {
    /**
     * 任务名称
     */
    @Value("${simpleJob.mySimpleJob.name}")
    private String mySimpleJobName;

    /**
     * cron表达式
     */
    @Value("${simpleJob.mySimpleJob.cron}")
    private String mySimpleJobCron;

    /**
     * 作业分片总数
     */
    @Value("${simpleJob.mySimpleJob.shardingTotalCount}")
    private int mySimpleJobShardingTotalCount;

    /**
     * 作业分片参数
     */
    @Value("${simpleJob.mySimpleJob.shardingItemParameters}")
    private String mySimpleJobShardingItemParameters;

    /**
     * 自定义参数
     */
    @Value("${simpleJob.mySimpleJob.jobParameters}")
    private String mySimpleJobParameters;

    @Autowired
    private ZookeeperRegistryCenter registryCenter;

    @Autowired
    private DataSource dataSource;;

    @Bean
    public MySimpleJob stockJob() {
        return new MySimpleJob();
    }

    @Bean(initMethod = "init")
    public JobScheduler simpleJobScheduler(final MySimpleJob mySimpleJob) {
        //添加事件数据源配置
        JobEventConfiguration jobEventConfig = new JobEventRdbConfiguration(dataSource);
        MyElasticJobListener elasticJobListener = new MyElasticJobListener();
        return new SpringJobScheduler(mySimpleJob, registryCenter, getLiteJobConfiguration(), jobEventConfig, elasticJobListener);
    }

    private LiteJobConfiguration getLiteJobConfiguration() {
        // 定义作业核心配置
        JobCoreConfiguration simpleCoreConfig = JobCoreConfiguration.newBuilder(mySimpleJobName, mySimpleJobCron, mySimpleJobShardingTotalCount).
                shardingItemParameters(mySimpleJobShardingItemParameters).jobParameter(mySimpleJobParameters).build();
        // 定义SIMPLE类型配置
        SimpleJobConfiguration simpleJobConfig = new SimpleJobConfiguration(simpleCoreConfig, MySimpleJob.class.getCanonicalName());
        // 定义Lite作业根配置
        LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).overwrite(true).build();
        return simpleJobRootConfig;
    }
}
```

在配置文件`application.properties`中配置好对应的`datasource`参数

```properties
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/example-elastic-job-test
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

2、启动项目，在`elastic-job-lite-console`控制台配置对应的数据源

![](\assets\images\2021\springcloud\elastic-job-mysql.jpg)

点击【作业轨迹】即可查看对应作业执行情况

![](\assets\images\2021\springcloud\elastic-job-log-1.jpg)

![](\assets\images\2021\springcloud\elastic-job-log-2.jpg)

### 总结

在分布式环境环境下，`elastic-job-lite`支持的弹性扩容、任务分片是最大的亮点，在实际使用的时候，任务分片总数尽可能大于服务实例个数，并且是倍数关系，这样任务在分片的时候，会更加均匀！如果想深入的了解`elasticjob`，大家可以访问官方文档，获取更加详细的使用教程！