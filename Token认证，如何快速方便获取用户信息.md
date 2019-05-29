---
title: Token认证，如何快速方便获取用户信息
date: 2019-05-20 14:31:29:029
tags: [Java] 
published: true
feature: https://user-gold-cdn.xitu.io/2019/5/19/16ace69edcd28f75?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文转载自 [https://juejin.im/post/5ce0dfb45188250c942f6c1d](https://juejin.im/post/5ce0dfb45188250c942f6c1d) 

# 背景

我们有一个Web项目，这个项目提供了很多的Rest API。也做了权限控制，访问API的请求必须要带上事先认证后获取的Token才可以。

认证的话就在Filter中进行的，会获取请求的Token进行验证，如果成功了可以得到Token中的用户信息，本文的核心就是讲解如何将用户信息（用户ID）优雅的传递给API接口（Controller）。

# 方式一（很挫）

我们在Filter中进行了统一拦截，在Controller中获取用户ID的话，仍然可以再次解析一遍Token获取用户ID
```
@GetMapping("/hello")
public String test(HttpServletRequest request) {
    String token = request.getHeader("token");
    JWTResult result = JWTUtils.checkToken(token);
    Long userId = result.getUserId();
}
```

# 方式二（优雅）

方式一需要重新解析一遍Token, 浪费资源。我们可以直接将Filter中解析好了的用户ID直接通过Header传递给接口啊。
```
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
    HttpServletRequest httpRequest = (HttpServletRequest) request;
	HttpServletResponse httpResponse = (HttpServletResponse) response;
    String token = request.getHeader("token");
    JWTResult result = JWTUtils.checkToken(token);
    Long userId = result.getUserId();
	HttpServletRequestWrapper requestWrapper = new HttpServletRequestWrapper(httpRequest) {
		@Override
		public String getHeader(String name) {
			if (name.equals("loginUserId")) {
				return userId .toString();
			}
			return super.getHeader(name);
		}
	};
	chain.doFilter(requestWrapper, httpResponse);
}
```

接口中直接从Header中获取解析好了的用户ID:

```
@GetMapping("/hello")
public String save2(HttpServletRequest request) {
	Long userId = Long.parseLong(request.getHeader("loginUserId"));	
}
```

# 方式三（很优雅）

通过Header传递确实很方便，但如果你有代码洁癖的话总会觉得怪怪的，能不能不用Header方式，比如说我就在方法上定义一个loginUserId的参数，你给我直接注入进来，这个有点意思哈，下面我们来实现下：

### GET参数方式

在Filter中追加参数：
```
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
    HttpServletRequest httpRequest = (HttpServletRequest) request;
	HttpServletResponse httpResponse = (HttpServletResponse) response;
    String token = request.getHeader("token");
    JWTResult result = JWTUtils.checkToken(token);
    Long userId = result.getUserId();
	HttpServletRequestWrapper requestWrapper = new HttpServletRequestWrapper(httpRequest) {
		    @Override
			public String[] getParameterValues(String name) {
				if (name.equals("loginUserId")) {
					return new String[] { userId .toString() };
				}
				return super.getParameterValues(name);
			}
			@Override
			public Enumeration<String> getParameterNames() {
				Set<String> paramNames = new LinkedHashSet<>();
				paramNames.add("loginUserId");
				Enumeration<String> names =  super.getParameterNames();
				while(names.hasMoreElements()) {
					paramNames.add(names.nextElement());
				}
				return Collections.enumeration(paramNames);
			}
	};
	chain.doFilter(requestWrapper, httpResponse);
}
```

接口中直接填写参数即可获取：

```
@GetMapping("/hello")
public String save2(String name, Long loginUserId) {
	// loginUserId 就是Filter中追加的值
}
```

对于post请求，也可以用这种方式：

```
@PostMapping("/hello")
public String save2(User user, Long loginUserId) {
	
}
```

可是往往我们在用post请求的时候，要么就是表单提交，要么就是json体的方式提交，一般不会使用get方式参数，这也就意味着这个loginUserId我们需要注入到对象中：

先创建一个参数实体类:
```
public class User {

	private String name;
	
	private Long loginUserId;
}
```

先模拟表单提交的方式，看看行不行：

```
@PostMapping("/hello")
public User save2(User user) {
	return user;
}
```

用PostMan测试一下，表单方式是直接支持的：

![](https://user-gold-cdn.xitu.io/2019/5/19/16ace69edcd28f75?imageView2/0/w/1280/h/960/ignore-error/1)

再次试下Json提交方式：
```
@PostMapping("/hello")
public User save2(@RequestBody User user) {
	return user;
}
```

看下图，失败了，得重新想办法实现下

![](https://user-gold-cdn.xitu.io/2019/5/19/16ace69edcc90ca7?imageView2/0/w/1280/h/960/ignore-error/1)

只需要在HttpServletRequestWrapper中重新对提交的内容进行修改即可：
```
@Override
public ServletInputStream getInputStream() throws IOException {
	byte[] requestBody = new byte[0];
	try {
		requestBody = StreamUtils.copyToByteArray(request.getInputStream());
		Map map = JsonUtils.toBean(Map.class, new String(requestBody));
		map.put("loginUserId", loginUserId);
		requestBody = JsonUtils.toJson(map).getBytes();
	} catch (IOException e) {
		throw new RuntimeException(e);
	}
	final ByteArrayInputStream bais = new ByteArrayInputStream(requestBody);
	return new ServletInputStream() {
		 @Override
		  public int read() throws IOException {
		       return bais.read();
		  }
		 
		  @Override
		   public boolean isFinished() {
		       return false;
		   }
		 
		   @Override
		   public boolean isReady() {
		        return true;
		   }
		 
		   @Override
		    public void setReadListener(ReadListener listener) {
		 
		    }
    };
}
```

到此为止，我们就可以直接将Token解析的用户ID直接注入到参数中了，不用去Header中获取，是不是很方便。

![猿天地](https://user-gold-cdn.xitu.io/2019/5/15/16ab91d71d2ab715?imageView2/0/w/1280/h/960/ignore-error/1)

