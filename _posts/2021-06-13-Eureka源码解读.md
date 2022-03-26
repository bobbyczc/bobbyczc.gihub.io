---
layout: post
title: 'Eureka源码解读'
subtitle: '注册中心Eureka揭秘'
date: 2021-06-13
categories: eureka
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'
tags: eureka springcloud
---



# Eureka源码解读

## 1.Eureka Client

### 1.1 启动过程

我们看一下Eureka Client的代码：

```java 
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class,args);
    }
}
```

引入eureka相关依赖之后，只需要加入一个注解 `@EnableDiscoveryClient` 就可以启动一个eureka客户端，那么我们从这个注解开始探究。

#### 1.1.1 `@EnableDiscoveryClient` 做了什么

下面是`EnableDiscoveryClient` 注解的源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

	/**
	 * If true, the ServiceRegistry will automatically register the local server.
	 */
	boolean autoRegister() default true;
}
```

我们重点关注两个地方

- `@Import(EnableDiscoveryClientImportSelector.class)` :导入了`EnableDiscoveryClientImportSelector` 类
- `boolean autoRegister() default true;` ：这个注解有一个属性为 autoRegister表示是否自动注册到注册中心，默认为 `true`

`@Import` 可以将指定的类添加到spring的ioc容器，或者将实现了`ImportSelector` 接口的类中`selectImports()` 方法返回的String数组（全类名）所对应的的类添加到spring的ioc容器中。

接下来就看一下 `EnableDiscoveryClientImportSelector` 类做了什么：

```java
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableDiscoveryClientImportSelector
		extends SpringFactoryImportSelector<EnableDiscoveryClient> {

	@Override
	public String[] selectImports(AnnotationMetadata metadata) {
      //调用父类的方法，根据注解的元信息获取需要import的类
		String[] imports = super.selectImports(metadata);

		AnnotationAttributes attributes = AnnotationAttributes.fromMap(
				metadata.getAnnotationAttributes(getAnnotationClass().getName(), true));

		boolean autoRegister = attributes.getBoolean("autoRegister");

      //如果autoRegister为true，imports中加入一个AutoServiceRegistrationConfiguration配置类
		if (autoRegister) {
			List<String> importsList = new ArrayList<>(Arrays.asList(imports));
			importsList.add("org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
			imports = importsList.toArray(new String[0]);
		} else {
          //如果autoRegister为false，则更新环境变量
			Environment env = getEnvironment();
			if(ConfigurableEnvironment.class.isInstance(env)) {
				ConfigurableEnvironment configEnv = (ConfigurableEnvironment)env;
				LinkedHashMap<String, Object> map = new LinkedHashMap<>();
				map.put("spring.cloud.service-registry.auto-registration.enabled", false);
				MapPropertySource propertySource = new MapPropertySource(
						"springCloudDiscoveryClient", map);
				configEnv.getPropertySources().addLast(propertySource);
			}

		}

		return imports;
	}

	@Override
	protected boolean isEnabled() {
		return new RelaxedPropertyResolver(getEnvironment()).getProperty(
				"spring.cloud.discovery.enabled", Boolean.class, Boolean.TRUE);
	}

	@Override
	protected boolean hasDefaultFactory() {
		return true;
	}

}
```

我们知道默认情况下`EnableDiscoveryClient` 中的autoRegister属性是true，因此，默认情况下会导入`AutoServiceRegistrationConfiguration` 类，除此之外，还调用了父类的 `selectImports()` 方法，将此方法返回的类也一并添加到spring的ioc容器中。

那么父类 `SpringFactoryImportSelector<EnableDiscoveryClient>` 又添加了哪些类到ioc容器呢，看一下源码：

```java
public abstract class SpringFactoryImportSelector<T>
		implements DeferredImportSelector, BeanClassLoaderAware, EnvironmentAware {
	private ClassLoader beanClassLoader;
	private Class<T> annotationClass;
	private Environment environment;
	private final Log log = LogFactory.getLog(SpringFactoryImportSelector.class);

	@SuppressWarnings("unchecked")
	protected SpringFactoryImportSelector() {
		this.annotationClass = (Class<T>) GenericTypeResolver
				.resolveTypeArgument(this.getClass(), SpringFactoryImportSelector.class);
	}

	@Override
	public String[] selectImports(AnnotationMetadata metadata) {
		if (!isEnabled()) {
			return new String[0];
		}
      //获取当前注解的属性
		AnnotationAttributes attributes = AnnotationAttributes.fromMap(
				metadata.getAnnotationAttributes(this.annotationClass.getName(), true));

		Assert.notNull(attributes, "No " + getSimpleName() + " attributes found. Is "
				+ metadata.getClassName() + " annotated with @" + getSimpleName() + "?");
		//找到所有的自动配置类，根据当前注解
		// Find all possible auto configuration classes, filtering duplicates
		List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader
				.loadFactoryNames(this.annotationClass, this.beanClassLoader)));

		if (factories.isEmpty() && !hasDefaultFactory()) {
			throw new IllegalStateException("Annotation @" + getSimpleName()
					+ " found, but there are no implementations. Did you forget to include a starter?");
		}

		if (factories.size() > 1) {
			// there should only ever be one DiscoveryClient, but there might be more than
			// one factory
			log.warn("More than one implementation " + "of @" + getSimpleName()
					+ " (now relying on @Conditionals to pick one): " + factories);
		}

		return factories.toArray(new String[factories.size()]);
	}

	protected boolean hasDefaultFactory() {
		return false;
	}
	protected abstract boolean isEnabled();
  
	protected String getSimpleName() {
		return this.annotationClass.getSimpleName();
	}
	................
}
```

其中最重要的部分是，`loadFactoryNames()` 方法根据当前的注解类型，获取所有的自动配置类，我们看看这个方法：

```java
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
   String factoryClassName = factoryClass.getName();
   try {
      Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
      List<String> result = new ArrayList<String>();
      while (urls.hasMoreElements()) {
         URL url = urls.nextElement();
         Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
         String factoryClassNames = properties.getProperty(factoryClassName);
         result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
      }
      return result;
   }
   catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
            "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
   }
}
```

这个方法在 `FACTORIES_RESOURCE_LOCATION` 路径中寻找我们所需要的内容，继续跟进，我们可以发现下面的声明：

```java
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
```

我们去找一下这个`spring.factories` 文件，到底是什么内容呢？

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration

org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration

org.springframework.cloud.client.discovery.EnableDiscoveryClient=\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration
```

这样一切都明白了，`@EnableDiscoveryClient` 这个注解，从类路径下的 `META-INFO/spring.factories` 文件中加载该注解对应的自动配置类，并导入spring容器中；不要忘了前面提到还导入了`AutoServiceRegistrationConfiguration` 这个类到spring的容器里。

同理我们可以跟到 `@SpringBootApplication` 注解中，我们可以发现 `@EnableAutoConfiguration` 注解，此注解导入了 `EnableAutoConfigurationImportSelector` 类，而这个类继承自 `AutoConfigurationImportSelector` , 最终 `AutoConfigurationImportSelector` 类中实现了从 `spring.factories` 文件中获取 `EnableAutoConfiguration` 对应的自动配置类，并导入spring容器

##### 总结

Eureka客户端启动时，加了两个注解，`@SpringBootApplication` 和`@EnableDiscoveryClient` ，这两个注解最终实现了将classpath下面的 `META-INFO/spring.factories` 文件内定义的这两个注解对应的配置类导入到spring容器中。

> - `EurekaDiscoveryClientConfiguration`
> - `EurekaClientConfigServerAutoConfiguration`
> - `EurekaDiscoveryClientConfigServiceAutoConfiguration`
> - `EurekaClientAutoConfiguration`
> - `RibbonEurekaAutoConfiguration`



  