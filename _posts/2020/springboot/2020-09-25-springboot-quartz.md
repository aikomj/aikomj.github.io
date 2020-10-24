---
layout: post
title: Spring boot整合Quartz实现Java定时任务的动态配置
category: springboot
tags: [springboot]
keywords: springboot
excerpt: Java定时任务使用过的解决方案
lock: noneed
---

## 1、定时注解

Spring实现定时任务，首先说它支持的定时任务注解@Scheduled，支持cron表达式，代码内嵌简单的定时任务，例子：

```java
// 定时任务类
@Service
public class ScheduledService {

	// 每分钟执行一次
	@Scheduled(cron = "0 * * * * ?")
	public void hello(){
		System.out.println("Hello ......");
	}
} 
```

启动类开启定时任务

```java
@SpringBootApplication
@EnableAsync // 开启异步注解的支持
@EnableScheduling // 开启定时任务的支持
public class SpringBootDataStudyXjwApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootDataStudyXjwApplication.class, args);
	}
}
```

## 2、定时框架quartz

### springboot 整合

第二个用过的就是quartz，主要有三个核心概念

- scheduler 调度器
- trigger 触发器
- job 任务

之前使用renren_fast框架中的定时器也是使用quartz框架，有界面可以动态启动、停止任务，任务被执行时用到了反射的概念，下面是它的核心代码



1、导入依赖

```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.2.1</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
</dependency>
```

2、配置类QuartzConfigration

```java
@Configuration
public class QuartzConfigration {
    @Autowired
    private JobFactory jobFactory;

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean() {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        try {
            schedulerFactoryBean.setOverwriteExistingJobs(true);
            schedulerFactoryBean.setQuartzProperties(quartzProperties());
            schedulerFactoryBean.setJobFactory(jobFactory);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return schedulerFactoryBean;
    }

    // 指定quartz.properties，可在配置文件中配置相关属性
    @Bean
    public Properties quartzProperties() throws IOException {
        PropertiesFactoryBean propertiesFactoryBean = new PropertiesFactoryBean();
        propertiesFactoryBean.setLocation(new ClassPathResource("/config/quartz.properties"));
        propertiesFactoryBean.afterPropertiesSet();
        return propertiesFactoryBean.getObject();
    }

    // 创建schedule
    @Bean(name = "scheduler")
    public Scheduler scheduler() {
        return schedulerFactoryBean().getScheduler();
    }
}
```





### 集群化

