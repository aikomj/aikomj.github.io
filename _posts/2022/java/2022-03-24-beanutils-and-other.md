---
layout: post
title: Bean自动映射工具
category: java
tags: [java]
keywords: java
excerpt: 常用的get/set,spring的beanutils.copyProperties,推荐MapStruct
lock: noneed
---

以下文章内容来源小傅哥

dto到vo的属性拷贝工具，12种转换工具的性能比较

![](/assets/images/2022/java/copy-properties.jpg)

只要不使用Apache的BeanUtils.copyProperties，基本上没有多大性能问题，记住使用Spring的BeanUtils.copyPerties

## 1、12种案例

![](/assets/images/2022/java/copy-properties-2.jpg)



源码 :[https://github.com/fuzhengwei/guide-vo2dto](https://github.com/fuzhengwei/guide-vo2dto)

在案例工程下创建 interfaces.assembler 包，定义 `IAssembler<SOURCE, TARGET>#sourceToTarget(SOURCE var)` 接口，提供不同方式的对象转换操作类实现，学习的过程中可以直接下载运行调试。

### get/set

```java
@Component
public class GetSetAssembler implements IAssembler<UserVO, UserDTO> {

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        UserDTO userDTO = new UserDTO();
        userDTO.setUserId(var.getUserId());
        userDTO.setUserNickName(var.getUserNickName());
        userDTO.setCreateTime(var.getCreateTime());
        return userDTO;
    }
}
```

- **推荐**：★★★☆☆
- **性能**：★★★★★
- **手段**：手写
- **点评**：其实这种方式也是日常使用的最多的，性能肯定是杠杠的，就是操作起来有点麻烦。尤其是一大堆属性的 VO 对象转换为 DTO 对象时候。但其实也有一些快捷的操作方式，比如你可以通过 Shift+Alt 选中所有属性，Shift+Tab 归并到一列，接下来在使用 Alt 选中这一列，批量操作粘贴 `userDTO.set` 以及快捷键大写属性首字母，最后切换到结尾补充括号和分号，最终格式化一下就搞定了。

### json2json

```java
@Component
public class Json2JsonAssembler implements IAssembler<UserVO, UserDTO> {

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        String strJson = JSON.toJSONString(var);
        return JSON.parseObject(strJson, UserDTO.class);
    }
}
```

- **推荐**：☆☆☆☆☆
- **性能**：★☆☆☆☆
- **手段**：把对象转JSON串，再把JSON转另外一个对象
- **点评**：这么写多半有点烧！

### Apache BeanUtils

```java
@Component
public class ApacheCopyPropertiesAssembler implements IAssembler<UserVO, UserDTO> {

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        UserDTO userDTO = new UserDTO();
        try {
            BeanUtils.copyProperties(userDTO, var);
        } catch (IllegalAccessException | InvocationTargetException e) {
            e.printStackTrace();
        }
        return userDTO;
    }
}
```

- **推荐**：☆☆☆☆☆
- **性能**：★☆☆☆☆
- **手段**：Introspector 机制获取到类的属性来进行赋值操作
- **点评**：有坑，兼容性交差，不建议使用

### Spring BeanUtils

```java
@Component
public class SpringCopyPropertiesAssembler implements IAssembler<UserVO, UserDTO> {

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        UserDTO userDTO = new UserDTO();
        BeanUtils.copyProperties(var, userDTO);
        return userDTO;
    }
}
```

- **推荐**：★★★☆☆
- **性能**：★★★★☆
- **手段**：Introspector机制获取到类的属性来进行赋值操作
- **点评**：同样是反射的属性拷贝，Spring 提供的 copyProperties 要比 Apache 好用的多，只要你不用错，基本不会有啥问题。

### Bean Mapping

```java
import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.assembler.IAssembler;
import cn.itedus.demo.interfaces.dto.UserDTO;
import com.github.houbb.bean.mapping.core.util.BeanUtil;
import org.springframework.stereotype.Component;

@Component
public class BeanMappingAssembler implements IAssembler<UserVO, UserDTO> {

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        UserDTO userDTO = new UserDTO();
        BeanUtil.copyProperties(var, userDTO);
        return userDTO;
    }

}
```

- **推荐**：★★☆☆☆
- **性能**：★★★☆☆
- **手段**：属性拷贝
- **点评**：性能一般

### Bean Mapping ASM

```java
import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.assembler.IAssembler;
import cn.itedus.demo.interfaces.dto.UserDTO;
import com.github.houbb.bean.mapping.asm.util.AsmBeanUtil;
import org.springframework.stereotype.Component;

@Component
public class BeanMappingASMAssembler implements IAssembler<UserVO, UserDTO> {

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        UserDTO userDTO = new UserDTO();
        AsmBeanUtil.copyProperties(var, userDTO);
        return userDTO;
    }
}
```

- **推荐**：★★★☆☆
- **性能**：★★★★☆
- **手段**：基于ASM字节码框架实现
- **点评**：与普通的 Bean Mapping 相比，性能有所提升，可以使用。

### BeanCopier

```java
import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.assembler.IAssembler;
import cn.itedus.demo.interfaces.dto.UserDTO;
import org.springframework.cglib.beans.BeanCopier;
import org.springframework.stereotype.Component;

@Component
public class BeanCopierAssembler implements IAssembler<UserVO, UserDTO> {

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        UserDTO userDTO = new UserDTO();
        BeanCopier beanCopier = BeanCopier.create(var.getClass(), userDTO.getClass(), false);
        beanCopier.copy(var, userDTO, null);
        return userDTO;
    }
}
```

- **推荐**：★★★☆☆
- **性能**：★★★★☆
- **手段**：基于CGlib字节码操作生成get、set方法
- **点评**：整体性能很不错，使用也不复杂，可以使用

### Orika

```java
import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.assembler.IAssembler;
import cn.itedus.demo.interfaces.dto.UserDTO;
import ma.glasnost.orika.MapperFactory;
import ma.glasnost.orika.impl.DefaultMapperFactory;
import org.springframework.stereotype.Component;

@Component
public class OrikaAssembler implements IAssembler<UserVO, UserDTO> {

    /**
     * 构造一个MapperFactory
     */
    private static MapperFactory mapperFactory = new DefaultMapperFactory.Builder().build();

    static {
        mapperFactory.classMap(UserDTO.class, UserVO.class)
                .field("userId", "userId")  // 字段不一致时可以指定
                .byDefault()
                .register();
    }

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        return mapperFactory.getMapperFacade().map(var, UserDTO.class);
    }
}
```

- **官网**：https://orika-mapper.github.io/orika-docs/
- **推荐**：★★☆☆☆
- **性能**：★★★☆☆
- **手段**：基于字节码生成映射对象
- **点评**：测试性能不是太突出，如果使用的话需要把 MapperFactory 的构建优化成 Bean 对象

### Dozer

```java
package cn.itedus.demo.interfaces.assembler.dozer;

import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.assembler.IAssembler;
import cn.itedus.demo.interfaces.dto.UserDTO;
import org.dozer.DozerBeanMapper;
import org.springframework.stereotype.Component;

@Component
public class DozerAssembler implements IAssembler<UserVO, UserDTO> {

    private static DozerBeanMapper mapper = new DozerBeanMapper();

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        return mapper.map(var, UserDTO.class);
    }
}
```

- **官网**：http://dozer.sourceforge.net/documentation/gettingstarted.html
- **推荐**：★☆☆☆☆
- **性能**：★★☆☆☆
- **手段**：属性映射框架，递归的方式复制对象
- **点评**：性能有点差，不建议使用

### ModelMapper

```java
package cn.itedus.demo.interfaces.assembler.model_mapper;

import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.assembler.IAssembler;
import cn.itedus.demo.interfaces.dto.UserDTO;
import org.modelmapper.ModelMapper;
import org.modelmapper.PropertyMap;
import org.springframework.stereotype.Component;

@Component
public class ModelMapperAssembler implements IAssembler<UserVO, UserDTO> {

    private static ModelMapper modelMapper = new ModelMapper();

    static {
        modelMapper.addMappings(new PropertyMap<UserVO, UserDTO>() {
            @Override
            protected void configure() {
                // 属性值不一样可以自己操作
                map().setUserId(source.getUserId());
            }
        });
    }

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        return modelMapper.map(var, UserDTO.class);
    }
}
```

- **官网**：http://modelmapper.org
- **推荐**：★★★☆☆
- **性能**：★★★☆☆
- **手段**：基于ASM字节码实现
- **点评**：转换对象数量较少时性能不错，如果同时大批量转换对象，性能有所下降

### JMapper

```java
package cn.itedus.demo.interfaces.assembler.jmapper;

import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.assembler.IAssembler;
import cn.itedus.demo.interfaces.dto.UserDTO;
import org.springframework.stereotype.Component;

@Component
public class JMapperAssembler implements IAssembler<UserVO, UserDTO> {

    @Override
    public UserDTO sourceToTarget(UserVO var) {
       /* JMapper<UserDTO, UserVO> jMapper = new JMapper<>(UserDTO.class, UserVO.class, new JMapperAPI()
                .add(JMapperAPI.mappedClass(UserDTO.class)
                        .add(JMapperAPI.attribute("userId")
                                .value("userId"))
                        .add(JMapperAPI.attribute("userNickName")
                                .value("userNickName"))
                        .add(JMapperAPI.attribute("createTime")
                                .value("createTime"))
                ));
        return jMapper.getDestination(var);*/
       return null;
    }
}
```

- **官网**：https://github.com/jmapper-framework/jmapper-core/wiki
- **推荐**：★★★★☆
- **性能**：★★★★★
- **手段**：Elegance, high performance and robustness all in one java bean mapper
- **点评**：速度真心可以，不过结合 SpringBoot 感觉有的一点点麻烦，可能姿势不对

### MapStruct

```java
package cn.itedus.demo.interfaces.assembler.map_struct;

import org.mapstruct.InheritConfiguration;
import org.mapstruct.InheritInverseConfiguration;
import org.mapstruct.MapperConfig;
import org.mapstruct.Mapping;
import java.util.List;
import java.util.stream.Stream;

/**
 * @description: 对象映射接口
 * @author: 小傅哥，微信：fustack
 * @date: 2021/9/14
 * @github: https://github.com/fuzhengwei
 * @Copyright: 公众号：bugstack虫洞栈 | 博客：https://bugstack.cn - 沉淀、分享、成长，让自己和他人都能有所收获！
 */
@MapperConfig
public interface IMapping<SOURCE, TARGET>{

    /**
     * 映射同名属性
     */
    @Mapping(target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    TARGET sourceToTarget(SOURCE var1);

    /**
     * 反向，映射同名属性
     */
    @InheritInverseConfiguration(name = "sourceToTarget")
    SOURCE targetToSource(TARGET var1);

    /**
     * 映射同名属性，集合形式
     */
    @InheritConfiguration(name = "sourceToTarget")
    List<TARGET> sourceToTarget(List<SOURCE> var1);

    /**
     * 反向，映射同名属性，集合形式
     */
    @InheritConfiguration(name = "targetToSource")
    List<SOURCE> targetToSource(List<TARGET> var1);

    /**
     * 映射同名属性，集合流形式
     */
    List<TARGET> sourceToTarget(Stream<SOURCE> stream);

    /**
     * 反向，映射同名属性，集合流形式
     */
    List<SOURCE> targetToSource(Stream<TARGET> stream);
}
```

这个，我在美的工作时，国内销售微服务平台就是用MapStruct

```java
package cn.itedus.demo.interfaces.assembler.map_struct;

import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.dto.UserDTO;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.ReportingPolicy;
import org.mapstruct.factory.Mappers;

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE, unmappedSourcePolicy = ReportingPolicy.IGNORE)
public interface UserDTOMapping extends IMapping<UserVO, UserDTO> {

    /** 用于测试的单例 */
    IMapping<UserVO, UserDTO> INSTANCE = Mappers.getMapper(UserDTOMapping.class);

    @Mapping(target = "userId", source = "userId")
    @Mapping(target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    @Override
    UserDTO sourceToTarget(UserVO var1);

    @Mapping(target = "userId", source = "userId")
    @Mapping(target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    @Override
    UserVO targetToSource(UserDTO var1);
}
```

```java
package cn.itedus.demo.interfaces.assembler.map_struct;

import cn.itedus.demo.domain.vo.UserVO;
import cn.itedus.demo.interfaces.assembler.IAssembler;
import cn.itedus.demo.interfaces.dto.UserDTO;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

@Component
public class MapStructAssembler implements IAssembler<UserVO, UserDTO> {

    @Resource
    private IMapping<UserVO, UserDTO> userDTOMapping;

    @Override
    public UserDTO sourceToTarget(UserVO var) {
        return userDTOMapping.sourceToTarget(var);
    }
}
```

- **官网**：https://github.com/mapstruct/mapstruct
- **推荐**：★★★★★
- **性能**：★★★★★
- **手段**：直接在编译期生成对应的get、set，像手写的代码一样
- **点评**：速度很快，不需要到运行期处理，结合到框架中使用方便

### 总结

- 其实对象属性转换的操作无非是基于反射、AOP、CGlib、ASM、Javassist 在编译时和运行期进行处理，再有好的思路就是在编译前生成出对应的get、set，就像手写出来的一样。
- 所以我更推荐我喜欢的 MapStruct，这货用起来还是比较舒服的，一种是来自于功能上的拓展性，易用性和兼容性