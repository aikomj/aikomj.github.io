---
layout: post
title: Bean自动映射工具
category: java
tags: [java]
keywords: java
excerpt: 常用的get/set,spring的beanutils.copyProperties,详细介绍MapStruct的使用
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

这个，我在美的工作时，国内销售微服务平台也是用MapStruct进行entity、dto之间的转换

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

> 例子2

User.java

```java
@AllArgsConstructor  
@Data  
public class User {  
    private Long id;  
    private String username;  
    private String password;  
    private String phoneNum;  
    private String email;  
    private Role role;  
}  
```

Role.java

```java
@AllArgsConstructor  
@Data  
public class Role {  
    private Long id;  
    private String roleName;  
    private String description;  
}  
```

UserRoleDto.java

```java
@Data  
public class UserRoleDto {  
    /**  
     * 用户id  
     */  
    private Long userId;  
    /**  
     * 用户名  
     */  
    private String name;  
    /**  
     * 角色名  
     */  
    private String roleName;  
}  
```

新建一个UserRoleMapper.java，这个来用来定义User.java、Role.java和UserRoleDto.java之间属性对应规则：

```java
import org.mapstruct.Mapper;  
import org.mapstruct.Mapping;  
import org.mapstruct.Mappings;  
import org.mapstruct.factory.Mappers;  
  
/**  
 * @Mapper 定义这是一个MapStruct对象属性转换接口，在这个类里面规定转换规则  
 *          在项目构建时，会自动生成改接口的实现类，这个实现类将实现对象属性值复制  
 */  
@Mapper  
public interface UserRoleMapper {  
  
    /**  
     * 获取该类自动生成的实现类的实例  
     * 接口中的属性都是 public static final 的 方法都是public abstract的  
     */  
    UserRoleMapper INSTANCES = Mappers.getMapper(UserRoleMapper.class);  
  
    /**  
     * 这个方法就是用于实现对象属性复制的方法  
     *  
     * @Mapping 用来定义属性复制规则 source 指定源对象属性 target指定目标对象属性  
     *  
     * @param user 这个参数就是源对象，也就是需要被复制的对象  
     * @return 返回的是目标对象，就是最终的结果对象  
     */  
    @Mappings({  
            @Mapping(source = "id", target = "userId"),  
            @Mapping(source = "username", target = "name"),  
            @Mapping(source = "role.roleName", target = "roleName")  
    })  
    UserRoleDto toUserRoleDto(User user);  
}  
```

**2.添加默认方法**

添加默认方法是为了这个类（接口）不只是为了做数据转换用的，也可以做一些其他的事。

```java
import org.mapstruct.Mapper;  
import org.mapstruct.Mapping;  
import org.mapstruct.Mappings;  
import org.mapstruct.factory.Mappers;  
  
/**  
 * @Mapper 定义这是一个MapStruct对象属性转换接口，在这个类里面规定转换规则  
 *          在项目构建时，会自动生成改接口的实现类，这个实现类将实现对象属性值复制  
 */  
@Mapper  
public interface UserRoleMapper {  
  
    /**  
     * 获取该类自动生成的实现类的实例  
     * 接口中的属性都是 public static final 的 方法都是public abstract的  
     */  
    UserRoleMapper INSTANCES = Mappers.getMapper(UserRoleMapper.class);  
  
    /**  
     * 这个方法就是用于实现对象属性复制的方法  
     *  
     * @Mapping 用来定义属性复制规则 source 指定源对象属性 target指定目标对象属性  
     *  
     * @param user 这个参数就是源对象，也就是需要被复制的对象  
     * @return 返回的是目标对象，就是最终的结果对象  
     */  
    @Mappings({  
            @Mapping(source = "id", target = "userId"),  
            @Mapping(source = "username", target = "name"),  
            @Mapping(source = "role.roleName", target = "roleName")  
    })  
    UserRoleDto toUserRoleDto(User user);  
  
    /**  
     * 提供默认方法，方法自己定义，这个方法是我随便写的，不是要按照这个格式来的  
     * @return  
     */  
    default UserRoleDto defaultConvert() {  
        UserRoleDto userRoleDto = new UserRoleDto();  
        userRoleDto.setUserId(0L);  
        userRoleDto.setName("None");  
        userRoleDto.setRoleName("None");  
        return userRoleDto;  
    }  
}  
```

测试代码

```java
@Test  
public void test3() {  
    UserRoleMapper userRoleMapperInstances = UserRoleMapper.INSTANCES;  
    UserRoleDto userRoleDto = userRoleMapperInstances.defaultConvert();  
    System.out.println(userRoleDto);  
}  
```

**3.使用抽象类来代替接口**

```java
import org.mapstruct.Mapper;  
import org.mapstruct.Mapping;  
import org.mapstruct.Mappings;  
import org.mapstruct.factory.Mappers;  
  
/**  
 * @Mapper 定义这是一个MapStruct对象属性转换接口，在这个类里面规定转换规则  
 *          在项目构建时，会自动生成改接口的实现类，这个实现类将实现对象属性值复制  
 */  
@Mapper  
public abstract class UserRoleMapper {  
  
    /**  
     * 获取该类自动生成的实现类的实例  
     * 接口中的属性都是 public static final 的 方法都是public abstract的  
     */  
    public static final UserRoleMapper INSTANCES = Mappers.getMapper(UserRoleMapper.class);  
  
    /**  
     * 这个方法就是用于实现对象属性复制的方法  
     *  
     * @Mapping 用来定义属性复制规则 source 指定源对象属性 target指定目标对象属性  
     *  
     * @param user 这个参数就是源对象，也就是需要被复制的对象  
     * @return 返回的是目标对象，就是最终的结果对象  
     */  
    @Mappings({  
            @Mapping(source = "id", target = "userId"),  
            @Mapping(source = "username", target = "name"),  
            @Mapping(source = "role.roleName", target = "roleName")  
    })  
    public abstract UserRoleDto toUserRoleDto(User user);  
  
    /**  
     * 提供默认方法，方法自己定义，这个方法是我随便写的，不是要按照这个格式来的  
     * @return  
     */  
    UserRoleDto defaultConvert() {  
        UserRoleDto userRoleDto = new UserRoleDto();  
        userRoleDto.setUserId(0L);  
        userRoleDto.setName("None");  
        userRoleDto.setRoleName("None");  
        return userRoleDto;  
    }  
}  
```

**4.使用多个参数**

可以绑定多个对象的属性值到目标对象中：

```java
package com.mapstruct.demo;  
  
import org.mapstruct.Mapper;  
import org.mapstruct.Mapping;  
import org.mapstruct.Mappings;  
import org.mapstruct.factory.Mappers;  
  
/**  
 * @Mapper 定义这是一个MapStruct对象属性转换接口，在这个类里面规定转换规则  
 *          在项目构建时，会自动生成改接口的实现类，这个实现类将实现对象属性值复制  
 */  
@Mapper  
public interface UserRoleMapper {  
  
    /**  
     * 获取该类自动生成的实现类的实例  
     * 接口中的属性都是 public static final 的 方法都是public abstract的  
     */  
    UserRoleMapper INSTANCES = Mappers.getMapper(UserRoleMapper.class);  
  
    /**  
     * 这个方法就是用于实现对象属性复制的方法  
     *  
     * @Mapping 用来定义属性复制规则 source 指定源对象属性 target指定目标对象属性  
     *  
     * @param user 这个参数就是源对象，也就是需要被复制的对象  
     * @return 返回的是目标对象，就是最终的结果对象  
     */  
    @Mappings({  
            @Mapping(source = "id", target = "userId"),  
            @Mapping(source = "username", target = "name"),  
            @Mapping(source = "role.roleName", target = "roleName")  
    })  
    UserRoleDto toUserRoleDto(User user);  
  
    /**  
     * 多个参数中的值绑定   
     * @param user 源1  
     * @param role 源2  
     * @return 从源1、2中提取出的结果  
     */  
    @Mappings({  
            @Mapping(source = "user.id", target = "userId"), // 把user中的id绑定到目标对象的userId属性中  
            @Mapping(source = "user.username", target = "name"), // 把user中的username绑定到目标对象的name属性中  
            @Mapping(source = "role.roleName", target = "roleName") // 把role对象的roleName属性值绑定到目标对象的roleName中  
    })  
    UserRoleDto toUserRoleDto(User user, Role role);  
```

**5.直接使用参数作为属性值**

```java
import org.mapstruct.Mapper;  
import org.mapstruct.Mapping;  
import org.mapstruct.Mappings;  
import org.mapstruct.factory.Mappers;  
  
/**  
 * @Mapper 定义这是一个MapStruct对象属性转换接口，在这个类里面规定转换规则  
 *          在项目构建时，会自动生成改接口的实现类，这个实现类将实现对象属性值复制  
 */  
@Mapper  
public interface UserRoleMapper {  
  
    /**  
     * 获取该类自动生成的实现类的实例  
     * 接口中的属性都是 public static final 的 方法都是public abstract的  
     */  
    UserRoleMapper INSTANCES = Mappers.getMapper(UserRoleMapper.class);  
  
    /**  
     * 直接使用参数作为值  
     * @param user  
     * @param myRoleName  
     * @return  
     */  
    @Mappings({  
            @Mapping(source = "user.id", target = "userId"), // 把user中的id绑定到目标对象的userId属性中  
            @Mapping(source = "user.username", target = "name"), // 把user中的username绑定到目标对象的name属性中  
            @Mapping(source = "myRoleName", target = "roleName") // 把role对象的roleName属性值绑定到目标对象的roleName中  
    })  
    UserRoleDto useParameter(User user, String myRoleName);  
}  
```

测试类

```java
public class Test1 {  
    Role role = null;  
    User user = null;  
  
    @Before  
    public void before() {  
        role = new Role(2L, "administrator", "超级管理员");  
        user = new User(1L, "zhangsan", "12345", "17677778888", "123@qq.com", role);  
    }  
    @Test  
    public void test1() {  
        UserRoleMapper instances = UserRoleMapper.INSTANCES;  
        UserRoleDto userRoleDto = instances.useParameter(user, "myUserRole");  
        System.out.println(userRoleDto);  
    }  
}  
```

**6.更新对象属性**

在之前的例子中`UserRoleDto useParameter(User user, String myRoleName);`都是通过类似上面的方法来生成一个对象。而MapStruct提供了另外一种方式来更新一个对象中的属性，@MappingTarget

```java
public interface UserRoleMapper1 {  
    UserRoleMapper1 INSTANCES = Mappers.getMapper(UserRoleMapper1.class);  
  
    @Mappings({  
            @Mapping(source = "userId", target = "id"),  
            @Mapping(source = "name", target = "username"),  
            @Mapping(source = "roleName", target = "role.roleName")  
    })  
    void updateDto(UserRoleDto userRoleDto, @MappingTarget User user);  
  
  
    @Mappings({  
            @Mapping(source = "id", target = "userId"),  
            @Mapping(source = "username", target = "name"),  
            @Mapping(source = "role.roleName", target = "roleName")  
    })  
    void update(User user, @MappingTarget UserRoleDto userRoleDto);  
 
}  
```

通过@MappingTarget来指定目标类是谁（谁的属性需要被更新）。@Mapping还是用来定义属性对应规则。

```java
@Mappings({  
            @Mapping(source = "id", target = "userId"),  
            @Mapping(source = "username", target = "name"),  
            @Mapping(source = "role.roleName", target = "roleName")  
    })  
    void update(User user, @MappingTarget UserRoleDto userRoleDto);  
```

`@MappingTarget`标注的类UserRoleDto 为目标类，user类为源类，调用此方法，会把源类中的属性更新到目标类中。更新规则还是由`@Mapping`指定。

**7.没有getter/setter也能赋值**

```java
public class Customer {  
  
    private Long id;  
    private String name;  
  
    //getters and setter omitted for brevity  
}  
  
public class CustomerDto {  
  
    public Long id;  
    public String customerName;  
}  

@Mapper  
public interface CustomerMapper {  
  
    CustomerMapper INSTANCE = Mappers.getMapper( CustomerMapper.class );  
  
    @Mapping(source = "customerName", target = "name")  
    Customer toCustomer(CustomerDto customerDto);  
  
    @InheritInverseConfiguration  
    CustomerDto fromCustomer(Customer customer);  
}  
```

`@Mapping(source = “customerName”, target = “name”)`不是用来指定属性映射的，如果两个对象的属性名相同是可以省略@Mapping的。

上面MapStruct生成的实现类：

```java
@Generated(  
    value = "org.mapstruct.ap.MappingProcessor",  
    date = "2019-02-14T15:41:21+0800",  
    comments = "version: 1.3.0.Final, compiler: javac, environment: Java 1.8.0_181 (Oracle Corporation)"  
)  
public class CustomerMapperImpl implements CustomerMapper {  
  
    @Override  
    public Customer toCustomer(CustomerDto customerDto) {  
        if ( customerDto == null ) {  
            return null;  
        }  
  
        Customer customer = new Customer();  
  
        customer.setName( customerDto.customerName );  
        customer.setId( customerDto.id );  
  
        return customer;  
    }  
  
    @Override  
    public CustomerDto toCustomerDto(Customer customer) {  
        if ( customer == null ) {  
            return null;  
        }  
  
        CustomerDto customerDto = new CustomerDto();  
  
        customerDto.customerName = customer.getName();  
        customerDto.id = customer.getId();  
  
        return customerDto;  
    }  
}  
```

`@InheritInverseConfiguration`在这里的作用就是实现`customerDto.customerName = customer.getName();`功能的。如果没有这个注解，toCustomerDto这个方法则不会有customerName 和name两个属性的对应关系的。

**8.使用Spring依赖注入**

```java
@Data  
@NoArgsConstructor  
@AllArgsConstructor  
public class Customer {  
    private Long id;  
    private String name;  
}  
  
@Data  
public class CustomerDto {  
    private Long id;  
    private String customerName;  
}  
  
// 这里主要是这个componentModel 属性，它的值就是当前要使用的依赖注入的环境  
@Mapper(componentModel = "spring")  
public interface CustomerMapper {  
  
    @Mapping(source = "name", target = "customerName")  
    CustomerDto toCustomerDto(Customer customer);  
}  
```

`@Mapper(componentModel = “spring”)`，表示把当前Mapper类纳入spring容器。可以在其它类中直接注入了：

```java
@SpringBootApplication  
@RestController  
public class DemoMapstructApplication {  
  
 // 注入Mapper  
    @Autowired  
    private CustomerMapper mapper;  
  
    public static void main(String[] args) {  
        SpringApplication.run(DemoMapstructApplication.class, args);  
    }  
  
    @GetMapping("/test")  
    public String test() {  
        Customer customer = new Customer(1L, "zhangsan");  
        CustomerDto customerDto = mapper.toCustomerDto(customer);  
        return customerDto.toString();  
    }  
}  
```

看一下MapStruct生成的实现类，会发现标记了`@Component`注解。

```java
@Generated(  
    value = "org.mapstruct.ap.MappingProcessor",  
    date = "2019-02-14T15:54:17+0800",  
    comments = "version: 1.3.0.Final, compiler: javac, environment: Java 1.8.0_181 (Oracle Corporation)"  
)  
@Component  
public class CustomerMapperImpl implements CustomerMapper {  
  
    @Override  
    public CustomerDto toCustomerDto(Customer customer) {  
        if ( customer == null ) {  
            return null;  
        }  
  
        CustomerDto customerDto = new CustomerDto();  
  
        customerDto.setCustomerName( customer.getName() );  
        customerDto.setId( customer.getId() );  
  
        return customerDto;  
    }  
} 
```

**9.自定义类型转换**

这个赞，有时候，在对象转换的时候可能会出现这样一个问题，就是源对象中的类型是Boolean类型，而目标对象类型是String类型，这种情况可以通过`@Mapper`的uses属性来实现：

```java
@Data  
@NoArgsConstructor  
@AllArgsConstructor  
public class Customer {  
    private Long id;  
    private String name;  
    private Boolean isDisable;  
}  
  
@Data  
public class CustomerDto {  
    private Long id;  
    private String customerName;  
    private String disable;  
}  
```

定义转换规则的类：

```java
public class BooleanStrFormat {  
    public String toStr(Boolean isDisable) {  
        if (isDisable) {  
            return "Y";  
        } else {  
            return "N";  
        }  
    }  
    public Boolean toBoolean(String str) {  
        if (str.equals("Y")) {  
            return true;  
        } else {  
            return false;  
        }  
    }  
}  
```

定义Mapper，`@Mapper( uses = { BooleanStrFormat.class})`，注意，这里的users属性用于引用之前定义的转换规则的类：

```java
@Mapper( uses = { BooleanStrFormat.class})  
public interface CustomerMapper {  
  
    CustomerMapper INSTANCES = Mappers.getMapper(CustomerMapper.class);  
  
    @Mappings({  
            @Mapping(source = "name", target = "customerName"),  
            @Mapping(source = "isDisable", target = "disable")  
    })  
    CustomerDto toCustomerDto(Customer customer);  
}  
```

这样子，Customer类中的isDisable属性的true就会转变成CustomerDto中的disable属性的yes。

MapStruct自动生成的实现类

```java
@Generated(  
    value = "org.mapstruct.ap.MappingProcessor",  
    date = "2019-02-14T16:49:18+0800",  
    comments = "version: 1.3.0.Final, compiler: javac, environment: Java 1.8.0_181 (Oracle Corporation)"  
)  
public class CustomerMapperImpl implements CustomerMapper {  
  
 // 引用 uses 中指定的类  
    private final BooleanStrFormat booleanStrFormat = new BooleanStrFormat();  
  
    @Override  
    public CustomerDto toCustomerDto(Customer customer) {  
        if ( customer == null ) {  
            return null;  
        }  
  
        CustomerDto customerDto = new CustomerDto();  
  // 转换方式的使用  
        customerDto.setDisable( booleanStrFormat.toStr( customer.getIsDisable() ) );  
        customerDto.setCustomerName( customer.getName() );  
        customerDto.setId( customer.getId() );  
  
        return customerDto;  
    }  
}  
```

我们看到它是创建了一个BooleanStrFormat对象，,要注意的是，<mark>如果使用了例如像spring这样的环境，Mapper引入uses类实例的方式将是自动注入，那么这个类也应该纳入Spring容器：</mark>

CustomerMapper.java指定使用spring

```java
@Mapper(componentModel = "spring", uses = { BooleanStrFormat.class})  
public interface CustomerMapper {  
  
    CustomerMapper INSTANCES = Mappers.getMapper(CustomerMapper.class);  
  
    @Mappings({  
            @Mapping(source = "name", target = "customerName"),  
            @Mapping(source = "isDisable", target = "disable")  
    })  
    CustomerDto toCustomerDto(Customer customer);  
}  
```

转换类加上@Component注解，加入Spring容器：

```java
@Component  
public class BooleanStrFormat {  
    public String toStr(Boolean isDisable) {  
        if (isDisable) {  
            return "Y";  
        } else {  
            return "N";  
        }  
    }  
    public Boolean toBoolean(String str) {  
        if (str.equals("Y")) {  
            return true;  
        } else {  
            return false;  
        }  
    }  
}  
```

MapStruct自动生成的实现类：

```java
@Generated(  
    value = "org.mapstruct.ap.MappingProcessor",  
    date = "2019-02-14T16:55:35+0800",  
    comments = "version: 1.3.0.Final, compiler: javac, environment: Java 1.8.0_181 (Oracle Corporation)"  
)  
@Component  
public class CustomerMapperImpl implements CustomerMapper {  
  
 // 使用自动注入的方式引入  
    @Autowired  
    private BooleanStrFormat booleanStrFormat;  
  
    @Override  
    public CustomerDto toCustomerDto(Customer customer) {  
        if ( customer == null ) {  
            return null;  
        }  
  
        CustomerDto customerDto = new CustomerDto();  
  
        customerDto.setDisable( booleanStrFormat.toStr( customer.getIsDisable() ) );  
        customerDto.setCustomerName( customer.getName() );  
        customerDto.setId( customer.getId() );  
  
        return customerDto;  
    }  
}
```





### 总结

- 其实对象属性转换的操作无非是基于反射、AOP、CGlib、ASM、Javassist 在编译时和运行期进行处理，再有好的思路就是在编译前生成出对应的get、set，就像手写出来的一样。
- 所以我更推荐我喜欢的 MapStruct，这货用起来还是比较舒服的，一种是来自于功能上的拓展性，易用性和兼容性