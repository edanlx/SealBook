# 【优雅代码】09-idea断点、插件、模板合集
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [视频讲解](https://www.bilibili.com/video/BV1pY41187rm)  
* [上一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/08commonPool.md)构建自己的连接池
* [下一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/10front.md)拒绝if/else数据校验及转换

## 1.背景
用好idea可以辅助程序员更快的开发，从效率和bug上都能取得更优秀的成绩
## 2.ideaDebug
简单的或者意义不大的就不介绍了
```java
// 测试代码如下
static class TestForIdea {
    public String name;
}

public static int test1() {
    int z = 0;
    for (int i = 0; i < 10; i++) {
        log.info(String.valueOf(i));
        z = i;
    }
    List<Integer> collect = Stream.of(1, 4, 6).filter(a -> (a & 1) == 1).map(a -> a + a)
            .collect(Collectors.toList());
    TestForIdea testForIdea = new TestForIdea();
    testForIdea.name = "1";
    testForIdea.name = "2";
    return z;
}
```
### 2.1强制回退
一般调试程序的时候会有错误数据，当已知数据错误的时候为了避免入库可以使用该方法
![forceReturn](http://seal_li.gitee.io/sealbook/pic/grace_09idea_forceReturn.png)
### 2.2丢弃当前栈帧并回退
当进入某个method后过了一个断点需要重新观察再次进入该method时可以使用该方案
![dropFrame](http://seal_li.gitee.io/sealbook/pic/grace_09idea_dropFrame.png)
### 2.3debug时修改变量
当进入某个method调试时因为过程过于复杂不易于修改源头数据，则可以直接修改最终返回数据即可
![setValue](http://seal_li.gitee.io/sealbook/pic/grace_09idea_setValue.png)
### 2.4debug时输出表达式
当需要执行复杂代码操作或者查看深层变量可以使用该方法
![evaluate](http://seal_li.gitee.io/sealbook/pic/grace_09idea_evaluate.png)
### 2.5stream流断点
虽然stream流简化了很多代码，但代码调试是真的什么都看不到，用该方法可以将执行链完整显示
![TraceCurrentStreamChain](http://seal_li.gitee.io/sealbook/pic/grace_09idea_TraceCurrentStreamChain.png)
### 2.6对象属性监控
有时候代码多起来不知道某个值在什么时候变了可以用该方法监控，非常好用
![setValue](http://seal_li.gitee.io/sealbook/pic/grace_09idea_valueWatch.png)
### 2.7打印当前栈信息
有些时候某个方法被多个地方调用需要打印栈(调用链)可以使用该方式
![printStack1](http://seal_li.gitee.io/sealbook/pic/grace_09idea_printStack1.png)
![printStack2](http://seal_li.gitee.io/sealbook/pic/grace_09idea_printStack2.png)
### 2.8指定属性进入断点
在for循环中经常会值过多找不到特定值，这时候使用该方式精准断点
![specialValue](http://seal_li.gitee.io/sealbook/pic/grace_09idea_specialValue.png)
### 2.9多线程调试
有时候调试多线程需要查看每个线程的运行情况，实际因为并行断点并不能查看每个线程可能就直接过去了，可以采用如下方法
![ThreadFrame](http://seal_li.gitee.io/sealbook/pic/grace_09idea_ThreadFrame.png)
但更多的时候需要盯着一个线程看，参考上一个方法
![ThreadValue](http://seal_li.gitee.io/sealbook/pic/grace_09idea_ThreadValue.png)
## 3.idea设置
1. 设置悬浮提示
Preferences | Editor | General | Code Completion->勾选Show quick documentation
![ThreadValue](http://seal_li.gitee.io/sealbook/pic/grace_09idea_Completion.png)
2. 自动导入包
Preferences | Editor | General | Auto Import->勾选Optimize imports on the fly
![ThreadValue](http://seal_li.gitee.io/sealbook/pic/grace_09idea_Optimizeimports.png)
3. 去掉import *
Preferences | Editor | Code Style | Java | imports->class count...和 Names count...都改大即可
![import](http://seal_li.gitee.io/sealbook/pic/grace_09idea_import.png)

4. 设置自定义模板
* Preferences | Editor | Live Templates->右侧add新增组->然后再新增模板
* 方法注释
```java
/**
 * description
 * 
$params$
 * @return {@link $returns$}
 * @author seal 876651109@qq.com
 * @date $date$ $time$
 */
```
|Name|Expression|
|--|--|
|params|groovyScript("def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {result+=' * @param ' + params[i] + ((i < params.size() - 1) ? '\\r\\n' : '')}; return result", methodParameters())|
|returns|methodReturnType()|
|date|date()|
|time|time()|  

因为md格式问题在这里重新输出下params:
```groovyScript
groovyScript("def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {result+=' * @param ' + params[i] + ((i < params.size() - 1) ? '\\r\\n' : '')}; return result", methodParameters())
```
* lombok注释
```java
@Data
@EqualsAndHashCode(callSuper = true)
@AllArgsConstructor
@ToString(callSuper = true)
@NoArgsConstructor
@Builder
@JsonInclude(JsonInclude.Include.NON_EMPTY)
```
* log日志注释,打印出行号比较省心，不用给每次日志的关键字起名字
```java
log.info("$methodeName$:$line$line:{}","")
```
|Name|Expression|
|--|--|
|methodeName|methodeName()|
|line|lineNumber()|
5. 序列化提示 
Preferences | Editor | Inspections | serializable ->勾选class without 'serialVersionUID'
![serializable](http://seal_li.gitee.io/sealbook/pic/grace_09idea_serializable.png)
6. properties编码
![propertiesUTF8](http://seal_li.gitee.io/sealbook/pic/grace_09idea_propertiesUTF8.png)
Editor | File Encodings->勾选 Transp....
## 4.idea插件
这部分推荐的比较少，个人在用的不止这几个
1. lombok
主要用@slfj和javaBean相关的注解，插入后可以再用idea反编译成实体代码，非常舒适
2. free mybaties
最舒适的功能是辅助检查xml的错误，当然跳转这功能也很给力
3. alibabacode
规范代码，每次上线前可以用一下，能处理掉一些bug
4. maven helper
maven的优秀伙伴，但是越发觉得maven臃肿了，项目体量一上来jar包冲突惨不忍睹
5. jclasslib
学习字节码的好帮手
![ThreadValue](http://seal_li.gitee.io/sealbook/pic/grace_09idea_jclasslib.png)
6. SequenceDiagaram
生成时序图
7. QAfindBug
静态检测代码是否有死循环、无限递归等潜在因素
8. bito
利用chatgpt分析代码，复杂代码快速分析其作用，避免没有注释看不懂

## 5.idea快捷键
|快捷键|介绍|
|--|--|
|Ctrl + Alt + L|格式化代码，代码就是我们写的诗，至少雷军大大也是这么说的|
|Ctrl + C|CV大法好|
|Ctrl + V|CV大法好|
|Ctrl + Z|撤销|
|Ctrl + F|当前页查找|
|Ctrl +shift+ F|全局查找,还能选正则和文件格式，非常舒心，而且可以查jar包里面的东西|
|shift+shift|全局查找方法，能查jar包里面的方法，很好用|
|Ctrl + Alt + 左方向键/右方向键|前进/后退到上一个操作的地方|
|CTRL+ALT+T|方法块进行try/catch|
|CTRL+shift+T|创建单元测试|
## 6.idea辅助功能
1. 查看方法调用链
导航窗navigate->Call Hierarchy
![ThreadValue](http://seal_li.gitee.io/sealbook/pic/grace_09idea_navigatStackChain.png)
2. 优化代码
右键项目->analyze
3. 查找未使用的方法
导航窗Analyze->run inspection by name->输入unused declaration
