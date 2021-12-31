# 【优雅代码】02-自动化工具合集介绍
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/junit) 
* [视频讲解](https://www.bilibili.com/video/BV1Bq4y1174Q/)
* [上一篇](./01lombok.md)lombok精选注解及原理
* [下一篇](./03optional.md)optional杜绝空指针异常

## 1.背景介绍
在日常工作中总会需要重复的工作，而作为一个现代人，应该学会使用工具避免重复的工作。java能做很多事情不止是web方向，而如果不限于java能做的事情就更多了。 
## 2.黑盒自动化
以下介绍的软件基本以python为主，当然有些也可以用java编写
|平台|软件|
|--|--|
|web|Selenuim|
|Android|appium|
|ios|xcode|
|windows|pyautogui/AutoIt|
|mac|pyautogui/AppleScript|  

这里以pyautogui作为介绍，因为其泛用性最广，手机端也可以用模拟器配合这个东西跑，但是web和移动端并不如专门的那么好用就是。如下程序演示了使用谷歌浏览器的必应查询Hello world后截取左上角微软图标文字同时进行输出。  
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import subprocess
import time
import pyautogui
from PIL import Image
import pytesseract

subprocess.Popen('C:\Program Files (x86)\Google\Chrome\Application\chrome.exe')
time.sleep(1)
# pip install pyautogui
pyautogui.click(773, 326)
pyautogui.typewrite('Hello world!\n')
# pip install pillow
time.sleep(1)
# 截图 x,y,宽,高
path = 'F:\屏幕截图.png'
im = pyautogui.screenshot(region=(42, 140, 100, 50))
im.save(path)
# pip install pytesseract
pytesseract.pytesseract.tesseract_cmd = r'D:\Tesseract\tesseract.exe'
# https://digi.bib.uni-mannheim.de/tesseract/tesseract-ocr-w64-setup-v5.0.0-alpha.20200328.exe
text = pytesseract.image_to_string(Image.open(path))  # 调用识别引擎识别
# 输出 Microsoft Bing
print(text.replace("\n", "").replace("\f", ""))
print("finish")

```
整理一下逻辑就是对于定位一般性可以采用pyautogui的图片定位(如果不行采用坐标定位)，如果需要截取文字，能够复制的就直接双击鼠标选择当行然后ctrl+c复制下来，再通过剪切板拿到数据，不能复制的就通过坐标截取图片再用OCR获取内容。以此为基础基本可以满足所有个人自动化需求了，写个普通脚本都不在话下。不吹不黑，靠这些东西笔者是真的写过游戏脚本和交易脚本。

## 4.黑盒抓包

抓包不只是在pc端最关键是它可以辅助在app上抓包，抓到未加密连接破解后配合黑盒自动化效果更佳。这里仅介绍两个代表，windows使用fiddler，mac使用charless

### 4.1fiddler

![fiddler](https://seal_li.gitee.io/sealbook/pic/grace_junit_fidller.jpg)
抓包测试链接
https://movie.douban.com/j/search_subjects?type=movie&tag=%E7%83%AD%E9%97%A8&sort=recommend&page_limit=20&page_start=20

### 4.2charless

![2charless](http://seal_li.gitee.io/sealbook/pic/grace_junit_charless.png)

## 4.白盒测试工具

这里仅介绍postman和jmeter，使用方式上jmeter要更加全面，支持更多的协议。postman作为http测试一直非常优秀，而jmeter在性能测试方面则非常优异，甚至是数据库。
### 4.1postman
![postman](http://seal_li.gitee.io/sealbook/pic/grace_junit_postman.png)
### 4.2jmeter
![jmeter](http://seal_li.gitee.io/sealbook/pic/grace_junit_jmeter1.png)
![jmeter](http://seal_li.gitee.io/sealbook/pic/grace_junit_jmeter2.jpg)

## 5.软件打包转为可执行程序
将自动化程序打包成软件自动运行也是非常重要的一步，当你的自动化软件日趋成熟就会有分享，甚至本身就是为了解决某个MM的问题，总不能给别人是bat文件吧。桌面端推荐vue+electron进行打包核心逻辑可调用java。推荐这个主要是vscode、typora都是以electron进行研发的，并且中文文档很详细。
* [官网快速入门地址](https://www.electronjs.org/zh/docs/latest/tutorial/quick-start)
![文档截图](http://seal_li.gitee.io/sealbook/pic/grace_junit_electron.png)

***最重要的一点***：不要把语言学死了，主力开发肯定是以熟悉的语言为主，最需要掌握的是语言之间相互调用就可以完成很多看似很难的工作。

## 6.junit单元测试
### 6.1准备工作
注意scope是test，所以要在test目录下才能生效
```java
 <!-- spring-test-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <exclusions>
        <exclusion>
            <groupId>org.junit.vintage</groupId>
            <artifactId>junit-vintage-engine</artifactId>
        </exclusion>
    </exclusions>
</dependency>	
```
### 6.2注解运行流程
![junit](http://seal_li.gitee.io/sealbook/pic/grace_junit_junit1.png)
### 6.3建立单元测试
建立两个一样的类
```java
public class MyJunit {
    public void method1() {

    }

    public void method2() {

    }

    public void method3() {

    }

    public static void main(String[] args) {

    }
}
public class MyJunit2 {
    public void method1() {

    }

    public void method2() {

    }

    public void method3() {

    }

    public static void main(String[] args) {

    }
}
```
ctrl+shift+t建立相应单元测试，并填写assert方法
```java
// 如果有需要运行springBoot容器填写该注解
// @SpringBootTest
class MyJunitTest {

    @Test
    void method1() {
        assertEquals(1, 1);
    }

    // 注意此处不相等会报错
    @Test
    void method2() {
        assertEquals(1, 2);
    }

    @Test
    void method3() {
        assertEquals(1, 1);
    }
}
```
运行结果如下，每次都可以批量执行单元测试避免因为小改动改出某个bug
![junit](http://seal_li.gitee.io/sealbook/pic/grace_junit_junit2.png)

