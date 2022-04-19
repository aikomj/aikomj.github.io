---

layout: post
title: 10w 级别Excel数据导入的优化记录
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 使用EasyExcel提升读取excel的性能，缓存数据库查询结果集HashMap匹配校验，insert批量插入，适当使用并行流优化插入速度，利用掉网络IO等待时间，避免循环打印无用日志,导入结果回写原excel文件提供下载，异步分页导出大量数据
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

[飞天班21节-企业项目研发poi实际应用](/icoding-edu/2020/04/12/icoding-note-021.html)

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
    <version>3.0.5</version>
</dependency>
```

demo示例根据官方例子快速开始就可以了，不难。

### EasyExcel的高阶使用

EasyExcel和EasyPoi的使用非常类似，都是通过注解来控制导入导出。接下来我们以会员信息和订单信息的导入导出为例，分别实现下简单的单表导出和具有一对多关系的复杂导出。

- **简单导出**

  首先创建一个会员对象`Member`，封装会员信息，这里使用了EasyExcel的注解；

  ```java
  @Data
  @EqualsAndHashCode(callSuper = false)
  public class Member {
      @ExcelProperty("ID")
      @ColumnWidth(10)
      private Long id;
    
      @ExcelProperty("用户名")
      @ColumnWidth(20)
      private String username;
    
      @ExcelIgnore
      private String password;
    
      @ExcelProperty("昵称")
      @ColumnWidth(20)
      private String nickname;
    
      @ExcelProperty("出生日期")
      @ColumnWidth(20)
      @DateTimeFormat("yyyy-MM-dd")
      private Date birthday;
    
      @ExcelProperty("手机号")
      @ColumnWidth(20)
      private String phone;
    
      @ExcelIgnore
      private String icon;
    
      @ExcelProperty(value = "性别", converter = GenderConverter.class)
      @ColumnWidth(10)
      private Integer gender;
  }
  ```

  - @ExcelProperty：核心注解，`value`属性可用来设置表头名称，`converter`属性可以用来设置类型转换器；
  - @ColumnWidth：用于设置表格列的宽度；
  - @DateTimeFormat：用于设置日期转换格式。

  在EasyExcel中，如果你想实现枚举类型到字符串的转换（比如gender属性中，`0->男`，`1->女`），需要自定义转换器，下面为自定义的`GenderConverter`代码实现；

  ```java
  public class GenderConverter implements Converter<Integer> {
      @Override
      public Class<?> supportJavaTypeKey() {
          //对象属性类型
          return Integer.class;
      }
  
      @Override
      public CellDataTypeEnum supportExcelTypeKey() {
          //CellData属性类型
          return CellDataTypeEnum.STRING;
      }
  
      @Override
      public Integer convertToJavaData(ReadConverterContext<?> context) throws Exception {
          //CellData转对象属性
          String cellStr = context.getReadCellData().getStringValue();
          if (StrUtil.isEmpty(cellStr)) return null;
          if ("男".equals(cellStr)) {
              return 0;
          } else if ("女".equals(cellStr)) {
              return 1;
          } else {
              return null;
          }
      }
  
      @Override
      public WriteCellData<?> convertToExcelData(WriteConverterContext<Integer> context) throws Exception{
          //对象属性转CellData
          Integer cellValue = context.getValue();
          if (cellValue == null) {
              return new WriteCellData<>("");
          }
          if (cellValue == 0) {
              return new WriteCellData<>("男");
          } else if (cellValue == 1) {
              return new WriteCellData<>("女");
          } else {
              return new WriteCellData<>("");
          }
      }
  }
  ```

  接下来我们在Controller中添加一个接口，用于导出会员列表到Excel，还需给响应头设置下载excel的属性，具体代码如下

  ```java
  @Controller
  @Api(tags = "EasyExcelController", description = "EasyExcel导入导出测试")
  @RequestMapping("/easyExcel")
  public class EasyExcelController {
      @SneakyThrows(IOException.class)
      @ApiOperation(value = "导出会员列表Excel")
      @RequestMapping(value = "/exportMemberList", method = RequestMethod.GET)
      public void exportMemberList(HttpServletResponse response) {
          setExcelRespProp(response, "会员列表");
          List<Member> memberList = LocalJsonUtil.getListFromJson("json/members.json", Member.class);
          EasyExcel.write(response.getOutputStream())
                  .head(Member.class)
                  .excelType(ExcelTypeEnum.XLSX)
                  .sheet("会员列表")
                  .doWrite(memberList);
      }
      
    /**
     * 设置excel下载响应头属性
     */
    private void setExcelRespProp(HttpServletResponse response, String rawFileName) throws UnsupportedEncodingException {
      response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
      response.setCharacterEncoding("utf-8");
      String fileName = URLEncoder.encode(rawFileName, "UTF-8").replaceAll("\\+", "%20");
      response.setHeader("Content-disposition", "attachment;filename*=utf-8''" + fileName + ".xlsx");
    }
  }
  ```

  运行项目，通过Swagger测试接口，注意在Swagger中访问接口无法直接下载，需要点击返回结果中的`下载按钮`才行，访问地址：http://localhost:8088/swagger-ui/

  ![](/assets/images/2022/springboot/easyexcel-1.png)

  下载完成后，查看下文件，一个标准的Excel文件已经被导出了

- **简单导入**

  在Controller中添加会员信息导入的接口，这里需要注意的是使用`@RequestPart`注解修饰文件上传参数，否则在Swagger中就没法显示上传按钮了；

  ```java
  @Controller
  @Api(tags = "EasyExcelController", description = "EasyExcel导入导出测试")
  @RequestMapping("/easyExcel")
  public class EasyExcelController {
      
      @SneakyThrows
      @ApiOperation("从Excel导入会员列表")
      @RequestMapping(value = "/importMemberList", method = RequestMethod.POST)
      @ResponseBody
      public CommonResult importMemberList(@RequestPart("file") MultipartFile file) {
          List<Member> memberList = EasyExcel.read(file.getInputStream())
                  .head(Member.class)
                  .sheet()
                  .doReadSync();
          return CommonResult.success(memberList);
      }
  }
  ```

  然后在Swagger中测试接口，选择之前导出的Excel文件即可，导入成功后会返回解析到的数据。

  ![](/assets/images/2022/springboot/easyexcel-2.png)

- **复杂导出**

  使用EasyPoi实现，由于EasyPoi本来就支持嵌套对象的导出，直接使用内置的`@ExcelCollection`注解即可实现，非常方便也符合面向对象的思想。

  ![](/assets/images/2022/springboot/easyexcel-3.png)

  > 由于EasyExcel本身并不支持这种一对多的信息导出，所以我们得自行实现下，这里分享一个我平时常用的`快速查找解决方案`的办法。

  我们可以直接从开源项目的`issues`里面去搜索，比如搜索下`一对多`，会直接找到`有无一对多导出比较优雅的方案`这个issue。

  ![](/assets/images/2022/springboot/easyexcel-4.png)

从此issue的回复我们可以发现，项目维护者建议`创建自定义合并策略`来实现，有位回复的老哥已经给出了实现代码，接下来我们就用这个方案来实现下。

![](/assets/images/2022/springboot/easyexcel-5.png)

**解决思路**

为什么自定义单元格合并策略能实现一对多的列表信息的导出呢？首先我们来看下将嵌套数据平铺，不进行合并导出的Excel。

![](/assets/images/2022/springboot/easyexcel-6.png)

看完之后我们很容易理解解决思路，只要把`订单ID`相同的列中需要合并的列给合并了，就可以实现这种一对多嵌套信息的导出了。

> 实现过程

- 首先我们得把原来嵌套的订单商品信息给平铺了，创建一个专门的导出对象`OrderData`，包含订单和商品信息，二级表头可以通过设置`@ExcelProperty`的value为数组来实现；

  ```java
  @Data
  @EqualsAndHashCode(callSuper = false)
  public class OrderData {
      @ExcelProperty(value = "订单ID")
      @ColumnWidth(10)
      @CustomMerge(needMerge = true, isPk = true)
      private String id;
    
      @ExcelProperty(value = "订单编码")
      @ColumnWidth(20)
      @CustomMerge(needMerge = true)
      private String orderSn;
    
      @ExcelProperty(value = "创建时间")
      @ColumnWidth(20)
      @DateTimeFormat("yyyy-MM-dd")
      @CustomMerge(needMerge = true)
      private Date createTime;
    
      @ExcelProperty(value = "收货地址")
      @CustomMerge(needMerge = true)
      @ColumnWidth(20)
      private String receiverAddress;
    
      @ExcelProperty(value = {"商品信息", "商品编码"})
      @ColumnWidth(20)
      private String productSn;
    
      @ExcelProperty(value = {"商品信息", "商品名称"})
      @ColumnWidth(20)
      private String name;
    
      @ExcelProperty(value = {"商品信息", "商品标题"})
      @ColumnWidth(30)
      private String subTitle;
    
      @ExcelProperty(value = {"商品信息", "品牌名称"})
      @ColumnWidth(20)
      private String brandName;
    
      @ExcelProperty(value = {"商品信息", "商品价格"})
      @ColumnWidth(20)
      private BigDecimal price;
    
      @ExcelProperty(value = {"商品信息", "商品数量"})
      @ColumnWidth(20)
      private Integer count;
  }
  ```

- 然后将原来嵌套的`Order`对象列表转换为`OrderData`对象列表；

  ```java
  @Controller
  @Api(tags = "EasyExcelController", description = "EasyExcel导入导出测试")
  @RequestMapping("/easyExcel")
  public class EasyExcelController {
      private List<OrderData> convert(List<Order> orderList) {
          List<OrderData> result = new ArrayList<>();
          for (Order order : orderList) {
              List<Product> productList = order.getProductList();
              for (Product product : productList) {
                  OrderData orderData = new OrderData();
                  BeanUtil.copyProperties(product,orderData);
                  BeanUtil.copyProperties(order,orderData);
                  result.add(orderData);
              }
          }
          return result;
      }
  }
  ```

- 再创建一个自定义注解`CustomMerge`，用于标记哪些属性需要合并，哪个是主键；

  ```java
  /**
   * 自定义注解，用于判断是否需要合并以及合并的主键
   */
  @Target({ElementType.FIELD})
  @Retention(RetentionPolicy.RUNTIME)
  @Inherited
  public @interface CustomMerge {
  
      /**
       * 是否需要合并单元格
       */
      boolean needMerge() default false;
  
      /**
       * 是否是主键,即该字段相同的行合并
       */
      boolean isPk() default false;
  }
  ```

- 再创建自定义单元格合并策略类`CustomMergeStrategy`，当Excel中两列主键相同时，合并被标记需要合并的列；

  ```java
  /**
   * 自定义单元格合并策略
   */
  public class CustomMergeStrategy implements RowWriteHandler {
      /**
       * 主键下标
       */
      private Integer pkIndex;
  
      /**
       * 需要合并的列的下标集合
       */
      private List<Integer> needMergeColumnIndex = new ArrayList<>();
  
      /**
       * DTO数据类型
       */
      private Class<?> elementType;
  
      public CustomMergeStrategy(Class<?> elementType) {
          this.elementType = elementType;
      }
  
      @Override
      public void afterRowDispose(WriteSheetHolder writeSheetHolder, WriteTableHolder writeTableHolder, Row row, Integer relativeRowIndex, Boolean isHead) {
          // 如果是标题,则直接返回
          if (isHead) {
              return;
          }
  
          // 获取当前sheet
          Sheet sheet = writeSheetHolder.getSheet();
  
          // 获取标题行
          Row titleRow = sheet.getRow(0);
  
          if (null == pkIndex) {
              this.lazyInit(writeSheetHolder);
          }
  
          // 判断是否需要和上一行进行合并
          // 不能和标题合并，只能数据行之间合并
          if (row.getRowNum() <= 1) {
              return;
          }
          // 获取上一行数据
          Row lastRow = sheet.getRow(row.getRowNum() - 1);
          // 将本行和上一行是同一类型的数据(通过主键字段进行判断)，则需要合并
          if (lastRow.getCell(pkIndex).getStringCellValue().equalsIgnoreCase(row.getCell(pkIndex).getStringCellValue())) {
              for (Integer needMerIndex : needMergeColumnIndex) {
                  CellRangeAddress cellRangeAddress = new CellRangeAddress(row.getRowNum() - 1, row.getRowNum(),
                          needMerIndex, needMerIndex);
                  sheet.addMergedRegionUnsafe(cellRangeAddress);
              }
          }
      }
  
      /**
       * 初始化主键下标和需要合并字段的下标
       */
      private void lazyInit(WriteSheetHolder writeSheetHolder) {
  
          // 获取当前sheet
          Sheet sheet = writeSheetHolder.getSheet();
  
          // 获取标题行
          Row titleRow = sheet.getRow(0);
          // 获取DTO的类型
          Class<?> eleType = this.elementType;
  
          // 获取DTO所有的属性
          Field[] fields = eleType.getDeclaredFields();
  
          // 遍历所有的字段，因为是基于DTO的字段来构建excel，所以字段数 >= excel的列数
          for (Field theField : fields) {
              // 获取@ExcelProperty注解，用于获取该字段对应在excel中的列的下标
              ExcelProperty easyExcelAnno = theField.getAnnotation(ExcelProperty.class);
              // 为空,则表示该字段不需要导入到excel,直接处理下一个字段
              if (null == easyExcelAnno) {
                  continue;
              }
              // 获取自定义的注解，用于合并单元格
              CustomMerge customMerge = theField.getAnnotation(CustomMerge.class);
  
              // 没有@CustomMerge注解的默认不合并
              if (null == customMerge) {
                  continue;
              }
  
              for (int index = 0; index < fields.length; index++) {
                  Cell theCell = titleRow.getCell(index);
                  // 当配置为不需要导出时，返回的为null，这里作一下判断，防止NPE
                  if (null == theCell) {
                      continue;
                  }
                  // 将字段和excel的表头匹配上
                  if (easyExcelAnno.value()[0].equalsIgnoreCase(theCell.getStringCellValue())) {
                      if (customMerge.isPk()) {
                          pkIndex = index;
                      }
  
                      if (customMerge.needMerge()) {
                          needMergeColumnIndex.add(index);
                      }
                  }
              }
          }
  
          // 没有指定主键，则异常
          if (null == this.pkIndex) {
              throw new IllegalStateException("使用@CustomMerge注解必须指定主键");
          }
  
      }
  }
  ```

- 接下来在Controller中添加导出订单列表的接口，将我们自定义的合并策略`CustomMergeStrategy`给注册上去；

  ```java
  @Controller
  @Api(tags = "EasyExcelController", description = "EasyExcel导入导出测试")
  @RequestMapping("/easyExcel")
  public class EasyExcelController {
      
      @SneakyThrows
      @ApiOperation(value = "导出订单列表Excel")
      @RequestMapping(value = "/exportOrderList", method = RequestMethod.GET)
      public void exportOrderList(HttpServletResponse response) {
          List<Order> orderList = getOrderList();
          List<OrderData> orderDataList = convert(orderList);
          setExcelRespProp(response, "订单列表");
          EasyExcel.write(response.getOutputStream())
                  .head(OrderData.class)
                  .registerWriteHandler(new CustomMergeStrategy(OrderData.class))
                  .excelType(ExcelTypeEnum.XLSX)
                  .sheet("订单列表")
                  .doWrite(orderDataList);
      }
  }
  ```

在Swagger中访问接口测试，导出订单列表对应Excel；

![](/assets/images/2022/springboot/easyexcel-1.png)

下载完成后，查看下文件，由于EasyExcel需要自己来实现，对比之前使用EasyPoi来实现麻烦了不少。

![](/assets/images/2022/springboot/easyexcel-8.png)

**其他使用**

由于EasyExcel的官方文档介绍的比较简单，如果你想要更深入地进行使用的话，建议大家看下官方Demo

![](/assets/images/2022/springboot/easyexcel-7.png)

参考地址：

- 项目地址：https://github.com/alibaba/easyexcel
- 官方文档：https://www.yuque.com/easyexcel/doc/easyexcel

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

1）使用更快的 Excel 读取框架(推荐使用阿里 EasyExcel)

2）对于需要与数据库交互的校验、按照业务逻辑适当的使用缓存。用空间换时间

3）使用 values(),(),() 拼接长 SQL 一次插入多行数据

4）使用多线程插入数据，利用掉网络IO等待时间(推荐使用并行流，简单易用)

5）避免在循环中打印无用的日志

## 5、信息回写

上面使用了easyExcel导入文件内容到数据库，导入过程中某些行出错，我们可以把出错信息回写到excel对应行中，返回一个excel文件的下载路径给前端，用户下载后了解具体的行错误信息。

### POI方式导入

在ccs项目组的时候，导入库存账龄保留天数就是这样实现的。

1、controller层请求接口StockInvAccountAgeController.java

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

2、工具类ExcelImportUtils.java，导入功能可以使用easyExcel替代

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
		if (map == null || workbook == null || StringUtils.isEmpty(downloadFileName)) {
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

### EasyExcel方式导入

rcc 对账服务佣金点位配置异步导入excel

1、Controller层接口

```java
@Api(tags = "佣金点位配置")
@RestController
@RequestMapping("/pointConfig")
public class PointConfigController {
    @Autowired
    PointConfigService pointConfigService;
  
    @ApiOperation("导入")
    @PostMapping("/importFile.do")
    public Result<String> importFile(@RequestParam(name = "file") MultipartFile file) throws IOException {
        return Result.getSuccessResult(pointConfigService.importFile(file));
    }
}
```

2、业务层实现类`PointConfigServiceImpl`

```java
@Service
public class PointConfigServiceImpl extends ServiceImpl<PointConfigMapper, PointConfig> implements PointConfigService {
    @Autowired
    RccExcelServiceImpl rccExcelService;
  
  	// 新增点位数据
   @Override
    public void addBatch(String tenantCode,String userCode,List<PointConfigFormDTO> dtoList) {
        // 尽量保证校验唯一与插入的原子性，避免插入重复数据
        RLock rLock = redisLockHelper.tryLock("addPointConfig", 3, -1, TimeUnit.SECONDS);
        Assert.notNull(rLock,"获取锁失败,请稍后重试");
        try{
            // 1.唯一判断校验，满足以下场景
            // 相同模板+品牌+品类，允许商品编码空与非空的两种情况存在
            // 相同模板+品牌+品类+商品编码空的唯一
            // 相同模板+品牌+品类+商品编码非空的唯一
            // 过滤商品编码空的记录,再进行分组，如果每组的数量大于1，则存在重复
            Map<String, List<PointConfigFormDTO>> group1 = dtoList.stream().filter(dto -> StringUtils.isBlank(dto.getItemCode()))
                    .collect(Collectors.groupingBy(dto -> dto.getTemplateId()+"-"+dto.getBrandCode() + "-" + dto.getCategoryCode()));
            group1.forEach((k,v)->{
                Assert.isTrue(v.size()==1,"模板-品牌-品类("+k+")存在重复点位配置");
                // 判断数据库是否已存在该点位配置
                PointConfig one = this.getOne(new QueryWrapper<PointConfig>().select(PointConfig.ID)
                        .eq(PointConfig.TENANT_CODE,tenantCode).eq(PointConfig.TEMPLATE_ID,v.get(0).getTemplateId())
                        .eq(PointConfig.BRAND_CODE, v.get(0).getBrandCode()).eq(PointConfig.CATEGORY_CODE, v.get(0).getCategoryCode())
                        .and(wrap ->wrap.isNull(PointConfig.ITEM_CODE).or().eq(PointConfig.ITEM_CODE,"")).last("LIMIT 1"));
                if(Objects.nonNull(one)){
                    throw new RuntimeException("模板-品牌-品类("+k+")已存在点位配置，ID："+one.getId());
                }
            });
            // 过滤商品编码非空的记录,再进行分组，如果每组的数量大于1，则存在重复
            Map<String, List<PointConfigFormDTO>> group2 = dtoList.stream().filter(dto -> StringUtils.isNotBlank(dto.getItemCode()))
                    .collect(Collectors.groupingBy(dto -> dto.getTemplateId()+ dto.getBrandCode() + "-" + dto.getCategoryCode() + "-" + dto.getItemCode()));
            group2.forEach((k,v) ->{
                Assert.isTrue(v.size()==1,"模板-品牌-品类-商品("+k+")存在重复点位配置");
                PointConfig one = this.getOne(new QueryWrapper<PointConfig>().select(PointConfig.ID)
                        .eq(PointConfig.TENANT_CODE,tenantCode).eq(PointConfig.TEMPLATE_ID,v.get(0).getTemplateId())
                        .eq(PointConfig.BRAND_CODE, v.get(0).getBrandCode()).eq(PointConfig.CATEGORY_CODE, v.get(0).getCategoryCode())
                        .eq(PointConfig.ITEM_CODE,v.get(0).getItemCode()).last("LIMIT 1"));
                if(Objects.nonNull(one)){
                    throw new RuntimeException("模板-品牌-品类-商品("+k+")已存在点位配置，ID："+one.getId());
                }
            });

            // 2.插入
            transactionTemplate.execute(status -> {
                dtoList.forEach(dto ->{
                    PointConfig entity = new PointConfig();
                    BeanUtils.copyProperties(dto,entity);
                    entity.setId(DateUtils.generateId());
                    entity.setTenantCode(tenantCode);
                    entity.setCreatedBy(userCode);
                    entity.setUpdatedBy(userCode);
                    this.save(entity);

                    // 操作日志
                    RccOperateLog operateLog = new RccOperateLog().setTenantCode(tenantCode).setForeignKey(entity.getId().toString())
                            .setCreatedBy(userCode).setUpdatedBy(userCode).setOperateRecord("新增"+getFormString(dto));
                    operateLogGenMapper.insert(operateLog);
                });
                return true;
            });
        }finally {
            rLock.unlock();
        }
    }
  
    @Override
    public String importFile(MultipartFile file) {
        String tenantCode = SecurityUtils.getBaseRequest().getTenantCode();
        String userCode = SecurityUtils.getUserCode();
        Assert.isTrue(StringUtils.isNotBlank(tenantCode),"租户编码为空，请重新登录");
      // excel文件工作簿对象
        Workbook workbook = RccExcelUtils.getWorkBook(file);
      // 导出文件的名称
        String fileName = LocalDateTime.now().format(DateTimeFormatter.ofPattern(DateUtils.DF_YMDHMS_S))+file.getOriginalFilename();
      // 调用基础中心保存一条文件导出记录
        ComExportRecordGenDTO fileRecord = rccExcelService.saveFileRecord("pointConfig", fileName);

      // 异步导入
        CompletableFuture.runAsync(()->{
            try {
                PointConfigExcelListener excelListener=new PointConfigExcelListener().setTenantCode(tenantCode).setUserCode(userCode)
                        .setPointConfigService(this).setPointTemplateConfigService(pointTemplateConfigService)
                        .setWorkbook(workbook).setExcelService(rccExcelService).setFileRecord(fileRecord);
                // 读取第一个sheet,默认第1行是标题，忽略空行,文件流自动关闭
                EasyExcel.read(file.getInputStream(), PointConfigFormDTO.class,excelListener).registerConverter(new LocalDateTimeConverter()).sheet().doRead();
            } catch (IOException e) {
                log.error("pointConfig importFile error {}",e);
            }
        });
        return fileName;
    }
}
```

整个系统是微服务应用，先调用基础中心生成一条文件导出记录，生成成功，开启子线程异步导入excel，并返回文件名称给前端

```java
 ComExportRecordGenDTO fileRecord = rccExcelService.saveFileRecord("pointConfig", fileName);
```

实现类`RccExcelServiceImpl` 

```java
/**
 * @Author xiejw17
 * @Date 2021/11/15 17:45
 */
@Lazy
@Service
@Slf4j
public class RccExcelServiceImpl {
    @Value("${server.tomcat.basedir:/tmp/}")
    private String tempDir;

    @Resource
    private ExportRecordAtoFeign exportRecordAtoFeign;
    @Resource
    private ComExportRecordGenFeign exportRecordGenFeign;

    /**
     * 保存文件记录
     * @param prefix
     * @param fileName
     * @return
     */
    public ComExportRecordGenDTO saveFileRecord(String prefix, String fileName){
        LockExportRequest request = new LockExportRequest();
        request.setFilePrefix(prefix);
        request.setFileName(prefix+"-"+fileName);
        request.setCountLimit(5);
        ComExportRecordGenDTO record = this.exportRecordAtoFeign.lockExportRecord(CommonRequest.build(request)).getData();
        if (Objects.isNull(record)) {
            throw new BussinessException(ModuleErrorEnum.EXCEL_PROGRESS_LIMIT, new Object[0]);
        }
        return record;
    }

    /**
     * 文件上传OSS
     * @param workbook
     * @param record
     * @param startTime
     */
    public void uploadFile(Workbook workbook,ComExportRecordGenDTO record){
        File file = new File(this.tempDir +"/"+ record.getFileName());
        try{
            // 输出到文件
            FileOutputStream out = new FileOutputStream(file);
            workbook.write(out);
            out.close();
            //  文件上传OSS
            UploadMediaRequest request = new UploadMediaRequest();
            request.setSystem(GlobalConstants.OssSystem.PASS_OSS.getSystemSign());
            request.setBucket("mcsp-public-store");
            UploadResponse response = MediaUploadHelper.upload(file, request);
            record.setFileSize(file.length());
            if(Objects.isNull(response)){
                record.setExportStatus(DictEnum.ExportStatus.FAIL.getStatus());
                record.setRemark("上传OSS返回空");
            }else{
                record.setDownload(response.getUrl());
                record.setExportStatus(DictEnum.ExportStatus.COMPLETED.getStatus());
                record.setRemark("success");
            }
        }catch (Exception e){
            record.setExportStatus(DictEnum.ExportStatus.FAIL.getStatus());
            record.setRemark("导入结果文件上传OSS失败："+e.getMessage());
        }finally {
            file.delete();
        }
    }

    // 更新文件记录
    public void updateFileRecord(ComExportRecordGenDTO record,int tryTimes) throws InterruptedException{
        try{
            exportRecordGenFeign.updateComExportRecord(CommonRequest.build(record));
        }catch (Exception e ){
            tryTimes = tryTimes -1;
            if(tryTimes<=0){
                log.error("文件记录<"+record.getFileName()+">更新失败："+e.getMessage());
            }else{
                // 2秒后重试
                try {
                    TimeUnit.SECONDS.sleep(2);
                    updateFileRecord(record,tryTimes);
                } catch (InterruptedException interruptedException) {
                    log.error("文件记录<"+record.getFileName()+">更新失败："+e.getMessage());
                    throw interruptedException;
                }
            }
        }
    }
}
```

feign接口

```java
@Component
@FeignClient(
    value = "base-atomic-service",
    path = "/base-atomic"
)
public interface ExportRecordAtoFeign {
    @PostMapping({"/export/lockExportRecord.ato.do"})
    Result<ComExportRecordGenDTO> lockExportRecord(@RequestBody CommonRequest<LockExportRequest> request);

    @PostMapping({"/export/exportRecordList.ato.do"})
    PaginationResult<List<ExportRecordAtoRespDTO>> exportRecordList(@RequestBody PaginationRequest<ExportQueryAtoReqDTO> request);
}

@Component
@FeignClient(
    value = "base-atomic-service",
    path = "/base-atomic"
)
public interface ComExportRecordGenFeign {
    @PostMapping({"/comExportRecord/updateComExportRecord.gen.do"})
    Result<Boolean> updateComExportRecord(@RequestBody CommonRequest<ComExportRecordGenDTO> request);
}
```

这里整合EasyExcel导入文件，需要继承`AnalysisEventListener`重写数据处理监听器

```java
@Slf4j
public class PointConfigExcelListener extends AnalysisEventListener<PointConfigFormDTO> {
    private PointConfigService pointConfigService;
    private PointTemplateConfigService pointTemplateConfigService;
    private RccExcelServiceImpl excelService;
    private ComExportRecordGenDTO fileRecord;
    private String tenantCode;
    private String userCode;
    private Long startTime;
    //private static final int BATCH_COUNT = 1;
    private List<PointConfigFormDTO> configList;
    private Map<String,Long> templateMap;
    private Map<Integer,String> failMessageMap; // 行导入失败的原因
    Workbook workbook;

    public PointConfigExcelListener(){
        this.configList = new ArrayList<>();
        this.templateMap = new HashMap<>();
        this.failMessageMap = new HashMap<>();
        startTime = System.currentTimeMillis();
    }

    public PointConfigExcelListener setTenantCode(String tenantCode){
        this.tenantCode = tenantCode;
        return this;
    }

    public PointConfigExcelListener setUserCode(String userCode){
        this.userCode = userCode;
        return this;
    }

    public PointConfigExcelListener setPointConfigService(PointConfigService pointConfigService) {
        this.pointConfigService = pointConfigService;
        return this;
    }

    public PointConfigExcelListener setPointTemplateConfigService(PointTemplateConfigService pointTemplateConfigService) {
        this.pointTemplateConfigService = pointTemplateConfigService;
        return this;
    }

    public PointConfigExcelListener setExcelService(RccExcelServiceImpl excelService) {
        this.excelService = excelService;
        return this;
    }

    public PointConfigExcelListener setWorkbook(Workbook workbook) {
        this.workbook = workbook;
        return this;
    }

    public PointConfigExcelListener setFileRecord(ComExportRecordGenDTO fileRecord) {
        this.fileRecord = fileRecord;
        return this;
    }

    /**
     * 每条数据解析都会来调用
     */
    @Override
    public void invoke(PointConfigFormDTO data, AnalysisContext context) {
        int rowIndex = context.readRowHolder().getRowIndex();
        if(StringUtils.isBlank(data.getBrandCode())||StringUtils.isBlank(data.getCategoryCode())){
            failMessageMap.put(rowIndex,"品牌编码、品类编码必填");
            //context.readRowHolder().getCellMap().put(10, new CellData("品牌编码、品类编码必填"));
            return;
        }
        // 检验品牌、品类的正确性

        // 查询模板id
        Long templateId = Optional.ofNullable(templateMap.get(data.getTemplateName())).orElseGet(()->{
            PointTemplateConfig one = pointTemplateConfigService.getOne(new QueryWrapper<PointTemplateConfig>()
                    .select(PointTemplateConfig.ID)
                    .eq(PointTemplateConfig.TENANT_CODE, tenantCode).eq(PointTemplateConfig.TEMPLATE_NAME, data.getTemplateName())
                    .last("limit 1"));
            return Optional.ofNullable(one).map(r->r.getId()).orElse(null);
        });
        if(Objects.isNull(templateId)){
            failMessageMap.put(rowIndex,"当前租户没配置模板："+data.getTemplateName());
            return;
        }
        templateMap.put(data.getTemplateName(),templateId);
        data.setTemplateId(templateId);
        // 现在每读一条数据保存一次，因为要获取成功失败信息
        configList.add(data);
        try{
            pointConfigService.addBatch(tenantCode,userCode,configList);
        }catch (Exception e){
            failMessageMap.put(rowIndex,e.getMessage());
        }
        configList.clear();
    }

    /**
     * 所有数据解析完后调用
     */
    @SneakyThrows
    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
        // 每条记录的导入结果更新到原文件
        RccExcelUtils.updateMessage4Excel(workbook,failMessageMap);
        // 文件上传OSS
        excelService.uploadFile(workbook,fileRecord);
        // 更新文件记录
        this.fileRecord.setCostTime(System.currentTimeMillis()-startTime);
        excelService.updateFileRecord(fileRecord,3);
        // 一定要关闭文件流
        workbook.close();
    }

    /**
     * 每条数据转换时的异常处理，默认会抛出异常，停止读取，这里不抛异常，将异常信息保存下来，继续读取下一行
     * @param exception
     * @param context
     * @throws Exception
     */
    @Override
    public void onException(Exception exception, AnalysisContext context) throws Exception {
        failMessageMap.put(context.readRowHolder().getRowIndex(),"数据解析异常"+exception.getMessage());
    }
}
```

工具类`RccExcelUtils`

```java
public class RccExcelUtils {
      @SneakyThrows
    public static Workbook getWorkBook(MultipartFile file) {
        String originalFilename = file.getOriginalFilename();
        if(null == originalFilename){
            throw new BussinessException(ErrorEnum.FAIL, "excel文件名称不能为空");
        }
        String fileName = originalFilename.toLowerCase();
        if(fileName.endsWith(".xls")){
            return new HSSFWorkbook(file.getInputStream());
        }else if(fileName.endsWith(".xlsx")){
            return new XSSFWorkbook(file.getInputStream());
        }
        throw new RuntimeException("excel文件格式错误");
    }

    public static void updateMessage4Excel(Workbook workbook, Map<Integer, String> map) {
        Sheet sheet = workbook.getSheetAt(0);
        int columnIndex = 0;
        // 第一行是标题
        Row row = sheet.getRow(sheet.getFirstRowNum());
        columnIndex = row.getLastCellNum();
        Cell cell = row.createCell(columnIndex);
        cell.setCellValue("提示");
        for (int i = sheet.getFirstRowNum() + 1; i < sheet.getLastRowNum() + 1; i++) {
            row = sheet.getRow(i);
            cell = row.createCell(columnIndex);
            cell.setCellValue(StringUtils.defaultString(map.get(i), "success"));
        }
    }
}
```

## 6、异步分页导出

1、Controller层接口

```java
@Api(tags = "电商对账-销售发票核对")
@RestController
@RequestMapping("/reconciliation/saleInvoice")
public class SaleInvoiceRecController {
      @ApiOperation("导出对账明细,返回导出的任务id")
    @PostMapping("/exportDetails.do")
    public Result<String> exportDetails(@RequestBody PaginationRequest<FindSaleInvoiceDetailReqDTO> pageRequest){
        String downloadId = saleInvoiceRecFacade.exportDetails(pageRequest);
        return Result.getSuccessResult(downloadId);
    }
}
```

2、业务层实现类

```java
@Slf4j
@Service
public class SaleInvoiceRecFacadeImpl implements SaleInvoiceRecFacade {
    @Override
    public String exportDetails(PaginationRequest<FindSaleInvoiceDetailReqDTO> pageRequest) {
        String tenantCode = pageRequest.getHeadParams().getTenantCode();
        Assert.isTrue(StringUtils.isNotBlank(tenantCode) &&
                StringUtils.isNotBlank(pageRequest.getRestParams().getPeriod()),"租户编码与账期必填");
        ExcelServiceImpl.ExcelParam param = new ExcelServiceImpl.ExcelParam("销售发票核对明细导出", "租户编码"+tenantCode);
        param.addHead("checkBatchNum","核对批次号");
        param.addHead("period","账期");
        param.addHead("merchantCode","商户编码");
        param.addHead("merchantName","商户名称");
        param.addHead("customerCode","客户编码");
        param.addHead("customerName","客户名称");
        param.addHead("shopCode","店铺编码");
        param.addHead("shopName","店铺名称");
        param.addHead("outTradeNo","订单号");
        param.addHead("settleAmount","本期结算金额");
        param.addHead("settleTaxAmount","本期结算税额");
        param.addHead("invoiceAmount","本期开票金额");
        param.addHead("invoiceTaxAmount","本期开票税额");
        param.addHead("diffAmount","本期差异");
        param.addHead("taxDiffAmount","本期税额差异");
        param.addHead("startDiffAmount","期初差异");
        param.addHead("startTaxDiffAmount","期初税额差异");
        param.addHead("endDiffAmount","期末差异");
        param.addHead("endTaxDiffAmount","期末税额差异");
        param.addHead("diffJudgeTypeName","差异判断");

        String downloadId = excelService.asyncExport(pr ->{
            List<SaleInvoiceDetailRespDTO> list = listDetails(pr);
            list.parallelStream().forEach(dto->{
                dto.setDiffJudgeTypeName(getDiffJudgeTypeName(dto.getDiffJudgeType()));
            });
            return PaginationResult.getSuccessResult(list,pr.getPagination());
        },pageRequest,param);
        return downloadId;
    }
}
```

主要工具类`ExcelServiceImpl`

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;

@Lazy
@Service
public class ExcelServiceImpl {
    private static final Logger log = LoggerFactory.getLogger(ExcelServiceImpl.class);
    @Resource
    private ComExportRecordGenFeign exportRecordGenFeign;
    @Resource
    private ExportRecordAtoFeign exportRecordAtoFeign;
    private static final String EXCEL_SUFFIX = ".xlsx";
    private static final int COUNT_PER_PERSON = 5;
    private static final int PROGRESS_TIMEOUT = 3600;
    private static final DateTimeFormatter DF = DateTimeFormatter.ofPattern("yyyyMMddHHmmssSSS");
    @Value("${server.tomcat.basedir:/tmp/}")
    private String tempDir;
    private static final ObjectMapper objectMapper = new ObjectMapper();
  
    public String asyncExport(Function<PaginationRequest, PaginationResult<List>> function, PaginationRequest request, ExcelServiceImpl.ExcelParam param) {
        this.initParam(param, false);
        ExecutorConfig.TASK_EXECUTOR.execute(() -> {
            try {
                String fileName = this.exportEasy(function, request, param);
                this.exportEasySuccess(param, fileName);
            } catch (InterruptedException var10) {
                log.warn("导出取消，任务ID: [{0}]", param.record.getId());
                this.exportCancel(param, "手动取消任务");
                Thread.currentThread().interrupt();
            } catch (NullPointerException var11) {
                log.error("导出失败：空指针异常", var11);
                this.exportFail(param, "空指针异常");
            } catch (Exception var12) {
                log.error("导出失败：", var12);
                this.exportFail(param, StringUtils.substring(this.getExceptionMessage(var12), 0, 1000));
            } finally {
                ExportProgressHelper.clearProgress(param.record.getId());
            }

        });
        return param.record.getId().toString();
    }
  
    private void initParam(ExcelServiceImpl.ExcelParam param, boolean async) {
        param.async = async;
        param.startTime = System.currentTimeMillis();
        param.fileName = param.docName + "-" + LocalDateTime.now().format(DF) + ".xlsx";
        param.record = (ComExportRecordGenDTO)this.exportRecordAtoFeign.lockExportRecord(CommonRequest.build(this.getRecord(param.docName, param.fileName, 5))).getData();
        if (param.record == null) {
            throw new BussinessException(ModuleErrorEnum.EXCEL_PROGRESS_LIMIT, new Object[0]);
        } else {
          // 初始文件导出进度到redis
            ExportProgressHelper.setProgress(param.record.getId(), "0", 3600L, TimeUnit.SECONDS);
        }
    }
  
  public String exportEasy(Function<PaginationRequest, PaginationResult<List>> function, PaginationRequest request, ExcelServiceImpl.ExcelParam param) throws InterruptedException {
        Pagination pagination = request.getPagination(1, 5000, true);
        String fileName = this.tempDir + "/" + param.fileName;
        String sheetName = param.sheetName;
        Long progress = 0L;
        int lineCount = 0;
        int sheetNum = 0;
        String[] keys = (String[])param.head.keySet().toArray(new String[param.head.size()]);
        ExcelWriter excelWriter = EasyExcel.write(fileName).build();
        WriteSheet writeSheet = ((ExcelWriterSheetBuilder)EasyExcel.writerSheet(sheetNum, sheetName).needHead(false)).build();
        List<List<String>> headList = new LinkedList();
        headList.add(new ArrayList(param.head.values()));
        excelWriter.write(headList, writeSheet);

        while(progress < 100L && !Thread.currentThread().isInterrupted()) {
            request.setPagination(pagination);
            PaginationResult<List> result = (PaginationResult)function.apply(request);
            if (CollectionUtils.isEmpty((Collection)result.getData())) {
                ExportProgressHelper.setProgress(param.record.getId(), "100", 3600L, TimeUnit.SECONDS);
                break;
            }

            if (lineCount + ((List)result.getData()).size() > 1000000) {
                ++sheetNum;
                sheetName = param.sheetName + "-" + sheetNum;
                writeSheet = ((ExcelWriterSheetBuilder)EasyExcel.writerSheet(sheetNum, sheetName).needHead(false)).build();
                excelWriter.write(headList, writeSheet);
                lineCount = 0;
            }
          // 导出数据写入文件
            lineCount = writeDataEasy(excelWriter, writeSheet, keys, (ArrayNode)objectMapper.valueToTree(result.getData()), lineCount);
            if (pagination.getCount() == 0L) {
                pagination.setCount(result.getPagination().getCount());
            }

            if (pagination.getCount() > 0L) {
                progress = (long)(pagination.getPageNo() * pagination.getPageSize() * 100) / pagination.getCount();
                ExportProgressHelper.setProgress(param.record.getId(), progress.toString(), 3600L, TimeUnit.SECONDS);
            }

            pagination.setCountFlag(false);
            pagination.next();
            Thread.sleep(5L);
        }

        excelWriter.finish();
        return fileName;
    }
  
      public static int writeDataEasy(ExcelWriter excelWriter, WriteSheet writeSheet, String[] keys, ArrayNode datas, int lineCount) {
        List<List> dataList = new LinkedList();

        for(int i = 0; i < datas.size(); ++i) {
            List list = new LinkedList();

            for(int j = 0; j < keys.length; ++j) {
                String key = keys[j];
                String value = getValue(key, datas.get(i));
                list.add(value);
            }

            dataList.add(list);
        }

        excelWriter.write(dataList, writeSheet);
        return lineCount + datas.size();
    }
  
      public void exportEasySuccess(ExcelServiceImpl.ExcelParam param, String fileName) {
        File file = new File(fileName);
        try {
            UploadMediaRequest request = new UploadMediaRequest();
            request.setSystem(OssSystem.PASS_OSS.getSystemSign());
            request.setBucket(StringUtils.isBlank(param.bucket) ? "mcsp-public-store" : param.bucket);
            UploadResponse response = MediaUploadHelper.upload(file, request);
            param.record.setDownload(response == null ? "" : response.getUrl());
            param.record.setFileSize(file.length());
            param.record.setCostTime(System.currentTimeMillis() - param.startTime);
            param.record.setExportStatus(null == response ? ExportStatus.FAIL.getStatus() : ExportStatus.COMPLETED.getStatus());
            param.record.setRemark(param.getRemark());
            this.exportRecordGenFeign.updateComExportRecord(CommonRequest.build(param.record));
        } finally {
            FileUtil.delteTempFile(file);
        }
    }
  
      public void exportCancel(ExcelServiceImpl.ExcelParam param, String msg) {
        param.record.setCostTime(System.currentTimeMillis() - param.startTime);
        param.record.setExportStatus(ExportStatus.CANCEL.getStatus());
        param.record.setRemark(msg);
        this.exportRecordGenFeign.updateComExportRecord(CommonRequest.build(param.record));
    }
  
      public void exportFail(ExcelServiceImpl.ExcelParam param, String msg) {
        param.record.setCostTime(System.currentTimeMillis() - param.startTime);
        param.record.setExportStatus(ExportStatus.FAIL.getStatus());
        param.record.setRemark(msg);
        this.exportRecordGenFeign.updateComExportRecord(CommonRequest.build(param.record));
    }
  
  public static class ExcelParam {
        private String docName;
        private String sheetName;
        private LinkedHashMap<String, String> head;
        private String bucket;
        private String remark;
        private Long startTime;
        private String fileName;
        private ComExportRecordGenDTO record;
        private boolean async;

        public ExcelParam(String docName, String sheetName) {
            this.docName = docName;
            this.sheetName = sheetName;
        }

        public ExcelServiceImpl.ExcelParam setDocName(String docName) {
            this.docName = docName;
            return this;
        }

        public ExcelServiceImpl.ExcelParam setSheetName(String sheetName) {
            this.sheetName = sheetName;
            return this;
        }

        public ExcelServiceImpl.ExcelParam setBucket(String bucket) {
            this.bucket = bucket;
            return this;
        }

        public ExcelServiceImpl.ExcelParam setRemark(String remark) {
            this.remark = remark;
            return this;
        }

        public String getRemark() {
            return this.remark;
        }

        public ExcelServiceImpl.ExcelParam addHead(String colunm, String name) {
            this.head = (LinkedHashMap)Optional.ofNullable(this.head).orElse(new LinkedHashMap());
            this.head.put(colunm, name);
            return this;
        }
    }
} 
```

用到了EasyExcel 依赖与 jackson-databind依赖

![](\assets\images\2021\springcloud\jackson-databind.png)

DTO对象

```java
public class ComExportRecordGenDTO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String filePrefix;
    private String fileName;
    private String download;
    private Long fileSize;
    private Long costTime;
    private Integer exportStatus;
}
```

微服务Feign接口`ExportRecordAtoFeign`

```java
@Component
@FeignClient(
    value = "base-atomic-service",
    path = "/base-atomic"
)
public interface ExportRecordAtoFeign {
    @PostMapping({"/export/lockExportRecord.ato.do"})
    Result<ComExportRecordGenDTO> lockExportRecord(@RequestBody CommonRequest<LockExportRequest> request);

    @PostMapping({"/export/exportRecordList.ato.do"})
    PaginationResult<List<ExportRecordAtoRespDTO>> exportRecordList(@RequestBody PaginationRequest<ExportQueryAtoReqDTO> request);
}

@Component
@FeignClient(
    value = "base-atomic-service",
    path = "/base-atomic"
)
public interface ComExportRecordGenFeign {
      @PostMapping({"/comExportRecord/updateComExportRecord.gen.do"})
    Result<Boolean> updateComExportRecord(@RequestBody CommonRequest<ComExportRecordGenDTO> request);
}
```

导出文件进度工具类`ExportProgressHelper`

```java
public class ExportProgressHelper {
    private static final String EXPORT_PROGRESS_PREFIX = "EXPORT_PROGRESS:";

    public ExportProgressHelper() {
    }

    public static void clearProgress(Long id) {
        InitRedisConfig.BASE_REDIS_TEMPLATE.delete("EXPORT_PROGRESS:" + id);
    }

    public static void setProgress(Long id, String progress, long timeout, TimeUnit timeUnit) {
        InitRedisConfig.BASE_REDIS_TEMPLATE.opsForValue().set("EXPORT_PROGRESS:" + id, "0", timeout, TimeUnit.SECONDS);
    }

    public static String getProgress(Long jobId) {
        return (String)InitRedisConfig.BASE_REDIS_TEMPLATE.opsForValue().get("EXPORT_PROGRESS:" + jobId);
    }

    public static Map<Long, String> listProgress(List<Long> jobIds) {
        Map<Long, String> result = new HashMap();
        List<String> keys = new ArrayList();
        Iterator var3 = jobIds.iterator();

        while(var3.hasNext()) {
            Long jobId = (Long)var3.next();
            keys.add("EXPORT_PROGRESS:" + jobId);
        }

        Map<String, String> data = mget(keys);
        Iterator var7 = jobIds.iterator();

        while(var7.hasNext()) {
            Long jobId = (Long)var7.next();
            result.put(jobId, data.getOrDefault("EXPORT_PROGRESS:" + jobId, ""));
        }

        return result;
    }

    private static Map<String, String> mget(List<String> keys) {
        Map<String, String> result = new HashMap();
        List<String> data = InitRedisConfig.BASE_REDIS_TEMPLATE.opsForValue().multiGet(keys);

        for(int i = 0; i < keys.size(); ++i) {
            result.put(keys.get(i), data.get(i));
        }

        return result;
    }
}
```

美的云上传OSS工具类`MediaUploadHelper`

```java
public class MediaUploadHelper {
    private static final Logger log = LoggerFactory.getLogger(MediaUploadHelper.class);
    private static final Map<String, OssUploadInterface> EXECUTOR_MAP = new ConcurrentHashMap();

    public MediaUploadHelper() {
    }
  
      public static UploadResponse upload(File file, UploadMediaRequest req) {
        UploadResponse response = null;

        try {
            if (StringUtils.isBlank(req.getSystem())) {
                throw new BussinessException(ModuleErrorEnum.UPLOAD_EXECUTOR_EMPTY, new Object[0]);
            } else {
                List<OssUploadInterface> executors = getExecutors(req.getSystem());

                OssUploadInterface executor;
                for(Iterator var4 = executors.iterator(); var4.hasNext(); response = executor.upload(file, (String)null, req.getBucket())) {
                    executor = (OssUploadInterface)var4.next();
                }

                return response;
            }
        } catch (IllegalAccessException | InstantiationException var6) {
            throw new SystemException("OssUploadInterface init error");
        }
    }
}
```



