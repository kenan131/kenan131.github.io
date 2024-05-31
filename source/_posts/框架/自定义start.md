---
title: 网关项目网络请求工具
date: 2024-04-25 15:17:24
tags: 项目
categories: 项目
---

### 介绍

在微服务部署的架构中，通常网络请求都是先通过请求网关，再由网关请求具体的服务，服务返回结果给网关，再由网关返回给客户端。

在我的网关项目中 ，为了便于客户端发送网络请求，不用每个接口每个地址都用http工具构建请求，故此模仿feign框架的请求方式，做了一个简易版的请求工具。该工具通过构建spingboot的自定义start实现，即使用方只需要引入对应的maven坐标，即可自动注入相关的bean到项目的spring容器中。

SpringBoot starter介绍

- SpringBoot中的starter是一种非常重要的机制，能够抛弃以前繁杂的配置，将其统一集成进starter，应用者只需要在maven中引入starter依赖，SpringBoot就能自动扫描到要加载的信息并启动相应的默认配置。starter让我们摆脱了各种依赖库的处理，需要配置各种信息的困扰。SpringBoot会自动通过classpath路径下的类发现需要的Bean，并注册进IOC容器。SpringBoot提供了针对日常企业应用研发各种场景的spring-boot-starter依赖模块。所有这些依赖模块都遵循着约定成俗的默认配置，并允许我们调整这些配置，即遵循“约定大于配置”的理念。
- 简而言之，starter就是一个外部的项目，我们需要使用它的时候就可以在当前springboot项目中引入它。

### 需求

通过@CoolMethod注解去定义接口请求路径和请求方法

```java
public interface BService {

    @CoolMethod(path = "/serviceB/aaa",method = "get")
    public UserDto getAAA(String name, int age);

    @CoolMethod(path = "/serviceB/bbb",method = "post")
    public UserDto postBBB(UserDto dto);

}
```

然后通过@CoolReference注入需要使用的类中

```java
@CoolReference
private BService bService;

public void test(){
    UserDto dto = bService.getAAA("入参A", 88); // 即可调用bservice的接口。
}
```

### 实现

通常在spring中定义一个bean的方法一下几种：

- 通过@Component注解定义
- 通过@Configuration注解的类中，搭配@Bean注解配置bean。
- 通过@Import注解，将类导入到spring容器中。如@EnableTransactionManagement相关的注解，就是使用该方式导入bean。

而springboot自定义的start通常都是根据@Configuration注解的类中，搭配@Bean注解来将bean注入容器的。

1、先定义需要注入容器中的bean

```java
@Slf4j
public class BeanScannerFactory implements InstantiationAwareBeanPostProcessor {
    // 网关域名地址
    private String domain;

    private ServiceFactory serviceFactory;

    public BeanScannerFactory(String domain) {
        this.domain = domain;
        serviceFactory = new ServiceFactory(domain);
    }

    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        ReflectionUtils.doWithFields(bean.getClass(), new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                // 根据注解CoolReference 去过滤。只获取类中成员字段上加了@CoolReference注解的bean
                if(field.isAnnotationPresent(CoolReference.class)){
                    Class iface = field.getType();
                    if(!iface.isInterface()) {
                        log.error("不允许将CoolReference注解加在非接口上");
                        throw new RuntimeException("不允许将CoolReference注解加在非接口上");
                    }
                    // 根据成员字段的类型 生成代理对象。
                    Object proxy = serviceFactory.getProxy(iface);
                    field.setAccessible(true);//允许反射设置值
                    field.set(bean,proxy);//将代理对象set到该bean的成员字段中
                }
            }
        });
        return true;
    }
}
```

2、定义配置类，该类也可以通过@Value注解获取配置类的信息，传入给定义的bean

```java
@Configuration
public class AutoConfiguration {

    @Value("${cool.gateway.domain}")
    public String gatewayDomain;

    @Bean
    @ConditionalOnClass(BeanScannerFactory.class)
    public BeanScannerFactory jwtSignerHolder(){
        return new BeanScannerFactory(gatewayDomain);
    }
}
```

3、在resource目录下新建META-INF文件夹，在该文件夹下新建spring.factories，配置下列信息。

```tex
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.bin.coolgatewaycore.config.AutoConfiguration
```

4、在maven工具中，执行 mvn clean package命令，将模块打包到本地仓库，在另外一个项目模块中引入，即可实现springbootstart。

补充：生成代理对象逻辑代码

```java
@Slf4j
public class ServiceFactory {
    
    static private RequestClient requestClient = new RequestClient();
    
    private String domain;

    public ServiceFactory(String domain) {
        this.domain = domain;
    }

    public Object getProxy(Class iface){
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class[]{iface}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Class<?> returnType = method.getReturnType();
                String className = method.getDeclaringClass().getName();
                // 过滤 object 默认的方法。hashCode，toString，equals
                if (className.equals(Object.class.getName())) {
                    throw new RuntimeException("暂不支持Object 原生方法。");
                }
                GatewayResponseRes responseRes ;
                CoolMethod annotation = method.getAnnotation(CoolMethod.class);
                String path = annotation.path();
                String url = domain + path;
                Parameter[] parameters = method.getParameters();
                ArrayList<String> parameterName = new ArrayList<>();
                for(Parameter parameter : parameters){
                    parameterName.add(parameter.getName());
                }
                if(annotation.method().equals("get")){
                    responseRes = requestClient.sendGetRequest(url, parameterName, args);
                }
                else{
                    responseRes = requestClient.sendPostRequest(url, parameterName, args);
                }
                Object res;
                if(responseRes.getGatewayCode() == 10001){
                    res = JSON.parseObject(responseRes.getData().toString(), returnType);
                    if(res == null){
                        log.error("json 转换 对象失败。");
                        throw new RuntimeException("json 转换 对象失败。");
                    }
                }
                else{
                    throw new RuntimeException("下游服务，请求失败" + responseRes.getMessage());
                }
                return res;
            }
        });
    }
}
```

构建请求工具类

```java
@Slf4j
public class RequestClient {
    static private AsyncHttpClient asyncHttpClient;

    static {
        DefaultAsyncHttpClientConfig.Builder builder = new DefaultAsyncHttpClientConfig.Builder()
                .setEventLoopGroup(new NioEventLoopGroup())
                .setAllocator(PooledByteBufAllocator.DEFAULT) // 使用池化的ByteBuf分配器以提升性能
                .setCompressionEnforced(true);// 强制压缩
        asyncHttpClient = new DefaultAsyncHttpClient(builder.build());
    }


    public GatewayResponseRes sendGetRequest(String url, List parameterName, Object[] parameterArgs){
        String pathParameter = getParameterPath(parameterName,parameterArgs);
        String finUrl = url +  pathParameter;
        try{
            Response response = asyncHttpClient.prepareGet(finUrl)
                    .setReadTimeout(3000)
                    .execute().get();
            String responseBody = response.getResponseBody(StandardCharsets.UTF_8);
            GatewayResponseRes responseRes = JSON.parseObject(responseBody, GatewayResponseRes.class);
            return responseRes;
        }catch (Exception e){
            log.error("get 请求发送失败。请求路径："+finUrl);
            throw new RuntimeException(e.getMessage());
        }
    }

    //构造get请求，路径中的参数。
    private String getParameterPath(List parameterName, Object[] parameterArgs) {
        //参数为空，参数和名称不相等都不设置参数。
        if(parameterName.size() == 0 || parameterArgs == null || parameterName.size() != parameterArgs.length){
            return "";
        }
        StringBuilder stringBuilder = new StringBuilder("?");
        for(int i=0;i<parameterName.size();i++){
            if(i!=0){
                stringBuilder.append("&");
            }
            stringBuilder.append(parameterName.get(i)+"="+parameterArgs[i].toString());
        }
        return stringBuilder.toString();
    }

    public GatewayResponseRes sendPostRequest(String finUrl, List<String> parameterNames, Object[] args){
        BoundRequestBuilder requestBuilder = asyncHttpClient.preparePost(finUrl)
                .setRequestTimeout(3000);//默认超时3s
        try{
            if(parameterNames.size() > 1){
                //如果参数大于1 则通过form表单传输
                for(int i =0;i<parameterNames.size();i++){
                    requestBuilder.addFormParam(parameterNames.get(i),args[i].toString());
                }
            }else{
                //否则，通过json串进行传输。默认取第一个参数
                if(args != null && args[0] != null){
                    Object arg = args[0];
                    JSONObject jsonObject = JSON.parseObject(JSON.toJSONString(arg));
                    requestBuilder.setBody(jsonObject.toJSONString());
                    requestBuilder.setHeader("Content-Type","application/json");
                }
            }
            Response response = requestBuilder.execute().get();
            String responseBody = response.getResponseBody(StandardCharsets.UTF_8);
            GatewayResponseRes responseRes = JSON.parseObject(responseBody, GatewayResponseRes.class);
            return responseRes;
        }catch (Exception e) {
            log.error("post请求异常 请求路径：" + finUrl);
            throw new RuntimeException(e.getMessage());
        }
    }

}
```