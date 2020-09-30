---
layout: page
title: Selenium(ChromeDriver)的一次趟坑
subtitle: Selenium(ChromeDriver)的一次趟坑
date: 2020-01-31 22:51:01
author: donaldhan
catalog: true
category: util
categories:
    - util
tags:
    - selenium
---

# 引言

最近的项目有很多的画图的需求，前端资源不足，同时画图时间长，存在拿不到资源的情况，这个重任就落到了后端的身上，一开始使用JAVA G2D，画图，时间上可能会很久，又很繁琐，必须要一点一点的堆。在痛苦中挣扎着，想到能不能使用其他的方式方案替换，试过HTML2Image，验证失败；有位同事，脑洞大开，使用无头浏览器进行截图，这边文章记录一下在使用无头浏览器截图过程中所遇到的坑及教训（selenium-chrome-driver，一般测试使用，我们用来画图，作死的节奏）。


# 目录
* [截图实操](#截图实操)
    * [本地测试验证](#本地测试验证)
    * [画图方案](#画图方案)
* [经验教训](#经验教训)
* [附](#附)


# 截图实操

初步想法，使用浏览器直接打开html内容，针对可变内容，打算使用${title}占位符形式替换，打开html先将占位符替换后，进行渲染截图。

## 本地测试验证
首先下载[Chrome浏览器][]，同时下载[Chrome驱动][]，

[Chrome浏览器]:https://www.techspot.com/downloads/4718-google-chrome-for-windows.html "Chrome浏览器"

[Chrome驱动]:https://sites.google.com/a/chromium.org/chromedriver/downloads "Chrome驱动"


引入jar包

```
 <dependency>
            <groupId>org.xhtmlrenderer</groupId>
            <artifactId>flying-saucer-core</artifactId>
            <version>9.0.6</version>
        </dependency>
        <dependency>
            <groupId>net.sourceforge.nekohtml</groupId>
            <artifactId>nekohtml</artifactId>
            <version>1.9.14</version>
        </dependency>
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>3.4.0</version>
            <exclusions>
                <exclusion>
                    <!-- declare the exclusion here -->
                    <groupId>com.google.guava</groupId>
                    <artifactId>guava</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-chrome-driver</artifactId>
            <version>3.4.0</version>
        </dependency>
        <dependency>
            <groupId>ru.yandex.qatools.ashot</groupId>
            <artifactId>ashot</artifactId>
            <version>1.5.4</version>
</dependency>
```
注意如果guava包冲突，需要剔除guava包。

编写测试类
``` java
package cn.home.image;

import lombok.extern.slf4j.Slf4j;
import org.openqa.selenium.Dimension;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

import java.io.FileOutputStream;
import java.io.IOException;
import java.util.Set;
import java.util.concurrent.TimeUnit;

/**
 * @ClassName: ChromeDriverTest
 * @Description:
 * chrome禁止本地浏览时加载本地其他文件，可以采用添加启动参数的方式来支持
 * 添加参数为 --allow-file-access-from-files  或者　--disable-web-security
 * –user-data-dir=”[PATH]“  自定义用户数据目录
 * –start-maximized                启动就最大化
 * –no-sandbox                         取消沙盒模式
 * –single-process                    单进程运行
 * –process-per-tab                 每个标签使用单独进程
 * –process-per-site                每个站点使用单独进程
 * –in-process-plugins            插件不启用单独进程
 * –disable-popup-blocking 禁用弹出拦截
 * –disable-javascript             禁用JavaScript
 * –disable-java                         禁用Java
 * –disable-plugins                   禁用插件
 * –disable-images                   禁用图像
 * -incognito                               启动进入隐身模式
 * –enable-udd-profiles        启用账户切换菜单
 * –proxy-pac-url                   使用pac代理 [via 1/2]
 * –lang=zh-CN                        设置语言为简体中文
 * –disk-cache-dir=”[PATH]“ 自定义缓存目录
 * –disk-cache-size=              自定义缓存最大值（单位byte）
 * –media-cache-size=         自定义多媒体缓存最大值（单位byte）
 * –bookmark-menu              在工具栏增加一个书签按钮
 * –enable-sync                       启用书签同步
 * @Author: Donaldhan
 * @Date: 2019-11-08 17:21
 */
@Slf4j
public class ChromeDriverTest {
    public static void main(String[] args) throws IOException {
        // cherome driver
        System.setProperty("webdriver.chrome.driver",
                "E:\\chromedriver\\chromedriver.exe");
        ChromeOptions chromeOptions = new ChromeOptions();
        // 设置为 headless 模式 （无头浏览器）
        chromeOptions.addArguments("--headless");
//        --disable-web-security --allow-file-access-from-files --allow-file-access
        chromeOptions.addArguments("--disable-web-security");
        chromeOptions.addArguments("--allow-file-access-from-files");
        chromeOptions.addArguments("--allow-file-access");
        chromeOptions.addArguments("--no-startup-window");
        chromeOptions.addArguments("--disable-features=VizDisplayCompositor");
        chromeOptions.addArguments("--disable-gpu");
        chromeOptions.addArguments("disable-infobars");
        chromeOptions.addArguments("--disable-extensions");
//            chromeOptions.addArguments("window-size=1200x600");
        chromeOptions.addArguments("--no-sandbox");
//            chromeOptions.addArguments("start-maximized");
//            chromeOptions.addArguments("disable-notifications");
        chromeOptions.addArguments("allow-running-insecure-content");
        //--no-startup-window
        ChromeDriver driver = new ChromeDriver(chromeOptions);
        driver.manage().window().setSize(new Dimension(800, 600));
        // 设置隐性等待时间
        driver.manage().timeouts().implicitlyWait(8, TimeUnit.SECONDS);
        String imgPath= "file://F:/bg.jpg";
//        String imgPath= "/bg.jpg";
//        String imgPath=  "https://cdn.buttercms.com/bHS9cJFqTkaG6H6E1Sxj";
        String title = "网页截图";
        String count = "5";
        String html_content1 = "<!DOCTYPE html>\n" +
                "<html>\n" +
                "  <head>\n" +
                "    <title>${title}</title>\n" +
                "    <meta charset=\"utf-8\" />\n" +
                "    <meta name=\"viewport\" content=\"width=device-width,initial-scale=1.0\" />\n" +
                "    <meta name=\"referrer\" content=\"no-referrer\" />\n" +
                "    <title>DEMO</title>\n" +
                "  </head>\n" +
                "  <body>\n" +
                "    <p>数量:${count}天</p>\n" +
                "  <img src=\""+imgPath+"\" />\n" +
                "  </body>\n" +
                "</html>";

        html_content1 = html_content1.replace("${title}",title);
        html_content1 = html_content1.replace("${dakaCount}",dakaCount);
        log.info("html_content1:{}",html_content1);
        long begin = System.currentTimeMillis();
        for(int i= 0 ;i< 100;i++){
            // before opening the new window
            String parentWindow = driver.getWindowHandle();
            // after the new window was closed
            driver.switchTo().window(parentWindow);
            if(i%2==0){
                driver.get("data:text/html;charset=utf-8," + html_content1);
            }
            else {
                driver.get("data:text/html;charset=utf-8," + html_content);
            }
            byte[] screenshotAs = driver.getScreenshotAs(OutputType.BYTES);
            FileOutputStream fw = new FileOutputStream("F:\\image\\"+i+"hless.png");
            fw.write(screenshotAs);
            fw.close();
        }
        long end = System.currentTimeMillis();
        log.info("time:{}",(end - begin)/1000);
//        Thread.sleep(3000);
        Set<String> windowHandles = driver.getWindowHandles();
        for (String string : windowHandles) {
            System.out.println("find windows:"+string);
            driver.switchTo().window(string).close();
        }
        driver.close();
    }
}

```
本地测试没有问题，另外针对cdn的图片，html中需要添加如下内容
```
     "    <meta name=\"viewport\" content=\"width=device-width,initial-scale=1.0\" />\n" +
                "    <meta name=\"referrer\" content=\"no-referrer\" />\n" +
```
否则无法渲染。 这个只是测试，要能在生产环境中可用，还不行，因为使用selenium-chrome-driver进行截图，具体过程为，使用无头浏览器打开一个tab，然后截图，这样会存在一个并发的问题里，想到了一个方案。

## 画图方案

 *1.所有任务全部丢到任务队列中（Redis 列表(List)，ConcurrentLinkedQueue，如果需要，则存储的数据库中）；  
 2.开多个线程去做队列中拿任务；  
 3.当任务处理完，通知到调用方；  
 4.针对实时请求的，轮询结果；  
 5.针对redis，list， 如果并发leftPOP，rightPush，存在并发leftPop, rightPush可能失败，可以使用阻塞模式[BLOP][]， 即leftPOP(timeout, 0为无线)，  
 
 这里之所以使用redis队列一个主要的原因，放到内存队列中，重启时，任务会丢失。

[BLOP]:http://redis.io/commands/blpop "BLOP"

这个方案幸亏没有上线通过，不然上线了，还有坑，线上Redis集群不支持BLOP， 这个可能是BLOP，阻塞会占用资源的原因。[PIKA][] 同样不支持这个操作。


[PIKA]:ttps://github.com/Qihoo360/pika "PIKA"


按照上述方案，发现截图为空，本地可以，开发环境（linux）无法截图，肯能是直接打开*data:text/html* 的原因，只能换另外一种方案，

画图的时候，用浏览器打开web容器的一个网页，这回功能是实现了，但测试服务器因为画图，CPU使用过高，同事由于姿势不多，内存被暂满，直接导致测试服务器down机； 最终排查的原因为，在服务器重启时，使用的虚拟机挂钩，去关闭浏览器，在这个过程中，chrome没有正确的关闭，最重要的原因是挂钩这部分没有验证，这也是一个教训，应该将这些关闭操作与Spring的bean的生命周期结合起来。

最后方案没有上线，MD，周末加班，使用G2D重新实现功能。



# 经验教训

这次浏览器截图给了深刻的教训，在项目时间不充足的情况下，老老实实的使用成熟的方案，针对新方案，需要调研，实验后才能采用（这个过程可能还有坑）。

还有画图这个事情，前端甩给后端，后端想让前端去做，互相搞得不愉快，这一块是一个锅；其实最应该推动的是产品需求的简化。

# 附

## 截图问题
在截图的过程中，针对有滑动条的截图，将会把滑动条截出来，修改配置项无果，最后使用了Ashot进行全屏解决，最后对图片进行的裁剪（获取Image的长宽，并裁剪），奥西把奶。

## deban中安装chrome的

```
sudo -iu root
dpkg -i ./google-chrome-stable_current_amd64.deb
```
如果存在依赖问题，使用

```
apt-get -f install
```
[解决依赖][]

[解决依赖]: https://www.jianshu.com/p/6d7323bfaa0c "解决依赖"




安装成功后，检查版本，修改修改权限

```
google-chrome --version
chmod 744 chromedriver
chown donald_draper:users chromedriver
```


## 在linux环境中,截图时，如果遇到如下问题：
The driver executable does not exist
The error message "The driver executable does not exist" occurs most commonly in one of the scenarios:
1. The path to the driver executable file set in your System.setProperty() method is incorrect.
Solution: Check whether the file actually exists at the specified path by searching the path in your File Explorer(Windows Explorer(Windows)/ Find files(Mac)/ Nautilus(Ubuntu)).
2. Another reason that is usually seen in Linux/Mac is lack of executable permissions for the specified driver file.
Solution: Check the permissions for the file in your Terminal and use chmod command to add the ‘x’ permissi...(more)


首先查看路径下的chromdriver存不存在，然后确认文件可不可以执行，不能添加执行权限

```
chmod 755 chromedriver
```

