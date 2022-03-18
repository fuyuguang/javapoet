AOP:面向切面编程(Aspect-Oriented Programming)。如果说，OOP如果是把问题划分到单个模块的话，那么AOP就是把涉及到众多模块的某一类问题进行统一管理。

Android AOP就是通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，提高开发效率。本文仅做知识介绍，相关详细内容不做过多描述，全部代码在项目[T-MVP](https://link.jianshu.com?t=https://github.com/north2016/T-MVP)。

话不多说，先上图：

![img](https:////upload-images.jianshu.io/upload_images/751860-0641778f0bc265ad.png?imageMogr2/auto-orient/strip|imageView2/2/w/540/format/webp)

APT,AspectJ,Javassist对应的编译时期

AOP在Java后台，已经被各路大神研发出各种框架风生水起，例如SSH、SpringMVC等等殿堂级框架。在Android端，近年来也是异军突起。

# APT

代表框架：DataBinding,Dagger2, ButterKnife, EventBus3 、DBFlow、AndroidAnnotation

注解处理器 Java5 中叫APT(Annotation Processing Tool)，在Java6开始，规范化为 Pluggable Annotation Processing。Apt应该是这其中我们最常见到的了，难度也最低。定义编译期的注解，再通过继承Proccesor实现代码生成逻辑，实现了编译期生成代码的逻辑。

使用姿势  ：
 1、建立一个java的Module,写一个继承AbstractProcessor的类

![img](https:////upload-images.jianshu.io/upload_images/751860-a8354794eecb4487.png?imageMogr2/auto-orient/strip|imageView2/2/w/669/format/webp)

AbstractProcessor

2、在工具类里处理我们自定义的注解、生成代码：

![img](https:////upload-images.jianshu.io/upload_images/751860-96b09bef51da7f14.png?imageMogr2/auto-orient/strip|imageView2/2/w/1069/format/webp)

Processor

3、在Gradle中添加 dependencies   annotationProcessor project(':apt')
 低版本需要使用第三方插件  apply plugin: 'com.neenbedankt.android-apt'
 然后apt  project(':apt')

生成的源代码在build/generated/source/apt下可以看到

![img](https:////upload-images.jianshu.io/upload_images/751860-efc9f4bdf0aa21bd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1056/format/webp)

apt生成代码的路径


 难点：
 就apt本身来说没有任何难点可言，难点一在于设计模式和解耦思想的灵活应用，二在与代码生成的繁琐，你可以手动字符串拼接，当然有更高级的玩法用squareup的javapoet，用建造者的模式构建出任何你想要的源代码。
 想详细了解可以看官网或这篇博客：
[Android 利用 APT 技术在编译期生成代码](https://link.jianshu.com?t=http://brucezz.itscoder.com/articles/2016/08/06/use-apt-in-android/)



优点：
 它的强大之处无需多言，看代表框架的源码，你可以学到很多新姿势。总的一句话：它可以做任何你不想做的繁杂的工作，它可以帮你写任何你不想重复代码。懒人福利，老司机必备神技，可以提高车速，让你以任何姿势漂移。它可以生成任何源代码供你在任何地方使用，就像剑客的剑，快疾如风，无所不及。

# AspectJ

代表框架： Hugo(Jake Wharton)

AspectJ支持编译期和加载时代码注入，在开始之前，我们先看看需要了解的词汇：
 **Advice（通知）:** 典型的 Advice 类型有 before、after 和 around，分别表示在目标方法执行之前、执行后和完全替代目标方法执行的代码。

**Joint point（连接点）:** 程序中可能作为代码注入目标的特定的点和入口。

**Pointcut（切入点）:** 告诉代码注入工具，在何处注入一段特定代码的表达式。

**Aspect（切面）:** Pointcut 和 Advice 的组合看做切面。例如，在本例中通过定义一个 pointcut 和给定恰当的advice，添加一个了内存缓存的切面。

**Weaving（织入）:** 注入代码（advices）到目标位置（joint points）的过程。

下面这张图简要总结了一下上述这些概念。

![img](https:////upload-images.jianshu.io/upload_images/30689-55846998f4f5b4ce.png?imageMogr2/auto-orient/strip|imageView2/2/w/674/format/webp)

AOP概念图

使用姿势：
 1、建立一个android lib Module，定义一个切片，处理自定义注解，和添加切片逻辑

![img](https:////upload-images.jianshu.io/upload_images/751860-949bca6f4d271416.png?imageMogr2/auto-orient/strip|imageView2/2/w/740/format/webp)

AspectJ

2、自定义一个gradle插件，使用 AspectJ 的编译器（ajc，一个java编译器的扩展),对所有受 aspect 影响的类进行织入，在 gradle 的编译 task 中增加额外配置，使之能正确编译运行。

![img](https:////upload-images.jianshu.io/upload_images/751860-72735c6584271c15.png?imageMogr2/auto-orient/strip|imageView2/2/w/747/format/webp)

AspectjPlugin

3、在grade中apply plugin:com.app.plugin.AspectjPlugin

生成的class文件在build/intermediates/classes下可以看到

![img](https:////upload-images.jianshu.io/upload_images/751860-61108b9120452cb0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Aspectj编织后的文件路径

难点：
 AspectJ语法比较多，但是掌握几个简单常用的，就能实现绝大多数切片，完全兼容Java（纯Java语言开发，然后使用AspectJ注解，简称@AspectJ。）想详细了解可以看官网或这篇博客：
 [深入理解AndroidAOP](https://link.jianshu.com?t=http://blog.csdn.net/innost/article/details/49387395)

优点：
 AspectJ除了hook之外，AspectJ还可以为目标类添加变量,接口。另外，AspectJ也有抽象，继承等各种更高级的玩法。它能够在编译期间直接修改源代码生成class，强大的团战切入功能，指哪打哪，鞭辟入里。有了此神器，编程亦如庖丁解牛，游刃而有余。

# Javassist

代表框架：热修复框架HotFix 、Savior（InstantRun）等

Javassist作用是在编译器间修改class文件，与之相似的ASM（热修复框架女娲）也有这个功能，可以让我们直接修改编译后的class二进制代码，首先我们得知道什么时候编译完成，并且我们要赶在class文件被转化为dex文件之前去修改。在Transfrom这个api出来之前，想要在项目被打包成dex之前对class进行操作，必须自定义一个Task，然后插入到predex或者dex之前，在自定义的Task中可以使用javassist或者asm对class进行操作。而Transform则更为方便，Transfrom会有他自己的执行时机，不需要我们插入到某个Task前面。Tranfrom一经注册便会自动添加到Task执行序列中，并且正好是项目被打包成dex之前。

使用姿势
 1、定义一个buildSrc module添加自定义Plugin



![img](https:////upload-images.jianshu.io/upload_images/751860-7eaf4b6e094cad12.png?imageMogr2/auto-orient/strip|imageView2/2/w/482/format/webp)

自定义Plugin

2、自定义Transform

![img](https:////upload-images.jianshu.io/upload_images/751860-afe80d7a631197d4.png?imageMogr2/auto-orient/strip|imageView2/2/w/799/format/webp)

自定义Transform

3、在Transform里处理Task，通过inputs拿到一些东西，处理完毕之后就输出outputs，而下一个Task的inputs则是上一个Task的outputs。

![img](https:////upload-images.jianshu.io/upload_images/751860-51e35b3289e4c526.png?imageMogr2/auto-orient/strip|imageView2/2/w/798/format/webp)

处理Task

4、使用Javassist操作字节码，添加新的逻辑或者修改原有逻辑



![img](https:////upload-images.jianshu.io/upload_images/751860-c3493f3e1db72951.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Javassist操作字节码

5、在grade中apply plugin:com.app.plugin.MyPlugin

修改后的class文件在build/intermediates/transforms/MyTrans下可以看到

![img](https:////upload-images.jianshu.io/upload_images/751860-788b456e6747f8d3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Javassist修改后的文件路径

难点：
 相比ASM，Javassist对java极度友好的api更容易快速上手，难点在思想的应用，小到切片逻辑的控制，如本例中的性能log打印日志，大到宏观的热修复，插件化中对preDex的操作修改，剑客精神到了这一层级，已经是上帝视角，无所不能。

优点：
 由于Javassist可以直接操作修改编译后的字节码，直接绕过了java编译器，所以可以做很多突破限制的事情，例如，跨dex引用，解决热修复中CLASS_ISPREVERIFIED的问题。

想详细了解可以看官网或这篇博客：
 [Android热补丁动态修复技术](https://link.jianshu.com?t=http://blog.csdn.net/u010386612/article/details/51131642)

[基于Instant Run思想的HotFix方案实现](https://link.jianshu.com?t=https://halfstackdeveloper.github.io/2016/09/23/基于Instant-Run思想的HotFix方案实现/)

# AOP

AOP技术常用在以下方面：
 1、日志记录：业务埋点
 2、持久化
 3、性能监控：性能日志
 4、数据校验：方法的参数校验
 5、缓存：内存缓存和持久缓存
 6、权限检查：业务权限（如登陆，或用户等级）、系统权限（如拍照定位）
 7、异常处理

利用AOP技术将这些功能代码从业务逻辑代码中划分出来，通过对这些行为的分离，可以将它们独立到非业务逻辑。无论是日后新增，或是修改，都手到擒来易如反掌。

例如新的tmvp的demo中 apt用于生成实例化工厂，替换掉(对于小项目来说)繁杂冗余的Dagger2,实现了初始化功能的aop ；aspectj 的切片主要用在缓存和日志，用注解实现方法级别的内存缓存和方法耗时日志；Javassist 这里只是做了个示例，也是通过注解实现方法耗时日志的自动打印功能，当然这些都只是AOP的九牛一毛，AOP还可以做很多事，弥补OOP的不足，把所有跨对象的横切面关注点的功能都可以提取出来用AOP去实现 ，好处显而易见，将来要改的地方永远只有一处，而不是像OOP那样牵扯很多模块很多代码很多类。

当然还有更多未知的可能，需要等各位大侠来研究开发，让Aop在Android上的应用更加广泛。

PS：刚发现一个歪果仁的框架 [http://6thsolution.github.io/EasyMVP](https://link.jianshu.com?t=http://6thsolution.github.io/EasyMVP)  基于Clean Architecture 用了apt、aspectj、javassisit 不多说赶紧看源代码学习去了

QQ群：AndroidMVP   [555343041](https://link.jianshu.com?t=http://shang.qq.com/wpa/qunwpa?idkey=14f9009a0276624f6abf3221fe131c57ff05b70b5b4b922ed2c4aa4156155e73)

## 更新日志：

2017/1／31：AOP新增SysPermissionAspect支持动态申请系统权限切片，轻松适配6.0+

2017/1／27：AOP新增DbRealmAspect支持Realm数据库，数据库突破你想像的简单(年夜特供)

2017/1／8： 使用Apt封装Retrofit生成ApiFactory替换掉所有的Repository，狂删代码

2017/1／7： 使用DataBinding替换掉所有的ButterKnife，狂删代码

2017/1／6： 使用DataBinding替换掉所有的ViewHolder，狂删代码，从此迈向新时代

2016/12／30：使用Apt生成全局路由TRouter，更优雅的页面跳转，支持传递参数和共享view转场动画

2016/12／29：去掉BaseMultiVH新增VHClassSelector支持更完美的多ViewHolder

2016/12／28：使用Apt生成全局的ApiFactory替代所有的Model

2016/12／27：增加了BaseMultiVH扩展支持多类型的ViewHolder

2016/12／26：抽离CoreAdapterPresenter优化TRecyclerView

## [安卓AOP实战：面向切片编程](https://www.jianshu.com/p/b96a68ba50db)

## [Android实用技巧之:用好泛型,少写代码](https://www.jianshu.com/p/0f6800ded3da)

## [安卓AOP实战:APT打造极简路由](https://www.jianshu.com/p/6ccfa7b50f0e)

> 全局路由TRouter，更优雅的页面跳转

## [安卓AOP实战:Javassist强撸EventBus](https://www.jianshu.com/p/33d8a3165b07)

> 加入OkBus，实现注解传递事件

## [安卓AOP三剑客:APT,AspectJ,Javassist](https://www.jianshu.com/p/dca3e2c8608a)

> 1、去掉所有反射>2、新增apt初始化工厂，替换掉了dagger2。>3、新增aop切片，处理缓存和日志



作者：North_2016
链接：https://www.jianshu.com/p/dca3e2c8608a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。