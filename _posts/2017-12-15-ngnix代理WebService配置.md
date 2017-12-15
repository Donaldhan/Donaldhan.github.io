---
layout: page
title: nginx代理WebService配置
subtitle: nginx代理WebService配置
date: 2017-12-14 21:00:00
author: donaldhan
catalog: true
category: nginx
categories:
    -  nginx
tags:
    - WebService
---
在使用ngnix做WebService服务器的负载均衡器的时候出现出现如下错误：
```java
Exception in thread "main" org.apache.cxf.service.factory.ServiceConstructionException: Failed to create service.
	at org.apache.cxf.wsdl11.WSDLServiceFactory.<init>(WSDLServiceFactory.java:76)
	at org.apache.cxf.endpoint.dynamic.DynamicClientFactory.createClient(DynamicClientFactory.java:296)
	at org.apache.cxf.endpoint.dynamic.DynamicClientFactory.createClient(DynamicClientFactory.java:241)
	at org.apache.cxf.endpoint.dynamic.DynamicClientFactory.createClient(DynamicClientFactory.java:234)
	at org.apache.cxf.endpoint.dynamic.DynamicClientFactory.createClient(DynamicClientFactory.java:189)
	at com.eips.ws.client.WsClient.invoke(WsClient.java:40)
	at com.eips.ws.test.TestMiddleOrgnization.main(TestMiddleOrgnization.java:24)
Caused by: javax.wsdl.WSDLException: WSDLException (at /definitions/types/xsd:schema): faultCode=PARSER_ERROR: Problem parsing 'http://www.donald.com:80/ws/service?xsd=1'.: java.io.FileNotFoundException: http://www.donald.com:80/ws/service?xsd=1
	at com.ibm.wsdl.xml.WSDLReaderImpl.getDocument(Unknown Source)
	at com.ibm.wsdl.xml.WSDLReaderImpl.parseSchema(Unknown Source)
	at com.ibm.wsdl.xml.WSDLReaderImpl.parseSchema(Unknown Source)
	at com.ibm.wsdl.xml.WSDLReaderImpl.parseTypes(Unknown Source)
	at com.ibm.wsdl.xml.WSDLReaderImpl.parseDefinitions(Unknown Source)
	at com.ibm.wsdl.xml.WSDLReaderImpl.readWSDL(Unknown Source)
	at com.ibm.wsdl.xml.WSDLReaderImpl.readWSDL(Unknown Source)
	at org.apache.cxf.wsdl11.WSDLManagerImpl.loadDefinition(WSDLManagerImpl.java:236)
	at org.apache.cxf.wsdl11.WSDLManagerImpl.getDefinition(WSDLManagerImpl.java:163)
	at org.apache.cxf.wsdl11.WSDLServiceFactory.<init>(WSDLServiceFactory.java:74)
	... 6 more
Caused by: java.io.FileNotFoundException: http://www.donald.com:80/ws/service?xsd=1
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1457)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.setupCurrentEntity(XMLEntityManager.java:675)
	at com.sun.org.apache.xerces.internal.impl.XMLVersionDetector.determineDocVersion(XMLVersionDetector.java:186)
	at com.sun.org.apache.xerces.internal.parsers.XML11Configuration.parse(XML11Configuration.java:772)
	at com.sun.org.apache.xerces.internal.parsers.XML11Configuration.parse(XML11Configuration.java:737)
	at com.sun.org.apache.xerces.internal.parsers.XMLParser.parse(XMLParser.java:119)
	at com.sun.org.apache.xerces.internal.parsers.DOMParser.parse(DOMParser.java:232)
	at com.sun.org.apache.xerces.internal.jaxp.DocumentBuilderImpl.parse(DocumentBuilderImpl.java:284)
	... 16 more

```

我很是疑问，命名ngnix监听端口是8080，为什么，调用WebService的时候，会跑到80端口，我们ngnix代理声明如下：

```
upstream www.donald.com {
       least_conn;
   server       www.donald.com:8380;
   server       www.donald.com:8180;
   server       www.donald.com:8280;
   }
   server {
       listen       8080;
       server_name  www.donald.com;

       charset UTF-8;

       access_log  logs/host.access.log;
    location / {
             root    /ws;
             index index.jsp index.php index.html index.htm

             proxy_connect_timeout   10;
             proxy_send_timeout      30;
             proxy_read_timeout      30;
             proxy_pass http://www.donald.com;  #upstream www.donald.com 代理
    }
```

访问WebService服务的wsdl，发现 *soap:address location* 地址确实80，而不是8080，很是疑问：
```xml
<xsd:schema><xsd:import namespace="http://www.cxf.service/service/" schemaLocation="http://www.donald.com:80/ws/service?xsd=1"/>
</xsd:schema>
...
<service name="serviceService"><port name="servicePort" binding="tns:servicePortBinding"><soap:address location="http://www.donald.com:80/ws/service"/></port></service>
```

 搜索相关资料发现，Nginx默认反向后的端口为80，因此存在被代理后的端口为80的问题，这就导致访问出错。主要原因在Nginx的配置文件的host配置时没有设置响应的端口。Host配置只有host，没有对应的port,这就导致在被代理的地方取得错误的端口。所以修改配置如下：

```
upstream www.donald.com {
       least_conn;
   server       www.donald.com:8380;
   server       www.donald.com:8180;
   server       www.donald.com:8280;
   }
   server {
       listen       8080;
       server_name  www.donald.com;

       charset UTF-8;

       access_log  logs/host.access.log;
        location / {
                 root    /ws;
                 index index.jsp index.php index.html index.htm

                 proxy_connect_timeout   10;
                 proxy_send_timeout      30;
                 proxy_read_timeout      30;
                 proxy_pass http://www.donald.com;  #upstream www.donald.com 代理
                 proxy_set_header   Host $host:$server_port;
                 proxy_set_header   X-Real-IP        $remote_addr;
                 proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
```

主要是代理头部配置proxy_set_header：
```
proxy_set_header   Host $host:$server_port;
proxy_set_header   X-Real-IP        $remote_addr;
proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
```

再次访问WebService的服务wsdl，这是 *soap:address location* 端口为8080，可以正常访问。

```xml
<xsd:schema><xsd:import namespace="http://www.cxf.service/orginfo/" schemaLocation="http://www.donald.com:8080/bankWs/orgInfo?xsd=1"/></xsd:schema>

service name="orginfoService"><port name="orginfoPort" binding="tns:orginfoPortBinding"><soap:address location="http://www.donald.com:8080/bankWs/orgInfo"/></port></service>
```
