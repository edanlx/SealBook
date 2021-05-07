# 我偷偷改了你编译后的class文件
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md)  
[可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/jvm/clazz)  
[视频讲解](https://www.bilibili.com/video/BV1454y1r7mf/)   
[文字版](https://github.com/edanlx/SealBook/blob/master/jvm/clazz.md)

如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.准备工作
准备一份代码
```java
public class TestEntity {
    private String name;
}
```

编译所得
```text
cafe babe 0000 0034 0012 0a00 0300 0f07
0010 0700 1101 0004 6e61 6d65 0100 124c
6a61 7661 2f6c 616e 672f 5374 7269 6e67
3b01 0006 3c69 6e69 743e 0100 0328 2956
0100 0443 6f64 6501 000f 4c69 6e65 4e75
6d62 6572 5461 626c 6501 0012 4c6f 6361
6c56 6172 6961 626c 6554 6162 6c65 0100
0474 6869 7301 002e 4c63 6f6d 2f65 7861
6d70 6c65 2f64 656d 6f2f 6c65 7373 6f6e
2f6a 766d 2f63 6c61 7a7a 2f54 6573 7445
6e74 6974 793b 0100 0a53 6f75 7263 6546
696c 6501 000f 5465 7374 456e 7469 7479
2e6a 6176 610c 0006 0007 0100 2c63 6f6d
2f65 7861 6d70 6c65 2f64 656d 6f2f 6c65
7373 6f6e 2f6a 766d 2f63 6c61 7a7a 2f54
6573 7445 6e74 6974 7901 0010 6a61 7661
2f6c 616e 672f 4f62 6a65 6374 0021 0002
0003 0000 0001 0002 0004 0005 0000 0001
0001 0006 0007 0001 0008 0000 002f 0001
0001 0000 0005 2ab7 0001 b100 0000 0200
0900 0000 0600 0100 0000 0300 0a00 0000
0c00 0100 0000 0500 0b00 0c00 0000 0100
0d00 0000 0200 0e
```
可使用javap -verbose TestEntity.class辅助查看

安装jclasslib
## 2.class文件结构
|类型|名称|数量|
|---|----|----|
|u4|magic|1|
|u2|minor_version|1|
|u2|major_version|1|
|u2|constant_pool_count|1|
|cp_info|constant_pool|constant_pool_count - 1|
|u2|access_flags|1|
|u2|this_class|1|
|u2|super_class|1|
|u2|interfaces_count|1|
|u2|interfaces|interfaces_count|
|u2|fields_count|1|
|field_info|fields|fields_count|
|u2|methods_count|1|
|method_info|methods|methods_count|
|u2|attributes_count|1|
|attribute_info|attributes|attributes_count|
具体的
## 3.实例解析
1)cafe babe  
这个就是开头魔数占4个字节  
2)0000  
代表次版本号占2个字节，16进制转10进制为0，表示此版本号为0  
3)0034  
代表主版本号占2个字节，16进制转10进制为52，表示此版本号为1.8。51代表的是1.7以此类推，可以在idea打开class文件有提示。  

javap的部分编译片段
```text
Constant pool:
   #1 = Methodref          #3.#15         // java/lang/Object."<init>":()V
   #2 = Class              #16            // com/example/demo/lesson/jvm/clazz/TestEntity
   #3 = Class              #17            // java/lang/Object
   #4 = Utf8               name
   #5 = Utf8               Ljava/lang/String;
   #6 = Utf8               <init>
   #7 = Utf8               ()V
   #8 = Utf8               Code
   #9 = Utf8               LineNumberTable
  #10 = Utf8               LocalVariableTable
  #11 = Utf8               this
  #12 = Utf8               Lcom/example/demo/lesson/jvm/clazz/TestEntity;
  #13 = Utf8               SourceFile
  #14 = Utf8               TestEntity.java
  #15 = NameAndType        #6:#7          // "<init>":()V
  #16 = Utf8               com/example/demo/lesson/jvm/clazz/TestEntity
  #17 = Utf8               java/lang/Object
```
4)0012  
16进制转10进制为18，代表常量池内数量为17个。至于哪17个javap命令写的还是很清楚的。但实际上是18个，因为默认有个null占了0号位。  
5)0a  
16进制转10进制为10,在常量池中代表方法引用，可查看附录
5)00 03  
16进制转10进制为3,在常量池中表示索引，即第3个常量，查看可知指向了Object
6)00 0f
16进制转10进制为15,在常量池中表示索引，即第15个常量，连起来的意思就是常量池里面有个object的初始化方法
7)07
16进制转10进制为70,在常量池中代表class，可查看附录

字节码文件以此类推

## 4.字节码命令
```text
0 aload_0
1 invokespecial #1 <java/lang/Object.<init>>
4 return
```
如上命令分别表示1.将this入栈，2.执行object的init方法3.出栈

## 5.知识补充

1)java agent可以在main函数之前执行，可以用于传统aop，也是依赖字节码

2)静态代理是直接修改字节码完成的

3)编译检查也是可以通过class文件完成的

## 6.附录

### 6.1访问权限查询手册
|flag_name|value|desc|
|---|----|----|
|ACC_PUBLIC|0x0001|public修饰符号|
|ACC_PRIVATE|0x0002|表示私有的|
|ACC_PRIVATE|0x0004|protected|
|ACC_PRIVATE|0x0008|static|
|ACC_FINAL|Ox0010|final|
|ACC_SUPER|Ox0020|通过invokeSpecial指令可以调用父类的方法|
|ACC_INTERFACE|0x0200|标识是一个接口|
|ACC_ABSTRACT|0x0400|表示是一个抽象类|
|ACC_SYNTHETIC|0x1000|该class是动态生成的没有源文件|
|ACC_ANNOTATION|0x2000|是一个注解类型类的访问权限查询手册|
|ACC_ENUM|0x4000|表示是一个枚举类型|
|ACC_PRIVATE|0x8000|表示这是一个模块|


### 6.2Field_info 字段表结构
|类型|名称|数量|
|---|----|----|
|u2|access_flag(权限修饰符)|1|
|u2|name_index(字段名称索引)|1|
|u2|descciptor_index(字段描述索引)|1|
|u2|attribute_count(属性表个数)|1|
|attribute_info|attributes|attribute_count|

### 6.3Method_info 字段表结构
|类型|名称|数量|
|---|----|----|
|u2|access_flag(权限修饰符)|1|
|u2|name_index(方法名称索引)|1|
|u2|descciptor_index(方法描述索引)|1|
|u2|attribute_count(属性表个数)|1|
|attribute_info|attributes|attribute_count|

### 6.4attribute_info 结构
|类型|名称|数量|
|---|----|----|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u1|info[attribute_length]|1|

### 6.5常量池flag标注
|常量|项目|类型|描述|
|---|----|----|---|
|CONSTANT_Utf8_info|tag|u1|值为1|
|-|length|u2|UTF-8编码的字符串占用了的字节数|
|-|bytes|u1|长度为length的UTF-8编码的字符串|
|CONSTANT_Integer_info|tag|u1|值为3|
|-|bytes|u4|按照高位在前存储的int值|
|CONSTANT_Float_info|tag|u1|值为4|
|-|bytes|u4|按照高位在前存储的float值|
|CONSTANT_Long_info|tag|u1|值为5|
|-|bytes|u8|按照高位在前存储的long值|
|CONSTANT_Double_info|tag|u1|值为6|
|-|bytes|u8|按照高位在前存储的double值|
|CONSTANT_Class_info|tag|u1|值为7|
|-|index|u2|指向全限定名常量项的索引|
|CONSTANT_String_info|tag|u1|值为8|
|-|inedx|u2|指向字符串字面量的索引|
|CONSTANT_Fieldref_info|tag|u1|值为9|
|-|index|u2|指向声明字段的类或者接口描述符CONSTANT_Class_info的索引项|
|-|index|u2|指向字段描述符CONSTANT_NameAndType_info的索引项|
|CONSTANT_Methodref_info|tag|u1|值为10|
|-|index|u2|指向声明字段的类或者接口描述符CONSTANT_Class_info的索引项|
|-|index|u2|指向字段描述符CONSTANT_NameAndType_info的索引项|
|CONSTANT_InterfaceMethodref_info|tag|u1|值为11|
|-|index|u2|指向声明字段的类或者接口描述符CONSTANT_Class_info的索引项|
|-|index|u2|指向字段描述符CONSTANT_NameAndType_info的索引项|
|CONSTANT_NameAndType_info|tag|u1|值为12|
|-|index|u2|指向该字段或方法名称常量项的索引|
|-|index|u2|指向该字段或描述名称常量项的索引|
|CONSTANT_MethodHandle_info|tag|u1|值为15|
|-|reference_kind|u1|值必须在1~9之间它决定了方法句柄的类型。方法类型的句柄类型的值表示方法句柄的字节码行为|
|-|reference_index|u2|值必须是对常量池的有效索引|
|CONSTANT_MethodHandle_info|tag|u1|值为16|
|-|descriptor_index|u2|值必须是对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示方法的描述符|
|CONSTANT_MethodHandle_info|tag|u1|值为17|
|-|bootstrap_method_attr_index|u2|值必须是对当前class文件中引导方法表的bootstrap_methods[]数组的有效索引|
|-|name_and_type_index|u2|值必须是对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示方法的描述符||
|CONSTANT_InvokeDynamic_info|tag|u1|值为18|
|-|bootstrap_method_attr_index|u2|值必须是对当前class文件中引导方法表的bootstrap_methods[]数组的有效索引|
|-|name_and_type_index|u2|值必须是对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示方法的描述符||
|CONSTANT_Module_info|tag|u1|值为19|
|-|name_index|u2|值必须是对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示模块名字||
|CONSTANT_Module_info|tag|u1|值为20|
|-|name_index|u2|值必须是对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示模包名称||

## 参考资料
《深入理解Java虚拟机》-周志明