# Web容器

## Servlet规范

HTTP服务器不直接调用 业务类，而是把请求交给容器来处理，容器通过Servlet接口调用业务类。因此Servlet接口和Servlet容器的 出现，达到了HTTP服务器与业务类解耦的目的。

而Servlet接口和Servlet容器这一整套规范叫作Servlet规范

## Servlet生命周期

1. init
2. service
3. destroy

## Servlet扩展

1. Filter
2. Listener
   1. 比如Spring就实现了自己的监听器，来监听ServletContext的 启动事件，目的是当Servlet容器启动时，创建并初始化全局的Spring容器。

### 区别

* Filter是干预过程的，它是过程的一部分，是基于过程行为的。
* Listener是基于状态的，任何行为改变同一个状态，触发的事件是一致的。

## Tomcat系统架构

### 连接器是如何设计的

### Valve vs Filter

* Valve是Tomcat的私有机制，与Tomcat的基础架构/API是紧耦合的。Servlet API是公有的标准，所有的 Web容器包括Jetty都支持Filter机制。

* 另一个重要的区别是Valve工作在Web容器级别，拦截所有应用的请求;而Servlet Filter工作在应用级别， 只能拦截某个Web应用的所有请求。如果想做整个Web容器的拦截器，必须通过Valve来实现。

### 问题

1. Tomcat内的Context组件跟Servlet规范中的ServletContext接口有什么区别?跟Spring中的 ApplicationContext又有什么关系?
   1. Servlet规范中ServletContext表示web应用的上下文环境，而web应用对应tomcat的概念是Context， 所以从设计上，ServletContext自然会成为tomcat的Context具体实现的一个成员变量。

## Java NIO

1. 三个关键组件
   1. Channel
   2. Buffer
   3. Selector

## 自定义Web容器

1. 通过ServletContextInitializer接口可以向Web容器注册Servlet

2. 可以通过`WebServlet`注解直接在web容器中注入Servlet，这样就绕过了spring的那一套东西了

3. 定制web容器

   1. 通过ConfigurableServletWebServerFactory

   ```java
   @Component
     public class MyGeneralCustomizer implements
       WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
         public void customize(ConfigurableServletWebServerFactory factory) {
             factory.setPort(8081);
             factory.setContextPath("/hello");
   } }
   ```

   2. 通过特定Web容器的工厂比如TomcatServletWebServerFactory来进一步定制。下面的例子 里，我们给Tomcat增加一个Valve，这个Valve的功能是向请求头里添加traceid，用于分布式追踪。
      
   1. TraceValve的定义如下:
      
         ```java
         class TraceValve extends ValveBase {
               @Override
               public void invoke(Request request, Response response) throws IOException, ServletException {
                   request.getCoyoteRequest().getMimeHeaders().
                   addValue("traceid").setString("1234xxxxabcd");
                   Valve next = getNext();
                   if (null == next) { return; }
                   next.invoke(request, response);
               }
         }
      ```
      
   2. 再添加一个定制器，代码如下:	
      
         ```java
         @Component
         public class MyTomcatCustomizer implements
                   WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
               @Override
               public void customize(TomcatServletWebServerFactory factory) {
                   factory.setPort(8081);
                   factory.setContextPath("/hello");
                   factory.addEngineValves(new TraceValve() );
         } }
      ```
      
         
   
4. spring boot中配置tomcat使用nio2

   ```java
   @Configuration
   public class TomcatConfig {
       @Bean
       public EmbeddedServletContainerFactory servletContainer1() {
           TomcatEmbeddedServletContainerFactory tomcat = new TomcatEmbeddedServletContainerFactory();
           tomcat.setUriEncoding(Charset.forName("UTF-8"));
           /*通过addAdditionalTomcatConnectors方法添加多个监听连接;*/
           tomcat.addAdditionalTomcatConnectors(createNioConnector1());
           return tomcat;
       }
       public Connector createNioConnector1(){
           Connector connector=new Connector("org.apache.coyote.http11.Http11Nio2Protocol");
           Http11Nio2Protocol protocol = (Http11Nio2Protocol) connector.getProtocolHandler();
           // 设置超时时间
           protocol.setConnectionTimeout(3000);
           // 设置最大线程数
           protocol.setMaxThreads(200);
           // 设置最大连接数
           protocol.setMaxConnections(1000);
           // 请求方式
           connector.setScheme("http");
           
           connector.setPort(8015);                    //自定义的
           connector.setRedirectPort(8443);
           return connector;
       }
   }
   ```

   

