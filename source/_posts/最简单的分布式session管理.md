---
title: 最简单的分布式session管理
date: 2018-01-20 19:08:15
update: 2018-01-20 19:08:15
categories: 互联网技术
tags: [互联网技术]
---
前几天和朋友讨论前后端分离和分布式session问题。  
可以用一种最简单的办法来实现: 把用户验证信息保存在redis，设置过期时间。当用户有任何操作时，刷新redis过期时间。 超时时间内用户没有任何操作，则验证信息失效，需要重新登录。这样可以做到模拟单机环境session的功能。      

可以用到下面的场景：  
1，对于分布式网站，其应用服务器(如tomcat)不止一台，请求在不同的服务器跳转，需要保持服务器之间的session同步。  
2，前后端分离的用户验证：前端把userId和password，提交到服务端的登录api，服务端验证正确后，生成一个token，并把token和userId，存在缓存里redis，然后把token返回给前端。前端每次的请求头中带token。  

直接上代码：  
1，保存数据到redis

```
<dependency>
	<groupId>redis.clients</groupId>
	<artifactId>jedis</artifactId>
	<version>2.9.0</version>
</dependency>
```

```
/**
 * 保存key-value到redis
 * @param key
 * @param value
 * @param seconds 以秒为单位
 */
public synchronized static void set(String key, String value, int seconds) {
    try {
        value = StringUtils.isBlank(value) ? "" : value;
        Jedis jedis = getJedis();
        jedis.setex(key, seconds, value);
        jedis.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
2，简单的模拟登录controller,把sessionID和用户名存到redis, 设置过期时间为10分钟。  

```
@RequestMapping("/login")
@ResponseBody
public String login(String userName,String passwd,HttpServletRequest request){
    if("admin".equals(userName) && "admin".equals(passwd)){
        HttpSession session = request.getSession();
        RedisUtil.set( "login_user_session_" + session.getId(),userName,600);
        LOGGER.debug("save the session to redis:"+session.getId());
        return "hello";
    }else{
        return "error";
    }
}
```
3, Filter刷新redis session， 当用户有操作时，刷新session。  
```
public void doFilter(ServletRequest sRequest, ServletResponse sResponse,
		FilterChain filterChain) throws IOException, ServletException {
	HttpServletRequest request = (HttpServletRequest) sRequest;
	HttpServletResponse response = (HttpServletResponse) sResponse;
	response.setContentType("text/html;charset=utf-8");
	response.setCharacterEncoding("utf-8");
	request.setCharacterEncoding("utf-8");
	HttpSession session = request.getSession();
	LOGGER.debug(" ===> AccessControlFilter");
	try{
		String path = request.getRequestURI();
		LOGGER.debug("=======>"+path);
		String redidsession = RedisUtil.get("login_user_session_" + session.getId());
		LOGGER.debug("get session["+session.getId()+"] from redis:"+redidsession);
		if(redidsession!=null){
			//refresh the session
			RedisUtil.set(SHOP_WEB_SERVER_SESSION_ID+ "_" + session.getId(),redidsession,60);
			LOGGER.debug("refresh the session to redis:"+session.getId());
		}

		filterChain.doFilter(sRequest, sResponse);
	}catch(Exception e){
		LOGGER.error(e.getMessage(),e);
	}
}
```

最简单的思路。实际的业务场景肯定比这复杂的多。灵活变化即可。  