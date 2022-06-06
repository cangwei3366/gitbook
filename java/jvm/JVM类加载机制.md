# JVM类加载机制

### 1. Java代码执行流程

类的加载指的是将类的.class文件中的二进制数据读入到内存中，将其放在运行时数据区的方法区内，然后在堆区创建一个 java.lang.Class对象，用来封装类在方法区内的数据结构。类的加载的最终产品是位于堆区中的 Class对象， Class对象封装了类在方法区内的数据结构，并且向Java程序员提供了访问方法区内的数据结构的接口。

**类加载器流程**

| **（类加载器子系统）** |                                      |          |            |
| ---------------------- | ------------------------------------ | -------- | ---------- |
| 字节码文件             | 加载阶段                             | 链接阶段 | 初始化阶段 |
|                        | 应用类加载器 Application ClassLoader | 验证     | 初始化     |
|                        | 扩展类加载器 Extension ClassLoader   | 准备     |            |
|                        | 系统类加载器 Bootstrap ClassLoader   | 解析     |            |

**加载阶段**：

* 通过双亲委派机制，通过一个类的全限定名获取定义此类的二进制字节流
* 转为方法区（jdk1.7以前永久代，1.8版本元空间）运行时数据结构
* 在Java堆中生成一个代表这个类的 java.lang.Class对象，作为对方法区中这些数据的访问入口

**链接阶段**：

* 验证(Verify):目的在于确保class文件的字节流信息符合当前虚拟机要求，

  主要包括四种验证，文件格式验证，元数据验证，字节码验证，符号引用验证

* 准备(Perpare)：为类变量分配内存并设置初始值

  不包含final修饰的static，因为final在编译时候就会分配

  这里不会为实例变量分配初始化，会随着对象一起分配到堆中

* 解析(Resolve)

  符号引用转换为直接引用，一般在初始化之后开始解析

**初始化阶段**： 收集类中的所有类变量的赋值动作和静态代码块中的语句进行初始化

类初始化的场景：

* 创建类的实例，也就是new的方式
* 访问某个类或接口的静态变量，或者对该静态变量赋值
* 调用类的静态方法
* 反射（如 Class.forName(“com.shengsiyuan.Test”)）
* 初始化某个类的子类，则其父类也会被初始化
* Java虚拟机启动时被标明为启动类的类（ JavaTest），直接使用 java.exe命令来运行某个主类

>**启动类加载器：** Bootstrap ClassLoader，负责加载存放在JDK\jre\lib(JDK代表JDK的安装目录，下同)下，或被-Xbootclasspath参数指定的路径中的，并且能被虚拟机识别的类库（如rt.jar，所有的java.*开头的类均被Bootstrap ClassLoader加载）。启动类加载器是无法被Java程序直接引用的。
>
>**扩展类加载器：** Extension ClassLoader，该加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载DK\jre\lib\ext目录中，或者由java.ext.dirs系统变量指定的路径中的所有类库（如javax.*开头的类），开发者可以直接使用扩展类加载器。
>
>**应用程序类加载器：** Application ClassLoader，该类加载器由sun.misc.Launcher$AppClassLoader来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

## 2.双亲委派模型

双亲委派机制

1. 当AppClassLoader加载一个class时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器ExtClassLoader去完成。
2. 当ExtClassLoader加载一个class时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给BootStrapClassLoader去完成。

3. 如果BootStrapClassLoader加载失败（例如在$JAVA_HOME/jre/lib里未查找到该class），会使用ExtClassLoader来尝试加载；

4. 若ExtClassLoader也加载失败，则会使用AppClassLoader来加载，如果AppClassLoader也加载失败，则会报出异常ClassNotFoundException。

>双亲委派模型意义：
>
>系统类防止内存中出现多份同样的字节码
>
>保证Java程序安全稳定运行

### 3. 破坏双亲委派机制

**第一次破坏**：jdk1.2的java.lang.ClassLoader添加findClass()

用户在自定义类加载逻辑时尽可能重写这个方法。父类加载器加载失败，会自动调用自己的findClass完成加载。

**第二次破坏**：解决各个类加载器协作时基础类型的一致性问题。引入线程上下文类加载器，jdk6提供了java.util.ServiceLoader 以责任链模式加载。

**第三次破坏**：用户追求程序动态性：代码热替换、模块热部署等。osgi框架（一个Bundle就有一个类加载器）

1. java.\*开头的类，委派给父类加载器加载
2. 将委托名单的类，委派给父类加载器加载
3. 将import列表的类，委派给Export这个类的bundle的类加载器加载
4. 否则使用当前bundle的类加载器加载
5. 查找是否类在fragment bundle中
6. 查找是否类Dynamic import列表对应的bundle

例如：jdbc

java中driver类位于启动类加载器路径下lib目录，具体实现是通过各个厂商的驱动jar包

通过设置上下文类加载器，就可以通过系统加载器进行加载

## 4.自定义类加载器

```java
//创建自定义加载器类继承ClassLoader类
public class MyClassLoader extends ClassLoader{
//    包路径
    private String Path;
 
//    构造方法，用于初始化Path属性
    public MyClassLoader(String path) {
        this.Path = path;
    }
 
//    重写findClass方法，参数name表示要加载类的全类名(包名.类名)
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        System.out.println("findclass方法执行");
 
//        检查该类的class文件是否已被加载，如果已加载则返回class文件(字节码文件)对象，如果没有加载返回null
        Class<?> loadedClass = findLoadedClass(name);
//        如果已加载直接返回该类的class文件(字节码文件)对象
        if (loadedClass != null){
            return loadedClass;
        }
 
//        字节数组，用于存储class文件的字节流
        byte[] bytes = null;
        try {
//            获取class文件的字节流
            bytes = getBytes(name);
        } catch (Exception e) {
            e.printStackTrace();
        }
 
 
        if (bytes != null){
//        如果字节数组不为空，则将class文件加载到JVM中
            System.out.println(bytes.length);
//            将class文件加载到JVM中，返回class文件对象
            Class<?> aClass = this.defineClass(name, bytes, 0, bytes.length);
            return aClass;
        }else {
            throw new ClassNotFoundException();
        }
    }
 
//    获取class文件的字节流
    private byte[] getBytes(String name) throws Exception{
//        拼接class文件路径 replace(".",File.separator) 表示将全类名中的"."替换为当前系统的分隔符，File.separator返回当前系统的分隔符
        String FileUrl = Path + name.replace(".", File.separator) + ".class";
        byte[] bytes;
//        相当于一个缓存区，动态扩容，也就是随着写入字节的增加自动扩容
        ByteArrayOutputStream arrayOutputStream = new ByteArrayOutputStream();
        File file = new File(FileUrl);
//        创建输入流
        InputStream inputStream = new FileInputStream(file);
        int content;
//        循环将输入流中的所有数据写入到缓存区中
        while ((content = inputStream.read()) != -1){
            arrayOutputStream.write(content);
            arrayOutputStream.flush();
        }
        bytes = arrayOutputStream.toByteArray();
        return bytes;
    }
    
       public static void main(String[] args)  {

        MyClassLoader classLoader = new MyClassLoader();
        classLoader.setRoot("E:\\temp");

        Class<?> testClass = null;
        try {
            testClass = classLoader.loadClass("com.classloader.MyClassLoader");
            Object object = testClass.newInstance();
            System.out.println(object.getClass().getClassLoader());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

**为什么要自定义类加载器**

隔离加载类（使得各个模块路径类名一样的文件互不影响）

修改类加载方式

扩展加载源

防止源码泄漏
