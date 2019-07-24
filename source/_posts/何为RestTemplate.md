title: RestTemplate学习
tags:
  - spring
  - RestTemplate
  - Java
categories:
  - 基础知识
author: 天渊
date: 2019-01-21 11:18:00
---
`RestTemplate`是spring framework中对Http请求封装的一套方法，广泛运用于springboot和springcloud中的Http数据传输，官方文档描述如下：

> `RestTemplate` is a synchronous client to perform HTTP requests. It is the original Spring REST client and exposes a simple, template-method API over underlying HTTP client libraries.

<!-- more -->

不过在spring 5.0发行版中推出了基于NIO的响应式Http客户端：`WebClient`，将在未来代替RestTemplate:

> As of 5.0, the non-blocking, reactive `WebClient` offers a modern alternative to the `RestTemplate`, with efficient support for both synchronous and asynchronous, as well as streaming scenarios. The `RestTemplate` will be deprecated in a future version and will not have major new features added going forward.

### RestTemplate的特点
- 使用方便：可直接传递Java实体对象，无需人工配置ContentType和Charset，自动识别序列化方式和消息格式，无需用户自己encode url，并对多种http传输方法进行了相应封装

- 可扩展性强：RestTemplate可配置多种底层通信框架如JDK HttpURLConnection、Apache HttpClient、OkHttp以及netty等

- 可配置性强：用户可灵活多种第三方组件，除了底层通信框架，还可扩展配置消息转换器，异常处理器，uri模板解析器和请求拦截器等组件

- RestTemplate的多种组件过于依赖spring-framework，如果脱离了spring环境，用起来就很不方便了



# RestTemplate使用方法

## 1. 初始化
RestTemplate有两种初始化方式：`构造方法形式`，`RestTemplateBuilder形式`

### 构造方法形式

```java
@Bean
public RestTemplate restTemplate(){
	return new RestTemplate();
}
```



RestTemplate有三种构造方法：

- 无参数构造方法，也是使用得最普遍的一个：

  ```java
  public RestTemplate() {
  	this.messageConverters.add(new ByteArrayHttpMessageConverter());
  	this.messageConverters.add(new StringHttpMessageConverter());
  	this.messageConverters.add(new ResourceHttpMessageConverter());
  	this.messageConverters.add(new SourceHttpMessageConverter<Source>());
  	this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
  	if (romePresent) {
  		this.messageConverters.add(new AtomFeedHttpMessageConverter());
  		this.messageConverters.add(new RssChannelHttpMessageConverter());
  	}
  	if (jaxb2Present) {
  		this.messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
  	}
  	if (jackson2Present) {
  		this.messageConverters.add(new MappingJackson2HttpMessageConverter());
  	}
  	else if (jacksonPresent) {
  		this.messageConverters.add(new MappingJacksonHttpMessageConverter());
  	}
  }
  ```

  RestTemplate中内置了多种HttpMessageConverter，用于对不同场景下的输入输出流进行序列化和反序列化，无参数构造方法主要对HttpMessageConverter列表进行初始化

- 有参数构造方法 — 初始化ClientHttpRequestFactory

  ```java
  public RestTemplate(ClientHttpRequestFactory requestFactory) {
  	this();
  	setRequestFactory(requestFactory);
  }
  ```

  用户可以自己指定需要的ClientHttpRequestFactory，用于进行http连接和请求，默认是采用`SimpleClientHttpRequestFactory`，底层封装的是JDK的`HttpURLConnection`，用户可以指定其他种类的factory，比如以下几种：

  - `BufferingClientHttpRequestFactory`：可在内存中建立输入数据的缓存

  - `HttpComponentsClientHttpRequestFactory`：采用Apache的HttpClient进行远程调用，可以配置连接池和证书信息，不过需要在pom.xml中加入以下依赖：

    ```xml
    <dependency>
        <groupId>org.apache.httpcomponents</groupId>
        <artifactId>httpclient</artifactId>
        <version>4.5.2</version>
    </dependency>
    ```

  - `InterceptingClientHttpRequestFactory`：可以配置ClientHttpRequestInterceptor拦截器对http请求进行拦截处理，springcloud中的Ribbon就用到了这个factory用于将server name url转换为实际调用的url
  - 基于Netty4的`Netty4ClientHttpRequestFactory`
  - 基于OkHttp2的`OkHttpClientHttpRequestFactory`

- 有参数构造方法 — 添加多个HttpMessageConverter

  ```java
  public RestTemplate(List<HttpMessageConverter<?>> messageConverters) {
  	Assert.notEmpty(messageConverters, "'messageConverters' must not be empty");
  	this.messageConverters.addAll(messageConverters);
  }
  ```

  如果用户觉得RestTemplate默认的几个序列化API无法满足要求，可以自己指定MessageConverter

### RestTemplateBuilder形式
如果用户要对RestTemplate进行多种初始化配置的话，推荐使用RestTemplateBuilder建造器，属于高级用法：
  ```java
    @Bean
	public RestTemplate myRestTemplate() {
		RestTemplateBuilder builder = new RestTemplateBuilder();
		RestTemplate restTemplate = builder
            			//配置ClientHttpRequestFactory
						.requestFactory(HttpComponentsClientHttpRequestFactory.class)
            			//配置MessageConverter
						.messageConverters(new MappingJackson2HttpMessageConverter())
            			//配置ResponseErrorHandler
						.errorHandler(new DefaultResponseErrorHandler())
            			//配置UriTemplateHandler
						.uriTemplateHandler(new DefaultUriTemplateHandler())
            			//配置连接超时时间和连接过期时间
						.setConnectTimeout(10000)
						.setReadTimeout(5000)
						.build();
        return restTemplate;
	}
  ```
## 2. 执行Http请求

RestTemplate对多种http method的请求进行了封装，用户可以直接进行使用

- RestTemplate中几种常用的方法：

| 方法名          | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| getForObject    | 通过get方法获取资源，返回用户指定的Object对象类型            |
| getForEntity    | 通过get方法获取资源，返回一个封装好的HttpEntiry对象          |
| postForObject   | 通过post方法发送Object对象，返回用户指定的Object对象类型     |
| postForEntity   | 通过post方法发送Object对象，返回封装好的HttpEntiry对象       |
| put             | 通过put方法上载资源                                          |
| delete          | 通过delete方法删除服务器数据                                 |
| optionsForAllow | 获取目的资源支持的method                                     |
| exchange        | 一种比较通用的方法，接受的参数分别为url，method，response数据类型，url参数，以及一个封装了http header数据和body数据的HttpEntity对象，统一返回一个封装了所有response数据的ResponseEntity对象 |
| execute         | 该方法是RestTemplate中通用性最强的方法，以上所有方法最终都调用的是execute方法，接受的参数包括RequestCallback对象（用于选择合适的MessageConverter对requestBody数据进行解析并将结果封装到ClientHttpRequest中，进行最终的http请求），以及ResponseExtractor对象（用于将response数据解析为用户需要的数据类型） |

- get方法

  发起get请求，如果只是想获取ResponseBody数据的话直接采用getForObject()的一系列重载方法：

  - `Object`方式

    将返回数据封装到指定的实体类CommonResponse中，RestTemplate会自动进行序列化和反序列化

  ```java
  CommonResponse response = restTemplate.getForObject(url, CommonResponse.class);
  CommonResponse response = restTemplate.getForObject(url, CommonResponse.class, pathVariables);
  ```


  - `HttpEntity`方式

    如果想获取更详细的数据（比如响应头和http状态码等信息）就使用getForEntity()

  ```java
  ResponseEntity<CommonResponse> responseEntity = restTemplate.getForEntity(url, CommonResponse.class, pathVariables);
  CommonResponse responseBody = responseEntity.getBody();
  HttpStatus status = responseEntity.getStatusCode();
  HttpHeaders headers = responseEntity.getHeaders();
  ```

  ​	跟Object方式类似，不过返回的结果数据封装到了一个ResponseEntity对象中

  - 配置`queryParameters`和`pathVariables`

    RestTemplate中没有为queryParameters设置对应的传参，需要用户自己将queryParameters写到url里面，不过RestTemplate为restful风格的pathVariables配置了专用的传参：

    （看得出来RestTemplate专业服务于restful API调用）

    ```java
    url = "http://127.0.0.1:8081/persons/{id}"
    Map<String, Object> urlVariables = new HashMap<>();
    pathVariables.put("id", "0");
    CommonResponse response = restTemplate.getForObject(url, CommonResponse.class, pathVariables);
    ```

  - `UriTemplateHandler`（高级用法）

    虽然RestTemplate中没有为queryParameters设置对应的传参，但是用户可以自己实现一个UriTemplateHandler：

    ```java
    public class QueryParamsUrlTemplateHandler extends DefaultUriTemplateHandler {
    
    @Override
    public URI expand(String uriTemplate, Map<String, ?> params) {
    		UriComponentsBuilder componentsBuilder = UriComponentsBuilder.fromHttpUrl(uriTemplate);
    		for(Map.Entry<String, ?> varEntry : params.entrySet()){
    			componentsBuilder.queryParam(varEntry.getKey(), varEntry.getValue());
    		}
    		uriTemplate = componentsBuilder.build().toUriString();
    		return super.expand(uriTemplate, params);
    	}
    }
    ```

    ```java
    restTemplate.setUriTemplateHandler(urlTemplateHandler);
    ```

    通过配置该UriTemplateHandler，就可以以Map的形式配置queryParameters了：

    ```java
    Map<String, Object> params = new HashMap<>();
    params.put("name", "张三");
    ResponseEntity<CommonResponse> responseEntity = restTemplate.getForEntity(url, CommonResponse.class, params);
    ```

- post方法

  发起post请求跟get很类似，唯一不同的地方在于需要设置`RequestBody`参数：

  - `Object`方式：

    无需自己设置contentType和charset，只需要直接传递实体对象作为RequestBody，RestTemplate会进行自动判断并选择合适的MeesageConverter

  ```java
  CommonResponse response = restTemplate.postForObject(url, person, CommonResponse.class);
  ```

  使用实体类Person封装上传数据，也可以直接使用Map封装数据

  - `HttpEntity`方式

    跟Object方式类似，request数据可以直接传递实体，也可以将请求头和请求体封装到一个HttpEntity对象中进行发送，返回数据都是ResponseEntity对象，可以取HttpStatus和HttpHeaders

    ```java
    //设置请求头
    HttpHeaders headers = new HttpHeaders();
    MediaType type = MediaType.parseMediaType("application/json; charset=UTF-8");
    headers.setContentType(type);
    headers.set("headerName", "headerValue");
    //设置HttpEntity
    HttpEntity<List<Person>> request = new HttpEntity<>(personList, headers);
    //返回ResponseEntity对象，对Object实体数据进行封装
    ResponseEntity<CommonResponse> response = restTemplate.postForEntity(url, request, CommonResponse.class);
    ```

  - post方法上传文件

    上传文件需要用`MultiValueMap`进行文件数据的封装：

  ```java
  MultipartBodyBuilder builder = new MultipartBodyBuilder();
  File file = new File("D:\\xxx\\xxx.png");
  builder.part("file", new FileSystemResource(file));
  MultiValueMap<String, Object> request = builder.build();
  //上传文件
  CommonResponse response = restTemplate.postForObject(url, request, CommonResponse.class)
  ```

- put和delete实现方式和上述方法都类似，不过这两种请求没有返回数据，不太实用

  - 在RestTemplate中，对GET, POST, PUT, DELETE, OPTIONS, HEAD 这几种http方法都有相应的封装，如果不想用它封装好的方法，可以选择exchange()方法自定义http请求

- exchange方法：

  可以自己指定请求头和http method，其他的细节跟前面几种方法差不多

  对于某些不太常见的方法（比如HEAD或者TRACE），就需要使用exchange()方法了， exchange()方法也有多种重载

  以发起POST请求为例：

  ```java
  //设置requestBody数据
  Person person = new Person();
  //设置queryParameters和pathVariables
  Map<String, Object> params = new HashMap<>();
  params.put("name", "value");
  //设置请求头
  HttpHeaders headers = new HttpHeaders();
  MediaType type = MediaType.parseMediaType("application/json; charset=UTF-8");
  headers.setContentType(type);
  headers.set("headerName", "headerValue");
  //配置requestEntity
  HttpEntity<MyData> requestEntity = new HttpEntity(person, headers);
  //发起请求
  ResponseEntity<Map> responseEntity = restTemplate.exchange(url, HttpMethod.POST, requestEntity, Map.class, params);
  ```

- execute方法（一般不用）：

  可以定制request和response的序列化和反序列化方式：

  ```java
  public <T> T execute(URI url, HttpMethod method, RequestCallback requestCallback,
  	ResponseExtractor<T> responseExtractor) throws RestClientException {
  	return doExecute(url, method, requestCallback, responseExtractor);
  }
  ```

  - RequestCallback用于封装request信息，并对这部分信息进行解析
  - ResponseExtractor用于对response返回数据进行解析

  用户可以自由定制序列化方式，并以回调函数的形式传入execute()中；RestTemplate默认实现是两个静态内部类，默认选取RestTemplate中已经初始化完成的那部分HttpMessageConverter实现，进行序列化和反序列化操作



## 3. 异常捕捉

RestTemplate内部已经把http请求过程中会出现的各种异常，例如404或者500等异常，都包装为了RestClientException抛出

在RestTemplate中进行异常处理的组件是`ResponseErrorHandler`，默认是`DefaultResponseErrorHandler`

用户可以自己实现ResponseErrorHandler来处理http异常：

```java
public class MyselfResponseErrorHandler extends DefaultResponseErrorHandler {

	private final Logger logger = LoggerFactory.getLogger(MyselfResponseErrorHandler.class);

	@Override
	public void handleError(ClientHttpResponse response) throws IOException {
		HttpStatus statusCode = getHttpStatusCode(response);
		String code = statusCode.toString();
		String msg = statusCode.getReasonPhrase();
		switch (statusCode.series()) {
			case CLIENT_ERROR:
				logger.error("客户端请求错误，错误码：" + code + ", 错误信息：" + msg);
				break;
			case SERVER_ERROR:
				logger.error("服务器端错误，错误码：" + code + ", 错误信息：" + msg);
				break;
			default:
				logger.error("不知道什么错误，错误码：" + code + ", 错误信息：" + msg);
		}
	}
}
```

```java
restTemplate.setErrorHandler(responseErrorHandler);
```



# RestTemplate内部源码分析

## doExecute方法


以上所有方法都有一个最终方法，也是RestTemplate的核心方法：doExecute()

```java
	protected <T> T doExecute(URI url, HttpMethod method, RequestCallback requestCallback,
			ResponseExtractor<T> responseExtractor) throws RestClientException {
		//url和method都不能为空
		Assert.notNull(url, "'url' must not be null");
		Assert.notNull(method, "'method' must not be null");
		ClientHttpResponse response = null;
		try {
            //使用ClientHttpRequestFactory创建一个ClientHttpRequest对象
			ClientHttpRequest request = createRequest(url, method);
			if (requestCallback != null) {
                //调用RequestCallback对象对requestBody数据进行解析
                //序列化后的数据封装到ClientHttpRequest对象中
				requestCallback.doWithRequest(request);
			}
            //ClientHttpRequest对象执行http请求得到ClientHttpResponse对象
			response = request.execute();
			if (!getErrorHandler().hasError(response)) {
				logResponseStatus(method, url, response);
			}
			else {
				handleResponseError(method, url, response);
			}
			if (responseExtractor != null) {
                //将ClientHttpResponse对象反序列化为用户指定的数据类型
				return responseExtractor.extractData(response);
			}
			else {
				return null;
			}
		}
		catch (IOException ex) {
			throw new ResourceAccessException("I/O error on " + method.name() +
					" request for \"" + url + "\": " + ex.getMessage(), ex);
		}
		finally {
			if (response != null) {
				response.close();
			}
		}
	}
```

可以看出，实际执行http请求的是ClientHttpRequest，用户可以通过设置不同的ClientHttpRequestFactory自己定制http连接方式，例如HttpComponentsClientHttpRequestFactory就是基于Apache HttpClient实现：

```java
//需要将HttpComponentsClientHttpRequestFactory暴露为spring bean，因为其实现了DisposableBean，可以在bean销毁后自动关闭连接池
@Bean
public HttpComponentsClientHttpRequestFactory getFactory(){
	HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
    //设置socket请求连接超时时间
	factory.setConnectTimeout(5000);
    //设置socket读取数据阻塞超时时间
	factory.setReadTimeout(5000);
	return factory;
}
```



## MessageConverter

- 当调用不同的RestTemplate方法传输数据时，RestTemplate会自动检查ContentType并采用合适的MessageConverter进行序列化和反序列化，如果找不到合适的MessageConverter，将会报错

- RestTemplate里面默认的几种MessageConverter已经能够满足大多数应用场景了

| MessageConverter                    | 描述                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| StringHttpMessageConverter          | 支持文本类型的数据格式：text/plain，text/*                    |
| FormHttpMessageConverter            | 支持表单类型的数据格式：application/x-www-form-urlencoded    |
| MappingJackson2HttpMessageConverter | 支持json类型的数据格式：application/json                     |
| ResourceHttpMessageConverter        | 可用于对Resource类型的文件io流进行序列化，并支持任意的MediaType |
| BufferedImageHttpMessageConverter | 用于读取java.awt.image.BufferedImage格式的图片文件 |

- 用户可以自己定义满足自己业务需求的MessageConverter



# WebClient

spring 5.0全面引入了reactive响应式编程模式，同时也就有了RestTemplate的reactive版：WebClient