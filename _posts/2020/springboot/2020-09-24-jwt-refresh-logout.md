---
layout: post
title: spring oauth2+JWT后端自动刷新access_token
category: springboot
tags: [springboot]
keywords: springboot
excerpt: JWT 存储在客户端的认证字符串，无状态化，它可以存储当前登录的用户等认证信息
lock: noneed
---

## 1、前言

spring boot微服务里经常用到 oauth2 和 jwt整合，做用户鉴权，难点在于token的刷新和注销。

### 区别

- spring security

  用户认证（账号密码）与 授权验证（url请求接口权限）的 安全框架，基于RBAC(角色的权限控制)对用户的访问权限进行控制，核心思想是通过一系列的filter chain 来进行拦截过滤的。使用起来就是继承WebSecurityConfigurerAdapter进行权限配置

- oauth2

  一个关于授权token的开放标准，核心思路是通过各类认证手段（shiro、security）认证用户身份，并颁发token令牌，使得第三方应用可以使用该令牌在**限定时间**、**限定范围**访问指定资源。主要涉及的RFC规范有

  1. `RFC6749`（整体授权框架）、

  2. `RFC6750`（令牌使用）、

  3. `RFC6819`（威胁模型）

  一般我们需要了解的就是**`RFC6749`**整体授权框架

  获取令牌的方式主要有四种：授权码模式、简单模式、密码模式、客户端模式

  想了解具体的oauth2授权步骤可以移步阮一峰老师的[理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)，里面有非常详细的说明。

  oauth2 要明确的几个角色概念

  1. 客户应用
  2. 授权服务器，用来进行用户认证并颁发token
  3. 资源服务器，拥有被访问资源的服务器，需要通过token来确定是否有权限访问

- jwt

  json web token 一种token格式，与普通token的区别是它携带了用户信息（状态化的东西），由三部分组成

  1. 头部header
  2. 载体部分payload
  3. 签名signature（自定义一段secret 字符串，指定加密算法生成的签名字符串）

  jwt可以实现分布式的token验证功能，即资源服务器通过事先维护好的对称或者非对称密钥（非对称的话就是认证服务器提供的公钥），直接在本地验证token，去中心化的验证机制正适合分布式架构，它解决了两大痛点：

  1. 通过验证签名，token的验证可以直接在本地完成，不需要连接认证服务器
  2. 在payload中可以定义用户相关信息，轻松实现了token和用户信息的绑定

### JWT的应用场景

就像布鲁克斯在《人月神话》中所说的名言一样：“没有银弹”，jwt不能解决所有的认证问题。

适用场景：充分发挥jwt无状态以及分布式验证的优势

- 一次性的身份认证
- api的鉴权

不适用的场景：单机应用没必要搞jwt

- 传统的基于session的用户会话保持

**不要试图用jwt去代替session。**

这种模式下其实传统的session+cookie机制工作的更好，jwt因为其无状态和分布式，事实上只要在有效期内，是无法作废的，用户的签退更多是一个客户端的签退，服务端token仍然有效，你只要使用这个token，仍然可以登陆系统。另外一个问题是续签问题，使用token，无疑令续签变得十分麻烦，当然你也可以通过redis去记录token状态，并在用户访问后更新这个状态，但这就是硬生生把jwt的无状态搞成有状态了，而这些在传统的session+cookie机制中都是不需要去考虑的。这种场景下，考虑高可用，我更加推荐采用分布式的session机制，现在已经有很多的成熟框架可供选择了（比如spring session）。



## 2、JWT 注销

### 将JWT存储在数据库中

您可以检查哪些令牌有效以及哪些令牌已被撤销，但这在我看来完全违背了使用JWT的目的，因为它把JWT 变成有状态了，中心化了

开源的renren_fast框架也是这么做的，用来对接APP端，注销就把token 从数据库删除

renren框架使用shiro+token做登录认证

Shiroconfig配置类

```java
@Configuration
public class ShiroConfig {
    @Bean("securityManager")
    public SecurityManager securityManager(OAuth2Realm oAuth2Realm) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setRealm(oAuth2Realm);
        securityManager.setRememberMeManager(null);
        return securityManager;
    }

    @Bean("shiroFilter")
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilter = new ShiroFilterFactoryBean();
        shiroFilter.setSecurityManager(securityManager);

        //oauth过滤
        Map<String, Filter> filters = new HashMap<>();
        filters.put("oauth2", new OAuth2Filter());
        shiroFilter.setFilters(filters);

        Map<String, String> filterMap = new LinkedHashMap<>();
        filterMap.put("/webjars/**", "anon");
        filterMap.put("/druid/**", "anon");
        filterMap.put("/app/**", "anon");
        filterMap.put("/sys/login", "anon");
        filterMap.put("/swagger/**", "anon");
        filterMap.put("/v2/api-docs", "anon");
        filterMap.put("/swagger-ui.html", "anon");
        filterMap.put("/swagger-resources/**", "anon");
        filterMap.put("/captcha.jpg", "anon");
        filterMap.put("/images/**", "anon");
        filterMap.put("/upload/**", "anon");
        filterMap.put("/temp/**", "anon");
        filterMap.put("/sys/pay/success", "anon");
        filterMap.put("/sys/pay/cancel","anon");
        filterMap.put("/**", "oauth2");
        shiroFilter.setFilterChainDefinitionMap(filterMap);

        return shiroFilter;
    }

    @Bean("lifecycleBeanPostProcessor")
    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }

    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(securityManager);
        return advisor;
    }
}登录时就生成token保存到数据库，登出就删除token	
```

登录与登出操作

```java
	/**
	 * 登录
	 */
	@PostMapping("/sys/login")
	public Map<String, Object> login(@RequestBody SysLoginForm form)throws IOException {
		boolean captcha = sysCaptchaService.validate(form.getUuid(), form.getCaptcha());
		if(!captcha){
			return R.error("验证码不正确");
		}

		//用户信息
		SysUserEntity user = sysUserService.queryByUserName(form.getUsername());

		//账号不存在、密码错误
		if(user == null || !user.getPassword().equals(new Sha256Hash(form.getPassword(), user.getSalt()).toHex())) {
			return R.error("账号或密码不正确");
		}

		//账号锁定
		if(user.getStatus() == 0){
			return R.error("账号已被锁定,请联系管理员");
		}

		//生成token，并保存到数据库
		R r = sysUserTokenService.createToken(user.getUserId());
		return r;
	}


	/**
	 * 退出
	 */
	@PostMapping("/sys/logout")
	public R logout() {
		sysUserTokenService.logout(getUserId());
		return R.ok();
	}
```

### 从客户端删除令牌

前端把token从cookie中删除，这将阻止客户端进行经过身份验证的请求，但如果令牌仍然有效且其他人可以访问它，则仍可以使用该令牌。

### 刷新令牌

当用户登录时，为他们提供JWT和刷新令牌ref_token。将刷新令牌存储在数据库中。

对于经过身份验证的请求，客户端可以使用JWT，但是当令牌过期（或即将过期）时，让客户端使用刷新令牌发出请求以换取新的JWT。这样，您只需在用户登录或要求新的JWT时访问数据库。当用户注销时，您需要使存储的刷新令牌无效。否则，即使用户已经注销，有人在监听连接时仍然可以获得新的JWT。但注销到JWT过期仍然有一个时间窗口，JWT是依然可用的。这里只是解决了用户无感刷新JWT的问题，客户端携带刷新令牌获取新的JWT。
### 创建JWT黑名单

当后端收到注销请求时，将JWT存储在redis，并设置过期时间等于JWT的剩余存活时间。

对于每个经过身份验证的请求，您需要检查Redis查看令牌是否已失效。根据令牌剩余有效期设置内存数据失效时间，达到自动清除的目的，JWT可以携带一个UUID，作为黑名单中JWT的key，該key对应的value就是redis的过期时间，redis可以使用Map的数据类型存储。

个人觉得JWT黑名单是较好的JWT注销方案。

## 3、JWT 刷新

刷新方案有两种：

- 一种是校验token前刷新
- 第二种是校验失败后刷新。

**为什么要刷新?**

因为JWT并不具有续期的功能，所以在判断token过期后，立刻使用refresh_token刷新。并且在response的header里面添加标识告诉前端你的token实际上已经过期了需要更新。其他的类似memory token、redis token可以延期的，直接延长过期时间并且不需要更新token。

### 方案一校验token前刷新

springboot 版本是2.2.x

pom.xml导入依赖

```xml
<!-- spring security + OAuth2 + JWT    start  -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-jwt</artifactId>
    <version>1.0.9.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.2.1.RELEASE</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

1、重写`JwtAccessTokenConverter`的enhance方法，把refresh_token、client_id、client_secret放入到access_token中，以便刷新

```java
public class OauthJwtAccessTokenConverter extends JwtAccessTokenConverter {
    private JsonParser objectMapper = JsonParserFactory.create();
 
    public OauthJwtAccessTokenConverter(SecurityUserService userService) {
        // 使用SecurityContextHolder.getContext().getAuthentication()能获取到User信息
        super.setAccessTokenConverter(new OauthAccessTokenConverter(userService));
    }
 
    @Override
    public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
        DefaultOAuth2AccessToken result = new DefaultOAuth2AccessToken(accessToken);
        Map<String, Object> info = new LinkedHashMap<String, Object>(accessToken.getAdditionalInformation());
        String tokenId = result.getValue();
        if (!info.containsKey(TOKEN_ID)) {
            info.put(TOKEN_ID, tokenId);
        } else {
            tokenId = (String) info.get(TOKEN_ID);
        }
 
        // access_token 包含自动刷新过期token需要的数据(client_id/secret/refresh_token)
        Map<String, Object> details = (Map<String, Object>) authentication.getUserAuthentication().getDetails();
        if (!Objects.isNull(details) && details.size() > 0) {
            info.put(OauthConstant.OAUTH_CLIENT_ID,
                    details.getOrDefault("client_id", details.get(OauthConstant.OAUTH_CLIENT_ID)));
 
            info.put(OauthConstant.OAUTH_CLIENT_SECRET,
                    details.getOrDefault("client_secret", details.get(OauthConstant.OAUTH_CLIENT_SECRET)));
        }
 
        OAuth2RefreshToken refreshToken = result.getRefreshToken();
        if (refreshToken != null) {
            DefaultOAuth2AccessToken encodedRefreshToken = new DefaultOAuth2AccessToken(accessToken);
            encodedRefreshToken.setValue(refreshToken.getValue());
            // Refresh tokens do not expire unless explicitly of the right type
            encodedRefreshToken.setExpiration(null);
            try {
                Map<String, Object> claims = objectMapper
                        .parseMap(JwtHelper.decode(refreshToken.getValue()).getClaims());
                if (claims.containsKey(TOKEN_ID)) {
                    encodedRefreshToken.setValue(claims.get(TOKEN_ID).toString());
                }
            } catch (IllegalArgumentException e) {
            }
            Map<String, Object> refreshTokenInfo = new LinkedHashMap<String, Object>(
                    accessToken.getAdditionalInformation());
            refreshTokenInfo.put(TOKEN_ID, encodedRefreshToken.getValue());
            // refresh token包含client id/secret, 自动刷新过期token时用到。
            if (!Objects.isNull(details) && details.size() > 0) {
                refreshTokenInfo.put(OauthConstant.OAUTH_CLIENT_ID,
                        details.getOrDefault("client_id", details.get(OauthConstant.OAUTH_CLIENT_ID)));
 
                refreshTokenInfo.put(OauthConstant.OAUTH_CLIENT_SECRET,
                        details.getOrDefault("client_secret", details.get(OauthConstant.OAUTH_CLIENT_SECRET)));
            }
            refreshTokenInfo.put(ACCESS_TOKEN_ID, tokenId);
            encodedRefreshToken.setAdditionalInformation(refreshTokenInfo);
            DefaultOAuth2RefreshToken token = new DefaultOAuth2RefreshToken(
                    encode(encodedRefreshToken, authentication));
            if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
                Date expiration = ((ExpiringOAuth2RefreshToken) refreshToken).getExpiration();
                encodedRefreshToken.setExpiration(expiration);
                token = new DefaultExpiringOAuth2RefreshToken(encode(encodedRefreshToken, authentication), expiration);
            }
            result.setRefreshToken(token);
            info.put(OauthConstant.OAUTH_REFRESH_TOKEN, token.getValue());
        }
        result.setAdditionalInformation(info);
        result.setValue(encode(result, authentication));
        return result;
    }
}
```

2、重写`DefaultTokenServices`的loadAuthentication方法，开始处理刷新

```java
public class OauthTokenServices extends DefaultTokenServices {
    private static final Logger logger = LoggerFactory.getLogger(OauthTokenServices.class);
    private TokenStore tokenStore;
    // 自定义的token刷新处理器
    private TokenRefreshExecutor executor;
    public OauthTokenServices(TokenStore tokenStore, TokenRefreshExecutor executor) {
        super.setTokenStore(tokenStore);
        this.tokenStore = tokenStore;
        this.executor = executor;
    }
 
    @Override
    public OAuth2Authentication loadAuthentication(String accessTokenValue) throws AuthenticationException, InvalidTokenException {
        OAuth2AccessToken accessToken = tokenStore.readAccessToken(accessTokenValue);
        executor.setAccessToken(accessToken);
        // 是否刷新token
        if (executor.shouldRefresh()) {
            try {
                logger.info("refresh token.");
                String newAccessTokenValue = executor.refresh();
                // token如果是续期不做remove操作，如果是重新生成则删除旧的token
                if (!newAccessTokenValue.equals(accessTokenValue)) {
                    tokenStore.removeAccessToken(accessToken);
                }
                accessTokenValue = newAccessTokenValue;
            } catch (Exception e) {
                logger.error("token refresh failed.", e);
            }
        }
 
        return super.loadAuthentication(accessTokenValue);
    }
}
```

自定义的token刷新处理器接口TokenRefreshExecutor主要有两个方法

- shouldRefresh：是否需要刷新
- refresh：刷新

```java
public interface TokenRefreshExecutor {
  /**
     * 是否需要刷新
     * @return
     */
    boolean shouldRefresh();
  
    /**
     * 执行刷新
     * @return
     * @throws Exception
     */
    String refresh() throws Exception;
 
    void setTokenStore(TokenStore tokenStore);
 
    void setAccessToken(OAuth2AccessToken accessToken);
 
    void setClientService(ClientDetailsService clientService);
}
```

3、配置类TokenConfig

```java
@Configuration
public class TokenConfig {
    @Bean
    public TokenStore tokenStore(AccessTokenConverter converter) {
        return new JwtTokenStore((JwtAccessTokenConverter) converter);
        // return new InMemoryTokenStore();
    }
    @Bean
    public AccessTokenConverter accessTokenConverter(SecurityUserService userService) {
        JwtAccessTokenConverter accessTokenConverter = new OauthJwtAccessTokenConverter(userService);
        accessTokenConverter.setSigningKey("sign_key");
        return accessTokenConverter;
        /*DefaultAccessTokenConverter converter = new DefaultAccessTokenConverter();
        DefaultUserAuthenticationConverter userTokenConverter = new DefaultUserAuthenticationConverter();
        userTokenConverter.setUserDetailsService(userService);
        converter.setUserTokenConverter(userTokenConverter);
        return converter;*/
    }
    @Bean
    public TokenRefreshExecutor tokenRefreshExecutor(TokenStore tokenStore,
                                                     ClientDetailsService clientService) {
        TokenRefreshExecutor executor = new OauthJwtTokenRefreshExecutor();
        // TokenRefreshExecutor executor = new OauthTokenRefreshExecutor();
        executor.setTokenStore(tokenStore);
        executor.setClientService(clientService);
        return executor;
    }
    @Bean
    public AuthorizationServerTokenServices tokenServices(TokenStore tokenstore,
                                                          AccessTokenConverter accessTokenConverter,
                                                          ClientDetailsService clientService,
                                                          TokenRefreshExecutor executor) {
 
        OauthTokenServices tokenServices = new OauthTokenServices(tokenstore, executor);
        // 非jwtConverter可注释setTokenEnhancer
        tokenServices.setTokenEnhancer((TokenEnhancer) accessTokenConverter);
        tokenServices.setSupportRefreshToken(true);
        tokenServices.setClientDetailsService(clientService);
        tokenServices.setReuseRefreshToken(true);
        return tokenServices;
    }
}
```

实现类OauthJwtTokenRefreshExecutor.java

```java
public class OauthJwtTokenRefreshExecutor extends AbstractTokenRefreshExecutor {
    private static final Logger logger = LoggerFactory.getLogger(OauthJwtTokenRefreshExecutor.class);
 
    @Override
    public boolean shouldRefresh() {
        // 旧token过期才刷新
        return getAccessToken() != null && getAccessToken().isExpired();
    }
 
    @Override
    public String refresh() throws Exception{
        HttpServletRequest request = ServletUtil.getRequest();
        HttpServletResponse response = ServletUtil.getResponse();
        MultiValueMap<String, Object> parameters = new LinkedMultiValueMap<>();
        // OauthJwtAccessTokenConverter中存入access_token中的数据，在这里使用
        parameters.add("client_id", TokenUtil.getStringInfo(getAccessToken(), OauthConstant.OAUTH_CLIENT_ID));
        parameters.add("client_secret", TokenUtil.getStringInfo(getAccessToken(), OauthConstant.OAUTH_CLIENT_SECRET));
        parameters.add("refresh_token", TokenUtil.getStringInfo(getAccessToken(), OauthConstant.OAUTH_REFRESH_TOKEN));
        parameters.add("grant_type", "refresh_token");
        // 发送刷新的http请求
        Map result = RestfulUtil.post(getOauthTokenUrl(request), parameters);
 
        if (Objects.isNull(result) || result.size() <= 0 || !result.containsKey("access_token")) {
            throw new IllegalStateException("refresh token failed.");
        }
 
        String accessToken = result.get("access_token").toString();
        OAuth2AccessToken oAuth2AccessToken = getTokenStore().readAccessToken(accessToken);
        OAuth2Authentication auth2Authentication = getTokenStore().readAuthentication(oAuth2AccessToken);
        // 保存授权信息，以便全局调用
        SecurityContextHolder.getContext().setAuthentication(auth2Authentication);
 
        // 前端收到该event事件时，更新access_token
        response.setHeader("event", "token-refreshed");
        response.setHeader("access_token", accessToken);
        // 返回新的token信息
        return accessToken;
    }
 
    private String getOauthTokenUrl(HttpServletRequest request) {
        return String.format("%s://%s:%s%s%s",
                request.getScheme(),
                request.getLocalAddr(),
                request.getLocalPort(),
                Strings.isNotBlank(request.getContextPath()) ? "/" + request.getContextPath() : "",
                "/oauth/token");
    }
}
```

4、认证服务器相关代码，相当于SpringSecurity的鉴权

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfiguration extends AuthorizationServerConfigurerAdapter {
 
    @Autowired
    private AuthenticationManager manager;
    @Autowired
    private SecurityUserService userService;
    @Autowired
    private TokenStore tokenStore;
    @Autowired
    private AccessTokenConverter tokenConverter;
    @Autowired
    private AuthorizationServerTokenServices tokenServices;
 
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.tokenStore(tokenStore)
                .authenticationManager(manager)
                .allowedTokenEndpointRequestMethods(HttpMethod.GET, HttpMethod.POST)
                .userDetailsService(userService)
                .accessTokenConverter(tokenConverter)
                .tokenServices(tokenServices);
    }
 
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        security.tokenKeyAccess("permitAll()") //url:/oauth/token_key,exposes public key for token verification if using JWT tokens
                .checkTokenAccess("isAuthenticated()") //url:/oauth/check_token allow check token
                .allowFormAuthenticationForClients();
    }
 
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.withClientDetails(clientDetailsService());
    }
 
    public ClientDetailsService clientDetailsService() {
        return new OauthClientService();
    }
}
```

5、前端使用axios请求

```javascript
// 拦截后端请求的返回
service.interceptors.response.use(res => {
    // 缓存自动刷新生成的新token
    if (res.headers['event'] && "token-refreshed" === res.headers['event']) {
      setToken(res.headers['access_token'])
      store.commit('SET_TOKEN', res.headers['access_token'])
    }
    // 忽略部分代码
}
```

这样就做到了jwt无感刷新。　　

> 项目测试



### 方案二校验失败后刷新

验证失效后，Oauth2框架会把异常信息发送到OAuth2AuthenticationEntryPoint类里处理。这时候我们可以在这里做jwt token刷新并跳转。失效后，使用refresh_token获取新的access_token。并将新的access_token设置到response.header然后跳转，前端接收并无感更新access_token。

参考这两篇文章：

- [https://www.cnblogs.com/xuchao0506/p/13073913.html](https://www.cnblogs.com/xuchao0506/p/13073913.html)

  思路：JWT过期使用刷新token请求获得新的JWT和刷新token，刷新token过期就提示用户重新登录

  github: [https://github.com/xuchao6969/springsecurity-oauth2-jwt](https://github.com/xuchao6969/springsecurity-oauth2-jwt)

- [https://blog.csdn.net/m0_37834471/article/details/83213002](https://blog.csdn.net/m0_37834471/article/details/83213002)

> 问题

使用第二种方案并且jwt token刷新功能正常使用后，想换一种token方案做兼容。

切换成memory token的时候，发现OAuth2AuthenticationEntryPoint里面拿不到旧的token信息导致刷新失败。

查看源码DefaultTokenServices.java

```java
public OAuth2Authentication loadAuthentication(String accessTokenValue) throws AuthenticationException,
InvalidTokenException {
  OAuth2AccessToken accessToken = tokenStore.readAccessToken(accessTokenValue);
  if (accessToken == null) {
    throw new InvalidTokenException("Invalid access token: " + accessTokenValue);
  } else if (accessToken.isExpired()) {
    // 失效后accessToken即被删除
    tokenStore.removeAccessToken(accessToken);
    throw new InvalidTokenException("Access token expired: " + accessTokenValue);
  }

  // 忽略部分代码
  return result;
}
```

看下面截图，可以看到JwtTokenStore的removeAccessToken：它是一个空方法，什么也没做。所以我们在OAuth2AuthenticationEntryPoint依然能拿到旧的token并作处理。

![](\assets\images\2020\springcloud\jwt-1.png)

## 4、Memory token的刷新

memory token刷新策略比较简单，每次请求过来直接给token延期即可。

```java
public class OauthTokenRefreshExecutor extends AbstractTokenRefreshExecutor {
    private int accessTokenValiditySeconds = 60 * 60 * 12;
 
    @Override
    public boolean shouldRefresh() {
        // 与jwt不同，因为每次请求都需要延长token失效时间，所以这里是token未过期时就需要刷新
        return getAccessToken() != null && !getAccessToken().isExpired();
    }
 
    @Override
    public String refresh() {
        int seconds;
        if (getAccessToken() instanceof DefaultOAuth2AccessToken) {
            // 获取client中的过期时间, 没有则默认12小时
            if (getClientService() != null) {
                OAuth2Authentication auth2Authentication = getTokenStore().readAuthentication(getAccessToken());
                String clientId = auth2Authentication.getOAuth2Request().getClientId();
                ClientDetails client = getClientService().loadClientByClientId(clientId);
                seconds = client.getAccessTokenValiditySeconds();
            } else {
                seconds = accessTokenValiditySeconds;
            }
            // 只修改token失效时间
            ((DefaultOAuth2AccessToken) getAccessToken()).setExpiration(new Date(System.currentTimeMillis() + (seconds * 1000l)));
        }
        // 返回的还是旧的token
        return getAccessToken().getValue();
    }
}
```

然后修改TokenConfig相关bean注册即可。
