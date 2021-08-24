tomcat
第1章 一个简单的web服务器
1.创建客户端socket去访问服务端
2.创建服务端ServerSocket，进行监听。具体为await方法对齐进行创建，等待accept()返回并循环。接收到请求后accept()可以获取InputStream和OutputStream。并创建new Request(input)，request.parse()进行解析。解析完毕后会new Response(output),接着response.setRequest(request)、response.sendStaticReponse()如果uri请求的路径存在则返回该静态资源并写入outputStream,若不存在则返回错误消息。最后判断是否结束循环根据uri关键字是否关闭。

第2章 一个简单的servlet容器
Servlet接口中声明了5个方法，方法签名如下:
public coid init(ServletConfig config) throws ServletException
public void service(ServletRequest request, ServletResponse response) throws ServletException, java.io.IOException
public void destroy()
public ServletConfig getServletConfig()
public String getServletInfo()
其中init()\service()和destroy()是与servlet的生命周期相关的方法。当实例化某个servlet容器后悔调用其init()方法进行初始化。这个也是在引入springMVC与Struts2之前时需要实现的类,service()就是主要逻辑。即使是现在，在用到request和response时也可以看到期继承了servletRequest和servletReponse。为了动态的jsp技术则需要URLClassLoader的loadClass()动态载入servlet类

第3章 连接器
Catailna中有两个主要的模块，连接器(connector)和容器(container)
StringManager，像tomcat这种国际化程序，其对语言也是有一定支持的，方便admin管理。如果将异常全部放在一个properties,那将是一场噩梦,其采用了分包的方式，将properties放入不同的包中
连接器程序主要包含3各模块:连机器模块、启动模块和核心模块。
	启动模块只有一个类(Bootstrap)
	连接器模块可分为5个类型
		连接器机器支持类(HttpConector和HttpProcessor)
		HTTP请求类(HttpRequest)
		HTTP响应类(HttpResponse)
		外观类(HttpRequestFacade和HttpResponseFacade)
		常量类
	核心模块包含两个类,servletProcessor和StaticProcessor

连接器就是起了个服务，然后等待监听，来一个就创建HttpRequest和HttpResponse。将对象传递给servletProcessor或StaticResourceProcessor的process()方法
第4章tomcat默认连接器
第5章serrvlet容器
对于Catalina中的容器，需要注意的是共有4中类型的容器，分别对应不同的概念层次
Engine:表示Catalina servlet引擎
Host:表示一个或多个Context容器的虚拟主机
Context:表示一个Web应用程序。一个Context可以有多个Wrapper
Wrapper:表示一个独立的servlet
上述的每个概念层级有org.apache.catalina包内的一个接口表示，并且都集成子Container接口
第6章生命周期
第7章日志记录器
第8章载入器
第9章session管理
第10章安全性
第11章StandardWrapper
第12章StandardContext
第13章Host和Engine
第14章服务器组件和服务组件
第15章Digester
第16章关闭钩子
第17章启动Tomcat
第18章部署器
第19章Manager应用程序的servlet
第20章基于JMX的管理
