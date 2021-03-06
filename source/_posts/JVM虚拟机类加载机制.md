---
title: JVM虚拟机类加载机制
date: 2017-07-17 00:00:00
tags:
- JVM
categories: 
- JAVA
---

本文主要介绍了编译好的字节码是如何加载到虚拟机中的，在加载的过程中又发生了什么，虚拟机要做哪些准备和检查。

<!--more-->

# 虚拟机类加载机制

​    现在我们来学习一下JVM虚拟机中类的加载机制，也就是一个类是如何被加载进入虚拟机中的，在加载的过程中虚拟机会完成哪些工作。需要注意一点，这里我们说的类即Class并不是狭义的.class 文件，而是指的是一串二进制字节流，它具有一定的格式，可以被虚拟机识别。它可以来自.class文件，也可以来自jar包，也可以是从网络上下载下来的一段代码。下面首先来介绍一下类的加载时机。

## 类的加载时机

​    类从加载到虚拟机内存中到卸载出内存为止，整个生命周期包括七个阶段，这七个阶段分别是加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）和卸载（Unloading）七个阶段，其中验证准备解析三个部分统称为连接，七个阶段的发生顺序如图所示：

![img](ClassLoading.png)

​    在图中，加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，但是类的解析阶段则不一定，这里解析我们可以认为是获得类、方法等的实际内存地址的一个过程（具体内容后面会讲），这是为了支持Java中的动态绑定。这里我们说的五个阶段的执行顺序是确定的只是说他们的开始时间必须要按照这个顺序，但是一个过程的开始不必等到另一个过程结束，这些阶段通常都是相互交叉地混合式的进行的。

​    Java虚拟机规范中没有对类的加载时机作出强制性的要求，但是对于初始化阶段进行了严格的规定，有且只有五种情况下必须立即对类进行“初始化”，这五种情况分别是：

- 遇到new、getstatic、putstatic或者invokestatic这四条字节码指令时，如果没有进行过初始化，需要先触发初始化。这些指令是字节码指令，在什么情况下会生成这些字节码指令呢？使用new关键字实例化对象的时候、读取或者设置一个类的静态字段的时候（被final修饰、已经在编译期把结果放入常量池的静态字段除外），以及调用一个类的静态方法的时候。

- 使用java.lang.reflect包的方法对类进行反射调用的时候。

- 当初始化一个类的时候，需要先初始化父类。

- 当虚拟机启动时需要先初始化包含main方法的那个类。

- 当使用JDK1.7的动态语言支持的时候，如果一个java.lang.invoke.MethodHandle实例最后的解析结果是REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要首先触发其初始化。

这五种场景被称为对一个类的主动引用，除此之外所有引用类的方式都被称为被动引用，它们都不会触发初始化。下面举三个例子来帮助大家理解：
```
    public class SuperClass{
        static{;
          System.out.println("hello superclass");
        };
    	public static String a = "hello";
    
    }
    
    public class ChildClass extends SuperClass&{
    	static{
          System.out.println("hello childclass");
    	}
    }
    
    public class Initialization{;
      	public static void main(String[] args){;
          System.out.println(ChildClass.a);
    	}  
    }
```

​    在这里我们省略了package语句和import语句。在这个例子中，我们运行之后发现，只有父类被初始化了。这是因为静态变量a是定义在父类中的，这里是读取了父类的静态字段，对应于上面的第一条。所以只有父类被初始化了。

 下面一个例子是：

`SuperClass[] sca = new SuperClass[10];`

​    在这个例子中，SuperClass仍然没有被初始化，但是在实际执行过程中，初始化的是另一个类，这个类由虚拟机自动生成，名称同样是SuperClass，这个类对用户代码来说不可见，直接继承于Object。另外，创建动作由字节码`newarray`触发。

​    这个自动生成的类代表了一个元素类型为SuperClass（用户创建的类）的一维数组，数组中应有的属性和方法都在这个类中实现，也就相当于虚拟机帮我们创建了一个SuperClass的数组类，对数组的操作都转换成这个类中的方法，因此相对于使用数组指针来说可以更加方便地进行越界检查等，这也是相对于C/C++数组更加安全的原因。

​    最后一个例子也是关于上面第一条的。我们知道不管是内部类还是其他类，在生成字节码文件的时候，每个类都会单独生成一个.class文件，一般的类是`类名.class` 内部类是`$类名.class` ，若类A中引用类B中定义的一个常量a，我们假定a的定义如下：

`public static final int a = 0;`

​    那么B类中的a在编译期会被复制到类A中的常量池中，带来的影响就是，A中对a的引用，实际上已经不是对B中a的引用，而是对自己常量池中同名常量a的引用。当我们在A中使用a时，不会触发B类的初始化。（更具体的说，A与B的class文件在编译完成后已经没有任何联系）

​    接口的初始化过程与类只有第三条不同，接口在进行初始化时，不要求父接口先初始化，而是真正使用到父接口的时候才会初始化。

## 类加载的过程

### 加载

 在加载阶段，虚拟机主要完成以下三件事情

- 通过一个类的全限定名来获取此类的二进制字节流
- 将字节流代表的静态存储结构转化为方法区的运行时数据结构
- 
在内存中生成一个代表这个类的java.lang.Class对象，作为方法区的这个类的各种数据的访问入口

​正如我们之前提到的，这里的二进制字节流可以来自.class文件，也可以来自zip文件（也就是我们日后的JAR、EAR、WAR格式的基础）、网络中获取（Applet）或者是运行时计算生成等等。

对于数组类而言，他本身并不是类加载器创建的，而是Java虚拟机直接创建的。但是数组类与类加载器仍然有很密切的关系，因为数组类的元素类型（Element Type，指的是数组去掉所有维度的类型）最终是由类加载器加载的，一个数组类（下面简称为C）的创建过程遵循以下的规则：

- 
如果数组的组件类型（Component Type，指的是数组去掉一个维度的类型）是引用类型，那就递归采用本节中定义的加载过程去加载组件类型，数组C将在加载该 组件类型 的 类加载器 的 类名称空间 上被标识。

- 如果组件类型不是引用类型，虚拟机将会把数组C与引导类加载器关联
- 数组类的可见性与它的组件类的可见性一致，如果组件类型不是引用类型，那数组类的可见性将默认为public

### 验证

​    验证阶段的目的是为了保证一方面class字节流是符合当前虚拟机的要求的，另一方面不会危害虚拟机自身的安全。验证的过程又包含以下几个阶段：

- 文件格式验证：这一阶段的验证是为了保证字节流能正确解析并存储于方法区之内，格式上符合描述一个Java类信息的要求（魔数、主次版本号等等），这一阶段是对字节流进行验证，之后，字节流将被转换成方法区的动态数据结构，后面的检查都是基于这个数据结构进行的。
- 元数据验证：第二阶段是对方法区的数据结构进行语义分析，也就是对类的元数据进行语义校验，保证符合Java语言规范。
- 字节码验证：这一个阶段是最复杂的一个阶段，保证类不会做出危害虚拟机安全的事件。
- 符号引用验证：这个阶段的验证将发生在虚拟机将符号引用转换成直接引用的时候。通俗的来讲是确保能够找的到所引用的符号。

### 准备

​    准备阶段是为类变量分配内存并设置初始值的阶段，变量所使用的内存将在方法区内进行分配，有三点需要说明：

- 进行内存分配的仅仅是类变量（被static修饰的），不包括实例变量，实例变量将在对象实例化的时候在Java堆中进行分配。
- 初始值并不是程序中设定的初始值。比如说`public static int value = 9;` 这里初始化之后的value是0，而不是9。这是因为将value赋值为9的语句存放在类构造器中，所以这些动作将在初始化阶段完成。
- 但是如果value是final修饰的，也就是说后面不会有变化，在准备阶段会完成初始化，赋值为9。

### 解析

​    解析阶段就是虚拟机将常量池内的符号引用替换为直接引用的过程。在这里我们首先来介绍一下符号引用和直接引用

- 符号引用：符号引用以一组符号来描述所引用的目标。符号引用与虚拟机实现的内存布局无关，所引用的目标不一定加载到内存中。
- 直接引用：直接引用可以是指向目标的指针、相对偏移量或者是一个间接定位到目标的句柄。直接引用是和虚拟机的内存布局相关的，同一个符号引用在不同的虚拟机上翻译成的直接引用一般不同。所引用的目标必定已经在内存中存在。

### 初始化

​    初始化阶段就是根据程序员通过程序制定的主观计划去初始化类变量和其他资源。

## 类与类加载器

​    对于任意一个类，都需要由类本身和它的加载器来确定类在虚拟机中的唯一身份，换句话说，即使是同一个类，只要加载他们的虚拟机不同，那么他们在虚拟机看来，就不是同一个类。这里说的“同一”包括Class对象的`equals()`方法、`isAssignableFrom()`方法、`isInstance()`方法的返回结果也包括使用`instanceof`关键字的判定结果。

## 双亲委派模型

​    说到双亲委派模型，首先要介绍一下下面这张图

![img](16465fe65c8f85b4.png)

​    这张图里面有四类ClassLoader，下面一一介绍。

- Bootstrap ClassLoader: 启动类加载器，使用C++实现，是虚拟机自身的一部分（其余类加载器都是Java语言实现，独立于虚拟机外部）。负责加载\lib 目录中的或者被-Xbootclasspath参数所指定的路径中的且是虚拟机识别的（仅仅按照文件名识别，名字不符合的不会加载）加载到虚拟机内存中。无法被Java程序直接引用。
- Extension ClassLoader：扩展类加载器，负责加载\lib\ext目录中的或者被java.ext.dirs系统变量指定的路径中的所有类库，开发者可以直接使用。
- Application ClassLoader：应用程序类加载器，负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器。如果没有自定义过类加载器，这个加载器就是默认加载器。
- User ClassLoader：自定义类加载器。

​    那么说完这几个类加载器之后双亲委派模型就很简单了。当一个类加载器要加载一个类时，他首先会把这个加载任务托付给他的父类来完成，如果父类无法完成的话再由子类来完成。因此每一个加载请求都会从上到下进行尝试加载，这样做的好处是显而易见的，它可以保证**Java类随着它的类加载器一起具备了一种带有优先级的层次关系**。前面提到过一个类的身份由其本身和它的类加载器一同决定，双亲委派模型就保证了同一优先级的类具有相同的类加载器，在虚拟机看来，他们都是“同一”个。 
