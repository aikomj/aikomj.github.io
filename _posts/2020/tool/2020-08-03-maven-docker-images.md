---
layout: post
title: 使用Maven插件构建Docker镜像 
category: tool
tags: [docker]
keywords: docker, maven
excerpt: Docker Registry 服务器镜像仓库，项目工程的pom.xml文件中添加docker-maven-plugin的依赖
lock: noneed
---

## 1、本地私有仓库

```sh
# 先拉取镜像
docker pull registry:2
# 依赖的基础镜像需要先行下载
docker pull java:8
```

### docker开启远程API

1、修改docker.service文件

```sh
vi /usr/lib/systemd/system/docker.service
# 需要修改的部分：
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# 修改后:
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock
```

2、让Docker支持http上传镜像

```sh
vi /etc/docker/daemon.json 
{
  "registry-mirrors": ["https://dyv68vb0.mirror.aliyuncs.com"], # 阿里云镜像加速
	"insecure-registries": ["139.199.13.139:5000"] # 云服务器的外网ip
}
# 重启daemon进程
systemctl daemon-reload
# 重新启动Docker服务
systemctl restart docker
```

3、开启防火墙的Docker构建端口

```sh
firewall-cmd --zone=public --add-port=2375/tcp --permanent
firewall-cmd --reload
```

4、启动镜像仓库

```sh
# --restar=always docker重启，镜像仓库也会重启
docker run -d -p 5000:5000 --restart=always --name registry2 registry:2
```



### docker-maven-plugin依赖

在应用的pom.xml文件中添加docker-maven-plugin的依赖

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>docker-maven-plugin</artifactId>
  <version>1.1.0</version>
  <executions>
    <execution>
      <id>build-image</id>
      <phase>package</phase>
      <goals>
        <goal>build</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <imageName>mall-tiny/${project.artifactId}:${project.version}</imageName>
    <dockerHost>http://139.199.13.139:2375</dockerHost>
    <baseImage>java:8</baseImage>
    <entryPoint>["java", "-jar","/${project.build.finalName}.jar"]
    </entryPoint>
    <resources>
      <resource>
        <targetPath>/</targetPath>
        <directory>${project.build.directory}</directory>
        <include>${project.build.finalName}.jar</include>
      </resource>
    </resources>
  </configuration>
</plugin>
```

相关配置说明：

- executions.execution.phase:此处配置了在maven打包应用时构建docker镜像；
- imageName：用于指定镜像名称，mall-tiny是仓库名称，${project.artifactId}为镜像名称，${project.version}为镜像版本号；
- dockerHost：打包后上传到的docker服务器地址；
- baseImage：该应用所依赖的基础镜像，此处为java；
- entryPoint：docker容器启动时执行的命令；
- resources.resource.targetPath：将打包后的资源文件复制到该目录；
- resources.resource.directory：需要复制的文件所在目录，maven打包的应用jar包保存在target目录下面；
- resources.resource.include：需要复制的文件，打包好的应用jar包。

模块mall-admin 的pom.xml 解开注释

![](/assets/images/2020/icoding/imall/docker-maven-plugin.jpg)

直接总项目mall 打包

![](/assets/images/2020/icoding/imall/mall-package.jpg)

![](/assets/images/2020/icoding/imall/mall-package-2.jpg)

可以看到它在复制jar包在构建镜像，发现是在项目的target目录有个docker目录，里面有Dockerfile文件

![](/assets/images/2020/icoding/imall/mall-package-3.jpg)

所以docker-maven-plugin插件是使用Dockerfile构建镜像的，${docker.host}在父依赖pom.xml已配置

![](/assets/images/2020/icoding/imall/mall-package-4.jpg)

去服务器镜像仓库，查看镜像，构建成功

![](/assets/images/2020/icoding/imall/mall-package-5.jpg)



> 参考

https://mp.weixin.qq.com/s/q2KDzHbPkf3Q0EY8qYjYgw