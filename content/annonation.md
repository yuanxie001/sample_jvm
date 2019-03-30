# 字节码中的注解说明

注解是java中说明性的信息,程序运行时可以根据注解的信息来改变自己的行为.注解可以标注在类,字段,方法,构造器以及方法入参上.并且注解又分为运行时可见和运行时不可见的情况.所以,在class文件中,有4个属性用来单独保存注解的.分别是

| 类型| 说明 | 作用点 |
| -- | -- | -- |
| RuntimeVisibleAnnotations | 运行时可见注解 | class_info,field_info和method_info上 |
|RuntimeInvisibleAnnotations | 运行时不可见注解 |  class_info,field_info和method_info上 |
| RuntimeInvisibleParameterAnnotations | 运行时不可见 | method_info |
| RuntimeVisibleParameterAnnotations |运行时可见 | method_info |

RuntimeVisibleAnnotations和RuntimeInvisibleAnnotations结构是一样的,RuntimeInvisibleParameterAnnotations和RuntimeVisibleParameterAnnotations结构也是一样的.我们分成两组来说.

## RuntimeVisibleAnnotations说明
RuntimeVisibleAnnotations属性的结构如下所示:

```C++
RuntimeVisibleAnnotations_attribute {
    u2 attribute_name_index; // 属性表名称,这里固定是RuntimeVisibleAnnotations
    u4 attribute_length; // 属性表长度(字节数),不包括开始的6个字节.
    u2 num_annotations; // 
    annotation annotations[num_annotations]; // 注解内容.
}

annotation {
    u2 type_index; // 表示注解类型,指向常量池中一个CONSTANT_Utf8_info结构
    u2 num_element_value_pairs; // 当前注解的属性个数
    { 
        u2 element_name_index; //表示注解中属性名称.指向常量池中一个CONSTANT_Utf8_info结构
        element_value value; // 元素的值.
    } element_value_pairs[num_element_value_pairs];
}

element_value {
    u1 tag; // 标注类型
    union { // 表示值
        u2 const_value_index; // 如果是基本数据类型或者是String的时候,这个必须有值.
        { 
            u2 type_name_index; // 表示枚举的二进制描述.必须是CONSTANT_Utf8_info结构
            u2 const_name_index;// 表示枚举项的简单名称,也是一个ONSTANT_Utf8_info结构
        } enum_const_value; // 如果是枚举类型的,才会有这个值.
        u2 class_info_index; // 如果是class类型,则这个必须有值.
        annotation annotation_value; // 注解里面还可以继续放注解.这个是注解值
        { 
            u2 num_values; // 数组元素个数.
            element_value values[num_values];//每个数组元素的值.是一个自嵌套结构.
        } array_value; // 有数组的时候才会出现这个值.
    } value;
}

```

这里面是用element_value的tag表示注解属性的类型的.对照表如下:

| tag Item|  Type| value Item |Constant Type|
| -- | -- |  -- |--|
|B  |byte |const_value_index| CONSTANT_Integer |
|C | char |const_value_index|CONSTANT_Integer|
|D|double|const_value_index|CONSTANT_Double|
|F|float|const_value_index|CONSTANT_Float|
|I|int|const_value_index|CONSTANT_Integer|
|J|long|const_value_index|CONSTANT_Long|
|S|short|const_value_index|CONSTANT_Integer|
|Z|boolean|const_value_index|CONSTANT_Integer|
|s|String|const_value_index|CONSTANT_Utf8|
|e|Enum|type|enum_const_value|Not applicable|
|c|Class|class_info_index|Not applicable|
|@|Annotation type|annotation_value|Not applicable|
|[| Array type| array_value| Not applicable|

这里说明,注解必须要一个在编译期就能确定的数据.比如基本数据类型或者String类型的字面量,或者class类型,注解类型或者枚举,数组也是上面类型的集合. 所有不确定性的东西都不能放在注解中来表示.

RuntimeInvisibleAnnotations和RuntimeVisibleAnnotations注解一样,唯一区别在于运行时期可不可见.我觉得jvm之所以这么设计,就是在类加载的时候,很容易区分这部分需不需要在class元数据中保留.直接通过属性的名就可以判断了.十分快捷.

同时RuntimeInvisibleAnnotations和RuntimeVisibleAnnotations属性在对应的class_info,field_info或method_info中最多只有一个.多个注解用放在数组表示即可.

## RuntimeVisibleParameterAnnotations注解
RuntimeVisibleParameterAnnotations是一种方法上的属性表.主要用于存储方法入参的注解信息的.其结构如下

```C++
RuntimeVisibleParameterAnnotations_attribute {
    u2 attribute_name_index; // 属性名称,固定值为RuntimeVisibleParameterAnnotations
    u4 attribute_length; // 属性长度,不包括开始的6个字节
    u1 num_parameters; // 方法入参的个数
    { 
        u2 num_annotations; //表示这个参数上有多少个注解
        annotation annotations[num_annotations];// 表示参数上注解的内容
    } parameter_annotations[num_parameters]; // parameter_annotations第i项就表示i个入参上的注解信息.
}

```

annonation的信息和上面一致,就不在细说了.这里主要为方法入参用的.每个method结构最多也只有1个RuntimeVisibleParameterAnnotations结构.

这就是注解是如何保存在字节码文件的主要内容.当然这里关注的更多的是使用注解的场景,而不是定义注解的场景
