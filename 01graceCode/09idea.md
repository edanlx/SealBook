# 【优雅代码】09idea调优
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [上一篇](./08commonPool.md)构建自己的连接池
* [下一篇](./10front.md)优雅和前端交互

## 1.背景
用好idea可以辅助程序员更快的开发，从效率和bug上都能取得更优秀的成绩
## 2.idea设置
1. 设置悬浮提示
Preferences | Editor | General | Code Completion->勾选Show quick documentation
2. 自动导入包
Preferences | Editor | General | Auto Import->勾选Optimize imports on the fly
3. 去掉import *
Preferences | Editor | Code Style | Java | imports->class count...和 Names count...都改大即可
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
6. properties编码
Editor | File Encodings->勾选 Transp....
## 3.idea插件
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

## 4.idea快捷键
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
## 5.idea辅助功能
1. 查看方法调用链
导航窗navigate->Call Hierarchy
2. 优化代码
右键项目->analyze
3. 查找未使用的方法
导航窗Analyze->run inspection by name->输入unused declaration
