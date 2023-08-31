---
layout: post
title: 基于模板设计模式对数据迁移封装
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 只需实现不同业务的具体读写逻辑代码，数据迁移任务通过分页切分数据发送MQ消息，消费者接受MQ消息迁移指定页的数据
lock: noneed
---

## 1、数据迁移任务封装

### 请求控制层DataTransferController

通过Reqeust请求，启动一次数据迁移任务

```java
@RequiredArgsConstructor(onConstructor_={@Autowired})
@RestController
@RequestMapping("/v1/dataTransfer")
public class DataTransferController {

    private final DataTransferJobManager dataTransferJobManager;

    //用户名
    @Value("${business.data-transfer.user:aa64865487710aba549516e5c81fa2a1}")
    private String startUser;
    //用户密码
    @Value("${business.data-transfer.pwd:518be6ab336bccd6cc7e93c227fd1ac6}")
    private String startPwd;

    /**
     * 启动作业
     * @param jobName 作业名称
     * @param user 用户
     * @param pwd 密码
     * @return R
     */
    @ApiOperation(value = "启动作业", notes = "启动作业")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "user",value="帐号",required = true),
            @ApiImplicitParam(name = "pwd", value = "密码", required = true),
    })
    @GetMapping("/start")
    public R<String> start(@RequestParam("jobName") String jobName,@RequestParam("user")String user,@RequestParam("pwd")String pwd){
        if(!startUser.equals(user) || !startPwd.equals(pwd)){
            throw new SystemRuntimeException("启动作业认证用户或密码不正确");
        }

        LoginAccountVO loginUser = HttpRequestHandler.getRequestUser(true);
        dataTransferJobManager.start(jobName,loginUser.getLoginName());
        return R.buildSuccess("启动作业成功");
    }

    /**
     * 重启作业
     * @param jobId 作业Id
     * @param user 用户
     * @param pwd 密码
     * @return R
     */
    @ApiOperation(value = "重启作业", notes = "重启作业")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "user",value="帐号",required = true),
            @ApiImplicitParam(name = "pwd", value = "密码", required = true),
    })
    @GetMapping("/restart")
    public R<String> restart(@RequestParam("jobId") String jobId,@RequestParam("user")String user,@RequestParam("pwd")String pwd){
        if(!startUser.equals(user) || !startPwd.equals(pwd)){
            throw new SystemRuntimeException("重启作业认证用户或密码不正确");
        }

        LoginAccountVO loginUser = HttpRequestHandler.getRequestUser(true);
        dataTransferJobManager.restart(jobId,loginUser.getLoginName());
        return R.buildSuccess("重启作业成功");
    }
}
```

![](\assets\images\2023\springboot\data-transfer-1.png)

![](../../..\assets\images\2023\springboot\data-transfer-1.png)

### Job管理接口DataTransferJobManager

根据作业名称jobName启动任务，看数据迁移作业管理接口 `DataTransferJobManager`

```java
/**
 * <p>
 * 数据迁移作业 管理器接口
 * </p>
 *
 * @author liJY
 * @Date 2023-06-20
 */
public interface DataTransferJobManager {
    /**
     * 添加作业
     * @param job 作业
     */
    void add(DataTransferJob job);

    /**
     * 初始化作业
     */
    void initJobs();

    /**
     * 启动作业
     * @param jobName 作业名称
     * @param userAccount 用户帐号
     */
    void start(String jobName,String userAccount);

    /**
     * 重动作业
     * @param jobId 作业Id
     * @param userAccount 用户帐号
     */
    void restart(String jobId,String userAccount);

    /**
     * 获取执行器
     * @param jobName 作业名称
     * @param executorKey 执行器键名
     * @return 执行器
     */
    DataTransferExecutor getExecutor(String jobName,String executorKey);
}
```

接口实现类`DataTransferJobManagerImpl`

```java
@RequiredArgsConstructor(onConstructor_={@Autowired})
@Slf4j
@RefreshScope
public class DataTransferJobManagerImpl implements DataTransferJobManager {
    //作业映射.结构:Map<作业名称,DataTransferJob>
    private Map<String,DataTransferJob> jobMap=new HashMap<>();

    //MQ批量消息发送大小
    @Value(value="${rocketmq.batch-message-size.data-transfer:3500}")
    private int mqBatchMessageSize;
    //消息主题前缀
    @Value(value="${rocketmq.topic-prefix.data-transfer:TOPIC_DATA_TRANSFER}")
    private String mqTopicPrefix;
    //消费者组前缀
    @Value(value = "${rocketmq.consumer.group.prefix.data-transfer:CONSUMER_DATA_TRANSFER}")
    private String mqConsumerGroupPrefix;
    //NameServer
    @Value(value="${rocketmq.name-server}")
    private String mqNameServer;

    private DataTransferTaskService dataTransferTaskService;
    private IdGenerator idGenerator;
    private RocketMQTemplate rocketMQTemplate;
    private DataTransferItemService dataTransferItemService;
    private ApplicationContext applicationContext;

    @Autowired
    public void setDataTransferTaskService(DataTransferTaskService dataTransferTaskService) {
        this.dataTransferTaskService = dataTransferTaskService;
    }
    @Autowired
    public void setIdGenerator(IdGenerator idGenerator) {
        this.idGenerator = idGenerator;
    }
    @Autowired
    public void setRocketMQTemplate(RocketMQTemplate rocketMQTemplate) {
        this.rocketMQTemplate = rocketMQTemplate;
    }
    @Autowired
    public void setDataTransferItemService(DataTransferItemService dataTransferItemService) {
        this.dataTransferItemService = dataTransferItemService;
    }
    @Autowired
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    /**
     * 添加作业
     * @param job 作业
     */
    @Override
    public void add(DataTransferJob job) {
        if(jobMap.containsKey(job.getName())){
            log.warn("作业已存在,jobName:{}",job.getName());
            throw new SystemRuntimeException("当前作业已存在");
        }
        jobMap.put(job.getName(),job);
    }

    /**
     * 初始化作业
     */
    @Override
    @PostConstruct
    public void initJobs(){
        //获取所有作业
        if(jobMap.isEmpty()){
            log.warn("缺少作业信息,退出处理");
            return;
        }
        jobMap.forEach((jobName,job)->{
            try{
                log.info("开始初始化Job:{}",jobName);
                //根据作业的执行器,初始化MQ消费者.执行器结构:Map<执行器键名标识,DataTransferExecutor>
                Map<String,DataTransferExecutor> executorMap=job.getExecutorMap();
                if(executorMap.isEmpty()){
                    log.warn("作业缺少执行器信息,退出处理jobName:{}",job.getName());
                    return;
                }
                executorMap.forEach((executorKey,executor)->{
                    try{
                        log.info("开始初始化执行器:{}",executor.getKey());
                        initMqConsumer(job.getName(),executor);
                        log.info("完成初始化执行器:{}",executor.getKey());
                    }catch (Exception e){
                        log.error(MessageFormat.format("失败初始化执行器:{0}",executor.getKey()),e);
                    }
                });
                log.info("完成初始化Job:{}",jobName);
            }catch (Exception e){
                log.error(MessageFormat.format("失败初始化Job:{0}",jobName),e);
            }
        });
    }

    /**
     * 启动作业
     * @param jobName 作业名称
     * @param userAccount 用户帐号
     */
    @Override
    public void start(String jobName,String userAccount) {
        //1.获取作业任务执行器映射
        DataTransferJob job=jobMap.get(jobName);
        if(job==null){
            log.warn("缺少作业信息,jobName:{}",jobName);
            throw new SystemRuntimeException("缺少作业信息");
        }
        //结构:Map<执行器键名标识,DataTransferExecutor>
        Map<String,DataTransferExecutor> executorMap=job.getExecutorMap();

        //2.每一次运行,则属不同的作业实例,生成作业id
        String jobId=idGenerator.nextId("");

        //3.根据执行器创建任务实例
        executorMap.forEach((executorKey,executor)->{
            //通过处理器,获取数据总量
            if(executor.getHandler()==null){
                log.warn("执行器缺少处理器,executorKey:{}",executor.getKey());
                throw new SystemRuntimeException("执行器缺少处理器");
            }
            int total=executor.getHandler().readTotal();

            //创建启动模式任务实例.新启动任务时,来源任务Id为'空'
            DataTransferTaskInstance taskInstance=this.createTask(jobName,jobId,"",executor,DataTransferStartModeEnum.START,total,userAccount);
            if(taskInstance.getTotal()>0){
                //启动任务
                this.startTask(taskInstance);
            }
        });
    }
    
      /**
     * 重动作业
     * @param jobId 作业Id
     * @param userAccount 用户帐号
     */
    @Override
    public void restart(String jobId,String userAccount){
        //1.获取作业下可重启的任务
        List<DataTransferTask> tasks=this.dataTransferTaskService.getRestartTasks(jobId);
        if(CollectionUtils.isEmpty(tasks)){
            log.warn("没有需要重启的任务,jobId:{}",jobId);
            throw new SystemRuntimeException("没有需要重启的任务");
        }

        //2.获取作业任务执行器映射
        //==获取作业名称(同一作业id,作业名称一致)
        String jobName=tasks.get(0).getJobName();
        DataTransferJob job=jobMap.get(jobName);
        if(job==null){
            log.warn("缺少作业信息,jobName:{}",jobName);
            throw new SystemRuntimeException("缺少作业信息");
        }
        //结构:Map<执行器键名标识,DataTransferExecutor>
        Map<String,DataTransferExecutor> executorMap=job.getExecutorMap();

        //3.每一次运行,则属不同的作业实例,生成作业id
        String newJobId=idGenerator.nextId("");

        //4.根据重启的任务列表,创建重启任务实例
        tasks.stream().forEach(task->{
            //获取执行器
            DataTransferExecutor executor=executorMap.get(task.getExecutorKey());

            //获取处理器,并获取任务下迁移失败事项的总数据量
            if(executor.getHandler()==null){
                log.warn("执行器缺少处理器,executorKey:{}",executor.getKey());
                throw new SystemRuntimeException("执行器缺少处理器");
            }
            int total=this.dataTransferItemService.getRestartItemCount(task.getId());

            //创建重启模式的任务实例.来源任务Id为待重启的任务Id
            DataTransferTaskInstance taskInstance=this.createTask(jobName,newJobId,task.getId(),executor,DataTransferStartModeEnum.RESTART,total,userAccount);
            //启动任务
            this.startTask(taskInstance);
        });
    }

     /**
     * 获取执行器
     * @param jobName 作业名称
     * @param executorKey 执行器键名
     * @return 执行器
     */
    @Override
    public DataTransferExecutor getExecutor(String jobName,String executorKey){
        //获取作业
        DataTransferJob job=jobMap.get(jobName);
        if(job==null){
            throw new SystemRuntimeException("缺少作业信息");
        }

        //获取执行器
        DataTransferExecutor executor=job.getExecutorMap().get(executorKey);
        if(executor==null){
            throw new SystemRuntimeException("缺少执行器信息");
        }
        return executor;
    }
    
    /**
     * 初始化MQ消费者
     * @param jobName 作业名称
     * @param executor 执行器
     */
    private void initMqConsumer(String jobName, DataTransferExecutor executor){
        String consumerGroup=getConsumerGroup(mqConsumerGroupPrefix,jobName,executor.getKey());
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(consumerGroup);
        consumer.setConsumeThreadMin(executor.getThreadSize());
        consumer.setConsumeThreadMax(executor.getThreadSize());
        //每次只消费1条消息,批量消费的最大消息数量，缓存的消息数量达到参数设置的值，Push消费者SDK会将缓存的消息统一提交给消费线程，实现批量消费
        consumer.setConsumeMessageBatchMaxSize(1);
        consumer.setConsumeTimeout(executor.getTimeout());
        consumer.setNamesrvAddr(mqNameServer);
        //不作重试处理,若想重试任务,需使用任务的restart功能
        consumer.setMaxReconsumeTimes(0);
        //获取topic
        String topic=this.getMsgTopic(mqTopicPrefix,jobName);
        //获取tag
        String tag=this.getMsgTag(executor.getKey());
        try {
            /*
            说明:
            1.一个作业,使用一个topic
            2.一个执行器,使用一个consumer,但使用不同tag
            */
            consumer.subscribe(topic, tag);
            consumer.registerMessageListener(new MessageListenerConcurrently() {
                @Override
                public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                    //每次只会消费1条消息
                    MessageExt msgExt=msgs.get(0);
                    //输出消息日志
                    Map<String,Object> msgFields=new HashMap<>();
                    msgFields.put("msgId",msgExt.getMsgId());
                    msgFields.put("reconsumeTimes",msgExt.getReconsumeTimes());
                    msgFields.put("keys",msgExt.getKeys());
                    msgFields.put("topic",msgExt.getTopic());
                    msgFields.put("tags",msgExt.getTags());
                    String body=new String(msgExt.getBody(), StandardCharsets.UTF_8);
                    msgFields.put("body",body);
                    log.info("接收到数据迁移消息,msgFields:{}", msgFields);

                    DataTransferBatch batch=null;
                    try{
                        //转换为批次信息
                        batch=JsonUtils.toObject(body,DataTransferBatch.class);
                    }catch (Exception e){
                       log.error(MessageFormat.format("数据迁移批次信息转换异常,msgFields:{}",msgFields),e);

                        //无论如何,均返回消费成功
                        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                    }

                    //执行单批次数据迁移
                    try {
                        log.info("执行单批次数据迁移开始.jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
                        DataTransferBatchHandler batchHandle = (DataTransferBatchHandler) applicationContext.getBean(batch.getModel().getBeanName());
                        batchHandle.execute(batch);
                        log.info("执行单批次数据迁移完成.jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
                    }catch (Exception e){
                        log.error(MessageFormat.format("执行单批次数据迁移失败.jobName:{0},taskId:{1},batchNum:{2}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum()),e);
                    }

                    //无论如何,均返回消费成功
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                }
            });
            consumer.start();
        }catch (Exception e){
            log.error(MessageFormat.format("数据迁移MQ消费者启动失败,jobName:{0},executorKey:{1}",jobName,executor.getKey()),e);
            throw new SystemRuntimeException("数据迁移MQ消费者启动失败");
        }
    }

    /**
     * 创建任务
     * @param jobName 作业名称
     * @param jobId 作业Id
     * @param sourceId 来源任务Id
     * @param executor 执行器
     * @param startMode 启动模式
     * @param total 总数据量
     * @param userAccount 用户帐号
     * @return 任务实例
     */
    private DataTransferTaskInstance createTask(String jobName, String jobId, String sourceId,DataTransferExecutor executor, DataTransferStartModeEnum startMode,int total,String userAccount){
        //获取处理的总批次量
        int batchCount=this.getBatchCount(total,executor.getBatchSize());

        //保存任务
        //==新启动任务,来源任务Id为空
        DataTransferTask task=new DataTransferTask();
        task.setJobName(jobName);
        task.setJobId(jobId);
        task.setExecutorKey(executor.getKey());
        task.setBatchSize(executor.getBatchSize());
        task.setBatchCount(batchCount);
        task.setModuleType(executor.getModuleType());
        task.setBusinessType(executor.getBusinessType());
        task.setTotal(total);
        task.setSourceId(sourceId);
        String taskId=dataTransferTaskService.add(task,userAccount);

        //创建任务实例
        DataTransferTaskInstance taskInstance=new DataTransferTaskInstance();
        taskInstance.setJobName(jobName);
        taskInstance.setJobId(jobId);
        taskInstance.setExecutorKey(executor.getKey());
        taskInstance.setTaskId(taskId);
        taskInstance.setSourceId(sourceId);
        taskInstance.setBatchSize(executor.getBatchSize());
        taskInstance.setBatchCount(batchCount);
        taskInstance.setTotal(total);
        taskInstance.setModuleType(executor.getModuleType());
        taskInstance.setBusinessType(executor.getBusinessType());
        taskInstance.setThreadSize(executor.getThreadSize());
        taskInstance.setTimeout(executor.getTimeout());
        taskInstance.setModel(startMode);
        return taskInstance;
    }


    /**
     * 获取数据批次处理总量
     * @param total 总数量
     * @param batchSize 批次大小
     * @return 批次总量
     */
    private int getBatchCount(Integer total,Integer batchSize){
        if(total==0){
            return 0;
        }
        int count=total/batchSize;
        if(total % batchSize !=0){
            count=count+1;
        }
        return count;
    }

    /**
     * 运行任务实例
     * @param taskInstance 任务实例
     */
    private void startTask(DataTransferTaskInstance taskInstance){
        //根据处理的批次总量,按批次发送MQ消息
        List<Message<DataTransferBatch>> messages = new ArrayList<>();

        for(int batchNum=1;batchNum<=taskInstance.getBatchCount();batchNum++){
            //创建批次信息
            DataTransferBatch batch=new DataTransferBatch();
            batch.setJobName(taskInstance.getJobName());
            batch.setExecutorKey(taskInstance.getExecutorKey());
            batch.setTaskId(taskInstance.getTaskId());
            batch.setSourceId(taskInstance.getSourceId());
            batch.setBatchSize(taskInstance.getBatchSize());
            batch.setBatchCount(taskInstance.getBatchCount());
            batch.setBatchNum(batchNum);
            batch.setTotal(taskInstance.getTotal());
            batch.setModel(taskInstance.getModel());
            //获取消息Key
            String msgKey=this.getMsgKey(batch.getExecutorKey(),batchNum);
            messages.add(MessageBuilder.withPayload(batch).setHeader("KEYS",msgKey).build());

            //判断是否需执行消息发送
            boolean isSend=this.isSendMessage(messages.size(),mqBatchMessageSize,batchNum,taskInstance.getBatchCount());
            if(isSend){
                //获取消息目的地
                String msgDest=MessageFormat.format("{0}:{1}",this.getMsgTopic(mqTopicPrefix,batch.getJobName()),this.getMsgTag(batch.getExecutorKey()));
                //发送消息
                SendResult sendRet=rocketMQTemplate.syncSend(msgDest,messages,10*1000);
                if(!SendStatus.SEND_OK.equals(sendRet.getSendStatus())){
                    log.warn("数据迁移消息发送失败,batch:{},sendRet:{}",batch,sendRet);
                    throw new SystemRuntimeException("消息发送失败");
                }
                //消息发送完毕,则清除
                messages.clear();
            }
        }
    }

    /**
     * 是否执行消息发送
     * @param currentMsgSize 当前消息大小
     * @param sendSize 消息发送大小
     * @param currentMsgCount 当前消息总量
     * @param msgTotal 消息总量
     * @return 是、否
     */
    private Boolean isSendMessage(Integer currentMsgSize,Integer sendSize,Integer currentMsgCount,Integer msgTotal){
        if(currentMsgSize % sendSize==0 || currentMsgCount>=msgTotal){
            //当前消息大小到达发送大小,或当前消息已是最后一条消息,则执行发送
            return true;
        }
        return false;
    }

    /**
     * 获取消息Topic
     * @param topicPrefix 消息主题前缀
     * @param jobName 作业名称
     */
    private String getMsgTopic(String topicPrefix,String jobName){
        //作业名称:作为topic
        return MessageFormat.format("{0}_{1}",topicPrefix,jobName.toUpperCase());
    }

    /**
     * 获取消息tag
     * @param executorKey 执行器键名标识
     * @return 消息tag
     */
    private String getMsgTag(String executorKey){
        //执行器键名标识:作为tag
        return MessageFormat.format("TAG_{0}",executorKey.toUpperCase());
    }

    /**
     * 获取消息Key
     * @param executorKey 执行器键名标识
     * @param batchNum 批次序号
     * @return 消息Key
     */
    private String getMsgKey(String executorKey,int batchNum){
        //Key=执行器键名标识_批次序号
        String result=MessageFormat.format("{0}_{1}",executorKey,batchNum);
        return result;
    }

    /**
     * 获取消费者组
     * @param groupPrefix 消费者组前缀
     * @param jobName 作业名称
     * @param executorKey 执行器名称
     * @return 消费者组全名称
     */
    private String getConsumerGroup(String groupPrefix,String jobName,String executorKey){
        String result=MessageFormat.format("{0}_{1}_{2}",groupPrefix,jobName.toUpperCase(),executorKey.toUpperCase());
        return result;
    }
}
```



### 数据迁移作业类DataTransferJob

```java
/**
 * <p>
 * 数据迁移作业 对象
 * </p>
 *
 * @author liJY
 * @Date 2023-06-20
 */
@Slf4j
@Getter
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@ApiModel(value="DataTransferJob对象", description="数据迁移作业 对象")
public class DataTransferJob{

    /**
     * 名称
     */
    private String name;

    /**
     * 任务执行器映射.结构:Map(执行器键名标识,DataTransferExecutor)
     */
    private Map<String,DataTransferExecutor> executorMap=new HashMap<>();

    /**
     * 构造函数
     * @param name 名称
     */
    public DataTransferJob(String name){
        this.name=name;
    }

    /**
     * 添加执行器
     * @param executor 执行器
     */
    public void add(DataTransferExecutor executor){
        if(StringUtils.isEmpty(executor.getKey())){
            log.warn("执行器缺少键名标识");
            throw new SystemRuntimeException("执行器缺少键名标识");
        }
        if(executorMap.containsKey(executor.getKey())){
            log.warn("执行器已存在,jobName:{},key:{}",this.name,executor.getKey());
            throw new SystemRuntimeException("执行器已存在");
        }
        executorMap.put(executor.getKey(),executor);
    }
}
```

### 数据迁移执行器DataTransferExecutor

一个作业可配多个执行器

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
public class DataTransferExecutor<SOURCE,TARGET>{
    /**
     * 键名标识
     */
    private String key;

    /**
     * 数据处理批次大小
     */
    private int batchSize;

    /**
     * 处理线程大小(消费者线程数)
     */
    private int threadSize;

    /**
     * 处理超时时间.单位:分钟
     */
    private int timeout=15;

    /**
     * 模块类型
     */
    private DataTransferModuleTypeEnum moduleType;

    /**
     * 业务类型
     */
    private DataTransferBusinessTypeEnum businessType;

    /**
     * 处理器
     */
    private DataTransferHandler<SOURCE,TARGET> handler;
}
```

### 数据迁移处理接口DataTransferHandler

```java
/**
 * 数据迁移处理器
 * @author liJY
 * @Date 2023-06-20
 * @param <SOURCE> 数据来源对象类型
 * @param <TARGET> 数据目标对象类型
 */
public interface DataTransferHandler<SOURCE,TARGET> {
    /**
     * 读取来源处理事项总数(用于首次启动任务场景)
     * @return 总数量
     */
    int readTotal();

    /**
     * 读取来源数据(用于首次启动任务场景)
     * @param batchSize 批次大小
     * @param batchNum 批次序号
     * @return 来源数据列表.返回数量比批次数量少的,则为跳过处理,空:没有待处理数据
     */
    List<DataTransferSource<SOURCE>> read(int batchSize, int batchNum);

    /**
     * 读取来源数据(用于重启任务场景)
     * @param businessIds 业务Id列表
     * @return 来源数据列表.返回数量比入参数量少的,则为跳过处理,空:没有待处理数据
     */
    List<DataTransferSource<SOURCE>> read(List<String> businessIds);

    /**
     * 数据处理
     * @param sources 来源数据列表
     * @return 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
     */
    List<DataTransferResult<TARGET>> process(List<DataTransferSource<SOURCE>> sources);

    /**
     * 写入目标数据
     * @param targets 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
     */
    List<DataTransferResult> write(List<TARGET> targets);
}
```

### 数据迁移配置类DataTransferConfig

```java
@Configuration
public class DataTransferConfig {
    private final DataTransferBusinessProperties dataTransferBusinessProperties;
    private final ApplicationContext applicationContext;

    public DataTransferConfig(DataTransferBusinessProperties dataTransferBusinessProperties,ApplicationContext applicationContext){
        this.dataTransferBusinessProperties=dataTransferBusinessProperties;
        this.applicationContext=applicationContext;
    }
    
    /**
     * 实例对象注入Spring IOC容器
     * @return
     */
    @Bean
    public DataTransferJobManager jobManager(){
        DataTransferJobManager manager=new DataTransferJobManagerImpl();
        //合同交付模块Job
        manager.add(this.buildDeliverRegisterJob());
        //停复工模块Job
        manager.add(this.buildWorkResumeJob());
        //停复工模块-复工确认Job，必须跑完停复工申请的Job才能跑复工确认的Job，因为业务数据存在前后时间关联
        manager.add(this.buildResumeConfirmDetailJob());
        return manager;
    }

    /**
     * 构建合同交付数据迁移作业
     * @return 数据迁移作业
     */
    private DataTransferJob buildDeliverRegisterJob(){
        //定义合同交付申请任务执行器
        DataTransferBusinessProperties.TaskConfig applyTaskConfig=dataTransferBusinessProperties.getDeliverRegisterApply();
        DataTransferExecutor applyExecutor=new DataTransferExecutor();
        applyExecutor.setKey("apply");
        applyExecutor.setBatchSize(applyTaskConfig.getBatchSize());
        applyExecutor.setThreadSize(applyTaskConfig.getThreadSize());
        applyExecutor.setTimeout(applyTaskConfig.getTimeout());
        applyExecutor.setModuleType(DataTransferModuleTypeEnum.DELIVER_REGISTER);
        applyExecutor.setBusinessType(DataTransferBusinessTypeEnum.DELIVER_REGISTER_APPLY);
        applyExecutor.setHandler(this.applicationContext.getBean(DeliverRegisterDtApplyHandler.class));

        //定义合同交付申请明细任务执行器
        DataTransferBusinessProperties.TaskConfig detailTaskConfig=dataTransferBusinessProperties.getDeliverRegisterDetail();
        DataTransferExecutor detailExecutor=new DataTransferExecutor();
        detailExecutor.setKey("detail");
        detailExecutor.setBatchSize(detailTaskConfig.getBatchSize());
        detailExecutor.setThreadSize(detailTaskConfig.getThreadSize());
        detailExecutor.setTimeout(detailTaskConfig.getTimeout());
        detailExecutor.setModuleType(DataTransferModuleTypeEnum.DELIVER_REGISTER);
        detailExecutor.setBusinessType(DataTransferBusinessTypeEnum.DELIVER_REGISTER_APPLY_DETAIL);
        detailExecutor.setHandler(this.applicationContext.getBean(DeliverRegisterDtDetailHandler.class));

        DataTransferJob job=new DataTransferJob("deliverRegister");
        job.add(applyExecutor);
        job.add(detailExecutor);
        return job;
    }

    /**
     * 构建停复工数据迁移作业
     * @return 数据迁移作业
     */
    private DataTransferJob buildWorkResumeJob(){
        //定义停工计划申请任务执行器
        DataTransferBusinessProperties.TaskConfig stopPlanApplyTaskConfig=dataTransferBusinessProperties.getWorkResumeStopPlanApply();
        DataTransferExecutor stopPlanApplyExecutor=new DataTransferExecutor();
        stopPlanApplyExecutor.setKey("stopPlanApply");
        stopPlanApplyExecutor.setBatchSize(stopPlanApplyTaskConfig.getBatchSize());
        stopPlanApplyExecutor.setThreadSize(stopPlanApplyTaskConfig.getThreadSize());
        stopPlanApplyExecutor.setTimeout(stopPlanApplyTaskConfig.getTimeout());
        stopPlanApplyExecutor.setModuleType(DataTransferModuleTypeEnum.WORK_RESUME);
        stopPlanApplyExecutor.setBusinessType(DataTransferBusinessTypeEnum.WORK_RESUME_STOP_PLAN_APPLY);
        stopPlanApplyExecutor.setHandler(this.applicationContext.getBean(WorkResumeDtStopPlanApplyHandler.class));
        
        //定义停工计划明细任务执行器
        DataTransferBusinessProperties.TaskConfig resumeDetailTaskConfig=dataTransferBusinessProperties.getWorkResumeStopPlanDetail();
        DataTransferExecutor stopPlanDetailExecutor = new DataTransferExecutor();
        stopPlanDetailExecutor.setKey("stopPlanDetail");
        stopPlanDetailExecutor.setBatchSize(resumeDetailTaskConfig.getBatchSize());
        stopPlanDetailExecutor.setThreadSize(resumeDetailTaskConfig.getThreadSize());
        stopPlanDetailExecutor.setTimeout(resumeDetailTaskConfig.getTimeout());
        stopPlanDetailExecutor.setModuleType(DataTransferModuleTypeEnum.WORK_RESUME);
        stopPlanDetailExecutor.setBusinessType(DataTransferBusinessTypeEnum.WORK_RESUME_STOP_PLAN_DETAIL);
        stopPlanDetailExecutor.setHandler(this.applicationContext.getBean(WorkResumeDtStopPlanDetailHandler.class));
        
        //定义复工确认申请任务执行器
        DataTransferBusinessProperties.TaskConfig confirmApplyTaskConfig=dataTransferBusinessProperties.getWorkResumeConfirmApply();
        DataTransferExecutor confirmApplyExecutor=new DataTransferExecutor();
        confirmApplyExecutor.setKey("confirmApply");
        confirmApplyExecutor.setBatchSize(confirmApplyTaskConfig.getBatchSize());
        confirmApplyExecutor.setThreadSize(confirmApplyTaskConfig.getThreadSize());
        confirmApplyExecutor.setTimeout(confirmApplyTaskConfig.getTimeout());
        confirmApplyExecutor.setModuleType(DataTransferModuleTypeEnum.WORK_RESUME);
        confirmApplyExecutor.setBusinessType(DataTransferBusinessTypeEnum.WORK_RESUME_CONFIRM_APPLY);
        confirmApplyExecutor.setHandler(this.applicationContext.getBean(WorkResumeDtConfirmApplyHandler.class));
        
        DataTransferJob job=new DataTransferJob("workResume");
        job.add(stopPlanApplyExecutor);
        job.add(confirmApplyExecutor);
        job.add(stopPlanDetailExecutor);
        return job;
    }
    
    /**
     * 构建复工确认明细数据迁移作业
     */
    private DataTransferJob buildResumeConfirmDetailJob(){
        //定义停复工确认明细执行器
        DataTransferBusinessProperties.TaskConfig confirmDetailTaskConfig=dataTransferBusinessProperties.getWorkResumeConfirmDetail();
        DataTransferExecutor confirmDetailExecutor = new DataTransferExecutor();
        confirmDetailExecutor.setKey("resumeConfirmDetail");
        confirmDetailExecutor.setBatchSize(confirmDetailTaskConfig.getBatchSize());
        confirmDetailExecutor.setThreadSize(confirmDetailTaskConfig.getThreadSize());
        confirmDetailExecutor.setTimeout(confirmDetailTaskConfig.getTimeout());
        confirmDetailExecutor.setModuleType(DataTransferModuleTypeEnum.WORK_RESUME);
        confirmDetailExecutor.setBusinessType(DataTransferBusinessTypeEnum.WORK_RESUME_CONFIRM_DETAIL);
        confirmDetailExecutor.setHandler(this.applicationContext.getBean(WorkResumeDtConfirmDetailHandler.class));
        DataTransferJob job=new DataTransferJob("resumeConfirmDetail");
        job.add(confirmDetailExecutor);
        return job;
    }
}
```

数据迁移业务模块属性类`DataTransferBusinessProperties`

```java
@Component
@Data
@RefreshScope
@ConfigurationProperties(prefix = "data-transfer.business")
public class DataTransferBusinessProperties {
    /**
     * 合同交付申请迁移任务配置
     */
    TaskConfig deliverRegisterApply;

    /**
     * 合同交付申请明迁移任务配置
     */
    TaskConfig deliverRegisterDetail;
    
    /**
     * 停复工_计划停工申请任务配置
     */
    TaskConfig workResumeStopPlanApply;
    
    TaskConfig workResumeStopPlanDetail;

    /**
     * 停复工_复工确认申请任务配置
     */
    TaskConfig workResumeConfirmApply;
    
    TaskConfig workResumeConfirmDetail;

    /**
     * 任务配置
     */
    @Data
    public static class TaskConfig{
        /**
         * 数据处理批次大小
         */
        private Integer batchSize;

        /**
         * 处理线程大小
         */
        private Integer threadSize;

        /**
         * 处理超时时间.单位:分钟
         */
        private Integer timeout;

        /**
         * 来源数据迁移条件
         */
        private DataTransferSourceQuery query;
    }
}
```

注意增加了注解@RefreshScope，当配置变更时可以在不重启应用的前提下刷新bean中相关的属性值，配置文件bootstrap.properties增加属性配置

```properties
#数据迁移配置
##合同交付申请
data-transfer.business.deliver-register-apply.batch-size=500
data-transfer.business.deliver-register-apply.thread-size=10
data-transfer.business.deliver-register-apply.timeout=15
#data-transfer.business.deliver-register-apply.query.begin-time=2023-08-07 10:01:01
#data-transfer.business.deliver-register-apply.query.end-time=2023-08-07 10:01:01
data-transfer.business.deliver-register-apply.query.ids[0]=2271D0F8-3D2D-4561-9DE6-ACCA790A5BF7
##合同交付申请明细
data-transfer.business.deliver-register-detail.batch-size=200
data-transfer.business.deliver-register-detail.thread-size=10
data-transfer.business.deliver-register-detail.timeout=20
#data-transfer.business.deliver-register-detail.query.begin-time=2023-08-07 10:01:01
#data-transfer.business.deliver-register-detail.query.end-time=2023-08-07 10:01:01
data-transfer.business.deliver-register-detail.query.ids[0]=2271D0F8-3D2D-4561-9DE6-ACCA790A5BF7
##停复工_计划停工申请
data-transfer.business.work-resume-stop-plan-apply.batch-size=500
data-transfer.business.work-resume-stop-plan-apply.thread-size=10
data-transfer.business.work-resume-stop-plan-apply.timeout=15
#data-transfer.business.work-resume-stop-plan-apply.query.begin-time=2023-08-07 10:01:01
#data-transfer.business.work-resume-stop-plan-apply.query.end-time=2023-08-07 10:01:01
data-transfer.business.work-resume-stop-plan-apply.query.ids[0]=4DFE0593-2B50-4DDF-A511-B57554D141B4

##停复工_计划停工申请明细
data-transfer.business.work-resume-stop-plan-detail.batch-size=1000
data-transfer.business.work-resume-stop-plan-detail.thread-size=1
data-transfer.business.work-resume-stop-plan-detail.timeout=30
#data-transfer.business.work-resume-stop-plan-detail.query.begin-time=2023-08-07 10:01:01
#data-transfer.business.work-resume-stop-plan-detail.query.end-time=2023-08-07 10:01:01
data-transfer.business.work-resume-stop-plan-detail.query.ids[0]=4DFE0593-2B50-4DDF-A511-B57554D141B4

##停复工_复工确认申请
data-transfer.business.work-resume-confirm-apply.batch-size=500
data-transfer.business.work-resume-confirm-apply.thread-size=10
data-transfer.business.work-resume-confirm-apply.timeout=15
#data-transfer.business.work-resume-confirm-apply.query.begin-time=2023-08-07 10:01:01
#data-transfer.business.work-resume-confirm-apply.query.end-time=2023-08-07 10:01:01
#data-transfer.business.work-resume-confirm-apply.query.ids[0]=83877BBA-573D-4A20-B736-7E244859B627
##停复工_复工确认申请明细
data-transfer.business.work-resume-confirm-detail.batch-size=1000
data-transfer.business.work-resume-confirm-detail.thread-size=20
data-transfer.business.work-resume-confirm-detail.timeout=30
#data-transfer.business.work-resume-confirm-detail.query.begin-time=2023-08-07 10:01:01
#data-transfer.business.work-resume-confirm-detail.query.end-time=2023-08-07 10:01:01
#data-transfer.business.work-resume-confirm-detail.query.ids[0]=5B8E8553-8997-4457-AC7A-94A4172FAF17
```

迁移条件类`DataTransferSourceQuery`

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@ApiModel(value = "数据迁移_来源数据查询条件对象", description = "数据迁移_来源数据查询条件对象")
public class DataTransferSourceQuery {
    /**
     * 开始时间
     */
    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime beginTime;
    /**
     * 结束时间
     */
    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime endTime;
    /**
     * Id列表
     */
    private List<String> ids;
}
```

### 数据迁移任务表DataTransferTask

```java
/**
 * <p>
 * 数据迁移任务表
 * </p>
 *
 * @author liJY
 * @Date 2023-06-20
 */
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@TableName("t_data_transfer_task")
public class DataTransferTask extends Model<DataTransferTask> {

    private static final long serialVersionUID = 1L;

    /**
     * 主键名称
     */
    @TableId(value = "id", type = IdType.ASSIGN_ID)
    private String id;

    /**
     * 创建人bip
     */
    @TableField("create_user")
    private String createUser;

    /**
     * 创建时间
     */
    @TableField("create_time")
    private LocalDateTime createTime;

    /**
     * 更新人bip
     */
    @TableField("update_user")
    private String updateUser;

    /**
     * 更新时间
     */
    @TableField("update_time")
    private LocalDateTime updateTime;

    /**
     * 租户编码
     */
    @TableField("tenant_id")
    private String tenantId;

    /**
     * 版本号
     */
    @TableField("version_number")
    private String versionNumber;

    /**
     * 备注
     */
    @TableField("remark")
    private String remark;

    /**
     * 作业名称
     */
    @TableField("job_name")
    private String jobName;

    /**
     * 作业Id
     */
    @TableField("job_id")
    private String jobId;

    /**
     * 执行器键名标识
     */
    @TableField("executor_key")
    private String executorKey;

    /**
     * 来源任务Id
     */
    @TableField("source_id")
    private String sourceId;

    /**
     * 批次大小
     */
    @TableField("batch_size")
    private Integer batchSize;

    /**
     * 批次总量
     */
    @TableField("batch_count")
    private Integer batchCount;

    /**
     * 开始时间
     */
    @TableField("start_time")
    private LocalDateTime startTime;

    /**
     * 结束时间
     */
    @TableField("end_time")
    private LocalDateTime endTime;

    /**
     * 模块类型
     */
    @TableField("module_type")
    private DataTransferModuleTypeEnum moduleType;

    /**
     * 业务类型
     */
    @TableField("business_type")
    private DataTransferBusinessTypeEnum businessType;

    /**
     * 状态
     */
    @TableField("status")
    private DataTransferTaskStatusEnum status;

    /**
     * 总数据量
     */
    @TableField("total")
    private Integer total;

    /**
     * 成功事项数量
     */
    @TableField("success")
    private Integer success;

    /**
     * 失败事项数量
     */
    @TableField("fail")
    private Integer fail;

    /**
     * 跳过事项数量
     */
    @TableField("skip")
    private Integer skip;

    @Override
    protected Serializable pkVal() {
        return this.id;
    }

}
```

- 枚举类

  ```java
  /**
   * 数据迁移模块类型枚举
   *
   * @author liJY
   */
  
  public enum DataTransferModuleTypeEnum implements BaseEnum<Integer> {
      /**
       * 合同交付
       */
      DELIVER_REGISTER(1,"合同交付"),
      /**
       * 停复工
       */
      WORK_RESUME(2, "停复工"),
      ;
  
  
      private final Integer value;
  
      private final String displayName;
  
      DataTransferModuleTypeEnum(Integer value, String displayName) {
          this.value = value;
          this.displayName = displayName;
      }
  
      /**
       * 枚举value
       *
       * @return
       */
      @Override
      public Integer getValue() {
          return value;
      }
  
      /**
       * 枚举的显示名字
       *
       * @return
       */
      @Override
      public String getDisplayName() {
          return displayName;
      }
  }
  
  /**
   * 数据迁移事项业务类型枚举
   *
   * @author liJY
   */
  
  public enum DataTransferBusinessTypeEnum implements BaseEnum<Integer> {
      /**
       * 合同交付_申请
       */
      DELIVER_REGISTER_APPLY(1,"合同交付_申请"),
      /**
       * 合同交付_申请明细
       */
      DELIVER_REGISTER_APPLY_DETAIL(2, "合同交付_申请明细"),
      /**
       * 停复工_停工计划申请
       */
      WORK_RESUME_STOP_PLAN_APPLY(3,"停复工_停工计划申请"),
      /**
       * 停复工_复工确认申请
       */
      WORK_RESUME_CONFIRM_APPLY(4,"停复工_复工确认申请"),
      /**
       * 停复工_停工计划明细
       */
      WORK_RESUME_STOP_PLAN_DETAIL(5,"停复工_停工计划明细"),
      /**
       * 停复工_复工确认明细
       */
      WORK_RESUME_CONFIRM_DETAIL(6,"停复工_复工确认明细"),
      ;
  
      private final Integer value;
  
      private final String displayName;
  
      DataTransferBusinessTypeEnum(Integer value, String displayName) {
          this.value = value;
          this.displayName = displayName;
      }
  
      /**
       * 枚举value
       *
       * @return
       */
      @Override
      public Integer getValue() {
          return value;
      }
  
      /**
       * 枚举的显示名字
       *
       * @return
       */
      @Override
      public String getDisplayName() {
          return displayName;
      }
  }
  
  /**
   * 数据迁移任务状态枚举
   *
   * @author liJY
   */
  
  public enum DataTransferTaskStatusEnum implements BaseEnum<Integer> {
      /**
       * 进行中
       */
      PROGRESS(1,"进行中"),
      /**
       * 完成
       */
      FINISH(2, "完成");
  
      private final Integer value;
  
      private final String displayName;
  
      DataTransferTaskStatusEnum(Integer value, String displayName) {
          this.value = value;
          this.displayName = displayName;
      }
  
      /**
       * 枚举value
       *
       * @return
       */
      @Override
      public Integer getValue() {
          return value;
      }
  
      /**
       * 枚举的显示名字
       *
       * @return
       */
      @Override
      public String getDisplayName() {
          return displayName;
      }
  }
  ```

- Service接口DataTransferTaskService

  ORM框架使用MybatisPlus

  ```java
  public interface DataTransferTaskService extends IService<DataTransferTask> {
      /**
       * 添加新任务
       * @param task 任务信息
       * @param userAccount 用户帐号
       * @return 任务Id
       */
      String add(DataTransferTask task, String userAccount);
  
      /**
       * 递增事项状态数量
       * @param taskId 任务Id
       * @param success 递增成功数量.0:不作改变
       * @param fail 递增失败数量.0:不作改变
       * @param skip 递增跳过数量.0:不作改变
       * @return 是、否已完成任务
       */
      boolean incrItemCount(String taskId,int success,int fail,int skip);
  
      /**
       * 获取作业下可重启的任务列表
       * @param jobId 作业Id
       * @return 任务列表
       */
      List<DataTransferTask> getRestartTasks(String jobId);
  }
  ```

  服务接口实现类DataTransferTaskServiceImpl

  ```java
  @RequiredArgsConstructor(onConstructor_={@Autowired})
  @Slf4j
  @Service
  public class DataTransferTaskServiceImpl extends ServiceImpl<DataTransferTaskMapper, DataTransferTask> implements DataTransferTaskService {
      /**
       * 添加新任务
       * @param task 任务信息
       * @param userAccount 用户帐号
       * @return 任务Id
       */
      @Override
      public String add(DataTransferTask task, String userAccount){
          //保存前填充信息
          LocalDateTime now=LocalDateTime.now();
          task.setCreateUser(userAccount);
          task.setCreateTime(now);
          task.setStartTime(now);
          if(task.getTotal()==0){
              //总数据量为0,表示完成
              task.setStatus(DataTransferTaskStatusEnum.FINISH);
              task.setEndTime(now);
          }else{
              task.setStatus(DataTransferTaskStatusEnum.PROGRESS);
          }
          task.setSuccess(0);
          task.setFail(0);
          task.setSkip(0);
          baseMapper.insert(task);
  
          return task.getId();
      }
  
      /**
       * 递增事项状态数量
       * @param taskId 任务Id
       * @param success 递增成功数量.0:不作改变
       * @param fail 递增失败数量.0:不作改变
       * @param skip 递增跳过数量.0:不作改变
       * @return 是、否已完成任务
       */
      @Override
      public boolean incrItemCount(String taskId,int success,int fail,int skip){
          //递增成功数量
          this.incrSuccess(taskId,success);
          //递增失败数量
          this.incrFail(taskId,fail);
          //递增跳过数量
          this.incrSkip(taskId,skip);
  
          //检查任务是否已完成
          int count=success+fail+skip;
          if(count<=0){
              //数量没发生变化,并未改变完成状态,因此返回'未完成任务'
              return false;
          }
          return this.checkFinish(taskId);
      }
  
       /**
       * 获取作业下可重启的任务列表
       * @param jobId 作业Id
       * @return 任务列表
       */
      @Override
      public List<DataTransferTask> getRestartTasks(String jobId){
          //只对存在:事项主体类型为'数据',且事项状态为'异常'的任务,用任务状态为'完成'的任务执行重启
          return baseMapper.getRestartTasks(jobId, DataTransferTaskStatusEnum.FINISH,DeletedEnum.NOT_DELETE, DataTransferSubjectTypeEnum.DATA
                  , Arrays.asList(DataTransferItemStatusEnum.BUSINESS_ERROR,DataTransferItemStatusEnum.SYSTEM_ERROR));
      }
      
      /**
       * 递增成功数量
       * @param taskId 任务Id
       * @param value 递增值
       */
      private void incrSuccess(String taskId,int value){
          if(value<=0){
              return;
          }
          int row= baseMapper.incrSuccess(taskId,value);
          if(row==0){
              log.warn("更新成功数量失败,taskId:{}",taskId);
          }
      }
  
      /**
       * 递增失败数量
       * @param taskId 任务Id
       * @param value 递增值
       */
      private void incrFail(String taskId,int value){
          if(value<=0){
              return;
          }
          int row= baseMapper.incrFail(taskId,value);
          if(row==0){
              log.warn("更新失败数量失败,taskId:{}",taskId);
          }
      }
  
      /**
       * 递增跳过数量
       * @param taskId 任务Id
       * @param value 递增值
       */
      private void incrSkip(String taskId,int value){
          if(value<=0){
              return;
          }
          int row= baseMapper.incrSkip(taskId,value);
          if(row==0){
              log.warn("更新跳过数量失败,taskId:{}",taskId);
          }
      }
  
      /**
       * 检查完成状态
       * @param taskId 任务Id
       * @return  是、否任务已完成
       */
      private boolean checkFinish(String taskId){
          DataTransferTask task=super.getById(taskId);
          if(task==null){
              log.warn("获取任务信息失败,taskId:{}",taskId);
              return false;
          }
  
          int currentTotal=task.getSuccess()+task.getFail()+task.getSkip();
          //成功+失败+跳过数量>=总数量,则表示处理完成
          if(currentTotal>=task.getTotal()){
              //只允许更新状态为'进行中'的任务为'完成'
              LambdaUpdateWrapper<DataTransferTask> updateWrapper=new LambdaUpdateWrapper<>();
              updateWrapper.eq(DataTransferTask::getId,taskId);
              updateWrapper.eq(DataTransferTask::getStatus,DataTransferTaskStatusEnum.PROGRESS);
  
              DataTransferTask updateTask=new DataTransferTask();
              updateTask.setStatus(DataTransferTaskStatusEnum.FINISH);
              updateTask.setEndTime(LocalDateTime.now());
              int row=baseMapper.update(updateTask,updateWrapper);
              log.info("任务状态更新影响行数,taskId:{},row:{}",taskId,row);
              if(row>0){
                  return true;
              }
          }
          return false;
      }
  }
  ```

- DAO接口DataTransferTaskMapper

  ```java
  public interface DataTransferTaskMapper extends BaseMapper<DataTransferTask> {
      /**
       * 递增成功数量
       * @param taskId 任务Id
       * @param value 递增值
       * @return 影响行数
       */
      int incrSuccess(@Param("taskId") String taskId, @Param("value")int value);
  
      /**
       * 递增失败数量
       * @param taskId 任务Id
       * @param value 递增值
       * @return 影响行数
       */
      int incrFail(@Param("taskId")String taskId,@Param("value")int value);
  
      /**
       * 递增跳过数量
       * @param taskId 任务Id
       * @param value 递增值
       * @return 影响行数
       */
      int incrSkip(@Param("taskId")String taskId,@Param("value")int value);
  
      /**
       * 获取可重启的任务列表
       * @param jobId 作业Id
       * @param taskStatus 任务状态
       * @param deleteTag 删除标识
       * @param subjectType 主体类型
       * @param status 事项状态列表
       * @return 任务列表
       */
      List<DataTransferTask> getRestartTasks(@Param("jobId") String jobId, @Param("taskStatus") DataTransferTaskStatusEnum taskStatus, @Param("deleteTag") DeletedEnum deleteTag, @Param("subjectType")DataTransferSubjectTypeEnum subjectType, @Param("status")List<DataTransferItemStatusEnum> status);
  }
  ```

  DataTransferTaskMapper.xml

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
  <mapper namespace="com.lightning.channel.ww.ods.mapper.self.DataTransferTaskMapper">
      <!-- 通用查询映射结果 -->
      <resultMap id="BaseResultMap" type="com.lightning.channel.ww.ods.entity.DataTransferTask">
          <id column="id" property="id" />
          <result column="create_user" property="createUser" />
          <result column="create_time" property="createTime" />
          <result column="update_user" property="updateUser" />
          <result column="update_time" property="updateTime" />
          <result column="tenant_id" property="tenantId" />
          <result column="version_number" property="versionNumber" />
          <result column="remark" property="remark" />
          <result column="job_name" property="jobName" />
          <result column="job_id" property="jobId" />
          <result column="executor_key" property="executorKey" />
          <result column="source_id" property="sourceId" />
          <result column="batch_size" property="batchSize" />
          <result column="batch_count" property="batchCount" />
          <result column="start_time" property="startTime" />
          <result column="end_time" property="endTime" />
          <result column="module_type" property="moduleType" />
          <result column="business_type" property="businessType" />
          <result column="status" property="status" />
          <result column="total" property="total" />
          <result column="success" property="success" />
          <result column="fail" property="fail" />
          <result column="skip" property="skip" />
      </resultMap>
  
      <update id="incrSuccess">
          update t_data_transfer_task set success=success+#{value,jdbcType=INTEGER} where id=#{taskId,jdbcType=VARCHAR}
      </update>
  
      <update id="incrFail">
          update t_data_transfer_task set fail=fail+#{value,jdbcType=INTEGER} where id=#{taskId,jdbcType=VARCHAR}
      </update>
  
      <update id="incrSkip">
          update t_data_transfer_task set skip=skip+#{value,jdbcType=INTEGER} where id=#{taskId,jdbcType=VARCHAR}
      </update>
  
      <select id="getRestartTasks" resultMap="BaseResultMap">
          select t.*
          from t_data_transfer_task t
          where t.job_id=#{jobId,jdbcType=VARCHAR}
          and t.status=#{taskStatus,jdbcType=VARCHAR}
          and exists(
              select 1 from t_data_transfer_item i where t.id=i.task_id
              and i.delete_tag=#{deleteTag,jdbcType=VARCHAR}
              and i.subject_type=#{subjectType,jdbcType=VARCHAR}
              and i.status in
              <foreach item="statusItem" index="index" collection="status" open="(" separator="," close=")">
                  #{statusItem,jdbcType=VARCHAR}
              </foreach>
          )
      </select>
  </mapper>
  ```

### 数据迁移事项表DataTransferItem

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@TableName("t_data_transfer_item")
public class DataTransferItem extends Model<DataTransferItem> {

    private static final long serialVersionUID = 1L;

    /**
     * 主键名称
     */
    @TableId(value = "id", type = IdType.ASSIGN_ID)
    private String id;

    /**
     * 删除标识
     */
    @TableField("delete_tag")
    private DeletedEnum deleteTag;

    /**
     * 创建人bip
     */
    @TableField("create_user")
    private String createUser;

    /**
     * 创建时间
     */
    @TableField("create_time")
    private LocalDateTime createTime;

    /**
     * 更新人bip
     */
    @TableField("update_user")
    private String updateUser;

    /**
     * 更新时间
     */
    @TableField("update_time")
    private LocalDateTime updateTime;

    /**
     * 租户编码
     */
    @TableField("tenant_id")
    private String tenantId;

    /**
     * 版本号
     */
    @TableField("version_number")
    private String versionNumber;

    /**
     * 备注
     */
    @TableField("remark")
    private String remark;

    /**
     * 任务Id
     */
    @TableField("task_id")
    private String taskId;

    /**
     * 主体Id.数据:业务Id,批次:批次序号
     */
    @TableField("subject_id")
    private String subjectId;

    /**
     * 主体类型
     */
    @TableField("subject_type")
    private DataTransferSubjectTypeEnum subjectType;

    /**
     * 状态
     */
    @TableField("status")
    private DataTransferItemStatusEnum status;

    /**
     * 数量
     */
    @TableField("number")
    private Integer number;

    /**
     * 阶段
     */
    @TableField("stage")
    private DataTransferItemStageEnum stage;

    /**
     * 追踪Id
     */
    @TableField("trace_id")
    private String traceId;

    /**
     * 错误信息
     */
    @TableField("error")
    private String error;

    @Override
    protected Serializable pkVal() {
        return this.id;
    }
}
```

- Service接口 DataTransferItemService

  ```java
  public interface DataTransferItemService extends IService<DataTransferItem> {
      /**
       * 批量新增(用于新增任务后,保存事项)
       * @param taskId 任务Id
       * @param subjectType 主体类型
       * @param stage 阶段
       * @param items 事项列表
       * @param userAccount 用户帐号
       */
      void batchAdd(String taskId, DataTransferSubjectTypeEnum subjectType, DataTransferItemStageEnum stage, List<DataTransferItem> items, String userAccount);
  
      /**
       * 获取任务下的重启事项数量
       * @param taskId 任务Id
       * @return 事项数量
       */
      int getRestartItemCount(String taskId);
  
      /**
       * 获取重启迁移事项列表
       * @param taskId 任务Id
       * @param batchSize 批次大小
       * @param batchNum 批次序号
       * @return 迁移事项列表
       */
      List<DataTransferItem> listRestartItems(String taskId,int batchSize,int batchNum);
  }
  ```

  服务接口实现类 DataTransferItemServiceImpl

  ```java
  @RequiredArgsConstructor(onConstructor_={@Autowired})
  @Slf4j
  @Service
  public class DataTransferItemServiceImpl extends ServiceImpl<DataTransferItemMapper, DataTransferItem> implements DataTransferItemService {
      /**
       * 批量新增(用于新增任务后,保存事项)
       * @param taskId 任务Id
       * @param subjectType 主体类型
       * @param stage 阶段
       * @param items 事项列表
       * @param userAccount 用户帐号
       */
      @Override
      public void batchAdd(String taskId, DataTransferSubjectTypeEnum subjectType, DataTransferItemStageEnum stage,List<DataTransferItem> items, String userAccount){
          if(CollectionUtils.isEmpty(items)){
              log.warn("保存的事项列表为空,退出处时.taskId:{},stage:{}",taskId,stage);
              return;
          }
  
          //保存前填充信息
          LocalDateTime now=LocalDateTime.now();
          items.stream().forEach(item->{
              item.setDeleteTag(DeletedEnum.NOT_DELETE);
              item.setCreateUser(userAccount);
              item.setCreateTime(now);
              item.setTaskId(taskId);
              item.setSubjectType(subjectType);
              item.setStage(stage);
          });
  
          super.saveBatch(items);
      }
  
      /**
       * 获取任务下的重启事项数量
       * @param taskId 任务Id
       * @return 事项数量
       */
      @Override
      public int getRestartItemCount(String taskId){
          LambdaQueryWrapper<DataTransferItem> queryWrapper = new LambdaQueryWrapper<>();
          queryWrapper.eq(DataTransferItem::getTaskId,taskId);
          queryWrapper.eq(DataTransferItem::getDeleteTag,DeletedEnum.NOT_DELETE);
          //主体类型为'数据'
          queryWrapper.eq(DataTransferItem::getSubjectType,DataTransferSubjectTypeEnum.DATA);
          //状态为'异常'
          queryWrapper.in(DataTransferItem::getStatus,Arrays.asList(DataTransferItemStatusEnum.BUSINESS_ERROR,DataTransferItemStatusEnum.SYSTEM_ERROR));
          return baseMapper.selectCount(queryWrapper);
      }
  
      /**
       * 获取重启迁移事项列表
       * @param taskId 任务Id
       * @param batchSize 批次大小
       * @param batchNum 批次序号
       * @return 迁移事项列表
       */
      @Override
      public List<DataTransferItem> listRestartItems(String taskId,int batchSize,int batchNum){
          LambdaQueryWrapper<DataTransferItem> queryWrapper = new LambdaQueryWrapper<>();
          queryWrapper.eq(DataTransferItem::getTaskId,taskId);
          queryWrapper.eq(DataTransferItem::getDeleteTag,DeletedEnum.NOT_DELETE);
          //只查询主体类型为'数据'
          queryWrapper.eq(DataTransferItem::getSubjectType,DataTransferSubjectTypeEnum.DATA);
          //只处理状态为'异常'的事项
          queryWrapper.in(DataTransferItem::getStatus,Arrays.asList(DataTransferItemStatusEnum.BUSINESS_ERROR,DataTransferItemStatusEnum.SYSTEM_ERROR));
  
          queryWrapper.orderByAsc(DataTransferItem::getCreateTime).orderByAsc(DataTransferItem::getSubjectId);
  
          IPage<DataTransferItem> page = new Page<>(batchNum, batchSize, false);
          IPage<DataTransferItem> pageResult= baseMapper.selectPage(page,queryWrapper);
          return pageResult.getRecords();
      }
  }
  ```

- DAO接口 

  ```java
  public interface DataTransferItemMapper extends BaseMapper<DataTransferItem> {
  
  }
  ```

### start源码分析

上面 `DataTransferConfig`配置类，应用启动时就创建了`DataTransferJobManager`对象注入Sping IOC容器归Spring管理，并添加了合同交付、停复工两个业务模块的数据迁移作业对象，来到控制层DataTransferController的start方法启动作业

```java
dataTransferJobManager.start(jobName,loginUser.getLoginName());
```

方法start

```java
    @Override
    public void start(String jobName,String userAccount) {
        //1.获取作业任务执行器映射
        DataTransferJob job=jobMap.get(jobName);
        if(job==null){
            log.warn("缺少作业信息,jobName:{}",jobName);
            throw new SystemRuntimeException("缺少作业信息");
        }
        //结构:Map<执行器键名标识,DataTransferExecutor>
        Map<String,DataTransferExecutor> executorMap=job.getExecutorMap();

        //2.每一次运行,则属不同的作业实例,生成作业id
        String jobId=idGenerator.nextId("");

        //3.根据执行器创建任务实例
        executorMap.forEach((executorKey,executor)->{
            //通过处理器,获取数据总量
            if(executor.getHandler()==null){
                log.warn("执行器缺少处理器,executorKey:{}",executor.getKey());
                throw new SystemRuntimeException("执行器缺少处理器");
            }
            // 查询任务迁移总数据量
            int total=executor.getHandler().readTotal();

            //创建启动模式任务实例.新启动任务时,来源任务Id为'空'
            DataTransferTaskInstance taskInstance=this.createTask(jobName,jobId,"",executor,DataTransferStartModeEnum.START,total,userAccount);
            if(taskInstance.getTotal()>0){
                //启动任务
                this.startTask(taskInstance);
            }
        });
    }
```

指定作业的每个执行器分别创建一个任务实例，写入到表 t_data_transfer_task

> 任务实例类DataTransferTaskInstance

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@ApiModel(value="DataTransferTaskInstance对象", description="数据迁移任务实例 对象")
public class DataTransferTaskInstance {
    /**
     * 作业名称
     */
    private String jobName;

    /**
     * 作业Id
     */
    private String jobId;

    /**
     * 执行器键名标识
     */
    private String executorKey;

    /**
     * 任务Id
     */
    private String taskId;

    /**
     * 来源任务Id
     */
    private String sourceId;

    /**
     * 数据处理批次大小
     */
    private int batchSize;

    /**
     * 数据处理批次总量
     */
    private int batchCount;

    /**
     * 总数据量
     */
    private int total;

    /**
     * 处理线程大小
     */
    private int threadSize;

    /**
     * 处理超时时间.单位:分钟
     */
    private int timeout;

    /**
     * 模块类型
     */
    private DataTransferModuleTypeEnum moduleType;

    /**
     * 业务类型
     */
    private DataTransferBusinessTypeEnum businessType;

    /**
     * 启动模式
     */
    private DataTransferStartModeEnum model;

}
```

任务数据量大于0，启动迁移任务

```java
if(taskInstance.getTotal()>0){
    //启动任务
    this.startTask(taskInstance);
}
```

方法this.startTask在上面的类DataTransferJobManagerImpl

```java
/**
     * 运行任务实例
     * @param taskInstance 任务实例
     */
    private void startTask(DataTransferTaskInstance taskInstance){
        //根据处理的批次总量,按批次发送MQ消息
        List<Message<DataTransferBatch>> messages = new ArrayList<>();

        for(int batchNum=1;batchNum<=taskInstance.getBatchCount();batchNum++){
            //创建批次信息
            DataTransferBatch batch=new DataTransferBatch();
            batch.setJobName(taskInstance.getJobName());
            batch.setExecutorKey(taskInstance.getExecutorKey());
            batch.setTaskId(taskInstance.getTaskId());
            batch.setSourceId(taskInstance.getSourceId());
            batch.setBatchSize(taskInstance.getBatchSize());
            batch.setBatchCount(taskInstance.getBatchCount());
            batch.setBatchNum(batchNum);
            batch.setTotal(taskInstance.getTotal());
            batch.setModel(taskInstance.getModel());
            //获取消息Key
            String msgKey=this.getMsgKey(batch.getExecutorKey(),batchNum);
            messages.add(MessageBuilder.withPayload(batch).setHeader("KEYS",msgKey).build());

            //判断是否需执行消息发送
            boolean isSend=this.isSendMessage(messages.size(),mqBatchMessageSize,batchNum,taskInstance.getBatchCount());
            if(isSend){
                //获取消息目的地
                String msgDest=MessageFormat.format("{0}:{1}",this.getMsgTopic(mqTopicPrefix,batch.getJobName()),this.getMsgTag(batch.getExecutorKey()));
                //发送消息
                SendResult sendRet=rocketMQTemplate.syncSend(msgDest,messages,10*1000);
                if(!SendStatus.SEND_OK.equals(sendRet.getSendStatus())){
                    log.warn("数据迁移消息发送失败,batch:{},sendRet:{}",batch,sendRet);
                    throw new SystemRuntimeException("消息发送失败");
                }
                //消息发送完毕,则清除
                messages.clear();
            }
        }
    }
```

数据迁移批次消息类`DataTransferBatch`

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@ApiModel(value="DataTransferBatch对象", description="数据迁移任务批次 对象")
public class DataTransferBatch implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * 作业名称
     */
    private String jobName;

    /**
     * 执行器键名标识
     */
    private String executorKey;

    /**
     * 任务Id
     */
    private String taskId;

    /**
     * 来源任务Id
     */
    private String sourceId;

    /**
     * 数据处理批次大小
     */
    private int batchSize;

    /**
     * 数据处理批次总量
     */
    private int batchCount;

    /**
     * 数据处理批次序号
     */
    private int batchNum;

    /**
     * 总数据量
     */
    private int total;

    /**
     * 启动模式
     */
    private DataTransferStartModeEnum model;
}
```

枚举类 DataTransferStartModeEnum

```java
public enum DataTransferStartModeEnum implements BaseEnum<Integer> {
    /**
     * 启动
     */
    START(1,"启动","dataTransferBatchStartHandler"),
    /**
     * 重启
     */
    RESTART(2, "重启","dataTransferBatchRestartHandler");
    private final Integer value;

    private final String displayName;

    private final String beanName;

    DataTransferStartModeEnum(Integer value, String displayName,String beanName) {
        this.value = value;
        this.displayName = displayName;
        this.beanName=beanName;
    }

    /**
     * 枚举value
     *
     * @return
     */
    @Override
    public Integer getValue() {
        return value;
    }

    /**
     * 枚举的显示名字
     *
     * @return
     */
    @Override
    public String getDisplayName() {
        return displayName;
    }

    /**
     * Bean名称
     *
     * @return
     */
    public String getBeanName() {
        return beanName;
    }
}
```

一个数据迁移批次生成一个MQ消息，然后MQ消息被批量发送到RocketMQ Broker，迁移任务启动成功，下面看迁移任务消息的消费

## 2、消费数据迁移批次消息

消费者在应用启动的时候就会注册到RocketMQ，看 `DataTransferJobManagerImpl` 的 方法initJobs有@PostConstruct注解，初始化自定义Job会给Job里的每个业务执行器都注册一个消费者到RocketMQ，看 `DataTransferJobManagerImpl` 的 方法initMqConsumer，执行单批次消息数据迁移的代码片段

```java
//执行单批次数据迁移
try {
    log.info("执行单批次数据迁移开始.jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
    DataTransferBatchHandler batchHandle = (DataTransferBatchHandler) applicationContext.getBean(batch.getModel().getBeanName());
    batchHandle.execute(batch);
    log.info("执行单批次数据迁移完成.jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
}catch (Exception e){
    log.error(MessageFormat.format("执行单批次数据迁移失败.jobName:{0},taskId:{1},batchNum:{2}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum()),e);
}
```

### 单批次消息处理接口DataTransferBatchHander

```java
public interface DataTransferBatchHandler {
    /**
     * 执行单批次数据迁移
     * @param batch 批次信息
     */
    void execute(DataTransferBatch batch);
}
```

接口实现类根据场景实现有两个

- 启动 DataTransferBatchStartHandlerImpl

  ```java
  @RequiredArgsConstructor(onConstructor_={@Autowired})
  @Slf4j
  @Component(value="dataTransferBatchStartHandler")
  public class DataTransferBatchStartHandlerImpl extends DataTransferBatchHandlerSuper implements DataTransferBatchHandler {
      @Autowired
      public void setJobManager(DataTransferJobManager jobManager) {
          this.jobManager = jobManager;
      }
  
      @Autowired
      public void setDataTransferTaskService(DataTransferTaskService dataTransferTaskService) {
          this.dataTransferTaskService = dataTransferTaskService;
      }
  
      @Autowired
      public void setDataTransferItemService(DataTransferItemService dataTransferItemService) {
          this.dataTransferItemService = dataTransferItemService;
      }
  
      @Autowired
      public void setIdGenerator(IdGenerator idGenerator) {
          this.idGenerator = idGenerator;
      }
  
      @Autowired
      public void setApplicationContext(ApplicationContext applicationContext) {
          this.applicationContext = applicationContext;
      }
  
      /**
       * 执行单批次数据迁移
       * @param batch 批次信息
       */
      @Override
      public void execute(DataTransferBatch batch){
          //处理器
          DataTransferHandler handler=null;
          //来源数据
          List<DataTransferSource> sources=null;
          //处理成功列表
          List<DataTransferResult> processSuccess;
  
          //1.获取处理器
          try{
              //获取执行器
              DataTransferExecutor executor=this.jobManager.getExecutor(batch.getJobName(),batch.getExecutorKey());
              handler=executor.getHandler();
          }catch (Exception e){
              //整批次处理失败
              this.applicationContext.getBean(this.getClass()).batchHandleFail(batch,"获取执行器", DataTransferItemStageEnum.BATCH_BEFORE,e);
              return;
          }
  
          //2.读取来源数据
          try{
              sources=handler.read(batch.getBatchSize(),batch.getBatchNum());
              //对来源数据没返回的数据,生成单批次的跳过迁移事项
              this.applicationContext.getBean(this.getClass()).batchHandleSkip(batch,sources.size(),DataTransferItemStageEnum.READ);
  
              if(CollectionUtils.isEmpty(sources)){
                  //来源数据为空,同不再执行后续的迁移操作
                  log.info("来源数据为空,不再执行后续阶段的迁移操作.taskId:{},batchNum:{}",batch.getTaskId(),batch.getBatchNum());
                  return;
              }
          }catch (Exception e){
              //整批次处理失败,则生成单批次的失败迁移事项
              this.applicationContext.getBean(this.getClass()).batchHandleFail(batch,"读取来源数据",DataTransferItemStageEnum.READ,e);
              return;
          }
  
          //3.数据处理
          processSuccess=this.executeBatchProcess(batch,handler,sources);
          if(CollectionUtils.isEmpty(processSuccess)){
              //没有成功处理结果,则不需执行后续的迁移操作,退出处理
              return;
          }
  
          //4.写入目标数据
          this.executeBatchWrite(batch,handler,processSuccess);
      }
  }
  ```

- 重启 DataTransferBatchRestartHandlerImpl

  ```java
  @RequiredArgsConstructor(onConstructor_={@Autowired})
  @Slf4j
  @Component(value="dataTransferBatchRestartHandler")
  public class DataTransferBatchRestartHandlerImpl extends DataTransferBatchHandlerSuper implements DataTransferBatchHandler {
      @Autowired
      public void setJobManager(DataTransferJobManager jobManager) {
          this.jobManager = jobManager;
      }
  
      @Autowired
      public void setDataTransferTaskService(DataTransferTaskService dataTransferTaskService) {
          this.dataTransferTaskService = dataTransferTaskService;
      }
  
      @Autowired
      public void setDataTransferItemService(DataTransferItemService dataTransferItemService) {
          this.dataTransferItemService = dataTransferItemService;
      }
  
      @Autowired
      public void setIdGenerator(IdGenerator idGenerator) {
          this.idGenerator = idGenerator;
      }
  
      @Autowired
      public void setApplicationContext(ApplicationContext applicationContext) {
          this.applicationContext = applicationContext;
      }
  
      /**
       * 执行单批次数据迁移
       * @param batch 批次信息
       */
      @Override
      public void execute(DataTransferBatch batch){
          //当前批次处理的事项列表
          List<DataTransferItem> batchItems=null;
          //处理器
          DataTransferHandler handler=null;
          //来源数据
          List<DataTransferSource> sources=null;
          //处理成功列表
          List<DataTransferResult> processSuccess;
  
          //1.通过被重启的任务Id,获取当前批次的处理事项(重启任务时,被sourceId为被重启的任务Id)
          batchItems=this.dataTransferItemService.listRestartItems(batch.getSourceId(),batch.getBatchSize(),batch.getBatchNum());
          //对获取的处理事项列表,生成单个批次主体的跳过处理事项
          this.applicationContext.getBean(this.getClass()).batchHandleSkip(batch,batchItems.size(),DataTransferItemStageEnum.BATCH_BEFORE);
          if(CollectionUtils.isEmpty(batchItems)){
              //批次处理事项为空,则不再执行后续迁移操作
              log.info("处理事项为空,不再执行后续阶段的迁移操作.taskId:{},batchNum:{}",batch.getTaskId(),batch.getBatchNum());
              return;
          }
  
          //2.获取处理器
          try{
              //获取执行器
              DataTransferExecutor executor=this.jobManager.getExecutor(batch.getJobName(),batch.getExecutorKey());
              handler=executor.getHandler();
          }catch (Exception e){
              //整批次处理失败(业务Id为:当前批次的事项主体Id)
              List<String> businessIds=batchItems.stream().map(DataTransferItem::getSubjectId).collect(Collectors.toList());
              this.applicationContext.getBean(this.getClass()).batchDataHandleFail(batch,"获取执行器",DataTransferItemStageEnum.BATCH_BEFORE,businessIds,e);
              return;
          }
  
          //3.读取来源数据
          try{
              //根据迁移事项主体Id,作为业务Id,读取来源数据
              List<String> businessIds=batchItems.stream().map(DataTransferItem::getSubjectId).collect(Collectors.toList());
              sources=handler.read(businessIds);
              //对来源数据不返回的数据,生成跳过处理迁移事项
              this.applicationContext.getBean(this.getClass()).sourceDataHandleSkip(batch,batchItems,sources);
  
              if(CollectionUtils.isEmpty(sources)){
                  //来源数据为空,则不再执行后续的操作
                  log.info("来源数据为空,不再执行后续阶段的迁移操作.taskId:{},batchNum:{}",batch.getTaskId(),batch.getBatchNum());
                  return;
              }
          }catch (Exception e){
              //整批次处理失败(业务Id为:当前批次的事项主体Id)
              List<String> businessIds=batchItems.stream().map(DataTransferItem::getSubjectId).collect(Collectors.toList());
              this.applicationContext.getBean(this.getClass()).batchDataHandleFail(batch,"读取来源数据",DataTransferItemStageEnum.READ,businessIds,e);
              return;
          }
  
          //3.数据处理
          processSuccess=this.applicationContext.getBean(this.getClass()).executeBatchProcess(batch,handler,sources);
          if(CollectionUtils.isEmpty(processSuccess)){
              //没有成功处理结果,则不需执行后续的迁移操作,退出处理
              return;
          }
  
          //4.写入目标数据
          this.applicationContext.getBean(this.getClass()).executeBatchWrite(batch,handler,processSuccess);
      }
  
      /**
       * 对来源数据不返回的数据,生成跳过处理迁移事项
       * @param batch 批次信息
       * @param batchItems 当前批次迁移事项列表
       * @param sources 来源数据列表
       */
      @Transactional(rollbackFor = RuntimeException.class,transactionManager = "selfTransactionManager")
      public void sourceDataHandleSkip(DataTransferBatch batch,List<DataTransferItem> batchItems,List<DataTransferSource> sources){
          //获取迁移事项主体Id,作为源业务Id
          List<String> batchItemSubjectIds=batchItems.stream().map(DataTransferItem::getSubjectId).collect(Collectors.toList());
          //获取数据来源的业务Id,作为处理结果业务Id
          List<String> sourceBusinessIds=sources.stream().map(DataTransferSource::getBusinessId).collect(Collectors.toList());
  
          //筛选出来源数据中缺少的业务Id,作为跳过处理的事项
          List<DataTransferItem> skipItems=this.getSkipItems(batchItemSubjectIds,sourceBusinessIds);
  
          //保存跳过处理事项.主体类型为'数据'
          if(CollectionUtils.isEmpty(skipItems)){
              //没有跳过处理的事项,则退出处理
              log.info("不存在跳过处理事项,退出处理.jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
              return;
          }
          this.dataTransferItemService.batchAdd(batch.getTaskId(),DataTransferSubjectTypeEnum.DATA,DataTransferItemStageEnum.READ,skipItems,"");
  
          //更新任务跳过处理数量,并检测任务完成状态
          boolean isFinish=this.dataTransferTaskService.incrItemCount(batch.getTaskId(),0,0,skipItems.size());
          if(isFinish){
              log.info("数据迁移任务已完成,jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
          }
      }
  }
  ```

### 单批次消息处理父类DataTransferBatchHandlerSuper

```java
@Slf4j
public class DataTransferBatchHandlerSuper {
    protected DataTransferJobManager jobManager;
    protected DataTransferTaskService dataTransferTaskService;
    protected DataTransferItemService dataTransferItemService;
    protected IdGenerator idGenerator;
    protected ApplicationContext applicationContext;

    /**
     * 执行单批次数据处理
     * @param batch 批次信息
     * @param handler 处理器
     * @param sources 来源数据列表
     * @return 处理成功结果列表.null:没有成功处理结果或失败,不需执行后续迁移操作
     */
    public List<DataTransferResult> executeBatchProcess(DataTransferBatch batch,DataTransferHandler handler,List<DataTransferSource> sources){
        //处理成功结果
        List<DataTransferResult> processSuccess=null;
        try {
            List<DataTransferResult> processResult = handler.process(sources);
            //对处理结果生成失败事项,并返回处理成功结果.对没返回的数据,生成跳过处理事项
            //==源业务Id列表为:来源数据业务Id
            List<String> businessIds=sources.stream().map(DataTransferSource::getBusinessId).collect(Collectors.toList());
            processSuccess=this.applicationContext.getBean(this.getClass()).resultDataHandle(batch,DataTransferItemStageEnum.PROGRESS,processResult,businessIds);
            if(CollectionUtils.isEmpty(processSuccess)){
                //成功处理结果为空,则表示不需执行后续的'写入'操作
                log.info("数据处理成功的结果为空,不再执行后续阶段的迁移操作.taskId:{},batchNum:{}",batch.getTaskId(),batch.getBatchNum());
                return null;
            }
        }catch (Exception e){
            //整批次的数据处理失败,根据业务Id列表生成多个数据主体的失败事项(业务Id为:来源数据业务Id)
            List<String> businessIds=sources.stream().map(DataTransferSource::getBusinessId).collect(Collectors.toList());
            this.applicationContext.getBean(this.getClass()).batchDataHandleFail(batch,"数据处理",DataTransferItemStageEnum.PROGRESS,businessIds,e);
            return null;
        }
        return processSuccess;
    }

    /**
     * 执行单版本目标数据写入
     * @param batch 批次信息
     * @param handler 处理器
     * @param processSuccess 处理成功结果列表
     */
    public void executeBatchWrite(DataTransferBatch batch,DataTransferHandler handler,List<DataTransferResult> processSuccess){
        try {
            List<Object> targets = processSuccess.stream().map(DataTransferResult::getData).collect(Collectors.toList());
            List<DataTransferResult> writeResult = handler.write(targets);
            //对处理结果生成失败事项.对没返回的数据,生成跳过处理事项
            //==源业务Id列表为:处理成功结果业务Id
            List<String> businessIds=processSuccess.stream().map(DataTransferResult::getBusinessId).collect(Collectors.toList());
            this.applicationContext.getBean(this.getClass()).resultDataHandle(batch,DataTransferItemStageEnum.WRITE,writeResult,businessIds);
        }catch (Exception e){
            //整批次的数据处理失败,根据业务Id列表生成多个数据主体的失败事项(业务Id为:成功处理结果业务Id)
            List<String> businessIds=processSuccess.stream().map(DataTransferResult::getBusinessId).collect(Collectors.toList());
            // 获取bean的方式执行方法，避免了@Transactional等注解的失效问题
            this.applicationContext.getBean(this.getClass()).batchDataHandleFail(batch,"写入目标数据",DataTransferItemStageEnum.WRITE,businessIds,e);
        }
    }

    /**
     * 输出异常信息,并返回迁移失败状态
     * @param exception 异常信息
     * @param traceId 追踪Id
     * @param action 动作描述
     * @param jobName 作业名称
     * @param executorKey 执行器键名
     * @param batchNum 批次序号
     * @return 迁移失败状态
     */
    protected DataTransferItemStatusEnum handleException(Exception exception,String traceId,String action,String jobName,String executorKey,Integer batchNum){
        //日志提示
        String logTips=null;
        DataTransferItemStatusEnum status=null;
        if(exception instanceof SystemRuntimeException){
            //业务异常
            status=DataTransferItemStatusEnum.BUSINESS_ERROR;
            logTips= MessageFormat.format("数据迁移处理结束,{0}时出现业务异常.traceId:{1},jobName:{2},executorKey:{3},batchNum:{4}"
                    ,action,traceId,jobName,executorKey,batchNum);
        }else{
            //系统异常
            status=DataTransferItemStatusEnum.SYSTEM_ERROR;
            logTips=MessageFormat.format("数据迁移处理结束,{0}时出现系统异常.traceId:{1},jobName:{2},executorKey:{3},batchNum:{4}"
                    ,action,traceId,jobName,executorKey,batchNum);
        }
        log.error(logTips,exception);
        return status;
    }

    /**
     * 生成单个批次主体的跳过处理事项
     * @param batch 批次信息
     * @param realTotal 实际返回数量
     * @param stage 阶段
     */
    @Transactional(rollbackFor = RuntimeException.class,transactionManager = "selfTransactionManager")
    public void batchHandleSkip(DataTransferBatch batch, int realTotal,DataTransferItemStageEnum stage){
        //1.计算跳过处理数量
        //获取当前批次应处理数量
        int batchItemCount=this.getBatchItemCount(batch.getBatchSize(),batch.getBatchNum(),batch.getBatchCount(),batch.getTotal());
        //跳过数量=当前批次应处理数量-实际返回数量
        int skipCount=batchItemCount-realTotal;

        //2.生成迁移跳过处理事项
        if(skipCount<=0){
            //没有跳过处理的事项,则退出处理
            log.info("不存在跳过处理事项,退出处理.jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
            return;
        }

        //3.保存跳过处理事项
        DataTransferItem skipItem=new DataTransferItem();
        //主体Id为'批次序号'
        skipItem.setSubjectId(String.valueOf(batch.getBatchNum()));
        skipItem.setStatus(DataTransferItemStatusEnum.SKIP);
        //跳过处理数量
        skipItem.setNumber(skipCount);
        skipItem.setTraceId("");
        skipItem.setError("");
        this.dataTransferItemService.batchAdd(batch.getTaskId(),DataTransferSubjectTypeEnum.BATCH,stage,Arrays.asList(skipItem),"");

        //4.递增跳过处理事项状态数量,并检测任务完成状态
        boolean isFinish=this.dataTransferTaskService.incrItemCount(batch.getTaskId(),0,0,skipCount);
        if(isFinish){
            log.info("数据迁移任务已完成,jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
        }
    }

    /**
     * 获取批次实际事项数量
     * @param batchSize 批次大小
     * @param batchNum 批次序号.从1开始
     * @param batchCount 批次数量
     * @param total 总数据量
     * @return 事项数量
     */
    protected int getBatchItemCount(Integer batchSize,Integer batchNum,Integer batchCount,Integer total){
        if(batchNum<batchCount){
            //不是最后一批,事项数量为批次大小
            return batchSize;
        }
        //最后一批,事项数量=总数据量-批次大小*(批次序号-1)
        return total-batchSize*(batchNum-1);
    }

    /**
     * 整批次处理失败,生成单个批次主体的失败事项
     * @param batch 批次信息
     * @param action 动作描述
     * @param stage 阶段
     * @param exception 异常
     */
    @Transactional(rollbackFor = RuntimeException.class,transactionManager = "selfTransactionManager")
    public void batchHandleFail(DataTransferBatch batch, String action, DataTransferItemStageEnum stage, Exception exception){
        //生成追踪Id
        String traceId=idGenerator.nextId("");

        //1.处理异常信息,并返回迁移失败状态
        DataTransferItemStatusEnum status=this.handleException(exception,traceId,action,batch.getJobName(),batch.getExecutorKey(),batch.getBatchNum());

        //2.创建失败迁移事项
        //获取批次实际失败数量
        int batchFailCount=this.getBatchItemCount(batch.getBatchSize(),batch.getBatchNum(),batch.getBatchCount(),batch.getTotal());

        DataTransferItem item=new DataTransferItem();
        item.setSubjectId(String.valueOf(batch.getBatchNum()));
        item.setStatus(status);
        item.setNumber(batchFailCount);
        item.setTraceId(traceId);
        item.setError(exception.getMessage());
        this.dataTransferItemService.batchAdd(batch.getTaskId(),DataTransferSubjectTypeEnum.BATCH,stage, Arrays.asList(item),"");

        //3.递增失败数量并检测任务完成状态
        boolean isFinish=this.dataTransferTaskService.incrItemCount(batch.getTaskId(),0,batchFailCount,0);
        if(isFinish){
            log.info("数据迁移任务已完成,jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
        }
    }

    /**
     * 整批次的数据处理失败,根据业务Id列表生成多个数据主体的失败事项
     * @param batch 批次信息
     * @param action 动作描述
     * @param stage 阶段
     * @param businessIds 业务Id列表
     * @param exception 异常
     */
    @Transactional(rollbackFor = RuntimeException.class,transactionManager = "selfTransactionManager")
    public void batchDataHandleFail(DataTransferBatch batch, String action, DataTransferItemStageEnum stage, List<String> businessIds, Exception exception){
        //生成追踪Id
        String traceId=idGenerator.nextId("");

        //处理异常信息,并返回迁移失败状态
        DataTransferItemStatusEnum status=this.handleException(exception,traceId,action,batch.getJobName(),batch.getExecutorKey(),batch.getBatchNum());

        //1.根据业务Id,生成整批次的迁移事项
        List<DataTransferItem> items=businessIds.stream().map(businessId->{

            DataTransferItem item=new DataTransferItem();
            //处理结果失败时,主体Id为业务Id
            item.setSubjectId(businessId);
            item.setStatus(status);
            //一个业务Id,数量值为:1
            item.setNumber(1);
            item.setTraceId(traceId);
            item.setError(exception.getMessage());
            return item;
        }).collect(Collectors.toList());
        //保存迁移事项.主体类型为'数据'
        this.dataTransferItemService.batchAdd(batch.getTaskId(),DataTransferSubjectTypeEnum.DATA,stage,items,"");

        //2.递增任务失败数量,并检测完成状态
        //检测任务完成状态
        boolean isFinish=this.dataTransferTaskService.incrItemCount(batch.getTaskId(),0,items.size(),0);
        if(isFinish){
            log.info("数据迁移任务已完成,jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
        }
    }

    /**
     * 根据处理结果与源业务Id对比,生成跳过事项、失败事项,并返回成功结果列表
     * @param batch 批次信息
     * @param stage 阶段
     * @param results 处理结果列表
     * @param businessIds 源业务Id列表
     * @return 处理成功结果列表
     */
    @Transactional(rollbackFor = RuntimeException.class,transactionManager = "selfTransactionManager")
    public List<DataTransferResult> resultDataHandle(DataTransferBatch batch,DataTransferItemStageEnum stage,List<DataTransferResult> results,List<String> businessIds){
        //处埋成功结果
        List<DataTransferResult> successResults=new ArrayList<>();
        //失败事项
        List<DataTransferItem> failItems= new ArrayList<>();

        //1.获取处理成功结果、失败事项
        for(int i=0;i<results.size();i++){
            DataTransferResult result=results.get(i);
            switch (result.getStatus()){
                case SUCCESS:
                    //处理成功结果
                    successResults.add(result);
                    continue;
                case SKIP:
                    //跳过
                    throw new SystemRuntimeException(MessageFormat.format("发现无效状态处理结果.状态:{}",result.getStatus()));
            }

            //生成处理失败事项
            DataTransferItem item=new DataTransferItem();
            //==主体Id为业务Id
            item.setSubjectId(result.getBusinessId());
            item.setStatus(result.getStatus());
            item.setTraceId(result.getTraceId());
            item.setError(result.getError());
            //==一个结果对应数量为:1
            item.setNumber(1);
            failItems.add(item);
        }

        //2.生成跳过处理事项
        //获取处理结果的业务Id列表
        List<String> resultBusinessIds=results.stream().map(DataTransferResult::getBusinessId).collect(Collectors.toList());
        List<DataTransferItem> skipItems=this.getSkipItems(businessIds,resultBusinessIds);

        //3.保存迁移失败事项、跳过事项.主体类型为'数据'
        List<DataTransferItem> addItems=new ArrayList<>();
        addItems.addAll(failItems);
        addItems.addAll(skipItems);

        //5.更新迁移事项
        if(CollectionUtils.isNotEmpty(addItems)){
            this.dataTransferItemService.batchAdd(batch.getTaskId(),DataTransferSubjectTypeEnum.DATA,stage,addItems,"");
        }

        //6.更新任务失败、跳过处理数量,以及完成状态
        //获取迁移失败事项数量
        int failCount=failItems.size();
        //获取跳过处理数量
        int skipCount=skipItems.size();
        //获取迁移功事项数量
        int successCount=0;
        if(DataTransferItemStageEnum.WRITE.equals(stage)){
            //只有在写入目标数据阶段为成功,才递增成功数量
            successCount=successResults.size();
        }
        //递增事项状态数量,并检测任务完成状态
        boolean isFinish=this.dataTransferTaskService.incrItemCount(batch.getTaskId(),successCount,failCount,skipCount);
        if(isFinish){
            log.info("数据迁移任务已完成,jobName:{},taskId:{},batchNum:{}",batch.getJobName(),batch.getTaskId(),batch.getBatchNum());
        }
        return successResults;
    }

    /**
     * 根据源业务Id与返回结果业务Id对比,获得跳过的处理事项
     * @param fromBusinessIds 源业务Id
     * @param resultBusinessIds 结果业务Id
     * @return 跳过事项列表
     */
    protected List<DataTransferItem> getSkipItems(List<String> fromBusinessIds,List<String> resultBusinessIds){
        //获取跳过处理的业务Id列表
        List<String> skipBusinessIds=new ArrayList<>();
        skipBusinessIds.addAll(fromBusinessIds);
        skipBusinessIds.removeAll(resultBusinessIds);
        //生成跳过事项
        List<DataTransferItem> skipItems=skipBusinessIds.stream().map(skipBusinessId->{
            DataTransferItem item=new DataTransferItem();
            item.setSubjectId(skipBusinessId);
            item.setStatus(DataTransferItemStatusEnum.SKIP);
            item.setTraceId("");
            item.setError("");
            //一个业务Id对应数量为:1
            item.setNumber(1);
            return item;
        }).collect(Collectors.toList());
        return skipItems;
    }
}
```

### 停复工模块迁移处理器

实现DataTransferHandler接口

> 停复工申请处理器 WorkResumeDtStopPlanApplyHandler

```java
/**
 * <p>
 * 停复工_停工计划申请数据迁移处理器 实现类
 * </p>
 *
 * @author liJY
 * @Date 2023-06-20
 */
@RequiredArgsConstructor(onConstructor_={@Autowired})
@Slf4j
@Component
public class WorkResumeDtStopPlanApplyHandler extends DataTransferHandlerSuper implements DataTransferHandler<StopOrResumeWork, WorkResumeApplyResult> {
    private DataTransferBusinessProperties dataTransferBusinessProperties;
    private WorkflowInstanceService workflowInstanceService;
    private SysOrgService sysOrgService;
    private FlowBillService flowBillService;
    private OldProjectService projectService;
    private StopOrResumeWorkService stopOrResumeWorkService;
    private WorkResumeApplyService workResumeApplyService;
    private BasicDataTreeService basicDataTreeService;
    @Value("${data-transfer.business.work-resume-stop-plan-apply.overview-length:5000}")
    private Integer lengthLimit;

    @Autowired
    public void setIdGenerator(IdGenerator idGenerator) {
        this.idGenerator = idGenerator;
    }

    @Autowired
    public void setDataTransferBusinessProperties(DataTransferBusinessProperties dataTransferBusinessProperties) {
        this.dataTransferBusinessProperties = dataTransferBusinessProperties;
    }

    @Autowired
    public void setWorkflowInstanceService(WorkflowInstanceService workflowInstanceService) {
        this.workflowInstanceService = workflowInstanceService;
    }

    @Autowired
    public void setSysOrgService(SysOrgService sysOrgService) {
        this.sysOrgService = sysOrgService;
    }

    @Autowired
    public void setFlowBillService(FlowBillService flowBillService) {
        this.flowBillService = flowBillService;
    }

    @Autowired
    public void setProjectService(OldProjectService projectService) {
        this.projectService = projectService;
    }

    @Autowired
    public void setStopOrResumeWorkService(StopOrResumeWorkService stopOrResumeWorkService) {
        this.stopOrResumeWorkService = stopOrResumeWorkService;
    }

    @Autowired
    public void setWorkResumeApplyService(WorkResumeApplyService workResumeApplyService) {
        this.workResumeApplyService = workResumeApplyService;
    }

    @Autowired
    public void setBasicDataTreeService(BasicDataTreeService basicDataTreeService) {
        this.basicDataTreeService = basicDataTreeService;
    }

    /**
     * 读取来源处理事项总数(用于首次启动任务场景)
     * @return 总数量
     */
    @Override
    public int readTotal(){
        //使用配置的条件查询
        return this.stopOrResumeWorkService.getTotal(dataTransferBusinessProperties.getWorkResumeStopPlanApply().getQuery());
    }

    /**
     * 读取来源数据(用于首次启动任务场景)
     * @param batchSize 批次大小
     * @param batchNum 批次序号
     * @return 来源数据列表.返回数量比批次数量少的,则为跳过处理,空:没有待处理数据
     */
    @Override
    public List<DataTransferSource<StopOrResumeWork>> read(int batchSize, int batchNum){
        //使用配置的条件查询
        List<StopOrResumeWork> list= this.stopOrResumeWorkService.pageQuery(batchSize,batchNum, dataTransferBusinessProperties.getWorkResumeStopPlanApply().getQuery());
        if(CollectionUtils.isEmpty(list)){
            log.info("停复工_停工计划申请来源数据为空,退出处理");
            return new ArrayList<>();
        }

        //定义返回结果
        List<DataTransferSource<StopOrResumeWork>> result=list.stream().map(apply -> {
            DataTransferSource<StopOrResumeWork> source=new DataTransferSource<>();
            //定义业务Id
            source.setBusinessId(apply.getWorkID());
            source.setData(apply);
            return source;
        }).collect(Collectors.toList());
        return result;
    }

    /**
     * 读取来源数据(用于重启任务场景)
     * @param businessIds 业务Id列表
     * @return 来源数据列表.返回数量比入参数量少的,则为跳过处理,空:没有待处理数据
     */
    @Override
    public List<DataTransferSource<StopOrResumeWork>> read(List<String> businessIds){
        if(CollectionUtils.isEmpty(businessIds)){
            log.info("停复工_停工计划申请业务id为空,退出处理");
            return new ArrayList<>();
        }

        //固定使用id查询
        List<StopOrResumeWork> list=this.stopOrResumeWorkService.findByIds(businessIds);
        if(CollectionUtils.isEmpty(list)){
            log.info("停复工_停工计划申请来源数据为空,退出处理");
            return new ArrayList<>();
        }
        //定义返回结果
        List<DataTransferSource<StopOrResumeWork>> result=list.stream().map(apply -> {
            DataTransferSource<StopOrResumeWork> source=new DataTransferSource<>();
            //定义业务Id
            source.setBusinessId(apply.getWorkID());
            source.setData(apply);
            return source;
        }).collect(Collectors.toList());
        return result;
    }

    /**
     * 数据处理
     * @param sources 来源数据列表
     * @return 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
     */
    @Override
    public List<DataTransferResult<WorkResumeApplyResult>> process(List<DataTransferSource<StopOrResumeWork>> sources){
        if(CollectionUtils.isEmpty(sources)){
            log.info("停复工_停工计划申请来源数据为空,退出处埋");
            return new ArrayList<>();
        }

        //1.对已处理的数据进行过滤,防止重复迁移
        //获取来源数据的申请Id
        List<String> sourceApplyIds=sources.stream().map(DataTransferSource::getBusinessId).collect(Collectors.toList());
        //获取目标数据已存在的申请.结构:Map<申请Id,WorkResumeApply>
        Map<String,WorkResumeApply> existsApplys=this.workResumeApplyService.getByIds(sourceApplyIds);
        //过滤得到未迁移的来源数据
        List<DataTransferSource<StopOrResumeWork>> useSources=sources.stream().filter(source->{
            if(existsApplys.containsKey(source.getBusinessId())){
                return false;
            }
            return true;
        }).collect(Collectors.toList());
        if(CollectionUtils.isEmpty(useSources)){
            log.info("停复工_停工计划申请,待迁移的来源数据为空,退出处理");
            return new ArrayList<>();
        }

        //2.获取流程信息(流程信息会与申请信息一同写入,不需判断重复迁移).结构:Map<业务Id,WorkflowInstance>
        List<String> useApplyIds=useSources.stream().map(DataTransferSource::getBusinessId).collect(Collectors.toList());
        Map<String, WorkflowInstance> workflowMap=this.workflowInstanceService.getByBusinessIds(useApplyIds);

        //3.获取项目信息.结构:Map<项目Id,Project>
        List<String> projectIds=useSources.stream().map(source->{
            return source.getData().getProjectID();
        }).collect(Collectors.toList());
        Map<String, OldProject> projectMap=this.projectService.getByIds(projectIds);

        //4.获取旧平台项目所属区域信息(目的是获取区域编码、区域名称).结构:Map<区域Id,SysOrg>
        List<String> oldRegionIds=new ArrayList<>();
        projectMap.forEach((projectId,project)->{
            oldRegionIds.add(project.getRegionID());
        });
        Map<String, SysOrg> oldRegionMap=this.sysOrgService.getByOrgIds(oldRegionIds);

        //5.获取所属新平台区域信息(目的是获取新平台区域Id).结构:Map<区域编码,BasicDataTree>
        List<String> regionCodes=new ArrayList<>();
        oldRegionMap.forEach((regionId,SysOrg)->{
            regionCodes.add(SysOrg.getOrgCode());
        });
        Map<String, BasicDataTree> newRegionMap=this.basicDataTreeService.findByMdgId(regionCodes);

        //6.构建处理结果
        List<DataTransferResult<WorkResumeApplyResult>> result=useSources.stream().map(source->{
            //获取申请信息
            StopOrResumeWork apply = source.getData();
            //获取流程信息
            WorkflowInstance workflowInstance = workflowMap.get(apply.getWorkID());
            //获取项目信息
            OldProject project=projectMap.get(apply.getProjectID());
            //获取旧平台区域信息
            SysOrg oldRegion = null;
            if(project!=null){
                oldRegion=oldRegionMap.get(project.getRegionID());
            }
            //获取新平台区域信息
            BasicDataTree newRegion=null;
            if(oldRegion!=null){
                newRegion=newRegionMap.get(oldRegion.getOrgCode());
            }

            //构建处理结果
            DataTransferResult<WorkResumeApplyResult> processResult=buildProcessResult(apply,workflowInstance,project,oldRegion,newRegion);
            return  processResult;
        }).collect(Collectors.toList());
        return result;
    }

    /**
     * 构建处理结果
     * @param apply 申请信息
     * @param workflowInstance 流程实例信息
     * @param project 项目信息
     * @param oldRegion 旧平台区域信息
     * @param newRegion 新平台区域信息
     * @return 处理结果
     */
    private DataTransferResult<WorkResumeApplyResult> buildProcessResult(StopOrResumeWork apply, WorkflowInstance workflowInstance
            , OldProject project, SysOrg oldRegion,BasicDataTree newRegion){
        DataTransferResult<WorkResumeApplyResult> processResult=new DataTransferResult<>();
        //申请Id
        String applyId=apply.getWorkID();
        try{
            if (workflowInstance == null) {
                throw new SystemRuntimeException("缺少流程信息");
            }
            if(project==null){
                throw new SystemRuntimeException("缺少项目信息");
            }
            if (oldRegion == null) {
                throw new SystemRuntimeException("缺少旧平台区域信息");
            }
            if(newRegion==null){
                throw new SystemRuntimeException("缺少新平台区域信息,"+oldRegion.getOrgCode()+","+oldRegion.getOrgName());
            }
            if(StringUtils.length(apply.getOverview()) >= lengthLimit){
                throw new SystemRuntimeException("overview字段字符串长度超过"+lengthLimit);
            }

            //构建申请处理结果
            WorkResumeApplyResult applyResult = new WorkResumeApplyResult();
            applyResult.setApply(WorkResumeUtils.buildStopPlanApplyByOld(apply, workflowInstance, project,oldRegion,newRegion));
            applyResult.setFlow(WorkResumeUtils.buildFlowBillByOld(project, workflowInstance));

            //定义处理成功结果
            processResult.setData(applyResult);
            processResult.setBusinessId(applyId);
            processResult.setError("");
            processResult.setTraceId("");
            processResult.setStatus(DataTransferItemStatusEnum.SUCCESS);
        }catch (Exception e){
            //生成追踪Id,并对异常日志输出
            String traceId=idGenerator.nextId("");
            //输出异常日志,并返回处理状态
            DataTransferItemStatusEnum status=super.handleException(e,traceId,"停复工_停工计划申请","数据处理");

            //定义处理失败结果
            processResult.setData(null);
            processResult.setBusinessId(applyId);
            processResult.setError(e.getMessage());
            processResult.setTraceId(traceId);
            processResult.setStatus(status);
        }
        return processResult;
    }

    /**
     * 写入目标数据
     * @param targets 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
     */
    @Override
    @Transactional(rollbackFor = RuntimeException.class,transactionManager = "plan-rebuildTransactionManager")
    public List<DataTransferResult> write(List<WorkResumeApplyResult> targets){
        //使用事务写入,使申请与流程信息同时整批写入成功或失败

        //1.保存申请信息
        //获取需保存的申请信息
        List<WorkResumeApply> applys=targets.stream().map(WorkResumeApplyResult::getApply).collect(Collectors.toList());
        this.workResumeApplyService.batchSave(applys);

        //2.保存流程信息
        List<FlowBill> flowBills=targets.stream().map(WorkResumeApplyResult::getFlow).collect(Collectors.toList());
        this.flowBillService.batchSave(flowBills);

        //3.对整批次的目标写入数据,构建整批成功写入结果
        //获取写入目标数据业务Id
        List<String> businessIds=targets.stream().map(applyResult->{
            return applyResult.getApply().getId();
        }).collect(Collectors.toList());
        List<DataTransferResult> result=buildBatchWriteResultSuccess(businessIds);
        return result;
    }
}
```

来源数据类 StopOrResumeWork

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@TableName("StopOrResumeWork")
public class StopOrResumeWork extends Model<StopOrResumeWork> {

    private static final long serialVersionUID = 1L;
    /**
     * 申请ID
     */
    @TableField("WorkID")
    private String workID;
    /**
     * 运营项目Id
     */
    @TableField("ProjectID")
    private String projectID;
    /**
     * 创建时间
     */
    @TableField("InsertOn")
    private LocalDateTime insertOn;
    /**
     * 更新时间(审批状态更新时间)
     */
    @TableField("UpdateOn")
    private LocalDateTime updateOn;
    /**
     * 更新用户
     */
    @TableField("UpdateBy")
    private String updateBy;
    /**
     * 情况说明
     */
    @TableField("Overview")
    private String overview;

    /**
     * 创建人姓名(流程创建人姓名)
     */
    @TableField("CreatedUserName")
    private String createdUserName;
}
```

目标数据类

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@ApiModel(value="WorkResumeApplyResult对象", description="停复工申请信息数据迁移处理结果 对象")
public class WorkResumeApplyResult {
    /**
     * 申请信息
     */
    private WorkResumeApply apply;
    /**
     * 流程信息
     */
    private FlowBill flow;
}
```



> 复工确认申请处理器 WorkResumeDtConfirmApplyHandler

```java
/**
 * <p>
 * 停复工_复工确认申请数据迁移处理器 实现类
 * </p>
 *
 * @author liJY
 * @Date 2023-06-20
 */
@RequiredArgsConstructor(onConstructor_={@Autowired})
@Slf4j
@Component
public class WorkResumeDtConfirmApplyHandler extends DataTransferHandlerSuper implements DataTransferHandler<ResumeWorkConfirmInfo, WorkResumeApplyResult> {
    private DataTransferBusinessProperties dataTransferBusinessProperties;
    private WorkflowInstanceService workflowInstanceService;
    private SysOrgService sysOrgService;
    private FlowBillService flowBillService;
    private OldProjectService projectService;
    private ResumeWorkConfirmInfoService resumeWorkConfirmInfoService;
    private WorkResumeApplyService workResumeApplyService;
    private BasicDataTreeService basicDataTreeService;
    @Value("${data-transfer.business.work-resume-stop-plan-apply.overview-length:5000}")
    private Integer lengthLimit;

    @Autowired
    public void setIdGenerator(IdGenerator idGenerator) {
        this.idGenerator = idGenerator;
    }

    @Autowired
    public void setDataTransferBusinessProperties(DataTransferBusinessProperties dataTransferBusinessProperties) {
        this.dataTransferBusinessProperties = dataTransferBusinessProperties;
    }

    @Autowired
    public void setWorkflowInstanceService(WorkflowInstanceService workflowInstanceService) {
        this.workflowInstanceService = workflowInstanceService;
    }

    @Autowired
    public void setSysOrgService(SysOrgService sysOrgService) {
        this.sysOrgService = sysOrgService;
    }

    @Autowired
    public void setFlowBillService(FlowBillService flowBillService) {
        this.flowBillService = flowBillService;
    }
    
    @Autowired
    public void setProjectService(OldProjectService projectService) {
        this.projectService = projectService;
    }

    @Autowired
    public void setResumeWorkConfirmInfoService(ResumeWorkConfirmInfoService resumeWorkConfirmInfoService) {
        this.resumeWorkConfirmInfoService = resumeWorkConfirmInfoService;
    }

    @Autowired
    public void setWorkResumeApplyService(WorkResumeApplyService workResumeApplyService) {
        this.workResumeApplyService = workResumeApplyService;
    }

    @Autowired
    public void setBasicDataTreeService(BasicDataTreeService basicDataTreeService) {
        this.basicDataTreeService = basicDataTreeService;
    }

    /**
     * 读取来源处理事项总数(用于首次启动任务场景)
     * @return 总数量
     */
    @Override
    public int readTotal(){
        //使用配置的条件查询
        return this.resumeWorkConfirmInfoService.getTotal(dataTransferBusinessProperties.getWorkResumeConfirmApply().getQuery());
    }

    /**
     * 读取来源数据(用于首次启动任务场景)
     * @param batchSize 批次大小
     * @param batchNum 批次序号
     * @return 来源数据列表.返回数量比批次数量少的,则为跳过处理,空:没有待处理数据
     */
    @Override
    public List<DataTransferSource<ResumeWorkConfirmInfo>> read(int batchSize, int batchNum){
        //使用配置的条件查询
        List<ResumeWorkConfirmInfo> list= this.resumeWorkConfirmInfoService.pageQuery(batchSize,batchNum, dataTransferBusinessProperties.getWorkResumeConfirmApply().getQuery());
        if(CollectionUtils.isEmpty(list)){
            log.info("停复工_复工确认申请来源数据为空,退出处理");
            return new ArrayList<>();
        }

        //定义返回结果
        List<DataTransferSource<ResumeWorkConfirmInfo>> result=list.stream().map(apply -> {
            DataTransferSource<ResumeWorkConfirmInfo> source=new DataTransferSource<>();
            //定义业务Id
            source.setBusinessId(apply.getId());
            source.setData(apply);
            return source;
        }).collect(Collectors.toList());
        return result;
    }

    /**
     * 读取来源数据(用于重启任务场景)
     * @param businessIds 业务Id列表
     * @return 来源数据列表.返回数量比入参数量少的,则为跳过处理,空:没有待处理数据
     */
    @Override
    public List<DataTransferSource<ResumeWorkConfirmInfo>> read(List<String> businessIds){
        if(CollectionUtils.isEmpty(businessIds)){
            log.info("停复工_复工确认申请业务id为空,退出处理");
            return new ArrayList<>();
        }

        //固定使用id查询
        List<ResumeWorkConfirmInfo> list=this.resumeWorkConfirmInfoService.findByIds(businessIds);
        if(CollectionUtils.isEmpty(list)){
            log.info("停复工_复工确认申请来源数据为空,退出处理");
            return new ArrayList<>();
        }
        //定义返回结果
        List<DataTransferSource<ResumeWorkConfirmInfo>> result=list.stream().map(apply -> {
            DataTransferSource<ResumeWorkConfirmInfo> source=new DataTransferSource<>();
            //定义业务Id
            source.setBusinessId(apply.getId());
            source.setData(apply);
            return source;
        }).collect(Collectors.toList());
        return result;
    }

    /**
     * 数据处理
     * @param sources 来源数据列表
     * @return 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
     */
    @Override
    public List<DataTransferResult<WorkResumeApplyResult>> process(List<DataTransferSource<ResumeWorkConfirmInfo>> sources){
        if(CollectionUtils.isEmpty(sources)){
            log.info("停复工_复工确认申请来源数据为空,退出处埋");
            return new ArrayList<>();
        }

        //1.对已处理的数据进行过滤,防止重复迁移
        //获取来源数据的申请Id
        List<String> sourceApplyIds=sources.stream().map(DataTransferSource::getBusinessId).collect(Collectors.toList());
        //获取目标数据已存在的申请.结构:Map<申请Id,WorkResumeApply>
        Map<String, WorkResumeApply> existsApplys=this.workResumeApplyService.getByIds(sourceApplyIds);
        //过滤得到未迁移的来源数据
        List<DataTransferSource<ResumeWorkConfirmInfo>> useSources=sources.stream().filter(source->{
            if(existsApplys.containsKey(source.getBusinessId())){
                return false;
            }
            return true;
        }).collect(Collectors.toList());
        if(CollectionUtils.isEmpty(useSources)){
            log.info("停复工_复工确认申请,待迁移的来源数据为空,退出处理");
            return new ArrayList<>();
        }

        //2.获取流程信息(流程信息会与申请信息一同写入,不需判断重复迁移).结构:Map<业务Id,WorkflowInstance>
        List<String> useApplyIds=useSources.stream().map(DataTransferSource::getBusinessId).collect(Collectors.toList());
        Map<String, WorkflowInstance> workflowMap=this.workflowInstanceService.getByBusinessIds(useApplyIds);
    
        //3.获取项目信息.结构:Map<项目Id,Project>
        List<String> projectIds=useSources.stream().map(source->{
            return source.getData().getProjectID();
        }).collect(Collectors.toList());
        Map<String, OldProject> projectMap=this.projectService.getByIds(projectIds);
        
        //4.获取旧平台项目所属区域信息(目的区域编码).结构:Map<区域Id,SysOrg>
        List<String> oldRegionIds=new ArrayList<>();
        projectMap.forEach((projectId,project)->{
            oldRegionIds.add(project.getRegionID());
        });
        Map<String, SysOrg> oldRegionMap=this.sysOrgService.getByOrgIds(oldRegionIds);

        //5.获取所属新平台区域信息(目的是获取新平台区域Id).结构:Map<区域编码,BasicDataTree>
        List<String> regionCodes=new ArrayList<>();
        oldRegionMap.forEach((regionId,SysOrg)->{
            regionCodes.add(SysOrg.getOrgCode());
        });
        Map<String, BasicDataTree> newRegionMap=this.basicDataTreeService.findByMdgId(regionCodes);

        //6.构建处理结果
        List<DataTransferResult<WorkResumeApplyResult>> result=useSources.stream().map(source->{
            //获取申请信息
            ResumeWorkConfirmInfo apply = source.getData();
            //获取流程信息
            WorkflowInstance workflowInstance = workflowMap.get(apply.getId());
            //获取项目信息
            OldProject project=projectMap.get(apply.getProjectID());
            //获取旧平台区域信息
            SysOrg oldRegion = null;
            if(project!=null){
                oldRegion=oldRegionMap.get(project.getRegionID());
            }
            //获取新平台区域信息
            BasicDataTree newRegion=null;
            if(oldRegion!=null){
                newRegion=newRegionMap.get(oldRegion.getOrgCode());
            }

            //构建处理结果
            DataTransferResult<WorkResumeApplyResult> processResult=buildProcessResult(apply,workflowInstance,oldRegion,newRegion);
            return  processResult;
        }).collect(Collectors.toList());
        return result;
    }

    /**
     * 构建处理结果
     * @param apply 申请信息
     * @param workflowInstance 流程实例信息
     * @param oldRegion 旧平台区域信息
     * @param newRegion 新平台区域信息
     * @return 处理结果
     */
    private DataTransferResult<WorkResumeApplyResult> buildProcessResult(ResumeWorkConfirmInfo apply,WorkflowInstance workflowInstance
            ,SysOrg oldRegion,BasicDataTree newRegion){
        DataTransferResult<WorkResumeApplyResult> processResult=new DataTransferResult<>();
        //申请Id
        String applyId=apply.getId();
        try{
            if (workflowInstance == null) {
                throw new SystemRuntimeException("缺少流程信息");
            }
            if (oldRegion == null) {
                throw new SystemRuntimeException("缺少旧平台区域信息");
            }
            if(newRegion==null){
                throw new SystemRuntimeException("缺少新平台区域信息,"+oldRegion.getOrgCode()+","+oldRegion.getOrgName());
            }
            if(StringUtils.length(apply.getOverview()) >= lengthLimit){
                throw new SystemRuntimeException("overview字段字符串长度超过"+lengthLimit);
            }
            //构建申请处理结果
            WorkResumeApplyResult applyResult = new WorkResumeApplyResult();
            applyResult.setApply(WorkResumeUtils.buildConfirmApplyByOld(apply, workflowInstance, oldRegion,newRegion));
            applyResult.setFlow(WorkResumeUtils.buildFlowBillByOld(new OldProject().setProjectID(apply.getProjectID()).setProjectName(apply.getProjectName()), workflowInstance));

            //定义处理成功结果
            processResult.setData(applyResult);
            processResult.setBusinessId(applyId);
            processResult.setError("");
            processResult.setTraceId("");
            processResult.setStatus(DataTransferItemStatusEnum.SUCCESS);
        }catch (Exception e){
            //生成追踪Id,并对异常日志输出
            String traceId=idGenerator.nextId("");
            //输出异常日志,并返回处理状态
            DataTransferItemStatusEnum status=super.handleException(e,traceId,"停复工_复工确认申请","数据处理");

            //定义处理失败结果
            processResult.setData(null);
            processResult.setBusinessId(applyId);
            processResult.setError(e.getMessage());
            processResult.setTraceId(traceId);
            processResult.setStatus(status);
        }
        return processResult;
    }

    /**
     * 写入目标数据
     * @param targets 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
     */
    @Override
    @Transactional(rollbackFor = RuntimeException.class,transactionManager = "plan-rebuildTransactionManager")
    public List<DataTransferResult> write(List<WorkResumeApplyResult> targets){
        //使用事务写入,使申请与流程信息同时整批写入成功或失败

        //1.保存申请信息
        //获取需保存的申请信息
        List<WorkResumeApply> applys=targets.stream().map(WorkResumeApplyResult::getApply).collect(Collectors.toList());
        this.workResumeApplyService.batchSave(applys);

        //2.保存流程信息
        List<FlowBill> flowBills=targets.stream().map(WorkResumeApplyResult::getFlow).collect(Collectors.toList());
        this.flowBillService.batchSave(flowBills);

        //3.对整批次的目标写入数据,构建整批成功写入结果
        //获取写入目标数据业务Id
        List<String> businessIds=targets.stream().map(applyResult->{
            return applyResult.getApply().getId();
        }).collect(Collectors.toList());
        List<DataTransferResult> result=buildBatchWriteResultSuccess(businessIds);
        return result;
    }
}
```

> WorkResumeDtStopPlanDetailHandler

```java
/**
 * 停复工申请明细迁移处理器，实现类
 * @author xiejinwei02
 * @date 2023/8/10 16:39
 */
@RequiredArgsConstructor(onConstructor_={@Autowired})
@Slf4j
@Component
public class WorkResumeDtStopPlanDetailHandler extends DataTransferHandlerSuper implements DataTransferHandler<BuildingStopOrResumeDTO, BuildingStopOrResumeDTO> {
	@Resource
	private DataTransferBusinessProperties dataTransferBusinessProperties;
	@Resource
	private BuildingStopOrResumeService buildingStopOrResumeService;
	@Resource
	private OldProjectService projectService;
	@Resource
	private SysOrgService sysOrgService;
	@Resource
	private WorkResumeRecordService workResumeRecordService;
	@Resource
	private WorkResumePlanService workResumePlanService;
	@Resource
	private BasicDataTreeService basicDataTreeService;
	@Resource
	@Qualifier("plan-rebuildTransactionManager")
	private PlatformTransactionManager planRebuildTransactionManager;
	private List<StopTypeEnum> springWinterBreakList = Arrays.asList(StopTypeEnum.SPRING_BREAK,StopTypeEnum.WINTER_BREAK);
	private final int longStopYear = 9990;
	
	@Autowired
	public void setIdGenerator(IdGenerator idGenerator) {
		this.idGenerator = idGenerator;
	}
	
	@Override
	public int readTotal() {
		return buildingStopOrResumeService.getTotal(dataTransferBusinessProperties.getWorkResumeStopPlanDetail().getQuery());
	}
	
	/**
	 * 读取来源数据(用于首次启动任务场景)
	 * @param batchSize 批次大小
	 * @param batchNum 批次序号
	 * @return
	 */
	@Override
	public List<DataTransferSource<BuildingStopOrResumeDTO>> read(int batchSize, int batchNum) {
		List<BuildingStopOrResumeDTO> dtoList = buildingStopOrResumeService.pageQuery(batchSize,batchNum,dataTransferBusinessProperties.getWorkResumeStopPlanDetail().getQuery());
		if(CollectionUtils.isEmpty(dtoList)){
			log.info("停复工申请明细来源数据为空,退出处理");
			return new ArrayList<>();
		}
		//定义返回结果
		List<DataTransferSource<BuildingStopOrResumeDTO>> result = dtoList.stream().map(dto->{
			DataTransferSource<BuildingStopOrResumeDTO> source = new DataTransferSource<>();
			source.setBusinessId(dto.getDetailId()); // 明细id
			source.setData(dto);
			return source;
		}).collect(Collectors.toList());
		return result;
	}
	
	/**
	 * 读取来源数据(用于重启任务场景)
	 * @param businessIds 业务Id列表
	 * @return
	 */
	@Override
	public List<DataTransferSource<BuildingStopOrResumeDTO>> read(List<String> businessIds) {
		if(CollectionUtils.isEmpty(businessIds)){
			log.info("停复工申请明细业务id为空,退出处理");
			return new ArrayList<>();
		}
		//固定使用id查询
		List<BuildingStopOrResumeDTO> dtoList = buildingStopOrResumeService.findByIds(businessIds);
		if(CollectionUtils.isEmpty(dtoList)){
			log.info("停复工申请明细来源数据为空,退出处理");
			return new ArrayList<>();
		}
		//定义返回结果
		List<DataTransferSource<BuildingStopOrResumeDTO>> result = dtoList.stream().map(dto->{
			DataTransferSource<BuildingStopOrResumeDTO> source = new DataTransferSource<>();
			source.setBusinessId(dto.getDetailId()); // 明细id
			source.setData(dto);
			return source;
		}).collect(Collectors.toList());
		return result;
	}
	
	/**
	 * 数据处理
	 * @param sources
	 * @return
	 */
	@Override
	public List<DataTransferResult<BuildingStopOrResumeDTO>> process(List<DataTransferSource<BuildingStopOrResumeDTO>> sources) {
		// 1、排除已迁移的数据
		List<String> sourceDetailIds = sources.stream().map(s->s.getBusinessId()).collect(Collectors.toList());
		List<String> exitsDetailIds = workResumeRecordService.listObjs(new LambdaQueryWrapper<WorkResumeRecord>().select(WorkResumeRecord::getId).in(WorkResumeRecord::getId,sourceDetailIds),v->v.toString());
		// 过滤得到未迁移的来源数据
		List<DataTransferSource<BuildingStopOrResumeDTO>> useSources = sources.stream().filter(s-> !exitsDetailIds.contains(s.getBusinessId())).collect(Collectors.toList());
		if(org.apache.commons.collections4.CollectionUtils.isEmpty(useSources)){
			log.info("停复工申请明细,待迁移的来源数据为空,退出处理");
			return new ArrayList<>();
		}
		
		// 2、查询项目信息，结构:Map<项目Id,Project>
		List<String> projectIds = useSources.stream().map(s->s.getData().getProjectId()).collect(Collectors.toList());
		Map<String, OldProject> projectMap = projectService.getByIds(projectIds);
		// 3、查询项目所属区域信息，结构:Map<区域Id,SysOrg>
		List<String> regionIds=new ArrayList<>();
		projectMap.forEach((projectId,project)->{
			regionIds.add(project.getRegionID());
		});
		Map<String, SysOrg> oldRegionMap = sysOrgService.getByOrgIds(regionIds);
		// 4、获取所属新平台区域信息(目的是获取新平台区域Id).结构:Map<区域编码,BasicDataTree>
		List<String> regionCodes=new ArrayList<>();
		oldRegionMap.forEach((regionId,SysOrg)->{
			regionCodes.add(SysOrg.getOrgCode());
		});
		Map<String, BasicDataTree> newRegionMap=this.basicDataTreeService.findByMdgId(regionCodes);
		
		// 5、构建处理结果
		List<DataTransferResult<BuildingStopOrResumeDTO>> result = useSources.stream().map(s ->{
			BuildingStopOrResumeDTO detail = s.getData();
			OldProject project = projectMap.get(detail.getProjectId());
			SysOrg oldRegion = Optional.ofNullable(project).map(p->oldRegionMap.get(p.getRegionID())).orElse(null);
			BasicDataTree newRegion = Optional.ofNullable(oldRegion).map(r -> newRegionMap.get(r.getOrgCode())).orElse(null);
			
			DataTransferResult<BuildingStopOrResumeDTO> processResult = buildProcessResult(detail, project, oldRegion, newRegion);
			return processResult;
		}).collect(Collectors.toList());
		
		return result;
	}
	
	/**
	 * 写入目标数据
	 * @param targets 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
	 */
	@Override
	public List<DataTransferResult> write(List<BuildingStopOrResumeDTO> targets) {
		// 查询原因说明字典值
		List<StopReasonDTO> stopReasons = buildingStopOrResumeService.listStopReason();
		List<WorkResumeRecord> resumeRecordList = new ArrayList<>();
		List<WorkResumePlan> resumePlanInsertList = new ArrayList<>();
		List<WorkResumePlan> resumePlanUpdateList = new ArrayList<>();
		LocalDate now = LocalDate.now();
		
		for (BuildingStopOrResumeDTO target : targets) {
			WorkResumeRecord resumeRecord = buildWorkRecordByOld(target,stopReasons);
			resumeRecordList.add(resumeRecord);
			// 查询insert集合楼栋的停工计划是否已存在
			WorkResumePlan resumePlan = resumePlanInsertList.stream().filter(p->Objects.equals(p.getPlanBuildingCode(),target.getPlanBuildingCode()) && Objects.equals(p.getStopPlanTime(),target.getStopStartOn()))
					.findFirst().orElseGet(()->{
						// 查询数据库楼栋的停工计划是否已存在
						return workResumePlanService.getOne(new LambdaQueryWrapper<WorkResumePlan>().select(WorkResumePlan::getId,WorkResumePlan::getResumePlanTime)
								.eq(WorkResumePlan::getPlanBuildingCode, target.getPlanBuildingCode()).eq(WorkResumePlan::getStopPlanTime, target.getStopStartOn())
								.eq(WorkResumePlan::getInvalid, 0).notIn(WorkResumePlan::getStopType, springWinterBreakList).last("limit 1"));
					});
			if(Objects.isNull(resumePlan)){
				resumePlan = buildResumePlanByOld(target,now);
				resumePlanInsertList.add(resumePlan);
				resumeRecord.setResumePlanId(resumePlan.getId());
				resumePlan.setReason(resumeRecord.getReason());
				resumePlan.setReasonType(resumeRecord.getReasonType());
				// 停工类型，判断是否与春节/冬歇期停工计划重叠
				OverlapPutVO putVO = new OverlapPutVO();
				putVO.setProjectId(resumePlan.getProjectId());
				putVO.setPlanBuildingCode(resumePlan.getPlanBuildingCode());
				putVO.setAdjustStopPlanTime(resumePlan.getStopPlanTime());
				putVO.setAdjustResumePlanTime(resumePlan.getResumePlanTime());
				List<OverlapRetVO> overlapList = workResumePlanService.overlap(putVO);
				if(!CollectionUtils.isEmpty(overlapList)){
					if(Objects.equals(overlapList.get(0).getStopType(),StopTypeEnum.SPRING_BREAK)){
						resumePlan.setStopType(StopTypeEnum.INCLUDE_SPRING_BREAK);
					}else{
						resumePlan.setStopType(StopTypeEnum.INCLUDE_WINTER_BREAK);
					}
					for (OverlapRetVO retVO : overlapList) {
						WorkResumePlan invalidPlan = new WorkResumePlan();
						invalidPlan.setId(retVO.getId());
						invalidPlan.setInvalid(InvalidEnum.OVERLAP_INVALID);
						resumePlanUpdateList.add(invalidPlan);
					}
				}
			}else{
				resumeRecord.setResumePlanId(resumePlan.getId());
				// 判断是否停工申请，是则不更新，因为存在的停工计划是复工申请，已包含停复工的计划停工复工时间和实际停工复工时间
				if(0==target.getStopOrResumeChose()){
					// 复工申请,更新计划复工时间，实际复工时间
					updateResumePlanByOld(target,resumePlan);
					resumePlanUpdateList.add(resumePlan);
					// 停工类型，判断是否与春节/冬歇期停工计划重叠
					OverlapPutVO putVO = new OverlapPutVO();
					putVO.setProjectId(target.getProjectId());
					putVO.setPlanBuildingCode(target.getPlanBuildingCode());
					putVO.setAdjustStopPlanTime(target.getStopStartOn());
					putVO.setAdjustResumePlanTime(Optional.ofNullable(target.getApplyResumeOn()).orElse(target.getStopEndOn()));
					List<OverlapRetVO> overlapList = workResumePlanService.overlap(putVO);
					if(!CollectionUtils.isEmpty(overlapList)){
						if(Objects.equals(overlapList.get(0).getStopType(),StopTypeEnum.SPRING_BREAK)){
							resumePlan.setStopType(StopTypeEnum.INCLUDE_SPRING_BREAK);
						}else{
							resumePlan.setStopType(StopTypeEnum.INCLUDE_WINTER_BREAK);
						}
						for (OverlapRetVO retVO : overlapList) {
							WorkResumePlan invalidPlan = new WorkResumePlan();
							invalidPlan.setId(retVO.getId());
							invalidPlan.setInvalid(InvalidEnum.OVERLAP_INVALID);
							resumePlanUpdateList.add(invalidPlan);
						}
					}
				}
			}
		}
		// 同一事务控制保存
		DefaultTransactionDefinition def1 = new DefaultTransactionDefinition();
		def1.setName("WorkResumeDtStopPlanDetailHandler");
		def1.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
		TransactionStatus status = planRebuildTransactionManager.getTransaction(def1);
		try{
			workResumeRecordService.saveBatch(resumeRecordList);
			workResumePlanService.saveBatch(resumePlanInsertList);
			workResumePlanService.updateBatchById(resumePlanUpdateList);
		}catch (Exception e){
			planRebuildTransactionManager.rollback(status);
			throw e;
		}
		planRebuildTransactionManager.commit(status);
		//3.对整批次的目标写入数据,构建整批成功写入结果
		//获取写入目标数据业务Id
		List<String> businessIds=targets.stream().map(t->t.getDetailId()).collect(Collectors.toList());
		List<DataTransferResult> result=buildBatchWriteResultSuccess(businessIds);
		return result;
	}
	
	private DataTransferResult<BuildingStopOrResumeDTO> buildProcessResult(BuildingStopOrResumeDTO detail,OldProject project, SysOrg oldRegion,BasicDataTree newRegion){
		DataTransferResult<BuildingStopOrResumeDTO> processResult = new DataTransferResult<>();
		String detailId = detail.getDetailId();
		try {
			if(project==null){
				throw new SystemRuntimeException("缺少项目信息");
			}
			if (oldRegion == null) {
				throw new SystemRuntimeException("缺少旧平台区域信息");
			}
			if(newRegion==null){
				throw new SystemRuntimeException("缺少新平台区域信息,"+oldRegion.getOrgCode()+","+oldRegion.getOrgName());
			}
			if(Objects.isNull(detail.getStopStartOn())){
				throw new SystemRuntimeException("计划停工时间为空");
			}
			if(Objects.isNull(detail.getStopEndOn())){
				throw new SystemRuntimeException("计划复工时间为空");
			}
			if(!detail.getStopStartOn().isBefore(Optional.ofNullable(detail.getApplyResumeOn()).orElse(detail.getStopEndOn()))){
				throw new SystemRuntimeException("计划停工时间必须小于计划复工时间");
			}
			
			detail.setAreaId(newRegion.getWwId());
			detail.setAreaCode(oldRegion.getOrgCode());
			detail.setAreaName(oldRegion.getOrgName());
			detail.setProjectName(project.getProjectName());
			
			//定义处理成功结果
			processResult.setData(detail);
			processResult.setBusinessId(detailId);
			processResult.setError("");
			processResult.setTraceId("");
			processResult.setStatus(DataTransferItemStatusEnum.SUCCESS);
		}catch (Exception e) {
			//生成追踪Id,并对异常日志输出
			String traceId=idGenerator.nextId("");
			//输出异常日志,并返回处理状态
			DataTransferItemStatusEnum status=super.handleException(e,traceId,"停复工_申请明细","数据处理");
			
			//定义处理失败结果
			processResult.setData(null);
			processResult.setBusinessId(detailId);
			processResult.setError(e.getMessage());
			processResult.setTraceId(traceId);
			processResult.setStatus(status);
		}
		return processResult;
	}
	
	private WorkResumeRecord buildWorkRecordByOld(BuildingStopOrResumeDTO target,List<StopReasonDTO> stopReasons){
		WorkResumeRecord resumeRecord = new WorkResumeRecord();
		resumeRecord.setId(target.getDetailId());
		resumeRecord.setApplyId(target.getWorkId());
		resumeRecord.setPlanBuildingCode(target.getPlanBuildingCode());
		resumeRecord.setAdjustStopPlanTime(target.getStopStartOn());
		resumeRecord.setAdjustResumePlanTime(target.getStopEndOn());
		resumeRecord.setOperation(1);
		if(Objects.nonNull(target.getSubWorkflow())){
			stopReasons.stream().filter(s-> Objects.equals(s.getAttr1(),target.getSubWorkflow())).findFirst().ifPresent(s->{
				resumeRecord.setReason(s.getNarrationCn());
				resumeRecord.setReasonType(1==target.getStopOrResumeChose()?1:2);
			});
		}
		resumeRecord.setCreateTime(target.getInsertOn());
		resumeRecord.setCreateUser(OldPlanUtils.getUserAccount(target.getInsertBy()));
		resumeRecord.setTenantId(TenantEnum.DEFAULT.getValue());
		return resumeRecord;
	}
	
	private WorkResumePlan buildResumePlanByOld(BuildingStopOrResumeDTO target,LocalDate now){
		WorkResumePlan resumePlan = new WorkResumePlan();
		resumePlan.setId(IdWorker.getIdStr());
		resumePlan.setAreaId(target.getAreaId());
		resumePlan.setAreaCode(target.getAreaCode());
		resumePlan.setAreaName(OldPlanUtils.getRegionName(target.getAreaName()));
		resumePlan.setProjectId(target.getProjectId());
		resumePlan.setProjectName(target.getProjectName());
		resumePlan.setPlanBuildingCode(target.getPlanBuildingCode());
		resumePlan.setPlanBuildingName(target.getPlanBuildingName());
		resumePlan.setStopPlanTime(target.getStopStartOn());
		resumePlan.setResumePlanTime(Optional.ofNullable(target.getApplyResumeOn()).orElse(target.getStopEndOn()));
		// 计划停工时间<=迁移当天，实际停工时间=计划停工时间
		if(now.compareTo(target.getStopStartOn())>=0){
			resumePlan.setStopRealTime(target.getStopStartOn());
		}
		resumePlan.setResumeRealTime(target.getResumeOn());
		resumePlan.setStopType(StopTypeEnum.OTHER_BREAK);
		// 计划状态
		resumePlan.setStatus(ResumeStatusEnum.PASS_RESUME_PLAN);
		if(Objects.nonNull(resumePlan.getResumeRealTime())){
			// 6-已复工，有实际复工时间
			resumePlan.setStatus(ResumeStatusEnum.RESUMED);
		}else if(Objects.nonNull(resumePlan.getStopRealTime())){
			// 4-停工：有实际停工时间，没有实际复工时间
			resumePlan.setStatus(ResumeStatusEnum.STOPPED);
		}
		resumePlan.setCreateTime(target.getInsertOn());
		resumePlan.setCreateUser(OldPlanUtils.getUserAccount(target.getInsertBy()));
		resumePlan.setUpdateTime(target.getUpdateOn());
		resumePlan.setUpdateUser(OldPlanUtils.getUserAccount(target.getUpdateBy()));
		resumePlan.setTenantId(TenantEnum.DEFAULT.getValue());
		// 停复工记录增加一个“是否长期停工”字段，根据计划复工时间是否为9990-01-01来初始化
		if(Objects.isNull(resumePlan.getResumeRealTime()) && Objects.nonNull(resumePlan.getResumePlanTime()) && resumePlan.getResumePlanTime().getYear() == longStopYear){
			resumePlan.setLongStop(YesOrNoEnum.YES);
		}
		return resumePlan;
	}
	
	private void updateResumePlanByOld(BuildingStopOrResumeDTO target,WorkResumePlan resumePlan){
		resumePlan.setResumePlanTime(target.getApplyResumeOn());
		resumePlan.setResumeRealTime(target.getResumeOn());
		resumePlan.setStopType(StopTypeEnum.OTHER_BREAK);
		// 计划状态
		resumePlan.setStatus(ResumeStatusEnum.PASS_RESUME_PLAN);
		if(Objects.nonNull(resumePlan.getResumeRealTime())){
			// 6-已复工，有实际复工时间
			resumePlan.setStatus(ResumeStatusEnum.RESUMED);
		}
		resumePlan.setUpdateTime(target.getUpdateOn());
		resumePlan.setUpdateUser(OldPlanUtils.getUserAccount(target.getUpdateBy()));
		// 停复工记录增加一个“是否长期停工”字段，根据计划复工时间是否为9990-01-01来初始化
		if(Objects.isNull(resumePlan.getResumeRealTime()) && Objects.nonNull(resumePlan.getResumePlanTime()) && resumePlan.getResumePlanTime().getYear() == longStopYear){
			resumePlan.setLongStop(YesOrNoEnum.YES);
		}
	}
}
```

> WorkResumeDtConfirmDetailHandler

```java
/**
 * 复工确认明细迁移处理器，实现类
 * @author xiejinwei02
 * @date 2023/8/11 8:48
 */
@RequiredArgsConstructor(onConstructor_={@Autowired})
@Slf4j
@Component
public class WorkResumeDtConfirmDetailHandler extends DataTransferHandlerSuper implements DataTransferHandler<ResumeWorkConfirmDetail, WorkResumeRecord> {
	@Resource
	private DataTransferBusinessProperties dataTransferBusinessProperties;
	@Resource
	ResumeWorkConfirmDetailService resumeWorkConfirmDetailService;
	@Resource
	private WorkResumeRecordService workResumeRecordService;
	
	@Autowired
	public void setIdGenerator(IdGenerator idGenerator) {
		this.idGenerator = idGenerator;
	}
	
	/**
	 * 读取来源处理事项总数(用于首次启动任务场景)
	 * @return 总数量
	 */
	@Override
	public int readTotal() {
		return resumeWorkConfirmDetailService.getTotal(dataTransferBusinessProperties.getWorkResumeConfirmDetail().getQuery());
	}
	
	/**
	 * 读取来源数据(用于首次启动任务场景)
	 * @param batchSize 批次大小
	 * @param batchNum 批次序号
	 * @return 来源数据列表.返回数量比批次数量少的,则为跳过处理,空:没有待处理数据
	 */
	@Override
	public List<DataTransferSource<ResumeWorkConfirmDetail>> read(int batchSize, int batchNum) {
		List<ResumeWorkConfirmDetail> details = resumeWorkConfirmDetailService.pageQuery(batchSize,batchNum,dataTransferBusinessProperties.getWorkResumeConfirmDetail().getQuery());
		if(CollectionUtils.isEmpty(details)){
			log.info("复工确认明细来源数据为空,退出处理");
			return new ArrayList<>();
		}
		//定义返回结果
		List<DataTransferSource<ResumeWorkConfirmDetail>> result = details.stream().map(d->{
			DataTransferSource<ResumeWorkConfirmDetail> source = new DataTransferSource<>();
			source.setBusinessId(d.getId());
			source.setData(d);
			return source;
		}).collect(Collectors.toList());
		return result;
	}
	
	/**
	 * 读取来源数据(用于重启任务场景)
	 * @param businessIds 业务Id列表
	 * @return 来源数据列表.返回数量比入参数量少的,则为跳过处理,空:没有待处理数据
	 */
	@Override
	public List<DataTransferSource<ResumeWorkConfirmDetail>> read(List<String> businessIds) {
		if(CollectionUtils.isEmpty(businessIds)){
			log.info("复工确认明细业务id为空,退出处理");
			return new ArrayList<>();
		}
		//固定使用id查询
		List<ResumeWorkConfirmDetail> details = resumeWorkConfirmDetailService.findByIds(businessIds);
		if(CollectionUtils.isEmpty(details)){
			log.info("复工确认明细来源数据为空,退出处理");
			return new ArrayList<>();
		}
		//定义返回结果
		List<DataTransferSource<ResumeWorkConfirmDetail>> result = details.stream().map(d->{
			DataTransferSource<ResumeWorkConfirmDetail> source = new DataTransferSource<>();
			source.setBusinessId(d.getId());
			source.setData(d);
			return source;
		}).collect(Collectors.toList());
		return result;
	}
	
	/**
	 * 数据处理
	 * @param sources 来源数据列表
	 * @return 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
	 */
	@Override
	public List<DataTransferResult<WorkResumeRecord>> process(List<DataTransferSource<ResumeWorkConfirmDetail>> sources) {
		// 1、排除已迁移的数据
		List<String> sourceDetailIds = sources.stream().map(s->s.getBusinessId()).collect(Collectors.toList());
		List<String> exitsDetailIds = workResumeRecordService.listObjs(new LambdaQueryWrapper<WorkResumeRecord>().select(WorkResumeRecord::getId).in(WorkResumeRecord::getId,sourceDetailIds), v->v.toString());
		// 过滤得到未迁移的来源数据
		List<DataTransferSource<ResumeWorkConfirmDetail>> useSources = sources.stream().filter(s-> !exitsDetailIds.contains(s.getBusinessId())).collect(Collectors.toList());
		if(org.apache.commons.collections4.CollectionUtils.isEmpty(useSources)){
			log.info("复工确认明细,待迁移的来源数据为空,退出处理");
			return new ArrayList<>();
		}
		
		// 2、构建处理结果
		List<DataTransferResult<WorkResumeRecord>> result = new ArrayList<>();
		useSources.stream().forEach(s->{
			ResumeWorkConfirmDetail detail = s.getData();
			// 查询停复工申请明细是否已存在
			WorkResumeRecord one = workResumeRecordService.getOne(new LambdaQueryWrapper<WorkResumeRecord>().select(WorkResumeRecord::getId, WorkResumeRecord::getResumePlanId)
					.eq(WorkResumeRecord::getId, detail.getBuildingStopOrResumeId()));
			result.add(buildProcessResult(detail, Optional.ofNullable(one).map(r->r.getResumePlanId()).orElse(null)));
		});
		
		return result;
	}
	
	/**
	 * 写入目标数据
	 * @param targets 目标数据列表.返回数量比入参数量少的,则为跳过处理.空:没有待处理数据
	 */
	@Override
	@Transactional(rollbackFor = RuntimeException.class,transactionManager = "plan-rebuildTransactionManager")
	public List<DataTransferResult> write(List<WorkResumeRecord> targets) {
		workResumeRecordService.saveBatch(targets);
		
		//2.对整批次的目标写入数据,构建整批成功写入结果
		//获取写入目标数据业务Id
		List<String> businessIds=targets.stream().map(t->t.getId()).collect(Collectors.toList());
		List<DataTransferResult> result=buildBatchWriteResultSuccess(businessIds);
		return result;
	}
	
	private DataTransferResult<WorkResumeRecord> buildProcessResult(ResumeWorkConfirmDetail detail,String resumePlanId){
		DataTransferResult<WorkResumeRecord> processResult = new DataTransferResult<>();
		String detailId = detail.getId();
		try{
			if(Objects.isNull(resumePlanId)){
				throw new SystemRuntimeException("找不到对应的停复工申请明细,workResumeRecordId:"+detail.getBuildingStopOrResumeId());
			}
			
			WorkResumeRecord resumeRecord = new WorkResumeRecord();
			resumeRecord.setId(detailId);
			resumeRecord.setApplyId(detail.getResumeWorkConfirmId());
			resumeRecord.setResumePlanId(resumePlanId);
			resumeRecord.setPlanBuildingCode(detail.getPlanBuildingCode());
			resumeRecord.setOperation(ResumeOperationEnum.RESUME_CONFIRM.getValue());
			resumeRecord.setCreateTime(detail.getInsertOn());
			resumeRecord.setCreateUser(OldPlanUtils.getUserAccount(detail.getInsertBy()));
			resumeRecord.setTenantId(TenantEnum.DEFAULT.getValue());
			
			//定义处理成功结果
			processResult.setData(resumeRecord);
			processResult.setBusinessId(detailId);
			processResult.setError("");
			processResult.setTraceId("");
			processResult.setStatus(DataTransferItemStatusEnum.SUCCESS);
		}catch (Exception e){
			//生成追踪Id,并对异常日志输出
			String traceId=idGenerator.nextId("");
			//输出异常日志,并返回处理状态
			DataTransferItemStatusEnum status=super.handleException(e,traceId,"复工确认明细","数据处理");
			
			//定义处理失败结果
			processResult.setData(null);
			processResult.setBusinessId(detailId);
			processResult.setError(e.getMessage());
			processResult.setTraceId(traceId);
			processResult.setStatus(status);
		}
		return processResult;
	}
}
```

时间段类TimeSegment

```java
/**
 * @author xiejinwei02
 * @date 2023/7/11 20:09
 * 时间段重叠比较类，重写Comparable#compartTo()接口方法，定义规则，segment1在segment2的左测返回-1，segment1在segment2的右侧返回1，其他返回0，表示重叠
 */
public class TimeSegment implements Comparable {
	private Long start;
	private Long end;
	// 与其他时间段的重叠次数
	private Integer overlapCount = 0;
	
	public TimeSegment(Long start,Long end){
		this.start = start;
		this.end = end;
	}
	
	public Integer getOverlapCount(){
		return overlapCount;
	}
	
	
	@Override
	public int compareTo(Object o) {
		TimeSegment other =(TimeSegment) o;
		if(end < other.start){
			return -1;
		}else if(start > other.end){
			return 1;
		}
		overlapCount++;
		return 0;
	}
	
	/**
	 * 是否重叠
	 * @param other 另一个时间段
	 * @return
	 */
	public boolean isOverlap(TimeSegment other){
		return compareTo(other) == 0;
	}
}
```

表结构ER图

![](\assets\images\2023\springboot\workresume-tables.png)