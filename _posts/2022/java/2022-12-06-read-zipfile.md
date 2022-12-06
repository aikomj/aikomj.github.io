---
layout: post
title: 读取zip压缩文件内容
category: java
tags: [java]
keywords: java
excerpt: ZipInputStream直接读取压缩包内容，可读取UNI小程序获取版本信息
lock: noneed
---

## 1、前言

工作中，后台管理端需要读取uni小程序代码包文件获取版本信息，然后对小程序应用进行发布上架，代码包是wgt格式文件，可以修改为zip后缀进行解压读取

![image-20221206153205780](\assets\images\2022\java\uni-applet.png)

### ZipInputStream读取

java 可用`ZipInputStream`读取zip文件，参考 [https://geek-docs.com/java/java-tutorial/zipinputstream.html](https://geek-docs.com/java/java-tutorial/zipinputstream.html)

读取uni小程序代码包文件：

```java
    public static void main(String[] args) throws IOException {
        // 获取文件输入流
        String fileName = "C:\\jacob\\任务\\任务19-帷幄APP\\__UNI__0874BA0.wgt";
        try(FileInputStream fis = new FileInputStream(fileName); BufferedInputStream bis = new BufferedInputStream(fis); ZipInputStream zis = new ZipInputStream(bis);){
            // ZIP文件入口
            ZipEntry ze = null;
            while ((ze=zis.getNextEntry())!=null){
                // 找到manifest.json文件
                if(!ze.isDirectory() && ze.getName().equals("manifest.json")){
                    System.out.println("找到manifest.json文件");
                    // 读取
                    BufferedReader br = new BufferedReader(new InputStreamReader(zis));
                    StringBuffer stringBuffer = new StringBuffer();
                    String line;
                    // 内容不为空，输出
                    while ( (line = br.readLine())!=null){
                        stringBuffer.append(line);
                    }
                    System.out.println(stringBuffer.toString());
                    // 转换json对象，获取版本号、build
                    JSONObject jsonObject = JSONObject.parseObject(stringBuffer.toString());
                    System.out.println("获取版本号、build");
                    System.out.println(jsonObject.get("version"));
                }
            }
        }
    }
```

执行结果：

![](\assets\images\2022\java\uni-applet-read.png)

扩展到web 接口：

```java
@Api(tags = {"应用版本表"})
@RequiredArgsConstructor(onConstructor_={@Autowired})
@RestController
@RequestMapping("/v1/app/version")
public class AppVersionController{
    private final AppVersionService appVersionService;
    
        @PostMapping("/parse")
    public R<Map<String,Object>> parse(@RequestPart(name = "file") MultipartFile file, @RequestParam(name = "type")AppTypeEnum type) throws IOException {
        Map<String,Object> map = appVersionService.parse(file.getInputStream(),type);
        return R.buildSuccess(map);
    }
}
```

Service实现类

```java
    @Override
    public Map<String, Object> parse(InputStream is,AppTypeEnum type) throws IOException {
        Map<String,Object> map = new HashMap<>(2);
        try(BufferedInputStream bis = new BufferedInputStream(is); ZipInputStream zis = new ZipInputStream(bis);){
            // ZIP文件入口
            ZipEntry ze = null;
            while ((ze=zis.getNextEntry())!=null){
                // 找到manifest.json文件
                if(!ze.isDirectory() && ze.getName().equals("manifest.json")){
                    // log.info("解析应用版本小程序代码包,找到manifest.json文件");
                    // 读取
                    BufferedReader br = new BufferedReader(new InputStreamReader(zis));
                    StringBuffer stringBuffer = new StringBuffer();
                    String line;
                    while ((line = br.readLine())!=null){
                        stringBuffer.append(line);
                    }
                    // 转换json对象，获取版本号、build
                    JSONObject jsonObject = null;
                    try {
                        jsonObject = JSONObject.parseObject(stringBuffer.toString());
                        map.put("version",jsonObject.get("version"));
                    } catch (Exception e) {
                        log.error("解析应用版本小程序代码包，转换json对象失败：{}",e.getMessage());
                        throw new SystemRuntimeException("转换json对象失败"+e.getMessage());
                    }
                }
            }
        }
        return map;
    }
```

下面来简单学习一下ZipInputStream，参考 [https://geek-docs.com/java/java-tutorial/zipinputstream.html](https://geek-docs.com/java/java-tutorial/zipinputstream.html)

ZIP 是一种存档文件格式，支持无损数据压缩。 一个 ZIP 文件可能包含一个或多个已压缩的文件或目录

`ZipInputStream`构造函数

```java
ZipInputStream(InputStream in)
ZipInputStream(InputStream in, Charset charset)
```

`ZipInputStream's` `getNextEntry()`读取下一个 ZIP 文件条目，并将流定位在条目数据的开头。

> 读取ZIP例子

```java
public class JavaReadZip {

    private final static Long MILLS_IN_DAY = 86400000L;

    public static void main(String[] args) throws IOException {
        String fileName = "src/resources/myfile.zip";
        // FileInputStream用于读取原始字节流。
        // 为了获得更好的性能，我们将FileInputStream传递到BufferedInputStream中。
        try (FileInputStream fis = new FileInputStream(fileName);
                BufferedInputStream bis = new BufferedInputStream(fis);
                ZipInputStream zis = new ZipInputStream(bis)) {
            ZipEntry ze;
            while ((ze = zis.getNextEntry()) != null) {
                // getName()返回条目的名称，getSize()返回条目的未压缩大小，getTime()返回条目的最后修改时间。
                System.out.format("File: %s Size: %d Last Modified %s %n",
                        ze.getName(), ze.getSize(),
                        LocalDate.ofEpochDay(ze.getTime() / MILLS_IN_DAY));
            }
        }
    }
}
```

结果示例

```sh
File: maven.pdf Size: 6430817 Last Modified 2017-02-23 
```

> 解压示例

```java
import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

public class JavaUnzip {

    public static void main(String args[]) throws Exception {

        byte[] buffer = new byte[2048];
        // 解压目录
        Path outDir = Paths.get("src/resources/output/");
        String zipFileName = "src/resources/myfile.zip";

        try (FileInputStream fis = new FileInputStream(zipFileName);
                BufferedInputStream bis = new BufferedInputStream(fis);
                ZipInputStream stream = new ZipInputStream(bis)) {

            ZipEntry entry;
            // 这是我们提取 ZIP 文件内容的目录。
            while ((entry = stream.getNextEntry()) != null) {
                // 解压目录创建文件路径
                Path filePath = outDir.resolve(entry.getName());
                // 创建文件输出流
                try (FileOutputStream fos = new FileOutputStream(filePath.toFile());
                        BufferedOutputStream bos = new BufferedOutputStream(fos, buffer.length)) {

                    int len;
                    while ((len = stream.read(buffer)) > 0) {
                        bos.write(buffer, 0, len);
                    }
                }
            }
        }
    }
}
```





