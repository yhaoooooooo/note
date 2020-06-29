---
typora-copy-images-to: img
typora-root-url: img
---

# Zuul

## 源码分析

**ZuulServlet**

zuul创建一个servlet，用于接受所有的请求。

```java
@Override
    public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();
            try {
                //路由之前，
                preRoute();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                //路由
                route();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                //请求路由
                postRoute();
            } catch (ZuulException e) {
                error(e);
                return;
            }

        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }
```

三个route方法 preroute（），route（），postRoute（），分别会执行三种类型的filter，其中在postRoute（）方法中，执行了一个LocationRewriteFilter，引用RouteLocator，**可以通过重写RouteLocator**，实现自己的locator方法， 可从数据库中获取。

```java

public class DynamicRouteLocator extends DiscoveryClientRouteLocator {
	
    private ZuulProperties properties;
    private EurekaClientConfig clientCfg;
    private RestTemplate restTemplate;

    public DynamicRouteLocator(String servletPath, DiscoveryClient discovery, ZuulProperties properties, EurekaClientConfig clientConfig, RestTemplate restTemplate) {
        super(servletPath, discovery, properties);
        this.properties = properties;
        this.clientCfg = clientConfig;
        this.restTemplate = restTemplate;
    }

    /**
     * 重写路由配置
     * <p>
     * 1. properties 配置。
     * 2. eureka 默认配置。
     * 3. DB数据库配置。
     *
     * @return 路由表
     */
    @Override
    protected LinkedHashMap<String, ZuulProperties.ZuulRoute> locateRoutes() {
        LinkedHashMap<String, ZuulProperties.ZuulRoute> routesMap = new LinkedHashMap<>();
        //读取properties配置、eureka默认配置
        routesMap.putAll(super.locateRoutes());
        routesMap.putAll(locateRoutesFromDb());
        LinkedHashMap<String, ZuulProperties.ZuulRoute> values = new LinkedHashMap<>();
        for (Map.Entry<String, ZuulProperties.ZuulRoute> entry : routesMap.entrySet()) {
            String path = entry.getKey();
            if (!path.startsWith("/")) {
                path = "/" + path;
            }
            if (StrUtil.isNotBlank(this.properties.getPrefix())) {
                path = this.properties.getPrefix() + path;
                if (!path.startsWith("/")) {
                    path = "/" + path;
                }
            }
            values.put(path, entry.getValue());
        }
        return values;
    }

    /**
     * 从数据库拉取
     *
     * @return
     */
	private Map<String, ZuulProperties.ZuulRoute> locateRoutesFromDb() {
		
        Map<String, ZuulProperties.ZuulRoute> routes = new LinkedHashMap<>();
        List<SysZuulRoute> results = null;
        //调用的是远程接口，获取到的从数据库中封装的对象。
        String serverurl = clientCfg.getEurekaServerServiceUrls("defaultZone").get(0);
        if(StringUtils.isNotEmpty(serverurl)) {
        	String[] svrurls = serverurl.split("@");
        	String requrl = "http://" + svrurls[1].replaceAll("/eureka/", "/rest/zuul/routerInfo");
        	SysZuulRoute[] zuuls = restTemplate.getForObject(requrl, SysZuulRoute[].class);
        	results = Arrays.asList(zuuls);
        }
        if (results == null) {
            return routes;
        }

        for (SysZuulRoute result : results) {
            if (StrUtil.isBlank(result.getPath()) && StrUtil.isBlank(result.getUrl())) {
                continue;
            }

            ZuulProperties.ZuulRoute zuulRoute = new ZuulProperties.ZuulRoute();
            try {
                zuulRoute.setId(result.getServiceId());
                zuulRoute.setPath(result.getPath());
                zuulRoute.setServiceId(result.getServiceId());
                zuulRoute.setRetryable(StrUtil.equals(result.getRetryable(), "0") ? Boolean.FALSE : Boolean.TRUE);
                zuulRoute.setStripPrefix(StrUtil.equals(result.getStripPrefix(), "0") ? Boolean.FALSE : Boolean.TRUE);
                zuulRoute.setUrl(result.getUrl());
                List<String> sensitiveHeadersList = StrUtil.splitTrim(result.getSensitiveheadersList(), ",");
                if (sensitiveHeadersList != null) {
                    Set<String> sensitiveHeaderSet = CollUtil.newHashSet();
                    sensitiveHeadersList.forEach(sensitiveHeader -> sensitiveHeaderSet.add(sensitiveHeader));
                    zuulRoute.setSensitiveHeaders(sensitiveHeaderSet);
                    zuulRoute.setCustomSensitiveHeaders(true);
                }
            } catch (Exception e) {
                log.error("从数据库加载路由配置异常", e);
            }
            log.debug("添加数据库自定义的路由配置,path：{}，serviceId:{}", zuulRoute.getPath(), zuulRoute.getServiceId());
            routes.put(zuulRoute.getPath(), zuulRoute);
        }
        return routes;
    }
}

```

## Zuul整合Oauth2 JWT

**在已经获取到token以后**

1. 访问， 请求AccessFilter， 对token进行校验。
2. 经过ResourceServerConfiguration配置的`registry.anyRequest().access("@permissionService.hasPermission(request,authentication)");`,在permissionservice.haspermission中对用户的权限进行校验，从平台服务中根据角色获取到菜单权限。
3. 如果权限存在，则请求对应服务。

**获取token环节**

1. 请求/auth/token请求。带有Authetication头。
2. zuul请求忽略，直接转发到认证中心。
3. 认证中心。配置`WebSecurityConfigurerAdapter` 忽略部分请求。 配置`AuthorizationServerConfigurerAdapter` ，a。获取第三方客户端信息的client类型（Jdbc），包含客户密码加密类型，TokenEnhancer（token中可包含其他的自定义信息username,userid,tenantCode,organid,organname）。b。配置Token类型和UserDetailservice。

# Feign

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients{
}
```

## 收获

### **通过spring scanner扫描本地包**

FeignClientsRegistrar.registerBeanDefinitions方法内，获取到spring scanner，然后通过注解的属性内的包路径，进行扫描。**重点关注**    **FeignClientFactoryBean**

```java
BeanDefinitionBuilder definition = BeanDefinitionBuilder
      .genericBeanDefinition(FeignClientFactoryBean.class);
```

```java
class FeignClientFactoryBean
		implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
	getObject(){
		
	}		
	<T> T getTarget() {
		FeignContext context = this.applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
			this.url += cleanPath();
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient) client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
		return (T) targeter.target(this, builder, context,
				new HardCodedTarget<>(this.type, this.name, url));
	}
}
```





![Feign源码分析](/Feign源码分析.png)

在loadbalance方法内，是用jdk的动态代理，创建了一个新的代理对象，执行的**handler**为new ReflectiveFeign.FeignInvocationHandler(target, dispatch)；

# Ribbon

```java
@LoadBalance //@loadbalance里边是qulifier注解
@bean
public RestTemplate(){
	return new RestTemplate();
}
```



```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

    //这边会将上方带有loadbalnace注解的resttemplate注入进去。
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

    /**
    * 包含一个拦截器，会resttemplate.setInterceptor(loadBalancerInterceptor)
    **/
	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);
				}
			}
		});
	}
    
    
    /**
    * resttemplatecustomizer,相当于向resttemplate的拦截器中添加了一个loadbanalcerinterceptor
    **/
   		 @Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(
						restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
				restTemplate.setInterceptors(list);
			};
		}
}
```

Ribbon的作用是给RestTemplate对象增加要给请求拦截器，请求直接被Ribbon的拦截器拦截，进行ribbon的请求。



请求过程中，ribbon需要调用负载均衡器 **loadbalancer（zonexxxxloadbalancer)**

@param serverList  会被注册中心的包重写，比如eureka，会重新定义serverlist对象，会从eureka中获取server列表，如果是nacos，应该也会这么找做。

```java
class RibbonClientConfiguration{

    /**
    *@param serverList  会被注册中心的包重写，比如eureka，会重新定义serverlist对象，会从eureka中获取server列表，如果是nacos，应该也会这么找做。
    **/
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}
}
```

![Ribbon源码分析](/Ribbon源码分析.png)

### Ribbon实现自定义负载均衡

