# 深入理解java内存屏障(volatile实现原理)
### 文章目录

*   *   [一、前言](#_1)
    *   [二、CPU的内存一致性模型](#CPU_26)
    *   [三、java规范下的内存屏障](#java_68)
    *   [四、从字节码层面看volatile](#volatile_198)
    *   [五、从JDK源码层面看volatile](#JDKvolatile_413)
    *   [六、从x86架构下看内存屏障](#x86_630)
    *   [七、实际汇编下的内存屏障](#_699)
    *   [八、总结](#_715)

一、前言
----

阅读本文需要先了解以下：

> *   对java内存模型有一定的了解。 [浅谈java内存模型](https://mp.weixin.qq.com/s/xa4mmP-ImJYnALREWQ-0Lg)
> *   对CPU的cache一致性有一定了解。 [CPU的cache一致性](https://mp.weixin.qq.com/s/Kvv32tjHazVveUgQMubkRQ)

在上篇文章中，我们知道了[内存](https://so.csdn.net/so/search?q=%E5%86%85%E5%AD%98&spm=1001.2101.3001.7020)屏障用来解决多核CPU内缓存数据不一致的问题。

cpu为了提高性能，对内存一致性进行破坏：

> *   cpu的有序性破坏：实际上是乱序执行的，只保证单cpu在逻辑上的有序性。
> *   cpu的可见性破坏：并非严格数据一致，有可见性问题。

内存屏障就是一种指令， 告诉CPU不要重排序，且此处代码前后需要严格的数据一致性。

像是一道屏障，把屏障指令之前的代码和屏障指令之后的代码隔离开，防止CPU让前后两段指令的逻辑发生错乱。

通过使用各种CPU提供的不同内存屏障指令，维护[java内存模型](https://so.csdn.net/so/search?q=java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B&spm=1001.2101.3001.7020)的统一。

二、CPU的内存一致性模型
-------------

什么是CPU的内存一致性？内存一致性更关注于多个CPU看到多个内存地址的读写的顺序。

对读写操作进行全排序，就有以下四种读写的顺序问题：

> *   读-读（LoadLoad）:先进行load1操作再进行一个load2操作。当发生乱序时，看起来像是先进行load2操作再进行一个load1操作。
> *   写-写（StoreStore）:先进行store1操作再进行store2操作。当发生乱序时，看起来像是先进行store2操作再进行一个store1操作。
> *   读-写（LoadStore）:先进行load1操作再进行store2操作。当发生乱序时，看起来像是先进行store2操作再进行一个load1操作。
> *   写-读（StoreLoad）:先进行store1操作再进行load2操作。当发生乱序时，看起来像是先进行load2操作再进行一个store1操作。

理想状态下，多个CPU看到多个内存地址的读写的顺序是不会发生乱序的。

但是大多数CPU为了提高性能，会放松内存一致性的要求，导致乱序的发生。

不同的cpu可能采用不同的内存一致性模型，进行不同程度地放松内存一致性要求。

常见的内存一致性模型如下：

> *   TSO模型(total store ordering)：放松了写-读的内存一致性要求，即写-读，可能乱序成读-写。参照上一篇文章中的描述，CPU内部引入一个FIFO的store buffer，没有Invalidate Queue这样的。
> *   PSO模型(partial store order)：放松了写-读的基础上，又放松了写-写，即两个写可能会乱序。和tso模型对比的话，大致区别是store buffer不是FIFO的。
> *   RMO模型(relaxed memory order)：读-读，读-写，写-写，写-读，四种操作都允许乱序。

![image-20220401203050841](/Users/mipengcheng3/Library/Application Support/typora-user-images/image-20220401203050841.png)



为什么会发生乱序呢？

我理解的发生乱序的原因：

> *   写-读（StoreLoad）: 引入store buffer这样的结构，写操作不会马上生效，如果写在读之后才生效，即发生了乱序，看起来像是先读后写。
> *   写-写（StoreStore）：由于store buffer不是FIFO，则很明显两个store操作则可能发生乱序。
> *   读-读（LoadLoad）: 如果CPU内部有Invalidate Queue这样的数据结构，第二个读之前没有清除Invalidate Queue，则可能读到旧值，看起来像是发生了乱序，第二个读看起来像是更早执行的。
> *   读-写（LoadStore）: 这种乱序比较难以理解是怎么发生的，由于各种CPU内部实现不同硬件的内部我也很难去深究，这里不去较真理解了。（M：其实这个简单，主要针对的场景是在引入store buffer结构后，当store buffer中的数据还没有刷到cache时，CPU直接读了cache中的值）

三、java规范下的内存屏障
--------------

java为了屏蔽了不同处理器一致性模型的差异，同时能尽可能地抽象各种处理器的一致性模型，java按照最放松的一致性模型为基础，抽象了以下四种内存屏障：

1.  LoadLoad Barriers

作用在两个读（Load）操作之间内存屏障。

    Load1;
    LoadLoad;
    Load2;


该屏障可以确保在该屏障之后的第一个读操作（load2）之前，一定能先加载load1对应的数据。在上篇文章中的smp\_rmb就属于LoadLoad Barriers。

2.  StoreStore Barriers  
    作用在两个Store 操作之间的内存屏障。

    Store1;
    StoreStore;
    Store2;
    

该屏障可以确保在该屏障之后的第一个写操作（store2）之前，store1操作对其他处理器可见（刷新到内存）。在上篇文章中的smp\_wmb就属于StoreStore Barriers。

3.  LoadStore Barriers

作用在 Load 操作和Store 操作之间的内存屏障。

    Load1;
    StoreStore;
    Store2;


该屏障可以确保 Store2 写出的数据对其他处理器可见之前，Load1 读取的数据一定先读入缓存。

4.  StoreLoad Barriers

作用在 Store 操作和 Load 操作之间的内存屏障。

    Store1; 
    StoreLoad; 
    Load2;


该屏障可以确保store1操作对其他处理器可见（刷新到内存）之后才能读取 Load2 的数据到缓存。

可以看到，四种内存屏障对应读写操作的四种排序。

在以上的四种内存屏障中，StoreLoad屏障是性能开销最大的屏障，且几乎所有的多核处理器都支持该屏障。事实证明，使用StoreLoad内存屏障也可以获得和LoadLoad，StoreStore，LoadStore这三种内存屏障一样的效果。

java通过插入以上四种内存屏障在指令序列中，来达到正确的代码执行效果。

那具体会在什么情况下会插入以上四种内存屏障呢？

> 定义四种内存屏障是为了维护JMM内存模型，主要是以下三个准则：
>
> （1） 所有[volatile](https://so.csdn.net/so/search?q=volatile&spm=1001.2101.3001.7020)读写之间相互序列化。volatile属性进行写操作后，其他CPU能马上读到最新值。
>
> （2） volatile读取操作之后发生的非volatile读写不能乱序到其之前。非volatile读写发生在volatile读之前，可以乱序到其之后。
>
> （3） volatile写操作之前发生的非volatile读写不能乱序到其之后。非volatile读写发生在volatile写之后，可以乱序到其之前。

以上要求摘自内存屏障的jdk[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)的注释中。

怎么看待以上三句话呢？

> 第一点很好理解，就是volatile的可见性要求。
>
> 第二点是为了维护happens-before准则。给个具体场景进行理解：比如我要在进行volatile读后，根据读到的值进行一些代码逻辑操作，如果这些逻辑重排到了volatile读之前，则可以理解这些逻辑代码都是基于一个旧 volatile 值做的，即逻辑上不满足volatile是最新的值。
>
> 第三点和第二点类似，也是为了维护happens-before准则。比如我先进行一段代码逻辑，再进行volatile写，如果这些逻辑重排到volatile写之后，当其他cpu看到volatile写操作时，就无法确实volatile写操作之前的操作是否已经确实地发生了。

下表显示了怎么样的两个操作步骤之间需要插入内存屏障（JSR133规范）：

![image-20220401203125565](/Users/mipengcheng3/Library/Application Support/typora-user-images/image-20220401203125565.png)

从上面这个表格我们可以看出：

> 1.  两个普通读或写之间是不需要内存屏障的；
> 2.  先进行了volatile读之后再进行读写操作都是需要内存屏障的，这是为了维护第二条准则；
> 3.  进行读写操作再进行volatile写中间都都是需要内存屏障的，这是为了维护第三条准则；
> 4.  volatile 读和同步块入口（monitor enter）等价，volatile 写和同步块出口（monitor exit）等价；
> 5.  使用synchronized同步块也会获得volatile的可见性效果（但是最好不要用synchronized来获得可见性，synchronized性能消耗大且不严谨，比如synchronized可能被编译器进行锁消除处理）。

所以，最后总结，java会在进行读写操作时，会在以下两种场景下生成内存屏障：

在volatile读的后面都会加上LoadLoad和LoadStore两个屏障:

    int a = b; // b是volatile变量
    LoadLoad
    LoadStore
    ...其他代码


在volatile写的前面都会加上LoadStore和StoreStore两个屏障，在后面加上StoreLoad屏障：

    ...其他代码
    LoadStore
    StoreStore
    a = 0; // a是volatile变量
    StoreLoad
    ...其他代码


理论上在进行volatile写之后，只有后续进行volatile读才需要插入storeLoad屏障。

但是，编译器又不能感知到多线程环境下在volatile写时，后续各个cpu是否有volatile读操作。

所以，java插入内存屏障时采用了保守策略，进行volatile写后一定会插入storeLoad屏障来保证可见性。

四、从字节码层面看volatile
-----------------

上面讲了这么多关于java规范中关于内存屏障的生成规范，我们好奇内存屏障到底长什么样。是不是在java代码编译成字节码的时候，在volatile属性的读写操作前后加上这四种屏障指令呢。

为了查看volatile修饰对应的字节码内容，我们来看下以下java代码编译成字节码：

    public class Test{
        int a,b;
        volatile int v,u;
    
        void f() {
            int i, j;
            i = a;// load a
            j = b;// load b
            i = v;// load v
            // LoadLoad
           j = u;// load u
            // LoadStore
            a = i;// store a
            b = j;// store b
            // StoreStore
           v = i;// store v
            // StoreStore
            u = j;// store u
            // StoreLoad
            i = u;// load u
            // LoadLoad
            // LoadStore
            j = b;// load b
            a = i;// store a
        }
    public static void main(String[] args) {
           new Test().f(); 
        }  
    }


通过javap -verbose Test，字节码如下：

    Classfile /D:/soft_space/jitwatch/sandbox/classes/Test.class
      Last modified 2021-1-27; size 664 bytes
      MD5 checksum 83984d12c97a1fce7c5cd2b5802bbc0f
      Compiled from "Test.java"
    public class Test
      minor version: 0
      major version: 52
      flags: ACC_PUBLIC, ACC_SUPER
    Constant pool:
       #1 = Methodref          #9.#31         // java/lang/Object."<init>":()V
       #2 = Fieldref           #6.#32         // Test.a:I
       #3 = Fieldref           #6.#33         // Test.b:I
       #4 = Fieldref           #6.#34         // Test.v:I
       #5 = Fieldref           #6.#35         // Test.u:I
       #6 = Class              #36            // Test
       #7 = Methodref          #6.#31         // Test."<init>":()V
       #8 = Methodref          #6.#37         // Test.f:()V
       #9 = Class              #38            // java/lang/Object
      #10 = Utf8               a
      #11 = Utf8               I
      #12 = Utf8               b
      #13 = Utf8               v
      #14 = Utf8               u
      #15 = Utf8               <init>
      #16 = Utf8               ()V
      #17 = Utf8               Code
      #18 = Utf8               LineNumberTable
      #19 = Utf8               LocalVariableTable
      #20 = Utf8               this
      #21 = Utf8               LTest;
      #22 = Utf8               f
      #23 = Utf8               i
      #24 = Utf8               j
      #25 = Utf8               main
      #26 = Utf8               ([Ljava/lang/String;)V
      #27 = Utf8               args
      #28 = Utf8               [Ljava/lang/String;
      #29 = Utf8               SourceFile
      #30 = Utf8               Test.java
      #31 = NameAndType        #15:#16        // "<init>":()V
      #32 = NameAndType        #10:#11        // a:I
      #33 = NameAndType        #12:#11        // b:I
      #34 = NameAndType        #13:#11        // v:I
      #35 = NameAndType        #14:#11        // u:I
      #36 = Utf8               Test
      #37 = NameAndType        #22:#16        // f:()V
      #38 = Utf8               java/lang/Object
    {
      int a;
        descriptor: I
        flags:
    
      int b;
        descriptor: I
        flags:
    
      volatile int v;
        descriptor: I
        flags: ACC_VOLATILE
    
      volatile int u;
        descriptor: I
        flags: ACC_VOLATILE
    
      public Test();
        descriptor: ()V
        flags: ACC_PUBLIC
        Code:
          stack=1, locals=1, args_size=1
             0: aload_0
             1: invokespecial #1                  // Method java/lang/Object."<init>":()V
             4: return
          LineNumberTable:
            line 1: 0
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0       5     0  this   LTest;
    
      void f();
        descriptor: ()V
        flags:
        Code:
          stack=2, locals=3, args_size=1
          //从这里开始阅读是代码f方法对应的字节码
             0: aload_0  //加载本地变量表下标0位置，即Test对象（this）
             1: getfield //获得Test的属性a     #2  // Field a:I
             4: istore_1 //赋值本地变量表下标1对应的变量，即将a赋值给i
             5: aload_0
             6: getfield      #3                  // Field b:I
             9: istore_2
            10: aload_0
            11: getfield      #4                  // Field v:I
            //对volatile属性v进行了读，这里应该要有内存屏障才对
            14: istore_1
            15: aload_0
            16: getfield      #5                  // Field u:I
             //同理对volatile属性u进行了读，这里应该要有内存屏障才对
            19: istore_2
            20: aload_0
            21: iload_1
            22: putfield      #2                  // Field a:I
            25: aload_0
            26: iload_2
            27: putfield      #3                  // Field b:I
            30: aload_0
            31: iload_1
            //对volatile属性v进行了写，这里应该要有内存屏障才对
            32: putfield      #4                  // Field v:I
            //对volatile属性v进行了写，这里应该要有内存屏障才对
            35: aload_0
            36: iload_2
            //对volatile属性u进行了写，这里应该要有内存屏障才对
            37: putfield      #5                  // Field u:I
            //对volatile属性u进行了写，这里应该要有内存屏障才对
            40: aload_0
            41: getfield      #5                  // Field u:I
            //对volatile属性u进行了读，这里应该要有内存屏障才对
            44: istore_1
            45: aload_0
            46: getfield      #3                  // Field b:I
            49: istore_2
            50: aload_0
            51: iload_1
            52: putfield      #2                  // Field a:I
            55: return
          LineNumberTable:
            line 7: 0
            line 8: 5
            line 9: 10
            line 11: 15
            line 13: 20
            line 14: 25
            line 16: 30
            line 18: 35
            line 20: 40
            line 23: 45
            line 24: 50
            line 25: 55
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0      56     0  this   LTest;
                5      51     1     i   I
               10      46     2     j   I
    
      public static void main(java.lang.String[]);
        descriptor: ([Ljava/lang/String;)V
        flags: ACC_PUBLIC, ACC_STATIC
        Code:
          stack=2, locals=1, args_size=1
             0: new           #6                  // class Test
             3: dup
             4: invokespecial #7                  // Method "<init>":()V
             7: invokevirtual #8                  // Method f:()V
            10: return
          LineNumberTable:
            line 27: 0
            line 28: 10
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0      11     0  args   [Ljava/lang/String;
    }
    SourceFile: "Test.java"


查看字节码我们发现，在volatile读写的前后并没有内存屏障信息的生成。即一个属性有没有加volatile进行修饰，对java代码编译成字节码指令没有影响，生成的字节码指令都一样的。

虽然生成的字节码指令是一样的。但是我们还是能发现属性描述的不同。

当属性被修饰为volatile时，在生成的字节码的class内属性对应access\_flags是不一样的（比如上文字节码的代码行数59和63的地方）。添加了volatile的属性，对应的字节码属性描述中，access\_flag会多了一个ACC\_VOLATILE的标记。

那volatile修饰符是怎么生效的呢？带着好奇心我们看一下在jdk源码中volatile是如何处理的。

五、从JDK源码层面看volatile
-------------------

在openjdk的/src/hotspot/share/interpreter/bytecodeInterpreter.cpp中，可以看到jdk源码中对volatile的处理逻辑。

bytecodeInterpreter.cpp是字节码解释器的实现类，我们看下bytecodeInterpreter.cpp中的以下关键代码：

    //属性的读指令，getfield对应对象的属性，getstatic对应静态变量，即类的属性
    CASE(_getfield):
    CASE(_getstatic):
    {
          //省略若干代码
          ......
          //是否有volatile修饰
          if (cache->is_volatile()) {
                //省略若干代码
                ......
                //根据属性的类型不同，进行不同的读操作（对象类型或各种基础类型）
                if (tos_type == atos) {
                  VERIFY_OOP(obj->obj_field_acquire(field_offset));
                  SET_STACK_OBJECT(obj->obj_field_acquire(field_offset), -1);
                } else if (tos_type == itos) {
                  SET_STACK_INT(obj->int_field_acquire(field_offset), -1);
                } else if (tos_type == ltos) {
                  SET_STACK_LONG(obj->long_field_acquire(field_offset), 0);
                  MORE_STACK(1);
                } else if (tos_type == btos || tos_type == ztos) {
                  SET_STACK_INT(obj->byte_field_acquire(field_offset), -1);
                } else if (tos_type == ctos) {
                  SET_STACK_INT(obj->char_field_acquire(field_offset), -1);
                } else if (tos_type == stos) {
                  SET_STACK_INT(obj->short_field_acquire(field_offset), -1);
                } else if (tos_type == ftos) {
                  SET_STACK_FLOAT(obj->float_field_acquire(field_offset), -1);
                } else {
                  SET_STACK_DOUBLE(obj->double_field_acquire(field_offset), 0);
                  MORE_STACK(1);
                }
          } else {
                //根据属性的类型不同，进行不同的读操作
                if (tos_type == atos) {
                  VERIFY_OOP(obj->obj_field(field_offset));
                  SET_STACK_OBJECT(obj->obj_field(field_offset), -1);
                } else if (tos_type == itos) {
                  SET_STACK_INT(obj->int_field(field_offset), -1);
                } else if (tos_type == ltos) {
                  SET_STACK_LONG(obj->long_field(field_offset), 0);
                  MORE_STACK(1);
                } else if (tos_type == btos || tos_type == ztos) {
                  SET_STACK_INT(obj->byte_field(field_offset), -1);
                } else if (tos_type == ctos) {
                  SET_STACK_INT(obj->char_field(field_offset), -1);
                } else if (tos_type == stos) {
                  SET_STACK_INT(obj->short_field(field_offset), -1);
                } else if (tos_type == ftos) {
                  SET_STACK_FLOAT(obj->float_field(field_offset), -1);
                } else {
                  SET_STACK_DOUBLE(obj->double_field(field_offset), 0);
                  MORE_STACK(1);
                }
          }
          UPDATE_PC_AND_CONTINUE(3);
    }
    //属性的写指令，getfield对应对象的属性，getstatic对应静态变量，即类的属性
    CASE(_putfield):
    CASE(_putstatic):
        {
          //省略若干代码
          ......
        //是否有volatile修饰
        if (cache->is_volatile()) {
            //根据属性的类型不同，进行不同的写操作
            if (tos_type == itos) {
              obj->release_int_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == atos) {
              VERIFY_OOP(STACK_OBJECT(-1));
              obj->release_obj_field_put(field_offset, STACK_OBJECT(-1));
            } else if (tos_type == btos) {
              obj->release_byte_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == ztos) {
              int bool_field = STACK_INT(-1);  // only store LSB
              obj->release_byte_field_put(field_offset, (bool_field & 1));
            } else if (tos_type == ltos) {
              obj->release_long_field_put(field_offset, STACK_LONG(-1));
            } else if (tos_type == ctos) {
              obj->release_char_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == stos) {
              obj->release_short_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == ftos) {
              obj->release_float_field_put(field_offset, STACK_FLOAT(-1));
            } else {
              obj->release_double_field_put(field_offset, STACK_DOUBLE(-1));
            }
            //StoreLoad屏障
            OrderAccess::storeload();
        } else {
            //根据属性的类型不同，进行不同的写操作
            if (tos_type == itos) {
              obj->int_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == atos) {
              VERIFY_OOP(STACK_OBJECT(-1));
              obj->obj_field_put(field_offset, STACK_OBJECT(-1));
            } else if (tos_type == btos) {
              obj->byte_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == ztos) {
              int bool_field = STACK_INT(-1);  // only store LSB
              obj->byte_field_put(field_offset, (bool_field & 1));
            } else if (tos_type == ltos) {
              obj->long_field_put(field_offset, STACK_LONG(-1));
            } else if (tos_type == ctos) {
              obj->char_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == stos) {
              obj->short_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == ftos) {
              obj->float_field_put(field_offset, STACK_FLOAT(-1));
            } else {
              obj->double_field_put(field_offset, STACK_DOUBLE(-1));
            }
        }
        UPDATE_PC_AND_TOS_AND_CONTINUE(3, count);
    }


上述代码是字节码解释器在处理getfield,getstatic,putfield,putstatic指令的具体逻辑。具体逻辑类似以下伪代码：

    if(指令==getfield || 指令==getstatic){
         ......
         //就是判断access_flag中是否含有ACC_VOLATILE
         if(常量池拿到的属性描述 is volatile){
             //volatile读，xxx这里是笔者简化表示对属性类型进行各种判断的逻辑（引用类型和几种基础类型对应的指令不同）
             //xxx_field_acquire最终调用OrderAccess::load_acquire
             //OrderAccess::load_acquire
             xxx_field_acquire 
         }else{
             //正常读         
             xxx_field
         }
         ......
    //如果是putfield指令或者putstatic指令
    //putfield对应实例对象的属性，putstatic对应静态变量，即类对象的属性    
    }else if(指令==putfield || 指令==putstatic){
        ......
        //就是判断access_flag中是否含有ACC_VOLATILE
        if(常量池拿到的属性描述 is volatile){
            //volatile写，xxx这里是笔者简化表示对属性类型进行各种判断的逻辑（引用类型和几种基础类型对应的指令不同）
            //release_xxx_field_put最终调用OrderAccess::release_store
            //OrderAccess::release_store
            release_xxx_field_put
            //写入StoreLoad
            OrderAccess::storeload();  
        }else{
            //正常写入
            xxx_field_put 
        }
        ......    
    }


通过上面得代码，我们大概可以了解到，在程序的实际运行时，JVM对字节码进行解释时，碰到getfield和putfield这些对对象的属性进行读写操作的指令时，判断指令对应的属性的access\_flag为ACC\_VOLATILE时，就会在汇编指令中添加上内存屏障指令。

进一步梳理整个逻辑，如下所示：


    if(属性读指令操作){
         if(属性被volatile修饰){
             OrderAccess::load_acquire
         }else{
             正常读
         }
         ......
    }else if(属性写指令操作){
        if(属性被volatile修饰){
            OrderAccess::release_store
            OrderAccess::storeload
        }else{
            正常写
        }
    }


如上，关键涉及OrderAccess::load\_acquire,OrderAccess::release\_store，OrderAccess::storeload这三个方法。

很明显，OrderAccess::storeload 就对应java虚拟机抽象出来的StoreLoad屏障指令。

而OrderAccess::release\_store， OrderAccess::load\_acquire又是什么东西呢。

OrderAccess就是openjdk8路径/hotspot/src/share/vm/runtime下的orderAccess.hpp文件。在orderAccess.hpp的代码头部注释里有着对这些方法的详细描述。

注释里是这么说的：

> acquire 等价于LoadLoad屏障加上 LoadStore屏障。  
> load\_acquire 等价于 load + acquire ，即等价于load指令 +LoadLoad屏障 + LoadStore屏障。  
> 应证了前文中我们说到volatile属性的读，在读指令后加上了LoadLoad屏障、LoadStore屏障两个屏障。

同理，

> release等价于LoadStore屏障加上 StoreStore屏障。  
> release\_store 等价于 release + store ，即等价于LoadStore屏障+StoreStore屏障+store指令。  
> 应证了前文中我们说到volatile属性的写，在写指令前面加上了LoadStore屏障、StoreStore屏障两个屏障。

而

> OrderAccess::release\_store的下一行跟着OrderAccess::storeload 指令。  
> 即印证了写指令前面加上LoadStore屏障、StoreStore屏障两个屏障，写指令后面加上StoreLoad屏障，与虚拟机规范对应上了。  
> 而读指令后面会加上LoadLoad屏障和 LoadStore屏障两个屏障，也与前文中的虚拟机规范对应上了。

* * *

OrderAccess可以理解为是一个接口，根据不同操作系统不同CPU对应不同的实现。

六、从x86[架构](https://so.csdn.net/so/search?q=%E6%9E%B6%E6%9E%84&spm=1001.2101.3001.7020)下看内存屏障
--------------------------------------------------------------------------------------------

从上图可以看到有多个操作系统及cpu架构的实现，这里以x86为例进一步了解。（因为x86是最常见的）

OrderAccess在linux系统，x86架构下的实现是上图中的orderAccess\_linux\_x86.inline.hpp。部分代码如下所示:

    inline jbyte    OrderAccess::load_acquire(volatile jbyte*   p) { return *p; }
    inline jshort   OrderAccess::load_acquire(volatile jshort*  p) { return *p; }
    inline jint     OrderAccess::load_acquire(volatile jint*    p) { return *p; }
    ......
    .......................................................
    inline void     OrderAccess::release_store(volatile jbyte*   p, jbyte   v) { *p = v; }
    inline void     OrderAccess::release_store(volatile jshort*  p, jshort  v) { *p = v; }
    inline void     OrderAccess::release_store(volatile jint*    p, jint    v) { *p = v; }
    ......


如上，OrderAccess::release\_store和OrderAccess::load\_acquire方法，最终的实现是使用C++的volatile关键字。

C++的volatile关键字和java的语义是不同的，C++的volatile关键字表示变量每次都从主存里读不从cpu缓存读，且禁止编译器做重排序之类的优化。

OrderAccess::storeload 这部分代码如下：

    inline void OrderAccess::storeload()  { fence(); }
    inline void OrderAccess::fence() {
      //如果是多核CPU（multi-processing）
      if (os::is_MP()) {
        // always use locked addl since mfence is sometimes expensive
    #ifdef AMD64
        __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
    #else
        __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
    #endif
      }
    }


从上述代码我们可以看出，是多核处理器才会执行处理，单核就不需要内存屏障了。实际指令是lock; addl $0,0(%%esp)。rsp是esp对应的64位指令。

addl $0,0(%%rsp) 的意思就是把寄存器里的值加0，也就是说这是个空操作。重点在于lock; addl $0,0(%%rsp)里的lock前缀。

我们再回想起来上一篇文章中MESI中的内容。加了lock前缀的指令会严格保证MESI协议中的数据一致性，保证对某个内存的独占使用，保证该CPU对应缓存行为独占，其他CPU的缓存行则失效。在 x86 上，任何带 lock 前缀的指令都可以可以当成一个 StoreLoad 屏障。

再看x86架构中其他的内存屏障指令：

    inline void OrderAccess::loadload()   { compiler_barrier(); }
    inline void OrderAccess::storestore() { compiler_barrier(); }
    inline void OrderAccess::loadstore()  { compiler_barrier(); }
    inline void OrderAccess::storeload()  { fence();            }
    
    inline void OrderAccess::acquire()    { compiler_barrier(); }
    inline void OrderAccess::release()    { compiler_barrier(); }
    static inline void compiler_barrier() {
      __asm__ volatile ("" : : : "memory");
    }


其他内存屏障的实现都是compiler\_barrier()。compiler\_barrier的实现是伪指令“memory”。

memory是一个编译屏障，编译屏障告诉编译器不能把“memory”执行前后的代码混淆在一起。编译屏障仅作用于编译时，在实际运行时，可以理解为一个空的指令。内存屏障都会带有编译屏障的效果。

所以，在x86环境下，只有StoreLoad 屏障是有实际操作的。

x86架构cpu使用tso一致性模型，上文已经说明了，只有StoreLoad会发生乱序。该处和前文呼应上了。

七、实际汇编下的内存屏障
------------

通过JITWatch查看上文代码和字节码实际解释成汇编代码是什么样子的。

以我本地windows系统intel i5的cpu（x86）为例，使用c2编译器进行编译的结果如下：

《图片丢了》

上图左侧是java源码，中间是class字节码，右侧是c2的汇编代码。

从上图可以看到，最终生成的StoreLoad屏障指令是lock addl 0 x 0 , ( 0x0,( 0x0,(rsp)。

而其他屏障指令并没有相关指令生成。此处也就呼应了jdk源码。

八、总结
----

本文先介绍cpu的几种内存一致性模型，引出了java为何以及如何在上层抽象的四种内存屏障。

再通过阐述JMM模型的三个准则，讲述java如何根据场景插入内存屏障，以及为何要在这些场景插入内存屏障的原因。

最后说明java语言编译时如何插入内存屏障，分析volatile关键字从字节码层面到jdk源码层面再到汇编层面的整个实现原理。

可以看出来，java语言设计的巧妙：

> *   对上层提供简单明了的编程范式给程序员使用，内存模型简单好理解，volatile修饰符编程简单方便。
> *   对底层，屏蔽掉不同底层实现的差异，尽量少地去束缚编译器和处理器:
>
> 只有必要的场景才会通过插入内存屏障的方式，保证处理器能够提供正确的内存可见性。
>
> 不care你处理器会如何进行优化提高性能而打乱我代码执行顺序，只要你别改变我程序的执行结果就行（单线程及正确的多线程编程获得正确的执行结果）。

写在最后，在写本文时用了很多时间在试图弄懂CPU四种乱序的原因（主要是LoadStore）。翻阅了许多相关资料，很多硬件细节压根找不到啥资料，最终其实也没有搞明白，有些钻牛角尖的味道了。