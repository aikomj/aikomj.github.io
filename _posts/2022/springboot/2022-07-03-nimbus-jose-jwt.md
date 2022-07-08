---
layout: post
title: 推荐一个好用的JWT库nimbus-jose-jwt
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 对称加密、非对称加密token的java实现
lock: noneed
---

## 1、JWT概念关系

这里我们需要了解下JWT、JWS、JWE三者之间的关系，其实JWT(JSON Web Token)指的是一种规范，这种规范允许我们使用JWT在两个组织之间传递安全可靠的信息。而JWS(JSON Web Signature)和JWE(JSON Web Encryption)是JWT规范的两种不同实现，我们平时最常使用的实现就是JWS。

## 2、对称加密HMAC

对称加密指的是使用相同的秘钥来进行加密和解密，如果你的秘钥不想暴露给解密方，考虑使用非对称加密。

1、pom.xml导入依赖

```xml
<dependency>
    <groupId>com.nimbusds</groupId>
    <artifactId>nimbus-jose-jwt</artifactId>
    <version>8.16</version>
</dependency>
```

2、创建JwtTokenServiceImpl作为JWT处理的业务类，添加根据HMAC算法生成和解析JWT令牌的方法，可以发现nimbus-jose-jwt库操作JWT的API非常易于理解

```java
@Service
public class JwtTokenServiceImpl implements JwtTokenService {
    @Override
    public String generateTokenByHMAC(String payloadStr, String secret) throws JOSEException {
        //创建JWS头，设置签名算法和类型
        JWSHeader jwsHeader = new JWSHeader.Builder(JWSAlgorithm.HS256).
                type(JOSEObjectType.JWT)
                .build();
        //将负载信息封装到Payload中
        Payload payload = new Payload(payloadStr);
        //创建JWS对象
        JWSObject jwsObject = new JWSObject(jwsHeader, payload);
        //创建HMAC签名器
        JWSSigner jwsSigner = new MACSigner(secret);
        //签名
        jwsObject.sign(jwsSigner);
        return jwsObject.serialize();
    }
 
    @Override
    public PayloadDto verifyTokenByHMAC(String token, String secret) throws ParseException, JOSEException {
        //从token中解析JWS对象
        JWSObject jwsObject = JWSObject.parse(token);
        //创建HMAC验证器
        JWSVerifier jwsVerifier = new MACVerifier(secret);
        if (!jwsObject.verify(jwsVerifier)) {
            throw new JwtInvalidException("token签名不合法！");
        }
        String payload = jwsObject.getPayload().toString();
        PayloadDto payloadDto = JSONUtil.toBean(payload, PayloadDto.class);
        if (payloadDto.getExp() < new Date().getTime()) {
            throw new JwtExpiredException("token已过期！");
        }
        return payloadDto;
    }
}
```

3、创建PayloadDto实体类，用于封装JWT中存储的信息；

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Builder
public class PayloadDto {
    @ApiModelProperty("主题")
    private String sub;
    @ApiModelProperty("签发时间")
    private Long iat;
    @ApiModelProperty("过期时间")
    private Long exp;
    @ApiModelProperty("JWT的ID")
    private String jti;
    @ApiModelProperty("用户名称")
    private String username;
    @ApiModelProperty("用户拥有的权限")
    private List<String> authorities;
}
```

在JwtTokenServiceImpl类中添加获取默认的PayloadDto的方法，JWT过期时间设置为60s；

```java
@Service
public class JwtTokenServiceImpl implements JwtTokenService {
    @Override
    public PayloadDto getDefaultPayloadDto() {
        Date now = new Date();
        Date exp = DateUtil.offsetSecond(now, 60*60);
        return PayloadDto.builder()
                .sub("macro")
                .iat(now.getTime())
                .exp(exp.getTime())
                .jti(UUID.randomUUID().toString())
                .username("macro")
                .authorities(CollUtil.toList("ADMIN"))
                .build();
    }
}
```

4、创建JwtTokenController类，添加根据HMAC算法生成和解析JWT令牌的接口，由于HMAC算法需要长度至少为32个字节的秘钥，所以我们使用MD5加密下；

```java
@Api(tags = "JwtTokenController", description = "JWT令牌管理")
@Controller
@RequestMapping("/token")
public class JwtTokenController {
 
    @Autowired
    private JwtTokenService jwtTokenService;
 
    @ApiOperation("使用对称加密（HMAC）算法生成token")
    @RequestMapping(value = "/hmac/generate", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult generateTokenByHMAC() {
        try {
            PayloadDto payloadDto = jwtTokenService.getDefaultPayloadDto();
            String token = jwtTokenService.generateTokenByHMAC(JSONUtil.toJsonStr(payloadDto), SecureUtil.md5("test"));
            return CommonResult.success(token);
        } catch (JOSEException e) {
            e.printStackTrace();
        }
        return CommonResult.failed();
    }
 
    @ApiOperation("使用对称加密（HMAC）算法验证token")
    @RequestMapping(value = "/hmac/verify", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult verifyTokenByHMAC(String token) {
        try {
            PayloadDto payloadDto  = jwtTokenService.verifyTokenByHMAC(token, SecureUtil.md5("test"));
            return CommonResult.success(payloadDto);
        } catch (ParseException | JOSEException e) {
            e.printStackTrace();
        }
        return CommonResult.failed();
    }
}
```

5、测试

调用使用HMAC算法生成JWT令牌的接口进行测试；

![](/assets/images/2022/springboot/nimbus-jose-jwt-1.jpg)

调用使用HMAC算法解析JWT令牌的接口进行测试。

![](/assets/images/2022/springboot/nimbus-jose-jwt-2.jpg)

## 3、非对称加密RSA

非对称加密指的是使用公钥和私钥来进行加密解密操作。对于加密操作，公钥负责加密，私钥负责解密，对于签名操作，私钥负责签名，公钥负责验证。非对称加密在JWT中的使用显然属于签名操作。

如果我们需要使用固定的公钥和私钥来进行签名和验证的话，我们需要生成一个证书文件，这里将使用Java自带的keytool工具来生成jks证书文件，该工具在JDK的bin目录下；

![](/assets/images/2022/springboot/nimbus-jose-jwt-3.jpg)

打开CMD命令界面，使用如下命令生成证书文件，设置别名为jwt，文件名为jwt.jks；

```sh
keytool -genkey -alias jwt -keyalg RSA -keystore jwt.jks
```

输入密码为123456，然后输入各种信息之后就可以生成证书jwt.jks文件了；

![](/assets/images/2022/springboot/nimbus-jose-jwt-4.jpg)

1、将证书文件jwt.jks复制到项目的resource目录下，然后需要从证书文件中读取RSAKey，这里我们需要在pom.xml中添加一个Spring Security的RSA依赖

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-rsa</artifactId>
    <version>1.0.7.RELEASE</version>
</dependency>
```

2、然后在JwtTokenServiceImpl类中添加方法，从类路径下读取证书文件并转换为RSAKey对象；

```java
@Service
public class JwtTokenServiceImpl implements JwtTokenService {
    @Override
    public RSAKey getDefaultRSAKey() {
        //从classpath下获取RSA秘钥对
        KeyStoreKeyFactory keyStoreKeyFactory = new KeyStoreKeyFactory(new ClassPathResource("jwt.jks"), "123456".toCharArray());
        KeyPair keyPair = keyStoreKeyFactory.getKeyPair("jwt", "123456".toCharArray());
        //获取RSA公钥
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        //获取RSA私钥
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        return new RSAKey.Builder(publicKey).privateKey(privateKey).build();
    }
}
```

3、我们可以在JwtTokenController中添加一个接口，用于获取证书中的公钥；

```java
@Api(tags = "JwtTokenController", description = "JWT令牌管理")
@Controller
@RequestMapping("/token")
public class JwtTokenController {
 
    @Autowired
    private JwtTokenService jwtTokenService;
    
    @ApiOperation("获取非对称加密（RSA）算法公钥")
    @RequestMapping(value = "/rsa/publicKey", method = RequestMethod.GET)
    @ResponseBody
    public Object getRSAPublicKey() {
        RSAKey key = jwtTokenService.getDefaultRSAKey();
        return new JWKSet(key).toJSONObject();
    }
}
```

调用该接口，查看公钥信息，公钥是可以公开访问的；

![](/assets/images/2022/springboot/nimbus-jose-jwt-5.jpg)

4、在JwtTokenServiceImpl中添加根据RSA算法生成和解析JWT令牌的方法，可以发现和上面的HMAC算法操作基本一致；

```java
@Service
public class JwtTokenServiceImpl implements JwtTokenService {
    @Override
    public String generateTokenByRSA(String payloadStr, RSAKey rsaKey) throws JOSEException {
        //创建JWS头，设置签名算法和类型
        JWSHeader jwsHeader = new JWSHeader.Builder(JWSAlgorithm.RS256)
                .type(JOSEObjectType.JWT)
                .build();
        //将负载信息封装到Payload中
        Payload payload = new Payload(payloadStr);
        //创建JWS对象
        JWSObject jwsObject = new JWSObject(jwsHeader, payload);
        //创建RSA签名器
        JWSSigner jwsSigner = new RSASSASigner(rsaKey, true);
        //签名
        jwsObject.sign(jwsSigner);
        return jwsObject.serialize();
    }
 
    @Override
    public PayloadDto verifyTokenByRSA(String token, RSAKey rsaKey) throws ParseException, JOSEException {
        //从token中解析JWS对象
        JWSObject jwsObject = JWSObject.parse(token);
        RSAKey publicRsaKey = rsaKey.toPublicJWK();
        //使用RSA公钥创建RSA验证器
        JWSVerifier jwsVerifier = new RSASSAVerifier(publicRsaKey);
        if (!jwsObject.verify(jwsVerifier)) {
            throw new JwtInvalidException("token签名不合法！");
        }
        String payload = jwsObject.getPayload().toString();
        PayloadDto payloadDto = JSONUtil.toBean(payload, PayloadDto.class);
        if (payloadDto.getExp() < new Date().getTime()) {
            throw new JwtExpiredException("token已过期！");
        }
        return payloadDto;
    }
}
```

5、在JwtTokenController类，添加根据RSA算法生成和解析JWT令牌的接口，使用默认的RSA钥匙对；

```java
@Api(tags = "JwtTokenController", description = "JWT令牌管理")
@Controller
@RequestMapping("/token")
public class JwtTokenController {
 
    @Autowired
    private JwtTokenService jwtTokenService;
 
    @ApiOperation("使用非对称加密（RSA）算法生成token")
    @RequestMapping(value = "/rsa/generate", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult generateTokenByRSA() {
        try {
            PayloadDto payloadDto = jwtTokenService.getDefaultPayloadDto();
            String token = jwtTokenService.generateTokenByRSA(JSONUtil.toJsonStr(payloadDto),jwtTokenService.getDefaultRSAKey());
            return CommonResult.success(token);
        } catch (JOSEException e) {
            e.printStackTrace();
        }
        return CommonResult.failed();
    }
 
    @ApiOperation("使用非对称加密（RSA）算法验证token")
    @RequestMapping(value = "/rsa/verify", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult verifyTokenByRSA(String token) {
        try {
            PayloadDto payloadDto  = jwtTokenService.verifyTokenByRSA(token, jwtTokenService.getDefaultRSAKey());
            return CommonResult.success(payloadDto);
        } catch (ParseException | JOSEException e) {
            e.printStackTrace();
        }
        return CommonResult.failed();
    }
}
```

6、测试

调用使用RSA算法生成JWT令牌的接口进行测试；

![](/assets/images/2022/springboot/nimbus-jose-jwt-6.jpg)

调用使用RSA算法解析JWT令牌的接口进行测试。

![](/assets/images/2022/springboot/nimbus-jose-jwt-7.jpg)