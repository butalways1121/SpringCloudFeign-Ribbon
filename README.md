# SpringCloudFeign-Ribbon
服务消费者(Feign)和负载均衡(Ribbon)功能的实现以及使用Feign结合Ribbon实现负载均衡
---
**本篇主要介绍的是SpringCloud中的服务消费者(Feign)和负载均衡(Ribbon)功能的实现以及使用Feign结合Ribbon实现负载均衡。戳[这里](https://github.com/butalways1121/SpringCloudFeign-Ribbon)下载源码。**
<!-- more -->
## 一、SpringCloud Feign
### 1.SpringCloud Feign简介
&emsp;&emsp;SpringCloud Feign是一个声明式的web service客户端，它让微服务之间的调用变得更简单了，类似controller调用service。在Spring Cloud中，使用Feign非常简单--创建一个接口，并在接口上添加一些注解，代码就完成了。Feign支持多种注解，例如Feign自带的注解或者JAX-RS注解等。Spring Cloud对Feign进行了增强，使Feign支持了Spring MVC注解，同时也集成了Ribbon和Eureka来提供均衡负载的HTTP客户端实现，让Feign的使用更加方便。

&emsp;&emsp;何谓声明式：Spring Cloud的声明式调用, 可以做到使用 HTTP请求远程服务时能就像调用本地方法一样的体验，开发者完全感知不到这是远程方法，更感知不到这是个HTTP请求。consumer直接调用接口方法调用provider，而不需要通过常规的Http Client构造请求再解析返回数据。它解决了让开发者调用远程接口就跟调用本地方法一样，无需关注与远程的交互细节，更无需关注分布式环境开发。
### 2.SpringCloud Feign示例
#### （1）服务端
&emsp;&emsp;首先建立一个springcloud-feign-eureka服务的Maven Project，pom.xml的配置如下，同样需要注意的是根据SpringBoot的版本添加Eureka的依赖，如下：
```bash
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>1.0</groupId>
	<artifactId>springcloud-eureka-feign</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>springcloud-eureka-feign</name>
	<url>http://maven.apache.org</url>

	<!-- Spring Boot启动器父类 -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath />
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Dalston.RELEASE</spring-cloud.version>
	</properties>

	<dependencies>
		<!-- eureka依赖 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
			<!-- springboot2.x以上 -->
			<!-- <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId> -->
		</dependency>
		<!-- Spring Boot Test 依赖 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<!-- feign依赖 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-feign</artifactId>
			<!-- <artifactId>spring-cloud-starter-openfeign</artifactId> -->
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```
application.properties文件的配置信息：
```bash
spring.application.name=springcloud-feign-eureka-server
server.port=8001
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8001/eureka/
```
启动类代码：
```bash
@SpringBootApplication
@EnableEurekaServer
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "feign注册中心服务启动..." );
    }
}
```
#### （2）客户端
&emsp;&emsp;这里定义两个消费者，即创建两个Maven Project项目springcloud-feign-consumer和springcloud-feign-consumer2，前者使用Feign做转发，后者就是一个普通的项目，两个项目的pom.xml同springcloud-feign-eureka即可。

##### 1）springcloud-feign-consumer项目

application.properties文件的配置信息如下：
```bash
spring.application.name=springcloud-feign-consumer
server.port=9002
eureka.client.serviceUrl.defaultZone=http://localhost:8001/eureka/
```
因为该项目要实现Fegin的功能，因此需要在启动类上添加@EnableFeignClients注解，表示启用Feign进行远程调用，如下：
```bash
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println("feign第一个消费者服务启动...");
    }
}
```
接着创建一个转发类把请求转发到另外一个服务，需要使用`@FeignClient`注解来实现，表示需要转发服务的名称，这里把请求转发到第二个服务springcloud-feign-consumer2，**需要注意的是这里转发服务的`@RequestMapping(value = "/helloworld")`要与第二个消费服务springcloud-feign-consumer2中的提供打印信息接口的`@RequestMapping("/helloworld")`保持一致**：
```bash
@FeignClient(name = "springcloud-feign-consumer2")
public interface HelloRemote {
	@RequestMapping(value = "/helloworld")
    //@RequestParam主要用于将请求参数区域的数据映射到控制层方法的参数上,即获取前端参数
	public String hello(@RequestParam(value = "name") String name);
}
```
另外，还需要提供一个入口供外部调用，然后调用上述的方法将请求进行转发：
```bash
@RestController
public class ConsumerController {
	@Autowired
	HelloRemote helloRemote;
	@RequestMapping("/hello/{name}")
    //可以将URL中占位符参数{xxx}绑定到处理器类的方法形参中@PathVariable("xxx")
    public String index(@PathVariable("name") String name) {
    	System.out.println("接受到请求参数:"+name+",进行转发到其他服务");
        return helloRemote.hello(name);
    }
}
```
##### 2）springcloud-feign-consumer2项目

这个服务的代码就比较简单，只是一个普通的项目，提供一个接口然后打印信息即可。

application.properties文件的配置信息如下：
```bash
spring.application.name=springcloud-feign-consumer2
server.port=9003
eureka.client.serviceUrl.defaultZone=http://localhost:8001/eureka/
```
启动类代码：
```bash
@SpringBootApplication
@EnableEurekaClient
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "feign第二个消费者服务启动..." );
    }
}
```
提供打印信息接口的控制层代码示例如下：
```bash
@RestController
public class ConsumerController {

	@RequestMapping("/helloworld")
	public String index(@RequestParam String name) {
        return name+",Hello World";
    }
}
```
#### （3）测试
&emsp;&emsp;完成如上三个项目之后，依次启动服务端和客户端的三个程序，然后在浏览器里输入:`http://localhost:8001/`，即可查看注册中心的信息，如下图，可以看到客户端的两个服务已经注册到Eureka服务端了：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/75.png)

然后在浏览器中输入`http://localhost:9003/hello?name=butalways`直接调用第二个消费服务springcloud-feign-consumer2的接口，浏览器界面回返回`butalways,Hello World`：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/76.png)

接着在浏览器中再输入`http://localhost:9002/hello/butalways`，来通过第一个消费服务springcloud-feign-consumer调用第二个服务springcloud-feign-consumer2，此时控制台会打印`接受到请求参数:butalways,进行转发到其他服务`，浏览器界面返回`butalways,Hello World`：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/77.png)

&emsp;&emsp;若结果如上，则说明客户端已经成功的通过Feign调用了远程服务hello，并且将结果返回到了浏览器。
## 二、SpringCloud Ribbon
### 1.SpringCloud Ribbon简介
&emsp;&emsp;Spring Cloud Ribbon是Netflix发布的开源项目，是一个基于Http和TCP的客服端负载均衡工具。Ribbon客户端组件提供一系列完善的配置项如连接超时、重试等。与Eureka配合使用时，Ribbon可自动从Eureka Server (注册中心)获取服务提供者地址列表，并基于负载均衡算法，通过在客户端中配置ribbonServerList来设置服务端列表去轮询访问以达到均衡负载的作用。简单的说，就是在配置文件中列出Load Balancer后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询、随即连接等）去连接这些机器，我们也很容易使用Ribbon实现自定义的负载均衡算法。简单地说，Ribbon是一个客户端负载均衡器。

&emsp;&emsp;使用负载均衡带来的好处很明显：当集群里的1台或者多台服务器down的时候，剩余的没有down的服务器可以保证服务的继续使用
使用了更多的机器保证了机器的良性使用，不会由于某一高峰时刻导致系统cpu急剧上升。

&emsp;&emsp;另外，在这里简单说一下Ribbon提供的7中负载均衡策略，如下图：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/78.png)

Ribbon默认使用的策略是轮询，想要更改的话可以在项目的application.properties中修改，如想将springcloud-ribbon-consumer服务的策略指定为“随机策略”，只需在配置中添加`springcloud-ribbon-consumer.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RandomRule`即可。
### 2.SpringCloud Ribbon示例
#### （1）服务端
&emsp;&emsp;首先建立一个springcloud-ribbon-eureka服务的Maven Project，用于做注册中心，在pom.xml文件中添加相关配置信息：
```bash
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>1.0</groupId>
	<artifactId>springcloud-eureka-ribbon</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>springcloud-eureka-ribbon</name>
	<url>http://maven.apache.org</url>

	<!-- Spring Boot启动器父类 -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath />
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Dalston.RELEASE</spring-cloud.version>
	</properties>

	<dependencies>
		<!-- eureka依赖 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
			<!-- springboot2.x以上 -->
			<!-- <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId> -->
		</dependency>
		<!-- Spring Boot Test 依赖 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<!-- 监控系统健康情况的工具 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<!-- ribbon依赖 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-ribbon</artifactId>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```
application.properties文件配置为：
```bash
spring.application.name=springcloud-ribbon-eureka-server
server.port=8003
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8003/eureka/
```
启动类代码示例：
```bash
@EnableEurekaServer
@SpringBootApplication
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "ribbon注册中心启动" );
    }
}
```
#### （2）客户端
&emsp;&emsp;这里我们定义三个服务，一个服务springcloud-ribbon-consumer使用Ribbon做负载均衡，另外两个springcloud-ribbon-consumer2和springcloud-ribbon-consumer3做普通的服务，三个同样均为Maven Project项目，pom.xml文件同springcloud-ribbon-eureka即可。
##### 1）springcloud-ribbon-consumer
application.properties文件配置信息为：
```bash
spring.application.name=springcloud-ribbon-consumer
server.port=9006
eureka.client.serviceUrl.defaultZone=http://localhost:8003/eureka/
```
该服务要使用Ribbon实现负载均衡，只需要在启动类中实例化RestTemplate，通过`@LoadBalanced`注解开启均衡负载能力即可，启动类代码：
```bash
@SpringBootApplication
@EnableDiscoveryClient
public class RibbonConsumerApplication {
	public static void main(String[] args) {
		SpringApplication.run(RibbonConsumerApplication.class, args);
		System.out.println("ribbon第一个消费者服务启动...");
	}
	@Bean
	@LoadBalanced
	public RestTemplate restTemplate() {
		return new RestTemplate();
	}
}
```
接着需要定义转发的服务，这里使用RestTemplate来调用另外一个服务springcloud-ribbon-consumer2：
```bash
@RestController
public class ConsumerController {
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/hello")
    public String hello() {
    	//进行远程调用
        return restTemplate.getForObject("http://springcloud-ribbon-consumer2/hello/?name=butalways", String.class);
    }
}
```
##### 2）springcloud-ribbon-consumer2
application.properties文件配置信息如下，这里指定服务名为springcloud-ribbon-consumer中需要转发到的服务名springcloud-ribbon-consumer2：
```bash
spring.application.name=springcloud-ribbon-consumer2
server.port=9007
eureka.client.serviceUrl.defaultZone=http://localhost:8003/eureka/
```
启动类代码为：
```bash
@SpringBootApplication
@EnableDiscoveryClient
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "ribbon第二个消费者服务启动..." );
    }
}
```
该服务只是一个普通的服务，再提供一个接口打印信息即可：
```bash
@RestController
public class ConsumerController {

	@RequestMapping("/hello")
	public String index(@RequestParam String name) {
		return name+",Hello World!";
	}
}
```
##### 3）springcloud-ribbon-consumer3
同服务springcloud-ribbon-consumer2一样，springcloud-ribbon-consumer3也只是一个普通的服务，只需提供一个接口打印信息即可。application.properties文件配置信息如下，这里指定的服务名要和springcloud-ribbon-consumer2指定的一样，这样springcloud-ribbon-consumer在转发服务时才可实现负载均衡：
```bash
spring.application.name=springcloud-ribbon-consumer2
server.port=9008
eureka.client.serviceUrl.defaultZone=http://localhost:8003/eureka/
```
控制层代码：
```bash
@SpringBootApplication
@EnableDiscoveryClient
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "ribbon第三个消费者服务启动..." );
    }
}
```
打印信息接口类代码：
```bash
@RestController
public class ConsumerController {
	
	@RequestMapping("/hello")
	public String index(@RequestParam String name) {
		return name+",Hello World!这是另一个服务！";
	}
}
```
#### （3）测试
&emsp;&emsp;完成之后依次启动服务端和客户端的四个程序，然后在浏览器输入`http://localhost:8003/`即可查看注册中心的信息，可以看到服务端的三个服务都已注册：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/79.png)
接着，在浏览器中输入`http://localhost:9006//hello`请求springcloud-ribbon-consumer的方法，并重复访问，浏览器返回结果为：
```bash
butalways,Hello World!
butalways,Hello World! 这是另一个服务！
butalways,Hello World!
butalways,Hello World! 这是另一个服务！
butalways,Hello World!
butalways,Hello World! 这是另一个服务！
```
至此，说明已经实现了负载均衡功能，而且是采用默认的轮询策略来回调用两个服务名相同的服务。
## 三、SpringCloud Fegin结合Ribbon实现负载均衡
&emsp;&emsp;Fegin包含了Ribbon，可以直接实现负载均衡功能，要完成Fegin结合Ribbon实现负载均衡，只需简单修改上述项目springcloud-ribbon-consumer即可实现远程调用服务，修改内容如下：

1.首先，在pom.xml中添加Feign依赖：
```bash
<!-- feign依赖 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-feign</artifactId>
			<!-- <artifactId>spring-cloud-starter-openfeign</artifactId> -->
		</dependency>
```
2.然后在启动类上添加`@EnableFeignClients`注解，表示启用Feign进行远程调用。

3.接着新建一个类，实现Feign远程调用：
```bash
@FeignClient(name= "springcloud-ribbon-consumer2") 
public interface HelloRemote {

	@RequestMapping(value = "/hello")
    public String hello(@RequestParam(value = "name") String name);
}
```
4.最后在提供一个新的接口供外部调用，在原来的控制层上添加如下接口即可：
```bash
@Autowired
HelloRemote helloRemote;
	
@RequestMapping("/hello/{name}")
public String index(@PathVariable("name") String name) {
    System.out.println("接受到请求参数:"+name+",进行转发到其他服务!");
    return helloRemote.hello(name);
    }
```
&emsp;&emsp;完成如上修改之后，重新启动服务端和客户端的四个程序，在浏览器中重复访问` http://localhost:9006//hello/butalways`，结果应为：
```bash
butalways,Hello World!
butalways,Hello World! 这是另一个服务！
butalways,Hello World!
butalways,Hello World! 这是另一个服务！
butalways,Hello World!
butalways,Hello World! 这是另一个服务！
```
至此，SpringCloud Fegin结合Ribbon实现负载均衡功能已完成。
