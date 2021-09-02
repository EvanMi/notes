# dailies

## rpc超时设置

作为客户端，如何实现超时？

不管是如何进行socket的请求，都会把请求操作进行异步，然后通过future模式来获取值。

在future模式中会提供指定超时时间方法，在该方法中真正的请求在上述的异步执行中，而本线程会使用wait(超时时间)的方式进行阻塞。阻塞以后有两种方式被唤醒：

（1）wait的时间到了，自我唤醒。那么此时一般是超时的，会判断一下结果是产出来返回是否超时。

（2）被异步的执行唤醒（notify），这时一般会有有效的返回结果。此时会根据结果来判断是继续进行等待、返回结果等判断。

设置超时时间是为了解决什么问题？

从宏观上来说，是为了确保服务链路的稳定性，提供了一种框架级别的容错能力。

（1）不设置超时时间就会无限制的等待下去，造成整个应用的瘫痪。尤其是一些核心的业务，在调用一些非核心业务的时候，如果有超时时间就能在非核心业务响应过慢的时候，在等待时间达到后，进行降级，从而避免了核心业务被非核心业务拖死。

（2）服务的不可用可能是应为瞬间的网络抖动或者高负载引起的，如果超时后直接放弃则可能对业务造成损害。所以进行重试的话可以进行挽救。

引入超时机制后带来的副作用？

（1）重复请求：可能服务端已经执行完成了，客户端超时重试就会再次执行。所以服务端需要保障幂等性。

（2）可能降低调用者的负载能力：也就是说服务端真的发生故障的时候，调用者会不断的重试造成整体性能的下降。如果consumer是一个QPS很高的服务，那么必然造成连锁的雪崩反应。

（3）重试风暴：A->B->C->D 当D出现问题时，A,B,C都在重试，就形成了重试风暴。

## Mac 根目录无法访问处理

sudo vim /etc/synthetic.conf

写入 data	/Users/tal/data (中间是tab)

保存后重启电脑，根目录下就会出现data文件夹了。 data -> /Users/tal/data

## brew切换镜像源

```
使用国内镜像源
# 步骤一
cd "$(brew --repo)"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git

# 步骤二
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git

#步骤三
brew update

还原
cd "$(brew --repo)"
git remote set-url origin https://github.com/Homebrew/brew.git
 
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://github.com/Homebrew/homebrew-core
 
brew update
```



## linux 命令收集

ss -lnt  # 查看socket状态

## 架构图

系统架构图是为了抽象的表示软件系统的整体轮廓、各个组件之间的相互联系、约束边界以及以及系统软件的物理部署和软件系统的演进方向的整体视图。

4+1视图

```
场景视图、逻辑视图、物理视图、处理流程视图和开发视图。

- 场景视图
场景视图用于表示系统的参与者与功能用例的关系，反映系统的最终需求和交互设计，通常由用例图进行表示。

- 逻辑视图
逻辑视图用于描写系统功能拆解后的组件关系、组件约束和边界，反映系统整体组成与系统如何构建的过程，通常由uml的组件图和类图来表示。

- 物理视图
物理视图用来描述系统软件到物理硬件的映射关系，反映出系统的组件是如何部署到一组可用的计算机节点上，用于指导软件系统的实施和部署过程。

- 处理流程视图
处理流程视图用于描述系统软件之间的的通信时序，数据的输入和输出，反映系统的功能流程和数据流程，通常由时序图和流程图来表示。

- 开发视图
开发视图用于描述系统的模块划分和组成，以及细化到内部包的组成设计，服务与开发人员，反映系统开发实施过程。


```



4c 画法

1、语境图(System Context Diagram)

![alt](imgs/system_context.png)

这样一个简单的图，可以告诉我们，要构建的系统是什么，它的用户是谁，要如何融入当前已有的IT环境。

怎么画？中间是自己的系统，周围是用户和其他与之相互作用的系统。这个图的关键就是梳理清楚带建设的系统的用户和高层次的依赖。

2、容器图(Container Diagram)

容器图是把语境图里面待建设的系统做了一个展示。

![alt](imgs/container_diagram.png)

这个图展现了软件系统的整体形态；体现了高层次的技术决策；系统中的职责是如何分布的，容器间是如何交互的；告诉开发者在哪里写代码。

怎么画？用一个框图来表示，内部可能包括名称、技术选择、职责，以及这些框图之间的交互，如果涉及外部系统，最好明确边界。

3、组件图(Component Diagram)

![alt](imgs/component_diagram.png)

组件图是把某个容器进行展开，描述其内部的模块。

这张图描述了系统由哪些组件/服务组成，厘清了组件之间的关系和依赖，为软件开发如何交付提供了框架。

4、类图(code/class diagram)

## MESI

原文地址：https://www.jianshu.com/p/0e036fa7af2a

| 状态         | 描述                                                         |
| :----------- | ------------------------------------------------------------ |
| M(Modifed)   | 这行数据有效，数据被修改了，和内存中的数据不一致，数据只存在本Cache中。 |
| E(Exclusive) | 这行数据有效，这行数据和内存中的数据保持一致，数据只存在本Cache中。 |
| S(Shared)    | 这行数据有效，这行数据和内存中的数据保持一致，数据存在于很多Cache中。 |
| I(Invalid)   | 这行数据无效。                                               |

状态E

![alt](imgs/mesi_e.png)

状态S

![alt](imgs/mesi_s.png)

状态M I

![alt](imgs/mesi_mi.png)

## CPU草构

原文地址：https://zhuanlan.zhihu.com/p/48157076

![alt](imgs/cpu.png)

cpu会将对cache的修改直接插入到storeBuffer中(如果缓存行的状态为M，E，那么就可以直接修改不会放到storeBuffer中。因为不需要通知其他cpu)；同时cpu在读取缓存的时候，会先存storeBuffer中读取，从而避免在storeBuffer中的修改读不到的问题。

当storeBuffer满了以后就会将buffer中的内容刷到cache中，从而触发cacheline的invalidate message，这些message会一起发送给其他的core，然后等到其他的core返回invalidate ack之后才能继续向下执行。为了进一步解耦，有引入了invalidate queue。

从SC中得到的保障：

**If L(a) <p L(b) ⇒ L(a) <m L(b) /\* Load→Load \*/**

**If L(a) <p S(b) ⇒ L(a) <m S(b) /\* Load→Store \*/**

**If S(a) <p S(b) ⇒ S(a) <m S(b) /\* Store→Store \*/**

**If S(a) <p L(b) ⇒ S(a) <m L(b) /\* Store→Load \*/**

每个共享内存的读取肯定是其在memory order中最近的一次写入的值。

### 内存屏障

smp_mb()是一种fullmemory barrier，会将store buffer 和 invalidate queue都flush一遍。

read memory barrier 会 flush invalidate queue。

write memory barrier 会 flush store buffer。



### program order VS memory order

program order：就是我们写的代码的顺序，这个是静态的也是每个CPU core各自拥有的。

memory order：就是代码执行的顺序，这个是全局的，每个CPU core对共享内存的执行都会出现在memory order中。

### SC (sequence consistency) definition

the result of any execution is the same as if the operations of all processors (cores) were executed in some sequential order, and the operations of each individual processor (core) appear in this sequence in the order specified by its program.

```
在任意的执行过程中，一些列的操在多核之间以某种线性顺序执行，同时同一个核中的所有的操作按照他自身程序所指定的顺序执行。
```

### TSO(Total Store Order)

在SC基础上的变化：

不保证storeload顺序 ：Core C1中S1和L1， S1先去L1执行，但是S1只是将值送入了write buffer就返回了，紧接着执行L1，L1在memory order中的点执行完之后，S1的write buffer这时候flush到内存，那么S1在memory order这条线上真正执行的点在L1之后了，那么这时候S1与L1就出现了reorder了。

load的最新值不一定是memory order中最近的，有可能是program order最近的store。

## 网络分区

网络分区分为对称网络分区与非对称网络分区，三个节点A，B，C，节点A是leader，A与B,C网络分区，这种是对称网络分区。如果A与B网络正常，与C网络分区，但是B与C之间网络正常，这种属于非对称网络分区。

对称网络分区

![alt](imgs/syn_network.png)

leader与follower出现非对称网络分区

![alt](imgs/leader_asyn_network.png)

follower与其他节点出现非对称分区

![alt](imgs/follower_asyn_network.png)

## 常见性能指标

QPS(Queries Per Second) : 每秒查询数。

TPS(Transactions Per Second): 每秒处理的事务数。

QPS 和 TPS的区别：

一个例子来说明：访问一个index页面会请求服务器3次，包括一次html，一次css，一次js，那么访问者一个页面就会产生一个“T”，产生三个“Q”。

并发数（并发度）：指系统同时能处理的请求数量，同样反映了系统的负载能力。这个数值可以分析机器1s内的访问日志数量来得到。

吞吐量：吞吐量是指系统在单位时间内处理请求的数量，TPS QPS都是吞吐量的量化指标。

PV(Page View) ：页面访问量，即页面浏览量或点击量，用户每次刷新就被计算一次。可以统一服务器一天的访问日志得到。

UV(Unique Visitor) : 独立访客，统计1天内访问某站点的用户数。

## Java Type

1.ParameterizedType

ParameterizedType表示参数化类型，也就是泛型，例如List<T> Set<T>等。

有三个方法，

1.1 getActualTypeArguments

获取泛型中的实际类型，可能会存在多个泛型，例如Map<K,V>，所以会返回Type[]数组。值得注意的是，无论<>中有几层嵌套（List<Map<String,Integer>）,该方法返回的都是脱去最外层的<>以后的内容（Map<Stirng,Ingeger>）。

1.2 getRawType

获取声明泛型的类或者接口，也就是泛型中<>前面的值。Map<String,Integet>返回 java.util.Map。

1.3 getOwnerType

获取内部类的拥有者的类型，例如Map.Entry<String,String> 会返回 java.util.Map。

2.GenericArrayType

泛型数组类型，例如List<String>[] T[] 等。

2.1 getGenericCompontType 

返回泛型数组中的元素的Type类型，即List<String>[] 中的 List<String>(ParamerterizedTypeImpl)、T[]中的T（TypeVatiableImpl）。无论是几维数组，该方法都只会删除最右边的[]，返回剩余的内容。

3.TypeVariable

泛型的类型变量，指的是List<T> , Map<K,V>中的T,K,V等值。

3.1getBounds

获得该类型变量的上限，也就是泛型中extends右边的值；例如List<T extends Number> 返回的是Number；没有指明默认返回Object。值得注意的是，类型变量的上限可以为多个，必须使用&符号相连接，例如 List<T extends Number & Serializable>；其中，& 后必须为接口。

3.2getGenericDeclaration

获取声明该类型变量的实体，例如 Map<K,V>中的Map。而其中的GenericDeclaration（可以是Class Constructor Method）。

3.3getName

获取类型变量在源码中定义的名称。

4.Class 

5.WildcardType

？---通配符表达式，表示通配符泛型，但是WildcardType并不属于Java-Type中的一种；例如：List<? extends Number> 和 List<? super Integer>。

5.1getUpperBounds

例如 List<? extends Number> 返回 Number

5.2getLowerBounds

例如 List<? super String> 返回 String

## 泛型擦除

泛型实参只会在类、字段及方法参数内保存其签名，无法通过反射动态获取泛型实例的具体实参。

大白话就是，在编译为字节码后具体的.class文件中会保存该class的类、字段及方法参数上的泛型签名，定义的时候什么样获取的时候就是什么样。

在JVM中，方法体内中定义的***<u>对象</u>***的泛型、field上的泛型都会被擦除，变成类型转换。



如何获取泛型实参？

1、通过传递实参类型 ---- 也就是明确通过传递class的方式指定

2、明确定义泛型实参类型，通过反射获取签名 ---- 说白了，就是写代码的时候就在在类、字段、方法参数上定义了具体的泛型，利用了签名会被保存的原理。

3、通过匿名类捕获相关的泛型实参 ---- 创建了匿名类，jvm会为它生成class文件，也就能够保存签名信息了。

### 一个有趣的泛型擦除实验

```java
public abstract class  A<T> {
    public T a(T t) { return t;}
}

public abstract class B<T> extends A<T> {

    abstract public void b(T t);
}

public class C extends B<String> {

    @Override
    public void b(String s) {

    }
}


```

在C中方法b已经明确的将B中的方法b的泛型T变成了String。所以通过方法参数可以直接获取泛型的信息（1.8以前），也可以通过类签名获取泛型的信息或者方法参数的签名信息获取泛型的信息（1.8）。

同时在进行实验的过程中，发现在C中有两个b方法，具体如下

```
Classfile /Users/mipengcheng/IdeaProjects/JavaHamcrest/learn/build/classes/java/test/com/mpc/hamcrest/C.class
  Last modified 2021-8-29; size 568 bytes
  MD5 checksum 863b227ebd48874fbc4e581d32601a26
  Compiled from "C.java"
public class com.mpc.hamcrest.C extends com.mpc.hamcrest.B<java.lang.String>
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #5.#22         // com/mpc/hamcrest/B."<init>":()V
   #2 = Class              #23            // java/lang/String
   #3 = Methodref          #4.#24         // com/mpc/hamcrest/C.b:(Ljava/lang/String;)V
   #4 = Class              #25            // com/mpc/hamcrest/C
   #5 = Class              #26            // com/mpc/hamcrest/B
   #6 = Utf8               <init>
   #7 = Utf8               ()V
   #8 = Utf8               Code
   #9 = Utf8               LineNumberTable
  #10 = Utf8               LocalVariableTable
  #11 = Utf8               this
  #12 = Utf8               Lcom/mpc/hamcrest/C;
  #13 = Utf8               b
  #14 = Utf8               (Ljava/lang/String;)V
  #15 = Utf8               s
  #16 = Utf8               Ljava/lang/String;
  #17 = Utf8               (Ljava/lang/Object;)V
  #18 = Utf8               Signature
  #19 = Utf8               Lcom/mpc/hamcrest/B<Ljava/lang/String;>;
  #20 = Utf8               SourceFile
  #21 = Utf8               C.java
  #22 = NameAndType        #6:#7          // "<init>":()V
  #23 = Utf8               java/lang/String
  #24 = NameAndType        #13:#14        // b:(Ljava/lang/String;)V
  #25 = Utf8               com/mpc/hamcrest/C
  #26 = Utf8               com/mpc/hamcrest/B
{
  public com.mpc.hamcrest.C();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method com/mpc/hamcrest/B."<init>":()V
         4: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/mpc/hamcrest/C;

  public void b(java.lang.String);
    descriptor: (Ljava/lang/String;)V
    flags: ACC_PUBLIC
    Code:
      stack=0, locals=2, args_size=2
         0: return
      LineNumberTable:
        line 15: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  this   Lcom/mpc/hamcrest/C;
            0       1     1     s   Ljava/lang/String;

  public void b(java.lang.Object);
    descriptor: (Ljava/lang/Object;)V
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: checkcast     #2                  // class java/lang/String
         5: invokevirtual #3                  // Method b:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/mpc/hamcrest/C;
}
Signature: #19                          // Lcom/mpc/hamcrest/B<Ljava/lang/String;>;
SourceFile: "C.java"
```

有Object和String两种入参的b方法，而Object入参的会强制转换类型后调用String入参的方法。同时调用isSynthetic来判断发现Object入参的方法是jvm生成的。

为什么会有这样的一个以Object为入参的方法出现呢？答案就在类B中，类B中的b方法会被泛型擦书为以Object为入参。

这样来说真正的C要覆盖的方法是Object入参的b，所以jvm只能生成一个方法来覆盖，然后委托给真正的以String为入参的实现方法来实现功能。从而在多态的情况下，定义 B b = new C()的时候，调用b.b("x")能够成功的被调用。

## session的进化

1、单机的时候session很简单，把session_id保存在cookie或者url中，而用户的权限信息保存在服务端。每次请求通过sessionId来获取用户的信息。

2、在有了集群以后，可以使用粘性session，也就是通过负载均衡将同一个客户的请求映射到一台服务器，session信息还是保存在服务端，同样通过session_id来获取。这样的结果导致对一台服务器更新的时候就会丢失用户的登录信息。

3、为了克服以上的问题，可以有三种办法：

（1）session集中存储，例如保存在redis中。

（2）session复制：例如tomcat集群支持session复制。

（3）客户端session：也就是将session信息保存在客户端，通过服务端解密信息后获取用户的登录信息。

4、以前的认证都是各个服务器自己做自己的，在微服务的情况下，自己做自己的就有点冗余了。所以专门抽离了认证中心进行认证和保存用户信息。用户在认证中心认证，微服务通过认证中心获取用户信息。（类比1）

5、JWT 使得微服务本身可以不用访问认证中心就可以获取用户信息。（类比3-(3)）。

## spring

### 为什么要三级缓存

一个重要的结论：

当某个bean进入2级缓存的时候，说明这个bean的早期对象被其他bean注入了，也就是说，这个bean还是一个半成品，还未完成创建的时候，已经被别人拿去使用了，所以必须要有3级缓存，2级缓存中存放的是早期被别人使用的对象，如果没有2级缓存，是无法判断这个对象在创建过程中，是否被别人拿去使用了。

三级缓存是为了判断循环依赖的时候，早期暴露出去已经被别人使用的bean和最终的bean是否是同一个bean，如果不是同一个则弹出异常，如果早期的对象没有被其他bean使用，而后期被修改了，不会产生异常。如果没有三级缓存，就无法判断是否有循环依赖，且早期的bean被循环依赖中的bean使用了。

同时三级缓存缓存一个objectFactory，里边会调用getEarlyBeanReference，也就是通过这个修改早期返回的bean是哪个。这个实际使用的场景就是AOP。