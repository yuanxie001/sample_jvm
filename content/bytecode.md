# 字节码简介
java虚拟机就是构建于class字节码文件的一套规范.所以,java虚拟机规范主要定义就是class文件的格式.以及对应的文件生成和在类加载时的解析过程.

至于我们经常提到的gc主要在于在于java虚拟机的实现,比如Hotspot,JRockit等.这些虚拟机实现都需要能够解析class文件,并且符合规范的生成实例等.
因为class文件的规范十分复杂,我就简单的从开发比较关注的几个特性来讲解下.

## 字节码主要包含的结构
```c++
// 字节码整体结构
ClassFile {
    u4 magic;// 魔数,固定字符,表示字节码文件
    u2 minor_version;// 次版本号
    u2 major_version;// 主版本号
    u2 constant_pool_count;// 常量池数量
    cp_info constant_pool[constant_pool_count-1];// 常量池
    u2 access_flags; //访问标记
    u2 this_class;// 本类
    u2 super_class;// 超类
    u2 interfaces_count;// 本来实现的接口数量
    u2 interfaces[interfaces_count];// 接口表,依次列举本类实现的接口,次序和源文件一致
    u2 fields_count;// 字段数量,包括静态和非静态字段
    field_info fields[fields_count];// 字段表详情
    u2 methods_count;// 方法数量,只包含自己本类定义的或者覆写的.
    method_info methods[methods_count];// 方法表信息 
    u2 attributes_count;// 属性表数量
    attribute_info attributes[attributes_count];//属性表集合
}
// 字段表结构
field_info {
    u2 access_flags;// 访问标记
    u2 name_index;// 名称索引
    u2 descriptor_index;// 字段描述
    u2 attributes_count;// 属性表数量
    attribute_info attributes[attributes_count];// 属性表
}
// 方法表结构
method_info {
    u2 access_flags;// 访问标记
    u2 name_index;// 名称索引
    u2 descriptor_index;// 方法描述
    u2 attributes_count;//属性表数量
    attribute_info attributes[attributes_count];// 属性表
}
```
这里先列出我们主要的class文件结构. 访问标记是一个u2类型的,就是2个字节,16个字符.class文件定义在不同位置不同值的含义.我们经常用的public,protected,private,static,volatile等,都在这个位置表示的.
名称索引主要是表示这个方法或者字段的名称,u2是一个常量池的地址.访问的是一种CONSTANT_Utf8_info结构的常量,表示一种索引.另外,字段或方法描述符,指向的也是常量池中的一种CONSTANT_Utf8_info结构,用的是java虚拟机的定义的字符串表示字段或者方法的定义.

java虚拟机的字段的字节码表示的主要规则如下:
| 字段类型| 类型 | 含义 |
| --- | -- | -- |
| B | byte | 有符号单字节整形 |
| C |char | 基于多文种平面中的Unicode代码点,UTF-16编码 |
| D | double | 双精度浮点数 |
| F | float | 单精度浮点数 |
| I | int | 整形 |
| J | long | 长整形 |
|L*classname;* | reference | classname类的实例 |
|S | short | 有符号短整形 |
| Z| boolean | boolean类型 |
| [ | reference | 数组类型|

java虚拟机方法的描述是在字段的基础上加上如下规则:
MethodDescriptor:
( {ParameterDescriptor} ) ReturnDescriptor
如果入参为空,则里面的ParameterDescriptor为空,如果ReturnDescriptor为void,则用V表示.

举例来说
| 类型| javacode| java虚拟机表现形式 |
| --- | -- | -- |
| 字段 | String b | Ljava/lang/String; |
| 方法| int[] get() | ()[I |
| 方法 | void set(int i) | (I)V |
| 方法 | String get(int i);| (I)Ljava/lang/String;|



## 类信息怎么表示的
这里的类信息单值类的访问标记,类的名称,继承父类,接口,注解以及源文件的路径名称的这些元信息.
类上可以定义的访问标记如下:
| 访问标记 | javacode| java虚拟机表现形式 |
| --- | -- | -- |
| ACC_PUBLIC | 0x0001 |声明 public; 表示可以包外访问.|
|ACC_FINAL| 0x0010| 声明为 final; 禁止子类继承.|
|ACC_SUPER| 0x0020 |当遇到invokespecial指令时,需要对父类方法特殊处理|
|ACC_INTERFACE |0x0200 | 表示是一个接口 interface,而不是要给类 |
|ACC_ABSTRACT |0x0400| 声明为抽象类abstract,不能被实例化|
|ACC_SYNTHETIC |0x1000| 声明为synthetic; 表示虚拟机自己生成的,不会出现在源码中.|
|ACC_ANNOTATION| 0x2000| 声明为一个注解类型 annonation.|
|ACC_ENUM| 0x4000| 声明一个枚举类型 enum.|
|ACC_MODULE |0x8000 |一个模块 module ,而非类或者接口,jdk9新加的|

类的名称是u2类型的,指向常量池中一个CONSTANT_Class_info结构.超类和接口表都是如此,指向的都是常量池的一个个CONSTANT_Class_info结构.

类上的注解存在于方法后面的属性表中.RuntimeVisibleAnnotations和RuntimeInvisibleAnnotations这两个属性主要用来表示类上注解的.区别就是RuntimeInvisibleAnnotations运行时不可见,但RuntimeVisibleAnnotations里面的信息运行时可见.这两个属性尤其复杂,我们另开一篇博客讨论.

类的源文件所在的说明主要在SourceFile属性中,这个属性详细记录了生成这个class文件的源文件路径等信息.

## 我的字段类型和初始化信息都在哪






## 方法存在哪

## 调试的时候,字节码怎么和代码对应起来的

## 注解都存哪了

## 异常处理是怎么反应在字节码的

## 我的泛型存在哪了