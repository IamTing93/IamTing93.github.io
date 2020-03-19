# Spring Security Architecture

### Authentication和Access Control

应用安全或多或少归结为两个独立的问题：认证（你是谁？）和授权（你能干啥？）。Spring Security就是针对这两个问题设计出来的框架，而且具有良好的拓展性。

##### Authentication

认证的主要接口是`AuthenticationManager`，这个接口只有一个方法：

```java
public interface AuthenticationManager {

  Authentication authenticate(Authentication authentication)
    throws AuthenticationException;

}
```

`AuthenticationManager`在`authenticate()`中做了以下3件事中的一件：

1. 如果输入的验证信息正确，那么返回一个`Authentication`对象（通常设置其属性`authenticated=true`）
2. 若果验证信息不正确，就直接抛出`AuthenticationException`异常
3. 若果不能判断信息正确与否就返回`null`

`AuthenticationException`是一个运行时异常。它通常由应用默认的一个通用方法处理掉，这取决于应用的风格和目的。换句话说就是开发者一般不应该在代码里直接捕获和处理它，直接抛出。

最常见的`AuthenticationManager`实现是`ProviderManager`，这个实现类利用一系列的`AuthenticationProvider`来进行认证。`AuthenticationProvider` 接口有点像`AuthenticationManager`，但是它有一个额外的方法可供调用，这个方法用来检查给定的认证方式是否支持：

```java
public interface AuthenticationProvider {

	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;

	boolean supports(Class<?> authentication);

}
```

`supports()`中的`Class<?>`参数实际上是`Class<? extends Authentication>`。通过委托一系列的`AuthenticationProviders`，应用里的一个`ProviderManager`可以支持多种不同的认证方式。若果`ProviderManager`不能认证某个特殊的`Authentication`实例，那么它就会被跳过。

`ProviderManager`有一个可选的父类，若果子类的所有providers都返回`null`，那么就到父类中进行验证。若果没有父类，并且认证失败的话将会抛出`AuthenticationException`。

有时候，一个应用会有关于某种资源的集合（譬如所有匹配`/api/**`URI的页面资源），每种集合都会有属于它自己的`AuthenticationManager`。有时候，这些集合的`AuthenticationManager`的实现类是一个`ProviderManager`，并且它们共享同一个父类。这个父类相当于一种“全局”资源，用作所有子类providers都认证失败时的一种替补验证方案。

![ProviderManagers with a common parent](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/authentication.png)

##### Customizing Authentication Managers

Spring Security提供一些配置helper来快速配置认证manager。最常见的helper就是`AuthenticationManagerBuilder`，适合用来创建in-memory，JDBC或者LDAP的`UserDetails`，或者是添加自定义的`UserDetailsService`。下面的例子展示了怎么配置一个全局的`AuthenticationManager`，即是上面提及的父类manager：

```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

   ... // web stuff here

  @Autowired
  public void initialize(AuthenticationManagerBuilder builder, DataSource dataSource) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }

}
```

上面是以web应用作为例子，但是`AuthenticationManagerBuilder`的应用场景不止如此。注意，这里的`AuthenticationManagerBuilder`是被用`@Autowired`注入到`@Bean`的方法中，这样可以让builder创建一个全局的父类`AuthenticationManager`。相反，若果我们用以下方式创建：

```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

  @Autowired
  DataSource dataSource;

   ... // web stuff here

  @Override
  public void configure(AuthenticationManagerBuilder builder) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }

}
```

(通过`@Override`父类的方法)那么`AuthenticationManagerBuilder`只会创建一个局部的`AuthenticationManager`，它是全局`AuthenticationManager`的一个子类。在Spring boot应用中可以`@Autowired`全局`AuthenticationManager`到其它bean中，但是不能够`@Autowird`一个局部的，除非把它暴露出来。

Spring Boot提供一个默认的全局 `AuthenticationManager`  ,除非自定义一个全局的`AuthenticationManager`Bean来替换它。一般情况下，默认的全局`AuthenticationManager`足够安全，无须过于担心其安全性，除非自定义一个全局`AuthenticationManager`。

##### Authorization or Access Control

一旦认证成功，我们就能进行下一步：授权。授权的核心是`AccessDecisionManager`。框架提供了三种实现类，这三种实现类都是委托一系列的`AccessDecisionVoter`来确定是不是要授权，有点像`ProviderManager`委托`AuthenticationProviders`。

一个`AccessDecisionVoter`需要一个`Authentication`（表示当前认证了的用户）和一个被`ConfigAttributes`描述的受保护对象来确定是否授权：

```java
boolean supports(ConfigAttribute attribute);

boolean supports(Class<?> clazz);

int vote(Authentication authentication, S object,
        Collection<ConfigAttribute> attributes);
```

`Object`完全可以在`AccessDecisionManager`和`AccessDecisionVoter`之间进行传递。它代表一个用户想要访问的东西（最常见的是一个web资源或者一个Java类中的方法）。`ConfigAttributes`也可以在`AccessDecisionManager`和`AccessDecisionVoter`之间进行传递，它携带了一些信息，这些信息代表了这个受保护的`Object`的授权规则。`ConfigAttribute`是一个接口，但是它只有一个返回字符串的方法，这些字符串表示了受保护资源的授权规则，并且通常有特殊格式（譬如`ROLE_`前缀）或者是一个需要计算的表达式。

大多开发者都只使用默认的`AccessDecisionManager`，具体实现类是`AffirmativeBased`（这个实现类的策略是只要有一个voter通过，就会被授权）。自定义是否授权，要么就是在voter里新增授权规则，要么就是修改原有的授权规则。

使用Spring Expression Language(SpEL)表达式来确定是否授权很常见，如` isFullyAuthenticated() && hasRole('FOO') `。若果需要自定义SpEL表达式，那么需要实现`SecurityExpressionRoot`  或者 `SecurityExpressionHandler `。

---

### Web Security

Spring Security 是基于Servlet的`Filters`，所以先看看`Filters`会很大有裨益。下图是一次HTTP请求从发起到被处理过程中所经过的handlers的层级图。

![Filter chain delegating to a Servlet](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/filters.png)

客户端发送一个请求到APP，然后container基于请求的URI路径决定哪个过滤器和哪个servlet会处理这次请求。一次请求最多能被一个servlet处理，但是过滤器则是组成一条有序链，请求可以在链中被顺序地处理，并且一个过滤器可以拦截请求，不让后续的过滤器处理。一个过滤器也可以修改传递给下游过滤器和servlet的请求和（或者）响应。过滤器链的顺序非常重要，而且Spring Boot通过2个机制来管理链的顺序：一个是有`@Order`注解或者实现了`Ordered`接口的`Filter`类`@Bean`，另外一种是把过滤器修饰成` FilterRegistrationBean `，这个` FilterRegistrationBean `有一个`order`的成员变量决定调用的顺序。一些现成的过滤器定义了它们自己的值来表明它们想要相对于彼此的顺序位置（例如来自Spring Session的`SessionRepositoryFilter`有一个值为`Integer.MIN_VALUE + 50`的`DEFAULT_ORDER`，这说明它想在链中早点被处理，同时不妨碍比它靠前的过滤器被处理)。

Spring Security本质上就是这条链中的一个`Filter`，具体类型是`FilterChainProxy`，

下文将会提及这个类。在一个Spring Boot app中，这个Spring Security的过滤器是`ApplicationContext`中的一个`@Bean`，而且默认是被添加到链中，所以它可以处理每一次请求。它在链中的位置由` SecurityProperties.DEFAULT_FILTER_ORDER`定义，该位置由` FilterRegistrationBean.REQUEST_WRAPPER_FILTER_MAX_ORDER `锚定。不仅如此，从container的角度看，Spring Security就只是一个过滤器，但是从Spring Security角度看，它自身内部包含了很多额外的过滤器，每一个都有特定的作用。如下图:



实际上，Spring Security的过滤器中甚至还有一层间接层：这个间接层在container中是以`DelegatingFilterProxy `形式存在，并且不是一个Spring `@Bean`。这个`DelegatingFilterProxy `在容器中代表着`FilterChainProxy `，这个`FilterChainProxy `有着一个固定的名字`springSecurityFilterChain`。

`FilterChainProxy`包含了所有的安全逻辑，并且在内部通过过滤器链来实现。链中的过滤器都有相同的API接口（从Servlet角度看它们都实现了`Filter`接口），并且可以拦截请求，不让后续的过滤器处理。

在Spring Security中，顶层的`FilterChainProxy`管理着多条的过滤器链，而且container不知道这些链的存在。Spring Security filter包含了一个过滤器链的列表，并且把请求分派给第一个匹配的过滤器链。下图展示了基于请求路径的分派例子（/foo/* *会比 /**先匹配)。这是最常见的但不是唯一的匹配方式。最重要的一点就是只有一条过滤器链处理一个请求。

![Security Filter Dispatch](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/security-filters-dispatch.png)

没有自定义安全配置的Spring Boot应用有n条过滤器链，一般n=6。第n - 1条链会忽略静态资源匹配，如`/css/** `和`/images/**`，以及错误页面`/error`（用户能通过配置`SecurityProperties`的`security.ignored`来配置这些路径）。最后一条过滤器链匹配全路径`/**`，而且更加灵活，这条链包含了认证，授权，异常处理，会话处理，头写入等等的逻辑。这条链默认一共有11个过滤器，但是用户一般不需要关心用到了哪些过滤器，以及什么时候用到。

```
Spring Security内所有的过滤器对于container来说都是未知的，这一点很重要，特别是在一个Spring Boot应用中，所有的`Filter`类型的`@Beans`都是自动被container注册。所以若果想在安全链中添加一个自定义的过滤器，就要不声明为`@Bean`，或者封装到`FilterRegistrationBean`中并且显式禁止container注册。
```

##### Creating and Customizing Filter Chains

在Spring Boot app中，默认备用的过滤器链（即匹配`/**`请求的链）有一个`SecurityProperties.BASIC_AUTH_ORDER`的预定义顺序。通过设置`security.basic.enabled=false`，或者设置一个lower的顺序就可以关掉这个预定义顺序。要这样做，只需添加一个`WebSecurityConfigurerAdapter`（或者`WebSecurityConfigurer`）的`@Bean`并且用`@Order`修饰这个类。例子如下：

```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
     ...;
  }
}
```

这个bean会导致Spring Security增加一条新的过滤器链并且放在备用链前。

许多应用对于不同的资源，都有不同的访问规则。每一个资源都有它自己的`WebSecurityConfigurerAdapter`，这个adapter有唯一的顺序，并且有自己的请求matcher。若果匹配规则有重叠，顺序靠前的过滤器将优先。

### Request Matching for Dispatch and Authorization

一条安全的过滤器链（等价于一个`WebSecurityCOnfigurerAdapter`）有一个用于请求的匹配器，这个匹配器决定这条过滤器链应用到这个HTTP请求。一旦一条过滤器链被应用，其它的就不会被应用了。但是在过滤器链中你能够通过在`HttpSecurity`中配置额外的匹配器来进行更细粒度的控制：

```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
      .authorizeRequests()
        .antMatchers("/foo/bar").hasRole("BAR")
        .antMatchers("/foo/spam").hasRole("SPAM")
        .anyRequest().isAuthenticated();
  }
}
```

最常见的错误是忘记这些匹配器是应用于不同的处理过程，一个是匹配整个过滤器链（antMatcher("/foo/**")），另外一个只是设置访问规则（antMatchers("/foo/bar").hasRole("BAR")）。

##### Combining Application Security Rules with Actuator Rules

若果使用Spring Boot Actuator来管理端点，那就希望这些端点都是安全的，并且默认就是如此。事实上只要添加Actuator到一个安全应用，就会得到一条额外的只应用到actuator端点的过滤器链。它定义了一个只匹配actuator端点的请求匹配器，并且有着一个值为5的` ManagementServerProperties.BASIC_AUTH_ORDER `，这个顺序要小于`SecurityProperties`的备用链，所以会在备用链前被调用。

若果需要应用安全规则到这些actuator端点，可以添加一个顺序要比默认actuator端点的链更早的，匹配所有端点的过滤器链。若果想用默认的配置，那么最简单的方式是添加一个顺序比默认actuator后，但比备用早（如` ManagementServerProperties.BASIC_AUTH_ORDER + 1 `的过滤器，如

```java
@Configuration
@Order(ManagementServerProperties.BASIC_AUTH_ORDER + 1)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
     ...;
  }
}
```

### Method Security

正如支持安全web应用那样，Spring Security也为安全访问Java方法提供支持。对于Spring Security来说这不过是一种不同“保护资源“。对于用户来说，这意味着访问规则使用相同格式的`ConfigAttribute`字符串（如角色或者表达式），但是编写代码的位置就不一样了。第一步是启动method security，例如在应用的顶层配置：

```java
@SpringBootApplication
@EnableGlobalMethodSecurity(securedEnabled = true)
public class SampleSecureApplication {
}
```

之后就能直接装饰方法资源，如

```java
@Service
public class MyService {

  @Secured("ROLE_USER")
  public String secure() {
    return "Hello Security";
  }

}
```

这个一个有着安全方法的service。若果Spring创建一个这样类型的`@Bean`，那么它会被代理，并且调用者将会被安全拦截器进行权限验证，通过后方法才能执行。若果验证失败，那么调用者将会得到`AccessDeniedException`而不是方法的执行结果。

还有其它用于方法的，可以增强安全性的注解，譬如`@PreAuthorize`和`@PostAuthorize`，分别能让你写一些关于方法参数和返回结果的表达式。

```
组合Web安全和方法安全的例子并不少见。过滤器链为用户提供常见的功能，如认证和重定向到登录页面等等，而方法安全提供更具颗粒度的保护。
```

### Working with Threads

Spring Security从根本上来说是线程绑定的，因为它需要确保当前的认证用户信息可以被下游的消费者获取。基本构件是`SecurityContext`，它可能包含了一个`Authenticaiton`（当一个用户登录后`Authentication`将会被明确地标记为`authenticated`）。通过`SecurityContextHolder`静态方法，你可以总是访问和操作`SecurityContext`

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
assert(authentication.isAuthenticated);
```

在user应用代码中这样做不常见，但是要自定义认证过滤器的话，这会很有用（尽管Spring Security中有基类可以实现相同功能，避免使用`SecurityContextHolder`)。

若果在一个web端点中访问当前认证用户，可以在`@RequestMapping`中传入参数：

```java
@RequestMapping("/foo")
public String foo(@AuthenticationPrincipal User user) {
  ... // do stuff with user
}
```

这个注解从`SecurityCOntext`中拉取当前`Authentication`，并且调用`getPrincipal()`方法得到参数。`Authentication`中的`Principal`的类型依赖于进行认证`AuthenticationManager`，因此这是一个获取安全用户数据引用的小技巧。

若果使用Spring Security，那么来自`HttpServletRequest`的`Principal`会是`Authentication`类型，那么可以直接这样定义：

```java
@RequestMapping("/foo")
public String foo(Principal principal) {
  Authentication authentication = (Authentication) principal;
  User = (User) authentication.getPrincipal();
  ... // do stuff with user
}
```

有时候需要代码在没用Spring Security情况下正常工作，这些样就会很有用（你将需要更加安全地加载`Authentication`类）。

##### Processing Secure Methods Asynchronously

因为`SecurityContext`是线程绑定的，若果想在后台处理中调用安全方法，如带有`@Async`，那么需要确保上下文被传递。这可以归结为用后台任务（`Runnable`，`Callable`等）封装`SecurityContext`。Spring Security提供一些helpers来简化这个操作，例如使用`Runnable`和`Callable`的wrapper。为了传递`SecurityContext`到`@Async`方法，需要提供一个`AsyncConfigurer`并且确保`Executor`是正确类型：

```java
@Configuration
public class ApplicationConfiguration extends AsyncConfigurerSupport {

  @Override
  public Executor getAsyncExecutor() {
    return new DelegatingSecurityContextExecutorService(Executors.newFixedThreadPool(5));
  }

}
```

