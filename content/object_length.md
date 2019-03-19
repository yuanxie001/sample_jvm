# 一个java对象占多大内存

java中,一个对象在虚拟机中占用多大的内存,大多都没有一个直观的答案.所以就想对这个问题深入了解下.

## java对象的组成
一个java对象主要由三个部分组成的.**对象头**,**实例数据**和**对齐填充(padding)**.对齐填充主要存在的意义是因为jvm中内存的分配和扩张都是以8个字节为单位的.所以如果分配了不足8个的.多余的会用字节补齐.

## 对象头
在32位jvm中,对象头是由8个字节组成的.主要分成两部分,前半部分是运行时markword,详细信息如下:

| 锁状态 | 25bit | 4bit | 1bit 是否偏向锁| 2bit锁标记位 |
| ----- | ------ | ------ | --- | -- |
|无锁 | 对象的hashcode | gc年龄分代 | 是 | 01 |
|偏向锁 | 23bit的线程ID,2bit的Epoch | gc年龄分代 | 是 | 01|
|轻量级锁 |指向栈中记录的指针 | | | 00 |
|重量级锁 | 指向重量级锁的指针 | | | 10 |
|GC标记 | 空| | | 11 |

无锁和偏向锁状态下,锁标记位都是01.区别就是前面的是否偏向锁的标记位.

后半部分是klass,指向方法区中当前实例所属类的元数据信息.这部分也占4个字节.
在64位虚拟机中,都会增大一倍,也就是整个对象头占16字节.如果开启了-XX:+UseCompressedOops指针压缩,只会压缩klass的寻址地址.不会压缩运行时的markword.所以,开启 -XX:+UseCompressedOops的对象头大小是12bit. UseCompressedOops参数是默认开启的.
这里的对象年龄分代信息占4bit,表示最大值是15,虚拟机调优标记中**-XX:InitialTenuringThreshold=N**, **-XX:MaxTenuringThreshold=N**这两个标记所产生的晋升年龄就是和这个值比较的决定晋升的.所以这个值的设置最大为15.当然各个垃圾收集器的行为不同.

## 实例数据
java对象包含的参数主要有如下一些类型,以及他们所占的空间.
| 类型 | 大小(字节) |
| -- | -- |
| byte | 1 |
|short | 2| 
|int | 4| 
|long| 8 |
|float| 4|
|double | 8| 
| char | 2|
| boolean | 1|
| reference | 在32位jvm上以及小于32G内存的64位jvm上是4,在启用大堆的64位jvm上是8|
> 注:reference是引用数据类型,java里面的数组,类和接口.在64位虚拟机32G内存一下开启UseCompressedOops指针压缩的话,就是4.这里32G内存一下是4是因为指针压缩默认开启.

实例数据的大小涉及一个jvm的概念,就是深堆,浅堆和保留堆.
- 深堆: 一个对象以及字段所包含对象的所有数据.
- 浅堆: 一个对象仅仅包含自己的存储空间的大小.
- 保留堆: 一个对像的字段如果只属于当前字段,则属于当前对象大小.

## 计算一个对象大小.
java中有jdk.nashorn.internal.ir.debug.ObjectSizeCalculator类可以用来计算一个对象的大小.
```java
class A{
    int a;
}
class B {
    int b;
    Locale locale = Locale.CANADA;
}

class C {
    int c;
    Map map = new HashMap()
}

public class SizeCalculatorTest {
    public static void main(String[] args) {
        long asize = ObjectSizeCalculator.getObjectSize(new A());
        System.out.println(asize);
        long bsize = ObjectSizeCalculator.getObjectSize(new B());
        System.out.println(bsize);
        long csize = ObjectSizeCalculator.getObjectSize(new C());
        System.out.println(csize);
    }
}

```

| | A | B |C |
|--|--|--|--|
|浅堆|16 |24|24|
|深堆|16 |224|72|
|保留堆|16 | 24 | 72|

B和C的差异主要在于,因为如果对象里面的一个字段只有自己用到.那么就算在保留堆里面.如果这个对象还和其他对象共享.即不算保留堆.这也是保留堆和深堆的主要差别.所以,我们说一个对象的大小的时候,还得看它所指的究竟是这三个概念里面的哪一个.垃圾回收机制主要关注的是保留堆.

## UseCompressedOops的原理
在新版的64位jvm中,在内存小于32G的时候,会开启指针压缩.即在32位寻址来寻址35位的地址空间.

我觉得这个设计十分巧妙.就是,我们虚拟机是8位对其的.这样,8位中间的位置我们可以不关心,只需要关注开始就可以了.所以在存储的时候,我们可以默认尾三位是0,从而将其截取掉.这样读出来的时候,我们再在尾3位加上三个0即可.

但这样也许还会问为什么是三个,而不是4个呢.如果是4个都取零.这样我们就可以用32位寻址支持64G内存了.这里主要是一个权衡.就是如果结尾是1个byte的.正好多个1位,这样就会浪费7位,如果是16位对其,就会浪费15位.这样会大大增加空间浪费的概率.所以选择8位,这样在少浪费空间的情况下,节约内存.