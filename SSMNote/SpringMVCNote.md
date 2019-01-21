# SpringMVC   
 执行逻辑图：    
 ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/springMVC.png)
###  DispatcherServlet   
  前端控制器   
  整个流程控制的中心，由它调用其他组件处理用户的请求。DispatcherServlet的存在降低了组件之间的耦合性。   
###  HandlerMapping   
  处理映射器   
* 负责根据用户请求找到 handler(处理器，如：用户自定义的Controller）,springmvc提供了不同的映射器实现不同的映射方式，例如：配置文件，注解方式等。   
* 映射器相当于配置信息或注解描述，内部封装一个类似map的数据结构，使用Url作为Key,HandlerExecutionChain作为value。核心控制器，可以通过请求对象（请求对象
中包含URL），在handlerMapping中查询HandlerExecutionChain对象。     
* 是SpringMVC核心组件之一，是必不可少的组件。无论是否配置，SpringMVC会有默认提供（RequestMappingHandlerMapping）。     
### HandlerAdapter  
  适配器   
* 通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。   
* 典型的适配器，SimpleControllerHandlerAdapter。最基础的，处理自定义控制器（handler）和SpringMVC控制器顶级接口Controller之间关联的。   
### Handler    
  是继DispatcherServlet前端控制器的后端控制器（自定义控制器），在DispatcherServlet的控制下Handler对具体的用户请求进行处理，由于Handler涉及具体的用户请求，
所以需要程序员根据具体的业务开发Handler。   
  springMVC框架是线程不安全的，一个Handler对象，处理所有的请求（单实例）。所以要定义服务类型变量，如    
```JAVA    
  //服务类型变量是提供业务逻辑的，是只读的，是根据输入信息计算返回输出结果的。
  //不是用于记录客户端状态的   
  private xxxService xxxservice;    
  
  //另一种：定义静态常量，一旦复制不可改变，也是只读数据，无关客户端状态   
  private | public static final xxx;

```
### ViewResolver 
 负责将处理结果生成View视图,ViewResolver首先根据逻辑视图名解析成物理视图名即具体的页面地址，再生成view对象，最后对view进行渲染将处理结果通过页面展示给用户.视图解析器是用来处理动态视图逻辑的。静态视图逻辑，不通过SpringMVC流程。直接通过WEB中间件（tomcat）就可以访问静态资源。
