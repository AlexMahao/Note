
我们可以配置tomcat-user.xml，达到管理tomcat中运行的项目以及通过MAVEN将代码直接部署到tomcat中。整体流程如下


### `tomcat-user.xml`中添加用户信息

```
<role rolename="admin-gui"/>
<role rolename="admin-script"/>
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="manager-jmx"/>
<role rolename="manager-status"/>

<user username="name" password="password" roles="manager-gui,manager-script,manager-jmx,manager-status,admin-script,admin-gui"/>

```


### `maven`的`pom.xml`中添加插件

```
		<plugin>
				<!-- 配置插件 -->
				<groupId>org.apache.tomcat.maven</groupId>
				<artifactId>tomcat7-maven-plugin</artifactId>
				<configuration>
					<!-- 注意此处的url,写死，指定tomcat对应的ip -->
					<url>http://localhost:8080/manager/text</url>
					<username>name</username> <!--tomcat-user中配置的-->
					<password>password</password><!--tomcat-user中配置的-->
					<path>/spearbothy</path> <!-- 此处的名字是项目发布的工程名 -->
				</configuration>
			</plugin>

```


### 通过`maven`发布工程到tomcat

在发布工程时，必须保证tomcat是运行状态.

通过`eclipse`运行，首次发布运行`tomcat7:deplay`。

其次再次部署运行`tomcat7:redeplay`


### 远程访问

在部署的时候，如果是本地的tomcat,通过本地的ecplise都没有问题，但是对于编译到远程,tomcat会有一个ip的判断，此时我们要配置它允许远程管理。

在目录`conf/Catalina/localhost/`添加文件`manager.xml`,如下配置：

```java 
<Context privileged="true" antiResourceLocking="false"   
         docBase="${catalina.home}/webapps/manager">  
             <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="^.*$" />  
</Context>

```

`allow`表示允许的ip，可以配置`allow:127.0.0.1`,这样就只允许本机访问。

