---
title: Spring Oauth2 入门
date: 2016-02-28 21:55
---

如果我们的客户端需要对用户 Github 进行一些操作，比如读取用户项目列表、上传 SSH Key 等，需要先获得 Gtihub 的授权。而 Github 使用 Oauth 2 进行授权。

本篇博客要介绍的 Spring Oauth2 是 Oauth 2.0 的实现，它简化了大部分的操作, 比如获得 access token、刷新 access token 的过程，使得通过简单配置，就可以轻松接入第三方的 Oauth 2.0 服务。

如果不了解 Oauth 2 的原理，推荐阅读[阮一峰的文章](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)。

{% asset_img bg2016022801.png %}

<!-- more -->

## 一、创建 Gtihub 应用

了解完 Oauth2 原理，我们通过一个 demo，了解 spring oauth2 使用方法。

登录 Github 后，依次选择 Settings -> OAuth applications -> Developer applications -> Register new application, 然后配置应用的信息, 依次填写 Application Name、Homepage URL、Application description、Authorization callback URL后，选择 Regiser application。

{% asset_img bg2016022803.png github application %}

其中 Authorization callback URL 需要特别注意下，当我们请求 Github 授权时，有一个可选参数 `redirect_uri`，该参数指定了 Github 完成授权后重定向的网址。如果省略该参数，Github 会默认重定向到 Authorization callback URL。除此之外 Authorization callback URL 对 redirect_uri 也有一定的[限制作用](https://developer.github.com/v3/oauth/#redirect-urls)，即 redirect_uri 只能是 Authorization callback URL 的子路径。

创建完 Github 应用，我们就得到了 Github 分发的 Client Id 与 Client Secret。

{% asset_img bg2016022802.png IdAndSecret %}

## 二、为项目添加 maven 依赖

```
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.0.9.RELEASE</version>
</dependency>
```

## 三、开启注解配置

```
@Configuration
@EnableOAuth2Client
public class OAuthConfig {

}
```

该注解提供将会做两件事：

1. 注册一个 Filter（oauth2ClientContextFilter）
2. 创建一个作用域为 request、类型为 AccessTokenRequest 的 bean。

## 三、配置客户端信息

OAuth 2.0 有四种授权方式。分别为：

> 授权码模式（authorization code）  
> 简化模式（implicit）  
> 密码模式（resource owner password credentials）  
> 客户端模式（client credentials）  

spring 提供了 4 种 bean，依次与这四种授权方式对应，用来设定每种方式的配置信息。

这 4 个 bean 依次为：AuthorizationCodeResourceDetails、ImplicitResourceDetails、ImplicitResourceDetails、ClientCredentialsResourceDetails。

我们使用授权码模式配置 github 的 id、secret、申请认证地址、申请令牌地址、[scope](https://developer.github.com/v3/oauth/#scopes)等信息，可以这么写：

```
@Bean
public OAuth2ProtectedResourceDetails github() {
    AuthorizationCodeResourceDetails details = new AuthorizationCodeResourceDetails();
    details.setId("github");
    details.setClientId("your client id");
    details.setClientSecret("your client secret");
    details.setAccessTokenUri("https://github.com/login/oauth/access_token");
    details.setUserAuthorizationUri("https://github.com/login/oauth/authorize");
    details.setScope(Arrays.asList("repo", "user:email", "write:public_key", "read:public_key"));
    return details;
}
```

## 四、配置 Oauth2Clientcontext

```
@Resource(name = "accessTokenRequest")
private AccessTokenRequest accessTokenRequest;

@Bean
@Scope(value = "session", proxyMode = INTERFACES)
public OAuth2ClientContext githubOauth2ClientContext() {
    return new DefaultOAuth2ClientContext(accessTokenRequest);
}
```

accessTokenRequest 的配置信息在 `OAuth2ClientConfiguration` 中, accessTokenRequest 的作用域是 request，即每一次 HTTP 请求都会产生一个新的 accessTokenRequest。

这里的 `@Scope(value = "session", proxyMode = INTERFACES)`， session 的意思是该 bean 的作用域为 session 级别，而 `proxyMode = INTERFACES` 保证了，即使该类被比作用域大于 session 的 bean 所持有，仍能保证在不同的 session，会构造不同的 OAuth2ClientContext。

## 五、配置 OAuth2RestTemplate

```
@Bean
public OAuth2RestOperations githubRestTemplate(@Qualifier("githubOauth2ClientContext") OAuth2ClientContext clientContext) {
    return new OAuth2RestTemplate(github(), clientContext);
}
```

通常，我们需要接入多个 oauth2 服务，因此会有多个 clientContext，这里我们使用 Qualifier 注解按照 bean 的名称注入。

如果我们确认只接入一个 oauth2 服务，那么就可以完全省略第四步，然后将第五步改为下面的代码：

```
@Bean
public OAuth2RestOperations githubRestTemplate(OAuth2ClientContext clientContext) {
    return new OAuth2RestTemplate(github(), clientContext);
}
```

可以直接注入 clientContext 的原因是 `OAuth2ClientConfiguration` 类已经默认为我们提供了。

## 六、配置 OAuth2RestTemplate

得到了 OAuth2RestTemplate，我们只需要使用 OAuth2RestTemplate 请求对应的服务就可以了。

比如要获取用户 github 所有的项目：

```
JsonObject repos = githubRestTemplate.getForObject("https://api.github.com/user/repos", JsonObject.class);
```

## 源码

demo 中包含了 Github 与 Bbitbucket 的 Oauth2.0 授权以及 Bitbucket 的 Oauth 1.0 授权配置。

[https://github.com/tanhe123/SpringOAuthSeample](https://github.com/tanhe123/SpringOAuthSeample)

## 参考

https://github.com/spring-projects/spring-security-oauth 中的 samples  
http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html  
https://developer.github.com/v3/  
http://projects.spring.io/spring-security-oauth/docs/oauth2.html  
http://www.oschina.net/translate/oauth-2-developers-guide  
