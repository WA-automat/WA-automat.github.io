---
title: 【微信小程序】：puppyhome
date: 2023-03-20 08:23:13
categories: 开发模块
tags:
    - 微信小程序
    - SpringBoot
    - MySQL
sticky: 120
photo: /2023/03/20/【微信小程序】：puppyhome/puppyhome.png
link_refer:
    -
     url: https://2aurora2.github.io/post/PuppyHome.html
     title: 第一个微信小程序 —— PuppyHome
    -
     url: https://www.w3school.com.cn/sql/sql_join_left.asp
     title: SQL LEFT JOIN 关键字
---

本博客主要介绍的是我和舍友最近开发的一个微信小程序——$puppyhome$

这个小程序是一个用来服务于修勾的平台，主要用户功能有发布、收藏、删除领养公告以及领养自己喜欢的修勾，以及本项目的一大亮点——“拍照识修勾”即用户拍照后平台进行分析得出相似度最高的修勾品种（小程序介绍是复制我荣哥的）

这篇博客主要用来放一些有用的知识点和可复用的代码段。

<!-- more -->

## **可复用的代码**

### **前后端跨域问题解决**

这个项目我们使用的是前后端分离的写法，前端和后端所在域不同，这时通常需要解决前后端跨域问题（$CORS$问题），一般来说把这段代码贴到后端的$config$文件夹就可以了。

```java
package com.puppyhome.backend.config;


import org.springframework.context.annotation.Configuration;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

/**
 * 解决前后端跨域问题的配置类
 *
 * @author WA_automat
 * @since 1.0
 */
@Configuration
public class CorsConfig implements Filter {
	@Override
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
		HttpServletResponse response = (HttpServletResponse) res;
		HttpServletRequest request = (HttpServletRequest) req;

		String origin = request.getHeader("Origin");
		if (origin != null) {
			response.setHeader("Access-Control-Allow-Origin", origin);
		}

		String headers = request.getHeader("Access-Control-Request-Headers");
		if (headers != null) {
			response.setHeader("Access-Control-Allow-Headers", headers);
			response.setHeader("Access-Control-Expose-Headers", headers);
		}

		response.setHeader("Access-Control-Allow-Methods", "*");
		response.setHeader("Access-Control-Max-Age", "3600");
		response.setHeader("Access-Control-Allow-Credentials", "true");

		chain.doFilter(request, response);
	}

	@Override
	public void init(FilterConfig filterConfig) {

	}

	@Override
	public void destroy() {
	}
}
```

### **配置$Redis$**

这里使用的是$Jedis$而不是$RedisTemplate$，主要是不知道为什么$RedisTemplate$一直连不上我本地的$Redis$。

使用$Jedis$的效率也比$RedisTemplate$高。

```java
package com.puppyhome.backend.utils;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

import java.util.ResourceBundle;

/**
 * jedis工具类
 * 用于配置jedis
 */
public class JedisUtils {
	private static final JedisPool jp;
	private static final String host;
	private static final int port;
	private static final int maxTotal;
	private static final int maxIdle;

	// 静态代码块初始化资源
	static {
		//读取配置文件 获得参数值
		ResourceBundle rb = ResourceBundle.getBundle("redis");
		host = rb.getString("redis.host");
		port = Integer.parseInt(rb.getString("redis.port"));
		maxTotal = Integer.parseInt(rb.getString("redis.maxTotal"));
		maxIdle = Integer.parseInt(rb.getString("redis.maxIdle"));
		JedisPoolConfig jpc = new JedisPoolConfig();
		jpc.setMaxTotal(maxTotal);
		jpc.setMaxIdle(maxIdle);
		jp = new JedisPool(jpc, host, port);
	}

	/**
	 * 对外访问接口，提供jedis连接对象，连接从连接池获取
	 */
	public static Jedis getJedis() {
		return jp.getResource();
	}
}
```

### **$JWT$配置**

本文章并不直接操作微信用户的$openId$，而是将其转为$token$形式便以进行加密操作，这里给出我配置$JWT$的代码。

```java
package com.puppyhome.backend.utils;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtBuilder;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;
import java.util.Date;
import java.util.UUID;

@Component
public class JwtUtil {
    public static final long JWT_TTL = 60 * 60 * 1000L * 24 * 3;  // 有效期3天
    public static final String JWT_KEY = "mySecretJwtKey";

    public static String getUUID() {
        return UUID.randomUUID().toString().replaceAll("-", "");
    }

    public static String createJWT(String subject) {
        JwtBuilder builder = getJwtBuilder(subject, null, getUUID());
        return builder.compact();
    }

    private static JwtBuilder getJwtBuilder(String subject, Long ttlMillis, String uuid) {
        SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;
        SecretKey secretKey = generalKey();
        long nowMillis = System.currentTimeMillis();
        Date now = new Date(nowMillis);
        if (ttlMillis == null) {
            ttlMillis = JwtUtil.JWT_TTL;
        }

        long expMillis = nowMillis + ttlMillis;
        Date expDate = new Date(expMillis);
        return Jwts.builder()
                .setId(uuid)
                .setSubject(subject)
                .setIssuer("sg")
                .setIssuedAt(now)
                .signWith(signatureAlgorithm, secretKey)
                .setExpiration(expDate);
    }

    public static SecretKey generalKey() {
        byte[] encodeKey = Base64.getDecoder().decode(JwtUtil.JWT_KEY);
        return new SecretKeySpec(encodeKey, 0, encodeKey.length, "HmacSHA256");
    }

    public static Claims parseJWT(String jwt) throws Exception {
        SecretKey secretKey = generalKey();
        return Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJws(jwt)
                .getBody();
    }
}
```

### **自定义响应体**

由于本项目的前后端并不是同一个人写，数据交互的统一性就变得尤为重要了，这里我们自定义响应体的类型，使得响应的格式统一，便于前端接收。

```java
package com.puppyhome.backend.utils;

import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * 封装了响应报文的信息
 * 后续会使用这个类作为controller的返回值
 *
 * @author CCL
 * @since 2021年12月24日
 */
@Data
@NoArgsConstructor
public class ResponseResult<T> {

	private int code;
	private String message;
	private T data;
	/**
	 * 成功,只能是0
	 */
	private static final int SUCCESS_CODE = 200;
	public static final String SUCCESS_MESSAGE = "操作成功";
	/**
	 * 失败
	 */
	public static final int FAIL_CODE = 400;
	public static final String FAIL_MESSAGE = "操作失败";


	public static <T> ResponseResult<T> success(T data) {
		return new ResponseResult<>(SUCCESS_CODE, SUCCESS_MESSAGE, data);
	}

	public static <T> ResponseResult<T> success(String message, T data) {
		return new ResponseResult<>(SUCCESS_CODE, message, data);
	}

	public static <T> ResponseResult<T> fail() {
		return new ResponseResult<>(FAIL_CODE, FAIL_MESSAGE, null);
	}

	public static <T> ResponseResult<T> fail(String message) {
		return new ResponseResult<>(FAIL_CODE, message, null);
	}

	public static <T> ResponseResult<T> fail(int code, String message) {
		return new ResponseResult<>(code, message, null);
	}

	public ResponseResult(int code, String message, T data) {
		this.code = code;
		this.message = message;
		this.data = data;
	}

}
```

### **调用其他rest消费**

为了调用微信小程序服务端接口还有我通过$python$搭建的$flask$后端的拍照识别接口，我们需要配置$restTemplate$，这个配置也是复制粘贴就行了，就不再赘述。

```java
package com.puppyhome.backend.config;

import org.apache.http.client.HttpClient;
import org.apache.http.conn.HttpClientConnectionManager;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.ClientHttpRequestFactory;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {
    
    /**
     * http连接管理器
     * @return
     */
    @Bean
    public HttpClientConnectionManager poolingHttpClientConnectionManager() {
        /*// 注册http和https请求
        Registry<ConnectionSocketFactory> registry = RegistryBuilder.<ConnectionSocketFactory>create()
                .register("http", PlainConnectionSocketFactory.getSocketFactory())
                .register("https", SSLConnectionSocketFactory.getSocketFactory())
                .build();
        PoolingHttpClientConnectionManager poolingHttpClientConnectionManager = new PoolingHttpClientConnectionManager(registry);*/
        
        PoolingHttpClientConnectionManager poolingHttpClientConnectionManager = new PoolingHttpClientConnectionManager();
        // 最大连接数
        poolingHttpClientConnectionManager.setMaxTotal(500);
        // 同路由并发数（每个主机的并发）
        poolingHttpClientConnectionManager.setDefaultMaxPerRoute(100);
        return poolingHttpClientConnectionManager;
    }
    
    /**
     * HttpClient
     * @param poolingHttpClientConnectionManager
     * @return
     */
    @Bean
    public HttpClient httpClient(HttpClientConnectionManager poolingHttpClientConnectionManager) {
        HttpClientBuilder httpClientBuilder = HttpClientBuilder.create();
        // 设置http连接管理器
        httpClientBuilder.setConnectionManager(poolingHttpClientConnectionManager);
        
        /*// 设置重试次数
        httpClientBuilder.setRetryHandler(new DefaultHttpRequestRetryHandler(3, true));*/
        
        // 设置默认请求头
        /*List<Header> headers = new ArrayList<>();
        headers.add(new BasicHeader("Connection", "Keep-Alive"));
        httpClientBuilder.setDefaultHeaders(headers);*/
        
        return httpClientBuilder.build();
    }
    
    /**
     * 请求连接池配置
     * @param httpClient
     * @return
     */
    @Bean
    public ClientHttpRequestFactory clientHttpRequestFactory(HttpClient httpClient) {
        HttpComponentsClientHttpRequestFactory clientHttpRequestFactory = new HttpComponentsClientHttpRequestFactory();
        // httpClient创建器
        clientHttpRequestFactory.setHttpClient(httpClient);
        // 连接超时时间/毫秒（连接上服务器(握手成功)的时间，超出抛出connect timeout）
        clientHttpRequestFactory.setConnectTimeout(60 * 1000);
        // 数据读取超时时间(socketTimeout)/毫秒（务器返回数据(response)的时间，超过抛出read timeout）
        clientHttpRequestFactory.setReadTimeout(60 * 1000);
        // 连接池获取请求连接的超时时间，不宜过长，必须设置/毫秒（超时间未拿到可用连接，会抛出org.apache.http.conn.ConnectionPoolTimeoutException: Timeout waiting for connection from pool）
        clientHttpRequestFactory.setConnectionRequestTimeout(60 * 1000);
        return clientHttpRequestFactory;
    }
    
    /**
     * rest模板
     * @return
     */
    @Bean
    public RestTemplate restTemplate(ClientHttpRequestFactory clientHttpRequestFactory) {
        // boot中可使用RestTemplateBuilder.build创建
        RestTemplate restTemplate = new RestTemplate();
        // 配置请求工厂
        restTemplate.setRequestFactory(clientHttpRequestFactory);
        return restTemplate;
    }
    
}
```

想要使用$restTemplate$来调用其他服务的接口时，直接像下面这样写就可以了。

```java
JSONObject json = restTemplate.getForEntity(url, JSONObject.class).getBody();
```

### **配置Swagger2以及放行静态文件**

这里主要是用来生成网页版接口文档的，但是由于我们是直接在本地写的代码，这就显得没什么必要了。

1. swagger配置
```java
package com.puppyhome.backend.config;


import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

import java.util.ArrayList;

/**
 * 关于Swagger的配置
 */
@Configuration
@EnableSwagger2
public class SwaggerConfig {
	@Bean
	public Docket docket(Environment environment) {
		//指定在dev/test环境下使用swagger
//		Profiles profiles = Profiles.of("dev", "test");
//		System.out.println(profiles);
//		boolean flag = environment.acceptsProfiles(profiles);
		return new Docket(DocumentationType.SWAGGER_2)
				.apiInfo(apiInfo())
//				.enable(flag)//关闭swagger,默认是true
				.select()
				//RequestHandlerSelectors：配置要扫描的方式，有basePackage("路径")、any():扫描全部，none():全部不扫描
				//RequestHandlerSelectors.withMethodAnnotation():扫描方法上的注解
				//.withClassAnnotation()：扫描类上的注解
				.apis(RequestHandlerSelectors.basePackage("com.puppyhome.backend.controller"))//指定扫描的包
				.build();
	}

	private ApiInfo apiInfo() {
		Contact contact = new Contact("WA_automat", "https://wa-automat.github.io/", "1577696824@qq.com");
		return new ApiInfo(
				"PuppyHome Api",
				"Api Documentation",
				"v1.0",
				"https://github.com/WA-automat/PuppyHome_SpringBoot",
				contact,
				"Apache 2.0",
				"http://www.apache.org/licenses/LICENSE-2.0",
				new ArrayList<>()
		);
	}
}
```

2. 放行静态文件

```java
package com.puppyhome.backend.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;

/**
 * @author cf
 * @Description: 静态资源配置 拦截器添加
 */
@Configuration
public class WebMvcConfigurerAdapter extends WebMvcConfigurationSupport {
	/**
	 * 配置静态资源
	 */
	@Override
	public void addResourceHandlers(ResourceHandlerRegistry registry) {
		registry.addResourceHandler("/static/**").addResourceLocations("classpath:/static/");
		registry.addResourceHandler("/templates/**").addResourceLocations("classpath:/templates/");
		/*放行swagger-ui与bootstrap-ui静态页面*/
		registry.addResourceHandler("/swagger-ui.html")
				.addResourceLocations("classpath:/META-INF/resources/swagger-ui.html");
		registry.addResourceHandler("/webjars/**")
				.addResourceLocations("classpath:/META-INF/resources/webjars/");
		registry.addResourceHandler("/doc.html")
				.addResourceLocations("classpath:/META-INF/resources/doc.html");
		super.addResourceHandlers(registry);
	}
}
```

## **一些简单的$SQL$语法**

左连接：$LEFT \ JOIN$

$LEFT \ JOIN$ 关键字会从左表 ($table$_$name1$) 那里返回所有的行，即使在右表 ($table$_$name2$) 中没有匹配的行。

常用在使用一对多或多对多的结构表中。

```mysql
SELECT * FROM table_name1 LEFT JOIN table_name2 ON table_name1.column_name = table_name2.column_name
```

## **项目源码**

看情况开源后端，但是荣哥应该会开源前端代码的！
