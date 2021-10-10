
##. JVM 以及垃圾回收器
![](https://xiazhenyu.oss-cn-hangzhou.aliyuncs.com/JVM%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E4%BB%A5%E5%8F%8AGC%E7%9B%B8%E5%85%B3%E7%9F%A5%E8%AF%86%E7%82%B9.jpg)
   
## 类加载机制

### 类的加载过程

类的加载过程非常的复杂，主要有这几个过程：加载、验证、准备、解析、初始化。

1. 加载

加载的主要作用是将外部的 .class 文件，加载到Java的方法区内。加载阶段主要是找到并加载类的二进制数据，比如从jar包或者war包里找到它们。

2. 验证

肯定不是任何的.class 文件都能加载，那样太不安全了，容易收到恶意的代码的攻击。验证阶段在虚拟机整个类加载过程中占了很大的一部分，不符合规范的将抛出java.lang.VerifyError错误。像一些低版本的JVM是无法加载一些高版本的类库的，就是在这个阶段完成的。

3. 准备

从这部分开始，将为一些类变量分配内存，并将其初始化为默认值。此时，实例对象还没有分配内存，所以这些动作是在方法区上进行的。

4. 解析

解析在类加载中是非常重要的一环，是将符号引用替换为直接引用的过程。
符号引用是一种定义，可以是任何字面上的含义，而直接引用就是直接指向目标的指针、相对偏移量。
直接引用的对象都存在于内存中，你可以把通讯录里的女友的手机号码类比为符号引用，把面对面和你吃饭的人类比为直接引用。
解析阶段负责把整个类激活，串成一个可以找到彼此的网，过程不可谓不重要。那这个阶段都做了哪些工作呢？大体可以分为：
* 类或接口的解析
* 方法的解析
* 接口方法的解析
* 字段解析
  我们来看几个经常发生的异常，就与这个阶段有关：。
* java.lang.NoSuchFieldError 根据继承关系从下往上，找不到相关的字段时的报错；
* Java.lang.IllegalAccessError字段或者方法，访问权限不具备时的错误；
* Java.lang.NoSuchMethodError 找不到相关方法的错误；
  解析过程保证了相互引用的完整性，把继承与组合推荐到运行时。

5. 初始化

如果前面的流程一切顺利的话，解析来该初始化成员变量了，到了这一步，才真正开始执行一些字节码。
下面的代码输出结果会是什么：  
``    public static class A {

        static int a = 0;

        static {
            a = 1;
            b = 1;
        }

        static int b = 0;

        public static void main(String[] args) {
            System.out.println(A.a);
            System.out.println(A.b);
        }
    }``


结果是1 0。 a和b的唯一区别就是它们的static代码块的位置。
这就引出一个规则：static 语句块，只能访问到定义在static语句块之前的变量。

第二个规则：JVM会保证在子类的初始化方法之前，父类的初始化方法已经执行完毕。
所以，JVM第一个被执行的类初始化方法一定是java.lang.Object。另外，也意味着父类中定义的static语句块要优先于子类的。

<clint> 与<init>

看下面的代码：

    public class A {

        static {
            System.out.println("1");
        }

        public A() {
            System.out.println("2");
        }
    }

    public class B extends A {

        static {
            System.out.println("a");
        }

        public B() {
            System.out.println("b");
        }

        public static void main(String[] args) {
            A ab = new B();
            ab = new B();
        }
    }




结果：
`1
a
2
b
2
b`

如下图所示。其中static字段和static代码块，是属于类的，在类的加载的初始化阶段就已经被执行了。类的信息会被放在方法区，在同一个类加载器下，这些信息有一份就够了，所以上面的static代码块只会执行一次，它对应的是<clint>方法。


而对象的初始化就不一样了。通常，我们在new一个对象的时候，都会调用它的构造方法，就是<init>，用来初始化对象的属性。每次新建对象的时候，都会执行。
所以上面的static代码块只会执行一次，对象的构造方法会执行两次。再加上继承关系的先后原则，不难分析出正确的结果。

### 类加载器

整个类的加载过程非常的繁重。类加载器做的就是上面的5个步骤的事情。

下面是几个不同等级的类加载器。
* Bootstrap ClassLoader
  这个是加载器的大Boss，任何类的加载行为，都要经过它。它的作用是加载核心类库，也就是rt.jar、resources.jar、charsets.jar等。当然这些jar包的路径是可以指定的。-Xbootclasspath参数可以完成制定操作。
  这个加载器是C++编写的，随着JVM而启动。
* Extention ClassLoader
  扩展类加载器，主要用作加载lib/ext目录下的jar包和.class文件。同样的，通过系统变量java.ext.dirs可以指定这个目录。
  这个加载器是个Java类，继承自URLClassLoader。
* App ClassLoader
  这是我们编写的Java类的默认加载器，有时候也叫做System ClassLoader。一般用来加载classpath下的其他的所有的jar包和.class文件。我们编写的代码，首先会使用这个类加载器进行加载。
* Custom ClassLoader
  自定义加载器，支持一些个性化的扩展功能。

### 双亲委派机制

双亲委派机制的意思是除了顶层的启动类加载器以外，其余的类加载器，在加载以前，都会委派给它的父类加载器进行加载，这样一层一层向上传递，直到祖先们都无法胜任，它才会真正的加载。


需要注意的是，ClassLoader#loadClass 方法是可以被覆盖的，也就是双亲委派机制并不一定生效。

这个模型的好处在于Java类已经有了一种优先级的层次划分关系。比如Object类，这个类毫无疑问是给最上层的加载器进行加载，即使是你覆盖了它，最终也是有系统默认的加载器进行加载的。
如果没有双亲委派模型，就会出现很多的Object类，应用程序一片混乱。

一些自定义的加载器

* tomcat

tomcat通过war包进行应用的发布，它其实是违反了双亲委派机制原则的。下面是tomcat类加载器的层次结构。



对于一些需要加载的非基础类，会有一个叫做WebAppClassLoader的类加载器优先加载。等它加载不到的时候，再交给上层的ClassLoader进行加载。这个加载器用来隔绝不同应用的.class文件，比如你的两个应用，可能会依赖同一个第三方的不同版本，它们是相互没有影响的。
如何在同一个JVM里，运行着不兼容的两个版本，当然是需要自定义加载器才能完成的事。
那么tomcat是怎么打破双亲委派机制的呢？可以看上图中的WebAppClassLoader，它加载自己目录下的.class文件，并不会传递给父类的加载器。但是它却可以使用SharedClassLoader所加载的类，实现了共享和分离的功能。
但是你可以自己写一个ArrayList，放在应用目录里，tomcat依然不会加载。它只是自定义的加载器顺序不同，但对于顶层来说，还是一样的。

* SPI
  Java中有一个SPI机制，全称是Service Provider Interface，是Java提供的一套用来被第三方实现或者扩展的API，它可以用来启动框架和替换组件。

SPI 实际是“基于接口编程+策略模式+配置文件”组合实现的动态加载机制，主要使用java.util.ServiceLoader类进行动态装载。


这种方式，同样打破了双亲委派机制。
DriverManager类和ServiceLoader类都是属于rt.jar的。它们的类加载器都是BootstrapClassLoader，也就是最上层的那个。而具体的数据库驱动，却属于业务代码，这个启动类加载器是无法加载的。

    //part1:DriverManager::loadInitialDrivers
    //jdk1.8 之后，变成了lazy的ensureDriversInitialized

    ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
    Iterator<Driver> driversIterator = loadedDrivers.iterator();


    //part2:ServiceLoader::load
    public static <T> ServiceLoader<T> load(Class<T> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }

通过代码你可以发现 Java 玩了个魔术，它把当前的类加载器，设置成了线程的上下文类加载器。那么，对于一个刚刚启动的应用程序来说，它当前的加载器是谁呢？也就是说，启动 main 方法的那个加载器，到底是哪一个？

所以我们继续跟踪代码。找到 Launcher 类，就是 jre 中用于启动入口函数 main 的类。我们在 Launcher 中找到以下代码。

    public Launcher() {
        Launcher.ExtClassLoader var1;
        try {
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }

        try {
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }
        Thread.currentThread().setContextClassLoader(this.loader)
    }

到此为止，事情就比较明朗了，当前线程上下文的类加载器，是应用程序类加载器。使用它来加载第三方驱动，是没有什么问题的。

OSGi

后面补充



如何替换JDK的类

当 Java 的原生 API 不能满足需求时，比如我们要修改 HashMap 类，就必须要使用到 Java 的 endorsed 技术。我们需要将自己的 HashMap 类，打包成一个 jar 包，然后放到 -Djava.endorsed.dirs 指定的目录中。注意类名和包名，应该和 JDK 自带的是一样的。但是，java.lang 包下面的类除外，因为这些都是特殊保护的。

因为我们上面提到的双亲委派机制，是无法直接在应用中替换 JDK 的原生类的。但是，有时候又不得不进行一下增强、替换，比如你想要调试一段代码，或者比 Java 团队早发现了一个 Bug。所以，Java 提供了 endorsed 技术，用于替换这些类。这个目录下的 jar 包，会比 rt.jar 中的文件，优先级更高，可以被最先加载到。
























   