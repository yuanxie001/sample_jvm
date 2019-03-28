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
```
java字节码整体结构如下. 这里主要说明下前面几个字段.u4的magic,魔数,表示这个文件是class文件,这是一个定值0xCAFEBABE.java虚拟机在读取字节码校验的时候.会先校验魔数是不是这个值,如果不是,虚拟机会拒绝解析,直接抛错.

其次,minor_version是次版本号,major_version是主版本号.java虚拟机向前兼容,所以主版本号越大的虚拟机能解析前面所有版本的class文件.但反过来不行,老版本的虚拟机不支持新版本的字节码文件.这个也是在虚拟机类加载的时候做的校验.

## 虚拟机字节码信息表示

虚拟机字段或者方法在java虚拟机有一套独特的表述形式的.其主要规则如下.

| 字段类型| 类型 | 含义 |
| --- | -- | -- |
| B | byte | 有符号单字节整形 |
| C |char | 基于多文种平面中的Unicode代码点,UTF-16编码 |
| D | double | 双精度浮点数 |
| F | float | 单精度浮点数 |
| I | int | 整形 |
| J | long | 长整形 |
| L*classname;* | reference | classname类的实例 |
| S | short | 有符号短整形 |
| Z| boolean | boolean类型 |
| [ | reference | 数组类型 |

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

| 访问标记 | 字节码表述 | 含义 |
| --- | -- | -- |
| ACC_PUBLIC | 0x0001 | 声明 public; 表示可以包外访问.|
| ACC_FINAL| 0x0010| 声明为 final; 禁止子类继承.|
| ACC_SUPER| 0x0020 |当遇到invokespecial指令时,需要对父类方法特殊处理|
| ACC_INTERFACE |0x0200 | 表示是一个接口 interface,而不是要给类 |
| ACC_ABSTRACT |0x0400| 声明为抽象类abstract,不能被实例化|
| ACC_SYNTHETIC |0x1000| 声明为synthetic; 表示虚拟机自己生成的,不会出现在源码中.|
| ACC_ANNOTATION| 0x2000| 声明为一个注解类型 annonation.|
| ACC_ENUM| 0x4000| 声明一个枚举类型 enum.|
| ACC_MODULE |0x8000 |一个模块 module ,而非类或者接口,jdk9新加的|

类的名称是u2类型的,指向常量池中一个CONSTANT_Class_info结构.超类和接口表都是如此,指向的都是常量池的一个个CONSTANT_Class_info结构.

类上的注解存在于方法后面的属性表中.RuntimeVisibleAnnotations和RuntimeInvisibleAnnotations这两个属性主要用来表示类上注解的.区别就是RuntimeInvisibleAnnotations运行时不可见,但RuntimeVisibleAnnotations里面的信息运行时可见.这两个属性尤其复杂,我们另开一篇博客讨论.

类的源文件所在的说明主要在SourceFile属性中,这个属性详细记录了生成这个class文件的源文件路径等信息.

## 我的字段类型和初始化信息都在哪
字段在字节码的表现形式如上面所示的.
```c++
// 字段表结构
field_info {
    u2 access_flags;// 访问标记
    u2 name_index;// 名称索引
    u2 descriptor_index;// 字段描述
    u2 attributes_count;// 属性表数量
    attribute_info attributes[attributes_count];// 属性表
}

```

| 访问标记 | 字节码表述 | 含义 |
| --- | -- | -- |
| ACC_PUBLIC | 0x0001 |  声明 public; 表示可以包外访问.|
| ACC_PRIVATE | 0x0002 | 声明为私有private,只能通过本类中的访问 |
| ACC_PROTECTED | 0x0004 | 声明为protected; 只能通过本类,同包或者子类访问. |
| ACC_STATIC | 0x0008 | 声明为静态的 static.|
| ACC_FINAL | 0x0010 | 声明为 final; 直接分配在实例化之后 |
| ACC_VOLATILE | 0x0040 | 声明为volatile; 不能被cpu缓存 |
| ACC_TRANSIENT | 0x0080 | 声明为 transient; 不会被持久化对象管理写入或者读取|
| ACC_SYNTHETIC | 0x1000| 声明为synthetic; 编译期生存,不会出现在源码中 |
| ACC_ENUM| 0x4000| 声明为一个枚举的枚举项|

名称索引是指向常量池一个CONSTANT_Utf8_info结构.表示字段的名称.而字段描述也是指向一个常量池中的一个CONSTANT_Utf8_info的结构.表述方式由上面所示的.
属性表里面主要存的有很多种说明信息,比如泛型,注解,初始化值等.这些都是在属性表中存在的.所以属性表会有多个.


## 方法存在哪

```c++
// 方法表结构
method_info {
    u2 access_flags;// 访问标记
    u2 name_index;// 名称索引
    u2 descriptor_index;// 方法描述
    u2 attributes_count;//属性表数量
    attribute_info attributes[attributes_count];// 属性表
}
```
访问标记的主要说明如下:

| 访问标记 | 字节码表述 | 含义 |
| --- | -- | -- |
| ACC_PUBLIC | 0x0001 | 声明为 public;  表示可以包外访问.|
| ACC_PRIVATE | 0x0002 | 声明为私有private,只能通过本类中的访问 |
| ACC_PROTECTED | 0x0004 | 声明为protected; 只能通过本类,同包或者子类访问. |
| ACC_STATIC | 0x0008 | 声明为静态的 static.|
| ACC_FINAL | 0x0010 | 声明为final; 禁止被子类覆写 |
| ACC_SYNCHRONIZED | 0x0020 |声明为synchronized; 调用会使用一个monitor监视 |
| ACC_BRIDGE | 0x0040 | 一个bridge 方法,表示编译器生成的.|
| ACC_VARARGS | 0x0080 | 声明可变参数. |
| ACC_NATIVE | 0x0100 |声明native; 被其他语言实现,而不是java |
| ACC_ABSTRACT | 0x0400 | 声明为abstract; 没有提供方法实现.|
| ACC_STRICT | 0x0800 | 声明为strictfp; 浮点计算模式为FPstrict.|
| ACC_SYNTHETIC | 0x1000 |声明为synthetic;不会出现在源码中.|

访问标记下面是字节码


## 调试的时候,字节码怎么和代码对应起来的

## 注解都存哪了

## 异常处理是怎么反应在字节码的

## 我的泛型存在哪了