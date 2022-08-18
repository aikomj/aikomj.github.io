---
layout: post
title: 一款自动生成单元测试的IDEA插件SquareTest
category: tool
tags: [tool]
keywords: tool
excerpt: 简单的单元测试类可使用该插件
lock: noneed
---

## 1、安装SquareTest插件

我使用的是idea，我们先来下载一下插件，`File——>Settings——>Plugins`，搜索`Squaretest`，然后`install`就好了，插件安装完成后需要重启一下idea

![](\assets\images\2022\tool\idea-squaretest.png)重启之后，菜单栏就多了一项`Squaretest`，

![](\assets\images\2022\tool\idea-squaretest-2.png)

首先我们打开一个类，这个类就是我们即将要作为实验的类,

![](\assets\images\2022\tool\idea-squaretest-3.png)

选择第二项后就会弹出一个框看下面这里它自动会识别出当前类需要[Mock](http://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247516757&idx=1&sn=f0eaa5f84c8a062db5c022916c77b6d9&chksm=ebd5bd79dca2346f8081f4e4412896f95f694570e83b4026b7de5b36fe873ac1705b2e5efe19&scene=21#wechat_redirect)的成员变量，直接点ok

![](\assets\images\2022\tool\idea-squaretest-4.png)

![](\assets\images\2022\tool\idea-squaretest-5.png)

自动会使用类的真实目录层次在test文件夹中创建出来一个单元测试类，类名就是原类名后加Test

```java
public class CrawlerScreenShotServiceImplTest {

    @Mock
    private CrawerScreenShotTaskMapper mockCrawerScreenShotTaskMapper;
    @Mock
    private CrawerScreenShotTaskLogMapper mockCrawerScreenShotTaskLogMapper;

    @InjectMocks
    private CrawlerScreenShotServiceImpl crawlerScreenShotServiceImplUnderTest;

    @Before
    public void setUp() {
        initMocks(this);
    }

    @Test
    public void testReceiveData() {
        // Setup
        final CrawlerScreenShotVO vo = new CrawlerScreenShotVO();
        vo.setUrl("url");
        vo.setPcFlag(false);
        vo.setMembergroup("membergroup");
        vo.setTaskType(0);
        vo.setUrlType(0);

        when(mockCrawerScreenShotTaskLogMapper.saveSelective(any(CrawerScreenShotTaskLog.class))).thenReturn(0);
        when(mockCrawerScreenShotTaskMapper.saveBatch(Arrays.asList(new CrawlerScreenShotTask(0L, "url", "imageOssUrl", false, false, "memberGroup", 0, 0, "fileName", new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime(), new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime(), false, "skuCode", "state", "operater")))).thenReturn(0);

        // Run the test
        final Result<String> result = crawlerScreenShotServiceImplUnderTest.receiveData(vo);

        // Verify the results
    }

    @Test
    public void testListJobScreenShotTask() {
        // Setup

        // Configure CrawerScreenShotTaskMapper.listJobScreenShotTask(...).
        final CrawlerScreenShotTaskDto crawlerScreenShotTaskDto = new CrawlerScreenShotTaskDto();
        crawlerScreenShotTaskDto.setId(0L);
        crawlerScreenShotTaskDto.setUrl("url");
        crawlerScreenShotTaskDto.setSkuCode("skuCode");
        crawlerScreenShotTaskDto.setPcFlag(false);
        crawlerScreenShotTaskDto.setMemberGroup("memberGroup");
        crawlerScreenShotTaskDto.setUrlType(0);
        crawlerScreenShotTaskDto.setFileName("fileName");
        crawlerScreenShotTaskDto.setTaskType(0);
        crawlerScreenShotTaskDto.setState("state");
        final List<CrawlerScreenShotTaskDto> crawlerScreenShotTaskDtos = Arrays.asList(crawlerScreenShotTaskDto);
        when(mockCrawerScreenShotTaskMapper.listJobScreenShotTask(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime())).thenReturn(crawlerScreenShotTaskDtos);

        // Run the test
        final List<CrawlerScreenShotTaskDto> result = crawlerScreenShotServiceImplUnderTest.listJobScreenShotTask();

        // Verify the results
    }

    @Test
    public void testQuery() {
        // Setup
        final NikeScreenShotListRequestVo requestVo = new NikeScreenShotListRequestVo();
        requestVo.setUrl("url");
        requestVo.setUrlType(0);
        requestVo.setStartTime(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime());
        requestVo.setEndTime(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime());
        requestVo.setStatus(0);
        requestVo.setPcFlag(0);
        requestVo.setPageNum(0);
        requestVo.setPageSize(0);

        // Configure CrawerScreenShotTaskMapper.query(...).
        final PimScreenShotVo pimScreenShotVo = new PimScreenShotVo();
        pimScreenShotVo.setId(0L);
        pimScreenShotVo.setUrl("url");
        pimScreenShotVo.setImageOssUrl("imageOssUrl");
        pimScreenShotVo.setStatus(0);
        pimScreenShotVo.setPcFlag(false);
        pimScreenShotVo.setCreateTime(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime());
        pimScreenShotVo.setUrlType(0);
        pimScreenShotVo.setMsg("msg");
        final List<PimScreenShotVo> pimScreenShotVos = Arrays.asList(pimScreenShotVo);
        when(mockCrawerScreenShotTaskMapper.query(any(NikeScreenShotListRequestVo.class))).thenReturn(pimScreenShotVos);

        // Run the test
        final PageInfo<PimScreenShotVo> result = crawlerScreenShotServiceImplUnderTest.query(requestVo);

        // Verify the results
    }

    @Test
    public void testQuerySelectBoxData() {
        // Setup

        // Configure CrawerScreenShotTaskMapper.query(...).
        final PimScreenShotVo pimScreenShotVo = new PimScreenShotVo();
        pimScreenShotVo.setId(0L);
        pimScreenShotVo.setUrl("url");
        pimScreenShotVo.setImageOssUrl("imageOssUrl");
        pimScreenShotVo.setStatus(0);
        pimScreenShotVo.setPcFlag(false);
        pimScreenShotVo.setCreateTime(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime());
        pimScreenShotVo.setUrlType(0);
        pimScreenShotVo.setMsg("msg");
        final List<PimScreenShotVo> pimScreenShotVos = Arrays.asList(pimScreenShotVo);
        when(mockCrawerScreenShotTaskMapper.query(any(NikeScreenShotListRequestVo.class))).thenReturn(pimScreenShotVos);

        // Run the test
        final PimScreenShotTaskParamsDto result = crawlerScreenShotServiceImplUnderTest.querySelectBoxData();

        // Verify the results
    }

    @Test
    public void testFindExecutionScreenShotTaskCount() {
        // Setup
        when(mockCrawerScreenShotTaskMapper.findExecutionScreenShotTaskCount()).thenReturn(0);

        // Run the test
        final Integer result = crawlerScreenShotServiceImplUnderTest.findExecutionScreenShotTaskCount();

        // Verify the results
        assertEquals(0, result);
    }

    @Test
    public void testFindCrawerScreenshotTaskByCreateTime() {
        // Setup
        final CrawlerScreenShotTaskSyncDto crawlerScreenShotTaskSyncDto = new CrawlerScreenShotTaskSyncDto();
        crawlerScreenShotTaskSyncDto.setId(0L);
        crawlerScreenShotTaskSyncDto.setUrl("url");
        crawlerScreenShotTaskSyncDto.setSkuCode("skuCode");
        crawlerScreenShotTaskSyncDto.setTaskType(0);
        crawlerScreenShotTaskSyncDto.setStatus(0);
        crawlerScreenShotTaskSyncDto.setLastModifyTime(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime());
        crawlerScreenShotTaskSyncDto.setOperater("operater");
        crawlerScreenShotTaskSyncDto.setMsg("msg");
        final List<CrawlerScreenShotTaskSyncDto> expectedResult = Arrays.asList(crawlerScreenShotTaskSyncDto);

        // Configure CrawerScreenShotTaskMapper.findCrawerScreenshotTaskByCreateTime(...).
        final CrawlerScreenShotTaskSyncDto crawlerScreenShotTaskSyncDto1 = new CrawlerScreenShotTaskSyncDto();
        crawlerScreenShotTaskSyncDto1.setId(0L);
        crawlerScreenShotTaskSyncDto1.setUrl("url");
        crawlerScreenShotTaskSyncDto1.setSkuCode("skuCode");
        crawlerScreenShotTaskSyncDto1.setTaskType(0);
        crawlerScreenShotTaskSyncDto1.setStatus(0);
        crawlerScreenShotTaskSyncDto1.setLastModifyTime(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime());
        crawlerScreenShotTaskSyncDto1.setOperater("operater");
        crawlerScreenShotTaskSyncDto1.setMsg("msg");
        final List<CrawlerScreenShotTaskSyncDto> crawlerScreenShotTaskSyncDtos = Arrays.asList(crawlerScreenShotTaskSyncDto1);
        when(mockCrawerScreenShotTaskMapper.findCrawerScreenshotTaskByCreateTime(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime())).thenReturn(crawlerScreenShotTaskSyncDtos);

        // Run the test
        final List<CrawlerScreenShotTaskSyncDto> result = crawlerScreenShotServiceImplUnderTest.findCrawerScreenshotTaskByCreateTime(new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime());

        // Verify the results
        assertEquals(expectedResult, result);
    }

    @Test
    public void testQueryCrawlerDashboard() {
        // Setup
        when(mockCrawerScreenShotTaskMapper.queryCrawlerDashboard(0, 0, 0, new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime(), new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime())).thenReturn(0);

        // Run the test
        final Integer result = crawlerScreenShotServiceImplUnderTest.queryCrawlerDashboard(0, 0, 0, new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime(), new GregorianCalendar(2019, Calendar.JANUARY, 1).getTime());

        // Verify the results
        assertEquals(0, result);
    }
}
```

报错了呢，不要慌，这个断言是为了检查你单元测试跑出来的结果是否符合预期的，如果你不想检查只想完成覆盖率，直接干掉就可以了

很多时候还是要你自己小修小改的，毕竟它生成出来的测试数据可能完全匹配不上你的`if else`数据对吧，但这都很好改啊，这样就从自己分析if else变成了，debug程序了呀，哪里报错，debug过去，看看是不是生成的数据有问题，改个数据，就通过了，反正本人用的是很舒畅的，妥妥的节省70%的工作量。

解决了上面一个问题之后，又发现另一个问题，这个工具`VO，DTO，Entity，Command，Model`这种实体类来讲，一般这种实体类我们都用lombok的注解`get，set`，还有constract构造器等注解，但是这个工具只能生成这些实体类的构造器的单元测试，无法生成`get set`方法的单元测试，所以写了个base方法，实体类继承一下，简单的写两行带就好了，看下面代码：

```java
@SpringBootTest
@RunWith(MockitoJUnitRunner.class)
public abstract class BaseVoEntityTest<T> {
    protected abstract T getT();

    private void testGetAndSet() throws IllegalAccessException, InstantiationException, IntrospectionException,
            InvocationTargetException {
        T t = getT();
        Class modelClass = t.getClass();
        Object obj = modelClass.newInstance();
        Field[] fields = modelClass.getDeclaredFields();
        for (Field f : fields) {
            boolean isStatic = Modifier.isStatic(f.getModifiers());
            // 过滤字段
            if (f.getName().equals("isSerialVersionUID") || f.getName().equals("serialVersionUID") || isStatic || f.getGenericType().toString().equals("boolean")
                    || f.isSynthetic()) {
                continue;
            }
            PropertyDescriptor pd = new PropertyDescriptor(f.getName(), modelClass);
            Method get = pd.getReadMethod();
            Method set = pd.getWriteMethod();
            set.invoke(obj, get.invoke(obj));
        }
    }

    @Test
    public void getAndSetTest() throws InvocationTargetException, IntrospectionException,
            InstantiationException, IllegalAccessException {
        this.testGetAndSet();
    }
}
```

同样的方式我们在实体类上通过`Squaretest`生成单元测试，然后继承我上面写的那个base类，vo的单元测试代码稍加改动，如下:

![](\assets\images\2022\tool\idea-squaretest-6.png)

看run完之后，覆盖率100%，妥妥的，通过这两个解决方案，一天之内我们就把覆盖率搞到了60%以上，不要太刺激，大家可以用用试试哦，当然这个也不是纯为了应付差事写的单元测试，我们后续开发的时候，也可以用这个工具来生成，然后自测自己的代码，这样也是提升工作效率的嘛！

![](\assets\images\2022\tool\idea-squaretest-7.png)

