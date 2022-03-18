#### AST简介

AST（Abstract syntax tree）即为“抽象语法树”，是编辑器对代码的第一步加工之后的结果，是一个树形式表示的源代码。源代码的每个元素映射到一个节点或子树。
 Java的编译过程可以分成三个阶段：

![img](https:////upload-images.jianshu.io/upload_images/751860-9add9ded278480d3.png?imageMogr2/auto-orient/strip|imageView2/2/w/600/format/webp)

image

1. 所有源文件会被解析成语法树。
2. 调用注解处理器。如果注解处理器产生了新的源文件，新文件也要进行编译。
3. 最后，语法树会被分析并转化成类文件。

例如：下面一段java代的抽象语法树大概长这样：



![img](https:////upload-images.jianshu.io/upload_images/751860-a0364782af03db6b.jpg!large?imageMogr2/auto-orient/strip|imageView2/2/w/720/format/webp)

AST

编辑器对代码处理的流程大概是：

> JavaTXT->词语法分析-> 生成AST  ->语义分析 -> 编译字节码

![img](https:////upload-images.jianshu.io/upload_images/751860-dd6a26f36f06b626.png?imageMogr2/auto-orient/strip|imageView2/2/w/540/format/webp)

操作AST时机

通过操作AST，可以达到修改源代码的功能，相比[AOP三剑客](https://www.jianshu.com/p/dca3e2c8608a)，他的时机更为提前：

![img](https:////upload-images.jianshu.io/upload_images/751860-903e2766be21750d.png?imageMogr2/auto-orient/strip|imageView2/2/w/870/format/webp)

操作AST

##### 什么是 AST 转换？

AST 转换 是在编译过程中用来修改抽象语法树结构的代码的名称。修改 AST，通过在将其转换为字节码之前增加附加节点，是更好的生成代码的方法。

之前我们了解到APT的三个弱点：



```undefined
1、预留入口不编译会报红，正常运行就可以
2、反射获得新的类效率又太差
3、无法实现定点插桩，只能生成新的类
```

AST则很好的解决了上面的问题。

##### 如何操作AST？

1、直接使用Javac语法生成AST:



```kotlin
/* final int PRIME = 31; */ {
 if (!fields.isEmpty() || callSuper) {
   statements.append(maker.VarDef(maker.Modifiers(Flags.FINAL),
       primeName, maker.TypeIdent(Javac.getCTCint(TypeTags.class, "INT")), 
       maker.Literal(31)));
 }
}
```

在javac.tree的JCTree里面，几乎可以看到所有常用语法的关键字：
 比如JCImport，JCClassDecl、JCIf、JCBreak、JCReturn、JCThrow
 、JCDoWhileLoop、JCTry、JCCatch、JCAnnotation等，你可以直接用这些对象的操作组合成你想要的源码，类似于javapoet的组装模式。

2、借助工具库，更加简单的操作AST
 Rewrite、JavaParser等开源工具可以帮助你更简单的操作AST
 3、扩展Lombok自定义注解处理器(自行了解)

#### AOP之AST：

AOP定位插桩，相比重量级的AspectJ，ASM、Javassisit，修改AST可以做更加轻量级的代码插桩实现方案：



```cpp
void onClick(View v)
{ 
   //插入你想要的埋点代码; 
    doSomeThing();
}
```

AST可以实现任意代码的增删修改，相比其他AOP手段，效率更高(编辑器级别)。如果拿做饭为例子，AST就是你躺着你老婆给你做饭喂你吃,APT就是你老婆做饭，你打下手(类似留口子手动调用)；AspectJ就是叫外卖，用别人的厨具食材(编译器)做好了给你送货上门，但是不能保证饭菜质量；ASM或Javassisit就是打车去饭店排队点菜等上菜(类似Gradle插件在编译过程中的Task流程)；而运行期间的AOP可以利用反射，也就是你自己动手做黑暗料理了。

##### 举个例子：

正常运行期间，我们程序里面的断言是不会起作用的：
 assert str != null : "Must not be null";

如果我们想，在编译期间断言自动转化成if，就可以使用操作AST来实现，把assert手动改成if判断：

基本步骤：

1、定义AbstractProcessor，注明@SupportedAnnotationTypes("*")
 2、初始化：



```java
    private int tally;
    private Trees trees;
    private TreeMaker make;
    private Name.Table names;

    @Override
    public synchronized void init(ProcessingEnvironment env) {
        super.init(env);
        trees = Trees.instance(env);
        Context context = ((JavacProcessingEnvironment)
                env).getContext();
        make = TreeMaker.instance(context);
        names = Names.instance(context).table;//Name.Table.instance(context);
        tally = 0;
    }
```

注意魔法：我们把ProcessingEnvironment强转成JavacProcessingEnvironment，后面的操作都变成了IDE编辑器内部的操作了。

3、处理所有输入的AST：



```dart
 Set<? extends Element> elements = roundEnv.getRootElements();
            for (Element each : elements) {
                if (each.getKind() == ElementKind.CLASS) {
                    JCTree tree = (JCTree) trees.getTree(each);
                    TreeTranslator visitor = new Inliner();
                    tree.accept(visitor);
                }
            }
```

4、操作AST增加代码



```java
@Override
        public void visitAssert(JCTree.JCAssert tree) {
            super.visitAssert(tree);
            JCTree.JCStatement newNode = makeIfThrowException(tree);
            result = newNode;
            tally++;
        }

        private JCTree.JCStatement makeIfThrowException(JCTree.JCAssert node) {
            // make: if (!(condition) throw new AssertionError(detail);
            List<JCTree.JCExpression> args = node.getDetail() == null
                    ? List.<JCTree.JCExpression>nil()
                    : List.of(node.detail);
            JCTree.JCExpression expr = make.NewClass(
                    null,
                    null,
                    make.Ident(names.fromString("AssertionError")),
                    args,
                    null);
            return make.If(
                    make.Unary(JCTree.Tag.NOT, node.cond),
                    make.Throw(expo),
                    null);
        }
```

5、查看最终结果：

![img](https:////upload-images.jianshu.io/upload_images/751860-6e07ffec2a42b313.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

源代码

![img](https:////upload-images.jianshu.io/upload_images/751860-19f84aa6d866de94.png?imageMogr2/auto-orient/strip|imageView2/2/w/1016/format/webp)

AST修改之后的代码

再来个例子：我们还可以使用AST自动清除线上Log，防止裸奔：



```kotlin
    private class LogClear extends TreeTranslator {
        @Override
        public void visitBlock(JCTree.JCBlock jcBlock) {
            super.visitBlock(jcBlock);
            final List<JCTree.JCStatement> statements = jcBlock.getStatements();
            if (statements != null && statements.size() > 0) {
                List<JCTree.JCStatement> out = List.nil();
                for (JCTree.JCStatement statement : statements) {
                    if (statement.toString().contains("Log.")) {
                        mMessager.printMessage(Diagnostic.Kind.WARNING, this.getClass().getCanonicalName() + " 自动清除Log: LogClear:" + statement.toString());
                    } else {
                        out = out.append(statement);
                    }
                }
                jcBlock.stats = out;
            }
        }
    }
```

同时还可以避免log参数的计算以及方法调用的额外无用开销。

#### 扩展AST：

##### 1、样板代码less：著名的Lombok，注解@Data，自动生成setter、getter，toString、equals、hashCode等模版方法

Lombok除了可以修改AST，还可以联合编辑器做消除警告和代码提示。在保存代码的时候，悄无声息的生成了新的AST，并且在编辑器上给予你代码提示的功能。然而你看到的，仍然是最初的简洁的代码。

![img](https:////upload-images.jianshu.io/upload_images/751860-c9d17b53c5915291.png?imageMogr2/auto-orient/strip|imageView2/2/w/1100/format/webp)

Lombok

简直可以媲美kotlin的data：



```kotlin
data class Mountain(val name: String, val age: Int)
```

##### 2、自定义Lint，实现CodeReview自动化

Lint从第一个版本就选择了lombok-ast作为自己的AST Parser，并且用了很久。但是Java语言本身在不断更新，Android也在不断迭代出新，lombok-ast慢慢跟不上发展，所以Lint在25.2.0版增加了IntelliJ的PSI（Program Structure Interface）作为新的AST Parser。但是PSI于IntelliJ、于Lint也只是个过渡性方案，事实上IntelliJ早已开始了新一代AST Parser，UAST（Unified AST）的开发，而Lint也将于即将发布的25.4.0版中将PSI更新为UAST。

##### 3、语法糖优化，空安全

kotlin的空安全：



```css
bob?.department?.head?.name
```

AST可以更简洁的实现



```css
bob.department.head.name
```

原理就是自动帮你加了空判断

诸如此类，AST可以帮你实现更多类似于kotlin的语法糖，有了AST，你不必再羡慕kotlin。

#### AST操作推荐库：

[Rewrite](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FNetflix-Skunkworks%2Frewrite)
 [JavaParser](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FJavaparser%2FJavaparser)

#### 推荐阅读

[annotation processing介绍](https://link.jianshu.com?t=http%3A%2F%2Fopenjdk.java.net%2Fgroups%2Fcompiler%2Fdoc%2Fcompilation-overview%2Findex.html)
 [AST介绍](https://link.jianshu.com?t=http%3A%2F%2Fwww.eclipse.org%2Farticles%2FArticle-JavaCodeManipulation_AST%2F%23sec-example-application)
 [Lombok原理分析与功能实现](https://link.jianshu.com?t=https%3A%2F%2Fblog.mythsman.com%2F2017%2F12%2F19%2F1%2F)

利用 Project Lombok 自定义 AST 转换
 [https://www.ibm.com/developerworks/cn/java/j-lombok/?ca=drs-](https://link.jianshu.com?t=https%3A%2F%2Fwww.ibm.com%2Fdeveloperworks%2Fcn%2Fjava%2Fj-lombok%2F%3Fca%3Ddrs-)
 Lombok自定义annotation扩展含Intellij插件  [http://www.alliedjeep.com/128803.htm](https://link.jianshu.com?t=http%3A%2F%2Fwww.alliedjeep.com%2F128803.htm)
 lombok如何做的冗余代码消除。[https://blog.csdn.net/faicm/article/details/46772591](https://link.jianshu.com?t=https%3A%2F%2Fblog.csdn.net%2Ffaicm%2Farticle%2Fdetails%2F46772591)
 如何巧妙利用JSR269来重写AST: [https://my.oschina.net/superpdm/blog/129715](https://link.jianshu.com?t=https%3A%2F%2Fmy.oschina.net%2Fsuperpdm%2Fblog%2F129715)

### 老司机赶紧进群开车：   [555343041](https://link.jianshu.com?t=http%3A%2F%2Fshang.qq.com%2Fwpa%2Fqunwpa%3Fidkey%3D14f9009a0276624f6abf3221fe131c57ff05b70b5b4b922ed2c4aa4156155e73)

例子比较简单，直接上源码：



```kotlin
import com.google.auto.service.AutoService;
import com.sun.source.util.Trees;
import com.sun.tools.javac.processing.JavacProcessingEnvironment;
import com.sun.tools.javac.tree.JCTree;
import com.sun.tools.javac.tree.TreeMaker;
import com.sun.tools.javac.tree.TreeTranslator;
import com.sun.tools.javac.util.Context;
import com.sun.tools.javac.util.List;
import com.sun.tools.javac.util.Name;
import com.sun.tools.javac.util.Names;

import java.util.Set;

import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.ProcessingEnvironment;
import javax.annotation.processing.Processor;
import javax.annotation.processing.RoundEnvironment;
import javax.annotation.processing.SupportedAnnotationTypes;
import javax.annotation.processing.SupportedSourceVersion;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.Element;
import javax.lang.model.element.ElementKind;
import javax.lang.model.element.TypeElement;
import javax.tools.Diagnostic;


/**
* Created by baixiaokang on 18/4/10.
*/
@AutoService(Processor.class)//自动生成 javax.annotation.processing.IProcessor 文件
@SupportedSourceVersion(SourceVersion.RELEASE_8)//java版本支持
@SupportedAnnotationTypes("*")
public class ForceAssertions extends AbstractProcessor {

   private int tally;
   private Trees trees;
   private TreeMaker make;
   private Name.Table names;

   @Override
   public synchronized void init(ProcessingEnvironment env) {
       super.init(env);
       trees = Trees.instance(env);
       Context context = ((JavacProcessingEnvironment)
               env).getContext();
       make = TreeMaker.instance(context);
       names = Names.instance(context).table;//Name.Table.instance(context);
       tally = 0;
   }

   @Override
   public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnv) {
       if (!roundEnv.processingOver()) {
           Set<? extends Element> elements = roundEnv.getRootElements();
           for (Element each : elements) {
               if (each.getKind() == ElementKind.CLASS) {
                   JCTree tree = (JCTree) trees.getTree(each);
                   TreeTranslator visitor = new Inliner();
                   tree.accept(visitor);
               }
           }
       } else
           processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE,
                   tally + " assertions inlined.");
       return false;
   }


   private class Inliner extends TreeTranslator {

       @Override
       public void visitAssert(JCTree.JCAssert tree) {
           super.visitAssert(tree);
           JCTree.JCStatement newNode = makeIfThrowException(tree);
           result = newNode;
           tally++;
       }

       private JCTree.JCStatement makeIfThrowException(JCTree.JCAssert node) {
           // make: if (!(condition) throw new AssertionError(detail);
           List<JCTree.JCExpression> args = node.getDetail() == null
                   ? List.<JCTree.JCExpression>nil()
                   : List.of(node.detail);
           JCTree.JCExpression expr = make.NewClass(
                   null,
                   null,
                   make.Ident(names.fromString("AssertionError")),
                   args,
                   null);
           return make.If(
                   make.Unary(JCTree.Tag.NOT, node.cond),
                   make.Throw(expo),
                   null);
       }

   }
}
```



作者：North_2016
链接：https://www.jianshu.com/p/5514cf705666
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
