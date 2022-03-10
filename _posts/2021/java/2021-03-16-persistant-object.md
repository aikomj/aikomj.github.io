---
layout: post
title: java 中那些持久化对象persistant object
category: java
tags: [java]
keywords: java
excerpt: PO,BO,VO,DTO,DAO
lock: noneed
---

## 1、持久化对象介绍

- PO(Persistant Object) 持久对象

  用于表示数据库中的一条记录映射成的 java 对象。PO 仅仅用于表示数据，没有任何数据操作。通常遵守 Java Bean 的规范，拥有 getter/setter 方法

- BO(Business Object) 业务对象

  BO 包括了业务逻辑，常常封装了对 DAO、RPC 等的调用，可以进行 PO 

  与 VO/DTO 之间的转换。BO 通常位于业务层，要区别于直接对外提供服务的服务层：BO 提供了基本业务单元的基本业务操作，在设计上属于被服务层业务流程调用的对象，一个业务流程可能需要调用多个 BO 来完成。

  比如一个简历，有教育经历、工作经历、社会关系等等。

  我们可以把教育经历对应一个PO，工作经历对应一个PO，社会关系对应一个PO。建立一个对应简历的BO对象处理简历，每个BO包含这些PO。

- VO(Value Object) 值对象

  用于表示一个与前端进行交互的 java 对象。一般，这里的 VO 只包含前端需要展示的数据即可，对于前端不需要的数据，比如数据创建和修改的时间等字段，出于减少传输数据量大小和保护数据库结构不外泄的目的，不应该在 VO 中体现出来。通常遵守 Java Bean 的规范，拥有 getter/setter 方法

- DTO(Data Transfer Object) 数据传输对象

  用于表示一个数据传输对象。DTO 通常用于不同服务或服务不同分层之间的数据传输。DTO 与 VO 概念相似，并且通常情况下字段也基本一致。但 DTO 与 VO 又有一些不同，这个不同主要是设计理念上的，比如 API 服务需要使用的 DTO 就可能与 VO 存在差异。通常遵守 Java Bean 的规范，拥有 getter/setter 方法

- DAO(Data access object) 数据访问对象

  用于表示一个数据访问对象。使用 DAO 访问数据库，包括插入、更新、删除、查询等操作，与 PO 一起使用。DAO 一般在持久层，完全封装数据库操作。

  

  **POJO**， Plain Ordinary Java Object 的缩写，表示一个简单 java 对象。上面说的 PO、VO、DTO 都是典型的 POJO。而 DAO、BO 一般都不是 POJO，只提供一些调用方法。

![](/assets/images/2021/javabase/java-pojo.png)



​		我们以一个实例来探讨下 POJO 的使用。假设我们有一个面试系统，数据库中存储了很多面试题，通过 web 和 API 提供服务。可能会做如下的设计：

数据表：表中的面试题包括编号、题目、选项、答案、创建时间、修改时间；

PO：包括题目、选项、答案、创建时间、修改时间；

VO：题目、选项、答案、上一题URL、下一题URL；

DTO：编号、题目、选项、答案、上一题编号、下一题编号；

DAO：数据库增删改查方法；

BO：业务基本操作。

可以看到，进行 POJO 划分后，我们得到了一个设计良好的架构，各层数据对象的修改完全可以控制在有限的范围内。

## 2、BO举例

之前在xx公司做图表指标分析，就是用BO类建立模型对象进行业务指标统计的

```java
/** 
 * @author 作者 : aknb206
 * @version 创建时间：2017年8月17日 上午8:34:25 
 * 碎片率，返工率的模型类
 */
public class SuipMap {
	private Map<String, GroupMap> suipMap = new LinkedHashMap<String, GroupMap>(4);
	private String company;
	private List<GroupCategories> categories= new ArrayList<GroupCategories>();// 分组类别
	private Map<String, GroupCategories> categoriesMap = new LinkedHashMap<String, GroupCategories>();
	private List<Object> seriesDatas= new ArrayList<Object>();        // 分组类数据
	
	public List<Object> getSeriesDatas() {
		return seriesDatas;
	}
	public void setSeriesDatas(List<Object> seriesDatas) {
		this.seriesDatas = seriesDatas;
	}
	public Map<String, GroupCategories> getCategoriesMap() {
		return categoriesMap;
	}
	public void setCategoriesMap(Map<String, GroupCategories> categoriesMap) {
		this.categoriesMap = categoriesMap;
	}
	
	public void setSuipMap(Map<String, GroupMap> suipMap) {
		this.suipMap = suipMap;
	}
	public List<GroupCategories> getCategories() {
		return categories;
	}
	public void setCategories(List<GroupCategories> categories) {
		this.categories = categories;
	}
	
	public Map<String, GroupMap> getSuipMap() {
		return suipMap;
	}
	
	public String getCompany() {
		return company;
	}


	public void setCompany(String company) {
		this.company = company;
	}


	public void groupByLevel(List<SuipData> ls,String level){
		SuipData data=null;
		for (int i = 0, len = (ls == null ? 0 : ls.size()); i < len; i++) {
			data = ls.get(i);
			// 对该行数据进行分组计算
			if(level.equals("company")){
				add2GroupMap(data.getCompany(),data,false);
			}else if(level.equals("factory")){
				add2GroupMap(data.getFactory(),data,false);
			}else if(level.equals("dept")){
				add2GroupMap(data.getDept(), data,false);
			}else if(level.equals("line")){
				add2GroupMap(data.getLine(), data,false);
			}else if(level.equals("gongxu")){
				add2GroupMap(data.getGongxu()+"-"+data.getGongxuName(), data,true);
			}else if(level.equals(KpiConstant.product)){
				add2GroupMapProduct(data.getProduct(),data);
			}else if(level.equals(KpiConstant.suppliers)){
				add2GroupMapProduct(data.getSupplier(),data);
			}else if(level.equals(KpiConstant.worknum)){
				add2GroupMapWN(data.getWorkNum(),data);
			}else if(level.equals(KpiConstant.batch)){
				add2GroupMapBC(data.getSupplier(),data);
			}
		}
	}
	
	public void add2GroupMap(String groupName,SuipData data,Boolean gongxu){
		if(data!=null && groupName != null && groupName.length() > 0){
			GroupMap groupMap=suipMap.get(groupName);
			if(groupMap==null){
				groupMap=new GroupMap();
				suipMap.put(groupName,groupMap);
			}
			if(gongxu){
				groupMap.addSpDataGx(data);	
			}else{
				groupMap.addSpData(data,data.getProduct(),true);
			}	
		}
	}
	
	public void add2GroupMapProduct(String groupName,SuipData data){
		if(data!=null && groupName != null && groupName.length() > 0){
			GroupMap groupMap=suipMap.get(groupName);
			if(groupMap==null){
				groupMap=new GroupMap();
				suipMap.put(groupName,groupMap);
			}
			if("1000".equals(data.getCompany())){
				groupMap.addSpData(data,"",false);
			}else if("2000".equals(data.getCompany())){
				groupMap.addSpData(data,data.getProduct(),true);
			}
							
		}
	}
	//二级模板(工单视点)
	public void add2GroupMapWN(String groupName,SuipData data){
		if(data!=null && groupName != null && groupName.length() > 0){
			GroupMap groupMap=suipMap.get(groupName);
			if(groupMap==null){
				groupMap=new GroupMap();
				suipMap.put(groupName,groupMap);
			}
			groupMap.addSpData(data,data.getWorkNum(),true);
		}
	}
	
	//二级模板(批次视点)
	public void add2GroupMapBC(String groupName,SuipData data){
		if(data!=null && groupName != null && groupName.length() > 0){
			GroupMap groupMap=suipMap.get(groupName);
			if(groupMap==null){
				groupMap=new GroupMap();
				suipMap.put(groupName,groupMap);
			}
			groupMap.addSpData(data,data.getBatch(),true);
		}
	}
	
	public void groupByLevelQs(List<SuipData> ls,String[] dates,String level){
		SuipData data=null;
		for (int i = 0, len = (ls == null ? 0 : ls.size()); i < len; i++) {
			data = ls.get(i);
			// 对该行数据进行分组计算
			if(level.equals("company")){
				add2GroupMapQs(data.getCompany(),data,dates,false);
			}else if(level.equals("factory")){
				add2GroupMapQs(data.getFactory(),data,dates,false);
			}else if(level.equals("dept")){
				add2GroupMapQs(data.getDept(), data,dates,false);
			}else if(level.equals("line")){
				add2GroupMapQs(data.getLine(), data,dates,false);
			}else if(level.equals("gongxu")){
				add2GroupMapQs(data.getGongxu()+"-"+data.getGongxuName(),data,dates,true);
			}else if(level.equals(KpiConstant.product)){
				add2GroupMapQs(data.getProduct(), data,dates,false);
			}else if(level.equals(KpiConstant.suppliers)){
				add2GroupMapQs(data.getSupplier(), data,dates,false);
			}			
		}
	}
	
	public void add2GroupMapQs(String groupName,SuipData data,String[] dates,boolean gongxu){
		if(data!=null && groupName != null && groupName.length() > 0){
			GroupMap groupMap=suipMap.get(groupName);
			if(groupMap==null){
				groupMap=new GroupMap();
				suipMap.put(groupName,groupMap);
			}
			groupMap.addSpDataQs(data,dates,gongxu);
		}
	}
	
	public void createCategories(String spulierCode, GroupMap groupMap) {
		// TODO Auto-generated method stub
		if(spulierCode!=null && groupMap!=null){
			GroupCategories gcs=categoriesMap.get(spulierCode);
			if(gcs==null){
				gcs= new GroupCategories();
				gcs.setName(groupMap.getName());
				categoriesMap.put(spulierCode, gcs);
				//List<String> cgs=new ArrayList<String>();
				
				for(Iterator<Map.Entry<String, ProductMap>> iter=groupMap.getpMap().entrySet().iterator();iter.hasNext();){
					Map.Entry<String, ProductMap> entry=iter.next();
					ProductMap pMap=entry.getValue();
					String pKey=entry.getKey();
					//cgs.add(pKey);
					gcs.addCategory(pKey);
					if (pMap.getTotalSize() == 0) {
						seriesDatas.add(0d);
					}else {
						Double percent = (double)(pMap.getSuipSize()) / (double)(pMap.getTotalSize()) * 100;
						seriesDatas.add(percent);
					}
				}	
				//gcs.setCategories(cgs);
				categories.add(gcs);
			}
		}
	}
}
```

PO类

```java
public class SuipData {
	private String company;     // 公司编码
	private String companyName; // 公司名称
	private String factory;//工厂编码
	private String factoryName;//工厂名称
	private String dept;        // 部门编码
	private String deptName;    // 部门名称
	private String line;        // 产线编码
	private String lineName;    // 产线名称
	private String gongxu;		//工序编号
	private String gongxuName;	//工序描述
	private String product;     // 产品
	private String supplier;   //供应商编码
	private String supplierName;//供应商名称
	private String date;        // 日期
	private int zcSuipSize; //制程碎片数
	private int rkSuipSize; //入库碎片数（O级）
	private int rkTotalSize;  // 入库总数
	private int totalSize; //总数
	private int suipSize; //碎片数
	private double percent;     //碎片率
	private String workNum;//工单号
	private String batch;//批次
}
```

