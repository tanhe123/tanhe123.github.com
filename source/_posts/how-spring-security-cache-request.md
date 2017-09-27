---
title: 探究 Spring Security 缓存请求
date: 2015-12-07 22:58:00
---

## 为什么要缓存?

为了更好的描述问题，我们拿使用表单认证的网站举例，简化后的认证过程分为7步:

1. 用户访问网站，打开了一个链接(origin url)。

2. 请求发送给服务器，服务器判断用户请求了受保护的资源。

3. 由于用户没有登录，服务器重定向到登录页面

4. 填写表单，点击登录

5. 浏览器将用户名密码以表单形式发送给服务器

6. 服务器验证用户名密码。成功，进入到下一步。否则要求用户重新认证（第三步）

7. 服务器对用户拥有的权限（角色）判定: 有权限，重定向到 origin url; 权限不足，返回状态码 403("forbidden").

从第3步，我们可以知道，用户的请求被中断了。

用户登录成功后（第7步），会被重定向到 origin url，spring security 通过使用缓存的 request，使得被中断的请求能够继续执行。

## 使用缓存

用户登录成功后，页面重定向到 origin url。浏览器发出的请求优先被拦截器 RequestCacheAwareFilter 拦截，RequestCacheAwareFilter 通过其持有的 RequestCache 对象实现 request 的恢复。

```
public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {

	// request匹配，则取出，该操作同时会将缓存的request从session中删除
	HttpServletRequest wrappedSavedRequest = requestCache.getMatchingRequest((HttpServletRequest) request, (HttpServletResponse) response);

	// 优先使用缓存的request
	chain.doFilter(wrappedSavedRequest == null ? request : wrappedSavedRequest, response);
}
```

## 何时缓存

首先，我们需要了解下 RequestCache 以及 ExceptionTranslationFilter。

### RequestCache

RequestCache 接口声明了缓存与恢复操作。默认实现类是 `HttpSessionRequestCache`。HttpSessionRequestCache 的实现比较简单，这里只列出接口的声明:

```
public interface RequestCache {

	// 将request缓存到session中
	void saveRequest(HttpServletRequest request, HttpServletResponse response);

	// 从session中取request
	SavedRequest getRequest(HttpServletRequest request, HttpServletResponse response);

	// 获得与当前request匹配的缓存，并将匹配的request从session中删除
	HttpServletRequest getMatchingRequest(HttpServletRequest request,
			HttpServletResponse response);

	// 删除缓存的request
	void removeRequest(HttpServletRequest request, HttpServletResponse response);
}
```

### ExceptionTranslationFilter

ExceptionTranslationFilter 是 Spring Security 的核心 filter 之一，用来处理 AuthenticationException 和 AccessDeniedException 两种异常。

在我们的例子中，AuthenticationException 指的是未登录状态下访问受保护资源，AccessDeniedException 指的是登陆了但是由于权限不足(比如普通用户访问管理员界面）。

ExceptionTranslationFilter 持有两个处理类，分别是 AuthenticationEntryPoint 和 AccessDeniedHandler。

ExceptionTranslationFilter 对异常的处理是通过这两个处理类实现的，处理规则很简单：

> **规则1:** 如果异常是 AuthenticationException，使用 AuthenticationEntryPoint 处理  
> **规则2:** 如果异常是 AccessDeniedException 且用户是匿名用户，使用 AuthenticationEntryPoint 处理  
> **规则3:** 如果异常是 AccessDeniedException 且用户不是匿名用户，如果否则交给 AccessDeniedHandler 处理  

对应以下代码

```
private void handleSpringSecurityException(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, RuntimeException exception)
			throws IOException, ServletException {
	if (exception instanceof AuthenticationException) {
		logger.debug(
			"Authentication exception occurred; redirecting to authentication entry point", exception);

			sendStartAuthentication(request, response, chain, (AuthenticationException) exception);
		}
	else if (exception instanceof AccessDeniedException) {
		if (authenticationTrustResolver.isAnonymous(SecurityContextHolder.getContext().getAuthentication())) {
			logger.debug("Access is denied (user is anonymous); redirecting to authentication entry point",	exception);

			sendStartAuthentication(request, response, chain, new InsufficientAuthenticationException("Full authentication is required to access this resource"));
		}
		else {
			logger.debug(
				"Access is denied (user is not anonymous); delegating to AccessDeniedHandler", exception);

			accessDeniedHandler.handle(request, response, (AccessDeniedException) exception);
		}
	}
}
```

AccessDeniedHandler 默认实现是 AccessDeniedHandlerImpl。该类对异常的处理是返回403错误码。

```
public void handle(HttpServletRequest request, HttpServletResponse response,
			AccessDeniedException accessDeniedException) throws IOException,
			ServletException {
	if (!response.isCommitted()) {
		if (errorPage != null) {  // 定义了errorPage
			// errorPage中可以操作该异常
			request.setAttribute(WebAttributes.ACCESS_DENIED_403,
					accessDeniedException);

			// 设置403状态码
			response.setStatus(HttpServletResponse.SC_FORBIDDEN);

			// 转发到errorPage
			RequestDispatcher dispatcher = request.getRequestDispatcher(errorPage);
			dispatcher.forward(request, response);
		}
		else { // 没有定义errorPage，则返回403状态码(Forbidden)，以及错误信息
			response.sendError(HttpServletResponse.SC_FORBIDDEN,
					accessDeniedException.getMessage());
		}
	}
}
```

AuthenticationEntryPoint 默认实现是 LoginUrlAuthenticationEntryPoint, 该类的处理是转发或重定向到登录页面

```
public void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException {

	String redirectUrl = null;

	if (useForward) {

		if (forceHttps && "http".equals(request.getScheme())) {
			// First redirect the current request to HTTPS.
			// When that request is received, the forward to the login page will be
			// used.
			redirectUrl = buildHttpsRedirectUrlForRequest(request);
		}

		if (redirectUrl == null) {
			String loginForm = determineUrlToUseForThisRequest(request, response,
					authException);

			if (logger.isDebugEnabled()) {
				logger.debug("Server side forward to: " + loginForm);
			}

			RequestDispatcher dispatcher = request.getRequestDispatcher(loginForm);

			// 转发
			dispatcher.forward(request, response);

			return;
		}
	}
	else {
		// redirect to login page. Use https if forceHttps true
		redirectUrl = buildRedirectUrlToLoginPage(request, response, authException);
	}

	// 重定向
	redirectStrategy.sendRedirect(request, response, redirectUrl);
}
```

了解完这些，回到我们的例子。

第3步时，用户未登录的情况下访问受保护资源，ExceptionTranslationFilter 会捕获到 AuthenticationException 异常(规则1)。页面需要跳转，ExceptionTranslationFilter 在跳转前使用 requestCache 缓存 request。

```
protected void sendStartAuthentication(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain,
			AuthenticationException reason) throws ServletException, IOException {
	// SEC-112: Clear the SecurityContextHolder's Authentication, as the
	// existing Authentication is no longer considered valid
	SecurityContextHolder.getContext().setAuthentication(null);
	// 缓存 request
	requestCache.saveRequest(request, response);
	logger.debug("Calling Authentication entry point.");
	authenticationEntryPoint.commence(request, response, reason);
}
```

## 一些坑

在开发过程中，如果不理解 Spring Security 如何缓存 request，可能会踩一些坑。

举个简单例子，如果网站认证是信息存放在 header 中。第一次请求受保护资源时，请求头中不包含认证信息 ，验证失败，该请求会被缓存，之后即使用户填写了信息，也会因为request被恢复导致信息丢失从而认证失败(问题描述可以参见[这里](http://stackoverflow.com/questions/33568237/how-to-disable-or-override-requestcacheawarefilter-in-spring-boot)。

最简单的方案当然是不缓存request。

spring security 提供了 NullRequestCache， 该类实现了 RequestCache 接口，但是没有任何操作。

```
public class NullRequestCache implements RequestCache {

	public SavedRequest getRequest(HttpServletRequest request,
			HttpServletResponse response) {
		return null;
	}

	public void removeRequest(HttpServletRequest request, HttpServletResponse response) {

	}

	public void saveRequest(HttpServletRequest request, HttpServletResponse response) {
	}

	public HttpServletRequest getMatchingRequest(HttpServletRequest request,
			HttpServletResponse response) {
		return null;
	}
}
```

配置 requestCache，使用如下代码即可:

```
http.requestCache().requestCache(new NullRequestCache());
```

## 补充

默认情况下，三种 request 不会被缓存。

1. 请求地址以 `/favicon.ico` 结尾
2. header中的 `content-type` 值为 `application/json`
3. header中的 `X-Requested-With` 值为 `XMLHttpRequest`

可以参见：`RequestCacheConfigurer` 类中的私有方法 `createDefaultSavedRequestMatcher`。

附上实例代码: [https://coding.net/u/tanhe123/p/SpringSecurityRequestCache](https://coding.net/u/tanhe123/p/SpringSecurityRequestCache)
