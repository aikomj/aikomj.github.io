---
layout: post
title: 使用一个规则执行器替代大量if判断
category: java
tags: [java]
keywords: java
excerpt: 自定义规则执行器
lock: noneed
---

## 1、业务场景

近日在公司领到一个小需求，需要对之前已有的试用用户申请规则进行拓展。我们的场景大概如下所示:

```java
if (是否海外用户) {
 return false;
}

if (刷单用户) {
  return false;
}

if (未付费用户 && 不再服务时段) {
  return false
}

if (转介绍用户 || 付费用户 || 内推用户) {
  return true;
}
```

按照业务语义，返回false说明该试用用户没有权限，都是基于and条件或者or条件判断，如果有一个不匹配的，后续的流程就不用执行，就是需要具备一个短路的功能。

## 2、规则执行器

![](\assets\images\2021\javabase\self-rule.jpg)

### 声明规则

对于规则的抽象并实现，代码如下：

```java
// 业务数据
@Data
public class RuleDto {
  private String address;
  private int age;
}

// 常量定义
public class RuleConstant {
  public static final String MATCH_ADDRESS_START= "北京";
  public static final String MATCH_NATIONALITY_START= "中国";
}

// 规则接口
public interface BaseRule {
  boolean execute(RuleDto dto);
}

// 规则模板
public abstract class AbstractRule implements BaseRule {
  protected <T> T convert(RuleDto dto) {
    return (T) dto;
  }

  @Override
  public boolean execute(RuleDto dto) {
    return executeRule(convert(dto));
  }

  protected <T> boolean executeRule(T t) {
    return true;
  }
}

// 具体规则- 例子1
public class AddressRule extends AbstractRule {
  @Override
  public boolean execute(RuleDto dto) {
    System.out.println("AddressRule invoke!");
    if (dto.getAddress().startsWith(MATCH_ADDRESS_START)) {
      return true;
    }
    return false;
  }
}

// 具体规则- 例子2
public class NationalityRule extends AbstractRule {
  @Override
  protected <T> T convert(RuleDto dto) {
    NationalityRuleDto nationalityRuleDto = new NationalityRuleDto();
    if (dto.getAddress().startsWith(MATCH_ADDRESS_START)) {
      nationalityRuleDto.setNationality(MATCH_NATIONALITY_START);
    }
    return (T) nationalityRuleDto;
  }

  @Override
  protected <T> boolean executeRule(T t) {
    System.out.println("NationalityRule invoke!");
    NationalityRuleDto nationalityRuleDto = (NationalityRuleDto) t;
    if (nationalityRuleDto.getNationality().startsWith(MATCH_NATIONALITY_START)) {
      return true;
    }
    return false;
  }
}
```

### 构建执行器

```java
public class RuleService {
    private Map<Integer, List<BaseRule>> hashMap = new HashMap<>();
    private static final int AND = 1;
    private static final int OR = 0;

    public static RuleService create() {
        return new RuleService();
    }

    public RuleService and(List<BaseRule> ruleList) {
        hashMap.put(AND, ruleList);
        return this;
    }

    public RuleService or(List<BaseRule> ruleList) {
        hashMap.put(OR, ruleList);
        return this;
    }

    public boolean execute(RuleDto dto) {
        for (Map.Entry<Integer, List<BaseRule>> item : hashMap.entrySet()) {
            List<BaseRule> ruleList = item.getValue();
            switch (item.getKey()) {
                case AND:
                    // 如果是 and 关系，同步执行
                    System.out.println("execute key = " + 1);
                    if (!and(dto, ruleList)) {
                        return false;
                    }
                    break;
                case OR:
                    // 如果是 or 关系，并行执行
                    System.out.println("execute key = " + 0);
                    if (!or(dto, ruleList)) {
                        return false;
                    }
                    break;
                default:
                    break;
            }
        }
        return true;
    }

    private boolean and(RuleDto dto, List<BaseRule> ruleList) {
        for (BaseRule rule : ruleList) {
            if (! rule.execute(dto)) {
                // and 关系匹配失败一次，返回 false
                return false;
            }
        }
        // and 关系全部匹配成功，返回 true
        return true;
    }

    private boolean or(RuleDto dto, List<BaseRule> ruleList) {
        for (BaseRule rule : ruleList) {
            if ( rule.execute(dto)) {
                // or 关系匹配到一个就返回 true
                return true;
            }
        }
        // or 关系一个都匹配不到就返回 false
        return false;
    }
}
```

### 调用执行器

```java
public class RuleServiceTest {
    @org.junit.Test
    public void execute() {
        //规则执行器
        //优点：比较简单，每个规则可以独立，将规则，数据，执行器拆分出来，调用方比较规整
        //缺点：数据依赖公共传输对象 dto

        //1. 定义规则  init rule
        AgeRule ageRule = new AgeRule();
        NameRule nameRule = new NameRule();
        NationalityRule nationalityRule = new NationalityRule();
        AddressRule addressRule = new AddressRule();
        SubjectRule subjectRule = new SubjectRule();

        //2. 构造需要的数据 create dto
        RuleDto dto = new RuleDto();
        dto.setAge(5);
        dto.setName("张三");
        dto.setAddress("北京");
        dto.setSubject("数学");;

        //3. 通过以链式调用构建和执行 rule execute
        boolean ruleResult = RuleService
                .create()
                .and(Arrays.asList(nationalityRule, nameRule, addressRule))
                .or(Arrays.asList(ageRule, subjectRule))
                .execute(dto);
        System.out.println("this student rule execute result :" + ruleResult);
    }
}
```

### 总结

优点

- 比较简单，每个规则可以独立，将规则，数据，执行器拆分出来，调用方比较规整；
- 我在 Rule 模板类中定义 convert 方法做参数的转换这样可以能够，为特定 rule 需要的场景数据提供拓展。

缺点

- 上下 rule 有数据依赖性，如果直接修改公共传输对象 dto 这样设计不是很合理，建议提前构建数据。

是否可以用责任链模式实现，责任链更像是and条件规则的全部校验。