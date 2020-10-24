---
layout: post
title: Postman后端接口测试工具
category: tool
tags: [tool]
keywords: postman
excerpt: 项目没有集成Swagger2在线接口测试，那postman就是第二选择了
lock: noneed
---

## 1、测试上传文件

![](\assets\images\tools\postman-test-file.png)

输入url：http://127.0.0.1:8081/uploadfile

1、选择post方式

2、选择body

​	选择form-data，text改为file

​	输入key：file  ，value：选择文件

3、send即可



### 数组

正常的方式是，前端POST请求，content-type是application/json，body的请求参数格式

```json
{
  "roleIds": [3928,3930]
}
```

后端是这样解析参数的

```java
	@SysLog("删除角色")
	@PostMapping("/delete")
	@RequiresPermissions("sys:role:delete")
	public R delete(@RequestBody Long[] roleIds){
		sysRoleService.deleteBatch(roleIds);
		return R.ok();
	}
```

但有些后端人员会使用@RequestParam 解析数组（不规范），如下：

```java
@ResponseBody
@PostMapping("/batchPublish")
public R batchPublish(@RequestParam("ids[]")List<Long> idList){
```

这时候前端数组参数是要放在url的param上的，而不是放在body里，content-type是text/html

下面使用postman测试的请求截图

![](\assets\images\tools\requestparam-array.jpg)



