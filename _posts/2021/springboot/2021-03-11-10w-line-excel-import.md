---
layout: post
title: 10w 级别Excel数据导入的优化记录
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 使用EasyExcel提升读取excel的性能，缓存数据库查询结果集HashMap匹配校验，insert批量插入，适当使用并行流优化插入速度，利用掉网络IO等待时间，避免循环打印无用日志
lock: noneed
---

## 1、需求说明

项目中有一个 Excel 导入的需求：缴费记录导入，由用户将别的系统数据填入我们系统中的 Excel 模板，应用将文件内容读取、校对、转换之后产生欠费数据、票据、票据详情并存储到数据库中。

在我接手之前可能由于之前导入的数据量并不多没有对效率有过高的追求。但是到了 4.0 版本，我预估导入时Excel 行数会是 10w+ 级别，而往数据库插入的数据量是大于 3N 的，也就是说 10w 行的 Excel，则至少向数据库插入 30w 行数据。

因此优化原来的导入代码是势在必行的。我逐步分析和优化了导入的代码，使之在百秒内完成。最终性能瓶颈在数据库的处理速度上，测试服务器 4g 内存不仅放了数据库，还放了很多微服务应用，处理能力不太行，数据库永远是最脆弱的部分。

导入 Excel 的需求在系统中还是很常见的，我的优化办法可能不是最优的，具体过程如下

**细节**

- 数据导入：导入使用的模板由系统提供，格式是 xlsx (支持 65535+行数据) ，用户按照表头在对应列写入相应的数据

- 数据校验：数据校验有两种

  1) 字段长度、字段正则表达式校验等，内存内校验，不存在外部数据交互。对性能影响较小；

  2) 数据重复性校验，如票据号是否和系统已存在的票据号重复(需要查询数据库，十分影响性能)，解决方案对查询做缓存

- 数据插入：测试环境数据库使用 MySQL 5.7，未分库分表，连接池使用 Druid

## 2、迭代记录

### 第一版POI

> POI + 逐行查询校对 + 逐行插入

这个版本是最古老的版本，采用原生 POI，手动将 Excel 中的行映射成 ArrayList 对象，然后存储到 List ，一般程序员写的都是这个版本，代码执行的步骤如下：

1. 手动读取 Excel 成 List
2. 循环遍历，在循环中进行以下步骤
3. a. 检验字段长度；

4. b. 一些查询数据库的校验，比如校验当前行欠费对应的房屋是否在系统中存在，需要查询房屋表

5. c. 写入当前行数据

6. 返回执行结果，如果出错 / 校验不合格。则返回提示信息并回滚数据

显而易见的，这样实现一定是赶工赶出来的，后续可能用的少也没有察觉到性能问题，但是它最多适用于个位数/十位数级别的数据。存在以下明显的问题：

- 查询数据库的校验对每一行数据都要查询一次数据库，应用访问数据库来回的网络IO次数被放大了 n 倍，时间也就放大了 n 倍
- 写入数据也是逐行写入的，问题和上面的一样
- 数据读取使用原生 POI，代码十分冗余，可维护性差。

### 第二版EasyPOI

> 缓存数据库查询操作 + 批量插入

针对第一版分析的三个问题，分别采用以下三个方法优化

1. 缓存数据，以空间换时间

   逐行查询数据库校验的时间成本主要在来回的网络IO中，优化方法也很简单。将参加校验的数据全部缓存到 HashMap 中。直接到 HashMap 去命中。

   例如：校验行中的房屋是否存在，原本是要用 区域 + 楼宇 + 单元 + 房号 去查询房屋表匹配房屋ID，查到则校验通过，生成的欠单中存储房屋ID，校验不通过则返回错误信息给用户。而房屋信息在导入欠费的时候是不会更新的。并且一个小区的房屋信息也不会很多(5000以内)因此我采用一条SQL，将该小区下所有的房屋以 <mark>区域/楼宇/单元/房号 作为 key，以 房屋ID 作为 value</mark>，存储到 HashMap 中，后续校验只需要在 HashMap 中命中。

   > 定义SessionMapper
   
   Mybatis 原生是不支持将查询到的结果直接写人一个 HashMap 中的，需要自定义 SessionMapper
   
   SessionMapper 中指定使用 MapResultHandler 处理 SQL 查询的结果集
   
   ```java
   @Repository
   public class SessionMapper extends SqlSessionDaoSupport {
     @Resource
     public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
       super.setSqlSessionFactory(sqlSessionFactory);
     }
   
     // 区域楼宇单元房号 - 房屋ID
     @SuppressWarnings("unchecked")
     public Map<String, Long> getHouseMapByAreaId(Long areaId) {
       MapResultHandler handler = new MapResultHandler();
       this.getSqlSession().select(BaseUnitMapper.class.getName()+".getHouseMapByAreaId", areaId, handler);
       Map<String, Long> map = handler.getMappedResults();
       return map;
     }
   }  
   ```
   
   MapResultHandler 处理程序，将结果集放入 HashMap
   
   ```java
   public class MapResultHandler implements ResultHandler {
       private final Map mappedResults = new HashMap();
   
       @Override
       public void handleResult(ResultContext context) {
           @SuppressWarnings("rawtypes")
           Map map = (Map)context.getResultObject();
           mappedResults.put(map.get("key"), map.get("value"));
       }
   
       public Map getMappedResults() {
           return mappedResults;
       }
   }
   ```
   
   Mapper
   
   ```java
   @Mapper
   @Repository 
   public interface BaseUnitMapper {
       // 收费标准绑定 区域楼宇单元房号 - 房屋ID
       Map<String, Long> getHouseMapByAreaId(@Param("areaId") Long areaId);
   }   
   ```
   
   Mapper.xml
   
   ```xml
   <select id="getHouseMapByAreaId" resultMap="mapResultLong">
       SELECT
           CONCAT( h.bulid_area_name, h.build_name, h.unit_name, h.house_num ) k,
           h.house_id v
       FROM
           base_house h
       WHERE
           h.area_id = #{areaId}
       GROUP BY
           h.house_id
   </select>
               
   <resultMap id="mapResultLong" type="java.util.HashMap">
       <result property="key" column="k" javaType="string" jdbcType="VARCHAR"/>
       <result property="value" column="v" javaType="long" jdbcType="INTEGER"/>
   </resultMap>  
   ```
   
   之后在代码中调用 SessionMapper 类对应的方法即可。获取一个小区的房屋信息。

2. **使用 values 批量插入**

   MySQL insert 语句支持使用 values (),(),() 的方式一次插入多行数据，通过 mybatis foreach 结合 java 集合可以实现批量插入，代码写法如下：

   ```xml
   <insert id="insertList">
       insert into table(colom1, colom2)
       values
       <foreach collection="list" item="item" index="index" separator=",">
        ( #{item.colom1}, #{item.colom2})
       </foreach>
   </insert>
   ```

3. 使用EasyPOI 读写Excel

   EasyPOI 采用基于注解的导入导出,修改注解就可以修改Excel，非常方便，代码维护起来也容易。

### 第三版EasyExcel

第二版采用 EasyPOI 之后，对于几千、几万的 Excel 数据已经可以轻松导入了，不过耗时有点久(5W 数据 10分钟左右写入到数据库)不过由于后来导入的操作基本都是开发在一边看日志一边导入，也就没有进一步优化。但是好景不长，有新小区需要迁入，票据 Excel 有 41w 行，这个时候使用 EasyPOI 在开发环境跑直接就 OOM 了，增大 JVM 内存参数之后，虽然不 OOM 了，但是 CPU 占用 100% 20 分钟仍然未能成功读取全部数据。故在读取大 Excel 时需要再优化速度。莫非要我这个渣渣去深入 POI 优化了吗？别慌，先上 GITHUB 找找别的开源项目。这时阿里 EasyExcel 映入眼帘：

飞天班相关文章[http://139.199.13.139/blog/icoding-edu/2020/04/12/icoding-note-021.html](http://139.199.13.139/blog/icoding-edu/2020/04/12/icoding-note-021.html)

![](\assets\images\2021\javabase\easyexcel.png)

easyexcel重写了poi对07版Excel的解析，能够原本一个3M的excel用POI  sax依然需要100M左右内存降低到几M，并且再大的excel不会出现内存溢出。

github地址：[https://github.com/alibaba/easyexcel](https://github.com/alibaba/easyexcel)

官方文档：[https://www.yuque.com/easyexcel/doc/easyexcel](https://www.yuque.com/easyexcel/doc/easyexcel)

EasyExcel 采用和 EasyPOI 类似的注解方式读写 Excel，因此从 EasyPOI 切换过来很方便，分分钟就搞定了。也确实如阿里大神描述的：41w行、25列、45.5m 数据读取平均耗时 50s，因此对于大 Excel 建议使用 EasyExcel 读取。

springboot导入依赖

```xml
<dependency>
   <groupId>com.alibaba</groupId>
   <artifactId>easyexcel</artifactId>
   <version>2.2.4</version>
</dependency>
```

demo示例根据官方例子快速开始就可以了，不难。



### 第四版优化插入速度

在第二版插入的时候，我使用了 values 批量插入代替逐行插入。每 30000 行拼接一个长 SQL、顺序插入。整个导入方法这块耗时最多，非常拉跨。后来我将每次拼接的行数减少到 10000、5000、3000、1000、500 发现执行最快的是 1000。结合网上一些对 innodb_buffer_pool_size 描述我猜是因为过长的 SQL 在写操作的时候由于超过内存阈值，发生了磁盘交换。限制了速度，另外测试服务器的数据库性能也不怎么样，过多的插入他也处理不过来。所以最终采用每次 1000 条插入。

每次 1000 条插入后，为了榨干数据库的 CPU，那么网络IO的等待时间就需要利用起来，这个需要多线程来解决，而最简单的多线程可以使用 并行流 来实现，接着我将代码用并行流来测试了一下：

10w行的 excel、42w 欠单、42w记录详情、2w记录、16 线程并行插入数据库、每次 1000 行。插入时间 72s，导入总时间 95 s。

![](\assets\images\2021\javabase\easyexcel-test.png)

并行插入工具类，封装成一个函数式编程的工具类

```java
/**
 * 功能：利用并行流快速插入数据
 *
 * @author Keats
 * @date 2020/7/1 9:25
 */
public class InsertConsumer {
    /**
     * 每个长 SQL 插入的行数，可以根据数据库性能调整
     */
    private final static int SIZE = 1000;

    /**
     * 如果需要调整并发数目，修改下面方法的第二个参数即可
     */
    static {
  System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "4");
    }

    /**
     * 插入方法
     *
     * @param list     插入数据集合
     * @param consumer 消费型方法，直接使用 mapper::method 方法引用的方式
     * @param <T>      插入的数据类型
     */
    public static <T> void insertData(List<T> list, Consumer<List<T>> consumer) {
        if (list == null || list.size() < 1) {
            return;
        }

        List<List<T>> streamList = new ArrayList<>();

        for (int i = 0; i < list.size(); i += SIZE) {
            int j = Math.min((i + SIZE), list.size());
            List<T> subList = list.subList(i, j);
            streamList.add(subList);
        }
        // 并行流使用的并发数是 CPU 核心数，不能局部更改。全局更改影响较大，斟酌
        streamList.parallelStream().forEach(consumer);
    }
}
```

方法用起来很简单，直接调用mapper的insert方法

```java
InsertConsumer.insertData(feeList, arrearageMapper::insertList);
```

## 3、其他影响性能的内容

### 减少输出日志

> 避免在 for 循环中打印过多的 info 日志

在优化的过程中，我还发现了一个**特别影响性能**的东西：info 日志，还是使用 41w行、25列、45.5m 数据，在 **开始-数据读取完毕** 之间每 1000 行打印一条 info 日志，**缓存校验数据-校验完毕** 之间每行打印 3+ 条 info 日志，日志框架使用 Slf4j 。打印并持久化到磁盘。下面是打印日志和不打印日志效率的差别

- 打印日志

  ![](\assets\images\2021\javabase\easyexcel-test-print-log.png)

- 不打印日志

  ![](\assets\images\2021\javabase\easyexcel-test-no-print-log.png)

缓存校验数据-校验完毕 不打印日志耗时仅仅是打印日志耗时的 1/10 ！

## 4、总结

提升Excel导入速度的方法：

- 使用更快的 Excel 读取框架(推荐使用阿里 EasyExcel)
- 对于需要与数据库交互的校验、按照业务逻辑适当的使用缓存。用空间换时间
- 使用 values(),(),() 拼接长 SQL 一次插入多行数据
- 使用多线程插入数据，利用掉网络IO等待时间(推荐使用并行流，简单易用)
- 避免在循环中打印无用的日志

## 5、信息回写

上面使用了easyExcel导入文件内容到数据库，导入过程中某些行出错，我们可以把出错信息回写到excel对应行中，返回一个excel文件的下载路径给前端，用户下载后了解具体的行错误信息。在ccs项目组的时候，导入库存账龄保留天数就是这样实现的。

web层请求接口StockInvAccountAgeController.java

```java
@RequestMapping("/mobile/stockInvAccountAgeCtrl")
public class StockInvAccountAgeController {
    private static final Logger logger = LoggerFactory.getLogger(StockInvAccountAgeController.class.getName());

  // 配置的excel文件夹路径
  @Value("${ccs.file.importResult.dir}")
  private String importResultDir;
  
  /**
     * 导入库存账龄保留天数
     */
    @ResponseBody
    @RequestMapping(value = "/excelImport", method = RequestMethod.POST)
    public Map<String, Object> importCreate(HttpServletRequest request){
        Map<String, Object> result = new HashMap<>(4);
        MultipartHttpServletRequest multiRequest = (MultipartHttpServletRequest)request;
        Iterator<String> ite = multiRequest.getFileNames();
        MultipartFile file = null;
        List<Map<Object, Object>> list = null;
        try{
            // 1、载入文件内容
            while(ite.hasNext()){
                file = multiRequest.getFile(ite.next());
                if(file != null){
                    try {
                        list= ExcelImportUtils.getUploadImportData(file.getOriginalFilename(), file.getInputStream());
                    } catch (IOException e) {
                        logger.error(e.getMessage());
                    }
                }
            }
            // 2、校验第一行模板
            if(list.isEmpty() || !"事业部账套".equals(list.get(0).get(0)) || !"商品编码".equals(list.get(0).get(1)) || !"保留天数".equals(list.get(0).get(2))){
                result.put("success", false);
                result.put("code", "0500");
                result.put("msg", "数据异常！");
                return result;
            }
            AtomicInteger successCount = new AtomicInteger(0); // 成功条数
            int failCount = 0; // 失败条数
            int importDataCnt = list.size() - 1;//去掉标题行
            Map<Integer,String> bsImportMsg =new ConcurrentHashMap<>(importDataCnt);
            CountDownLatch countDownLatch = new CountDownLatch(importDataCnt);
            // 查询所有的事业部账套ID
            List<Integer> setsOfBooksList = stockInvAccountAgeFacade.listAllBooksId();
            // 创建线程资源类
            DealProtectDays dealProtectDays = new DealProtectDays(list,successCount,bsImportMsg,setsOfBooksList);
            // 创建线程池
            ExecutorService threadPool = new ThreadPoolExecutor(7,50,10,TimeUnit.SECONDS
                    ,new LinkedBlockingDeque<>(100),new BlockRejectedExecutionHandler());

            try {
                // 逐条处理导入数据
                for (int i = 1; i <= importDataCnt; i++) {
                    final int tempRowNo = i;
                    threadPool.execute(()->{
                        dealProtectDays.importExcelData(tempRowNo);
                        countDownLatch.countDown();
                    });
                }
                // 阻塞主线程
                countDownLatch.await();
                // System.out.printf("导入完成了，呵呵");
            } finally {
                threadPool.shutdown();
            }

            failCount = importDataCnt - successCount.intValue();
            if(failCount > 0){
                // 有导入失败行，失败行信息写回原Excel文件
                String resultFileUrl = ExcelImportUtils.getResultFileUrl(request, file, bsImportMsg, importResultDir);
                result.put("msg", "导入总条数："+importDataCnt+"<br/>成功条数："+successCount.intValue()+"<br/>失败条数："+failCount);
                result.put("resultFileUrl", resultFileUrl);
            }else{
                result.put("msg", "导入总条数："+importDataCnt+"<br/>成功条数："+successCount);
            }
            result.put("success", true);
            result.put("code", "0000");
            result.put("successCount", successCount.intValue());
            result.put("failCount", failCount);
        }catch (Exception e) {
            result.put("success", false);
            result.put("code", "0500");
            result.put("msg", "数据异常！");
            logger.error("导入数据失败：", e);
        }

        return result;
    }
}
```

工具类ExcelImportUtils.java，导入功能可以使用easyExcel替代

```java
public class ExcelImportUtils {
  	public static List<Map<Object, Object>> getUploadImportData(String fileName, InputStream inputStream) {
		return getUploadImportData(fileName, inputStream, null);
	}
  
  // 获取excel文件数据
  public static List<Map<Object, Object>> getUploadImportData(String fileName, InputStream inputStream,
			List<String> headerList) {
		List<Map<Object, Object>> list = new ArrayList<Map<Object, Object>>();
		OPCPackage pkg = null;
		Workbook workBook = null;
		try {
			if (fileName.toLowerCase().endsWith(".xls")) { // 2003-2007版本的Excel文件
				 workBook = new HSSFWorkbook(inputStream);
				saveData4Excel(workBook.getSheetAt(0), list, headerList);
			} else if(fileName.toLowerCase().endsWith(".xlsx")) { // 2007版本或更高版本的Excel文件
				pkg = OPCPackage.open(inputStream);
				 workBook = new XSSFWorkbook(pkg);
		        saveData4Excel(workBook.getSheetAt(0), list, headerList);
			} else {
				return null;
			}
		} catch (Exception e) {
			logger.error("读取导入数据异常。", e);
		} finally {
			try {				
				if(workBook !=null){
					workBook.close();
				}				
				if (pkg != null) {
					pkg.close();
				}
				if (inputStream != null) {
					inputStream.close();
				}
			} catch (IOException e) {
				logger.error("IO流关闭异常。", e);
			}
		}
		return list;
	}
  
  // 表格数据读取到List
  private static void saveData4Excel(Sheet sheet, List<Map<Object, Object>> list, List<String> headerList) {
		if (sheet == null) {
			throw new IllegalArgumentException("sheet 不能为null");
		}
		if (list == null) {
			throw new IllegalArgumentException("list 不能为null");
		}
		// 定义 row、cell
		Object value = null;
		Row row = null;
		Cell cell = null;
		// 循环输出表格中的内容
		/*
		 * 如果中间各行或者隔列的话getPhysicalNumberOfRows和getPhysicalNumberOfCells就不能读取到所有的行和列了
		 * sheet.getLastRowNum() + 1
		 * row.getLastCellNum()
		 */
		int i=0;
		if(headerList!=null){
			i=sheet.getFirstRowNum()+1;
		}else{
			i=sheet.getFirstRowNum();
		}
		for (; i < sheet.getLastRowNum() + 1; i++) {
			row = sheet.getRow(i);
			if (isEmptyRow(row)) {
				continue;
			}
			Map<Object, Object> map = new HashMap<Object, Object>();
			int firstCellNum = row.getFirstCellNum();
			int lastCellNum = row.getLastCellNum();
			for (int j = firstCellNum; j < lastCellNum; j++) {
				cell = row.getCell(j);
				if(isEmptyCell(cell)) {
					continue;
				}
				value = getCellValue(cell);
				if (isEmptyValue(value)) {
					continue;
				}
				map.put(j, value);				
			}
			list.add(map);
		}
		sheet = null;
	}
}
```

线程资源类DealProtectDays.java

```java
// 线程资源类
public class DealProtectDays {
  private List<Map<Object, Object>> list;
  private AtomicInteger successCount;
  private Map<Integer,String> bsImportMsg;
  private List<Integer> setsOfBooksList;
  private String loginId;

  public DealProtectDays(List<Map<Object, Object>> list,AtomicInteger successCount,Map<Integer,String> bsImportMsg,List<Integer> setsOfBooksList){
    this.list = list;
    this.successCount = successCount;
    this.bsImportMsg = bsImportMsg;
    this.setsOfBooksList = setsOfBooksList;
    this.loginId = SecurityUtils.getSessionUser().getLoginId(); // 登录账号
  }

  public void importExcelData(int i){
    try{
      Map<Object,Object> data = list.get(i);
      data.put("loginId",loginId);
      stockInvAccountAgeFacade.saveImportData(data,setsOfBooksList);
      bsImportMsg.put(i, "导入成功");
      successCount.incrementAndGet();
    }catch (Exception e){
      logger.error(e.getMessage(), e);
      bsImportMsg.put(i, e.getMessage());
    }
  }
}
```

工具类ExcelImportUtils.java，错误信息写回excel文件部分

```java
// 有导入失败行，失败行信息写回原Excel文件
String resultFileUrl = ExcelImportUtils.getResultFileUrl(request, file, bsImportMsg, importResultDir);
```

```java
public class ExcelImportUtils {
  // 错误信息写入文件，返回文件路径
  public static String getResultFileUrl(HttpServletRequest request, MultipartFile importFile, 
                                        Map<Integer, String> errorMsgMap, String importResultDir) {
    // 保存导入结果文件的文件夹在服务器上的本地路径
    String importResultLocalDir =importResultDir;

    String fileDate = fileDateSdf.format(new Date());		
    importResultLocalDir += File.separator + fileDate;

    File tempFile = new File(importResultLocalDir);
    if (!tempFile.exists()) { // 文件夹不存在就新建
      tempFile.mkdirs();
    }

    // 通过UUID创建唯一编码
    String fileId = UUID.randomUUID().toString();
    // 导出结果文件的后缀名要与导入文件的一致
    String fileName = fileId + getFileSuffix(importFile.getOriginalFilename());
    // 导出结果文件的服务器路径
    String fileLocalPath = importResultLocalDir + File.separator + fileName;
    boolean result = false;
    try {
      result = writeImportResultFile(errorMsgMap, 0, importFile.getInputStream(), fileLocalPath);
    } catch (ApplicationException e) {
      logger.error("文件生成失败：", e);
      return null;
    } catch (IOException e) {
      logger.error("文件生成失败：", e);
      return null;
    }
    if (result) {
      return importResultLocalDir
        + File.separator  + fileName;
    }
    return null;
  }
  // 写入workbook  
	private static boolean writeImportResultFile(Map<Integer, String> map, int page, 
			InputStream inputStream,String fileName) throws IOException, ApplicationException {
		OPCPackage pkg = null;
		Workbook workbook = null;
		try{
			if (fileName.toLowerCase().endsWith(".xls")) {
				workbook = new HSSFWorkbook(inputStream);
				return updateMessage4Excel(workbook, page, map, fileName);
			} else if (fileName.toLowerCase().endsWith(".xlsx")) {
				pkg = OPCPackage.open(inputStream);
				workbook = new XSSFWorkbook(pkg);
				return updateMessage4Excel(workbook, page, map, fileName);
			}
		}catch(Exception e){
			logger.error("写入数据异常。", e);
			throw new ApplicationException("写入数据异常。");
		}finally {
			try {				
				if(workbook !=null){
					workbook.close();
				}				
				if (pkg != null) {					
					pkg.close();
				}
				if (inputStream != null) {
					inputStream.close();
				}
			} catch (IOException e) {
				logger.error("IO流关闭异常。", e);
			}
		}
		throw new ApplicationException("文件类型错误，必须是后缀名为.xls或者.xlsx的excel文件");
	}
  // 写入Sheet
  private static boolean updateMessage4Excel(Workbook workbook, int page, 
			Map<Integer, String> map,String downloadFileName) {
		if (map == null) {
			return false;
		}
		if (workbook == null) {
			return false;
		}
		if (StringUtils.isEmpty(downloadFileName)) {
			return false;
		}
		Sheet sheet = workbook.getSheetAt(page);
		if (sheet == null) {
			return false;
		}
		// 定义 row、cell
		Row row = null;
		Cell cell = null;
		int createColInd = 0;
				
		// 循环输出表格中的内容
		/*
		 * 如果中间各行或者隔列的话getPhysicalNumberOfRows和getPhysicalNumberOfCells就不能读取到所有的行和列了
		 * sheet.getLastRowNum() + 1
		 * row.getLastCellNum()
		 */
		int emptyRowNum = 0;
		for (int i = sheet.getFirstRowNum(); i < sheet.getLastRowNum() + 1; i++) {
			row = sheet.getRow(i);
			if (isEmptyRow(row)) {
				emptyRowNum++;
				continue;
			}
			if (i == 0) { // 以头行的第一个空白列做为错误信息列
				createColInd = row.getLastCellNum();
			} else if (createColInd == 0){ // 否则以第一非空行的第一个空白列做为错误信息列
				createColInd = row.getLastCellNum();
			}
			cell = row.createCell(createColInd);
			cell.setCellValue(map.get(i - emptyRowNum));
		}
		FileOutputStream fos = null;
		try {
			fos = new FileOutputStream(downloadFileName);
			workbook.write(fos);
			return true;
		} catch (Exception e) {
			logger.error("写入导入失败文件失败：", e);
			return false;
		} finally {
			sheet = null;
			if (fos != null) {
				try {
					fos.close();
				} catch (IOException e) {
					logger.error("关闭导入失败文件输出流失败：", e);
				}
			}
			if (workbook != null) {
				try {
					workbook.close();
				} catch (IOException e) {
					logger.error("关闭导入失败文件输出流失败：", e);
				}
			}
		}	
	}
}
```

如果是微服务，excel文件不能保留在服务单例上，应该上传到fastDFS或者OSS上





