# Servlet堆栈上的Web

这部分文档包括对基于Servlet API构建并部署到Servlet容器的Servlet-stack Web应用程序的支持。 各个章节包括Spring MVC，View Technologies，CORS支持和WebSocket支持。 对于反应堆栈Web应用程序，请参阅Web on Reactive Stack。

## 1. Spring Web MVC

Spring Web MVC是基于Servlet API构建的原始Web框架，从一开始就包含在Spring Framework中。 正式名称“Spring Web MVC”来自其源模块（spring-webmvc）的名称，但它通常被称为“Spring MVC”。

与Spring Web MVC并行，Spring Framework 5.0引入了一个反应堆栈Web框架，其名称“Spring WebFlux”也基于其源模块（spring-webflux）。 本节介绍Spring Web MVC。 下一节将介绍Spring WebFlux。

有关基准信息以及与Servlet容器和Java EE版本范围的兼容性，请参阅Spring Framework Wiki

### 1.1. DispatcherServlet

与许多其他Web框架一样，Spring MVC围绕前端控制器模式设计，其中中央Servlet DispatcherServlet为请求处理提供共享算法，而实际工作由可配置委托组件执行。 该模型非常灵活，支持多种工作流程。

DispatcherServlet与任何Servlet一样，需要使用Java配置或web.xml根据Servlet规范进行声明和映射。 反过来，DispatcherServlet使用Spring配置来发现请求映射，视图解析，异常处理等所需的委托组件。

以下Java配置示例注册并初始化DispatcherServlet，它由Servlet容器自动检测（请参阅Servlet配置）：

~~~~ java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletCxt) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
        ac.register(AppConfig.class);
        ac.refresh();

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(ac);
        ServletRegistration.Dynamic registration = servletCxt.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}
~~~~

**除了直接使用ServletContext API之外，您还可以扩展AbstractAnnotationConfigDispatcherServletInitializer并覆盖特定方法（请参阅Context Hierarchy下的示例）。**

以下web.xml配置示例注册并初始化DispatcherServlet：

~~~~ xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
~~~~

**Spring Boot遵循不同的初始化顺序。 Spring Boot使用Spring配置来引导自身和嵌入式Servlet容器，而不是挂钩到Servlet容器的生命周期。 在Spring配置中检测Filter和Servlet声明，并在Servlet容器中注册。 有关更多详细信息，请参阅Spring Boot文档。**

#### 1.1.1 上下文层次结构

DispatcherServlet需要一个WebApplicationContext（普通ApplicationContext的扩展）来进行自己的配置。 WebApplicationContext有一个指向ServletContext的链接以及与之关联的Servlet。它还绑定到ServletContext，以便应用程序可以在RequestContextUtils上使用静态方法，以便在需要访问WebApplicationContext时查找它。

对于许多应用程序，拥有单个WebApplicationContext很简单且足够。也可以有一个上下文层次结构，其中一个根WebApplicationContext在多个DispatcherServlet（或其他Servlet）实例之间共享，每个实例都有自己的子WebApplicationContext配置。有关上下文层次结构功能的更多信息，请参阅ApplicationContext的其他功能。

根WebApplicationContext通常包含基础结构bean，例如需要跨多个Servlet实例共享的数据存储库和业务服务。这些bean是有效继承的，可以在特定于Servlet的子WebApplicationContext中重写（即重新声明），它通常包含给定Servlet本地的bean。下图显示了这种关系：

![](image/mvc-context-hierarchy.png)

以下示例配置WebApplicationContext层次结构：

~~~~ java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { App1Config.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" };
    }
}
~~~~

如果不需要应用程序上下文层次结构，则应用程序可以通过getRootConfigClasses（）返回所有配置，并从getServletConfigClasses（）返回null。

以下示例显示了web.xml等效项：

~~~~ xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app1</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/app1-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app1</servlet-name>
        <url-pattern>/app1/*</url-pattern>
    </servlet-mapping>

</web-app>
~~~~

**如果不需要应用程序上下文层次结构，则应用程序可以仅配置“根”上下文，并将contextConfigLocation Servlet参数保留为空。**

#### 1.1.2 特殊bean类

DispatcherServlet委托特殊bean处理请求并呈现适当的响应。 “特殊bean”是指实现框架契约的Spring管理的Object实例。 那些通常带有内置合同，但您可以自定义其属性并扩展或替换它们。

下表列出了DispatcherServlet检测到的特殊bean：

| Bean 类型                                                    | 说明                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `HandlerMapping`                                             | 将请求映射到处理程序以及[拦截器](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-handlermapping-interceptor)列表用于预处理和后处理。 映射基于一些标准，其细节因`HandlerMapping`实现而异。两个主要的`HandlerMapping`实现是`RequestMappingHandlerMapping`（支持`@RequestMapping`注解方法）和`SimpleUrlHandlerMapping`（它维护显式注册） 处理程序的URI路径模式）。 |
| `HandlerAdapter`                                             | 无论实际调用处理程序如何，都可以帮助`DispatcherServlet`调用映射到请求的处理程序。 例如，调用带注解的控制器需要解析注解。 `HandlerAdapter`的主要目的是保护`DispatcherServlet`免受这些细节的影响。 |
| [`HandlerExceptionResolver`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-exceptionhandlers) | 解决异常的策略，可能将它们映射到处理程序，HTML错误视图或其他目标。 请参阅[例外](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-exceptionhandlers)。 |
| [`ViewResolver`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-viewresolver) | 将从处理程序返回的逻辑“基于字符串”的视图名称解析为用于呈现给响应的实际“视图”。 请参阅[查看分辨率](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-viewresolver)和[查看技术](https：//docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-view)。 |
| [`LocaleResolver`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-localeresolver), [LocaleContextResolver](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-timezone) | 解决客户正在使用的“区域设置”以及可能的时区，以便能够提供国际化视图。 请参阅[区域设置](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-localeresolver)。 |
| [`ThemeResolver`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-themeresolver) | 解决Web应用程序可以使用的主题 - 例如，提供个性化布局。 请参阅[主题](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-themeresolver)。 |
| [`MultipartResolver`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-multipart) | 在一些多部分解析库的帮助下，解析多部分请求（例如，浏览器表单文件上载）的抽象。 请参阅[Multipart Resolver](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-multipart)。 |
| [`FlashMapManager`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-flash-attributes) | 存储和检索“输入”和“输出”`FlashMap`，可用于将属性从一个请求传递到另一个请求，通常是通过重定向。 请参阅[Flash属性](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-flash-attributes)。 |

#### 1.1.3 Web MVC配置

应用程序可以声明处理请求所需的特殊Bean类型中列出的基础结构bean。 DispatcherServlet检查每个特殊bean的WebApplicationContext。 如果没有匹配的bean类型，它将回退到DispatcherServlet.properties中列出的默认类型。

在大多数情况下，MVC Config是最佳起点。 它以Java或XML声明所需的bean，并提供更高级别的配置回调API来自定义它。

**Spring Boot依赖于MVC Java配置来配置Spring MVC并提供许多额外的方便选项。**

#### 1.1.4 Servlet配置

在Servlet 3.0+环境中，您可以选择以编程方式配置Servlet容器作为替代方法，也可以与web.xml文件结合使用。 以下示例注册DispatcherServlet：

~~~~ java
import org.springframework.web.WebApplicationInitializer;

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
~~~~

WebApplicationInitializer是Spring MVC提供的一个接口，可确保检测到您的实现并自动用于初始化任何Servlet 3容器。 名为AbstractDispatcherServletInitializer的WebApplicationInitializer的抽象基类实现通过重写方法来指定servlet映射和DispatcherServlet配置的位置，从而更容易注册DispatcherServlet。

对于使用基于Java的Spring配置的应用程序，建议使用此方法，如以下示例所示：

~~~~ java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
~~~~

如果使用基于XML的Spring配置，则应直接从AbstractDispatcherServletInitializer扩展，如以下示例所示：

~~~~~ java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return cxt;
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
~~~~~

AbstractDispatcherServletInitializer还提供了一种方便的方法来添加Filter实例，并将它们自动映射到DispatcherServlet，如下例所示：

~~~~ java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ...

    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
~~~~

每个过滤器都根据其具体类型添加默认名称，并自动映射到DispatcherServlet。
AbstractDispatcherServletInitializer的isAsyncSupported受保护方法提供了一个单独的位置，以便在DispatcherServlet和映射到它的所有过滤器上启用异步支持。 默认情况下，此标志设置为true。
最后，如果您需要进一步自定义DispatcherServlet本身，则可以覆盖createDispatcherServlet方法。

#### 1.1.5 处理

DispatcherServlet按如下方式处理请求：

将WebApplicationContext作为控制器和进程中的其他元素可以使用的属性在请求中进行搜索和绑定。它默认绑定在DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE键下。

语言环境解析器绑定到请求，以允许进程中的元素解析处理请求时使用的语言环境（呈现视图，准备数据等）。如果您不需要区域设置解析，则不需要区域设置解析器。

主题解析器绑定到请求，以允许视图等元素确定要使用的主题。如果您不使用主题，则可以忽略它。

如果指定多部分文件解析程序，则会检查请求的多部分。如果找到多部分，请求将包装在MultipartHttpServletRequest中，以供进程中的其他元素进一步处理。有关多部分处理的更多信息，请参见Multipart Resolver。

搜索适当的处理程序。如果找到处理程序，则执行与处理程序（预处理程序，后处理程序和控制器）关联的执行链以准备模型或呈现。或者，对于带注解的控制器，可以呈现响应（在HandlerAdapter中）而不是返回视图。

如果返回模型，则呈现视图。如果没有返回模型（可能是由于预处理器或后处理器拦截请求，可能是出于安全原因），则不会呈现任何视图，因为该请求可能已经完成。

WebApplicationContext中声明的HandlerExceptionResolver bean用于解决在请求处理期间抛出的异常。这些异常解析器允许自定义逻辑以解决异常。有关详细信息，请参阅例外。

Spring DispatcherServlet还支持返回最后修改日期，如Servlet API所指定。确定特定请求的最后修改日期的过程很简单：DispatcherServlet查找适当的处理程序映射并测试找到的处理程序是否实现LastModified接口。如果是这样，LastModified接口的long getLastModified（request）方法的值将返回给客户端。

您可以通过将Servlet初始化参数（init-param元素）添加到web.xml文件中的Servlet声明来自定义各个DispatcherServlet实例。下表列出了支持的参数：

| 参数                             | 说明                                                         |
| :------------------------------- | :----------------------------------------------------------- |
| `contextClass`                   | 实现`ConfigurableWebApplicationContext`的类，由此Servlet实例化并本地配置。 默认情况下，使用`XmlWebApplicationContext`。 |
| `contextConfigLocation`          | 传递给上下文实例的字符串（由`contextClass`指定）以指示可以找到上下文的位置。 该字符串可能包含多个字符串（使用逗号作为分隔符）以支持多个上下文。 对于具有两次定义的bean的多个上下文位置，最新位置优先。 |
| `namespace`                      | `WebApplicationContext`的命名空间。 默认为`[servlet-name] -servlet`。 |
| `throwExceptionIfNoHandlerFound` | 当没有找到请求的处理程序时是否抛出`NoHandlerFoundException`。 然后可以使用`HandlerExceptionResolver`捕获异常（例如，通过使用`@ExceptionHandler`控制器方法）并像其他任何其他方法一样处理。默认情况下，这设置为`false`，在这种情况下`DispatcherServlet`集 响应状态为404（NOT_FOUND）而不引发异常。注意，如果[默认servlet处理](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web .html #mvc-default-servlet-handler)也被配置，未解析的请求总是被转发到默认的servlet，永远不会引发404。 |

#### 1.1.6	侦听

所有HandlerMapping实现都支持处理程序拦截器，当您想要将特定功能应用于某些请求时，这些拦截器很有用 - 例如，检查主体。拦截器必须使用三种方法从org.springframework.web.servlet包中实现HandlerInterceptor，这三种方法应该提供足够的灵活性来进行各种预处理和后处理：
preHandle（..）：在执行实际处理程序之前
postHandle（..）：处理程序执行后
afterCompletion（..）：完成请求完成后
preHandle（..）方法返回一个布尔值。您可以使用此方法来中断或继续执行链的处理。当此方法返回true时，处理程序执行链继续。当它返回false时，DispatcherServlet假定拦截器本身已处理请求（例如，呈现适当的视图）并且不继续执行执行链中的其他拦截器和实际处理程序。
有关如何配置拦截器的示例，请参阅MVC配置一节中的拦截器。您也可以使用各个HandlerMapping实现上的setter直接注册它们。
请注意，对于在HandlerAdapter中和在postHandle之前编写和提交响应的@ResponseBody和ResponseEntity方法，postHandle不太有用。这意味着对响应进行任何更改都为时已晚，例如添加额外的标头。对于此类方案，您可以实现ResponseBodyAdvice并将其声明为Controller Advice bean或直接在RequestMappingHandlerAdapter上进行配置。

#### 1.1.7	异常

如果在请求映射期间发生异常或从请求处理程序（例如@Controller）抛出异常，则DispatcherServlet委托给HandlerExceptionResolver bean链以解决异常并提供替代处理，这通常是错误响应。

下表列出了可用的HandlerExceptionResolver实现：

| `HandlerExceptionResolver`                                   | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `SimpleMappingExceptionResolver`                             | 异常类名称和错误视图名称之间的异常处理。 用于在浏览器应用程序中呈现错误页面。 |
| [`DefaultHandlerExceptionResolver`](https://docs.spring.io/spring-framework/docs/5.1.9.RELEASE/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html) | 解决Spring MVC引发的异常并将它们映射到HTTP状态代码。 另请参阅替代`ResponseEntityExceptionHandler`和[REST API例外](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-rest-exceptions)。 |
| `ResponseStatusExceptionResolver`                            | 使用`@ ResponseStatus`注解解析异常，并根据注解中的值将它们映射到HTTP状态代码。 |
| `ExceptionHandlerExceptionResolver`                          | 通过在`@Controller`或`@ControllerAdvice`类中调用`@ExceptionHandler`方法来解决异常。 请参阅[@ExceptionHandler方法](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-exceptionhandler)。 |

###### 解析器链

您可以通过在Spring配置中声明多个HandlerExceptionResolver bean并根据需要设置其顺序属性来形成异常解析程序链。 order属性越高，异常解析器定位的越晚。
HandlerExceptionResolver的契约指定它可以返回：
一个指向错误视图的ModelAndView。
如果在解析程序中处理异常，则为空的ModelAndView。
如果异常仍然未解析，则为null，以便后续解析器尝试，如果异常保留在最后，则允许冒泡到Servlet容器。
MVC Config自动声明内置的解析器，用于默认的Spring MVC异常，@ ResponseStatus带注解的异常，以及对@ExceptionHandler方法的支持。 您可以自定义该列表或替换它。

###### 容器错误页面

如果任何HandlerExceptionResolver仍未解析异常，并且因此将其传播或者如果响应状态设置为错误状态（即4xx，5xx），则Servlet容器可以在HTML中呈现默认错误页面。 要自定义容器的默认错误页面，可以在web.xml中声明错误页面映射。 以下示例显示了如何执行此操作：

~~~~ xml
<error-page>
    <location>/error</location>
</error-page>
~~~~

根据前面的示例，当异常冒泡或响应具有错误状态时，Servlet容器会在容器内对配置的URL进行ERROR调度（例如，/ error）。 然后由DispatcherServlet处理，可能将其映射到@Controller，可以实现它以返回带有模型的错误视图名称或呈现JSON响应，如以下示例所示：

~~~~ java
@RestController
public class ErrorController {

    @RequestMapping(path = "/error")
    public Map<String, Object> handle(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("status", request.getAttribute("javax.servlet.error.status_code"));
        map.put("reason", request.getAttribute("javax.servlet.error.message"));
        return map;
    }
}
~~~~

**Servlet API不提供在Java中创建错误页面映射的方法。 但是，您可以同时使用WebApplicationInitializer和最小的web.xml。**

#### 1.1.8 查看解析

Spring MVC定义了ViewResolver和View接口，使您可以在浏览器中呈现模型，而无需将您与特定的视图技术联系起来。 ViewResolver提供视图名称和实际视图之间的映射。 View在移交给特定视图技术之前解决了数据的准备问题。
下表提供了有关ViewResolver层次结构的更多详细信息：

| 视图解析                         | 描述                                                         |
| :------------------------------- | :----------------------------------------------------------- |
| `AbstractCachingViewResolver`    | `AbstractCachingViewResolver`的子类缓存它们解析的视图实例。 缓存可提高某些视图技术的性能。 您可以通过将`cache`属性设置为`false`来关闭缓存。 此外，如果必须在运行时刷新某个视图（例如，修改FreeMarker模板时），则可以使用`removeFromCache（String viewName，Locale loc）`方法。 |
| `XmlViewResolver`                | 实现`ViewResolver`，它接受用XML编写的配置文件，使用与Spring的XML bean工厂相同的DTD。 默认配置文件是`/ WEB-INF / views.xml`。 |
| `ResourceBundleViewResolver`     | `ViewResolver`的实现，它使用由bundle base name指定的`ResourceBundle`中的bean定义。 对于它应该解析的每个视图，它使用属性`[viewname]。（class）`的值作为视图类，并使用属性`[viewname] .url`的值作为视图URL。 您可以在[查看技术](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-view)一章中找到示例。 |
| `UrlBasedViewResolver`           | 简单实现`ViewResolver`接口，它影响逻辑视图名称直接解析为没有显式映射定义的URL。 如果您的逻辑名称以直接的方式与视图资源的名称匹配，则这是合适的，而不需要任意映射。 |
| `InternalResourceViewResolver`   | `UrlBasedViewResolver`的便捷子类，支持`InternalResourceView`（实际上是Servlets和JSP）和子类，如`JstlView`和`TilesView`。 您可以使用`setViewClass（..）`为此解析程序生成的所有视图指定视图类。 请参阅[`UrlBasedViewResolver`](https://docs.spring.io/spring-framework/docs/5.1.9.RELEASE/javadoc-api/org/springframework/web/reactive/result/view/UrlBasedViewResolver.html)javadoc了解详情。 |
| `FreeMarkerViewResolver`         | `UrlBasedViewResolver`的便捷子类，支持`FreeMarkerView`及其自定义子类。 |
| `ContentNegotiatingViewResolver` | 实现`ViewResolver`接口，该接口根据请求文件名或`Accept`标头解析视图。 请参阅[内容协商](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-multiple-representations)。 |

###### 处理

您可以通过声明多个解析程序bean链接视图解析程序，并在必要时通过设置order属性来指定排序。 请记住，order属性越高，视图解析器在链中的位置越晚。

ViewResolver的契约指定它可以返回null以指示无法找到该视图。 但是，对于JSP和InternalResourceViewResolver，确定JSP是否存在的唯一方法是通过RequestDispatcher执行调度。 因此，您必须始终将InternalResourceViewResolver配置为视图解析器的整体顺序中的最后一个。

配置视图解析就像将SpringResolver bean添加到Spring配置一样简单。 MVC Config为View Resolvers提供专用配置API，并添加无逻辑视图控制器，这些控制器对于没有控制器逻辑的HTML模板渲染非常有用。

###### 重定向

视图名称中的特殊重定向：前缀允许您执行重定向。 UrlBasedViewResolver（及其子类）将此识别为需要重定向的指令。 视图名称的其余部分是重定向URL。
净效果与控制器返回RedirectView的效果相同，但现在控制器本身可以按逻辑视图名称操作。 逻辑视图名称（例如redirect：/ myapp / some / resource）相对于当前Servlet上下文重定向，而诸如redirect：https：//myhost.com/some/arbitrary/path之类的名称重定向到绝对URL。
请注意，如果使用@ResponseStatus注解控制器方法，则注解值优先于RedirectView设置的响应状态。

###### 转发

您还可以为最终由UrlBasedViewResolver和子类解析的视图名称使用特殊的forward：前缀。 这将创建一个InternalResourceView，它执行RequestDispatcher.forward（）。 因此，此前缀对于InternalResourceViewResolver和InternalResourceView（对于JSP）没有用，但如果您使用其他视图技术但仍希望强制Servlet / JSP引擎处理资源的转发，则它可能会有所帮助。 请注意，您也可以链接多个视图解析器。

###### 内容协商

ContentNegotiatingViewResolver本身不解析视图，而是委托给其他视图解析器，并选择类似于客户端请求的表示的视图。可以从Accept标头或查询参数（例如，“/ path？format = pdf”）确定表示。
ContentNegotiatingViewResolver通过将请求媒体类型与与其每个ViewResolvers关联的视图支持的媒体类型（也称为Content-Type）进行比较，选择适当的View来处理请求。列表中具有兼容Content-Type的第一个View将表示返回给客户端。如果ViewResolver链无法提供兼容视图，则会查询通过DefaultViews属性指定的视图列表。后一个选项适用于单个视图，它可以呈现当前资源的适当表示，而不管逻辑视图名称如何。 Accept标头可以包含通配符（例如text / *），在这种情况下，Content-Type为text / xml的View是兼容匹配。
有关配置详细信息，请参阅MVC配置下的View Resolvers。

#### 1.1.9	当地

正如Spring Web MVC框架所做的那样，Spring架构的大多数部分都支持国际化。 DispatcherServlet允许您使用客户端的语言环境自动解析消息。这是通过LocaleResolver对象完成的。
当请求进入时，DispatcherServlet会查找区域设置解析程序，如果找到，则会尝试使用它来设置区域设置。通过使用RequestContext.getLocale（）方法，您始终可以检索由区域设置解析程序解析的区域设置。
除了自动语言环境解析之外，您还可以将拦截器附加到处理程序映射（有关处理程序映射拦截器的更多信息，请参阅拦截）以在特定情况下更改语言环境（例如，基于请求中的参数）。
区域设置解析器和拦截器在org.springframework.web.servlet.i18n包中定义，并以正常方式在应用程序上下文中进行配置。 Spring中包含以下选择的语言环境解析器。
时区
标题解析器
Cookie解析器
会话解析器
Locale Interceptor

###### 时区

除了获取客户端的语言环境之外，了解其时区通常也很有用。 LocaleContextResolver接口提供了LocaleResolver的扩展，它允许解析器提供更丰富的LocaleContext，其中可能包含时区信息。
可用时，可以使用RequestContext.getTimeZone（）方法获取用户的TimeZone。 时区信息由Spring的ConversionService注册的任何Date / Time Converter和Formatter对象自动使用。

###### 标题解析器

此区域设置解析程序检查客户端（例如，Web浏览器）发送的请求中的accept-language标头。 通常，此标头字段包含客户端操作系统的区域设置。 请注意，此解析程序不支持时区信息。

###### Cookie解析器

此区域设置解析程序检查客户端上可能存在的Cookie，以查看是否指定了Locale或TimeZone。 如果是，则使用指定的详细信息。 通过使用此区域设置解析程序的属性，您可以指定cookie的名称以及最大年龄。 以下示例定义CookieLocaleResolver：

~~~~ xml
<bean id="localeResolver" class="org.springframework.web.servlet.i18n.CookieLocaleResolver">

    <property name="cookieName" value="clientlanguage"/>

    <!-- in seconds. If set to -1, the cookie is not persisted (deleted when browser shuts down) -->
    <property name="cookieMaxAge" value="100000"/>

</bean>
~~~~

下表描述了属性`CookieLocaleResolver`：

| 属性           | 默认值                    | 描述                                                         |
| :------------- | :------------------------ | :----------------------------------------------------------- |
| `cookieName`   | classname + LOCALE        | Cookie的名称                                                 |
| `cookieMaxAge` | Servlet container default | Cookie在客户端上持续存在的最长时间。 如果指定了`-1`，则不会保留cookie。 它仅在客户端关闭浏览器之前可用。 |
| `cookiePath`   | /                         | 限制cookie对您网站某个部分的可见性。 指定`cookiePath`时，cookie仅对该路径及其下方的路径可见。 |

###### 会话解析器

SessionLocaleResolver允许您从可能与用户请求关联的会话中检索Locale和TimeZone。 与CookieLocaleResolver相比，此策略将本地选择的区域设置存储在Servlet容器的HttpSession中。 因此，这些设置对于每个会话都是临时的，因此在每个会话终止时丢失。

请注意，与外部会话管理机制没有直接关系，例如Spring Session项目。 此SessionLocaleResolver根据当前的HttpServletRequest评估和修改相应的HttpSession属性。

###### Locale Interceptor

您可以通过将LocaleChangeInterceptor添加到其中一个HandlerMapping定义来启用语言环境的更改。 它检测请求中的参数并相应地更改语言环境，在调度程序的应用程序上下文中调用LocaleResolver上的setLocale方法。 下一个示例显示，对包含名为siteLanguage的参数的所有* .view资源的调用现在更改了区域设置。 因此，例如，对URL的请求https://www.sf.net/home.view?siteLanguage=nl将网站语言更改为荷兰语。 以下示例显示如何拦截区域设置：

~~~~ xml
<bean id="localeChangeInterceptor"
        class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
    <property name="paramName" value="siteLanguage"/>
</bean>

<bean id="localeResolver"
        class="org.springframework.web.servlet.i18n.CookieLocaleResolver"/>

<bean id="urlMapping"
        class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="interceptors">
        <list>
            <ref bean="localeChangeInterceptor"/>
        </list>
    </property>
    <property name="mappings">
        <value>/**/*.view=someController</value>
    </property>
</bean>
~~~~

#### 1.1.10	主题

您可以应用Spring Web MVC框架主题来设置应用程序的整体外观，从而增强用户体验。 主题是静态资源的集合，通常是样式表和图像，它们会影响应用程序的视觉样式。

###### 定义主题

要在Web应用程序中使用主题，必须设置org.springframework.ui.context.ThemeSource接口的实现。 WebApplicationContext接口扩展了ThemeSource，但将其职责委托给专用实现。 默认情况下，委托是org.springframework.ui.context.support.ResourceBundleThemeSource实现，它从类路径的根目录加载属性文件。 要使用自定义ThemeSource实现或配置ResourceBundleThemeSource的基本名称前缀，可以在应用程序上下文中使用保留名称themeSource注册bean。 Web应用程序上下文自动检测具有该名称的bean并使用它。

使用ResourceBundleThemeSource时，主题在简单属性文件中定义。 属性文件列出构成主题的资源，如以下示例所示：

~~~~ properties
styleSheet=/themes/cool/style.css
background=/themes/cool/img/coolBg.jpg
~~~~

属性的键是从视图代码引用主题元素的名称。 对于JSP，通常使用spring：theme自定义标记执行此操作，该标记与spring：message标记非常相似。 以下JSP片段使用上一示例中定义的主题来自定义外观：

~~~~ html
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags"%>
<html>
    <head>
        <link rel="stylesheet" href="<spring:theme code='styleSheet'/>" type="text/css"/>
    </head>
    <body style="background=<spring:theme code='background'/>">
        ...
    </body>
</html>
~~~~

默认情况下，ResourceBundleThemeSource使用空的基本名称前缀。 因此，属性文件从类路径的根加载。 因此，您可以将cool.properties主题定义放在类路径根目录的目录中（例如，在/ WEB-INF / classes中）。 ResourceBundleThemeSource使用标准的Java资源包加载机制，允许主题的完全国际化。 例如，我们可以有一个/WEB-INF/classes/cool_nl.properties，它引用一个带有荷兰文本的特殊背景图像。

###### 解决主题

定义主题后，如上一节所述，您可以决定使用哪个主题。 DispatcherServlet查找名为themeResolver的bean，以找出要使用的ThemeResolver实现。 主题解析器的工作方式与LocaleResolver的工作方式大致相同。 它检测用于特定请求的主题，还可以更改请求的主题。 下表描述了Spring提供的主题解析器：

| 类                     | 描述                                                         |
| :--------------------- | :----------------------------------------------------------- |
| `FixedThemeResolver`   | 选择使用`defaultThemeName`属性设置的固定主题。               |
| `SessionThemeResolver` | 主题在用户的HTTP会话中维护。 它只需要为每个会话设置一次，但不会在会话之间保留。 |
| `CookieThemeResolver`  | 所选主题存储在客户端的cookie中。                             |

Spring还提供了一个ThemeChangeInterceptor，它允许使用简单的请求参数对每个请求进行主题更改。

#### 1.1.11  Multipart  解析器

org.springframework.web.multipart包中的MultipartResolver是一种用于解析包括文件上载在内的多部分请求的策略。 有一个基于Commons FileUpload的实现，另一个基于Servlet 3.0多部分请求解析。

要启用多部分处理，需要在DispatcherServlet Spring配置中使用multipartResolver的名称声明MultipartResolver bean。 DispatcherServlet检测到它并将其应用于传入请求。 当收到内容类型为multipart / form-data的POST时，解析器会解析内容并将当前的HttpServletRequest包装为MultipartHttpServletRequest，以提供对已解析部分的访问，并将其作为请求参数公开。

###### Apache Commons `FileUpload`

要使用Apache Commons FileUpload，您可以配置名为multipartResolver的CommonsMultipartResolver类型的bean。 您还需要将commons-fileupload作为类路径的依赖项。

###### Servlet 3.0

需要通过Servlet容器配置启用Servlet 3.0多部分解析。 为此：
在Java中，在Servlet注册上设置MultipartConfigElement。
在web.xml中，将“<multipart-config>”部分添加到servlet声明中。
以下示例显示如何在Servlet注册上设置MultipartConfigElement：

~~~~ java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // ...

    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {

        // Optionally also set maxFileSize, maxRequestSize, fileSizeThreshold
        registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
    }

}
~~~~

一旦Servlet 3.0配置到位，您就可以添加名为MultipartResolver的StandardServletMultipartResolver类型的bean。

#### 1.1.12. Logging

Spring MVC中的DEBUG级别日志记录旨在实现紧凑，简约和人性化。 它侧重于高价值的信息，这些信息对于仅在调试特定问题时有用的其他信息一次又一次有用。

TRACE级日志记录通常遵循与DEBUG相同的原则（例如，也不应该是消防软管），但可以用于调试任何问题。 此外，一些日志消息可能在TRACE与DEBUG中显示不同的详细程度。

良好的日志记录来自使用日志的经验。 如果您发现任何不符合既定目标的事件，请告诉我们。

###### 敏感数据

DEBUG和TRACE日志记录可能会记录敏感信息。 这就是默认情况下屏蔽请求参数和标头的原因，并且必须通过DispatcherServlet上的enableLoggingRequestDetails属性显式启用它们的完整日志记录。

以下示例说明如何使用Java配置执行此操作：

~~~~ java
public class MyInitializer
        extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return ... ;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return ... ;
    }

    @Override
    protected String[] getServletMappings() {
        return ... ;
    }

    @Override
    protected void customizeRegistration(Dynamic registration) {
        registration.setInitParameter("enableLoggingRequestDetails", "true");
    }

}
~~~~

### 1.2	过滤器

spring-web模块提供了一些有用的过滤器：
Form Data
Forwarded Headers
Shallow ETag
CORS

#### 1.2.1. Form Data

浏览器只能通过HTTP GET或HTTP POST提交表单数据，但非浏览器客户端也可以使用HTTP PUT，PATCH和DELETE。 Servlet API要求ServletRequest.getParameter *（）方法仅支持HTTP POST的表单字段访问。

spring-web模块提供FormContentFilter来拦截HTTP PUT，PATCH和DELETE请求，内容类型为application / x-www-form-urlencoded，从请求正文中读取表单数据，并包装ServletRequest以使 通过ServletRequest.getParameter *（）系列方法提供表单数据。

#### 1.2.2 转发标题

当请求通过代理（例如负载平衡器）时，主机，端口和方案可能会发生变化，这使得从客户端角度创建指向正确主机，端口和方案的链接成为一项挑战。

RFC 7239定义了代理可用于提供有关原始请求的信息的转发HTTP标头。还有其他非标准头文件，包括X-Forwarded-Host，X-Forwarded-Port，X-Forwarded-Proto，X-Forwarded-Ssl和X-Forwarded-Prefix。

ForwardedHeaderFilter是一个Servlet过滤器，它根据Forwarded标头修改请求的主机，端口和方案，然后删除这些标头。

转发标头存在安全注意事项，因为应用程序无法知道标头是由代理按预期添加还是由恶意客户端添加。这就是为什么应该将信任边界的代理配置为删除来自外部的不受信任的转发标头。您还可以使用removeOnly = true配置ForwardedHeaderFilter，在这种情况下，它会删除但不使用标头。

#### 1.2.3. Shallow ETag

ShallowEtagHeaderFilter过滤器通过缓存写入响应的内容并从中计算MD5哈希来创建Shallow ETag。 客户端下次发送时，它会执行相同的操作，但它也会将计算值与If-None-Match请求标头进行比较，如果两者相等，则返回304（NOT_MODIFIED）。

此策略可节省网络带宽，但不能节省CPU，因为必须为每个请求计算完整响应。 前面描述的控制器级别的其他策略可以避免计算。 请参阅HTTP缓存。

此过滤器具有writeWeakETag参数，该参数将过滤器配置为写入弱ETag，类似于以下内容：W /“02a2d595e6ed9a0b24f027f2b63b134d6”（如RFC 7232第2.3节中所定义）。

#### 1.2.4. CORS

Spring MVC通过控制器上的注解为CORS配置提供细粒度的支持。 但是，当与Spring Security一起使用时，我们建议依赖于必须在Spring Security的过滤器链之前订购的内置CorsFilter。

有关更多详细信息，请参阅CORS和CORS过滤器部分。

### 1.3 带注解的控制器

Spring MVC提供了一个基于注解的编程模型，其中@Controller和@RestController组件使用注解来表达请求映射，请求输入，异常处理等。 带注解的控制器具有灵活的方法签名，不必扩展基类，也不必实现特定的接口。 以下示例显示了由注解定义的控制器：

~~~~~ java
@Controller
public class HelloController {

    @GetMapping("/hello")
    public String handle(Model model) {
        model.addAttribute("message", "Hello World!");
        return "index";
    }
}
~~~~~

在前面的示例中，该方法接受Model并将视图名称作为String返回，但是存在许多其他选项，本章稍后将对其进行说明。

**有关spring.io的指南和教程使用本节中介绍的基于注解的编程模型。**

#### 1.3.1 声明

您可以使用Servlet的WebApplicationContext中的标准Spring bean定义来定义控制器bean。 @Controller构造型允许自动检测，与Spring一致支持，以检测类路径中的@Component类并自动注册它们的bean定义。 它还充当带注解的类的构造型，表明它作为Web组件的角色。
要启用此类@Controller bean的自动检测，您可以将组件扫描添加到Java配置中，如以下示例所示：

~~~~ java
@Configuration
@ComponentScan("org.example.web")
public class WebConfig {

    // ...
}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.example.web"/>

    <!-- ... -->

</beans>
~~~~

@RestController是一个组合注解，它本身用@Controller和@ResponseBody进行元注解，以指示一个控制器，它的每个方法都继承了类型级@ResponseBody注解，因此直接写入响应主体，而不是视图分辨率和使用 HTML模板。

###### AOP代理

在某些情况下，您可能需要在运行时使用AOP代理装饰控制器。 一个例子是如果您选择直接在控制器上使用@Transactional注解。 在这种情况下，对于控制器而言，我们建议使用基于类的代理。 这通常是控制器的默认选择。 但是，如果控制器必须实现不是Spring Context回调的接口（例如InitializingBean，* Aware等），则可能需要显式配置基于类的代理。 例如，使用<tx：annotation-driven />，您可以更改为<tx：annotation-driven proxy-target-class =“true”/>。

#### 1.3.2 请求映射

您可以使用@RequestMapping批注将请求映射到控制器方法。 它具有各种属性，可通过URL，HTTP方法，请求参数，标头和媒体类型进行匹配。 您可以在类级别使用它来表示共享映射，或者在方法级别使用它来缩小到特定的端点映射。
还有@RequestMapping的HTTP方法特定快捷方式变体：
@GetMapping
@PostMapping
@PutMapping
@DeleteMapping
@PatchMapping
快捷方式是提供的自定义注解，因为可以说，大多数控制器方法应该映射到特定的HTTP方法而不是使用@RequestMapping，默认情况下，它与所有HTTP方法匹配。 同样，在类级别仍然需要@RequestMapping来表示共享映射。

以下示例具有类型和方法级别映射：

~~~~ java
@RestController
@RequestMapping("/persons")
class PersonController {

    @GetMapping("/{id}")
    public Person getPerson(@PathVariable Long id) {
        // ...
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public void add(@RequestBody Person person) {
        // ...
    }
}
~~~~

###### URI模式

您可以使用以下全局模式和通配符映射请求：
？ 匹配一个字符
*匹配路径段中的零个或多个字符
**匹配零个或多个路径段
您还可以使用@PathVariable声明URI变量并访问它们的值，如以下示例所示：

~~~~ java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
~~~~

您可以在类和方法级别声明URI变量，如以下示例所示：

~~~~ java
@Controller
@RequestMapping("/owners/{ownerId}")
public class OwnerController {

    @GetMapping("/pets/{petId}")
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }
}
~~~~

URI变量自动转换为适当的类型，或引发TypeMismatchException。 默认情况下支持简单类型（int，long，Date等），您可以注册对任何其他数据类型的支持。 请参见类型转换和数据绑定器。
您可以显式命名URI变量（例如，@ PathVariable（“customId”）），但是如果名称相同并且您的代码是使用调试信息或使用Java 8上的-parameters编译器标志编译的，则可以保留该详细信息。。
语法{varName：regex}声明一个URI变量，其正则表达式的语法为{varName：regex}。 例如，给定URL“/spring-web-3.0.5 .jar”，以下方法提取名称，版本和文件扩展名：

~~~~ java
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariable String version, @PathVariable String ext) {
    // ...
}
~~~~

URI路径模式还可以嵌入$ {...}占位符，这些占位符在启动时通过对本地，系统，环境和其他属性源使用PropertyPlaceHolderConfigurer来解析。 例如，您可以使用此参数来基于某些外部配置参数化基本URL。

**Spring MVC使用PathMatcher契约和spring-core的AntPathMatcher实现进行URI路径匹配。**

###### 模式比较

当多个模式与URL匹配时，必须对它们进行比较以找到最佳匹配。 这是通过使用AntPathMatcher.getPatternComparator（String path）来完成的，它会查找更具体的模式。

如果URI变量的计数较少（计为1），单个通配符（计为1）和双通配符（计为2），则模式的特定性较低。 给定相等的分数，选择较长的模式。 给定相同的分数和长度，选择具有比通配符更多的URI变量的模式。

默认映射模式（/ **）从评分中排除，并始终排在最后。 此外，前缀模式（例如/ public / **）被认为不具有不具有双通配符的其他模式的特定性。

有关完整的详细信息，请参阅AntPathMatcher中的AntPatternComparator，并记住您可以自定义PathMatcher实现。 请参阅配置部分中的路径匹配。

###### 后缀匹配

默认情况下，Spring MVC执行。*后缀模式匹配，以便映射到/ person的控制器也隐式映射到/person.*。然后使用文件扩展名来解释用于响应的请求内容类型（即，而不是Accept标头） - 例如，/ person.pdf，/person.xml等。
当浏览器用于发送难以一致解释的Accept标头时，必须以这种方式使用文件扩展名。目前，这不再是必需品，使用Accept标头应该是首选。
随着时间的推移，文件扩展名的使用已经证明有多种方式存在问题。当使用URI变量，路径参数和URI编码进行覆盖时，它可能会导致歧义。关于基于URL的授权和安全性的推理（更多细节见下一节）也变得更加困难。
要完全禁用文件扩展名，必须同时设置以下两项：
useSuffixPatternMatching（false），请参阅PathMatchConfigurer
favorPathExtension（false），请参阅ContentNegotiationConfigurer
基于URL的内容协商仍然有用（例如，在浏览器中键入URL时）。为此，我们建议使用基于查询参数的策略来避免文件扩展名带来的大多数问题。或者，如果必须使用文件扩展名，请考虑通过ContentNegotiationConfigurer的mediaTypes属性将它们限制为显式注册的扩展名列表。

###### 后缀匹配和RFD

反射文件下载（RFD）攻击类似于XSS，因为它依赖于响应中反映的请求输入（例如，查询参数和URI变量）。但是，RFD攻击不依赖于将JavaScript插入HTML，而是依赖浏览器切换来执行下载，并在以后双击时将响应视为可执行脚本。
在Spring MVC中，@ ResponsePody和ResponseEntity方法存在风险，因为它们可以呈现不同的内容类型，客户端可以通过URL路径扩展来请求。禁用后缀模式匹配并使用路径扩展进行内容协商可降低风险，但不足以防止RFD攻击。
为了防止RFD攻击，在呈现响应主体之前，Spring MVC添加了一个Content-Disposition：inline; filename = f.txt标头来建议一个固定且安全的下载文件。仅当URL路径包含既未列入白名单也未明确注册用于内容协商的文件扩展名时，才会执行此操作。但是，当直接在浏览器中输入URL时，它可能会产生副作用。
默认情况下，许多常见路径扩展名列入白名单。具有自定义HttpMessageConverter实现的应用程序可以显式注册文件扩展名以进行内容协商，以避免为这些扩展添加Content-Disposition标头。请参阅内容类型。
有关RFD的其他建议，请参阅CVE-2015-5211。

###### Consumable 类型

您可以根据请求的Content-Type缩小请求映射范围，如以下示例所示：

~~~~ java
@PostMapping(path = "/pets", consumes = "application/json") 
public void addPet(@RequestBody Pet pet) {
    // ...
}
~~~~

consumes属性还支持否定表达式 - 例如，！text / plain表示除text / plain之外的任何内容类型。
您可以在类级别声明共享使用属性。 但是，与大多数其他请求映射属性不同，在类级别使用时，方法级别使用属性覆盖而不是扩展类级别声明。

**MediaType为常用媒体类型提供常量，例如APPLICATION_JSON_VALUE和APPLICATION_XML_VALUE。**

###### 可生产的媒体类型

您可以根据Accept请求标头和控制器方法生成的内容类型列表来缩小请求映射，如以下示例所示：

~~~~ java
@GetMapping(path = "/pets/{petId}", produces = "application/json;charset=UTF-8") 
@ResponseBody
public Pet getPet(@PathVariable String petId) {
    // ...
}
~~~~

媒体类型可以指定字符集。 支持否定表达式 - 例如，！text / plain表示“text / plain”以外的任何内容类型。

**对于JSON内容类型，即使RFC7159明确指出“没有为此注册定义charset参数”，也应指定UTF-8字符集，因为某些浏览器要求它正确解释UTF-8特殊字符。**

您可以在类级别声明共享的yield属性。 但是，与大多数其他请求映射属性不同，在类级别使用时，方法级别会生成属性覆盖，而不是扩展类级别声明。

**MediaType为常用媒体类型提供常量，例如APPLICATION_JSON_UTF8_VALUE和APPLICATION_XML_VALUE。**

###### 参数，标题

您可以根据请求参数条件缩小请求映射。 您可以测试是否存在请求参数（myParam），缺少一个（！myParam）或特定值（myParam = myValue）。 以下示例显示如何测试特定值：

~~~~ java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
~~~~

您还可以将其与请求标头条件一起使用，如以下示例所示：

~~~~ java
@GetMapping(path = "/pets", headers = "myHeader=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
~~~~

**您可以将Content-Type和Accept与headers条件匹配，但最好使用consume并生成。**

###### HTTP HEAD，OPTIONS

@GetMapping（和@RequestMapping（method = HttpMethod.GET））透明地支持HTTP HEAD以进行请求映射。控制器方法无需更改。应用于javax.servlet.http.HttpServlet的响应包装器确保将Content-Length头设置为写入的字节数（不实际写入响应）。
@GetMapping（和@RequestMapping（method = HttpMethod.GET））被隐式映射到并支持HTTP HEAD。处理HTTP HEAD请求就像它是HTTP GET一样，除了编写字节数而不是写入主体，并设置Content-Length头。
默认情况下，通过将Allow响应头设置为所有具有匹配URL模式的@RequestMapping方法中列出的HTTP方法列表来处理HTTP OPTIONS。
对于没有HTTP方法声明的@RequestMapping，Allow标头设置为GET，HEAD，POST，PUT，PATCH，DELETE，OPTIONS。控制器方法应始终声明支持的HTTP方法（例如，通过使用HTTP方法特定的变体：@ GetMapping，@ PostMapping等）。
您可以将@RequestMapping方法显式映射到HTTP HEAD和HTTP OPTIONS，但在常见情况下这不是必需的。

###### 自定义注解

Spring MVC支持使用组合注解进行请求映射。 这些注解本身是用@RequestMapping进行元注解的，并且用于重新声明具有更窄，更具体目的的@RequestMapping属性的子集（或全部）。
@ GetMapping，@ PostMapping，@ PutMapping，@ DeleMapping和@PatchMapping是组合注解的示例。 提供它们是因为，可以说，大多数控制器方法应该映射到特定的HTTP方法而不是使用@RequestMapping，默认情况下，它与所有HTTP方法匹配。 如果您需要组合注解的示例，请查看如何声明这些注解。
Spring MVC还支持使用自定义请求匹配逻辑的自定义请求映射属性。 这是一个更高级的选项，需要继承RequestMappingHandlerMapping并覆盖getCustomMethodCondition方法，您可以在其中检查自定义属性并返回自己的RequestCondition。

###### 明确的注册

您可以以编程方式注册处理程序方法，可以将其用于动态注册或高级情况，例如不同URL下的同一处理程序的不同实例。 以下示例注册处理程序方法：

~~~~ java
@Configuration
public class MyConfig {

    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler) 
            throws NoSuchMethodException {

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{id}").methods(RequestMethod.GET).build(); 

        Method method = UserHandler.class.getMethod("getUser", Long.class); 

        mapping.registerMapping(info, handler, method); 
    }

}
~~~~

#### 1.3.3 处理程序方法

@RequestMapping处理程序方法具有灵活的签名，可以从一系列受支持的控制器方法参数和返回值中进行选择。

###### 方法参数

下表描述了受支持的控制器方法参数。 任何参数都不支持反应类型。
JDK 8的java.util.Optional作为方法参数与具有必需属性的注解（例如，@ RequestParam，@ RequestHeader等）相结合，并且等效于required = false。

| Controller方法                                               | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `WebRequest`, `NativeWebRequest`                             | 无需直接使用Servlet API即可访问请求参数以及请求和会话属性。  |
| `javax.servlet.ServletRequest`, `javax.servlet.ServletResponse` | 选择任何特定的请求或响应类型 - 例如，`ServletRequest`，`HttpServletRequest`，或Spring的`MultipartRequest`，`MultipartHttpServletRequest`。 |
| `javax.servlet.http.HttpSession`                             | 强制进行会话。 因此，这样的论证永远不会是“无效”。 请注意，会话访问不是线程安全的。 如果允许多个请求同时访问会话，请考虑将`RequestMappingHandlerAdapter`实例的`synchronizeOnSession`标志设置为`true`。 |
| `javax.servlet.http.PushBuilder`                             | 用于编程HTTP / 2资源推送的Servlet 4.0推送构建器API。 请注意，根据Servlet规范，如果客户端不支持该HTTP / 2功能，则注入的`PushBuilder`实例可以为null。 |
| `java.security.Principal`                                    | 目前经过身份验证的用户 - 如果已知，可能是特定的`Principal`实现类。 |
| `HttpMethod`                                                 | 请求的HTTP方法                                               |
| `java.util.Locale`                                           | 当前请求语言环境，由最具体的`LocaleResolver`可用（实际上是配置的`LocaleResolver`或`LocaleContextResolver`）确定。 |
| `java.util.TimeZone` + `java.time.ZoneId`                    | 与当前请求关联的时区，由“LocaleContextResolver”确定。        |
| `java.io.InputStream`, `java.io.Reader`                      | 用于访问Servlet API公开的原始请求主体。                      |
| `java.io.OutputStream`, `java.io.Writer`                     | 用于访问Servlet API公开的原始响应主体。                      |
| `@PathVariable`                                              | 用于访问URI模板变量。 请参阅[URI模式](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-requestmapping-uri-templates)。 |
| `@MatrixVariable`                                            | 用于访问URI路径段中的名称 - 值对。 请参阅[Matrix Variables](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-matrix-variables)。 |
| `@RequestParam`                                              | 用于访问Servlet请求参数，包括多部分文件。 参数值将转换为声明的方法参数类型。 参见[`@RequestParam`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-requestparam)以及[Multipart](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-multipart-forms.Note)使用`@RequestParam`是可选的简单 参数值。 请参阅本表末尾的“任何其他参数”。 |
| `@RequestHeader`                                             | 用于访问请求标头。 标头值将转换为声明的方法参数类型。 参见[`@RequestHeader`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-requestheader)。 |
| `@CookieValue`                                               | 用于访问cookie。 Cookie值将转换为声明的方法参数类型。 参见[`@CookieValue`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-cookievalue)。 |
| `@RequestBody`                                               | 用于访问HTTP请求正文。 使用`HttpMessageConverter`实现将正文内容转换为声明的方法参数类型。 参见[`@RequestBody`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-requestbody)。 |
| `HttpEntity<B>`                                              | 用于访问请求标头和正文。 正文用`HttpMessageConverter`转换。 请参阅[HttpEntity](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-httpentity)。 |
| `@RequestPart`                                               | 要访问`multipart / form-data`请求中的一部分，请使用`HttpMessageConverter`转换部件的主体。 请参见[Multipart](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-multipart-forms)。 |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap` | 用于访问HTML控制器中使用的模型，并将其作为视图呈现的一部分暴露给模板。 |
| `RedirectAttributes`                                         | 指定在重定向（即，要附加到查询字符串）的情况下使用的属性，以及临时存储的flash属性，直到重定向后的请求为止。 请参阅[重定向属性](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-redirecting-passing-data)和[Flash属性](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-flash-attributes)。 |
| `@ModelAttribute`                                            | 用于访问模型中的现有属性（如果不存在则实例化），并应用数据绑定和验证。 请参阅[`@ ModelAttribute`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-modelattrib-method-args）以及 as [Model]（https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-modelattrib-methods)和[`DataBinder`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-initbinder).Note使用`@ ModelAttribute`是可选的（例如 ，设置其属性）。 请参阅本表末尾的“任何其他参数”。 |
| `Errors`, `BindingResult`                                    | 用于访问命令对象的验证和数据绑定错误（即`@ ModelAttribute`参数）或验证`@RequestBody`或`@RequestPart`参数的错误。 您必须在经过验证的方法参数后立即声明“错误”或“BindingResult”参数。 |
| `SessionStatus` + class-level `@SessionAttributes`           | 用于标记表单处理完成，它触发通过类级别`@ SessionAttributes`注解声明的会话属性的清除。 有关详细信息，请参阅[`@ SessionAttributes`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-sessionattributes)。 |
| `UriComponentsBuilder`                                       | 用于准备相对于当前请求的主机，端口，方案，上下文路径和servlet映射的文字部分的URL。 参见[URI链接](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-uri-building)。 |
| `@SessionAttribute`                                          | 用于访问任何会话属性，与由于类级别@ SessionAttributes声明而存储在会话中的模型属性相反。 有关详细信息，请参阅[`@ SessionAttribute`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-sessionattribute)。 |
| `@RequestAttribute`                                          | （用于访问请求属性。请参阅[`@RequestAttribute`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-requestattrib ) 更多细节。） |
| Any other argument                                           | 如果方法参数与此表中的任何早期值不匹配，并且它是一个简单类型（由BeanUtils＃isSimpleProperty确定，则它被解析为@RequestParam。否则，它将被解析为@ModelAttribute。 |

###### 返回值

下表描述了支持的控制器方法返回值。 所有返回值都支持反应类型。

| Controller 方法返回值                                        | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `@ResponseBody`                                              | 返回值通过`HttpMessageConverter`implementations转换并写入响应。 参见[`@ ResponseBody`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-responsebody)。 |
| `HttpEntity<B>`, `ResponseEntity<B>`                         | 指定完整响应（包括HTTP头和主体）的返回值将通过`HttpMessageConverter`实现转换并写入响应。 请参阅[ResponseEntity](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-responseentity)。 |
| `HttpHeaders`                                                | 用于返回带标题且没有正文的响应。                             |
| `String`                                                     | 使用`ViewResolver`实现解析的视图名称，与隐式模型一起使用 - 通过命令对象和`@ ModelAttribute`方法确定。 处理程序方法还可以通过声明一个`Model`参数以编程方式丰富模型（参见[Explicit Registrations](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web) html的＃MVC-ANN-requestmapping登记））。 |
| `View`                                                       | 用于与隐式模型一起渲染的`View`实例 - 通过命令对象和`@ ModelAttribute`方法确定。 处理程序方法还可以通过声明一个`Model`参数以编程方式丰富模型（参见[Explicit Registrations](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web) html的＃MVC-ANN-requestmapping登记））。 |
| `java.util.Map`, `org.springframework.ui.Model`              | 要添加到隐式模型的属性，通过`RequestToViewNameTranslator`隐式确定视图名称。 |
| `@ModelAttribute`                                            | 要添加到模型的属性，通过`RequestToViewNameTranslator`隐式确定视图名称。注意`@ ModelAttribute`是可选的。 请参见本表末尾的“任何其他返回值”。 |
| `ModelAndView` object                                        | 要使用的视图和模型属性，以及（可选）响应状态。               |
| `void`                                                       | 具有`void`返回类型（或`null`返回值）的方法被认为完全处理了响应，如果它还有一个`ServletResponse`，一个`OutputStream`参数或一个`@ ResponseStatus`注解。 如果控制器已经进行了积极的“ETag”或“lastModified”时间戳检查（参见[控制器]）(https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework)，情况也是如此。 -reference / web.html＃mvc-caching-etag-lastmodified）以获取详细信息。如果以上都不是真的，那么`void`返回类型也可以指示REST控制器的“无响应主体”或默认视图名称选择 用于HTML控制器。 |
| `DeferredResult<V>`                                          | 从任何线程异步生成任何前面的返回值 - 例如，由于某些事件或回调。 请参阅[异步请求](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async)和[`DeferredResult`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async-deferredresult)。 |
| `Callable<V>`                                                | 在Spring MVC管理的线程中异步生成上述任何返回值。 请参阅[异步请求](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async)和[`Callable`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async-callable)。 |
| ListenableFuture,CompletionStage,CompletableFuture           | 替代`DeferredResult`，为方便起见（例如，当底层服务返回其中一个时）。 |
| `ResponseBodyEmitter`, `SseEmitter`                          | 异步发送对象流以使用`HttpMessageConverter`实现写入响应。 也支持`ResponseEntity`的主体。 请参阅[异步请求](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async)和[HTTP Streaming](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async-http-streaming)。 |
| `StreamingResponseBody`                                      | 异步写入响应`OutputStream`。 也支持`ResponseEntity`的主体。 请参阅[异步请求](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async)和[HTTP Streaming](https：//docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async-http-streaming)。 |
| Reactive types — Reactor, RxJava, or others through `ReactiveAdapterRegistry` | 替换为带有多值流的`DeferredResult`（例如，`Flux`，`Observable`）收集到`List`。用于流式场景（例如，`text / event-stream`，`application / json + stream `），`SseEmitter`和`ResponseBodyEmitter`代替使用，其中`ServletOutputStream`阻塞I / O在Spring MVC管理的线程上执行，并且对每次写入的完成应用反压。参见[异步请求](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async)和[Reactive Types](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-async-reactive-types)。 |
| Any other return value                                       | 任何与此表中的任何早期值都不匹配的返回值，即`String`或`void`被视为视图名称（通过`RequestToViewNameTranslator`选择默认视图名称），前提是它不是一个简单的 类型，由[BeanUtils＃isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.9.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)确定。 简单类型的值仍未解决。 |

###### 类型转换

如果参数声明为String以外的其他参数，则表示基于String的请求输入的某些带注解的控制器方法参数（例如@ RequestParam，@ RequestHeader，@ PathVariable，@ MatrixVariable和@CookieValue）可能需要进行类型转换。
对于此类情况，将根据配置的转换器自动应用类型转换。 默认情况下，支持简单类型（int，long，Date和其他）。 您可以通过WebDataBinder（请参阅DataBinder）或使用FormattingConversionService注册Formatters来自定义类型转换。 请参阅Spring Field Formatting。

###### 矩阵变量

RFC 3986讨论了路径段中的名称 - 值对。 在Spring MVC中，我们将那些基于Tim Berners-Lee的“旧帖子”称为“矩阵变量”，但它们也可以称为URI路径参数。
矩阵变量可以出现在任何路径段中，每个变量用分号分隔，多个值用逗号分隔（例如，/ cars; color = red，green; year = 2012）。 也可以通过重复的变量名称指定多个值（例如，color = red; color = green; color = blue）。
如果URL预计包含矩阵变量，则控制器方法的请求映射必须使用URI变量来屏蔽该变量内容，并确保请求可以成功匹配，而与矩阵变量顺序和存在无关。 以下示例使用矩阵变量：

~~~ java
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
public void findPet(@PathVariable String petId, @MatrixVariable int q) {

    // petId == 42
    // q == 11
}
~~~

鉴于所有路径段都可能包含矩阵变量，您有时可能需要消除矩阵变量预期所在的路径变量的歧义。以下示例说明了如何执行此操作:

~~~~ java
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22
}
~~~~

矩阵变量可以定义为可选，并指定默认值，如以下示例所示：

~~~ java
// GET /pets/42

@GetMapping("/pets/{petId}")
public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {

    // q == 1
}
~~~

要获取所有矩阵变量，可以使用MultiValueMap，如以下示例所示:

~~~~ java
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 22, "s" : 23]
}
~~~~

请注意，您需要启用矩阵变量的使用。 在MVC Java配置中，您需要通过路径匹配将removeSemicolonContent = false设置为UrlPathHelper。 在MVC XML命名空间中，您可以设置<mvc：annotation-driven enable-matrix-variables =“true”/>。

##### `@RequestParam`

您可以使用@RequestParam批注将Servlet请求参数（即查询参数或表单数据）绑定到控制器中的方法参数。
以下示例显示了如何执行此操作：

~~~~ java
@Controller
@RequestMapping("/pets")
public class EditPetForm {

    // ...

    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) { 
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }

    // ...

}
~~~~

默认情况下，需要使用此批注的方法参数，但您可以通过将@RequestParam批注的required标志设置为false或通过使用java.util.Optional包装器声明参数来指定方法参数是可选的。
如果目标方法参数类型不是String，则会自动应用类型转换。请参阅类型转换。
将参数类型声明为数组或列表允许为同一参数名称解析多个参数值。
当@RequestParam注解声明为Map <String，String>或MultiValueMap <String，String>时，如果注解中未指定参数名称，则会使用每个给定参数名称的请求参数值填充映射。
请注意，使用@RequestParam是可选的（例如，设置其属性）。默认情况下，任何简单值类型的参数（由BeanUtils＃isSimpleProperty确定）并且不被任何其他参数解析器解析，都被视为使用@RequestParam进行注解。

##### `@RequestHeader`

您可以使用@RequestHeader批注将请求标头绑定到控制器中的方法参数。

考虑以下请求，标题为:

~~~~ properties
Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300
~~~~

以下示例获取Accept-Encoding和Keep-Alive标头的值:

~~~~ java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
~~~~

如果目标方法参数类型不是String，则会自动应用类型转换。 请参阅类型转换。
在Map <String，String>，MultiValueMap <String，String>或HttpHeaders参数上使用@RequestHeader注解时，将使用所有标头值填充映射。

**内置支持可用于将逗号分隔的字符串转换为字符串或字符串集或类型转换系统已知的其他类型。 例如，使用@RequestHeader（“Accept”）注解的方法参数可以是String类型，也可以是String []或List <String>。**

##### `@CookieValue`

您可以使用@CookieValue批注将HTTP cookie的值绑定到控制器中的方法参数。
考虑使用以下cookie的请求：

~~~~ properties
JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84
~~~~

以下示例显示了如何获取cookie值:

~~~~ java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
~~~~

如果目标方法参数类型不是String，则会自动应用类型转换。 请参阅类型转换。

###### `@ModelAttribute`

您可以在方法参数上使用@ModelAttribute批注来从模型访问属性，或者如果不存在则将其实例化。 model属性还覆盖了名称与字段名称匹配的HTTP Servlet请求参数的值。 这称为数据绑定，它使您不必处理解析和转换单个查询参数和表单字段。 以下示例显示了如何执行此操作：

~~~~ java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { } 
~~~~

上面的Pet实例解析如下：
如果已经使用Model添加了模型。
通过使用@SessionAttributes从HTTP会话。
从通过Converter的URI路径变量（参见下一个示例）。
从默认构造函数的调用。
从调用具有与Servlet请求参数匹配的参数的“主构造函数”。 参数名称通过JavaBeans @ConstructorProperties或字节码中的运行时保留参数名称确定。
虽然通常使用Model来使用属性填充模型，但另一种替代方法是依赖于Converter <String，T>和URI路径变量约定。 在以下示例中，模型属性名称account匹配URI路径变量account，并通过将String字符串号通过已注册的Converter <String，Account>来加载帐户：

~~~~ java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
~~~~

获取模型属性实例后，将应用数据绑定。 WebDataBinder类将Servlet请求参数名称（查询参数和表单字段）与目标Object上的字段名称进行匹配。 在必要时，在应用类型转换后填充匹配字段。 有关数据绑定（和验证）的更多信息，请参阅验证。 有关自定义数据绑定的更多信息，请参阅DataBinder。
数据绑定可能导致错误。 默认情况下，会引发BindException。 但是，要在控制器方法中检查此类错误，可以在@ModelAttribute旁边添加一个BindingResult参数，如以下示例所示：

~~~~ java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
~~~~

在某些情况下，您可能希望在没有数据绑定的情况下访问模型属性。 对于这种情况，您可以将模型注入控制器并直接访问它，或者设置@ModelAttribute（binding = false），如下例所示:

~~~~ java
@ModelAttribute
public AccountForm setUpForm() {
    return new AccountForm();
}

@ModelAttribute
public Account findAccount(@PathVariable String accountId) {
    return accountRepository.findOne(accountId);
}

@PostMapping("update")
public String update(@Valid AccountForm form, BindingResult result,
        @ModelAttribute(binding=false) Account account) { 
    // ...
}
~~~~

您可以通过添加javax.validation.Valid批注或Spring的@Validated批注（Bean验证和Spring验证），在数据绑定后自动应用验证。 以下示例显示了如何执行此操作：

~~~~ java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
~~~~

请注意，使用@ModelAttribute是可选的（例如，设置其属性）。 默认情况下，任何非简单值类型的参数（由BeanUtils＃isSimpleProperty确定）并且未被任何其他参数解析器解析，都被视为使用@ModelAttribute进行注解。

###### `@SessionAttributes`

@SessionAttributes用于在请求之间的HTTP Servlet会话中存储模型属性。 它是一个类型级别的注解，用于声明特定控制器使用的会话属性。 这通常列出模型属性的名称或模型属性的类型，这些属性应该透明地存储在会话中以供后续访问请求使用。
以下示例使用@SessionAttributes注解：

~~~~ java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {
    // ...
}
~~~~

在第一个请求中，当名称为pet的模型属性添加到模型中时，它会自动提升并保存在HTTP Servlet会话中。 它保持不变，直到另一个控制器方法使用SessionStatus方法参数来清除存储，如下例所示：

~~~~ java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) {
        if (errors.hasErrors) {
            // ...
        }
            status.setComplete(); 
            // ...
        }
    }
}
~~~~

###### `@SessionAttribute`

如果您需要访问全局管理的预先存在的会话属性（例如，在控制器外部 - 例如，通过过滤器）并且可能存在或不存在，则可以对方法参数使用@SessionAttribute批注，如 以下示例显示:

~~~~ java
@RequestMapping("/")
public String handle(@SessionAttribute User user) { 
    // ...
}
~~~~

对于需要添加或删除会话属性的用例，请考虑将org.springframework.web.context.request.WebRequest或javax.servlet.http.HttpSession注入控制器方法。
要在会话中临时存储模型属性作为控制器工作流的一部分，请考虑使用@SessionAttributes中所述的@SessionAttributes。

###### `@RequestAttribute`

与@SessionAttribute类似，您可以使用@RequestAttribute批注来访问先前创建的预先存在的请求属性（例如，通过Servlet过滤器或HandlerInterceptor）：

~~~~ java
@GetMapping("/")
public String handle(@RequestAttribute Client client) { 
    // ...
}
~~~~

###### 重定向属性

默认情况下，所有模型属性都被视为在重定向URL中作为URI模板变量公开。在其余属性中，原始类型或集合或基本类型数组的属性会自动附加为查询参数。
如果专门为重定向准备了模型实例，则将原始类型属性作为查询参数附加可以是期望的结果。但是，在带注释的控制器中，模型可以包含为渲染目的而添加的其他属性（例如，下拉字段值）。为了避免在URL中出现此类属性的可能性，@ RequestMapping方法可以声明RedirectAttributes类型的参数，并使用它来指定可供RedirectView使用的确切属性。如果方法重定向，则使用RedirectAttributes的内容。否则，使用模型的内容。
RequestMappingHandlerAdapter提供了一个名为ignoreDefaultModelOnRedirect的标志，您可以使用该标志指示如果控制器方法重定向，则永远不应使用默认模型的内容。相反，控制器方法应声明RedirectAttributes类型的属性，如果不这样做，则不应将任何属性传递给RedirectView。 MVC命名空间和MVC Java配置都将此标志设置为false，以保持向后兼容性。但是，对于新应用程序，我们建议将其设置为true。
请注意，扩展重定向URL时，当前请求中的URI模板变量会自动变为可用，您无需通过Model或RedirectAttributes显式添加它们。以下示例显示如何定义重定向：

~~~~ java
@PostMapping("/files/{path}")
public String upload(...) {
    // ...
    return "redirect:files/{path}";
}
~~~~

将数据传递到重定向目标的另一种方法是使用flash属性。 与其他重定向属性不同，Flash属性保存在HTTP会话中（因此，不会出现在URL中）。 有关更多信息，请参阅Flash属性。

###### Flash属性

Flash属性为一个请求提供了一种存储打算在另一个请求中使用的属性的方法。重定向时最常需要这种方法 - 例如，Post-Redirect-Get模式。 Flash重定向（通常在会话中）之前临时保存Flash属性，以便在重定向后对请求可用并立即删除。
Spring MVC有两个主要的抽象支持flash属性。 FlashMap用于保存Flash属性，而FlashMapManager用于存储，检索和管理FlashMap实例。
Flash属性支持始终处于“打开”状态，无需显式启用。但是，如果不使用，它永远不会导致HTTP会话创建。在每个请求中，都有一个“输入”FlashMap，其中包含从先前请求（如果有）传递的属性，以及一个“输出”FlashMap，其中包含要为后续请求保存的属性。两个FlashMap实例都可以通过RequestContextUtils中的静态方法从Spring MVC中的任何位置访问。
带注释的控制器通常不需要直接使用FlashMap。相反，@ RequestMapping方法可以接受RedirectAttributes类型的参数，并使用它为重定向方案添加flash属性。通过RedirectAttributes添加的Flash属性会自动传播到“输出”FlashMap。同样，在重定向之后，“输入”FlashMap的属性会自动添加到为目标URL提供服务的控制器的模型中。

~~~~ 
匹配flash属性的请求
Flash属性的概念存在于许多其他Web框架中，并且已经证明有时会暴露于并发问题。 这是因为，根据定义，flash属性将被存储直到下一个请求。 但是，非常“下一个”请求可能不是预期的接收者而是另一个异步请求（例如，轮询或资源请求），在这种情况下，过早删除flash属性。

为了减少此类问题的可能性，RedirectView使用目标重定向URL的路径和查询参数自动“标记”FlashMap实例。 反过来，默认的FlashMapManager在查找“输入”FlashMap时将该信息与传入请求进行匹配。

这并不能完全消除并发问题的可能性，但会使用重定向URL中已有的信息大大减少并发问题。 因此，我们建议您主要使用Flash属性进行重定向方案。
~~~~

###### Multipart

启用MultipartResolver后，将解析具有multipart / form-data的POST请求的内容，并将其作为常规请求参数进行访问。 以下示例访问一个常规表单字段和一个上载文件:

~~~~ java
@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(@RequestParam("name") String name,
            @RequestParam("file") MultipartFile file) {

        if (!file.isEmpty()) {
            byte[] bytes = file.getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
~~~~

将参数类型声明为List <MultipartFile>允许为同一参数名称解析多个文件。
当@RequestParam注解声明为Map <String，MultipartFile>或MultiValueMap <String，MultipartFile>时，如果注解中未指定参数名称，则会使用每个给定参数名称的多部分文件填充地图。

**使用Servlet 3.0多部分解析，您还可以将javax.servlet.http.Part而不是Spring的MultipartFile声明为方法参数或集合值类型.**

您还可以将多部分内容用作绑定到命令对象的数据的一部分。 例如，前面示例中的表单字段和文件可以是表单对象上的字段，如以下示例所示:

~~~~ java
class MyForm {

    private String name;

    private MultipartFile file;

    // ...
}

@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(MyForm form, BindingResult errors) {
        if (!form.getFile().isEmpty()) {
            byte[] bytes = form.getFile().getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
~~~~

还可以在RESTful服务方案中从非浏览器客户端提交多部分请求。 以下示例显示了带有JSON的文件:

~~~~ pro
POST /someUrl
Content-Type: multipart/mixed

--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json; charset=UTF-8
Content-Transfer-Encoding: 8bit

{
    "name": "value"
}
--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="file-data"; filename="file.properties"
Content-Type: text/xml
Content-Transfer-Encoding: 8bit
... File Data ...
~~~~

您可以使用@RequestParam作为String访问“元数据”部分，但您可能希望它从JSON反序列化（类似于@RequestBody）。 在使用HttpMessageConverter转换后，使用@RequestPart批注访问多部分：

~~~~ java
@PostMapping("/")
public String handle(@RequestPart("meta-data") MetaData metadata,
        @RequestPart("file-data") MultipartFile file) {
    // ...
}
~~~~

您可以将@RequestPart与javax.validation.Valid结合使用，或使用Spring的@Validated注解，这两种注解都会导致应用标准Bean验证。 默认情况下，验证错误会导致MethodArgumentNotValidException，并将其转换为400（BAD_REQUEST）响应。 或者，您可以通过Errors或BindingResult参数在控制器内本地处理验证错误，如以下示例所示：

~~~~ java
@PostMapping("/")
public String handle(@Valid @RequestPart("meta-data") MetaData metadata,
        BindingResult result) {
    // ...
}
~~~~

###### `@RequestBody`

您可以使用@RequestBody批注通过HttpMessageConverter将请求主体读取并反序列化为Object。 以下示例使用@RequestBody参数:

~~~~ java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) {
    // ...
}
~~~~

您可以使用MVC配置的“消息转换器”选项来配置或自定义消息转换。
您可以将@RequestBody与javax.validation.Valid或Spring的@Validated注解结合使用，这两种注解都会导致应用标准Bean验证。 默认情况下，验证错误会导致MethodArgumentNotValidException，并将其转换为400（BAD_REQUEST）响应。 或者，您可以通过Errors或BindingResult参数在控制器内本地处理验证错误，如以下示例所示：

```java
@PostMapping("/accounts")
public void handle(@Valid @RequestBody Account account, BindingResult result) {
    // ...
}
```

###### HttpEntity

HttpEntity与使用@RequestBody或多或少相同，但它基于一个公开请求标题和正文的容器对象。 以下清单显示了一个示例：

~~~~ java
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) {
    // ...
}
~~~~

###### `@ResponseBody`

您可以在方法上使用@ResponseBody批注，以通过HttpMessageConverter将返回序列化到响应主体。 以下清单显示了一个示例：

~~~~ java
@GetMapping("/accounts/{id}")
@ResponseBody
public Account handle() {
    // ...
}
~~~~

类级别也支持@ResponseBody，在这种情况下，它由所有控制器方法继承。 这是@RestController的效果，它只不过是一个用@Controller和@ResponseBody标记的元注释。
您可以将@ResponseBody与反应类型一起使用。 有关更多详细信息，请参阅异步请求和活动类型。
您可以使用MVC配置的“消息转换器”选项来配置或自定义消息转换。
您可以将@ResponseBody方法与JSON序列化视图结合使用。 有关详细信息，请参阅Jackson JSON。

###### ResponseEntity

ResponseEntity与@ResponseBody类似，但具有状态和标题。 例如;

~~~~ java
@GetMapping("/something")
public ResponseEntity<String> handle() {
    String body = ... ;
    String etag = ... ;
    return ResponseEntity.ok().eTag(etag).build(body);
}
~~~~

Spring MVC支持使用单值反应类型异步生成ResponseEntity，和/或主体的单值和多值反应类型。

###### Jackson JSON

Spring为Jackson JSON库提供支持。

Spring MVC为Jackson的序列化视图提供内置支持，允许仅渲染Object中所有字段的子集。 要将其与@ResponseBody或ResponseEntity控制器方法一起使用，您可以使用Jackson的@JsonView批注来激活序列化视图类，如以下示例所示：

~~~~ java
@RestController
public class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

public class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
~~~~

**@JsonView允许一组视图类，但每个控制器方法只能指定一个。 如果需要激活多个视图，可以使用复合接口。**

对于依赖于视图分辨率的控制器，可以将序列化视图类添加到模型中，如以下示例所示:

~~~~ java
@Controller
public class UserController extends AbstractController {

    @GetMapping("/user")
    public String getUser(Model model) {
        model.addAttribute("user", new User("eric", "7!jd#h23"));
        model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class);
        return "userView";
    }
}
~~~~

#### 1.3.4 模型

您可以使用@ModelAttribute注解：
在@RequestMapping方法中的方法参数，用于从模型创建或访问Object并通过WebDataBinder将其绑定到请求。
作为@Controller或@ControllerAdvice类中的方法级注解，有助于在任何@RequestMapping方法调用之前初始化模型。
在@RequestMapping方法上标记其返回值是一个模型属性。
本节讨论@ModelAttribute方法 - 前面列表中的第二项。控制器可以包含任意数量的@ModelAttribute方法。在同一控制器中的@RequestMapping方法之前调用所有这些方法。 @ModelAttribute方法也可以通过@ControllerAdvice在控制器之间共享。有关更多详细信息，请参阅控制器建议部分。
@ModelAttribute方法具有灵活的方法签名。它们支持许多与@RequestMapping方法相同的参数，但@ModelAttribute本身或与请求体相关的任何内容除外。
以下示例显示了@ModelAttribute方法：

~~~~ java
@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountRepository.findAccount(number));
    // add more ...
}
~~~~

以下示例仅添加一个属性：

~~~~ java
@ModelAttribute
public Account addAccount(@RequestParam String number) {
    return accountRepository.findAccount(number);
}
~~~~

**如果未明确指定名称，则会根据对象类型选择默认名称，如约定的javadoc中所述。 您始终可以使用重载的addAttribute方法或@ModelAttribute上的name属性（返回值）来指定显式名称。**

您还可以将@ModelAttribute用作@RequestMapping方法的方法级注解，在这种情况下，@ RequestMapping方法的返回值将被解释为模型属性。 这通常不是必需的，因为它是HTML控制器中的默认行为，除非返回值是否则将被解释为视图名称的String。 @ModelAttribute还可以自定义模型属性名称，如以下示例所示：

~~~~ java
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
public Account handle() {
    // ...
    return account;
}
~~~~

#### 1.3.5. `DataBinder`

@Controller或@ControllerAdvice类可以使用@InitBinder方法初始化WebDataBinder的实例，而这些方法又可以：
将请求参数（即表单或查询数据）绑定到模型对象。
将基于字符串的请求值（例如请求参数，路径变量，标题，cookie等）转换为目标类型的控制器方法参数。
在呈现HTML表单时将模型对象值格式化为String值。
@InitBinder方法可以注册特定于控制器的java.bean.PropertyEditor或Spring Converter和Formatter组件。 此外，您可以使用MVC配置在全局共享的FormattingConversionService中注册Converter和Formatter类型。
@InitBinder方法支持许多与@RequestMapping方法相同的参数，但@ModelAttribute（命令对象）参数除外。 通常，它们使用WebDataBinder参数（用于注册）和void返回值声明。 以下清单显示了一个示例：

~~~~ java
@Controller
public class FormController {

    @InitBinder 
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }

    // ...
}
~~~~

或者，当您通过共享的FormattingConversionService使用基于Formatter的设置时，您可以重复使用相同的方法并注册特定于控制器的Formatter实现，如以下示例所示:

~~~~ java
@Controller
public class FormController {

    @InitBinder 
    protected void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }

    // ...
}
~~~~

#### 1.3.6. Exceptions

@Controller和@ControllerAdvice类可以使用@ExceptionHandler方法来处理来自控制器方法的异常，如下例所示:

~~~~ java
@Controller
public class SimpleController {

    // ...

    @ExceptionHandler
    public ResponseEntity<String> handle(IOException ex) {
        // ...
    }
}
~~~~

该异常可能与传播的顶级异常（即抛出的直接IOException）或顶级包装器异常中的直接原因（例如，包含在IllegalStateException内的IOException）相匹配。
对于匹配的异常类型，最好将目标异常声明为方法参数，如前面的示例所示。 当多个异常方法匹配时，根异常匹配通常优先于原因异常匹配。 更具体地说，ExceptionDepthComparator用于根据抛出的异常类型的深度对异常进行排序。
或者，注释声明可以缩小要匹配的异常类型，如以下示例所示：

~~~~ java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(IOException ex) {
    // ...
}
~~~~

您甚至可以使用具有非常通用参数签名的特定异常类型列表，如以下示例所示：

~~~~ java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(Exception ex) {
    // ...
}
~~~~

**根和原因异常匹配之间的区别可能是令人惊讶的。
在前面显示的IOException变体中，通常使用实际的FileSystemException或RemoteException实例作为参数调用该方法，因为它们都从IOException扩展。 但是，如果任何此类匹配异常在包装器异常中传播，而该包装异常本身就是IOException，则传入的异常实例就是包装器异常。
在句柄（Exception）变体中，行为更简单。 这总是在包装场景中使用包装器异常调用，在这种情况下可以通过ex.getCause（）找到实际匹配的异常。 传入的异常仅在将这些异常作为顶级异常抛出时才是实际的FileSystemException或RemoteException实例。**

我们通常建议您在参数签名中尽可能具体，减少root和cause异常类型之间不匹配的可能性。考虑将多匹配方法分解为单独的@ExceptionHandler方法，每个方法通过其签名匹配单个特定异常类型。
在多@ControllerAdvice安排中，我们建议在@ControllerAdvice上声明您的主根异常映射，并使用相应的顺序进行优先级排序。虽然根异常匹配优先于某个原因，但这是在给定控制器或@ControllerAdvice类的方法中定义的。这意味着优先级较高的@ControllerAdvice bean上的原因匹配优先于较低优先级的@ControllerAdvice bean上的任何匹配（例如，root）。
最后但并非最不重要的是，@ ExceptionHandler方法实现可以选择通过以原始形式重新抛出它来退出处理给定的异常实例。这在您仅对根级别匹配或在无法静态确定的特定上下文中的匹配中感兴趣的情况下非常有用。重新抛出的异常通过剩余的解析链传播，就好像给定的@ExceptionHandler方法首先不匹配一样。
Spring MVC中对@ExceptionHandler方法的支持是基于DispatcherServlet级别的HandlerExceptionResolver机制构建的。

###### 方法参数

@ExceptionHandler方法支持以下参数：

| 方法论证                                                     | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| Exception type                                               | 用于访问引发的异常。                                         |
| `HandlerMethod`                                              | 用于访问引发异常的控制器方法。                               |
| `WebRequest`, `NativeWebRequest`                             | 无需直接使用Servlet API即可访问请求参数以及请求和会话属性。  |
| `javax.servlet.ServletRequest`, `javax.servlet.ServletResponse` | 选择任何特定的请求或响应类型（例如，`ServletRequest`或`HttpServletRequest`或Spring的`MultipartRequest`或`MultipartHttpServletRequest`）。 |
| `javax.servlet.http.HttpSession`                             | 强制进行会话。 因此，这样的论证永远不会是“无效”。 请注意，会话访问不是线程安全的。 如果允许多个请求同时访问会话，请考虑将`RequestMappingHandlerAdapter`实例的`synchronizeOnSession`标志设置为`true`。 |
| `java.security.Principal`                                    | 目前经过身份验证的用户 - 如果已知，可能是特定的`Principal`实现类。 |
| `HttpMethod`                                                 | 请求的HTTP方法。                                             |
| `java.util.Locale`                                           | 当前请求语言环境，由最具体的`LocaleResolver`可用 - 实际上是配置的`LocaleResolver`或`LocaleContextResolver`。 |
| `java.util.TimeZone`, `java.time.ZoneId`                     | 与当前请求关联的时区，由“LocaleContextResolver”确定。        |
| `java.io.OutputStream`, `java.io.Writer`                     | 用于访问原始响应主体，由Servlet API公开。                    |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap` | 用于访问模型以获取错误响应。 永远是空的。                    |
| `RedirectAttributes`                                         | 指定在重定向的情况下使用的属性。 请参阅[重定向属性](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-redirecting-passing-data)和[Flash属性](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-flash-attributes)。 |
| `@SessionAttribute`                                          | 用于访问任何会话属性，与由于类级别@ SessionAttributes声明而存储在会话中的模型属性相反。 有关详细信息，请参阅[`@ SessionAttribute`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-sessionattribute)。 |
| `@RequestAttribute`                                          | 用于访问请求属性。 有关详细信息，请参阅[`@RequestAttribute`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-requestattrib)。 |

###### 返回值

@ExceptionHandler方法支持以下返回值：

| 返回值                                          | 描述                                                         |
| :---------------------------------------------- | :----------------------------------------------------------- |
| `@ResponseBody`                                 | 返回值通过`HttpMessageConverter`实例转换并写入响应。 参见[`@ ResponseBody`](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-responsebody)。 |
| `HttpEntity<B>`, `ResponseEntity<B>`            | HttpMessageConverter`实例并写入响应。 请参阅[ResponseEntity](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/web.html#mvc-ann-responseentity)。 |
| `String`                                        | 使用`ViewResolver`解析的视图名称实现并与隐式模型一起使用 - 通过命令对象和`@ ModelAttribute`方法确定。 因此，处理程序方法可以通过声明一个`model`参数（如前所述）以编程方式丰富模型。 |
| `View`                                          | 与隐式模型一起使用的`View`实例 - 通过命令对象和`@ ModelAttribute`方法确定。 因此，处理程序方法可以通过声明“模型”参数（先前描述的）来以编程方式丰富模型。 |
| `java.util.Map`, `org.springframework.ui.Model` | 要添加到隐式模型的属性，其中视图名称由`RequestToViewNameTranslator`隐式确定。 |
| `@ModelAttribute`                               | 通过`RequestToViewNameTranslator`隐式确定。注意``ModelAttribute`是可选的。 请参见本表末尾的“任何其他返回值”。 |
| `ModelAndView` object                           | 要使用的视图和模型属性，以及（可选）响应状态。               |
| `void`                                          | 一个带有`void`返回类型（或`null`返回值）的方法，因此它有一个`ServletResponse`到`OutputStream`argument，或者一个`@ ResponseStatus`注解。 如果控制器进行了积极的“ETag”或“lastModified”时间戳检查，也是如此（参见[控制器](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework) -reference / web.html＃mvc-caching-etag-lastmodified）有关详细信息）。如果以上都不是真的，那么`void`返回类型可以指示REST控制器的“无响应主体”或默认视图名称选择 HTML控制器。 |
| Any other return value                          | 如果返回值与上述任何一个都不匹配，而不是简单类型（由[BeanUtils＃isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.9.RELEASE/javadoc-api / org / springframework / beans / BeanUtils.html＃isSimpleProperty-java.lang.Class-)确定，默认情况下，它被视为模型。 如果它是一个简单的类型，它仍然没有得到解决。 |

##### REST API例外

REST服务的一个常见要求是在响应正文中包含错误详细信息。 Spring Framework不会自动执行此操作，因为响应主体的详细信息是特定于应用程序的。 但是，@ RestController可以使用带有ResponseEntity返回值的@ExceptionHandler方法来设置响应的状态和正文。 因此，可以在@ControllerAdvice类中声明搜索方法以全局应用它们。
ResponseEntityExceptionHandler，它提供对Spring MVC引发的异常的处理，并提供钩子来定制响应主体。 要使用它，请创建ResponseEntityExceptionHandler的子类，使用@ControllerAdvice注释它，覆盖必要的方法，并将其声明为Spring bean。

#### 1.3.7 控制器建议

通常，@ ExceptionHandler，@ InitBinder和@ModelAttribute方法适用于声明它们的@Controller类（或类层次结构）。如果要应用全局应用方法（跨控制器），可以在使用@ControllerAdvice或@RestControllerAdvice注释的类中声明它们。
@ControllerAdvice用@Component注释，这意味着搜索bean可以通过组件扫描注册为Spring bean。 @RestControllerAdvice是一个用@ControllerAdvice和@ResponseBody注释的组合注释，这意味着@ExceptionHandler方法被呈现给body响应消息（与视图分辨率或模板呈现相对）。
在启动时，@ RequestMapping和@ExceptionHandler方法的基础结构类检测用@ControllerAdvice注释的Spring bean，然后在运行时应用它们的方法。全局@ExceptionHandler方法（来自@ControllerAdvice）应用于本地方法（来自@Controller）。相比之下，全局@ModelAttribute和@InitBinder方法在本地方法之前应用。
默认情况下，@ ControllerAdvice方法适用于每个请求（即所有控制器），但您可以通过使用注释上的属性将其缩小到控制器的子集，如以下示例所示：

~~~~ java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
~~~~

前面示例中的选择器在运行时进行评估，如果广泛使用，可能会对性能产生负面影响。 有关更多详细信息，请参阅@ControllerAdvice javadoc。

### 1.4 URI链接

Spring框架中提供了此部分以使用URI

#### 1.4.1	UriComponents

Spring MVC和Spring WebFlux
UriComponentsBuilder有助于使用变量从URI模板构建URI，如以下示例所示：

~~~~ java
UriComponents uriComponents = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")  
        .queryParam("q", "{q}")  
        .encode() 
        .build(); 

URI uri = uriComponents.expand("Westin", "123").toUri();  
~~~~

前面的示例可以合并到一个链中，并使用buildAndExpand缩短，如以下示例所示:

~~~~ java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .encode()
        .buildAndExpand("Westin", "123")
        .toUri();
~~~~

您还可以通过转到URI（这意味着编码）来缩短它，如下例所示:

~~~~ java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
~~~~

您将对完整的URI模板更满意，如以下示例所示:

~~~~ java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}?q={q}")
        .build("Westin", "123");
~~~~

#### 1.4.2	UriBuilder

Spring MVC和Spring WebFlux
UriComponentsBuilder实现了UriBuilder。 您可以使用UriBuilderFactory创建一个UriBuilder。 UriBuilderFactory和UriBuilder一起提供了一种可插拔机制，可以根据共享配置，搜索作为基本URL，编码首选项和其他详细信息，从URI模板构建URI。
您可以使用UriBuilderFactory配置RestTemplate和WebClient以自定义URI的准备。 DefaultUriBuilderFactory是UriBuilderFactory的默认实现，它在内部使用UriComponentsBuilder并公开共享配置选项。
以下示例显示如何配置RestTemplate：

~~~~ java
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode;

String baseUrl = "https://example.org";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl);
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VARIABLES);

RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);
~~~~

以下示例配置WebClient:

~~~~ java
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode;

String baseUrl = "https://example.org";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl);
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VARIABLES);

WebClient client = WebClient.builder().uriBuilderFactory(factory).build();
~~~~

此外，您可以直接使用DefaultUriBuilderFactory。 它类似于使用UriComponentsBuilder，但它不是静态工厂方法，而是一个包含配置和首选项的实际实例，如下例所示:

~~~~ java
String baseUrl = "https://example.com";
DefaultUriBuilderFactory uriBuilderFactory = new DefaultUriBuilderFactory(baseUrl);

URI uri = uriBuilderFactory.uriString("/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
~~~~

#### 1.4.3 URI编码

UriComponentsBuilder在两个级别公开编码选项：
UriComponentsBuilder #coding（）：首先对URI模板进行预编码，然后在扩展时严格编码URI变量。
UriComponents #coding（）：扩展URI变量后对URI组件进行编码。
这两个选项都使用转义的八位字节替换非ASCII和非法字符。 但是，第一个选项还会替换出现在URI变量中的保留含义的字符。

**考虑“;”，这在路径中是合法的但具有保留意义。 第一个选项取代“;” 在URI变量中使用“％3B”但在URI模板中没有。 相比之下，第二个选项永远不会替换“;”，因为它是路径中的合法字符。**

对于大多数情况，第一个选项可能会给出预期结果，因为它将URI变量视为完全编码的不透明数据，而选项2仅在URI变量故意包含保留字符时才有用。
以下示例使用第一个选项：

~~~~ java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
            .queryParam("q", "{q}")
            .encode()
            .buildAndExpand("New York", "foo+bar")
            .toUri();
~~~~

您可以通过直接转到URI（这意味着编码）来缩短前面的示例，如以下示例所示:

~~~~ java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
            .queryParam("q", "{q}")
            .build("New York", "foo+bar")
~~~~

您可以使用完整的URI模板进一步缩短它，如下例所示：

~~~~ java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}?q={q}")
            .build("New York", "foo+bar")
~~~~

WebClient和RestTemplate通过UriBuilderFactory策略在内部扩展和编码URI模板。 两者都可以配置自定义策略。 如下例所示:

~~~~ java
String baseUrl = "https://example.com";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl)
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);

// Customize the RestTemplate..
RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);

// Customize the WebClient..
WebClient client = WebClient.builder().uriBuilderFactory(factory).build();
~~~~

DefaultUriBuilderFactory实现在内部使用UriComponentsBuilder来扩展和编码URI模板。作为工厂，它提供了一个单独的位置来配置编码方法，基于以下编码模式之一：
TEMPLATE_AND_VALUES：使用UriComponentsBuilder＃encode（）（对应于前面列表中的第一个选项）对URI模板进行预编码，并在扩展时严格编码URI变量。
VALUES_ONLY：不对URI模板进行编码，而是在将URI变量扩展到模板之前，通过UriUtils＃encodeUriUriVariables对URI变量应用严格编码。
URI_COMPONENTS：使用UriComponents＃encode（）（对应于前面列表中的第二个选项），在URI变量展开后对URI组件值进行编码。
NONE：不应用编码。
出于历史原因和向后兼容性，RestTemplate设置为EncodingMode.URI_COMPONENTS。 WebClient依赖于DefaultUriBuilderFactory中的默认值，该值已从5.0.x中的EncodingMode.URI_COMPONENTS更改为5.1中的EncodingMode.TEMPLATE_AND_VALUES。

#### 1.4.4 相对Servlet请求

您可以使用ServletUriComponentsBuilder创建相对于当前请求的URI，如以下示例所示：

~~~~ java
HttpServletRequest request = ...

// Re-uses host, scheme, port, path and query string...

ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromRequest(request)
        .replaceQueryParam("accountId", "{id}").build()
        .expand("123")
        .encode();
~~~~

您可以创建相对于上下文路径的URI，如以下示例所示:

~~~~ java
// Re-uses host, port and context path...

ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromContextPath(request)
        .path("/accounts").build()
~~~~

您可以创建相对于Servlet的URI（例如，/ main / *），如以下示例所示:

~~~~ java
// Re-uses host, port, context path, and Servlet prefix...

ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromServletMapping(request)
        .path("/accounts").build()
~~~~

**从5.1开始，ServletUriComponentsBuilder忽略来自Forwarded和X-Forwarded- *标头的信息，这些标头指定了客户端发起的地址。 考虑使用ForwardedHeaderFilter来提取和使用或丢弃此类标头。**

#### 1.4.5 控制器的链接

Spring MVC提供了一种准备控制器方法链接的机制。 例如，以下MVC控制器允许创建链接：

~~~~ java
@Controller
@RequestMapping("/hotels/{hotel}")
public class BookingController {

    @GetMapping("/bookings/{booking}")
    public ModelAndView getBooking(@PathVariable Long booking) {
        // ...
    }
}
~~~~

您可以通过按名称引用方法来准备链接，如以下示例所示:

~~~~ java
UriComponents uriComponents = MvcUriComponentsBuilder
    .fromMethodName(BookingController.class, "getBooking", 21).buildAndExpand(42);

URI uri = uriComponents.encode().toUri();
~~~~

在前面的示例中，我们提供了实际的方法参数值（在本例中，long值：21），用作路径变量并插入到URL中。 此外，我们提供值42来填充任何剩余的URI变量，例如从类型级请求映射继承的hotel变量。 如果方法有更多参数，我们可以为URL不需要的参数提供null。 通常，只有@PathVariable和@RequestParam参数与构造URL相关。
还有其他方法可以使用MvcUriComponentsBuilder。 例如，您可以使用类似于通过代理进行模拟测试的技术，以避免按名称引用控制器方法，如以下示例所示（该示例假定MvcUriComponentsBuilder.on的静态导入）：

~~~~ java
UriComponents uriComponents = MvcUriComponentsBuilder
    .fromMethodCall(on(BookingController.class).getBooking(21)).buildAndExpand(42);

URI uri = uriComponents.encode().toUri();
~~~~

**当控制器方法签名可用于与fromMethodCall创建链接时，它们的设计受到限制。 除了需要正确的参数签名之外，返回类型还存在技术限制（即，为链接构建器调用生成运行时代理），因此返回类型不能是final。 特别是，视图名称的常见String返回类型在此处不起作用。 您应该使用ModelAndView甚至是普通的Object（带有String返回值）。**

前面的示例在MvcUriComponentsBuilder中使用静态方法。 在内部，它们依赖ServletUriComponentsBuilder从当前请求的方案，主机，端口，上下文路径和servlet路径准备基本URL。 这在大多数情况下效果很好。 但是，有时，它可能是不够的。 例如，您可能在请求的上下文之外（例如准备链接的批处理）或者您可能需要插入路径前缀（例如从请求路径中删除的区域设置前缀，并且需要重新 插入链接）。
对于这种情况，您可以使用接受UriComponentsBuilder的静态fromXxx重载方法来使用基本URL。 或者，您可以使用基本URL创建MvcUriComponentsBuilder的实例，然后使用基于实例的withXxx方法。 例如，以下列表使用withMethodCall：

~~~~ java
UriComponentsBuilder base = ServletUriComponentsBuilder.fromCurrentContextPath().path("/en");
MvcUriComponentsBuilder builder = MvcUriComponentsBuilder.relativeTo(base);
builder.withMethodCall(on(BookingController.class).getBooking(21)).buildAndExpand(42);
URI uri = uriComponents.encode().toUri();
~~~~

从5.1开始，MvcUriComponentsBuilder忽略来自Forwarded和X-Forwarded- *标头的信息，这些标头指定了客户端发起的地址。 考虑使用ForwardedHeaderFilter来提取和使用或丢弃此类标头。

#### 1.4.6 视图中的链接

在Thymeleaf，FreeMarker或JSP等视图中，您可以通过引用每个请求映射的隐式或显式指定名称来构建指向带注释控制器的链接。
请考虑以下示例：

~~~~~ java
@RequestMapping("/people/{id}/addresses")
public class PersonAddressController {

    @RequestMapping("/{country}")
    public HttpEntity getAddress(@PathVariable String country) { ... }
}
~~~~~

给定前面的控制器，您可以从JSP准备链接，如下所示:

~~~~ jsp
<%@ taglib uri="http://www.springframework.org/tags" prefix="s" %>
...
<a href="${s:mvcUrl('PAC#getAddress').arg(0,'US').buildAndExpand('123')}">Get Address</a>
~~~~

前面的示例依赖于Spring标记库（即META-INF / spring.tld）中声明的mvcUrl函数，但很容易定义自己的函数或为其他模板技术准备类似函数
这是如何工作的。 在启动时，每个@RequestMapping都通过HandlerMethodMappingNamingStrategy分配一个默认名称，其默认实现使用类的大写字母和方法名称（例如，ThingController中的getThing方法变为“TC＃getThing”）。 如果存在名称冲突，则可以使用@RequestMapping（name =“..”）分配显式名称或实现自己的HandlerMethodMappingNamingStrategy。

### 1.5 异步请求

Spring MVC与Servlet 3.0异步请求处理有广泛的集成：
DeferredResult和Callable在控制器方法中返回值，并为单个异步返回值提供基本支持。
控制器可以流式传输多个值，包括SSE和原始数据。
控制器可以使用响应客户端并返回响应类型以进行响应处理。

#### 1.5.1	DeferredResult

一旦在Servlet容器中启用了异步请求处理功能，控制器方法就可以使用DeferredResult包装任何支持的控制器方法返回值，如以下示例所示:

~~~~ java
@GetMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
    DeferredResult<String> deferredResult = new DeferredResult<String>();
    // Save the deferredResult somewhere..
    return deferredResult;
}

// From some other thread...
deferredResult.setResult(data);
~~~~

控制器可以从不同的线程异步生成返回值 - 例如，响应外部事件（JMS消息），计划任务或其他事件。

#### 1.5.2. Callable

控制器可以使用java.util.concurrent.Callable包装任何受支持的返回值，如以下示例所示:

~~~~ java
@PostMapping
public Callable<String> processUpload(final MultipartFile file) {

    return new Callable<String>() {
        public String call() throws Exception {
            // ...
            return "someView";
        }
    };

}
~~~~

然后可以通过配置的TaskExecutor运行给定任务来获取返回值。

#### 1.5.3	处理

以下是Servlet异步请求处理的简要概述：
可以通过调用request.startAsync（）将ServletRequest置于异步模式。这样做的主要作用是Servlet（以及任何过滤器）可以退出，但响应保持打开状态以便稍后处理完成。
对request.startAsync（）的调用返回AsyncContext，您可以使用它来进一步控制异步处理。例如，它提供了dispatch方法，类似于Servlet API中的forward，除了它允许应用程序在Servlet容器线程上恢复请求处理。
ServletRequest提供对当前DispatcherType的访问，您可以使用它来区分处理初始请求，异步调度，转发和其他调度程序类型。
DeferredResult处理的工作方式如下：
控制器返回DeferredResult并将其保存在可以访问它的某些内存中队列或列表中。
Spring MVC调用request.startAsync（）。
同时，DispatcherServlet和所有已配置的过滤器退出请求处理线程，但响应仍保持打开状态。
应用程序从某个线程设置DeferredResult，Spring MVC将请求调度回Servlet容器。
再次调用DispatcherServlet，并使用异步生成的返回值继续处理。
可调用处理的工作原理如下：
控制器返回Callable。
Spring MVC调用request.startAsync（）并将Callable提交给TaskExecutor，以便在单独的线程中进行处理。
同时，DispatcherServlet和所有过滤器退出Servlet容器线程，但响应仍保持打开状态。
最终，Callable生成一个结果，Spring MVC将请求调度回Servlet容器以完成处理。
再次调用DispatcherServlet，并使用Callable中异步生成的返回值继续处理。
有关更多背景和上下文，您还可以阅读在Spring MVC 3.2中引入异步请求处理支持的博客文章。

###### 异常处理

使用DeferredResult时，可以选择是否使用异常调用setResult或setErrorResult。 在这两种情况下，Spring MVC都会将请求发送回Servlet容器以完成处理。 然后对其进行处理，就像控制器方法返回给定值一样，或者好像它产生了给定的异常。 然后异常通过常规异常处理机制（例如，调用@ExceptionHandler方法）。
当您使用Callable时，会出现类似的处理逻辑，主要区别在于从Callable返回结果，或者由它引发异常。

###### 侦听

HandlerInterceptor实例可以是AsyncHandlerInterceptor类型，用于在启动异步处理的初始请求（而不是postHandle和afterCompletion）上接收afterConcurrentHandlingStarted回调。
HandlerInterceptor实现还可以注册CallableProcessingInterceptor或DeferredResultProcessingInterceptor，以更深入地集成异步请求的生命周期（例如，处理超时事件）。 有关更多详细信息，请参阅AsyncHandlerInterceptor。
DeferredResult提供onTimeout（Runnable）和onCompletion（Runnable）回调。 有关更多详细信息，请参阅DeferredResult的javadoc。 Callable可以替代WebAsyncTask，它公开了超时和完成回调的其他方法。

###### 与WebFlux相比

Servlet API最初是为了通过Filter-Servlet链进行单次传递而构建的。 Servlet 3.0中添加的异步请求处理允许应用程序退出Filter-Servlet链，但保持响应打开以进行进一步处理。 Spring MVC异步支持是围绕该机制构建的。当控制器返回DeferredResult时，将退出Filter-Servlet链，并释放Servlet容器线程。稍后，当设置DeferredResult时，会进行ASYNC调度（到同一个URL），在此期间再次映射控制器，但不使用它，而是使用DeferredResult值（就像控制器返回它一样）以恢复处理。
相比之下，Spring WebFlux既不是基于Servlet API构建的，也不需要这样的异步请求处理功能，因为它在设计上是异步的。异步处理内置于所有框架合同中，并通过请求处理的所有阶段进行内在支持。
从编程模型的角度来看，Spring MVC和Spring WebFlux都支持异步和反应类型作为控制器方法中的返回值。 Spring MVC甚至支持流媒体，包括反应性背压。但是，对响应的单独写入仍然是阻塞的（并且在单独的线程上执行），这与WebFlux不同，后者依赖于非阻塞I / O，并且每次写入都不需要额外的线程。
另一个根本区别是Spring MVC不支持控制器方法参数中的异步或反应类型（例如，@ RequestBody，@ RequestPart等），也不支持异步和反应类型作为模型属性。 Spring WebFlux确实支持所有这些。

#### 1.5.4 HTTP流媒体

您可以将DeferredResult和Callable用于单个异步返回值。 如果要生成多个异步值并将其写入响应，该怎么办？ 本节介绍如何执行此操作。

###### 对象

您可以使用ResponseBodyEmitter返回值来生成对象流，其中每个对象都使用HttpMessageConverter序列化并写入响应，如以下示例所示：

~~~~~ java
@GetMapping("/events")
public ResponseBodyEmitter handle() {
    ResponseBodyEmitter emitter = new ResponseBodyEmitter();
    // Save the emitter somewhere..
    return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();
~~~~~

您还可以使用ResponseBodyEmitter作为ResponseEntity中的正文，以便自定义响应的状态和标题。
当发射器抛出IOException时（例如，如果远程客户端消失），应用程序不负责清理连接，不应调用emitter.complete或emitter.completeWithError。 相反，servlet容器自动启动AsyncListener错误通知，其中Spring MVC进行completeWithError调用。 反过来，此调用会对应用程序执行一次最终ASYNC调度，在此期间，Spring MVC将调用已配置的异常解析程序并完成请求。

##### SSE

SseEmitter（ResponseBodyEmitter的子类）为Server-Sent Events提供支持，其中从服务器发送的事件根据W3C SSE规范进行格式化。 要从控制器生成SSE流，请返回SseEmitter，如以下示例所示：

~~~~ java
@GetMapping(path="/events", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter handle() {
    SseEmitter emitter = new SseEmitter();
    // Save the emitter somewhere..
    return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();
~~~~

虽然SSE是流式传输到浏览器的主要选项，但请注意Internet Explorer不支持Server-Sent Events。 考虑将Spring的WebSocket消息传递与针对各种浏览器的SockJS后备传输（包括SSE）一起使用。
有关异常处理的说明，另请参见上一节。

###### 原始数据

有时，绕过消息转换并直接流到响应OutputStream（例如，用于文件下载）是有用的。 您可以使用StreamingResponseBody返回值类型来执行此操作，如以下示例所示：

~~~~ java
@GetMapping("/download")
public StreamingResponseBody handle() {
    return new StreamingResponseBody() {
        @Override
        public void writeTo(OutputStream outputStream) throws IOException {
            // write...
        }
    };
}
~~~~

您可以使用StreamingResponseBody作为ResponseEntity中的主体来自定义响应的状态和标头。

#### 1.5.5 Reactive类型

Spring MVC支持在控制器中使用反应式客户端库（也可以阅读WebFlux部分中的Reactive Libraries）。 这包括来自spring-webflux的WebClient和其他，例如Spring Data反应数据存储库。 在这种情况下，能够从控制器方法返回反应类型是方便的。
反应性返回值的处理如下：
单值承诺适用于，类似于使用DeferredResult。 例子包括Mono（Reactor）或Single（RxJava）。
具有流媒体类型（例如应用程序/流+ json或文本/事件流）的多值流适用于类似于使用ResponseBodyEmitter或SseEmitter。 例子包括Flux（Reactor）或Observable（RxJava）。 应用程序还可以返回Flux <ServerSentEvent>或Observable <ServerSentEvent>。
与使用DeferredResult <List <？>>类似，适用于具有任何其他媒体类型（例如application / json）的多值流。

**Spring MVC通过Spring-core的ReactiveAdapterRegistry支持Reactor和RxJava，它允许它适应多个反应库。**

对于流式传输到响应，支持反应式反压，但对响应的写入仍然是阻塞的，并通过配置的TaskExecutor在单独的线程上执行，以避免阻塞上游源（例如从WebClient返回的Flux）。 默认情况下，SimpleAsyncTaskExecutor用于阻塞写入，但在加载时不适用。 如果计划使用响应类型进行流式处理，则应使用MVC配置来配置任务执行程序。

#### 1.5.6	断开

当远程客户端消失时，Servlet API不提供任何通知。 因此，在流式传输到响应时，无论是通过SseEmitter还是反应类型，定期发送数据都很重要，因为如果客户端断开连接，写入将失败。 发送可以采用空（仅注释）SSE事件或另一方必须解释为心跳并忽略的任何其他数据的形式。
或者，考虑使用具有内置心跳机制的Web消息传递解决方案（例如基于WebSocket的STOMP或具有SockJS的WebSocket）。

#### 1.5.7. 配置

必须在Servlet容器级别启用异步请求处理功能。 MVC配置还公开了几个异步请求选项。

###### Servlet容器

Filter和Servlet声明具有asyncSupported标志，需要将其设置为true以启用异步请求处理。此外，应声明Filter映射以处理ASYNC javax.servlet.DispatchType。
在Java配置中，当您使用AbstractAnnotationConfigDispatcherServletInitializer初始化Servlet容器时，这将自动完成。
在web.xml配置中，您可以将<async-supported> true </ async-supported>添加到DispatcherServlet和Filter声明，并添加<dispatcher> ASYNC </ dispatcher>以过滤映射。

###### Spring MVC

MVC配置公开以下与异步请求处理相关的选项：
Java配置：在WebMvcConfigurer上使用configureAsyncSupport回调。
XML命名空间：使用<mvc：annotation-driven>下的<async-support>元素。
您可以配置以下内容：
异步请求的默认超时值（如果未设置）取决于基础Servlet容器（例如，Tomcat上的10秒）。
AsyncTaskExecutor用于在使用Reactive Types进行流式处理时阻止写入，以及用于执行从控制器方法返回的Callable实例。如果您使用反应类型进行流式传输或者具有返回Callable的控制器方法，我们强烈建议您配置此属性，因为默认情况下，它是SimpleAsyncTaskExecutor。
DeferredResultProcessingInterceptor实现和CallableProcessingInterceptor实现。
请注意，您还可以在DeferredResult，ResponseBodyEmitter和SseEmitter上设置默认超时值。对于Callable，您可以使用WebAsyncTask来提供超时值。

### 1.6. CORS

Spring MVC允许您处理CORS（跨源资源共享）。 本节介绍如何执行此操作。

#### 1.6.1 介绍

出于安全原因，浏览器禁止对当前源外的资源进行AJAX调用。 例如，您可以将您的银行帐户放在一个标签页中，将evil.com放在另一个标签页中。 来自evil.com的脚本不应该使用您的凭据向您的银行API发出AJAX请求 - 例如从您的帐户中提取资金！
跨源资源共享（CORS）是大多数浏览器实现的W3C规范，允许您指定授权的跨域请求类型，而不是使用基于IFRAME或JSONP的安全性较低且功能较弱的变通方法。

#### 1.6.2	处理

CORS规范区分了预检，简单和实际请求。要了解CORS的工作原理，您可以阅读本文以及其他许多内容，或者查看规范以获取更多详细信息。
Spring MVC HandlerMapping实现为CORS提供内置支持。成功将请求映射到处理程序后，HandlerMapping实现检查给定请求和处理程序的CORS配置并采取进一步操作。直接处理预检请求，同时拦截，验证简单和实际的CORS请求，并设置所需的CORS响应头。
为了启用跨源请求（即，存在Origin头并且与请求的主机不同），您需要具有一些显式声明的CORS配置。如果未找到匹配的CORS配置，则拒绝预检请求。没有CORS头添加到简单和实际CORS请求的响应中，因此浏览器拒绝它们。
可以使用基于URL模式的CorsConfiguration映射单独配置每个HandlerMapping。在大多数情况下，应用程序使用MVC Java配置或XML命名空间来声明此类映射，这会导致将单个全局映射传递给所有HandlerMappping实例。
您可以将HandlerMapping级别的全局CORS配置与更细粒度的处理程序级CORS配置相结合。例如，带注解的控制器可以使用类或方法级别的@CrossOrigin注释（其他处理程序可以实现CorsConfigurationSource）。
组合全局和本地配置的规则通常是附加的 - 例如，所有全局和所有本地源。对于只能接受单个值的属性（例如allowCredentials和maxAge），本地会覆盖全局值。有关详细信息，请参阅CorsConfiguration＃combine（CorsConfiguration）。

要从源中了解更多信息或进行高级自定义，请检查后面的代码：
CorsConfiguration
CorsProcessor，DefaultCorsProcessor
AbstractHandlerMapping

#### 1.6.3. `@CrossOrigin`

@CrossOrigin注解在带注解的控制器方法上启用跨源请求，如以下示例所示：

~~~~ java
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
~~~~

默认情况下，@ CrossOrigin允许：
所有起源。
所有标题。
控制器方法映射到的所有HTTP方法。
默认情况下不启用allowedCredentials，因为它建立了一个信任级别，用于公开敏感的用户特定信息（例如cookie和CSRF令牌），并且只应在适当的地方使用。
maxAge设置为30分钟。
@CrossOrigin在类级别也受支持，并且由所有方法继承，如以下示例所示：

~~~~ java
@CrossOrigin(origins = "https://domain2.com", maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
~~~~

您可以在类级别和方法级别使用@CrossOrigin，如以下示例所示：

~~~~java
@CrossOrigin(maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin("https://domain2.com")
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
~~~~

#### 1.6.4 全局配置

除了细粒度的控制器方法级别配置之外，您可能还想要定义一些全局CORS配置。 您可以在任何HandlerMapping上单独设置基于URL的CorsConfiguration映射。 但是，大多数应用程序使用MVC Java配置或MVC XNM命名空间来执行此操作。
默认情况下，全局配置启用以下内容：
所有起源。
所有标题
GET，HEAD和POST方法。
默认情况下不启用allowedCredentials，因为它建立了一个信任级别，用于公开敏感的用户特定信息（例如cookie和CSRF令牌），并且只应在适当的地方使用。
maxAge设置为30分钟。

###### Java配置

要在MVC Java配置中启用CORS，可以使用CorsRegistry回调，如以下示例所示：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/api/**")
            .allowedOrigins("https://domain2.com")
            .allowedMethods("PUT", "DELETE")
            .allowedHeaders("header1", "header2", "header3")
            .exposedHeaders("header1", "header2")
            .allowCredentials(true).maxAge(3600);

        // Add more mappings...
    }
}
~~~~

###### XML配置

要在XML命名空间中启用CORS，可以使用<mvc：cors>元素，如以下示例所示：

~~~~ xml
<mvc:cors>

    <mvc:mapping path="/api/**"
        allowed-origins="https://domain1.com, https://domain2.com"
        allowed-methods="GET, PUT"
        allowed-headers="header1, header2, header3"
        exposed-headers="header1, header2" allow-credentials="true"
        max-age="123" />

    <mvc:mapping path="/resources/**"
        allowed-origins="https://domain1.com" />

</mvc:cors>
~~~~

#### 1.6.5 CORS过滤器

您可以通过内置的CorsFilter应用CORS支持。

**如果您尝试将CorsFilter与Spring Security一起使用，请记住Spring Security内置了对CORS的支持。**

要配置过滤器，请将CorsConfigurationSource传递给其构造函数，如以下示例所示：

~~~~~ java
CorsConfiguration config = new CorsConfiguration();

// Possibly...
// config.applyPermitDefaultValues()

config.setAllowCredentials(true);
config.addAllowedOrigin("https://domain1.com");
config.addAllowedHeader("*");
config.addAllowedMethod("*");

UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
source.registerCorsConfiguration("/**", config);

CorsFilter filter = new CorsFilter(source);
~~~~~

### 1.7 网络安全

Spring Security项目支持保护Web应用程序免受恶意攻击。 请参阅Spring Security参考文档，包括：
Spring MVC Security
Spring MVC测试支持
CSRF保护
安全响应标头
HDIV是另一个与Spring MVC集成的Web安全框架。

### 1.8 HTTP缓存

HTTP缓存可以显着提高Web应用程序的性能。 HTTP缓存围绕Cache-Control响应头，随后是条件请求头（例如Last-Modified和ETag）。 Cache-Control建议私有（例如，浏览器）和公共（例如，代理）缓存如何缓存和重用响应。 如果内容未更改，则ETag标头用于生成条件请求，该条件请求可能导致304（NOT_MODIFIED）没有正文。 ETag可以被视为Last-Modified标头的更复杂的继承者。
本节介绍Spring Web MVC中可用的与HTTP缓存相关的选项。

#### 1.8.1	CacheControl

CacheControl支持配置与Cache-Control标头相关的设置，并在许多地方被接受为参数：
WebContentInterceptor
WebContentGenerator
控制器
静态资源
虽然RFC 7234描述了Cache-Control响应头的所有可能的指令，但CacheControl类型采用面向用例的方法，该方法侧重于常见场景：

~~~~ java
// Cache for an hour - "Cache-Control: max-age=3600"
CacheControl ccCacheOneHour = CacheControl.maxAge(1, TimeUnit.HOURS);

// Prevent caching - "Cache-Control: no-store"
CacheControl ccNoStore = CacheControl.noStore();

// Cache for ten days in public and private caches,
// public caches should not transform the response
// "Cache-Control: max-age=864000, public, no-transform"
CacheControl ccCustom = CacheControl.maxAge(10, TimeUnit.DAYS).noTransform().cachePublic();
~~~~

WebContentGenerator还接受一个更简单的cachePeriod属性（以秒为单位定义），其工作方式如下：
A -1值不产生缓存控制响应头。
指令：A 0值通过使用“无存储缓存控制”防止缓存。
n> 0值通过使用'Cache-Control：max-age = n'指令将给定响应缓存n秒。

#### 1.8.2	控制器

控制器可以添加对HTTP缓存的显式支持。 我们建议这样做，因为资源的lastModified或ETag值需要先计算才能与条件请求标头进行比较。 控制器可以向ResponseEntity添加ETag标头和Cache-Control设置，如以下示例所示：

~~~~ java
@GetMapping("/book/{id}")
public ResponseEntity<Book> showBook(@PathVariable Long id) {

    Book book = findBook(id);
    String version = book.getVersion();

    return ResponseEntity
            .ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.DAYS))
            .eTag(version) // lastModified is also available
            .body(book);
}
~~~~

如果与条件请求标头的比较表明内容未更改，则前面的示例将发送带有空主体的304（NOT_MODIFIED）响应。 否则，ETag和Cache-Control标头将添加到响应中。
您还可以对控制器中的条件请求标头进行检查，如以下示例所示：

~~~~ java
@RequestMapping
public String myHandleMethod(WebRequest webRequest, Model model) {

    long eTag = ... 

    if (request.checkNotModified(eTag)) {
        return null; 
    }

    model.addAttribute(...); 
    return "myViewName";
}
~~~~

有三种变体可用于检查针对eTag值，lastModified值或两者的条件请求。 对于条件GET和HEAD请求，您可以将响应设置为304（NOT_MODIFIED）。 对于条件POST，PUT和DELETE，您可以将响应设置为409（PRECONDITION_FAILED），以防止并发修改。

#### 1.8.3 静态资源

您应该使用Cache-Control和条件响应头来提供静态资源，以获得最佳性能。 请参阅有关配置静态资源的部分。

#### 1.8.4 ETag过滤器

您可以使用ShallowEtagHeaderFilter添加从响应内容计算的“浅”eTag值，从而节省带宽但不节省CPU时间。 见浅ETag。

### 1.9 视图技术

在Spring MVC中使用视图技术是可插拔的，无论您决定使用Thymeleaf，Groovy标记模板，JSP还是其他技术，主要是配置更改的问题。 本章介绍了与Spring MVC集成的视图技术。 我们假设您已经熟悉View Resolution。

#### 1.9.1. Thymeleaf

Thymeleaf是一个现代服务器端Java模板引擎，强调自然HTML模板，可以通过双击在浏览器中预览，这对于UI模板的独立工作（例如，由设计师）非常有用，而无需 运行服务器。 如果您想要替换JSP，Thymeleaf提供了一组最广泛的功能，使这种过渡更容易。 Thymeleaf积极开发和维护。 有关更完整的介绍，请参阅Thymeleaf项目主页。
Thymeleaf与Spring MVC的集成由Thymeleaf项目管理。 配置涉及一些bean声明，例如ServletContextTemplateResolver，SpringTemplateEngine和ThymeleafViewResolver。 有关详细信息，请参阅Thymeleaf + Spring。

#### 1.9.2. FreeMarker

Apache FreeMarker是一个模板引擎，用于生成从HTML到电子邮件和其他人的任何类型的文本输出。 Spring Framework具有内置的集成，可以将Spring MVC与FreeMarker模板结合使用。

###### 视图配置

以下示例显示如何将FreeMarker配置为视图技术:

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.freemarker();
    }

    // Configure FreeMarker...

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath("/WEB-INF/freemarker");
        return configurer;
    }
}
```

以下示例显示如何在XML中配置相同的内容：

~~~~ xml
<mvc:annotation-driven/>

<mvc:view-resolvers>
    <mvc:freemarker/>
</mvc:view-resolvers>

<!-- Configure FreeMarker... -->
<mvc:freemarker-configurer>
    <mvc:template-loader-path location="/WEB-INF/freemarker"/>
</mvc:freemarker-configurer>
~~~~

或者，您也可以声明FreeMarkerConfigurer bean以完全控制所有属性，如以下示例所示：

~~~~~ xml
<bean id="freemarkerConfig" class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
    <property name="templateLoaderPath" value="/WEB-INF/freemarker/"/>
</bean>
~~~~~

您的模板需要存储在前面示例中显示的FreeMarkerConfigurer指定的目录中。 根据前面的配置，如果控制器返回欢迎的视图名称，解析器将查找/WEB-INF/freemarker/welcome.ftl模板。

###### FreeMarker配置

您可以通过在FreeMarkerConfigurer bean上设置适当的bean属性，将FreeMarker'Settings'和'SharedVariables'直接传递给FreeMarker配置对象（由Spring管理）。 freemarkerSettings属性需要java.util.Properties对象，而freemarkerVariables属性需要java.util.Map。 以下示例显示了如何执行此操作：

~~~~ xml
<bean id="freemarkerConfig" class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
    <property name="templateLoaderPath" value="/WEB-INF/freemarker/"/>
    <property name="freemarkerVariables">
        <map>
            <entry key="xml_escape" value-ref="fmXmlEscape"/>
        </map>
    </property>
</bean>
<bean id="fmXmlEscape" class="freemarker.template.utility.XmlEscape"/>
~~~~

有关适用于Configuration对象的设置和变量的详细信息，请参阅FreeMarker文档。

###### 表单处理

Spring提供了一个用于JSP的标记库，其中包含一个<spring：bind />元素。 此元素主要使表单显示来自表单支持对象的值，并显示来自Web或业务层中的Validator的验证失败的结果。 Spring还支持FreeMarker中的相同功能，还有用于生成表单输入元素的额外便利宏。

绑定宏
在spring-webmvc.jar文件中为两种语言维护一组标准宏，因此它们始终可用于适当配置的应用程序。
Spring库中定义的一些宏被认为是内部（私有），但宏定义中不存在这样的范围，使得所有宏对调用代码和用户模板都可见。 以下部分仅关注您需要在模板中直接调用的宏。 如果您希望直接查看宏代码，该文件名为spring.ftl，位于org.springframework.web.servlet.view.freemarker包中。
简单的绑定
在作为Spring MVC控制器的表单视图的HTML表单（vm或ftl模板）中，您可以使用与下一个示例类似的代码绑定到字段值，并以与JSP类似的方式显示每个输入字段的错误消息 当量。 以下示例显示了之前配置的personForm视图：

~~~~ html
<!-- freemarker macros have to be imported into a namespace. We strongly
recommend sticking to 'spring' -->
<#import "/spring.ftl" as spring/>
<html>
    ...
    <form action="" method="POST">
        Name:
        <@spring.bind "myModelObject.name"/>
        <input type="text"
            name="${spring.status.expression}"
            value="${spring.status.value?html}"/><br>
        <#list spring.status.errorMessages as error> <b>${error}</b> <br> </#list>
        <br>
        ...
        <input type="submit" value="submit"/>
    </form>
    ...
</html>
~~~~

<@ spring.bind>需要一个'path'参数，它包含命令对象的名称（它是'command'，除非你在FormController属性中更改它），后跟一个句点和字段的名称。 要绑定的命令对象。 您还可以使用嵌套字段，例如command.address.street。 bind宏假定由web.xml中的ServletContext参数defaultHtmlEscape指定的默认HTML转义行为。
名为<@ spring.bindEscaped>的宏的可选形式接受第二个参数，并显式指定是否应在状态错误消息或值中使用HTML转义。 您可以根据需要将其设置为true或false。 其他表单处理宏简化了HTML转义的使用，您应该尽可能使用这些宏。 它们将在下一节中介绍。

输入宏
两种语言的附加便利宏简化了绑定和表单生成（包括验证错误显示）。 永远不必使用这些宏来生成表单输入字段，您可以将它们与简单的HTML混合或匹配，或直接调用我们之前突出显示的spring绑定宏。
下表中的可用宏显示了FTL定义和每个参数列表：

| 宏                                                           | FTL定义                                                      |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `message` (根据代码参数从资源包中输出字符串)                 | <@spring.message code/>                                      |
| `messageText` (根据代码参数从资源包中输出一个字符串，然后返回默认参数的值) | <@spring.messageText code, text/>                            |
| `url` (使用应用程序的上下文根作为相对URL的前缀)              | <@spring.url relativeUrl/>                                   |
| `formInput` (于收集用户输入的标准输入字段)                   | <@spring.formInput path, attributes, fieldType/>             |
| `formHiddenInput` (用于提交非用户输入的隐藏输入字段)         | <@spring.formHiddenInput path, attributes/>                  |
| `formPasswordInput` (用于收集密码的标准输入字段。 请注意，此类型的字段中不会填充任何值) | <@spring.formPasswordInput path, attributes/>                |
| `formTextarea` (用于收集长，自由格式文本输入的大文本字段)    | <@spring.formTextarea path, attributes/>                     |
| `formSingleSelect` (下拉选项框，可以选择一个必需的值)        | <@spring.formSingleSelect path, options, attributes/>        |
| `formMultiSelect` (一个选项列表框，允许用户选择0或更多值)    | <@spring.formMultiSelect path, options, attributes/>         |
| `formRadioButtons` (一组单选按钮，可以从可用选项中进行单个选择) | <@spring.formRadioButtons path, options separator, attributes/> |
| `formCheckboxes` (一组复选框，允许选择0或更多值)             | <@spring.formCheckboxes path, options, separator, attributes/> |
| `formCheckbox` (一个复选框)                                  | <@spring.formCheckbox path, attributes/>                     |
| `showErrors` (简化绑定字段的验证错误显示)                    | <@spring.showErrors separator, classOrStyle/>                |

在FTL（FreeMarker）中，实际上不需要formHiddenInput和formPasswordInput，因为您可以使用普通的formInput宏，指定hidden或password作为fieldType参数的值。
任何上述宏的参数都具有一致的含义：
path：要绑定的字段的名称（即“command.name”）
options：可在输入字段中选择的所有可用值的映射。映射的键表示从表单返回并绑定到命令对象的值。针对键存储的映射对象是表单上显示给用户的标签，可能与表单发回的相应值不同。通常，控制器将这种映射作为参考数据提供。您可以使用任何Map实现，具体取决于所需的行为。对于严格排序的映射，您可以将SortedMap（例如TreeMap）与合适的Comparator一起使用，对于应按插入顺序返回值的任意Maps，使用来自commons-collections的LinkedHashMap或LinkedMap。
separator：多个选项可用作谨慎元素（单选按钮或复选框），用于分隔列表中每个元素的字符序列（例如<br>）。
attributes：HTML标记本身中包含的任意标记或文本的附加字符串。该字符串由宏按字面回显。例如，在textarea字段中，您可以提供属性（例如'rows =“5”cols =“60”'），或者您可以传递样式信息，例如'style =“border：1px solid silver”'。
classOrStyle：对于showErrors宏，包含每个错误的span元素使用的CSS类的名称。如果未提供任何信息（或值为空），则错误将包含在<b> </ b>标记中。
以下部分概述了宏的示例（一些在FTL中，一些在VTL中）。如果两种语言之间存在使用差异，则会在说明中对其进行说明。
输入字段
formInput宏采用path参数（command.name）和附加属性参数（在下一个示例中为空）。宏以及所有其他表单生成宏在path参数上执行隐式Spring绑定。绑定在发生新绑定之前一直有效，因此showErrors宏不需要再次传递path参数 - 它在最后创建绑定的字段上运行
showErrors宏接受一个separator参数（用于分隔给定字段上的多个错误的字符），还接受第二个参数 - 这次是类名或样式属性。请注意，FreeMarker可以为attributes参数指定默认值。以下示例显示如何使用formInput和showWErrors宏：

~~~~ html
<@spring.formInput "command.name"/>
<@spring.showErrors "<br>"/>
~~~~

下一个示例显示表单片段的输出，生成名称字段并在提交表单后在字段中没有值时显示验证错误。 验证通过Spring的验证框架进行。
生成的HTML类似于以下示例：

~~~~
Name:
<input type="text" name="name" value="">
<br>
    <b>required</b>
<br>
<br>
~~~~

formTextarea宏的工作方式与formInput宏的工作方式相同，并接受相同的参数列表。 通常，第二个参数（attributes）用于传递textarea的样式信息或rows和cols属性。
选择字段
您可以使用四个选择字段宏在HTML表单中生成常用UI值选择输入：
formSingleSelect
formMultiSelect
formRadioButtons
formCheckboxes
四个宏中的每一个都接受一个选项映射，其中包含表单字段的值和与该值对应的标签。 值和标签可以相同。
下一个示例是FTL中的单选按钮。 表单支持对象为此字段指定默认值“伦敦”，因此不需要进行验证。 渲染表单时，可以选择的整个城市列表作为参考数据在名为“cityMap”的模型中提供。 以下清单显示了示例：

~~~~ html
...
Town:
<@spring.formRadioButtons "command.address.town", cityMap, ""/><br><br>
~~~~

前面的列表呈现一行单选按钮，一个用于cityMap中的每个值，并使用“”的分隔符。 没有提供其他属性（缺少宏的最后一个参数）。 cityMap对映射中的每个键值对使用相同的String。 映射的键是表单实际提交为POSTed请求参数的键。 地图值是用户看到的标签。 在前面的示例中，给定了三个众所周知的城市的列表以及表单支持对象中的默认值，HTML类似于以下内容：

~~~~ html
Town:
<input type="radio" name="address.town" value="London">London</input>
<input type="radio" name="address.town" value="Paris" checked="checked">Paris</input>
<input type="radio" name="address.town" value="New York">New York</input>
~~~~

如果您的应用程序希望通过内部代码（例如）处理城市，则可以使用合适的密钥创建代码映射，如以下示例所示：

~~~~~ java
protected Map<String, String> referenceData(HttpServletRequest request) throws Exception {
    Map<String, String> cityMap = new LinkedHashMap<>();
    cityMap.put("LDN", "London");
    cityMap.put("PRS", "Paris");
    cityMap.put("NYC", "New York");

    Map<String, String> model = new HashMap<>();
    model.put("cityMap", cityMap);
    return model;
}
~~~~~

代码现在生成输出，其中无线电值是相关代码，但用户仍然可以看到更加用户友好的城市名称，如下所示：

~~~~ html
Town:
<input type="radio" name="address.town" value="LDN">London</input>
<input type="radio" name="address.town" value="PRS" checked="checked">Paris</input>
<input type="radio" name="address.town" value="NYC">New York</input>
~~~~

###### HTML Escaping

前面描述的表单宏的默认用法导致符合HTML 4.01的HTML元素，并且使用在web.xml文件中定义的HTML转义的默认值，如Spring的绑定支持所使用的那样。 要使元素符合XHTML或覆盖默认的HTML转义值，您可以在模板中指定两个变量（或在模型中指定模板可见的变量）。 在模板中指定它们的优点是，它们可以在模板处理中稍后更改为不同的值，以便为表单中的不同字段提供不同的行为。
要切换为标记的XHTML合规性，请为名为xhtmlCompliant的模型或上下文变量指定值true，如以下示例所示：

~~~~ properties
<#-- for FreeMarker -->
<#assign xhtmlCompliant = true>
~~~~

处理完该指令后，Spring宏生成的任何元素现在都符合XHTML标准。
以类似的方式，您可以指定每个字段的HTML转义，如以下示例所示：

~~~~~
<#-- until this point, default HTML escaping is used -->

<#assign htmlEscape = true>
<#-- next field will use HTML escaping -->
<@spring.formInput "command.name"/>

<#assign htmlEscape = false in spring>
<#-- all future fields will be bound with HTML escaping off -->
~~~~~

#### 1.9.3. Groovy Markup

Groovy标记模板引擎主要用于生成类似XML的标记（XML，XHTML，HTML5等），但您可以使用它来生成任何基于文本的内容。 Spring Framework有一个内置的集成，可以将Spring MVC与Groovy Markup结合使用。

**Groovy标记模板引擎需要Groovy 2.3.1+。**

###### 配置

以下示例显示如何配置Groovy标记模板引擎：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.groovy();
    }

    // Configure the Groovy Markup Template Engine...

    @Bean
    public GroovyMarkupConfigurer groovyMarkupConfigurer() {
        GroovyMarkupConfigurer configurer = new GroovyMarkupConfigurer();
        configurer.setResourceLoaderPath("/WEB-INF/");
        return configurer;
    }
}
~~~~

以下示例显示如何在XML中配置相同的内容：

~~~~ xml
<mvc:annotation-driven/>

<mvc:view-resolvers>
    <mvc:groovy/>
</mvc:view-resolvers>

<!-- Configure the Groovy Markup Template Engine... -->
<mvc:groovy-configurer resource-loader-path="/WEB-INF/"/>
~~~~

###### 例子

与传统的模板引擎不同，Groovy Markup依赖于使用构建器语法的DSL。 以下示例显示了HTML页面的示例模板：

~~~~ html
yieldUnescaped '<!DOCTYPE html>'
html(lang:'en') {
    head {
        meta('http-equiv':'"Content-Type" content="text/html; charset=utf-8"')
        title('My page')
    }
    body {
        p('This is an example of HTML contents')
    }
}
~~~~

#### 1.9.4 脚本视图

Spring Framework有一个内置的集成，可以将Spring MVC与任何可以在JSR-223 Java脚本引擎之上运行的模板库一起使用。 我们在不同的脚本引擎上测试了以下模板库：

| Scripting Library                                            | Scripting Engine                                      |
| :----------------------------------------------------------- | :---------------------------------------------------- |
| [Handlebars](https://handlebarsjs.com/)                      | [Nashorn](https://openjdk.java.net/projects/nashorn/) |
| [Mustache](https://mustache.github.io/)                      | [Nashorn](https://openjdk.java.net/projects/nashorn/) |
| [React](https://facebook.github.io/react/)                   | [Nashorn](https://openjdk.java.net/projects/nashorn/) |
| [EJS](https://www.embeddedjs.com/)                           | [Nashorn](https://openjdk.java.net/projects/nashorn/) |
| [ERB](https://www.stuartellis.name/articles/erb/)            | [JRuby](https://www.jruby.org/)                       |
| [String templates](https://docs.python.org/2/library/string.html#template-strings) | [Jython](https://www.jython.org/)                     |
| [Kotlin Script templating](https://github.com/sdeleuze/kotlin-script-templating) | [Kotlin](https://kotlinlang.org/)                     |

**集成任何其他脚本引擎的基本规则是它必须实现ScriptEngine和Invocable接口。**

###### 要求

您需要在类路径上安装脚本引擎，其详细信息因脚本引擎而异：
Nashorn JavaScript引擎随Java 8+一起提供。 强烈建议使用最新的更新版本。
应该添加JRuby作为Ruby支持的依赖项。
应该添加Jython作为Python支持的依赖项。
org.jetbrains.kotlin：kotlin-script-util依赖项和包含org.jetbrains.kotlin.script.jsr223.KotlinJsr223JvmLocalScriptEngineFactory行的META-INF / services / javax.script.ScriptEngineFactory文件应添加到Kotlin脚本支持中。 有关详细信息，请参阅此示例。
您需要拥有脚本模板库。 对Javascript这样做的一种方法是通过WebJars。

###### 脚本模板

您可以声明ScriptTemplateConfigurer bean以指定要使用的脚本引擎，要加载的脚本文件，要调用以呈现模板的函数，等等。 以下示例使用Mustache模板和Nashorn JavaScript引擎：

~~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.scriptTemplate();
    }

    @Bean
    public ScriptTemplateConfigurer configurer() {
        ScriptTemplateConfigurer configurer = new ScriptTemplateConfigurer();
        configurer.setEngineName("nashorn");
        configurer.setScripts("mustache.js");
        configurer.setRenderObject("Mustache");
        configurer.setRenderFunction("render");
        return configurer;
    }
}
~~~~~

以下示例显示了XML中的相同排列：

~~~~ xml
<mvc:annotation-driven/>

<mvc:view-resolvers>
    <mvc:script-template/>
</mvc:view-resolvers>

<mvc:script-template-configurer engine-name="nashorn" render-object="Mustache" render-function="render">
    <mvc:script location="mustache.js"/>
</mvc:script-template-configurer>
~~~~

对于Java和XML配置，控制器看起来没有什么不同，如以下示例所示：

~~~~ java
@Controller
public class SampleController {

    @GetMapping("/sample")
    public String test(Model model) {
        model.addObject("title", "Sample title");
        model.addObject("body", "Sample body");
        return "template";
    }
}
~~~~

以下示例显示了Mustache模板：

~~~~~ html
<html>
    <head>
        <title>{{title}}</title>
    </head>
    <body>
        <p>{{body}}</p>
    </body>
</html>
~~~~~

使用以下参数调用render函数：
字符串模板：模板内容
map模型：视图模型
RenderingContext renderingContext：RenderingContext，用于访问应用程序上下文，区域设置，模板加载器和URL（自5.0起）
Mustache.render（）本身与此签名兼容，因此您可以直接调用它。
如果您的模板技术需要一些自定义，您可以提供实现自定义渲染功能的脚本。 例如，Handlerbars需要在使用之前编译模板，并且需要使用polyfill来模拟服务器端脚本引擎中不可用的某些浏览器工具。
以下示例显示了如何执行此操作：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.scriptTemplate();
    }

    @Bean
    public ScriptTemplateConfigurer configurer() {
        ScriptTemplateConfigurer configurer = new ScriptTemplateConfigurer();
        configurer.setEngineName("nashorn");
        configurer.setScripts("polyfill.js", "handlebars.js", "render.js");
        configurer.setRenderFunction("render");
        configurer.setSharedEngine(false);
        return configurer;
    }
}
~~~~

**当您使用非线程安全脚本引擎以及不是为并发而设计的模板库（例如在Nashorn上运行的Handlebars或React）时，需要将sharedEngine属性设置为false。 在这种情况下，由于此错误，需要Java 8u60或更高版本。**

polyfill.js仅定义Handlebars正常运行所需的窗口对象，如下所示：

~~~~ javascript
var window = {};
~~~~

这个基本的render.js实现在使用之前编译模板。 生产就绪的实现还应该存储任何重用的缓存模板或预编译的模板。 您可以在脚本端执行此操作（并处理您需要的任何自定义 - 例如，管理模板引擎配置）。 以下示例显示了如何执行此操作：

~~~~ javascript
function render(template, model) {
    var compiledTemplate = Handlebars.compile(template);
    return compiledTemplate(model);
}
~~~~

有关更多配置示例，请查看Spring Framework单元测试，Java和资源。

#### 1.9.5. JSP and JSTL

Spring Framework有一个内置的集成，用于将Spring MVC与JSP和JSTL一起使用。

###### 查看解析器

使用JSP进行开发时，可以声明InternalResourceViewResolver或ResourceBundleViewResolver bean。
ResourceBundleViewResolver依赖于属性文件来定义映射到类和URL的视图名称。 使用ResourceBundleViewResolver，您可以通过仅使用一个解析器来混合不同类型的视图，如以下示例所示：

~~~~ xml
<!-- the ResourceBundleViewResolver -->
<bean id="viewResolver" class="org.springframework.web.servlet.view.ResourceBundleViewResolver">
    <property name="basename" value="views"/>
</bean>

# And a sample properties file is used (views.properties in WEB-INF/classes):
welcome.(class)=org.springframework.web.servlet.view.JstlView
welcome.url=/WEB-INF/jsp/welcome.jsp

productList.(class)=org.springframework.web.servlet.view.JstlView
productList.url=/WEB-INF/jsp/productlist.jsp
~~~~

InternalResourceViewResolver也可用于JSP。 作为最佳实践，我们强烈建议您将JSP文件放在“WEB-INF”目录下的目录中，以便客户端无法直接访问。

~~~~ xml
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
~~~~

###### JSP与JSTL

使用JSP标准标记库（JSTL）时，必须使用特殊的视图类JstlView，因为在I18N功能之类的东西可以工作之前，JSTL需要做一些准备。

###### Spring的JSP标记库

Spring提供了请求参数与命令对象的数据绑定，如前面章节所述。为了便于JSP页面的开发与这些数据绑定功能的结合，Spring提供了一些标签，使事情变得更加容易。所有Spring标记都具有HTML转义功能，以启用或禁用字符转义。
spring.tld标记库描述符（TLD）包含在spring-webmvc.jar中。有关各个标签的综合参考，请浏览API参考或查看标签库说明。

###### Spring的表单标签库

从2.0版开始，Spring提供了一组全面的数据绑定感知标记，用于在使用JSP和Spring Web MVC时处理表单元素。每个标记都支持其相应HTML标记对应的属性集，使标记熟悉且直观易用。标记生成的HTML符合HTML 4.01 / XHTML 1.0。
与其他表单/输入标记库不同，Spring的表单标记库与Spring Web MVC集成，使标记可以访问控制器处理的命令对象和引用数据。正如我们在以下示例中所示，表单标记使JSP更易于开发，读取和维护。
我们浏览表单标签，并查看每个标签的使用方式示例。我们已经包含生成的HTML代码段，其中某些代码需要进一步评论。

###### 组态

表单标记库捆绑在spring-webmvc.jar中。库描述符称为spring-form.tld。
要使用此库中的标记，请将以下指令添加到JSP页面的顶部：

~~~~ html
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
~~~~

###### 表单标签

此标记呈现HTML“表单”元素，并公开内部标记的绑定路径以进行绑定。 它将命令对象放在PageContext中，以便内部标记可以访问命令对象。 此库中的所有其他标记都是表单标记的嵌套标记。
假设我们有一个名为User的域对象。 它是一个JavaBean，具有firstName和lastName等属性。 我们可以将它用作表单控制器的表单支持对象，它返回form.jsp。 以下示例显示了form.jsp的外观：

~~~~ html
<form:form>
    <table>
        <tr>
            <td>First Name:</td>
            <td><form:input path="firstName"/></td>
        </tr>
        <tr>
            <td>Last Name:</td>
            <td><form:input path="lastName"/></td>
        </tr>
        <tr>
            <td colspan="2">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form:form>
~~~~

firstPage和lastName值是从页面控制器放置在PageContext中的命令对象中检索的。 继续阅读以查看内部标记与表单标记一起使用的更复杂示例。
以下清单显示了生成的HTML，它看起来像标准表单:

~~~~ html
<form method="POST">
    <table>
        <tr>
            <td>First Name:</td>
            <td><input name="firstName" type="text" value="Harry"/></td>
        </tr>
        <tr>
            <td>Last Name:</td>
            <td><input name="lastName" type="text" value="Potter"/></td>
        </tr>
        <tr>
            <td colspan="2">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form>
~~~~

前面的JSP假定表单支持对象的变量名是命令。 如果已将表单支持对象放在另一个名称下的模型中（绝对是最佳实践），则可以将表单绑定到命名变量，如以下示例所示：

~~~~ html
<form:form modelAttribute="user">
    <table>
        <tr>
            <td>First Name:</td>
            <td><form:input path="firstName"/></td>
        </tr>
        <tr>
            <td>Last Name:</td>
            <td><form:input path="lastName"/></td>
        </tr>
        <tr>
            <td colspan="2">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form:form>
~~~~

###### 输入标签

此标记默认使用绑定值和type ='text'呈现HTML输入元素。 有关此标记的示例，请参阅表单标记。 您还可以使用特定于HTML5的类型，例如电子邮件，电话，日期等。

###### 复选框标记

此标记呈现HTML输入标记，其类型设置为checkbox。
假设我们的用户具有诸如简报订阅和爱好列表之类的偏好。 以下示例显示了Preferences类：

~~~~ java
public class Preferences {

    private boolean receiveNewsletter;
    private String[] interests;
    private String favouriteWord;

    public boolean isReceiveNewsletter() {
        return receiveNewsletter;
    }

    public void setReceiveNewsletter(boolean receiveNewsletter) {
        this.receiveNewsletter = receiveNewsletter;
    }

    public String[] getInterests() {
        return interests;
    }

    public void setInterests(String[] interests) {
        this.interests = interests;
    }

    public String getFavouriteWord() {
        return favouriteWord;
    }

    public void setFavouriteWord(String favouriteWord) {
        this.favouriteWord = favouriteWord;
    }
}
~~~~

相应的form.jsp可能类似于以下内容：

~~~~ html
<form:form>
    <table>
        <tr>
            <td>Subscribe to newsletter?:</td>
            <%-- Approach 1: Property is of type java.lang.Boolean --%>
            <td><form:checkbox path="preferences.receiveNewsletter"/></td>
        </tr>

        <tr>
            <td>Interests:</td>
            <%-- Approach 2: Property is of an array or of type java.util.Collection --%>
            <td>
                Quidditch: <form:checkbox path="preferences.interests" value="Quidditch"/>
                Herbology: <form:checkbox path="preferences.interests" value="Herbology"/>
                Defence Against the Dark Arts: <form:checkbox path="preferences.interests" value="Defence Against the Dark Arts"/>
            </td>
        </tr>

        <tr>
            <td>Favourite Word:</td>
            <%-- Approach 3: Property is of type java.lang.Object --%>
            <td>
                Magic: <form:checkbox path="preferences.favouriteWord" value="Magic"/>
            </td>
        </tr>
    </table>
</form:form>
~~~~

checkbox标签有三种方法，可满足您的所有复选框需求。
方法一：当绑定值的类型为java.lang.Boolean时，如果绑定值为true，则输入（复选框）将标记为已检查。 value属性对应于setValue（Object）value属性的已解析值。
方法二：当绑定值的类型为array或java.util.Collection时，如果绑定的Collection中存在已配置的setValue（Object）值，则输入（复选框）将标记为已检查。
方法三：对于任何其他绑定值类型，如果配置的setValue（Object）等于绑定值，则将输入（复选框）标记为已检查。
请注意，无论采用何种方法，都会生成相同的HTML结构。 以下HTML代码段定义了一些复选框：

~~~~~ html
<tr>
    <td>Interests:</td>
    <td>
        Quidditch: <input name="preferences.interests" type="checkbox" value="Quidditch"/>
        <input type="hidden" value="1" name="_preferences.interests"/>
        Herbology: <input name="preferences.interests" type="checkbox" value="Herbology"/>
        <input type="hidden" value="1" name="_preferences.interests"/>
        Defence Against the Dark Arts: <input name="preferences.interests" type="checkbox" value="Defence Against the Dark Arts"/>
        <input type="hidden" value="1" name="_preferences.interests"/>
    </td>
</tr>
~~~~~

您可能不希望在每个复选框后看到额外的隐藏字段。如果未检查HTML页面中的复选框，则在提交表单后，其值不会作为HTTP请求参数的一部分发送到服务器，因此我们需要一种解决方法，以便在HTML中使用Spring表单数据绑定工作。 checkbox标签遵循现有的Spring约定，包括每个复选框以下划线（_）为前缀的隐藏参数。通过这样做，你有效地告诉Spring“复选框在表单中是可见的，我希望表单数据绑定的对象反映复选框的状态，无论如何。”

###### 标签复选框

此标记呈现多个HTML输入标记，其类型设置为复选框。
本节以上一个复选框标记部分的示例为基础。有时，您不希望在JSP页面中列出所有可能的爱好。您宁愿在运行时提供可用选项的列表，并将其传递给标记。这就是checkboxes标签的用途。您可以传入包含items属性中可用选项的Array，List或Map。通常，绑定属性是一个集合，以便它可以保存用户选择的多个值。以下示例显示了使用此标记的JSP：

~~~~ html
<form:form>
    <table>
        <tr>
            <td>Interests:</td>
            <td>
                <%-- Property is of an array or of type java.util.Collection --%>
                <form:checkboxes path="preferences.interests" items="${interestList}"/>
            </td>
        </tr>
    </table>
</form:form>
~~~~

此示例假定interestList是可用作模型属性的List，其中包含要从中选择的值的字符串。 如果使用Map，则使用映射条目键作为值，并将映射条目的值用作要显示的标签。 您还可以使用自定义对象，您可以使用itemValue提供值的属性名称，使用itemLabel提供标签。

###### radio-button标签

此标记呈现类型设置为radio的HTML输入元素。
典型的使用模式涉及绑定到同一属性但具有不同值的多个标记实例，如以下示例所示：

~~~~ html
<tr>
    <td>Sex:</td>
    <td>
        Male: <form:radiobutton path="sex" value="M"/> <br/>
        Female: <form:radiobutton path="sex" value="F"/>
    </td>
</tr>
~~~~

radiobuttons标签
此标记呈现多个HTML输入元素，类型设置为radio。
与checkboxes标记一样，您可能希望将可用选项作为运行时变量传递。 对于此用法，您可以使用radiobuttons标记。 传入包含items属性中可用选项的Array，List或Map。 如果使用Map，则使用映射条目键作为值，并将映射条目的值用作要显示的标签。 您还可以使用自定义对象，您可以使用itemValue提供值的属性名称，使用itemLabel提供标签，如以下示例所示：

~~~~ html
<tr>
    <td>Sex:</td>
    <td><form:radiobuttons path="sex" items="${sexOptions}"/></td>
</tr>
~~~~

###### 密码标签

此标记呈现HTML输入标记，其类型设置为带有绑定值的password。

~~~~ html
<tr>
    <td>Password:</td>
    <td>
        <form:password path="password"/>
    </td>
</tr>
~~~~

请注意，默认情况下，不显示密码值。 如果确实需要显示密码值，可以将showPassword属性的值设置为true，如以下示例所示：

~~~~ html
<tr>
    <td>Password:</td>
    <td>
        <form:password path="password" value="^76525bvHGq" showPassword="true"/>
    </td>
</tr>
~~~~

###### 选择标签

此标记呈现HTML“select”元素。 它支持数据绑定到所选选项以及使用嵌套选项和选项标记。
假设用户有一个技能列表。 相应的HTML可以如下：

~~~~ html
<tr>
    <td>Skills:</td>
    <td><form:select path="skills" items="${skills}"/></td>
</tr>
~~~~

如果用户的技能在Herbology中，“技能”行的HTML源可能如下：

~~~~ html
<tr>
    <td>Skills:</td>
    <td>
        <select name="skills" multiple="true">
            <option value="Potions">Potions</option>
            <option value="Herbology" selected="selected">Herbology</option>
            <option value="Quidditch">Quidditch</option>
        </select>
    </td>
</tr>
~~~~

选项标签
此标记呈现HTML选项元素。 它根据绑定值设置选中。 以下HTML显示了它的典型输出：

~~~~ html
<tr>
    <td>House:</td>
    <td>
        <form:select path="house">
            <form:option value="Gryffindor"/>
            <form:option value="Hufflepuff"/>
            <form:option value="Ravenclaw"/>
            <form:option value="Slytherin"/>
        </form:select>
    </td>
</tr>
~~~~

如果用户的房子在格兰芬多，那么'House'行的HTML源代码如下：

~~~~ html
<tr>
    <td>House:</td>
    <td>
        <select name="house">
            <option value="Gryffindor" selected="selected">Gryffindor</option> 
            <option value="Hufflepuff">Hufflepuff</option>
            <option value="Ravenclaw">Ravenclaw</option>
            <option value="Slytherin">Slytherin</option>
        </select>
    </td>
</tr>
~~~~

选项标签
此标记呈现HTML选项元素的列表。 它根据绑定值设置所选属性。 以下HTML显示了它的典型输出：

~~~~ html
<tr>
    <td>Country:</td>
    <td>
        <form:select path="country">
            <form:option value="-" label="--Please Select"/>
            <form:options items="${countryList}" itemValue="code" itemLabel="name"/>
        </form:select>
    </td>
</tr>
~~~~

如果用户居住在英国，“国家/地区”行的HTML源代码如下：

~~~~ html
<tr>
    <td>Country:</td>
    <td>
        <select name="country">
            <option value="-">--Please Select</option>
            <option value="AT">Austria</option>
            <option value="UK" selected="selected">United Kingdom</option> 
            <option value="US">United States</option>
        </select>
    </td>
</tr>
~~~~

如前面的示例所示，option标签与options标签的组合使用生成相同的标准HTML，但允许您在JSP中显式指定仅用于显示的值（它所属的位置），例如， 例如：“ - 请选择”。
items属性通常使用项目对象的集合或数组填充。 itemValue和itemLabel引用这些项目对象的bean属性（如果已指定）。 否则，项目对象本身将变为字符串。 或者，您可以指定项目的地图，在这种情况下，地图关键字被解释为选项值，地图值对应于选项标签。 如果恰好也指定了itemValue或itemLabel（或两者），则item value属性将应用于map键，而item label属性将应用于map值。
textarea标签
此标记呈现HTML textarea元素。 以下HTML显示了它的典型输出：

~~~~ html
<tr>
    <td>Notes:</td>
    <td><form:textarea path="notes" rows="3" cols="20"/></td>
    <td><form:errors path="notes"/></td>
</tr>
~~~~

隐藏的标签
此标记呈现HTML输入标记，其类型设置为隐藏，具有绑定值。 要提交未绑定的隐藏值，请使用类型设置为隐藏的HTML输入标记。 以下HTML显示了它的典型输出：

~~~~ html
<form:hidden path="house"/>
~~~~

如果我们选择将房屋价值作为隐藏房屋提交，则HTML将如下所示：

~~~~ html
<input name="house" type="hidden" value="Gryffindor"/>
~~~~

错误标签
此标记呈现HTML span元素中的字段错误。 它可以访问控制器中创建的错误或由与控制器关联的任何验证器创建的错误。
假设我们要在提交表单后显示firstName和lastName字段的所有错误消息。 我们有一个名为UserValidator的User类实例的验证器，如下例所示：

~~~~ java
public class UserValidator implements Validator {

    public boolean supports(Class candidate) {
        return User.class.isAssignableFrom(candidate);
    }

    public void validate(Object obj, Errors errors) {
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "required", "Field is required.");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "lastName", "required", "Field is required.");
    }
}
~~~~

form.jsp可以如下：

~~~~ html
<form:form>
    <table>
        <tr>
            <td>First Name:</td>
            <td><form:input path="firstName"/></td>
            <%-- Show errors for firstName field --%>
            <td><form:errors path="firstName"/></td>
        </tr>

        <tr>
            <td>Last Name:</td>
            <td><form:input path="lastName"/></td>
            <%-- Show errors for lastName field --%>
            <td><form:errors path="lastName"/></td>
        </tr>
        <tr>
            <td colspan="3">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form:form>
~~~~

如果我们在firstName和lastName字段中提交带有空值的表单，则HTML将如下所示：

~~~~ html
<form method="POST">
    <table>
        <tr>
            <td>First Name:</td>
            <td><input name="firstName" type="text" value=""/></td>
            <%-- Associated errors to firstName field displayed --%>
            <td><span name="firstName.errors">Field is required.</span></td>
        </tr>

        <tr>
            <td>Last Name:</td>
            <td><input name="lastName" type="text" value=""/></td>
            <%-- Associated errors to lastName field displayed --%>
            <td><span name="lastName.errors">Field is required.</span></td>
        </tr>
        <tr>
            <td colspan="3">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form>
~~~~

如果我们想显示给定页面的整个错误列表怎么办？ 下一个示例显示errors标记还支持一些基本的通配符功能。
path =“*”：显示所有错误。
path =“lastName”：显示与lastName字段关联的所有错误。
如果省略path，则仅显示对象错误。
以下示例显示页面顶部的错误列表，然后是字段旁边的字段特定错误：

~~~~ html
<form:form>
    <form:errors path="*" cssClass="errorBox"/>
    <table>
        <tr>
            <td>First Name:</td>
            <td><form:input path="firstName"/></td>
            <td><form:errors path="firstName"/></td>
        </tr>
        <tr>
            <td>Last Name:</td>
            <td><form:input path="lastName"/></td>
            <td><form:errors path="lastName"/></td>
        </tr>
        <tr>
            <td colspan="3">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form:form>
~~~~

HTML将如下：

~~~~ html
<form method="POST">
    <span name="*.errors" class="errorBox">Field is required.<br/>Field is required.</span>
    <table>
        <tr>
            <td>First Name:</td>
            <td><input name="firstName" type="text" value=""/></td>
            <td><span name="firstName.errors">Field is required.</span></td>
        </tr>

        <tr>
            <td>Last Name:</td>
            <td><input name="lastName" type="text" value=""/></td>
            <td><span name="lastName.errors">Field is required.</span></td>
        </tr>
        <tr>
            <td colspan="3">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form>
~~~~

spring-webmvc.jar中包含spring-form.tld标记库描述符（TLD）。 有关各个标签的综合参考，请浏览API参考或查看标签库说明。

###### HTTP方法转换

REST的一个关键原则是使用“统一接口”。这意味着可以使用相同的四种HTTP方法操作所有资源（URL）：GET，PUT，POST和DELETE。对于每个方法，HTTP规范定义了确切的语义。例如，GET应始终是一个安全的操作，这意味着它没有副作用，PUT或DELETE应该是幂等的，这意味着你可以一遍又一遍地重复这些操作，但最终结果应该是相同的。虽然HTTP定义了这四种方法，但HTML只支持两种：GET和POST。幸运的是，有两种可能的解决方法：您可以使用JavaScript来执行PUT或DELETE，也可以使用“real”方法作为附加参数进行POST（在HTML表单中建模为隐藏输入字段）。 Spring的HiddenHttpMethodFilter使用了后一种技巧。这个过滤器是一个普通的Servlet过滤器，因此，它可以与任何Web框架（不仅仅是Spring MVC）结合使用。将此过滤器添加到web.xml，并将带有隐藏方法参数的POST转换为相应的HTTP方法请求。
为了支持HTTP方法转换，更新了Spring MVC表单标记以支持

~~~~ html
<form:form method="delete">
    <p class="submit"><input type="submit" value="Delete Pet"/></p>
</form:form>
~~~~

上面的示例执行HTTP POST，其中“real”DELETE方法隐藏在请求参数后面。 它由HiddenHttpMethodFilter选取，它在web.xml中定义，如下例所示：

~~~~ xml
<filter>
    <filter-name>httpMethodFilter</filter-name>
    <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>httpMethodFilter</filter-name>
    <servlet-name>petclinic</servlet-name>
</filter-mapping>
~~~~

以下示例显示了相应的@Controller方法：

~~~~ java
@RequestMapping(method = RequestMethod.DELETE)
public String deletePet(@PathVariable int ownerId, @PathVariable int petId) {
    this.clinic.deletePet(petId);
    return "redirect:/owners/" + ownerId;
}
~~~~

###### HTML5标签

Spring表单标记库允许输入动态属性，这意味着您可以输入任何HTML5特定属性。
表单输入标记支持输入文本以外的类型属性。 这旨在允许呈现新的HTML5特定输入类型，例如电子邮件，日期，范围等。 请注意，不需要输入type ='text'，因为text是默认类型。

#### 1.9.6. Tiles

您可以在使用Spring的Web应用程序中集成Tiles  - 就像任何其他视图技术一样。 本节以广泛的方式介绍了如何执行此操作。

**本节重点介绍Spring在org.springframework.web.servlet.view.tiles3包中对Tiles版本3的支持。**

###### 依赖

为了能够使用Tiles，您必须在Tiles 3.0.1或更高版本上添加依赖项及其对项目的传递依赖性。

###### 配置

为了能够使用Tiles，您必须使用包含定义的文件对其进行配置（有关定义和其他Tiles概念的基本信息，请参阅https://tiles.apache.org）。 在Spring中，这是通过使用TilesConfigurer完成的。 以下示例ApplicationContext配置显示了如何执行此操作：

~~~~ xml
<bean id="tilesConfigurer" class="org.springframework.web.servlet.view.tiles3.TilesConfigurer">
    <property name="definitions">
        <list>
            <value>/WEB-INF/defs/general.xml</value>
            <value>/WEB-INF/defs/widgets.xml</value>
            <value>/WEB-INF/defs/administrator.xml</value>
            <value>/WEB-INF/defs/customer.xml</value>
            <value>/WEB-INF/defs/templates.xml</value>
        </list>
    </property>
</bean>
~~~~

上面的示例定义了包含定义的五个文件。 这些文件都位于WEB-INF / defs目录中。 在初始化WebApplicationContext时，将加载文件，并初始化定义工厂。 完成此操作后，定义文件中包含的Tiles可用作Spring Web应用程序中的视图。 为了能够使用这些视图，您必须拥有ViewResolver，就像使用Spring的任何其他视图技术一样。 您可以使用两种实现中的任何一种，UrlBasedViewResolver和ResourceBundleViewResolver。
您可以通过添加下划线然后添加区域设置来指定特定于区域设置的Tiles定义，如以下示例所示：

~~~~ xml
<bean id="tilesConfigurer" class="org.springframework.web.servlet.view.tiles3.TilesConfigurer">
    <property name="definitions">
        <list>
            <value>/WEB-INF/defs/tiles.xml</value>
            <value>/WEB-INF/defs/tiles_fr_FR.xml</value>
        </list>
    </property>
</bean>
~~~~

使用上述配置，tiles_fr_FR.xml用于具有fr_FR语言环境的请求，默认情况下使用tiles.xml。

**由于下划线用于表示区域设置，因此我们建议不要在Tiles定义的文件名中使用它们。**

###### UrlBasedViewResolver

UrlBasedViewResolver为它必须解析的每个视图实例化给定的viewClass。 以下bean定义了UrlBasedViewResolver：

~~~~ xml
<bean id="viewResolver" class="org.springframework.web.servlet.view.UrlBasedViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.tiles3.TilesView"/>
</bean>
~~~~

###### ResourceBundleViewResolver

必须为ResourceBundleViewResolver提供属性文件，该文件包含解析程序可以使用的视图名称和视图类。 以下示例显示了ResourceBundleViewResolver的bean定义以及相应的视图名称和视图类（取自Pet Clinic示例）:

~~~~ xml
<bean id="viewResolver" class="org.springframework.web.servlet.view.ResourceBundleViewResolver">
    <property name="basename" value="views"/>
</bean>
~~~~

~~~~ properties
...
welcomeView.(class)=org.springframework.web.servlet.view.tiles3.TilesView
welcomeView.url=welcome (this is the name of a Tiles definition)

vetsView.(class)=org.springframework.web.servlet.view.tiles3.TilesView
vetsView.url=vetsView (again, this is the name of a Tiles definition)

findOwnersForm.(class)=org.springframework.web.servlet.view.JstlView
findOwnersForm.url=/WEB-INF/jsp/findOwners.jsp
...
~~~~

使用ResourceBundleViewResolver时，可以轻松混合使用不同的视图技术。
请注意，TilesView类支持JSTL（JSP标准标记库）。
SimpleSpringPreparerFactory和SpringBeanPreparerFactory
作为一项高级功能，Spring还支持两种特殊的Tiles PreparerFactory实现。有关如何在Tiles定义文件中使用ViewPreparer引用的详细信息，请参阅Tiles文档。
您可以指定SimpleSpringPreparerFactory以基于指定的preparer类自动装配ViewPreparer实例，应用Spring的容器回调以及应用已配置的Spring BeanPostProcessors。如果已激活Spring的上下文范围注释配置，则会自动检测并应用ViewPreparer类中的注释。请注意，这需要Tiles定义文件中的preparer类，作为默认的PreparerFactory。
您可以指定SpringBeanPreparerFactory来操作指定的preparer名称（而不是类），从DispatcherServlet的应用程序上下文中获取相应的Spring bean。在这种情况下，完整的bean创建过程控制着Spring应用程序上下文，允许使用显式依赖项注入配置，作用域bean等。请注意，您需要为每个preparer名称定义一个Spring bean定义（在Tiles定义中使用）。以下示例显示如何在TilesConfigurer bean上定义SpringBeanPreparerFactory属性：

~~~~ xml
<bean id="tilesConfigurer" class="org.springframework.web.servlet.view.tiles3.TilesConfigurer">
    <property name="definitions">
        <list>
            <value>/WEB-INF/defs/general.xml</value>
            <value>/WEB-INF/defs/widgets.xml</value>
            <value>/WEB-INF/defs/administrator.xml</value>
            <value>/WEB-INF/defs/customer.xml</value>
            <value>/WEB-INF/defs/templates.xml</value>
        </list>
    </property>

    <!-- resolving preparer names as Spring bean definition names -->
    <property name="preparerFactoryClass"
            value="org.springframework.web.servlet.view.tiles3.SpringBeanPreparerFactory"/>

</bean>
~~~~

#### 1.9.7 RSS和Atom

AbstractAtomFeedView和AbstractRssFeedView都继承自AbstractFeedView基类，分别用于提供Atom和RSS Feed视图。 它们基于java.net的ROME项目，位于org.springframework.web.servlet.view.feed包中。
AbstractAtomFeedView要求您实现buildFeedEntries()方法并可选地覆盖buildFeedMetadata()方法（默认实现为空）。 以下示例显示了如何执行此操作:

~~~~ java
public class SampleContentAtomView extends AbstractAtomFeedView {

    @Override
    protected void buildFeedMetadata(Map<String, Object> model,
            Feed feed, HttpServletRequest request) {
        // implementation omitted
    }

    @Override
    protected List<Entry> buildFeedEntries(Map<String, Object> model,
            HttpServletRequest request, HttpServletResponse response) throws Exception {
        // implementation omitted
    }

}
~~~~

类似的要求适用于实现AbstractRssFeedView，如以下示例所示：

~~~~ java
public class SampleContentRssView extends AbstractRssFeedView {

    @Override
    protected void buildFeedMetadata(Map<String, Object> model,
            Channel feed, HttpServletRequest request) {
        // implementation omitted
    }

    @Override
    protected List<Item> buildFeedItems(Map<String, Object> model,
            HttpServletRequest request, HttpServletResponse response) throws Exception {
        // implementation omitted
    }
}
~~~~

如果您需要访问Locale，buildFeedItems（）和buildFeedEntries（）方法会传入HTTP请求。 HTTP响应仅用于设置cookie或其他HTTP标头。 方法返回后，Feed会自动写入响应对象。
有关创建Atom视图的示例，请参阅Alef Arendsen的Spring Team Blog条目。

#### 1.9.8. PDF and Excel

Spring提供了返回HTML以外输出的方法，包括PDF和Excel电子表格。 本节介绍如何使用这些功能。

###### 文档视图简介

HTML页面并不总是用户查看模型输出的最佳方式，而Spring可以从模型数据动态生成PDF文档或Excel电子表格。 该文档是视图，并从具有正确内容类型的服务器流式传输，以（希望）使客户端PC能够运行其电子表格或PDF查看器应用程序作为响应。
要使用Excel视图，需要将Apache POI库添加到类路径中。 对于PDF生成，您需要添加（最好）OpenPDF库。

**如果可能，您应该使用最新版本的基础文档生成库。 特别是，我们强烈建议使用OpenPDF（例如，OpenPDF 1.2.12）而不是过时的原始iText 2.1.7，因为OpenPDF是主动维护的，并修复了不受信任的PDF内容的重要漏洞。**

###### PDF视图

单词列表的简单PDF视图可以扩展org.springframework.web.servlet.view.document.AbstractPdfView并实现buildPdfDocument（）方法，如以下示例所示：

~~~~ java
public class PdfWordList extends AbstractPdfView {

    protected void buildPdfDocument(Map<String, Object> model, Document doc, PdfWriter writer,
            HttpServletRequest request, HttpServletResponse response) throws Exception {

        List<String> words = (List<String>) model.get("wordList");
        for (String word : words) {
            doc.add(new Paragraph(word));
        }
    }
}
~~~~

控制器可以从外部视图定义（通过名称引用它）返回这样的视图，也可以从处理程序方法返回View实例。

###### Excel视图

从Spring Framework 4.2开始，org.springframework.web.servlet.view.document.AbstractXlsView作为Excel视图的基类提供。 它基于Apache POI，具有专门的子类（AbstractXlsxView和AbstractXlsxStreamingView），取代了过时的AbstractExcelView类。
编程模型类似于AbstractPdfView，buildExcelDocument（）作为中心模板方法，控制器能够从外部定义（按名称）返回这样的视图，或者从处理程序方法返回View实例。

#### 1.9.9. Jackson

Spring为Jackson JSON库提供支持。

###### Jackson-based JSON MVC Views

MappingJackson2JsonView使用Jackson库的ObjectMapper将响应内容呈现为JSON。 默认情况下，模型映射的全部内容（特定于框架的类除外）都编码为JSON。 对于需要过滤地图内容的情况，您可以使用modelKeys属性指定要编码的特定模型属性集。 您还可以使用extractValueFromSingleKeyModel属性将单键模型中的值直接提取和序列化，而不是作为模型属性的映射。
您可以使用Jackson提供的注释根据需要自定义JSON映射。 当您需要进一步控制时，可以通过ObjectMapper属性注入自定义ObjectMapper，以用于需要为特定类型提供自定义JSON序列化程序和反序列化程序的情况。

###### 基于Jackson的XML视图

MappingJackson2XmlView使用Jackson XML扩展的XmlMapper将响应内容呈现为XML。 如果模型包含多个条目，则应使用modelKey bean属性显式设置要序列化的对象。 如果模型包含单个条目，则会自动序列化。
您可以根据需要使用JAXB或Jackson提供的注释来自定义XML映射。 当您需要进一步控制时，可以通过ObjectMapper属性注入自定义XmlMapper，以用于需要为特定类型提供序列化程序和反序列化程序的自定义XML的情况。

#### 1.9.10 XML编组

MarshallingView使用XML Marshaller（在org.springframework.oxm包中定义）将响应内容呈现为XML。 您可以使用MarshallingView实例的modelKey bean属性显式设置要编组的对象。 或者，视图会迭代所有模型属性，并编组Marshaller支持的第一种类型。 有关org.springframework.oxm包中功能的更多信息，请参阅使用O / X Mappers编组XML。

#### 1.9.11 XSLT视图

XSLT是XML的转换语言，在Web应用程序中作为视图技术很受欢迎。 如果您的应用程序自然地处理XML或者您的模型可以轻松转换为XML，那么XSLT可以作为视图技术的不错选择。 以下部分显示如何将XML文档生成为模型数据，并在Spring Web MVC应用程序中使用XSLT进行转换。
这个例子是一个简单的Spring应用程序，它在Controller中创建一个单词列表并将它们添加到模型映射中。 将返回映射以及XSLT视图的视图名称。 有关Spring Web MVC的Controller接口的详细信息，请参阅带注释的控制器。 XSLT控制器将单词列表转换为准备转换的简单XML文档。

###### Beans

配置是简单的Spring Web应用程序的标准配置：MVC配置必须定义XsltViewResolver bean和常规MVC注释配置。 以下示例显示了如何执行此操作：

~~~~ java
@EnableWebMvc
@ComponentScan
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public XsltViewResolver xsltViewResolver() {
        XsltViewResolver viewResolver = new XsltViewResolver();
        viewResolver.setPrefix("/WEB-INF/xsl/");
        viewResolver.setSuffix(".xslt");
        return viewResolver;
    }
}
~~~~

###### 控制器

我们还需要实现了我们的词语生成逻辑控制器。
控制器逻辑封装在@Controller类中，处理程序方法定义如下：

~~~~ java
@Controller
public class XsltController {

    @RequestMapping("/")
    public String home(Model model) throws Exception {
        Document document = DocumentBuilderFactory.newInstance().newDocumentBuilder().newDocument();
        Element root = document.createElement("wordList");
        List<String> words = Arrays.asList("Hello", "Spring", "Framework");
        for (String word : words) {
            Element wordNode = document.createElement("word");
            Text textNode = document.createTextNode(word);
            wordNode.appendChild(textNode);
            root.appendChild(wordNode);
        }

        model.addAttribute("wordList", root);
        return "home";
    }
}
~~~~

到目前为止，我们只创建了一个DOM文档并将其添加到Model映射中。 请注意，您还可以将XML文件作为资源加载，并使用它而不是自定义DOM文档。
有一些软件包可以自动“控制”对象图，但是，在Spring中，您可以完全灵活地以您选择的任何方式从模型中创建DOM。 这可以防止XML的转换在模型数据的结构中扮演太大的角色，这在使用工具管理DOM化过程时是一种危险。

###### Transformation

最后，XsltViewResolver解析“home”XSLT模板文件并将DOM文档合并到其中以生成我们的视图。 如XsltViewResolver配置中所示，XSLT模板位于WEB-INF / xsl目录中的war文件中，并以xslt文件扩展名结尾。
以下示例显示了XSLT转换：

~~~~ xml
<?xml version="1.0" encoding="utf-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

    <xsl:output method="html" omit-xml-declaration="yes"/>

    <xsl:template match="/">
        <html>
            <head><title>Hello!</title></head>
            <body>
                <h1>My First Words</h1>
                <ul>
                    <xsl:apply-templates/>
                </ul>
            </body>
        </html>
    </xsl:template>

    <xsl:template match="word">
        <li><xsl:value-of select="."/></li>
    </xsl:template>

</xsl:stylesheet>
~~~~

前面的转换呈现为以下HTML：

~~~~ html
<html>
    <head>
        <META http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <title>Hello!</title>
    </head>
    <body>
        <h1>My First Words</h1>
        <ul>
            <li>Hello</li>
            <li>Spring</li>
            <li>Framework</li>
        </ul>
    </body>
</html>
~~~~

### 1.10 MVC配置

MVC Java配置和MVC XML命名空间提供适用于大多数应用程序的默认配置，以及用于自定义它的配置API。
有关配置API中未提供的更高级自定义，请参阅高级Java配置和高级XML配置。
您无需了解MVC Java配置和MVC命名空间创建的基础bean。 如果您想了解更多信息，请参阅特殊Bean类型和Web MVC配置。

#### 1.10.1 启用MVC配置

在Java配置中，您可以使用@EnableWebMvc批注启用MVC配置，如以下示例所示:

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig {
}
~~~~

在XML配置中，您可以使用<mvc：annotation-driven>元素来启用MVC配置，如以下示例所示：

~~~~ xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven/>

</beans>
~~~~

前面的示例注册了许多Spring MVC基础结构bean，并适应类路径上可用的依赖项（例如，JSON，XML等的有效负载转换器）。

#### 1.10.2 MVC配置API

在Java配置中，您可以实现WebMvcConfigurer接口，如以下示例所示：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Implement configuration methods...
}
~~~~

在XML中，您可以检查<mvc：annotation-driven />的属性和子元素。 您可以查看Spring MVC XML模式或使用IDE的代码完成功能来发现可用的属性和子元素。

#### 1.10.3 类型转换

默认格式化，安装了Number和Date类型，包括对@NumberFormat和@DateTimeFormat注释的支持。 如果类路径中存在Joda-Time，则还会安装对Joda-Time格式库的完全支持。
在Java配置中，您可以注册自定义格式化程序和转换器，如以下示例所示：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        // ...
    }
}
~~~~

以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven conversion-service="conversionService"/>

    <bean id="conversionService"
            class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
        <property name="converters">
            <set>
                <bean class="org.example.MyConverter"/>
            </set>
        </property>
        <property name="formatters">
            <set>
                <bean class="org.example.MyFormatter"/>
                <bean class="org.example.MyAnnotationFormatterFactory"/>
            </set>
        </property>
        <property name="formatterRegistrars">
            <set>
                <bean class="org.example.MyFormatterRegistrar"/>
            </set>
        </property>
    </bean>

</beans>
~~~~

有关何时使用FormatterRegistrar实现的更多信息，请参阅FormatterRegistrar SPI和FormattingConversionServiceFactoryBean。

#### 1.10.4	验证

默认情况下，如果类路径上存在Bean Validation（例如，Hibernate Validator），则LocalValidatorFactoryBean将注册为全局Validator，以便与@Valid一起使用，并在控制器方法参数上使用Validated
在Java配置中，您可以自定义全局Validator实例，如以下示例所示：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public Validator getValidator(); {
        // ...
    }
}
~~~~

以下示例显示如何在XML中实现相同的配置：

~~~ xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven validator="globalValidator"/>

</beans>
~~~

请注意，您还可以在本地注册Validator实现，如以下示例所示：

~~~~ java
@Controller
public class MyController {

    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addValidators(new FooValidator());
    }

}
~~~~

**如果需要在某处注入LocalValidatorFactoryBean，请创建一个bean并使用@Primary标记它，以避免与MVC配置中声明的那个冲突。**

#### 1.10.5	拦截器

在Java配置中，您可以注册拦截器以应用传入请求，如以下示例所示：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor());
        registry.addInterceptor(new ThemeChangeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
        registry.addInterceptor(new SecurityInterceptor()).addPathPatterns("/secure/*");
    }
}
~~~~

以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<mvc:interceptors>
    <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor"/>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <mvc:exclude-mapping path="/admin/**"/>
        <bean class="org.springframework.web.servlet.theme.ThemeChangeInterceptor"/>
    </mvc:interceptor>
    <mvc:interceptor>
        <mvc:mapping path="/secure/*"/>
        <bean class="org.example.SecurityInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
~~~~

#### 1.10.6. Content Types

您可以配置Spring MVC如何根据请求确定所请求的媒体类型（例如，Accept标头，URL路径扩展，查询参数等）。
默认情况下，首先检查URL路径扩展 - 将json，xml，rss和atom注册为已知扩展（取决于类路径依赖性）。 第二次检查Accept标头。
请考虑将这些默认值更改为仅接受标头，如果必须使用基于URL的内容类型解析，请考虑使用查询参数策略而不是路径扩展。 有关详细信息，请参阅后缀匹配和后缀匹配以及RFD。
在Java配置中，您可以自定义请求的内容类型解析，如以下示例所示：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.mediaType("json", MediaType.APPLICATION_JSON);
        configurer.mediaType("xml", MediaType.APPLICATION_XML);
    }
}
~~~~

以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<mvc:annotation-driven content-negotiation-manager="contentNegotiationManager"/>

<bean id="contentNegotiationManager" class="org.springframework.web.accept.ContentNegotiationManagerFactoryBean">
    <property name="mediaTypes">
        <value>
            json=application/json
            xml=application/xml
        </value>
    </property>
</bean>
~~~~

#### 1.10.7 消息转换器

您可以通过覆盖configureMessageConverters（）（替换Spring MVC创建的默认转换器）或覆盖extendMessageConverters（）来自定义Java配置中的HttpMessageConverter（以自定义默认转换器或将其他转换器添加到默认转换器）。
以下示例使用自定义的ObjectMapper而不是默认的ObjectMapper添加XML和Jackson JSON转换器：

~~~~~ java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
                .indentOutput(true)
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
                .modulesToInstall(new ParameterNamesModule());
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
        converters.add(new MappingJackson2XmlHttpMessageConverter(builder.createXmlMapper(true).build()));
    }
}
~~~~~

在前面的示例中，Jackson2ObjectMapperBuilder用于为MappingJackson2HttpMessageConverter和MappingJackson2XmlHttpMessageConverter创建通用配置，并启用缩进，自定义日期格式以及jackson-module-parameter-names的注册，这增加了对访问参数名称的支持（添加了一项功能） 在Java 8）。
此构建器自定义Jackson的默认属性，如下所示：
DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES已禁用。
MapperFeature.DEFAULT_VIEW_INCLUSION已禁用。
如果在类路径中检测到它们，它还会自动注册以下众所周知的模块：
jackson-datatype-jdk7：支持Java 7类型，例如java.nio.file.Path。
jackson-datatype-joda：支持Joda-Time类型。
jackson-datatype-jsr310：支持Java 8 Date和Time API类型。
jackson-datatype-jdk8：支持其他Java 8类型，例如Optional。

使用Jackson XML支持启用缩进除了jackson-dataformat-xml之外还需要woodstox-core-asl依赖。

其他有趣的Jackson 模块可用：
jackson-datatype-money：支持javax.money类型（非官方模块）。
jackson-datatype-hibernate：支持特定于Hibernate的类型和属性（包括延迟加载方面）。
以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
            <property name="objectMapper" ref="objectMapper"/>
        </bean>
        <bean class="org.springframework.http.converter.xml.MappingJackson2XmlHttpMessageConverter">
            <property name="objectMapper" ref="xmlMapper"/>
        </bean>
    </mvc:message-converters>
</mvc:annotation-driven>

<bean id="objectMapper" class="org.springframework.http.converter.json.Jackson2ObjectMapperFactoryBean"
      p:indentOutput="true"
      p:simpleDateFormat="yyyy-MM-dd"
      p:modulesToInstall="com.fasterxml.jackson.module.paramnames.ParameterNamesModule"/>

<bean id="xmlMapper" parent="objectMapper" p:createXmlMapper="true"/>
~~~~

#### 1.10.8 视图控制器

这是定义ParameterizableViewController的快捷方式，该方法在调用时立即转发到视图。 如果在视图生成响应之前没有要执行的Java控制器逻辑，则可以在静态情况下使用它。
以下Java配置示例将请求转发给名为home的视图：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home");
    }
}
~~~~

以下示例与前面的示例实现相同的功能，但使用XML，使用<mvc：view-controller>元素：

~~~~ xml
<mvc:view-controller path="/" view-name="home"/>
~~~~

#### 1.10.9 视图解析器

MVC配置简化了视图解析器的注册。
以下Java配置示例使用JSP和Jackson作为JSON呈现的默认视图来配置内容协商视图解析：

~~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.enableContentNegotiation(new MappingJackson2JsonView());
        registry.jsp();
    }
}
~~~~~

以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<mvc:view-resolvers>
    <mvc:content-negotiation>
        <mvc:default-views>
            <bean class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"/>
        </mvc:default-views>
    </mvc:content-negotiation>
    <mvc:jsp/>
</mvc:view-resolvers>
~~~~

但请注意，FreeMarker，Tiles，Groovy Markup和脚本模板也需要配置底层视图技术。
MVC名称空间提供专用元素。 以下示例适用于FreeMarker：

~~~~ xml
<mvc:view-resolvers>
    <mvc:content-negotiation>
        <mvc:default-views>
            <bean class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"/>
        </mvc:default-views>
    </mvc:content-negotiation>
    <mvc:freemarker cache="false"/>
</mvc:view-resolvers>

<mvc:freemarker-configurer>
    <mvc:template-loader-path location="/freemarker"/>
</mvc:freemarker-configurer>
~~~~

在Java配置中，您可以添加相应的Configurer bean，如以下示例所示：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.enableContentNegotiation(new MappingJackson2JsonView());
        registry.freeMarker().cache(false);
    }

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath("/freemarker");
        return configurer;
    }
}
~~~~

#### 1.10.10 静态资源

此选项提供了一种从基于资源的位置列表中提供静态资源的便捷方法。
在下一个示例中，给定以/ resources开头的请求，相对路径用于在Web应用程序根目录下或/ static下的类路径上查找和提供相对于/ public的静态资源。 资源将在未来一年内到期，以确保最大限度地使用浏览器缓存并减少浏览器发出的HTTP请求。 还会评估Last-Modified标头，如果存在，则返回304状态代码。
以下清单显示了如何使用Java配置执行此操作：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
            .addResourceLocations("/public", "classpath:/static/")
            .setCachePeriod(31556926);
    }
}
~~~~

以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<mvc:resources mapping="/resources/**"
    location="/public, classpath:/static/"
    cache-period="31556926" />
~~~~

另请参见对静态资源的HTTP缓存支持。
资源处理程序还支持一系列ResourceResolver实现和ResourceTransformer实现，您可以使用它们创建工具链以使用优化的资源。
您可以将VersionResourceResolver用于基于从内容，固定应用程序版本或其他计算的MD5哈希的版本化资源URL。 ContentVersionStrategy（MD5哈希）是一个不错的选择 - 有一些值得注意的例外，例如与模块加载器一起使用的JavaScript资源。
以下示例显示如何在Java配置中使用VersionResourceResolver：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("/public/")
                .resourceChain(true)
                .addResolver(new VersionResourceResolver().addContentVersionStrategy("/**"));
    }
}
~~~~

以下示例显示如何在XML中实现相同的配置:

~~~~ xml
<mvc:resources mapping="/resources/**" location="/public/">
    <mvc:resource-chain resource-cache="true">
        <mvc:resolvers>
            <mvc:version-resolver>
                <mvc:content-version-strategy patterns="/**"/>
            </mvc:version-resolver>
        </mvc:resolvers>
    </mvc:resource-chain>
</mvc:resources>
~~~~

然后，您可以使用ResourceUrlProvider重写URL并应用完整的解析器和转换器链 - 例如，插入版本。 MVC配置提供了ResourceUrlProvider bean，以便可以将其注入其他bean。您还可以使用针对Thymeleaf，JSP，FreeMarker和其他具有依赖于HttpServletResponse #creditURL的URL标记的ResourceUrlEncodingFilter使重写透明。
请注意，在使用EncodedResourceResolver（例如，用于提供gzip或brotli编码的资源）和VersionResourceResolver时，必须按此顺序注册它们。这可确保始终基于未编码的文件可靠地计算基于内容的版本。
WebJars也通过WebJarsResourceResolver支持，当类路径中存在org.webjars：webjars-locator-core库时，WebJarsResourceResolver会自动注册。解析器可以重写URL以包含jar的版本，也可以匹配没有版本的传入URL  - 例如，从/jquery/jquery.min.js到/jquery/1.2.0/jquery.min.js。

#### 1.10.11 默认Servlet

Spring MVC允许将DispatcherServlet映射到/（从而覆盖容器的默认Servlet的映射），同时仍允许容器的默认Servlet处理静态资源请求。 它配置DefaultServletHttpRequestHandler，其URL映射为/ **，并且相对于其他URL映射具有最低优先级。
此处理程序将所有请求转发到默认Servlet。 因此，它必须按所有其他URL HandlerMappings的顺序保持最后。 如果您使用<mvc：annotation-driven>就是这种情况。 或者，如果您设置自己的自定义HandlerMapping实例，请确保将其order属性设置为低于DefaultServletHttpRequestHandler的值，即Integer.MAX_VALUE。
以下示例显示如何使用默认设置启用该功能：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
~~~~

以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<mvc:default-servlet-handler/>
~~~~

覆盖/ Servlet映射的警告是，必须通过名称而不是路径来检索默认Servlet的RequestDispatcher。 DefaultServletHttpRequestHandler尝试使用大多数主要Servlet容器（包括Tomcat，Jetty，GlassFish，JBoss，Resin，WebLogic和WebSphere）的已知名称列表，在启动时自动检测容器的默认Servlet。 如果默认Servlet已使用其他名称进行自定义配置，或者在默认Servlet名称未知的情况下使用其他Servlet容器，则必须显式提供默认Servlet名称，如以下示例所示：

~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable("myCustomDefaultServlet");
    }

}
~~~~

以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<mvc:default-servlet-handler default-servlet-name="myCustomDefaultServlet"/>
~~~~

#### 1.10.12 路径匹配

您可以自定义与路径匹配和URL处理相关的选项。 有关各个选项的详细信息，请参阅PathMatchConfigurer javadoc。
以下示例显示如何在Java配置中自定义路径匹配：

~~~~~ java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer
            .setUseSuffixPatternMatch(true)
            .setUseTrailingSlashMatch(false)
            .setUseRegisteredSuffixPatternMatch(true)
            .setPathMatcher(antPathMatcher())
            .setUrlPathHelper(urlPathHelper())
            .addPathPrefix("/api",
                    HandlerTypePredicate.forAnnotation(RestController.class));
    }

    @Bean
    public UrlPathHelper urlPathHelper() {
        //...
    }

    @Bean
    public PathMatcher antPathMatcher() {
        //...
    }
}
~~~~~

以下示例显示如何在XML中实现相同的配置：

~~~~ xml
<mvc:annotation-driven>
    <mvc:path-matching
        suffix-pattern="true"
        trailing-slash="false"
        registered-suffixes-only="true"
        path-helper="pathHelper"
        path-matcher="pathMatcher"/>
</mvc:annotation-driven>

<bean id="pathHelper" class="org.example.app.MyPathHelper"/>
<bean id="pathMatcher" class="org.example.app.MyPathMatcher"/>
~~~~

#### 1.10.13 高级Java配置

@EnableWebMvc导入DelegatingWebMvcConfiguration，其中：
为Spring MVC应用程序提供默认的Spring配置
检测并委派给WebMvcConfigurer实现以自定义该配置。
对于高级模式，您可以删除@EnableWebMvc并直接从DelegatingWebMvcConfiguration扩展而不是实现WebMvcConfigurer，如以下示例所示：

~~~~ java
@Configuration
public class WebConfig extends DelegatingWebMvcConfiguration {

    // ...

}
~~~~

您可以在WebConfig中保留现有方法，但现在您也可以从基类覆盖bean声明，并且您仍然可以在类路径上拥有任意数量的其他WebMvcConfigurer实现。

#### 1.10.14 高级XML配置

MVC命名空间没有高级模式。 如果您需要在bean上自定义一个您无法更改的属性，则可以使用Spring ApplicationContext的BeanPostProcessor生命周期钩子，如以下示例所示：

~~~~ java
@Component
public class MyPostProcessor implements BeanPostProcessor {

    public Object postProcessBeforeInitialization(Object bean, String name) throws BeansException {
        // ...
    }
}
~~~~

请注意，您需要将MyPostProcessor声明为bean，可以在XML中显式声明，也可以通过<component-scan />声明来检测它。

### 1.11	HTTP /2

Servlet 4容器需要支持HTTP / 2，Spring Framework 5与Servlet API 4兼容。从编程模型的角度来看，应用程序不需要特定的任何操作。 但是，存在与服务器配置相关的注意事项。 有关更多详细信息，请参阅HTTP / 2 Wiki页面。

Servlet API确实公开了一个与HTTP / 2相关的构造。 您可以使用javax.servlet.http.PushBuilder主动将资源推送到客户端，并且它被支持作为@RequestMapping方法的方法参数。

## 2. REST Clients

本节介绍客户端访问REST端点的选项。

### 2.1. `RestTemplate`

RestTemplate是一个执行HTTP请求的同步客户端。 它是最初的Spring REST客户端，在底层HTTP客户端库上公开了一个简单的模板方法API。

**从5.0开始，非阻塞，反应式WebClient提供了RestTemplate的现代替代方案，同时有效支持同步和异步以及流方案。 RestTemplate将在未来版本中弃用，并且不会在未来添加主要的新功能。**

有关详细信息，请参阅REST端点。

### 2.2	Web客户端

WebClient是一个执行HTTP请求的非阻塞，被动的客户端。 它是在5.0中引入的，它提供了RestTemplate的现代替代方案，同时有效支持同步和异步以及流方案。
与RestTemplate相比，WebClient支持以下内容：
非阻塞I / O.
Reactive Streams背压。
高并发性，硬件资源更少。
功能风格，流畅的API，利用Java 8 lambdas。
同步和异步交互。
从服务器流式传输或向下传输。
有关详细信息，请参阅WebClient。

## 3.测试

本节总结了Spring MVC应用程序spring-test中可用的选项。
Servlet API模拟：用于单元测试控制器，过滤器和其他Web组件的Servlet API契约的模拟实现。有关更多详细信息，请参阅Servlet API模拟对象。
TestContext Framework：支持在JUnit和TestNG测试中加载Spring配置，包括跨测试方法高效缓存加载的配置，以及支持使用MockServletContext加载WebApplicationContext。有关更多详细信息，请参阅TestContext Framework。
Spring MVC Test：一个框架，也称为MockMvc，用于通过DispatcherServlet测试带注释的控制器（即支持注释），完成Spring MVC基础结构但没有HTTP服务器。有关更多详细信息，请参阅Spring MVC Test。
客户端REST：spring-test提供了一个MockRestServiceServer，您可以将其用作模拟服务器，以测试内部使用RestTemplate的客户端代码。有关详细信息，请参阅客户端REST测试。
WebTestClient：专为测试WebFlux应用程序而构建，但它也可用于通过HTTP连接对任何服务器进行端到端集成测试。它是一个非阻塞，被动的客户端，非常适合测试异步和流式方案。

## 4. WebSockets

这部分参考文档包括对Servlet堆栈的支持，包括原始WebSocket交互的WebSocket消息传递，通过SockJS的WebSocket仿真，以及通过STOMP作为WebSocket上的子协议的发布 - 订阅消息传递。

### 4.1 WebSocket简介

WebSocket协议RFC 6455提供了一种标准化方法，通过单个TCP连接在客户端和服务器之间建立全双工双向通信通道。 它是来自HTTP的不同TCP协议，但设计为使用端口80和443通过HTTP工作，并允许重用现有的防火墙规则。
WebSocket交互以HTTP请求开始，该HTTP请求使用HTTP Upgrade标头进行升级，或者在这种情况下，切换到WebSocket协议。 以下示例显示了这样的交互:

~~~~ properties
GET /spring-websocket-portfolio/portfolio HTTP/1.1
Host: localhost:8080
Upgrade: websocket 
Connection: Upgrade 
Sec-WebSocket-Key: Uc9l9TMkWGbHFD2qnFHltg==
Sec-WebSocket-Protocol: v10.stomp, v11.stomp
Sec-WebSocket-Version: 13
Origin: http://localhost:8080
~~~~

具有WebSocket支持的服务器返回类似于以下内容的输出，而不是通常的200状态代码：

~~~~ properties
HTTP/1.1 101 Switching Protocols 
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: 1qVdfYHU9hPOl4JYYNXF623Gzn0=
Sec-WebSocket-Protocol: v10.stomp
~~~~

成功握手后，HTTP升级请求下的TCP套接字将保持打开状态，以便客户端和服务器继续发送和接收消息。
有关WebSockets如何工作的完整介绍超出了本文档的范围。 请参阅RFC 6455，HTML5的WebSocket章节，或Web上的许多介绍和教程。
请注意，如果WebSocket服务器在Web服务器（例如nginx）后面运行，则可能需要将其配置为将WebSocket升级请求传递到WebSocket服务器。 同样，如果应用程序在云环境中运行，请检查与WebSocket支持相关的云提供程序的说明。

#### 4.1.1 HTTP与WebSocket

尽管WebSocket被设计为与HTTP兼容并且以HTTP请求开始，但重要的是要理解这两种协议会导致非常不同的体系结构和应用程序编程模型。
在HTTP和REST中，应用程序被建模为多个URL。要与应用程序交互，客户端访问这些URL，请求 - 响应样式。服务器根据HTTP URL，方法和标头将请求路由到适当的处理程序。
相比之下，在WebSockets中，初始连接通常只有一个URL。随后，所有应用程序消息都在同一TCP连接上流动。这指向完全不同的异步，事件驱动的消息传递体系结构。
WebSocket也是一种低级传输协议，与HTTP不同，它不对消息内容规定任何语义。这意味着除非客户端和服务器就消息语义达成一致，否则无法路由或处理消息。
WebSocket客户端和服务器可以通过HTTP握手请求上的Sec-WebSocket-Protocol标头协商使用更高级别的消息传递协议（例如，STOMP）。如果没有，他们需要提出自己的惯例。

#### 4.1.2 何时使用WebSockets

WebSockets可以使网页变得动态和交互。但是，在许多情况下，Ajax和HTTP流式传输或长轮询的组合可以提供简单有效的解决方案。
例如，新闻，邮件和社交订阅源需要动态更新，但每隔几分钟就可以完全正常更新。另一方面，协作，游戏和财务应用程序需要更接近实时。
仅延迟不是决定因素。如果消息量相对较低（例如，监视网络故障），HTTP流式传输或轮询可以提供有效的解决方案。它是低延迟，高频率和高容量的组合，是使用WebSocket的最佳选择。
还要记住，在Internet上，受控制之外的限制性代理可能会妨碍WebSocket交互，因为它们未配置为传递Upgrade标头，或者因为它们关闭看似空闲的长期连接。这意味着将WebSocket用于防火墙内的内部应用程序是一个比面向公众的应用程序更直接的决策。

### 4.2. WebSocket API

Spring Framework提供了一个WebSocket API，您可以使用它来编写处理WebSocket消息的客户端和服务器端应用程序。

#### 4.2.1. `WebSocketHandler`

创建WebSocket服务器就像实现WebSocketHandler一样简单，或者更有可能扩展TextWebSocketHandler或BinaryWebSocketHandler。 以下示例使用TextWebSocketHandler:

~~~~ java
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.TextMessage;

public class MyHandler extends TextWebSocketHandler {

    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) {
        // ...
    }

}
~~~~

有专门的WebSocket Java配置和XML命名空间支持，用于将前面的WebSocket处理程序映射到特定的URL，如以下示例所示：

~~~~ java
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler");
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers>
        <websocket:mapping path="/myHandler" handler="myHandler"/>
    </websocket:handlers>

    <bean id="myHandler" class="org.springframework.samples.MyHandler"/>

</beans>
~~~~~

前面的示例用于Spring MVC应用程序，应包含在DispatcherServlet的配置中。 但是，Spring的WebSocket支持不依赖于Spring MVC。 在WebSocketHttpRequestHandler的帮助下，将WebSocketHandler集成到其他HTTP服务环境中相对简单。
直接与间接使用WebSocketHandler API时，例如 通过STOMP消息传递，应用程序必须同步消息的发送，因为底层标准WebSocket会话（JSR-356）不允许并发发送。 一种选择是使用ConcurrentWebSocketSessionDecorator包装WebSocketSession。

#### 4.2.2 WebSocket握手

自定义初始HTTP WebSocket握手请求的最简单方法是通过HandshakeInterceptor，它公开握手之前和之后的方法。 您可以使用此类拦截器来阻止握手或使WebSocketSession可以使用任何属性。 以下示例使用内置拦截器将HTTP会话属性传递给WebSocket会话：

~~~~ java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/myHandler")
            .addInterceptors(new HttpSessionHandshakeInterceptor());
    }

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers>
        <websocket:mapping path="/myHandler" handler="myHandler"/>
        <websocket:handshake-interceptors>
            <bean class="org.springframework.web.socket.server.support.HttpSessionHandshakeInterceptor"/>
        </websocket:handshake-interceptors>
    </websocket:handlers>

    <bean id="myHandler" class="org.springframework.samples.MyHandler"/>

</beans>
~~~~

更高级的选项是扩展执行WebSocket握手步骤的DefaultHandshakeHandler，包括验证客户端来源，协商子协议以及其他详细信息。 如果应用程序需要配置自定义RequestUpgradeStrategy以适应WebSocket服务器引擎和尚不支持的版本，则应用程序可能还需要使用此选项（有关此主题的更多信息，请参阅部署）。 Java配置和XML命名空间都可以配置自定义HandshakeHandler。

**Spring提供了一个WebSocketHandlerDecorator基类，您可以使用它来装饰具有其他行为的WebSocketHandler。 使用WebSocket Java配置或XML命名空间时，默认情况下会提供并添加日志记录和异常处理实现。 ExceptionWebSocketHandlerDecorator捕获由任何WebSocketHandler方法引起的所有未捕获的异常，并关闭状态为1011的WebSocket会话，这表示服务器错误。**

#### 4.2.3	部署

Spring WebSocket API易于集成到Spring MVC应用程序中，其中DispatcherServlet同时提供HTTP WebSocket握手和其他HTTP请求。通过调用WebSocketHttpRequestHandler，也可以轻松地集成到其他HTTP处理场景中。这很方便易懂。但是，特殊注意事项适用于JSR-356运行时。
Java WebSocket API（JSR-356）提供了两种部署机制。第一个涉及启动时的Servlet容器类路径扫描（Servlet 3功能）。另一个是在Servlet容器初始化时使用的注册API。这些机制都不能使用单个“前端控制器”进行所有HTTP处理 - 包括WebSocket握手和所有其他HTTP请求 - 例如Spring MVC的DispatcherServlet。
这是JSR-356的一个重要限制，Spring的WebSocket支持解决了特定于服务器的RequestUpgradeStrategy实现，即使在JSR-356运行时运行也是如此。目前，Tomcat，Jetty，GlassFish，WebLogic，WebSphere和Undertow（以及WildFly）都有这样的策略。

**已经创建了克服Java WebSocket API中的上述限制的请求，并且可以在eclipse-ee4j / websocket-api＃211中遵循该请求。 Tomcat，Undertow和WebSphere提供了自己的API替代方案，可以实现这一点，Jetty也可以实现。 我们希望更多的服务器能够做到这一点**

第二个考虑因素是具有JSR-356支持的Servlet容器应该执行ServletContainerInitializer（SCI）扫描，这可能会减慢应用程序的启动速度 - 在某些情况下会显着降低。 如果在升级到支持JSR-356的Servlet容器版本后观察到显着影响，则应该可以通过在web中使用<absolute-ordering />元素来选择性地启用或禁用Web片段（和SCI扫描） .xml，如下例所示：

~~~~ xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://java.sun.com/xml/ns/javaee
        https://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
    version="3.0">

    <absolute-ordering/>

</web-app>
~~~~

然后，您可以按名称有选择地启用Web片段，例如Spring自己的SpringServletContainerInitializer，它为Servlet 3 Java初始化API提供支持。 以下示例显示了如何执行此操作：

~~~~ xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://java.sun.com/xml/ns/javaee
        https://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
    version="3.0">

    <absolute-ordering>
        <name>spring_web</name>
    </absolute-ordering>

</web-app>
~~~~

#### 4.2.4 服务器配置

每个底层WebSocket引擎都公开控制运行时特征的配置属性，例如消息缓冲区大小，空闲超时等。
对于Tomcat，WildFly和GlassFish，您可以将ServletServerContainerFactoryBean添加到WebSocket Java配置中，如以下示例所示：

~~~~ java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Bean
    public ServletServerContainerFactoryBean createWebSocketContainer() {
        ServletServerContainerFactoryBean container = new ServletServerContainerFactoryBean();
        container.setMaxTextMessageBufferSize(8192);
        container.setMaxBinaryMessageBufferSize(8192);
        return container;
    }

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <bean class="org.springframework...ServletServerContainerFactoryBean">
        <property name="maxTextMessageBufferSize" value="8192"/>
        <property name="maxBinaryMessageBufferSize" value="8192"/>
    </bean>

</beans>
~~~~

**对于客户端WebSocket配置，您应该使用WebSocketContainerFactoryBean（XML）或ContainerProvider.getWebSocketContainer（）（Java配置）。**

对于Jetty，您需要提供预配置的Jetty WebSocketServerFactory并通过WebSocket Java配置将其插入Spring的DefaultHandshakeHandler。 以下示例显示了如何执行此操作：

~~~~ java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(echoWebSocketHandler(),
            "/echo").setHandshakeHandler(handshakeHandler());
    }

    @Bean
    public DefaultHandshakeHandler handshakeHandler() {

        WebSocketPolicy policy = new WebSocketPolicy(WebSocketBehavior.SERVER);
        policy.setInputBufferSize(8192);
        policy.setIdleTimeout(600000);

        return new DefaultHandshakeHandler(
                new JettyRequestUpgradeStrategy(new WebSocketServerFactory(policy)));
    }

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers>
        <websocket:mapping path="/echo" handler="echoHandler"/>
        <websocket:handshake-handler ref="handshakeHandler"/>
    </websocket:handlers>

    <bean id="handshakeHandler" class="org.springframework...DefaultHandshakeHandler">
        <constructor-arg ref="upgradeStrategy"/>
    </bean>

    <bean id="upgradeStrategy" class="org.springframework...JettyRequestUpgradeStrategy">
        <constructor-arg ref="serverFactory"/>
    </bean>

    <bean id="serverFactory" class="org.eclipse.jetty...WebSocketServerFactory">
        <constructor-arg>
            <bean class="org.eclipse.jetty...WebSocketPolicy">
                <constructor-arg value="SERVER"/>
                <property name="inputBufferSize" value="8092"/>
                <property name="idleTimeout" value="600000"/>
            </bean>
        </constructor-arg>
    </bean>

</beans>
~~~~

#### 4.2.5. Allowed Origins

从Spring Framework 4.1.5开始，WebSocket和SockJS的默认行为是仅接受同源请求。也可以允许所有或指定的起源列表。此检查主要是为浏览器客户端设计的。没有什么可以阻止其他类型的客户端修改Origin标头值（有关更多详细信息，请参阅RFC 6454：Web Origin Concept）。
三种可能的行为是：
仅允许同源请求（默认）：在此模式下，启用SockJS时，Iframe HTTP响应头X-Frame-Options设置为SAMEORIGIN，并禁用JSONP传输，因为它不允许检查源的请求。因此，启用此模式时不支持IE6和IE7。
允许指定的来源列表：每个允许的来源必须以http：//或https：//开头。在此模式下，启用SockJS时，将禁用IFrame传输。因此，启用此模式时，不支持IE6到IE9。
允许所有来源：要启用此模式，您应提供*作为允许的原始值。在此模式下，所有传输都可用。
您可以配置WebSocket和SockJS允许的源，如以下示例所示：

~~~~ java
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler").setAllowedOrigins("https://mydomain.com");
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers allowed-origins="https://mydomain.com">
        <websocket:mapping path="/myHandler" handler="myHandler" />
    </websocket:handlers>

    <bean id="myHandler" class="org.springframework.samples.MyHandler"/>

</beans>
~~~~

#### 4.3 SockJS回退

在公共Internet上，您控制之外的限制性代理可能会阻止WebSocket交互，因为它们未配置为传递Upgrade标头，或者因为它们关闭看似空闲的长期连接。
这个问题的解决方案是WebSocket仿真 - 也就是说，首先尝试使用WebSocket，然后再回到基于HTTP的技术，这些技术模拟WebSocket交互并公开相同的应用程序级API。
在Servlet堆栈上，Spring Framework为SockJS协议提供服务器（以及客户端）支持。

#### 4.3.1	概观

SockJS的目标是让应用程序使用WebSocket API，但在运行时必要时可以回退到非WebSocket替代方案，而无需更改应用程序代码。
SockJS包括：
SockJS协议以可执行的叙述测试的形式定义。
SockJS JavaScript客户端 - 用于浏览器的客户端库。
SockJS服务器实现，包括Spring Framework spring-websocket模块中的一个。
spring-websocket模块中的SockJS Java客户端（自4.1版本起）。
SockJS专为在浏览器中使用而设计。它使用各种技术来支持各种浏览器版本。有关SockJS传输类型和浏览器的完整列表，请参阅SockJS客户端页面。传输分为三大类：WebSocket，HTTP Streaming和HTTP Long Polling。有关这些类别的概述，请参阅此博客文章。
SockJS客户端首先发送GET / info以从服务器获取基本信息。之后，它必须决定使用什么传输。如果可能，使用WebSocket。如果没有，在大多数浏览器中，至少有一个HTTP流选项。如果不是，则使用HTTP（长）轮询。
所有传输请求都具有以下URL结构：

~~~~ properties
http://host:port/myApp/myEndpoint/{server-id}/{session-id}/{transport}
~~~~

WebSocket传输只需要一个HTTP请求即可进行WebSocket握手。此后的所有消息都在该套接字上交换。
HTTP传输需要更多请求。例如，Ajax / XHR流依赖于一个长期运行的服务器到客户端消息请求以及针对客户端到服务器消息的额外HTTP POST请求。长轮询类似，只是它在每个服务器到客户端发送后结束当前请求。
SockJS增加了最小的消息框架。例如，服务器最初发送字母o（“打开”帧），消息作为[“message1”，“message2”]（JSON编码数组）发送，字母h（“heartbeat”帧）如果没有消息流动25秒（默认情况下）和字母c（“关闭”框架）以关闭会话。
要了解更多信息，请在浏览器中运行示例并观察HTTP请求。 SockJS客户端允许修复传输列表，因此可以一次查看每个传输。 SockJS客户端还提供调试标志，该标志在浏览器控制台中启用有用的消息。在服务器端，您可以为org.springframework.web.socket启用TRACE日志记录。有关更多详细信息，请参阅SockJS协议叙述测试。

#### 4.3.2 启用SockJS

您可以通过Java配置启用SockJS，如以下示例所示：

~~~~ java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler").withSockJS();
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers>
        <websocket:mapping path="/myHandler" handler="myHandler"/>
        <websocket:sockjs/>
    </websocket:handlers>

    <bean id="myHandler" class="org.springframework.samples.MyHandler"/>

</beans>
~~~~

前面的示例用于Spring MVC应用程序，应包含在DispatcherServlet的配置中。 但是，Spring的WebSocket和SockJS支持并不依赖于Spring MVC。 在SockJsHttpRequestHandler的帮助下，将其集成到其他HTTP服务环境中相对简单。
在浏览器端，应用程序可以使用sockjs-client（版本1.0.x）。 它模拟W3C WebSocket API并与服务器通信以选择最佳传输选项，具体取决于运行它的浏览器。 请参阅sockjs-client页面和浏览器支持的传输类型列表。 客户端还提供了几个配置选项 - 例如，指定要包含的传输。

#### 4.3.3 IE 8和9

Internet Explorer 8和9仍在使用中。他们是拥有SockJS的关键原因。本节介绍在这些浏览器中运行的重要注意事项。
SockJS客户端通过使用Microsoft的XDomainRequest支持IE 8和9中的Ajax / XHR流。这适用于域，但不支持发送cookie。 Cookie通常对Java应用程序至关重要。但是，由于SockJS客户端可以与许多服务器类型（不仅仅是Java）一起使用，因此需要知道cookie是否重要。如果是这样，SockJS客户端更喜欢Ajax / XHR进行流式传输。否则，它依赖于基于iframe的技术。
来自SockJS客户端的第一个/ info请求是对可能影响客户选择传输的信息的请求。其中一个细节是服务器应用程序是否依赖于cookie（例如，用于身份验证或使用粘性会话进行群集）。 Spring的SockJS支持包含一个名为sessionCookieNeeded的属性。它默认启用，因为大多数Java应用程序都依赖于JSESSIONID cookie。如果您的应用程序不需要它，您可以关闭此选项，然后SockJS客户端应该在IE 8和9中选择xdr-streaming。
如果您确实使用基于iframe的传输，请记住，可以通过将HTTP响应标头X-Frame-Options设置为DENY，SAMEORIGIN或ALLOW-FROM <origin来指示浏览器阻止在给定页面上使用IFrame >。这用于防止点击劫持。

**Spring Security 3.2+支持在每个响应上设置X-Frame-Options。 默认情况下，Spring Security Java配置将其设置为DENY。 在3.2中，Spring Security XML命名空间默认情况下不设置该标头，但可以配置为执行此操作。 将来，它可以默认设置它。
有关如何配置X-Frame-Options标头设置的详细信息，请参阅Spring Security文档的默认安全标头。 您还可以查看SEC-2501的其他背景信息。**

如果您的应用程序添加X-Frame-Options响应标头（应该！）并依赖于基于iframe的传输，则需要将标头值设置为SAMEORIGIN或ALLOW-FROM <origin>。 Spring SockJS支持还需要知道SockJS客户端的位置，因为它是从iframe加载的。 默认情况下，iframe设置为从CDN位置下载SockJS客户端。 配置此选项以使用与应用程序相同的源的URL是个好主意。
以下示例显示了如何在Java配置中执行此操作：

~~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS()
                .setClientLibraryUrl("http://localhost:8080/myapp/js/sockjs-client.js");
    }

    // ...

}
~~~~~

XML命名空间通过<websocket：sockjs>元素提供了类似的选项。

**在初始开发期间，请启用SockJS客户端开发模式，以防止浏览器缓存否则将被缓存的SockJS请求（如iframe）。 有关如何启用它的详细信息，请参阅SockJS客户端页面。**

#### 4.3.4	心跳

SockJS协议要求服务器发送心跳消息以阻止代理断定连接已挂起。 Spring SockJS配置有一个名为heartbeatTime的属性，可用于自定义频率。 默认情况下，假设在该连接上没有发送其他消息，则会在25秒后发送心跳。 这个25秒的值符合以下IETF对公共互联网应用的建议。

**当通过WebSocket和SockJS使用STOMP时，如果STOMP客户端和服务器协商要交换的心跳，则禁用SockJS心跳。**

Spring SockJS支持还允许您配置TaskScheduler以安排心跳任务。 任务计划程序由线程池支持，默认设置基于可用处理器的数量。 您应该考虑根据您的特定需求自定义设置。

#### 4.3.5 客户端断开连接

HTTP流式传输和HTTP长轮询SockJS传输要求连接保持打开时间比平时长。 有关这些技术的概述，请参阅此博客文章。
在Servlet容器中，这是通过Servlet 3异步支持完成的，该支持允许退出Servlet容器线程，处理请求，并继续写入来自另一个线程的响应。
一个特定的问题是Servlet API不会为已经消失的客户端提供通知。 请参阅eclipse-ee4j / servlet-api＃44。 但是，Servlet容器会在后续尝试写入响应时引发异常。 由于Spring的SockJS服务支持服务器发送的心跳（默认情况下每25秒），这意味着通常在该时间段内检测到客户端断开连接（或者更早，如果更频繁地发送消息）。

**因此，可能会发生网络I / O故障，因为客户端已断开连接，这可能会使用不必要的堆栈跟踪填充日志。 Spring尽最大努力识别代表客户端断开连接（特定于每个服务器）的此类网络故障，并使用专用日志类别DISCONNECTED_CLIENT_LOG_CATEGORY（在AbstractSockJsSession中定义）记录最小消息。 如果需要查看堆栈跟踪，可以将该日志类别设置为TRACE。**

#### 4.3.6	SockJS和CORS

如果您允许跨源请求（请参阅允许的起源），则SockJS协议使用CORS在XHR流和轮询传输中进行跨域支持。因此，除非检测到响应中存在CORS头，否则会自动添加CORS头。因此，如果应用程序已配置为提供CORS支持（例如，通过Servlet过滤器），则Spring的SockJsService会跳过此部分。
也可以通过在Spring的SockJsService中设置suppressCors属性来禁用这些CORS头的添加。
SockJS需要以下标头和值：
Access-Control-Allow-Origin：从Origin请求头的值初始化。
Access-Control-Allow-Credentials：始终设置为true。
Access-Control-Request-Headers：从等效请求标头中的值初始化。
Access-Control-Allow-Methods：传输支持的HTTP方法（请参阅TransportType枚举）。
Access-Control-Max-Age：设置为31536000（1年）。
有关具体实现，请参阅AbstractSockJsService中的addCorsHeaders和源代码中的TransportType枚举。
或者，如果CORS配置允许，请考虑使用SockJS端点前缀排除URL，从而让Spring的SockJsService处理它。

#### 4.3.7	SockJsClient

Spring提供了一个SockJS Java客户端，无需使用浏览器即可连接到远程SockJS端点。当需要通过公共网络在两个服务器之间进行双向通信时（即，网络代理可以排除使用WebSocket协议的情况下），这尤其有用。 SockJS Java客户端对于测试目的也非常有用（例如，模拟大量并发用户）。
SockJS Java客户端支持websocket，xhr-streaming和xhr-polling传输。其余的仅适用于浏览器。
您可以使用以下命令配置WebSocketTransport：
JSR-356运行时中的StandardWebSocketClient。
JettyWebSocketClient使用Jetty 9+本机WebSocket API。
Spring的WebSocketClient的任何实现。
根据定义，XhrTransport支持xhr-streaming和xhr-polling，因为从客户端的角度来看，除了用于连接服务器的URL之外没有其他区别。目前有两种实现方式：
RestTemplateXhrTransport使用Spring的RestTemplate进行HTTP请求。
JettyXhrTransport使用Jetty的HttpClient进行HTTP请求。
以下示例显示如何创建SockJS客户端并连接到SockJS端点：

~~~~ java
List<Transport> transports = new ArrayList<>(2);
transports.add(new WebSocketTransport(new StandardWebSocketClient()));
transports.add(new RestTemplateXhrTransport());

SockJsClient sockJsClient = new SockJsClient(transports);
sockJsClient.doHandshake(new MyWebSocketHandler(), "ws://example.com:8080/sockjs");
~~~~

**SockJS使用JSON格式的数组进行消息传递。 默认情况下，使用Jackson 2并且需要在类路径上。 或者，您可以配置SockJsMessageCodec的自定义实现并在SockJsClient上配置它。**

要使用SockJsClient模拟大量并发用户，您需要配置底层HTTP客户端（用于XHR传输）以允许足够数量的连接和线程。 以下示例显示了如何使用Jetty执行此操作：

~~~~ java
HttpClient jettyHttpClient = new HttpClient();
jettyHttpClient.setMaxConnectionsPerDestination(1000);
jettyHttpClient.setExecutor(new QueuedThreadPool(1000));
~~~~

以下示例显示了您应该考虑自定义的服务器端SockJS相关属性（请参阅javadoc以获取详细信息）：

~~~~ java
@Configuration
public class WebSocketConfig extends WebSocketMessageBrokerConfigurationSupport {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/sockjs").withSockJS()
            .setStreamBytesLimit(512 * 1024) 
            .setHttpMessageCacheSize(1000) 
            .setDisconnectDelay(30 * 1000); 
    }

    // ...
}
~~~~

### 4.4. STOMP

WebSocket协议定义了两种类型的消息（文本和二进制），但它们的内容是未定义的。 该协议定义了一种机制，供客户端和服务器协商子协议（即更高级别的消息传递协议），以便在WebSocket上使用，以定义每种消息可以发送的消息类型，格式是什么，内容 每条消息，等等。 子协议的使用是可选的，但无论如何，客户端和服务器需要就定义消息内容的某些协议达成一致。

#### 4.4.1	概观

STOMP（简单文本导向的消息传递协议）最初是为脚本语言（如Ruby，Python和Perl）创建的，用于连接企业消息代理。 它旨在解决常用消息传递模式的最小子集。 STOMP可用于任何可靠的双向流网络协议，例如TCP和WebSocket。 虽然STOMP是面向文本的协议，但消息有效负载可以是文本或二进制。
STOMP是一种基于帧的协议，其帧在HTTP上建模。 以下清单显示了STOMP框架的结构：

~~~~
COMMAND
header1:value1
header2:value2

Body^@
~~~~

客户端可以使用SEND或SUBSCRIBE命令发送或订阅消息，以及描述消息内容和接收消息的目标头。这启用了一个简单的发布 - 订阅机制，您可以使用该机制通过代理将消息发送到其他连接的客户端，或者向服务器发送消息以请求执行某些工作。
当您使用Spring的STOMP支持时，Spring WebSocket应用程序充当客户端的STOMP代理。消息被路由到@Controller消息处理方法或简单的内存代理，该代理跟踪订阅并向订阅用户广播消息。您还可以将Spring配置为使用专用的STOMP代理（例如RabbitMQ，ActiveMQ等）来实现消息的实际广播。在这种情况下，Spring维护与代理的TCP连接，向其中继消息，并将消息从其传递到连接的WebSocket客户端。因此，Spring Web应用程序可以依赖统一的基于HTTP的安全性，通用验证以及用于消息处理的熟悉的编程模型。
以下示例显示订阅接收股票报价的客户，服务器可以定期发出（例如，通过SimpMessagingTemplate向代理发送消息的计划任务）：

~~~~
SUBSCRIBE
id:sub-1
destination:/topic/price.stock.*

^@
~~~~

以下示例显示了发送交易请求的客户端，服务器可以通过@MessageMapping方法处理该请求：

~~~~
SEND
destination:/queue/trade
content-type:application/json
content-length:44

{"action":"BUY","ticker":"MMM","shares",44}^@
~~~~

执行后，服务器可以向客户端广播交易确认消息和详细信息。
目的地的含义在STOMP规范中故意保持不透明。 它可以是任何字符串，完全取决于STOMP服务器来定义它们支持的目标的语义和语法。 然而，很常见的是，目标是类似路径的字符串，其中/ topic / ..意味着发布 - 订阅（一对多）和/ queue /意味着点对点（一对一）消息交流。
STOMP服务器可以使用MESSAGE命令向所有订户广播消息。 以下示例显示了向订阅客户端发送股票报价的服务器：

~~~~
MESSAGE
message-id:nxahklf6-1
subscription:sub-1
destination:/topic/price.stock.MMM

{"ticker":"MMM","price":129.45}^@
~~~~

服务器无法发送未经请求的消息。 来自服务器的所有消息必须响应特定的客户端订阅，并且服务器消息的subscription-id头必须与客户端订阅的id头匹配。
前面的概述旨在提供对STOMP协议的最基本的了解。 我们建议完整地查看协议规范。

#### 4.4.2	优点

使用STOMP作为子协议，Spring Framework和Spring Security提供了比使用原始WebSocket更丰富的编程模型。 关于HTTP与原始TCP以及它如何让Spring MVC和其他Web框架提供丰富的功能，可以做出同样的观点。 以下是一系列好处：
无需发明自定义消息传递协议和消息格式。
可以使用STOMP客户端，包括Spring Framework中的Java客户端。
您可以（可选）使用消息代理（例如RabbitMQ，ActiveMQ等）来管理订阅和广播消息。
可以在任意数量的@Controller实例中组织应用程序逻辑，并且可以基于STOMP目标标头将消息路由到它们，而不是使用给定连接的单个WebSocketHandler处理原始WebSocket消息。
您可以使用Spring Security根据STOMP目标和消息类型保护消息。

#### 4.4.3 启用STOMP

spring-messaging和spring-websocket模块提供STOMP over WebSocket支持。 一旦有了这些依赖项，就可以通过带有SockJS Fallback的WebSocket公开STOMP端点，如下例所示:

~~~~~ java
mport org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS();  
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.setApplicationDestinationPrefixes("/app"); 
        config.enableSimpleBroker("/topic", "/queue"); 
    }
}
~~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:message-broker application-destination-prefix="/app">
        <websocket:stomp-endpoint path="/portfolio">
            <websocket:sockjs/>
        </websocket:stomp-endpoint>
        <websocket:simple-broker prefix="/topic, /queue"/>
    </websocket:message-broker>

</beans>
~~~~

**对于内置的简单代理，/ topic和/ queue前缀没有任何特殊含义。 它们仅仅是区分pub-sub和点对点消息传递的惯例（即，许多订阅者与一个消费者）。 使用外部代理时，请检查代理的STOMP页面，以了解它支持的STOMP目标和前缀类型。**

要从浏览器连接，对于SockJS，您可以使用sockjs-client。 对于STOMP，许多应用程序使用了jmesnil / stomp-websocket库（也称为stomp.js），它是功能完备的，已经在生产中使用了多年但不再维护。 目前，JSteunou / webstomp-client是该库中最积极维护和不断发展的继承者。 以下示例代码基于它：

~~~~ javascript
var socket = new SockJS("/spring-websocket-portfolio/portfolio");
var stompClient = webstomp.over(socket);

stompClient.connect({}, function(frame) {
}
~~~~

或者，如果通过WebSocket连接（没有SockJS），则可以使用以下代码：

~~~~ java
var socket = new WebSocket("/spring-websocket-portfolio/portfolio");
var stompClient = Stomp.over(socket);

stompClient.connect({}, function(frame) {
}
~~~~

请注意，前面示例中的stompClient不需要指定登录和密码头。 即使它确实如此，它们也会在服务器端被忽略（或者更确切地说，被覆盖）。 有关身份验证的详细信息，请参阅连接到代理和身份验证。
有关更多示例代码，请参阅
使用WebSocket构建交互式Web应用程序 - 入门指南。
股票投资组合 - 示例应用程序。

#### 4.4.4. WebSocket Server

要配置基础WebSocket服务器，应用“服务器配置”中的信息。 但是对于Jetty，您需要通过StompEndpointRegistry设置HandshakeHandler和WebSocketPolicy：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").setHandshakeHandler(handshakeHandler());
    }

    @Bean
    public DefaultHandshakeHandler handshakeHandler() {

        WebSocketPolicy policy = new WebSocketPolicy(WebSocketBehavior.SERVER);
        policy.setInputBufferSize(8192);
        policy.setIdleTimeout(600000);

        return new DefaultHandshakeHandler(
                new JettyRequestUpgradeStrategy(new WebSocketServerFactory(policy)));
    }
}
~~~~

#### 4.4.5 消息流

一旦暴露了STOMP端点，Spring应用程序就成为连接客户端的STOMP代理。本节介绍服务器端的消息流。
Spring-messaging模块包含对源自Spring Integration的消息传递应用程序的基础支持，后来被提取并整合到Spring Framework中，以便在许多Spring项目和应用程序场景中得到更广泛的使用。以下列表简要介绍了一些可用的消息传递抽象：
消息：消息的简单表示，包括标头和有效负载。
MessageHandler：处理消息的合同。
MessageChannel：用于发送消息的合同，该消息允许生成者和使用者之间的松散耦合。
SubscribableChannel：具有MessageHandler订阅者的MessageChannel。
ExecutorSubscribableChannel：使用Executor传递消息的SubscribableChannel。
Java配置（即@EnableWebSocketMessageBroker）和XML命名空间配置（即<websocket：message-broker>）都使用前面的组件来组合消息工作流。下图显示了启用简单内置消息代理时使用的组件：

![](image/message-flow-simple-broker.png)



上图显示了三个消息通道：
clientInboundChannel：用于传递从WebSocket客户端收到的消息。
clientOutboundChannel：用于将服务器消息发送到WebSocket客户端。
brokerChannel：用于从服务器端应用程序代码中向消息代理发送消息。
下图显示了配置外部代理（例如RabbitMQ）以管理订阅和广播消息时使用的组件：

![](image/message-flow-broker-relay.png)

前两个图之间的主要区别在于使用“代理中继”通过TCP将消息传递到外部STOMP代理，以及将消息从代理传递到订阅客户端。
当从WebSocket连接接收消息时，它们被解码为STOMP帧，变为Spring消息表示，并发送到clientInboundChannel以进行进一步处理。例如，目标标头以/ app开头的STOMP消息可以路由到带注释的控制器中的@MessageMapping方法，而/ topic和/ queue消息可以直接路由到消息代理。
处理来自客户端的STOMP消息的带注释的@Controller可以通过brokerChannel向消息代理发送消息，并且代理通过clientOutboundChannel将消息广播给匹配的订户。同一个控制器也可以响应HTTP请求，因此客户端可以执行HTTP POST，然后@PostMapping方法可以向消息代理发送消息以向订阅的客户端广播。
我们可以通过一个简单的例子来追踪流程。请考虑以下示例，该示例设置服务器：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio");
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic");
    }
}

@Controller
public class GreetingController {

    @MessageMapping("/greeting") {
    public String handle(String greeting) {
        return "[" + getTimestamp() + ": " + greeting;
    }
}
~~~~

上面的示例支持以下流程：
客户端连接到http：// localhost：8080 / portfolio，一旦建立了WebSocket连接，STOMP帧就会开始流动。
客户端发送SUBSCRIBE帧，其目标头为/ topic / greeting。收到并解码后，消息将发送到clientInboundChannel，然后路由到存储客户端订阅的消息代理。
客户端向/ app / greeting发送aSEND帧。 / app前缀有助于将其路由到带注释的控制器。删除/ app前缀后，目标的剩余/问候部分将映射到GreetingController中的@MessageMapping方法。
从GreetingController返回的值变为Spring消息，其中有效负载基于返回值和/ topic / greeting的默认目标头（从/ topic替换为/ topic的输入目标派生）。生成的消息将发送到brokerChannel并由消息代理处理。
消息代理查找所有匹配的订阅者，并通过clientOutboundChannel向每个订阅者发送MESSAGE帧，消息被编码为STOMP帧并在WebSocket连接上发送。
下一节提供了有关注释方法的更多详细信息，包括支持的参数类型和返回值。

#### 4.4.6 带注释的控制器

应用程序可以使用带注释的@Controller类来处理来自客户端的消息。 这些类可以声明@MessageMapping，@ SubscribeMapping和@ExceptionHandler方法，如以下主题中所述：
@MessageMapping
@SubscribeMapping
@MessageExceptionHandler

###### @MessageMapping

您可以使用@MessageMapping来注释根据目标路由消息的方法。 它在方法级别和类型级别受支持。 在类型级别，@ MessageMapping用于表示控制器中所有方法的共享映射。
默认情况下，映射值是Ant样式的路径模式（例如/ thing *，/ thing / **），包括对模板变量的支持（例如，/ thing / {id}）。 可以通过@DestinationVariable方法参数引用这些值。 应用程序还可以切换到以点为单位的映射目标约定，如Dots as Separators中所述。
支持的方法参数
下表描述了方法参数：

| 方法参数                                                     | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `Message`                                                    | 用于访问完整的消息。                                         |
| `MessageHeaders`                                             | 用于访问`Message`中的标题。                                  |
| `MessageHeaderAccessor`, `SimpMessageHeaderAccessor`, and `StompHeaderAccessor` | 用于通过类型化访问器方法访问标头。                           |
| `@Payload`                                                   | 用于访问消息的有效负载，由已配置的`MessageConverter转换（例如，从JSON）。不需要此注释的存在，因为默认情况下，如果没有其他参数匹配则假定它。您可以注释 带有`@javax.validation.Valid`或Spring的`@ Validated`的有效负载参数，可以自动验证有效负载参数。 |
| eader`                                                       | 如有必要，用于访问特定标头值 - 以及使用`org.springframework.core.convert.converter.Converter`进行类型转换。 |
| `@Headers`                                                   | 用于访问消息中的所有标头。 该参数必须可赋值给`java.util.Map`。 |
| `@DestinationVariable`                                       | 用于访问从消息目标中提取的模板变量。 根据需要将值转换为声明的方法参数类型。 |
| `java.security.Principal`                                    | 反映WebSocket HTTP握手时登录的用户。                         |

###### 返回值

默认情况下，@MessageMapping方法的返回值通过匹配的MessageConverter序列化为有效负载，并作为消息发送到brokerChannel，从那里广播给订阅者。出站消息的目标与入站消息的目标相同，但前缀为/ topic。
您可以使用@SendTo和@SendToUser注释来自定义输出消息的目标。 @SendTo用于自定义目标目标或指定多个目标。 @SendToUser用于将输出消息仅定向到与输入消息关联的用户。请参阅用户目的地。
您可以在同一方法上同时使用@SendTo和@SendToUser，并且在类级别都支持它们，在这种情况下，它们充当类中方法的默认值。但是，请记住，任何方法级别的@SendTo或@SendToUser注释都会覆盖类级别的任何此类注释。
消息可以异步处理，@ MyMessageMapping方法可以返回ListenableFuture，CompletableFuture或CompletionStage。
请注意，@ SendTo和@SendToUser仅仅是一种便利，相当于使用SimpMessagingTemplate发送消息。如有必要，对于更高级的方案，@ MessessMapping方法可以直接使用SimpMessagingTemplate。这可以代替返回值，或者可能另外返回值。请参阅发送消息。

###### @SubscribeMapping

@SubscribeMapping类似于@MessageMapping，但仅将映射缩小为订阅消息。它支持与@MessageMapping相同的方法参数。但是，对于返回值，默认情况下，消息将直接发送到客户端（通过clientOutboundChannel，以响应订阅），而不是发送到代理（通过brokerChannel，作为匹配订阅的广播）。添加@SendTo或@SendToUser会覆盖此行为并发送给代理。
什么时候有用？假设代理被映射到/ topic和/ queue，而应用程序控制器映射到/ app。在此设置中，代理将所有订阅存储到/ topic和/ queue，用于重复广播，并且不需要应用程序参与。客户端还可以订阅某个/ app目的地，并且控制器可以返回响应于该订阅的值而不涉及代理而不再存储或使用订阅（实际上是一次性请求 - 回复交换）。一个用例是在启动时使用初始数据填充UI。
这什么时候没用？不要尝试将代理和控制器映射到相同的目标前缀，除非您由于某种原因希望两者都独立处理消息，包括订阅。入站消息是并行处理的。无法保证代理或控制器是否首先处理给定的消息。如果在存储订阅并准备好广播时通知目标，则客户端应该在服务器支持时询问收据（简单代理不支持）。例如，使用Java STOMP客户端，您可以执行以下操作来添加收据：

~~~~ java
@Autowired
private TaskScheduler messageBrokerTaskScheduler;

// During initialization..
stompClient.setTaskScheduler(this.messageBrokerTaskScheduler);

// When subscribing..
StompHeaders headers = new StompHeaders();
headers.setDestination("/topic/...");
headers.setReceipt("r1");
FrameHandler handler = ...;
stompSession.subscribe(headers, handler).addReceiptTask(() -> {
    // Subscription ready...
});
~~~~

服务器端选项是在brokerChannel上注册ExecutorChannelInterceptor，并实现在处理完消息（包括订阅）后调用的afterMessageHandled方法。

###### @MessageExceptionHandler

应用程序可以使用@MessageExceptionHandler方法来处理来自@MessageMapping方法的异常。 如果要访问异常实例，可以在注释本身或通过方法参数声明异常。 以下示例通过方法参数声明异常：

~~~~ java
@Controller
public class MyController {

    // ...

    @MessageExceptionHandler
    public ApplicationError handleException(MyException exception) {
        // ...
        return appError;
    }
}
~~~~

@MessageExceptionHandler方法支持灵活的方法签名，并支持相同的方法参数类型并返回值作为@MessageMapping方法。
通常，@ MessessExceptionHandler方法适用于声明它们的@Controller类（或类层次结构）。 如果您希望此类方法更全局地应用（跨控制器），则可以在标有@ControllerAdvice的类中声明它们。 这与Spring MVC中提供的类似支持相当。

#### 4.4.7 发送消息

如果您想从应用程序的任何部分向连接的客户端发送消息，该怎么办？ 任何应用程序组件都可以向brokerChannel发送消息。 最简单的方法是注入SimpMessagingTemplate并使用它来发送消息。 通常，您将按类型注入它，如以下示例所示：

~~~~ java
@Controller
public class GreetingController {

    private SimpMessagingTemplate template;

    @Autowired
    public GreetingController(SimpMessagingTemplate template) {
        this.template = template;
    }

    @RequestMapping(path="/greetings", method=POST)
    public void greet(String greeting) {
        String text = "[" + getTimestamp() + "]:" + greeting;
        this.template.convertAndSend("/topic/greetings", text);
    }

}
~~~~

但是，如果存在相同类型的另一个bean，您还可以通过其名称（brokerMessagingTemplate）对其进行限定。

#### 4.4.8 简单代理

内置的简单消息代理处理来自客户端的订阅请求，将它们存储在内存中，并将消息广播到具有匹配目标的连接客户端。 代理支持类似路径的目标，包括对Ant样式目标模式的预订。

**应用程序还可以使用点分隔（而不是斜线分隔）目标。 将点视为分隔符。**

如果配置了任务调度程序，则简单代理支持STOMP心跳。 为此，您可以声明自己的调度程序或使用在内部自动声明和使用的调度程序。 以下示例显示如何声明自己的调度程序：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private TaskScheduler messageBrokerTaskScheduler;

    @Autowired
    public void setMessageBrokerTaskScheduler(TaskScheduler taskScheduler) {
        this.messageBrokerTaskScheduler = taskScheduler;
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {

        registry.enableSimpleBroker("/queue/", "/topic/")
                .setHeartbeatValue(new long[] {10000, 20000})
                .setTaskScheduler(this.messageBrokerTaskScheduler);

        // ...
    }
}
~~~~

#### 4.4.9 外部代理

简单的代理非常适合入门，但仅支持STOMP命令的一部分（它不支持ack，收据和其他一些功能），依赖于简单的消息发送循环，不适合集群。 作为替代方案，您可以升级应用程序以使用功能齐全的消息代理。

请参阅STOMP文档以了解您选择的消息代理（例如RabbitMQ，ActiveMQ等），安装代理，并在启用STOMP支持的情况下运行它。 然后，您可以在Spring配置中启用STOMP代理中继（而不是简单代理）。
以下示例配置启用功能齐全的代理：

~~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableStompBrokerRelay("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
    }

}
~~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:message-broker application-destination-prefix="/app">
        <websocket:stomp-endpoint path="/portfolio" />
            <websocket:sockjs/>
        </websocket:stomp-endpoint>
        <websocket:stomp-broker-relay prefix="/topic,/queue" />
    </websocket:message-broker>

</beans>
~~~~

上述配置中的STOMP代理中继是Spring MessageHandler，它通过将消息转发到外部消息代理来处理消息。 为此，它建立与代理的TCP连接，将所有消息转发给它，然后通过其WebSocket会话将从代理接收的所有消息转发给客户端。 从本质上讲，它充当“转发”，可以在两个方向上转发消息。

**将io.projectreactor.netty：reactor-netty和io.netty：netty-all依赖项添加到项目中以进行TCP连接管理。**

此外，应用程序组件（例如HTTP请求处理方法，业务服务等）也可以向代理中继发送消息，如发送消息中所述，以向订阅的WebSocket客户端广播消息。
实际上，代理中继实现了健壮且可扩展的消息广播。

#### 4.4.10 连接到代理

STOMP代理中继维护与代理的单个“系统”TCP连接。 此连接仅用于源自服务器端应用程序的消息，而不用于接收消息。 您可以为此连接配置STOMP凭据（即STOMP帧登录和密码头）。 这在XML名称空间和Java配置中都显示为systemLogin和systemPasscode属性，其默认值为guest和guest。
STOMP代理中继还为每个连接的WebSocket客户端创建单独的TCP连接。 您可以配置用于代表客户端创建的所有TCP连接的STOMP凭据。 这在XML命名空间和Java配置中都公开为clientLogin和`clientPasscode属性，默认值为guest`和`guest`。

**STOMP代理中继始终在每个CONNECT帧上设置登录和密码头，它代表客户端转发给代理。 因此，WebSocket客户端无需设置这些标头。 他们被忽略了。 正如身份验证部分所述，WebSocket客户端应该依赖HTTP身份验证来保护WebSocket端点并建立客户端身份。**

STOMP代理中继还通过“系统”TCP连接向消息代理发送和接收心跳。 您可以配置发送和接收心跳的间隔（默认情况下每个10秒）。 如果与代理的连接丢失，代理中继将继续尝试每5秒重新连接一次，直到成功为止。
任何Spring bean都可以实现ApplicationListener <BrokerAvailabilityEvent>，以便在与代理的“系统”连接丢失并重新建立时接收通知。 例如，广播股票报价的股票报价服务可以在没有活动的“系统”连接时停止尝试发送消息。
默认情况下，STOMP代理中继始终连接，并在连接丢失时根据需要重新连接到同一主机和端口。 如果您希望提供多个地址，则在每次尝试连接时，您都可以配置地址供应商，而不是固定主机和端口。 以下示例显示了如何执行此操作：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {

    // ...

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableStompBrokerRelay("/queue/", "/topic/").setTcpClient(createTcpClient());
        registry.setApplicationDestinationPrefixes("/app");
    }

    private ReactorNettyTcpClient<byte[]> createTcpClient() {
        return new ReactorNettyTcpClient<>(
                client -> client.addressSupplier(() -> ... ),
                new StompReactorNettyCodec());
    }
}
~~~~

您还可以使用virtualHost属性配置STOMP代理中继。 此属性的值被设置为每个CONNECT帧的主机头，并且可能很有用（例如，在建立TCP连接的实际主机与提供基于云的STOMP服务的主机不同的云环境中））。

###### 4.4.11 点作为分隔符

当消息路由到@MessageMapping方法时，它们与AntPathMatcher匹配。 默认情况下，模式应使用斜杠（/）作为分隔符。 这是Web应用程序中的一个很好的约定，类似于HTTP URL。 但是，如果您更习惯于消息传递约定，则可以切换到使用点（。）作为分隔符。
以下示例显示了如何在Java配置中执行此操作：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    // ...

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setPathMatcher(new AntPathMatcher("."));
        registry.enableStompBrokerRelay("/queue", "/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }
}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:websocket="http://www.springframework.org/schema/websocket"
        xsi:schemaLocation="
                http://www.springframework.org/schema/beans
                https://www.springframework.org/schema/beans/spring-beans.xsd
                http://www.springframework.org/schema/websocket
                https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:message-broker application-destination-prefix="/app" path-matcher="pathMatcher">
        <websocket:stomp-endpoint path="/stomp"/>
        <websocket:stomp-broker-relay prefix="/topic,/queue" />
    </websocket:message-broker>

    <bean id="pathMatcher" class="org.springframework.util.AntPathMatcher">
        <constructor-arg index="0" value="."/>
    </bean>

</beans>
~~~~

之后，控制器可以使用点（。）作为@MessageMapping方法中的分隔符，如以下示例所示：

~~~~ java
@Controller
@MessageMapping("red")
public class RedController {

    @MessageMapping("blue.{green}")
    public void handleGreen(@DestinationVariable String green) {
        // ...
    }
}
~~~~

客户端现在可以向/app/red.blue.green123发送消息。
在前面的示例中，我们没有更改“代理中继”上的前缀，因为它们完全依赖于外部消息代理。 请参阅您使用的代理的STOMP文档页面，以查看它为目标标头支持的约定。
另一方面，“简单代理”依赖于配置的PathMatcher，因此，如果切换分隔符，则该更改也适用于代理以及代理将目标从消息与预订中的模式匹配的方式。

#### 4.4.12	验证

WebSocket消息传递会话中的每个STOMP都以HTTP请求开头。这可以是升级到WebSockets的请求（即WebSocket握手），或者在SockJS回退的情况下，是一系列SockJS HTTP传输请求。
许多Web应用程序已经具有用于保护HTTP请求的身份验证和授权。通常，通过使用某些机制（如登录页面，HTTP基本身份验证或其他方式）通过Spring Security对用户进行身份验证。经过身份验证的用户的安全上下文保存在HTTP会话中，并与同一个基于cookie的会话中的后续请求相关联。
因此，对于WebSocket握手或SockJS HTTP传输请求，通常已经通过HttpServletRequest＃getUserPrincipal（）访问已经过身份验证的用户。 Spring自动将该用户与为其创建的WebSocket或SockJS会话相关联，随后通过用户头与该会话上传输的所有STOMP消息相关联。
简而言之，典型的Web应用程序除了已经为安全性做的事情之外，不需要做任何事情。用户在HTTP请求级别进行身份验证，其安全上下文通过基于cookie的HTTP会话（然后与为该用户创建的WebSocket或SockJS会话相关联）进行维护，并导致在每个消息流上标记用户标头通过申请。
请注意，STOMP协议在CONNECT帧上确实有登录和密码头。这些最初设计用于并且仍然需要，例如，用于TCP上的STOMP。但是，对于STOMP over WebSocket，默认情况下，Spring会忽略STOMP协议级别的授权标头，假定用户已在HTTP传输级别进行了身份验证，并期望WebSocket或SockJS会话包含经过身份验证的用户。

**Spring Security提供WebSocket子协议授权，该授权使用ChannelInterceptor根据消息中的用户头来授权消息。 此外，Spring Session提供了WebSocket集成，可确保在WebSocket会话仍处于活动状态时用户HTTP会话不会过期。**

#### 4.4.13 令牌认证

Spring Security OAuth支持基于令牌的安全性，包括JSON Web令牌（JWT）。您可以将其用作Web应用程序中的身份验证机制，包括STOMP over WebSocket交互，如上一节所述（即通过基于cookie的会话维护身份）。
同时，基于cookie的会话并不总是最合适的（例如，在不维护服务器端会话的应用程序中，或者在通常使用标头进行身份验证的移动应用程序中）。
WebSocket协议，RFC 6455“没有规定服务器在WebSocket握手期间可以对客户端进行身份验证的任何特定方式。”但实际上，浏览器客户端只能使用标准身份验证标头（即基本HTTP身份验证）或cookie，而不能（例如）提供自定义标头。同样，SockJS JavaScript客户端不提供使用SockJS传输请求发送HTTP头的方法。请参阅sockjs-client问题196.相反，它确实允许发送可用于发送令牌的查询参数，但这有其自身的缺点（例如，令牌可能无意中使用服务器日志中的URL记录）。

**上述限制适用于基于浏览器的客户端，不适用于基于Spring Java的STOMP客户端，它支持使用WebSocket和SockJS请求发送标头。**

因此，希望避免使用cookie的应用程序可能没有任何良好的HTTP协议级别的身份验证替代方案。 他们可能更喜欢在STOMP消息传递协议级别使用标头进行身份验证而不是使用cookie。这样做需要两个简单的步骤：
使用STOMP客户端在连接时传递身份验证标头。
使用ChannelInterceptor处理身份验证标头。
下一个示例使用服务器端配置来注册自定义身份验证拦截器。 请注意，拦截器只需要在CONNECT消息上进行身份验证和设置用户头。 Spring会记录并保存经过身份验证的用户，并将其与同一会话中的后续STOMP消息相关联。 以下示例显示如何注册自定义身份验证拦截器：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class MyConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new ChannelInterceptor() {
            @Override
            public Message<?> preSend(Message<?> message, MessageChannel channel) {
                StompHeaderAccessor accessor =
                        MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
                if (StompCommand.CONNECT.equals(accessor.getCommand())) {
                    Authentication user = ... ; // access authentication header(s)
                    accessor.setUser(user);
                }
                return message;
            }
        });
    }
}
~~~~

另请注意，当您使用Spring Security的消息授权时，您需要确保在Spring Security之前订购身份验证ChannelInterceptor配置。 最好通过在自己的WebSocketMessageBrokerConfigurer实现中声明自定义拦截器来实现，该实现使用@Order（Ordered.HIGHEST_PRECEDENCE + 99）进行标记。

#### 4.4.14. User Destinations

应用程序可以发送针对特定用户的消息，Spring的STOMP支持可识别以/ user /为前缀的目标。例如，客户端可能订阅/ user / queue / position-updates目的地。此目标由UserDestinationMessageHandler处理，并转换为用户会话唯一的目标（例如/ queue / position-updates-user123）。这提供了订阅一般命名目的地的便利，同时确保不与订阅相同目的地的其他用户发生冲突，使得每个用户可以接收唯一的库存位置更新。
在发送方，可以将消息发送到目的地，例如/ user / {username} / queue / position-updates，然后由UserDestinationMessageHandler将其转换为一个或多个目的地，每个目的地一个用于与用户相关联的会话。这使得应用程序中的任何组件都可以发送针对特定用户的消息，而无需了解其名称和通用目标。通过注释和消息传递模板也支持此功能。
消息处理方法可以向与通过@SendToUser注释处理的消息相关联的用户发送消息（在类级别上也支持共享公共目标），如以下示例所示：

~~~~ java
@Controller
public class PortfolioController {

    @MessageMapping("/trade")
    @SendToUser("/queue/position-updates")
    public TradeResult executeTrade(Trade trade, Principal principal) {
        // ...
        return tradeResult;
    }
}
~~~~

如果用户具有多个会话，则默认情况下，订阅给定目标的所有会话都是目标。 但是，有时可能需要仅定位发送正在处理的消息的会话。 您可以通过将broadcast属性设置为false来执行此操作，如以下示例所示：

~~~~ java
@Controller
public class MyController {

    @MessageMapping("/action")
    public void handleAction() throws Exception{
        // raise MyBusinessException here
    }

    @MessageExceptionHandler
    @SendToUser(destinations="/queue/errors", broadcast=false)
    public ApplicationError handleException(MyBusinessException exception) {
        // ...
        return appError;
    }
}
~~~~

**虽然用户目的地通常意味着经过身份验证的用户，但并不是严格要求的。 与经过身份验证的用户无关的WebSocket会话可以订阅用户目标。 在这种情况下，@ SendToUser注释的行为与broadcast = false完全相同（即，仅定位发送正在处理的消息的会话）。**

您可以从任何应用程序组件向用户目标发送消息，例如，注入由Java配置或XML命名空间创建的SimpMessagingTemplate。 （如果需要使用@Qualifier进行限定，则bean名称为“brokerMessagingTemplate”。）以下示例说明如何执行此操作：

~~~~ java
@Service
public class TradeServiceImpl implements TradeService {

    private final SimpMessagingTemplate messagingTemplate;

    @Autowired
    public TradeServiceImpl(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    // ...

    public void afterTradeExecuted(Trade trade) {
        this.messagingTemplate.convertAndSendToUser(
                trade.getUserName(), "/queue/position-updates", trade.getResult());
    }
}
~~~~

将用户目标与外部消息代理一起使用时，应检查代理文档，了解如何管理非活动队列，以便在用户会话结束时删除所有唯一用户队列。 例如，当您使用目的地（例如/exchange/amq.direct/position-updates）时，RabbitMQ会创建自动删除队列。 因此，在这种情况下，客户端可以订阅/user/exchange/amq.direct/position-updates。 同样，ActiveMQ具有用于清除非活动目标的配置选项。

在多应用程序服务器方案中，用户目标可能仍未解析，因为用户已连接到其他服务器。 在这种情况下，您可以配置目标以广播未解析的消息，以便其他服务器有机会尝试。 这可以通过Java配置中的MessageBrokerRegistry的userDestinationBroadcast属性和XML中的message-broker元素的user-destination-broadcast属性来完成。

#### 4.4.15 消息顺序

来自代理的消息将发布到clientOutboundChannel，从那里将它们写入WebSocket会话。 由于通道由ThreadPoolExecutor支持，因此消息在不同的线程中处理，并且客户端接收的结果序列可能与发布的确切顺序不匹配
如果这是一个问题，请启用setPreservePublishOrder标志，如以下示例所示：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class MyConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    protected void configureMessageBroker(MessageBrokerRegistry registry) {
        // ...
        registry.setPreservePublishOrder(true);
    }

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:message-broker preserve-publish-order="true">
        <!-- ... -->
    </websocket:message-broker>

</beans>
~~~~

设置标志后，同一客户端会话中的消息将一次发布到clientOutboundChannel，以便保证发布顺序。 请注意，这会导致较小的性能开销，因此只有在需要时才应启用它。

#### 4.4.16. Events

发布了几个ApplicationContext事件，可以通过实现Spring的ApplicationListener接口来接收它们：
BrokerAvailabilityEvent：指示代理何时可用或不可用。虽然“简单”代理在启动时立即可用，并且在应用程序运行时仍然如此，但STOMP“代理中继”可能会失去与全功能代理的连接（例如，如果代理重新启动）。代理中继具有重新连接逻辑，并在它返回时重新建立与代理的“系统”连接。因此，只要状态从连接变为断开连接，就会发布此事件，反之亦然。使用SimpMessagingTemplate的组件应订阅此事件，并避免在代理不可用时发送消息。在任何情况下，他们都应该准备好在发送消息时处理MessageDeliveryException。
SessionConnectEvent：在收到新的STOMP CONNECT时发布，以指示新客户端会话的开始。该事件包含表示连接的消息，包括会话ID，用户信息（如果有）以及客户端发送的任何自定义标头。这对于跟踪客户端会话很有用。订阅此事件的组件可以使用SimpMessageHeaderAccessor或StompMessageHeaderAccessor包装所包含的消息。
SessionConnectedEvent：在代理已发送STOMP CONNECTED帧以响应CONNECT时，在SessionConnectEvent之后不久发布。此时，可以认为STOMP会话已完全建立。
SessionSubscribeEvent：在收到新的STOMP SUBSCRIBE时发布。
SessionUnsubscribeEvent：在收到新的STOMP UNSUBSCRIBE时发布。
SessionDisconnectEvent：STOMP会话结束时发布。 DISCONNECT可能已从客户端发送，或者可能在WebSocket会话关闭时自动生成。在某些情况下，此事件每次会话发布多次。对于多个断开连接事件，组件应该是幂等的。

**当您使用功能齐全的代理时，如果代理暂时不可用，STOMP“代理中继”会自动重新连接“系统”连接。 但是，客户端连接不会自动重新连接。 假设启用了心跳，客户端通常会注意到代理在10秒内没有响应。 客户端需要实现自己的重新连接逻辑。**

#### 4.4.17 侦听

事件提供STOMP连接生命周期的通知，但不是每个客户端消息的通知。 应用程序还可以注册ChannelInterceptor来拦截任何消息以及处理链的任何部分。 以下示例显示如何拦截来自客户端的入站邮件：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new MyChannelInterceptor());
    }
}
~~~~

自定义ChannelInterceptor可以使用StompHeaderAccessor或SimpMessageHeaderAccessor来访问有关消息的信息，如以下示例所示：

~~~~ java
public class MyChannelInterceptor implements ChannelInterceptor {

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        StompCommand command = accessor.getStompCommand();
        // ...
        return message;
    }
}
~~~~

应用程序还可以实现ExecutorChannelInterceptor，它是ChannelInterceptor的子接口，在处理消息的线程中具有回调。 虽然为发送到通道的每个消息调用一次ChannelInterceptor，但ExecutorChannelInterceptor在订阅来自通道的消息的每个MessageHandler的线程中提供挂钩。
请注意，与前面描述的SesionDisconnectEvent一样，DISCONNECT消息可以来自客户端，也可以在WebSocket会话关闭时自动生成。 在某些情况下，拦截器可能会为每个会话多次拦截此消息。 对于多个断开连接事件，组件应该是幂等的。

#### 4.4.18 STOMP客户端

Spring通过WebSocket客户端提供STOMP，通过TCP客户端提供STOMP。
首先，您可以创建和配置WebSocketStompClient，如以下示例所示：

~~~~~ java
WebSocketClient webSocketClient = new StandardWebSocketClient();
WebSocketStompClient stompClient = new WebSocketStompClient(webSocketClient);
stompClient.setMessageConverter(new StringMessageConverter());
stompClient.setTaskScheduler(taskScheduler); // for heartbeats
~~~~~

在前面的示例中，您可以将StandardWebSocketClient替换为SockJsClient，因为它也是WebSocketClient的实现。 SockJsClient可以使用WebSocket或基于HTTP的传输作为后备。 有关更多详细信息，请参阅SockJsClient。
接下来，您可以建立连接并为STOMP会话提供处理程序，如以下示例所示：

~~~~ java
String url = "ws://127.0.0.1:8080/endpoint";
StompSessionHandler sessionHandler = new MyStompSessionHandler();
stompClient.connect(url, sessionHandler);
~~~~

当会话准备好使用时，将通知处理程序，如以下示例所示：

~~~~ java
public class MyStompSessionHandler extends StompSessionHandlerAdapter {

    @Override
    public void afterConnected(StompSession session, StompHeaders connectedHeaders) {
        // ...
    }
}
~~~~

建立会话后，可以发送任何有效负载并使用配置的MessageConverter进行序列化，如以下示例所示：

~~~~ java
session.send("/topic/something", "payload");
~~~~

您也可以订阅目的地。 subscribe方法需要处理订阅消息的处理程序，并返回可用于取消订阅的订阅句柄。 对于每个收到的消息，处理程序可以指定要对其进行反序列化的目标对象类型，如以下示例所示：

~~~~ java
session.subscribe("/topic/something", new StompFrameHandler() {

    @Override
    public Type getPayloadType(StompHeaders headers) {
        return String.class;
    }

    @Override
    public void handleFrame(StompHeaders headers, Object payload) {
        // ...
    }

});
~~~~

要启用STOMP心跳，可以使用TaskScheduler配置WebSocketStompClient，并可选择自定义心跳间隔（写入不活动10秒，导致心跳发送，读取不活动10秒，关闭连接）。

**当您使用WebSocketStompClient进行性能测试以模拟来自同一台计算机的数千个客户端时，请考虑关闭心跳，因为每个连接都会调度自己的心跳任务，而不是针对在同一台计算机上运行的大量客户端进行优化。**

STOMP协议还支持收据，其中客户端必须添加收据标头，服务器在处理发送或订阅后用RECEIPT帧响应。 为了支持这一点，StompSession提供了setAutoReceipt（boolean），它会在每个后续的send或subscribe事件中添加一个收据标头。 或者，您也可以手动将收据标题添加到StompHeaders。 发送和订阅都返回一个Receiptable实例，您可以使用它来注册接收成功和失败回调。 对于此功能，您必须使用TaskScheduler和收据到期前的时间（默认为15秒）配置客户端。
请注意，StompSessionHandler本身是一个StompFrameHandler，除了处理消息的异常的handleException回调以及包含ConnectionLostException的传输级错误的handleTransportError之外，它还允许它处理ERROR帧。

#### 4.4.19 WebSocket范围

每个WebSocket会话都有一个属性映射。 映射作为标头附加到入站客户端消息，可以从控制器方法访问，如以下示例所示：

~~~~ java
@Controller
public class MyController {

    @MessageMapping("/action")
    public void handle(SimpMessageHeaderAccessor headerAccessor) {
        Map<String, Object> attrs = headerAccessor.getSessionAttributes();
        // ...
    }
}
~~~~

您可以在websocket范围中声明一个Spring管理的bean。 您可以将WebSocket范围的bean注入控制器以及在clientInboundChannel上注册的任何通道拦截器。 这些通常是单身，比任何单个WebSocket会话都活得更长。 因此，您需要为WebSocket范围的bean使用范围代理模式，如以下示例所示：\

~~~~ java
@Component
@Scope(scopeName = "websocket", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class MyBean {

    @PostConstruct
    public void init() {
        // Invoked after dependencies injected
    }

    // ...

    @PreDestroy
    public void destroy() {
        // Invoked when the WebSocket session ends
    }
}

@Controller
public class MyController {

    private final MyBean myBean;

    @Autowired
    public MyController(MyBean myBean) {
        this.myBean = myBean;
    }

    @MessageMapping("/action")
    public void handle() {
        // this.myBean from the current WebSocket session
    }
}
~~~~

与任何自定义作用域一样，Spring在第一次从控制器访问时初始化一个新的MyBean实例，并将该实例存储在WebSocket会话属性中。 随后返回相同的实例，直到会话结束。 WebSocket范围的bean调用了所有Spring生命周期方法，如前面的示例所示。

#### 4.4.20	性能

在性能方面没有银弹。许多因素会影响它，包括消息的大小和数量，应用程序方法是否执行需要阻塞的工作，以及外部因素（如网络速度和其他问题）。本部分的目标是提供可用配置选项的概述以及有关如何推理扩展的一些想法。
在消息传递应用程序中，消息通过通道进行传递，以进行由线程池支持的异步执行。配置此类应用程序需要充分了解通道和消息流。因此，建议查看消息流。
显而易见的起点是配置支持clientInboundChannel和clientOutboundChannel的线程池。默认情况下，两者都配置为可用处理器数量的两倍。
如果注释方法中的消息处理主要受CPU限制，则clientInboundChannel的线程数应保持接近处理器数。如果他们所做的工作更多地受IO限制并且需要阻塞或等待数据库或其他外部系统，则可能需要增加线程池大小。

**ThreadPoolExecutor有三个重要属性：核心线程池大小，最大线程池大小以及队列存储没有可用线程的任务的容量。
常见的混淆点是配置核心池大小（例如，10）和最大池大小（例如，20）会导致线程池具有10到20个线程。 实际上，如果容量保留为其默认值Integer.MAX_VALUE，则线程池永远不会超出核心池大小，因为所有其他任务都会排队。
请参阅ThreadPoolExecutor的javadoc以了解这些属性如何工作并理解各种排队策略。**

在clientOutboundChannel方面，它是关于向WebSocket客户端发送消息的全部内容。如果客户端位于快速网络上，则线程数应保持接近可用处理器的数量。如果它们很慢或带宽较低，则消耗消息所需的时间会更长，并给线程池带来负担。因此，增加线程池大小变得必要。
虽然clientInboundChannel的工作负载可以预测 - 毕竟，它基于应用程序的作用 - 如何配置“clientOutboundChannel”更难，因为它基于应用程序无法控制的因素。因此，另外两个属性与发送消息有关：sendTimeLimit和sendBufferSizeLimit。您可以使用这些方法配置允许发送的时间以及向客户端发送消息时可以缓冲的数据量。
一般的想法是，在任何给定时间，只有一个线程可用于发送给客户端。同时，所有其他消息都会被缓冲，您可以使用这些属性来决定允许发送消息的时间长度以及可以在此期间缓冲多少数据。有关重要的其他详细信息，请参阅XML模式的javadoc和文档。
以下示例显示了可能的配置：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
        registration.setSendTimeLimit(15 * 1000).setSendBufferSizeLimit(512 * 1024);
    }

    // ...

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:message-broker>
        <websocket:transport send-timeout="15000" send-buffer-size="524288" />
        <!-- ... -->
    </websocket:message-broker>

</beans>
~~~~

您还可以使用前面显示的WebSocket传输配置来配置传入STOMP消息的最大允许大小。 理论上，WebSocket消息的大小几乎是无限的。 实际上，WebSocket服务器施加了限制 - 例如，Tomcat上的8K和Jetty上的64K。 出于这个原因，STOMP客户端（例如JavaScript webstomp-client和其他客户端）将更大的STOMP消息拆分为16K边界，并将它们作为多个WebSocket消息发送，这需要服务器缓冲和重新组装。
Spring的STOMP-over-WebSocket支持实现了这一点，因此应用程序可以配置STOMP消息的最大大小，而不管WebSocket服务器特定的消息大小。 请记住，如有必要，WebSocket消息大小会自动调整，以确保它们至少可以携带16K WebSocket消息。
以下示例显示了一种可能的配置：

~~~~ java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
        registration.setMessageSizeLimit(128 * 1024);
    }

    // ...

}
~~~~

以下示例显示了与前面示例等效的XML配置：

~~~~ xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:message-broker>
        <websocket:transport message-size="131072" />
        <!-- ... -->
    </websocket:message-broker>

</beans>
~~~~

关于扩展的一个重点涉及使用多个应用程序实例。 目前，您无法使用简单代理执行此操作。 但是，当您使用功能齐全的代理（例如RabbitMQ）时，每个应用程序实例都会连接到代理，并且从一个应用程序实例广播的消息可以通过代理广播到通过任何其他应用程序实例连接的WebSocket客户端。

#### 4.4.21	监视

使用@EnableWebSocketMessageBroker或<websocket：message-broker>时，关键基础架构组件会自动收集统计信息和计数器，以便对应用程序的内部状态提供重要信息。该配置还声明了一个类型为WebSocketMessageBrokerStats的bean，它在一个地方收集所有可用信息，默认情况下，每隔30分钟在INFO级别收集一次。这个bean可以通过Spring的MBeanExporter导出到JMX，以便在运行时查看（例如，通过JDK的jconsole）。以下列表总结了可用信息：

###### 客户端WebSocket会话

**当前**
指示当前有多少客户端会话，其中计数通过WebSocket与HTTP流式传输和轮询SockJS会话进一步细分。
**总**
表示已建立的会话总数。
**异常关闭**
连接失败
已建立但在60秒内未收到任何消息后关闭的会话。这通常表示代理或网络问题。
**超出发送限制**
超过配置的发送超时或发送缓冲区限制后，会话关闭，这可能发生在慢客户端上（请参阅上一节）。
**运输错误**
传输错误后会话关闭，例如无法读取或写入WebSocket连接或HTTP请求或响应。

###### STOMP框架

处理的CONNECT，CONNECTED和DISCONNECT帧总数，表示STOMP级别连接的客户端数量。请注意，当会话异常关闭或客户端关闭而未发送DISCONNECT帧时，DISCONNECT计数可能会更低。
STOMP经纪人接力
**TCP连接**
指示代表客户端WebSocket会话建立到代理的TCP连接数。这应该等于客户端WebSocket会话的数量+ 1个用于从应用程序内发送消息的额外共享“系统”连接。
**STOMP框架**
代表客户端转发到代理或从代理接收的CONNECT，CONNECTED和DISCONNECT帧的总数。请注意，无论客户端WebSocket会话如何关闭，都会将DISCONNECT帧发送到代理。因此，较低的DISCONNECT帧计数表示代理主动关闭连接（可能是因为没有及时到达的心跳，无效的输入帧或其他问题）。
**客户端入站通道**
来自支持clientInboundChannel的线程池的统计信息，用于深入了解传入消息处理的运行状况。在此排队的任务表明应用程序可能太慢而无法处理消息。如果存在I / O绑定任务（例如，慢速数据库查询，对第三方REST API的HTTP请求等），请考虑增加线程池大小。
**客户出站频道**
来自支持clientOutboundChannel的线程池的统计信息，用于深入了解向客户端广播消息的运行状况。在此排队的任务表明客户端消耗消息的速度太慢。解决此问题的一种方法是增加线程池大小以适应预期的并发慢客户端数量。另一种选择是减少发送超时和发送缓冲区大小限制（参见上一节）。
**SockJS任务计划程序**
来自用于发送心跳的SockJS任务调度程序的线程池的统计信息。请注意，在STOMP级别协商心跳时，将禁用SockJS心跳。

#### 4.4.22	测试

当您使用Spring的STOMP-over-WebSocket支持时，有两种主要的方法来测试应用程序。第一种是编写服务器端测试来验证控制器的功能及其带注释的消息处理方法。第二种是编写涉及运行客户端和服务器的完整端到端测试。
这两种方法并不相互排斥。相反，每个人都在整体测试策略中占有一席之地。服务器端测试更集中，更易于编写和维护。另一方面，端到端集成测试更完整，测试更多，但它们也更多地参与编写和维护。
最简单的服务器端测试形式是编写控制器单元测试。但是，这还不够用，因为控制器的大部分功能取决于它的注释。纯单元测试根本无法测试。
理想情况下，测试中的控制器应该在运行时调用，就像测试使用Spring MVC Test框架处理HTTP请求的控制器的方法一样 - 也就是说，不运行Servlet容器，而是依赖Spring Framework来调用带注释的控制器。与Spring MVC Test一样，这里有两个可能的替代方案，使用“基于上下文”或使用“独立”设置：
在Spring TestContext框架的帮助下加载实际的Spring配置，将clientInboundChannel作为测试字段注入，并使用它发送要由控制器方法处理的消息。
手动设置调用控制器所需的最小Spring框架基础结构（即SimpAnnotationMethodMessageHandler），并将控制器的消息直接传递给它。
这两种设置方案都在股票投资组合样本应用程序的测试中得到了证明。
第二种方法是创建端到端集成测试。为此，您需要以嵌入模式运行WebSocket服务器并将其作为WebSocket客户端连接到该服务器，该客户端发送包含STOMP帧的WebSocket消息。股票投资组合示例应用程序的测试还通过使用Tomcat作为嵌入式WebSocket服务器和用于测试目的的简单STOMP客户端来演示此方法。

### 5.其他Web框架

本章详细介绍了Spring与第三方Web框架的集成。
Spring Framework的核心价值主张之一就是支持选择。 从一般意义上讲，Spring并没有强迫您使用或购买任何特定的架构，技术或方法（尽管它肯定会推荐一些其他架构，技术或方法）。 这种选择与开发人员及其开发团队最相关的架构，技术或方法的自由在Web领域最为明显，其中Spring提供了自己的Web框架（Spring MVC和Spring WebFlux），而在 同时，支持与众多流行的第三方Web框架集成

### 5.1 通用配置

在深入研究每个支持的Web框架的集成细节之前，让我们首先看一下不是特定于任何一个Web框架的常见Spring配置。 （本节同样适用于Spring自己的Web框架变体。）
Spring的轻量级应用程序模型支持的一个概念（想要一个更好的词）是一个分层架构。请记住，在“经典”分层架构中，Web层只是众多层中的一个。它作为服务器端应用程序的入口点之一，并委托给服务层中定义的服务对象（外观），以满足特定于业务（和表示技术不可知）的用例。在Spring中，这些服务对象，任何其他特定于业务的对象，数据访问对象和其他对象存在于不同的“业务上下文”中，该业务上下文不包含Web或表示层对象（表示对象，例如Spring MVC控制器，通常是在不同的“演示文稿上下文”中配置。本节详细介绍了如何配置包含应用程序中所有“业务bean”的Spring容器（WebApplicationContext）。
转到具体细节，您需要做的只是在Web应用程序的标准Java EE servlet web.xml文件中声明ContextLoaderListener，并添加contextConfigLocation <context-param />部分（在同一文件中），定义哪个集合要加载的Spring XML配置文件。
请考虑以下<listener />配置：

~~~~ xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
~~~~

进一步考虑以下<context-param />配置：

~~~~ xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/applicationContext*.xml</param-value>
</context-param>
~~~~

如果未指定contextConfigLocation上下文参数，则ContextLoaderListener将查找要加载的名为/WEB-INF/applicationContext.xml的文件。 加载上下文文件后，Spring会根据bean定义创建一个WebApplicationContext对象，并将其存储在Web应用程序的ServletContext中。
所有Java Web框架都构建在Servlet API之上，因此您可以使用以下代码片段来访问ContextLoaderListener创建的“业务上下文”ApplicationContext。
以下示例显示如何获取WebApplicationContext：

~~~~ java
WebApplicationContext ctx = WebApplicationContextUtils.getWebApplicationContext(servletContext);
~~~~

WebApplicationContextUtils类是为了方便起见，因此您无需记住ServletContext属性的名称。 如果WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE键下不存在对象，则其getWebApplicationContext（）方法返回null。 不要冒险在应用程序中获取NullPointerExceptions，最好使用getRequiredWebApplicationContext（）方法。 当ApplicationContext丢失时，此方法抛出异常。
一旦引用了WebApplicationContext，就可以按名称或类型检索bean。 大多数开发人员按名称检索bean，然后将其转换为其实现的接口之一。
幸运的是，本节中的大多数框架都有更简单的查找bean的方法。 它们不仅可以轻松地从Spring容器中获取bean，而且还允许您在其控制器上使用依赖注入。 每个Web框架部分都有关于其特定集成策略的更多详细信息。

### 5.2. JSF

JavaServer Faces（JSF）是JCP标准的基于组件，事件驱动的Web用户界面框架。 它是Java EE保护伞的官方部分，但也可单独使用，例如， 通过在Tomcat中嵌入Mojarra或MyFaces。
请注意，最近版本的JSF与应用程序服务器中的CDI基础结构紧密相关，一些新的JSF功能仅在这样的环境中工作。 Spring的JSF支持不再主动发展，主要用于迁移目的，以便对基于JSF的旧应用程序进行现代化改造。
Spring的JSF集成中的关键元素是JSF ELResolver机制。

#### 5.2.1. Spring Bean Resolver

SpringBeanFacesELResolver是一个符合JSF的ELResolver实现，与JSF和JSP使用的标准Unified EL集成。 它首先委托Spring的“业务上下文”WebApplicationContext，然后委托给底层JSF实现的默认解析器。
在配置方面，您可以在JSF faces-context.xml文件中定义SpringBeanFacesELResolver，如以下示例所示：

~~~~ xml
<faces-config>
    <application>
        <el-resolver>org.springframework.web.jsf.el.SpringBeanFacesELResolver</el-resolver>
        ...
    </application>
</faces-config>
~~~~

#### 5.2.2 使用FacesContextUtils

在faces-config.xml中将属性映射到bean时，自定义ELResolver运行良好，但有时，您可能需要显式获取bean。 FacesContextUtils类使这很容易。 它类似于WebApplicationContextUtils，除了它采用FacesContext参数而不是ServletContext参数。
以下示例显示如何使用FacesContextUtils：

~~~~ java
ApplicationContext ctx = FacesContextUtils.getWebApplicationContext(FacesContext.getCurrentInstance());
~~~~

### 5.3	Apache Struts 2.x

Struts由Craig McClanahan发明，是一个由Apache Software Foundation主持的开源项目。当时，它大大简化了JSP / Servlet编程范例，并赢得了许多使用专有框架的开发人员。它简化了编程模型，它是开源的（因此在啤酒中是免费的），它有一个庞大的社区，让项目成长并在Java Web开发人员中变得流行。
作为原始Struts 1.x的后续版本，请查看Struts 2.x和Struts提供的Spring插件，以实现内置的Spring集成。

### 5.4 Apache Tapestry 5.x

Tapestry是一个“”面向组件的框架，用于在Java中创建动态，健壮，高度可扩展的Web应用程序。“
虽然Spring拥有自己强大的Web层，但是通过将Tapestry用于Web用户界面和Spring容器用于较低层，构建企业Java应用程序有许多独特的优势。
有关更多信息，请参阅Tapestry针对Spring的专用集成模块。

### 5.5 更多资源

以下链接涉及有关本章中描述的各种Web框架的更多资源。
JSF主页
Struts主页
Tapestry主页
