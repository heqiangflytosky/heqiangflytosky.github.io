---
title: 服务端开发入门 -- 使用 Intellij IDEA 创建 Maven + SpringMVC 项目
categories: Web Server
comments: true
tags: [SpringMVC]
description: 使用 Intellij IDEA 创建 Maven + SpringMVC 项目
date: 2019-5-2 10:00:00
---

## 创建工程

打开 Intellij IDEA ，选择 File -> New -> Project ...，弹出下面的对话框：

<img src="/images/web-server-maven-spring-mvc-new-project/project-new.png" />

按照图中选择 Maven + 模板选项：maven-archetype-webapp，然后点击下一步。
填写 GroupId 和 ArtifactId，

<img src="/images/web-server-maven-spring-mvc-new-project/project-groupid.png"/>

点击下一步，是创建 Maven 的环境：

<img src="/images/web-server-maven-spring-mvc-new-project/project-maven.png" />

点击下一步，这里项目名称自动继承前面设置的 ArtifactId。
然后点击完成，会生成一个工程，目录结构如下。


<img src="/images/web-server-maven-spring-mvc-new-project/project-dir.png" />

## 创建Server

下面我们来创建一个服务器来验证一下Web环境。

首先点击项目右上角的 Add Configuration 或者选择 Run -> Edit Configurations，弹出对话框中点击 + 号：

<img src="/images/web-server-maven-spring-mvc-new-project/server-add.png" />

选择 Tomcat 或者 Jetty 服务器。
先在 Server 选项卡中选择 Application Server 的目录，然后切换到 Deployment 选项卡，点击 + 号，按照图中选择，然后点击OK。

<img src="/images/web-server-maven-spring-mvc-new-project/server-deploment.png" />

<img src="/images/web-server-maven-spring-mvc-new-project/server-deploment-2.png"  width="400" height="380"/>

这样 Web 服务器环境就配置好了，然后我们启动服务。点击工具栏上面的运行按钮。
运行成功后自动在浏览器打开 index.jsp 页面。



## 配置 SpringMVC


我们再加入 Spring MVC 的配置。
在 main 目录下面创建两个目录：java（用来放源代码），resources（用来放资源配置文件）。
通过下面的方式分别设置为 sources root 和 resources root


<img src="/images/web-server-maven-spring-mvc-new-project/spring-root.png" width="600" height="600"/>


然后再添加 Spring 的相关依赖，把下面的配置添加到 pom.xml 文件中：


```
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
      <version>1.7.21</version>
    </dependency>

    <!--J2EE-->
    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>javax.servlet-api</artifactId>
      <version>3.1.0</version>
    </dependency>

    <dependency>
      <groupId>javax.servlet.jsp</groupId>
      <artifactId>jsp-api</artifactId>
      <version>2.2</version>
    </dependency>

    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>jstl</artifactId>
      <version>1.2</version>
    </dependency>

    <!--mysql驱动包-->
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.35</version>
    </dependency>

    <!--springframework-->
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-web</artifactId>
      <version>4.2.6.RELEASE</version>
    </dependency>

    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>4.2.6.RELEASE</version>
    </dependency>

    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-context</artifactId>
      <version>4.2.6.RELEASE</version>
    </dependency>

    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-test</artifactId>
      <version>4.2.6.RELEASE</version>
    </dependency>

    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-jdbc</artifactId>
      <version>4.2.6.RELEASE</version>
    </dependency>

    <dependency>
      <groupId>com.github.stefanbirkner</groupId>
      <artifactId>system-rules</artifactId>
      <version>1.16.1</version>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>org.aspectj</groupId>
      <artifactId>aspectjweaver</artifactId>
      <version>1.8.9</version>
    </dependency>

    <!--其他需要的包-->
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-lang3</artifactId>
      <version>3.4</version>
    </dependency>

    <dependency>
      <groupId>commons-fileupload</groupId>
      <artifactId>commons-fileupload</artifactId>
      <version>1.3.1</version>
    </dependency>

    <!--添加java对象注解转json支持-->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-core</artifactId>
      <version>2.8.5</version>
    </dependency>

    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.8.5</version>
    </dependency>

    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-annotations</artifactId>
      <version>2.8.5</version>
    </dependency>
```

然后点击 IDE 右下角弹框中的 Import Changes，会自动下载和引入这些依赖。

<img src="/images/web-server-maven-spring-mvc-new-project/spring-import.png" />

接下来添加 Spring 的配置文件。

右键点击工程，然后选择 Add FrameWork Support ...，选择 Spring MVC：

<img src="/images/web-server-maven-spring-mvc-new-project/spring-add-framework-1.png" />

如果找不到 Spring 选项，表示项目中已经存在Spring项目相关文件，但不一定代表该项目中Spring MVC 的文件是完善的。就打开 Project Structure，选择 Module，把 Spring 删掉。

<img src="/images/web-server-maven-spring-mvc-new-project/spring-add-framework-2.png" />

创建完成后，工程中多出了 applicationContext.xml 和 dispatcher-servlet.xml 配置文件。web.xml  中也增加了一下配置。

<img src="/images/web-server-maven-spring-mvc-new-project/spring-add-framework-3.png" width="400" height="380"/>

到此，Spring MVC 配置完成。

## 实现 Controller

下面我来实现一下 dispatcher servlet 转发请求。
首先把 web.xml 中的 servlet-mapping 标签中的 `*.form` 改为 `/`，表示拦截所有请求。

```
  <servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
```

在 dispatcher-servlet.xml 中添加下面配置：

```
    <context:component-scan base-package="com.hq.web.springdemo"/>

    <!--默认注解支持-->
    <mvc:annotation-driven>
    </mvc:annotation-driven>

    <!--指定视图解析器-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <!-- 视图的路径 -->
        <property name="prefix" value=""/>
        <!-- 视图名称后缀  -->
        <property name="suffix" value=".jsp"/>
    </bean>
```

如果我们想支持返回结果对象自动转为 json 的话，`mvc:annotation-driven` 必须添加。
然后贴出所有的代码：

```
@Controller
@RequestMapping("/api")
public class ApiController {

    @RequestMapping("/show")
    public String say(HttpServletRequest request){
        System.out.println("show....");
        request.setAttribute("name","James"); // 指定Model的值
        //return "/index.jsp";
        return "/demo";
    }

    @RequestMapping("/get")
    @ResponseBody
    public Data get(Info info){
        System.out.println("get...."+info.toString());

        return dealWith(info);
    }

    private Data dealWith(Info info) {
        Data data = new Data();
        String nickName = info.getNickname();
        String gender = info.getGender();
        int age = info.getAge();

        if (StringUtils.isEmpty(nickName)){
            return data.setCode(-1).setResult("nick name is null");
        } else if (StringUtils.isEmpty(gender)){
            return data.setCode(-2).setResult("gender is null");
        } else if (age == 0){
            return data.setCode(-3).setResult("age is null");
        }

        return data.setCode(200).setResult(info.toString());
    }
}
```

 - @Controller：定义一个 Controller 控制器
 - @RequestMapping：映射 Request 请求与处理器
 - @ResponseBody：该注解用于将Controller的方法返回的对象，通过适当的HttpMessageConverter转换为指定格式后，写入到Response对象的body数据区。当返回的数据不是html标签的页面，而是其他某种格式的数据时（如json、xml等）使用；

```
public class Data {
    private int code;
    private String result;

    public Data setCode(int code) {
        this.code = code;
        return this;
    }

    public int getCode() {
        return code;
    }

    public Data setResult(String result) {
        this.result = result;
        return this;
    }

    public String getResult() {
        return result;
    }
}
```

```
public class Info {
    private String nickname;
    private String gender;
    private int age;

    public void setNickname(String nickname) {
        this.nickname = nickname;
    }

    public String getNickname() {
        return nickname;
    }

    public void setGender(String gender) {
        this.gender = gender;
    }

    public String getGender() {
        return gender;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public int getAge() {
        return age;
    }

    @Override
    public String toString() {
        return "nickname = "+nickname+", gender="+gender+",age="+age;
    }
}
```

demo.jsp 代码为：

```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<body>
姓名：${name}
</body>
</html>
```

这个时候打开页面：http://localhost:8080/SpringDemo_war/api/show，发现并没有把 显示出来，那是因为由于web.xml自动生成的是servlet 2.3版本的配置，而我们项目中需要用servlet 3.0。所以需要改一下web.xml servlet版本：

```
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
        http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">
```

再次运行，查看 name 传递成功。

我们再来测试一下 get 接口，在浏览器运行下面地址：
http://localhost:8080/SpringDemo_war/api/get?nickname=James&gender=0&age=30
数据正常显示。

<!-- 
https://www.cnblogs.com/cavinchen/p/9846427.html
https://blog.csdn.net/mj_yang/article/details/80141846
https://blog.csdn.net/dainandainan1/article/details/80318780
https://www.jianshu.com/p/41cbb9647cf4
-->