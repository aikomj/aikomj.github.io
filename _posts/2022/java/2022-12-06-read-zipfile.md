---
layout: post
title: 读取zip压缩文件内容
category: java
tags: [java]
keywords: java
excerpt: JDK压缩输入流ZipInputStream直接读取压缩包内容，可读取UNI小程序获取版本信息，使用ant解压工具类读取压缩文件
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

### ANT解压工具类

使用`ZipInputStream`读取zip文件可能会报异常<mark>java.util.zip.ZipException: only DEFLATED entries can have EXT descriptor</mark>

解决方案是使用apache的ant解压，maven依赖：

```xml
<dependency>
    <groupId>org.apache.ant</groupId>
    <artifactId>ant</artifactId>
    <version>1.10.5</version>
</dependency>
```

代码例子：

```java
import com.weihui.member.gateway.enums.ErrorCode;
import com.weihui.member.gateway.exception.BizException;
import org.apache.tools.zip.ZipEntry;
import org.apache.tools.zip.ZipFile;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.io.BufferedOutputStream;
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.Enumeration;

/**
 * chentao
 */
public class ZipUtilApache {
    private static final Logger log  = LoggerFactory.getLogger(ZipUtilApache.class);
    private static final int buffer = 2048;

    /**
     * 解压Zip文件
     *
     * @param path 文件目录
     */
    public static void unZip(String path) {
        int count = -1;
        String savepath = "";
        File file = null;
        InputStream is = null;
        FileOutputStream fos = null;
        BufferedOutputStream bos = null;
        savepath = new File(path).getParent() + File.separator; //保存解压文件目录
        new File(savepath).mkdir(); //创建保存目录
        ZipFile zipFile = null;
        try {
            zipFile = new ZipFile(path, "gbk"); //解决中文乱码问题
            Enumeration<?> entries = zipFile.getEntries();
            while (entries.hasMoreElements()) {
                byte buf[] = new byte[buffer];
                ZipEntry entry = (ZipEntry) entries.nextElement();
                String filename = entry.getName();
                boolean ismkdir = false;
                if (filename.lastIndexOf("/") != -1) { //检查此文件是否带有文件夹
                    ismkdir = true;
                }
                filename = savepath + filename;
                if (entry.isDirectory()) { //如果是文件夹先创建
                    file = new File(filename);
                    file.mkdirs();
                    continue;
                }
                file = new File(filename);
                if (!file.exists()) { //如果是目录先创建
                    if (ismkdir) {
                        new File(filename.substring(0, filename.lastIndexOf("/"))).mkdirs(); //目录先创建
                    }
                }
                file.createNewFile(); //创建文件
                is = zipFile.getInputStream(entry);
                fos = new FileOutputStream(file);
                bos = new BufferedOutputStream(fos, buffer);
                while ((count = is.read(buf)) > -1) {
                    bos.write(buf, 0, count);
                }
                bos.flush();
                bos.close();
                fos.close();
                is.close();
            }
            zipFile.close();
        } catch (IOException ioe) {
            log.error("ZipUtilApache-unZip-IOException",ioe);
            throw new BizException(ErrorCode.SYSTEM_ERROR,"解压缩商户资质文件["+path+"]失败","解压缩商户资质文件["+path+"]失败");
        } finally {
            try {
                if (bos != null) {
                    bos.close();
                }
                if (fos != null) {
                    fos.close();
                }
                if (is != null) {
                    is.close();
                }
                if (zipFile != null) {
                    zipFile.close();
                }
            } catch (Exception e) {
                log.error("ZipUtilApache-unZip-Exception",e);
            }
        }
    }

    public static void main(String[] args) {
        unZip("E:\\下载\\mgs1\\test\\GI20210303173812PXPKRV2304.zip");
        // unZip("E:\\下载\\mgs1\\test\\47kong.zip");
    }
}
```

读取小程序包的版本信息

```java
public static void readZipFileByAnt() throws IOException {
    String sourceZip = "C:\\jacob\\任务\\任务19-帷幄APP\\__UNI__01E620B.wgt";
    ZipFile zip = new ZipFile(new File(sourceZip));
    Enumeration entries = zip.getEntries();
    while(entries.hasMoreElements()){
        org.apache.tools.zip.ZipEntry ze = (org.apache.tools.zip.ZipEntry)entries.nextElement();
        if(!ze.isDirectory() && ze.getName().equals("manifest.json")){
            // 读取
            BufferedReader br = new BufferedReader(new InputStreamReader(zip.getInputStream(ze)));
            StringBuffer stringBuffer = new StringBuffer();
            String line;
            while ((line = br.readLine())!=null){
                stringBuffer.append(line);
            }
            System.out.println("读取版本信息:"+stringBuffer.toString());
            // 转换json对象，获取版本号、build
            JSONObject jsonObject = JSONObject.parseObject(stringBuffer.toString());
            System.out.println("获取版本号、build");
            System.out.println(jsonObject.get("version"));
        }
    }
}
```

Web接口我们接受的文件流是`MultipartFile`对象，需要转为`File`对象，工具类代码：

```java
public class FileUtils {

    public static File multipartFileToFile(MultipartFile multipartFile) throws IOException{
        File file = null;
        InputStream inputStream = null;
        OutputStream outputStream = null;
        try {
            inputStream = multipartFile.getInputStream();
            file = new File(multipartFile.getOriginalFilename());
            outputStream = new FileOutputStream(file);
            write(inputStream, outputStream);
        } finally {
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (outputStream != null) {
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return file;
    }

    private static void write(InputStream inputStream, OutputStream outputStream) {
        byte[] buffer = new byte[4096];
        try {
            int count = inputStream.read(buffer, 0, buffer.length);
            while (count != -1) {
                outputStream.write(buffer, 0, count);
                count = inputStream.read(buffer, 0, buffer.length);
            }
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeException(e.getMessage(), e);
        }
    }
}
```

Controller Web接口方法

```java
/**
     * 获取wgt文件小程序包的版本号和build
     */
@PostMapping("/parse")
public R<Map<String,Object>> parse(@RequestPart(name = "file") MultipartFile multipartFile, @RequestParam(name = "type")AppTypeEnum type) throws IOException {
    Map<String,Object> map = appVersionService.parse(multipartFile,type);
    return R.buildSuccess(map);
}
```

业务层AppVersionService实现类方法

```java
  @Override
    public Map<String, Object> parse(MultipartFile multipartFile, AppTypeEnum type) throws IOException {
        Map<String,Object> map = new HashMap<>(2);
        antParse(map,multipartFile);
        return map;
    }

    /**
     * ant解压
     */
    private void antParse(Map<String,Object> map,MultipartFile multipartFile) throws IOException {
        // MultipartFile 转换 File
        File file =  FileUtils.multipartFileToFile(multipartFile);
        ZipFile zip = new ZipFile(file);
        Enumeration entries = zip.getEntries();
        while(entries.hasMoreElements()){
            org.apache.tools.zip.ZipEntry ze = (org.apache.tools.zip.ZipEntry)entries.nextElement();
            if(!ze.isDirectory() && ze.getName().equals("manifest.json")){
                // 读取
                BufferedReader br = new BufferedReader(new InputStreamReader(zip.getInputStream(ze)));
                StringBuffer stringBuffer = new StringBuffer();
                String line;
                while ((line = br.readLine())!=null){
                    stringBuffer.append(line);
                }
                // log.info("读取版本信息:{}",stringBuffer.toString());
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

    /**
     *  jdk自带解压方式
     * @param map
     * @param multipartFile
     */
    private void jdkPares(Map<String,Object> map,MultipartFile multipartFile) throws IOException{
        InputStream is = multipartFile.getInputStream();
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
    }
```

