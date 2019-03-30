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

## 虚拟机字节码信息描述

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

## 字段信息的字节码表示
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


## 方法的字节码表示

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

访问标记下面是方法名称和方法描述.这两个都指向常量池的CONSTANT_Utf8_info结构.描述上文已经提过了.主要是我们经常关注的方法的code,异常处理,本地声明的变量,注解等信息,这些都存在属性表里面.我们后面详细分析方法的属性表.

## 方法的内容字节码表示
方法的主要内容都存在方法的一个叫Code的属性中,这是我们这节的主要说明.为什么把它要单独拿出来说,因为这个是我们最为关注的东西都在里面.它的主要结构是这样的:
```c
Code_attribute {
    u2 attribute_name_index; // 属性表的字段说明,code属性里面是常量Code
    u4 attribute_length;// 属性表字节长度,不包括开始的6个字节
    u2 max_stack; // 最大栈深度
    u2 max_locals; // 最大本地变量表长度
    u4 code_length; // 虚拟机指令长度
    u1 code[code_length]; // 虚拟机指令数组
    u2 exception_table_length; // 异常处理器个数
    { 
        u2 start_pc; // 开始接管位置
        u2 end_pc; // 结束位置
        u2 handler_pc; // 处理器位置开始
        u2 catch_type; // 处理异常
    } exception_table[exception_table_length]; // 处理器数组
    u2 attributes_count; // 属性表个数
    attribute_info attributes[attributes_count]; // 属性表数组.
}
```
我们写的代码,转化为字节码之后,会转化成一个个虚拟机指令.这些指令存在Code属性的code数组里面.code数组是u1类型的,表示一个字节表示所有的操作内容了.所以,虚拟机字节指令最多有256个. 虚拟机只用了两百多个指令就实现了我们程序所有的功能. 压缩度还是很高的. 

这其中的max_stack和max_locals是用来存储命令执行过程中的数据的.max_stack是最大操作数占深度,程序在运行这个方法是时候,这个操作数站的深度是固定的.不会发生改变. max_locals是局部变量表数量,在方法执行过程中,也是固定的.这两个参数都是基于方法编译时计算好的. 每个操作数栈和局部变量表所占空间是4个字节. 局部变量表单位是槽, 如果是double或者long类型,则会占据两个槽. 其他类型都只占一个槽,包括reference类型的对象.(就是引用类型的对象).如果方法执行过程中使用超过了这两个数的最大值,或者执行出空占的指令.都会抛错. 感觉虚拟机是将内存分配的大小计算提前到了编译期处理的.这样方法执行的时候,就非常方便的开辟和回收空间,以节约性能.

因为虚拟机操作指令是和数据类型绑定的,所以在读取的时候,会严格限制读取顺序的. double和long类型,虚拟机是绝对禁止只读取半个字节数据的.

异常表中,主要是表示java代码中tay-catch-finally的逻辑的. catch后面的异常,会出现在catch_type中.finally的处理逻辑是将catch_type表示为any,这样异常处理器开始和结束中间的任何代码最后都能被handler_pc的逻辑处理到.异常表出现顺序和代码里面出现的一致. 虚拟机在执行的时候,会一次查找异常表的处理逻辑是否匹配.如果有catch,也有final的情况下,finally的代码会生成两个异常处理器,用来分别处理正常和异常的逻辑.

再者就是,我们经常提到的局部变量的字段名,方法执行对应的行号等,这些存储在属性表的属性中,分别是LocalVariableTable,LineNumberTable. LocalVariableTable属性表存储了变量的作用范围,存储在局部变量表哪个槽,局部变量的类型,局部变量名称等信息. LineNumberTable是将字节指令对应到源码文件的哪一行上去的,主要用于开发时debug表示执行到哪一行,以及日志中出现的错误对应源文件行号等信息的处理的.
这两种对应的结构分别是:

```C
LineNumberTable_attribute {
    u2 attribute_name_index; //属性名称,固定为LineNumberTable
    u4 attribute_length; // 属性表长度.字节数,不包括开始的6个字节
    u2 line_number_table_length; // 方法的行数
    { 
        u2 start_pc; // 字节码开始序号,对应code属性中code数组的下标
        u2 line_number; // 对应源文件的行号
    } 
    line_number_table[line_number_table_length]; // 方法字段对应的数组
}

LocalVariableTable_attribute {
    u2 attribute_name_index; // 属性名称,固定为LocalVariableTable
    u4 attribute_length; // 属性表长度,字节数.不包括开始的6个字节
    u2 local_variable_table_length; // 局部变量表个数
    { 
        u2 start_pc; // 局部变量作用范围开始
        u2 length; // 局部变量作用范围结束.相当于在start_pc+length的位置结束
        u2 name_index; // 局部变量名称,指向常量池的一个常量
        u2 descriptor_index; // 局部变量描述符.用最开始的虚拟机形式表示字段类型
        u2 index; // 存储在局部变量表的位置索引.
    } 
    local_variable_table[local_variable_table_length];
}
```

## 异常处理是怎么反应在字节码的
上节简单的说明了在方法内try-catch异常的处理逻辑.但我们还会存在方法上声明我这个方法将会抛错那些异常这种场景,这个场景下,字节码又是怎么表示的呢.在方法的属性中,有一个异常表Exceptions_attribute.其结构如下:
```C
Exceptions_attribute {
    u2 attribute_name_index;// 属性表名称,固定位Exceptions
    u4 attribute_length; // 属性表长度,不包括最开始的6个字节
    u2 number_of_exceptions; // 异常数量
    u2 exception_index_table[number_of_exceptions]; // u2类型的异常排序.指向常量池的CONSTANT_Class_info结构.
}

```
方法可以抛出过个异常,每个异常以列表的形式存储在exception_index_table中,顺序和源码出现的顺序一致.

## 我的泛型存在哪了

因为泛型可能存在类上,字段或者方法上.那么,泛型应该属于class_info,field_info或者method_info上的属性.

泛型的属性名为Signature,用于保存泛型信息的.其主要结构是这样的.
```C
Signature_attribute {
    u2 attribute_name_index;//  属性名称,固定为Signature
    u4 attribute_length; // 泛型的这个字段是定值2
    u2 signature_index; // 泛型签名,指向常量池中一个CONSTANT_Utf8_info结构
}
```

以上就是泛型的结构.这里主要详细说下泛型的签名.因为在类上,方法上和字段上,所以.在这三个位置的属性表中,泛型的签名都有一套自己的规则.

先说类上的泛型签名:
类上签名的的格式为
> ClassSignature:
>     [TypeParameter]SuperclassSignature{SuperinterfaceSignature}


> TypeParameter:表示本类的泛型签名
> SuperclassSignature表示父类的签名,可能也含有泛型
> SuperinterfaceSignature表示父接口的签名,可能有多个.也可能含有签名.
> {} 表示可能不存在,也可能有多个.

如
```java
abstract class MyMap implements Map<String,Object>

//其中signature_index签名为:Ljava/lang/Object;Ljava/util/Map<Ljava/lang/String;Ljava/lang/Object;>;
//Ljava/lang/Object; 为父类签名
//Ljava/util/Map<Ljava/lang/String;Ljava/lang/Object;>; 为父接口签名.
//<>表示泛型信息
```

再说方法签名:
> MethodSignature:
[TypeParameters] ( {JavaTypeSignature} ) Result {ThrowsSignature}
TypeParameters:泛型签名
JavaTypeSignature 方法入参签名
Result 结果类型签名
ThrowsSignature 异常类型签名

如:
```C++
hashMap里面的put方法的签名
public V put(K k,V v)

签名是: (TK;TV;)TV;

T表示的是参数化类型,就是我们说的泛型.
K,V是类上定义好的泛型类型.

```
字段签名相对比较简单,就不说了.这里的签名在java里面还有更多详细的描述.比如泛型的上界下界的问题,都会在泛型里面体现的.这里不展开了.

另外,我们方法执行的时候,也会创建泛型.这个时候,泛型存在不是这里了.而是在Code属性的一个叫LocalVariableTypeTable的属性中.它的结构是这样的.
```C++
LocalVariableTypeTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 local_variable_type_table_length;
    { 
        u2 start_pc;
        u2 length;
        u2 name_index;
        u2 signature_index; // 这里就是保存局部变量的泛型签名的数据.形式和上面的泛型签名一致.
        u2 index;
    } local_variable_type_table[local_variable_type_table_length];
}
``