### Javac编译概述

将.java源文件编译成.class文件，这一步大致可以分为3个过程：
 1、把所有的源文件解析成语法树，输入到编译器的符号表；
 2、注解处理器的注解处理过程；
 3、分析语法树并生成字节码。



![img](https:////upload-images.jianshu.io/upload_images/11238893-a40f2a4cc51db870.png?imageMogr2/auto-orient/strip|imageView2/2/w/600/format/webp)

javac编译过程.png

#### Parse and Enter

**1.词法分析: 通过Scanner将源码的字符流解析成Token流**
 通过词法分析器分析源文件中的所有字符，将所有的单词或字符都转化成符合规范的Token，规范化的token可以分成一下三种类型：

- java关键字：public, static, final, String, int等等；
- 自定义的名称：包名，类名，方法名和变量名；
- 运算符或者逻辑运算符等符号：+、-、*、/、&&，|| 等等。



```cpp
int x=y+1;
```

*这一行代码解析成token流如下：*

![img](https:////upload-images.jianshu.io/upload_images/11238893-99427cef84a6881d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

token流.png



**2.语法分析: 根据token流，利用TreeMaker，以JCTree的子类作为语法节点来构建抽象语法树**
 抽象语法树（Abstract Syntax Tree）是一种用来描述程序代码语法结构的树形表示方式，语法树的每一个节点都代表着程序代码中的一个语法结构, 如包、类型、修饰符、运算符、接口、返回值都可以是一个语法结构。



```java
package com.example.adams.astdemo;
public class TestClass {
    int x = 0;
    int y = 1;
    public int testMethod(){
        int z = x + y;
        return z;
    }
}
```

对应的抽象语法树如下：



![img](https:////upload-images.jianshu.io/upload_images/11238893-fdca37e67c4c028d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

抽象语法树.png

**3.将java类中的符号输入到符号表中**
 符号表是由一组符号地址和符号信息构成的表格；符号表中所登记的信息在编译的不同阶段都要用到，在语法分析中, 符号表所登记的内容将用于语义检查和产生中间代码。在目标代码生成阶段, 符号表是当对符号名进行地址分配时的依据。

- 将所有类中出现的符号输入到类自身的符号表中，所有类符号、类的参数类型符号（泛型参数类型）、超类符号和继承的接口类型符号等都存储到一个未处理的列表（To Do List）中；
- 将这个未处理的列表中所有的类都解析到各自的类符号列表中，这个操作是在MemberEnter.complete()中完成(默认构造器也是在这里完成的)。

#### Annotation Processing

在JDK 1.5之后，Java语言提供了对注解（Annotation）的支持，注解与普通的Java关键字一样，而在JDK 1.6中实现了JSR-269规范JSR-269：Pluggable Annotations Processing API（插入式注解处理API）。提供了一组插入式注解处理器的标准API在编译期间对注解进行处理；**在注解处理期间，我们可以获取到所有的抽象语法树，可以对抽象语法树进行增删改查；**语法树被修改过之后，编译器将回到解析及填充符号表的过程重新处理，直到所有插入式注解处理器都没有再对语法树进行修改为止。

#### Parse and Enter

**1.语义分析**
 语义分析的主要任务是对结构正确的源程序进行上下文有关性质的审查，过程分为标注检查和数据及控制流分析两个步骤：

- 标注检查
   检查语义合法性、进行逻辑判断，如变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配等；
- 数据及控制流分析
   在Javac的源码中，数据及控制流分析的入口是图中的flow()，由com.sun.tools.javac.comp.Flow类来完成，作用是对程序上下文逻辑更进一步的验证，检查局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理了等问题。

**2.解语法糖**
 语法糖（Syntactic Sugar），也称糖衣语法，指在计算机语言中添加的某种语法，Java中最常用的语法糖主要是的泛型擦除、变长参数、自动装箱/拆箱、条件编译等，解语法糖就是还原回简单的基础语法结构。
 **3.生成字节码**
 字节码生成是Javac编译过程的最后一个阶段，由com.sun.tools.javac.jvm.Gen类来完成；把前面各个步骤所生成的信息（语法树、符号表）转化成字节码，再将字节码输出到*.class文件中。

##### 总结

> 通过javac的编译原理可以得出：
>  1.抽象语法树是一种描述程序代码语法结构的树形表示方式；
>  2.对java源文件经过词语法分析，构建出抽象语法树；
>  3.我们可以在注解处理器中获取到抽象语法树。





### JCTree类（com.sun.tools.javac.tree.JCTree）的简要分析

*[上一篇文章](https://www.jianshu.com/p/ff8ec920f5b9)
 讲解了抽象语法树的来源和获取时机，接下来要分析一下抽象语法树的内部结构。*

抽象语法树由JCTree的内部类（如JCCompilationUnit，JCClassDecl，JCMethodDecl等）作为语法节点构成。我们可以通过调用JCTree的accept()方法来访问抽象语法树的所有语法节点。



```csharp
public abstract void accept(JCTree.Visitor var1)；
```

accept()方法接收一个JCTree.Visitor的参数，通过执行这个方法，我们可以在Visitor类中获取到抽象语法树所有语法节点的数据。

##### Visitor

- 抽象内部类，内部定义了访问各种语法节点的方法，获取到对应的语法节点后我们可以对语法节点增加删除或者修改语句；
- Visitor派生子类有TreeScanner（扫描所有的语法节点）和TreeTranslator（扫描节点且可以把语法节点转换成另一种语法节点）；



```csharp
public abstract static class Visitor {

        //访问类节点
        public void visitClassDef(JCTree.JCClassDecl var1) {
            this.visitTree(var1);
        }

        //访问方法节点
        public void visitMethodDef(JCTree.JCMethodDecl var1) {
            this.visitTree(var1);
        }

        //访问变量节点
        public void visitVarDef(JCTree.JCVariableDecl var1) {
            this.visitTree(var1);
        }

        //访问skip节点
        public void visitSkip(JCTree.JCSkip var1) {
            this.visitTree(var1);
        }

        //访问代码块节点
        public void visitBlock(JCTree.JCBlock var1) {
            this.visitTree(var1);
        }

        //访问doWhild循环
        public void visitDoLoop(JCTree.JCDoWhileLoop var1) {
            this.visitTree(var1);
        }

        //访问whild循环
        public void visitWhileLoop(JCTree.JCWhileLoop var1) {
            this.visitTree(var1);
        }

        //访问for循环
        public void visitForLoop(JCTree.JCForLoop var1) {
            this.visitTree(var1);
        }

        //访问forEach循环
        public void visitForeachLoop(JCTree.JCEnhancedForLoop var1) {
            this.visitTree(var1);
        }

        //访问switch
        public void visitSwitch(JCTree.JCSwitch var1) {
            this.visitTree(var1);
        }

        //访问case
        public void visitCase(JCTree.JCCase var1) {
            this.visitTree(var1);
        }

        //访问同步关键字
        public void visitSynchronized(JCTree.JCSynchronized var1) {
            this.visitTree(var1);
        }

        //访问try
        public void visitTry(JCTree.JCTry var1) {
            this.visitTree(var1);
        }

        //访问catch
        public void visitCatch(JCTree.JCCatch var1) {
            this.visitTree(var1);
        }

        //访问三目运算符表达式
        public void visitConditional(JCTree.JCConditional var1) {
            this.visitTree(var1);
        }

        //访问if语句
        public void visitIf(JCTree.JCIf var1) {
            this.visitTree(var1);
        }

        //访问break语句
        public void visitBreak(JCTree.JCBreak var1) {
            this.visitTree(var1);
        }

        //访问continue语句
        public void visitContinue(JCTree.JCContinue var1) {
            this.visitTree(var1);
        }

        //访问return语句
        public void visitReturn(JCTree.JCReturn var1) {
            this.visitTree(var1);
        }

        //访问异常抛出表达式节点
        public void visitThrow(JCTree.JCThrow var1) {
            this.visitTree(var1);
        }

        //访问方法调用节点
        public void visitApply(JCTree.JCMethodInvocation var1) {
            this.visitTree(var1);
        }

        //访问new对象语句节点
        public void visitNewClass(JCTree.JCNewClass var1) {
            this.visitTree(var1);
        }

        //访问new数组语句节点
        public void visitNewArray(JCTree.JCNewArray var1) {
            this.visitTree(var1);
        }

        //访问赋值语句节点
        public void visitAssign(JCTree.JCAssign var1) {
            this.visitTree(var1);
        }

        //访问赋值操作语句节点
        public void visitAssignop(JCTree.JCAssignOp var1) {
            this.visitTree(var1);
        }

        //访问赋值操作语句节点
        public void visitUnary(JCTree.JCUnary var1) {
            this.visitTree(var1);
        }

        //访问一元运算节点
        public void visitBinary(JCTree.JCBinary var1) {
            this.visitTree(var1);
        }

        //访问对其他类的方法调用或者变量调用节点
        public void visitSelect(JCTree.JCFieldAccess var1) {
            this.visitTree(var1);
        }

        //访问标识符
        public void visitIdent(JCTree.JCIdent var1) {
            this.visitTree(var1);
        }

        //访问字面量，可以是数字1，字符串"x"
        public void visitLiteral(JCTree.JCLiteral var1) {
            this.visitTree(var1);
        }

        //获取访问修饰符
        public void visitModifiers(JCTree.JCModifiers var1) {
            this.visitTree(var1);
        }
    }
```

##### JCStatement

声明语句，继承JCStatement都是声明语句，子类有：
 JCBlock、JCReturn、JCClassDecl、JCVariableDecl、JCTry、JCThrow等；

##### JCExpression

表达式语句，继承JCExpression都是表达式语句，子类有：
 JCAssign，JCBinary，JCBreak，JCFieldAccess，JCIdent，JCLiteral等；

##### JCClassDecl

类定义



```php
public static class JCClassDecl extends JCTree.JCStatement implements ClassTree {
        public JCTree.JCModifiers mods;//访问修饰符 比如public, final
        public Name name;//类名
        public List<JCTree.JCTypeParameter> typarams;//泛型参数列表
        public JCTree.JCExpression extending;//父类
        public List<JCTree.JCExpression> implementing;//接口列表
        public List<JCTree> defs;//变量，方法定义列表
        public ClassSymbol sym;//包名+类名
}
```

##### JCMethodDecl

方法定义



```php
public static class JCMethodDecl extends JCTree implements MethodTree {
        public JCTree.JCModifiers mods;//访问修饰符 比如public, final，static
        public Name name;//方法名
        public JCTree.JCExpression restype;//返回类型
        public List<JCTree.JCTypeParameter> typarams;//泛型参数列表
        public JCTree.JCVariableDecl recvparam;//null
        public List<JCTree.JCVariableDecl> params;//方法参数列表
        public List<JCTree.JCExpression> thrown;//异常抛出列表
        public JCTree.JCBlock body;//方法体
        public JCTree.JCExpression defaultValue;//注解类的方法需要的defaultValue
        public MethodSymbol sym;//方法名+ （参数类型），如：setName(java.lang.String)
}
```

##### JCVariableDecl

变量定义



```php
public static class JCVariableDecl extends JCTree.JCStatement implements VariableTree {
        public JCTree.JCModifiers mods;//访问修饰符 比如public, final，static
        public Name name;//变量名
        public JCTree.JCExpression nameexpr;//null
        public JCTree.JCExpression vartype;//变量的类型
        public JCTree.JCExpression init;//初始化值
        public VarSymbol sym;//变量名
}

-->创建变量： private x = 1;
treeMaker.VarDef(treeMaker.Modifiers(Flags.PRIVATE), names.fromString"x", treeMaker.TypeIdent(TypeTag.INT),  treeMaker.Literal(1));
```

##### JCModifiers

访问修饰符



```java
public static class JCModifiers extends JCTree implements ModifiersTree {
        public long flags;//访问标志，例如：public，private，static
        public List<JCTree.JCAnnotation> annotations;//注解列表
}

-->flag通常用Flags（com.sun.tools.javac.code.Flags）类的常量来表示
public class Flags {
    public static final int PUBLIC = 1;
    public static final int PRIVATE = 2;
    public static final int PROTECTED = 4;
    public static final int STATIC = 8;
    public static final int FINAL = 16;
    public static final int SYNCHRONIZED = 32;
    public static final int VOLATILE = 64;
    public static final int TRANSIENT = 128;
    public static final int NATIVE = 256;

    ...
}

-->创建例子：public
treeMaker.Modifiers(Flags.PUBLIC);
```

##### JCBlock

代码块



```php
public static class JCBlock extends JCTree.JCStatement implements BlockTree {
        public long flags;//访问修复符
        public List<JCTree.JCStatement> stats;//多行代码列表
}

-->使用例子：
List<JCTree.JCStatement> jcStatementList = List.nil();
treeMaker.Block(0, jcStatementList);//构建代码块
```

##### JCIdent

标识符表达式，可以表示类、变量引用或者方法。



```java
public static class JCIdent extends JCTree.JCExpression implements IdentifierTree {
        public Name name;//标识符的名字
        public Symbol sym;//代表类时为包名+类名，代表其他类型数据时为null

        protected JCIdent(Name var1, Symbol var2) {
            this.name = var1;
            this.sym = var2;
        }
}

-->创建实例: 获取变量textView的引用
treeMaker.Ident(names.fromString("textView"))))
```

##### JCFieldAccess

其他类的变量、方法的访问表达式



```java
public static class JCFieldAccess extends JCTree.JCExpression implements MemberSelectTree {
        public JCTree.JCExpression selected;//类访问表达式
        public Name name;//变量名或方法名
        public Symbol sym;
}
```

##### JCLiteral

字面量表达式



```java
public static class JCLiteral extends JCTree.JCExpression implements LiteralTree {
        public TypeTag typetag;//常量的类型
        public Object value;//常量的值
}

-->
字面量类型的枚举类
public enum TypeTag {
    BYTE(1, 125, true),
    CHAR(2, 122, true),
    SHORT(4, 124, true),
    LONG(16, 112, true),
    FLOAT(32, 96, true),
    INT(8, 120, true),
    DOUBLE(64, 64, true),
    BOOLEAN(0, 0, true),
    CLASS,

    ...
}

-->使用例子：闭括号字符串"}"
treeMaker.Literal("}")
```

##### JCBinary

二元操作符



```java
    public static class JCBinary extends JCTree.JCExpression implements BinaryTree {
        private JCTree.Tag opcode;//操作符
        public JCTree.JCExpression lhs;//操作符左边
        public JCTree.JCExpression rhs;//操作符右边
        public Symbol operator;//null
}

-->二元运算符opcode可取值
public static enum Tag {
        OR,                              // ||
        AND,                             // &&
        BITOR,                           // |
        BITXOR,                          // ^
        BITAND,                          // &
        EQ,                              // ==
        NE,                              // !=
        LT,                              // <
        GT,                              // >
        LE,                              // <=
        GE,                              // >=
        SL,                              // <<
        SR,                              // >>
        USR,                             // >>>
        PLUS,                            // +
        MINUS,                           // -
        MUL,                             // *
        DIV,                             // /
        MOD,                             // %
}

-->创建例子 ： 1 + 1
treeMaker.Binary(JCTree.Tag.PLUS, treeMaker.Literal(1), treeMaker.Literal(1));
```

##### JCReturn

return语句



```java
public static class JCReturn extends JCTree.JCStatement implements ReturnTree {
        public JCTree.JCExpression expr;//返回语句的结果字段
}

-->例子：retrun this.xxx
treeMaker.Return(treeMaker.Select(treeMaker.Ident(names.fromString("this")),names.fromString("xxx")));
```

##### JCAssign

赋值语句



```java
public static class JCAssign extends JCTree.JCExpression implements AssignmentTree {
        public JCTree.JCExpression lhs;//赋值语句左边表达式
        public JCTree.JCExpression rhs;//赋值语句右边表达式
}

-->例子：x = 1
treeMaker.Assign(treeMaker.Ident(names.fromString("x")))), treeMaker.Literal(1))
```

##### JCAssignOp

赋值语句



```java
public static class JCAssignOp extends JCTree.JCExpression implements CompoundAssignmentTree {
        private JCTree.Tag opcode;
        public JCTree.JCExpression lhs;
        public JCTree.JCExpression rhs;
        public Symbol operator;
}

-->opcode可取值
public static enum Tag {
      BITOR_ASG(BITOR),                // |=
      BITXOR_ASG(BITXOR),              // ^=
      BITAND_ASG(BITAND),              // &=

      SL_ASG(SL),                      // <<=
      SR_ASG(SR),                      // >>=
      USR_ASG(USR),                    // >>>=
      PLUS_ASG(PLUS),                  // +=
      MINUS_ASG(MINUS),                // -=
      MUL_ASG(MUL),                    // *=
      DIV_ASG(DIV),                    // /=
      MOD_ASG(MOD),                    // %=
}

-->例子：x += 1
treeMaker.AssignOp(JCTree.Tag.PLUS_ASG, treeMaker.Ident(names.fromString("x")))), treeMaker.Literal(1))
```

##### JCIf

if代码块；  if(condition) {thenpart} else {elsepart}



```java
public static class JCIf extends JCTree.JCStatement implements IfTree {
        public JCTree.JCExpression cond;//条件语句
        public JCTree.JCStatement thenpart;//if的操作语句
        public JCTree.JCStatement elsepart;//else的操作语句
}
```

##### JCForLoop

for循环代码块；for (init; cond; step) {body}



```php
public static class JCForLoop extends JCTree.JCStatement implements ForLoopTree {
        public List<JCTree.JCStatement> init;
        public JCTree.JCExpression cond;
        public List<JCTree.JCExpressionStatement> step;
        public JCTree.JCStatement body;
}
```

##### JCTry、JCCatch

try、catch和finally代码块；



```php
public static class JCTry extends JCTree.JCStatement implements TryTree {
        public JCTree.JCBlock body;//try代码块
        public List<JCTree.JCCatch> catchers;//JCCatch
        public JCTree.JCBlock finalizer;//final代码块
        public List<JCTree> resources;//List.nil()，用不上的字段
        public boolean finallyCanCompleteNormally;//
}

-->JCCatch
public static class JCCatch extends JCTree implements CatchTree {
        public JCTree.JCVariableDecl param;//catch的异常类型
        public JCTree.JCBlock body;//catch代码块
}
```

##### JCMethodInvocation

方法调用表达式



```php
public static class JCMethodInvocation extends JCTree.JCPolyExpression implements MethodInvocationTree {
        public List<JCTree.JCExpression> typeargs;//参数类型列表
        public JCTree.JCExpression meth;//方法的调用语句，比如Log.i，setContentView
        public List<JCTree.JCExpression> args;//参数列表
        public Type varargsElement;//null
}
```

##### JCDoWhileLoop

doWhile循环 ; do（body）while(cond)



```java
public static class JCDoWhileLoop extends JCTree.JCStatement implements DoWhileLoopTree {
        public JCTree.JCStatement body;//do代码块
        public JCTree.JCExpression cond;//条件语句
}
```

##### JCEnhancedForLoop

增强for循环 ; for(var : expr){body}



```java
public static class JCEnhancedForLoop extends JCTree.JCStatement implements EnhancedForLoopTree {
        public JCTree.JCVariableDecl var;
        public JCTree.JCExpression expr;
        public JCTree.JCStatement body;//代码块
}
```

##### JCThrow

异常抛出



```php
public static class JCThrow extends JCTree.JCStatement implements ThrowTree {
        public JCTree.JCExpression expr;//异常表达式
}

-->例子：throw Exception
treeMaker.Throw(treeMaker.Ident(names.fromString("Exception")));
```

##### JCSwitch，JCCase

switch代码块；switch(selector) {case pat : stats}



```php
public static class JCSwitch extends JCTree.JCStatement implements SwitchTree {
        public JCTree.JCExpression selector;//判断条件
        public List<JCTree.JCCase> cases;//多个case
}

public static class JCCase extends JCTree.JCStatement implements CaseTree {
        public JCTree.JCExpression pat;//case逻辑表达式
        public List<JCTree.JCStatement> stats;//代码执行语句
}
```

##### JCConditional

三目运算表达式，cond ? truepart : falsepart



```java
public static class JCConditional extends JCTree.JCPolyExpression implements ConditionalExpressionTree {
        public JCTree.JCExpression cond;//判断条件
        public JCTree.JCExpression truepart;//判断为真时执行的表达式
        public JCTree.JCExpression falsepart;//判断为假时执行的表达式
}
```

##### JCSkip

空操作，即一个无效的分号 ";"

##### JCUnary

一元运算表达式，



```java
public static class JCUnary extends JCTree.JCExpression implements UnaryTree {
        private JCTree.Tag opcode;//操作运算符
        public JCTree.JCExpression arg;//
}

public static enum Tag {
  POS,                             // +
  NEG,                             // -
  NOT,                             // !
  COMPL,                           // ~
  PREINC,                          // ++ _；例子：++i
  PREDEC,                          // -- _； 例子：--i
  POSTINC,                         // _ ++; 例子：i++
  POSTDEC,                         // _ --; 例子：i--
}

-->例子：i++;
treeMaker.Unary(JCTree.Tag.POSTINC, treeMaker.Ident(names.fromString("i")));
```

##### JCContinue

continue，跳过本次循环, continue label



```java
public static class JCContinue extends JCTree.JCStatement implements ContinueTree {
        public Name label;//label标签
        public JCTree target;//null
}
```

##### JCBreak

break，跳出循环, break label



```java
public static class JCBreak extends JCTree.JCStatement implements BreakTree {
        public Name label;//label标签
        public JCTree target;//null
}
```

关于如何操作AST请看[抽象语法树AST的全面分析（三）](https://www.jianshu.com/p/68fcbc154c2f)



前面两篇文章写到了抽象语法树的生成过程和语法树的节点访问，这篇文章来写一下如何操作抽象语法树。



##### 操作AST可以完成什么事情？

拿到了抽象语法树，等于我们拿到了整份的代码，我们可以对所有的代码进行扫描，可以在特定的代码中写入一些逻辑：

- 清除或者添加日志；
- 对象调用的非空判断；
- 编写我们特定的语法规则，对不符合规则的代码进行修改或优化；
- 增删改查。。。

##### AST的优缺点

优点：AST操作属于编译器级别，对程序运行完全没有影响，效率相对其他AOP更高；
 缺点：没有官方文档，操作比较复杂，需要自己摸索。

##### AST实操

**一、清除Log日志**
 创建一个java-library，主module依赖这个library，library的gradle配置如下



```php
apply plugin: 'java-library'

dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
    implementation 'com.google.auto.service:auto-service:1.0-rc2'
    implementation files('libs/tools.jar')
    implementation project(':annotations')
}

sourceCompatibility = "1.8"
targetCompatibility = "1.8"
```

主项目依赖ast_processor



```bash
annotationProcessor project(':ast_processor')
```

library中创建一个ASTProcessor类，让它继承AbstractProcessor，实现最基本的配置。



```java
@AutoService(Processor.class)
@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class ASTProcessor extends AbstractProcessor {

    private Messager mMessager;  //用于打印数据
    private Trees trees;         //提供了待处理的抽象语法树
    private TreeMaker treeMaker;//TreeMaker 封装了创建AST节点的一些方法
    private Names names;        //提供了创建标识符的方法
    private ASTInterf astInterf;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);

        mMessager = processingEnvironment.getMessager();
        trees = Trees.instance(processingEnvironment);//通过trees可以获取到抽象语法书
        Context context = ((JavacProcessingEnvironment) processingEnvironment).getContext();
        treeMaker = TreeMaker.instance(context);
        names = Names.instance(context);
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> stringSet = new LinkedHashSet<>();
        stringSet.add("*");//* 指定所有注解
        return stringSet;
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        if (!roundEnvironment.processingOver()) {
            for (Element element : roundEnv.getRootElements()) {
                if (element.getKind() == ElementKind.CLASS) {
                    JCTree tree = (JCTree) trees.getTree(element);
                    LogClearTranslator logClearTranslator = new LogClearTranslator(mMessager);
                    tree.accept(logClearTranslator);
                }
            }
        }
        return false;
    }

}
```

LogClearTranslator类



```java
public class LogClearTranslator extends TreeTranslator {

    public final static String LOG_TAG = "Log.";

    private Messager messager;

    public LogClearTranslator(Messager messager) {
        this.messager = messager;
    }

    /**
     *  访问代码块
     * */
    @Override
    public void visitBlock(JCTree.JCBlock jcBlock) {
        super.visitBlock(jcBlock);
        //获取所有语句，JCStatement代表一行代码
        List<JCTree.JCStatement> jcStatementList = jcBlock.getStatements();
        if (jcStatementList == null || jcStatementList.isEmpty()){
            return;
        }
        List<JCTree.JCStatement> newList = List.nil();//创建一个新的list,用于装载不包含log的代码语句
        for (JCTree.JCStatement jcStatement : jcStatementList) {
            if (!jcStatement.toString().contains(LOG_TAG)){
                newList = newList.append(jcStatement);//加入非log的代码语句
            }else{
                messager.printMessage(Diagnostic.Kind.NOTE, "clearLog: " + jcStatement.toString());
            }
        }
        jcBlock.stats = newList;//修改代码块的语句list
    }
}
```

小结：复写visitBlock()方法，获取到所有的代码块，对代码块中所有的代码语句进行遍历，去掉包括Log的代码行，重新赋值给jcBlock，非常简单。

![img](https:////upload-images.jianshu.io/upload_images/11238893-94e9511d3f873330.png?imageMogr2/auto-orient/strip|imageView2/2/w/752/format/webp)

MainActivit的java文件.png



![img](https:////upload-images.jianshu.io/upload_images/11238893-85db6c8e441121d0.png?imageMogr2/auto-orient/strip|imageView2/2/w/788/format/webp)

MainActivit的class文件.png


*上面两张图片，带Log语句的java文件编译成.class文件后，Log语句成功的被去掉了。*
**二、难度升级一下，手撸Getter, Setter, toString, hashCode, equals**
 1、自定义Data注解





```css
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.SOURCE)
public @interface Data {}
```

2.给自定义的Bean添加@Data注解



```java
@Data
public class TestBean {
    private int heigth;
    private int age;
    private String nickName;
    private int sex;
}
```

3、创建DataOperationTranslator类（继承TreeTranslator），并且在ASTProcessor中调用



```dart
public class DataOperationTranslator extends TreeTranslator {
}


public class ASTProcessor extends AbstractProcessor {

    ...

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> stringSet = new LinkedHashSet<>();
        stringSet.add("com.example.adams.annotations.Data");
        return stringSet;
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        if (!roundEnvironment.processingOver()) {
            for (Element element : roundEnv.getElementsAnnotatedWith(Data.class)) {//获取@Data注解的元素
                if (element.getKind() == ElementKind.CLASS) {
                    JCTree tree = (JCTree) trees.getTree(element);
                    //创建DataOperationTranslator，传给tree
                    DataOperationTranslator operationTranslator = new DataOperationTranslator(mMessager, treeMaker, names);
                    tree.accept(operationTranslator);
                }
            }
        }
        return false;
    }
}
```

4、getter



```php
private JCTree.JCMethodDecl makeGetterMethod(JCTree.JCVariableDecl jcVariableDecl){
        JCTree.JCModifiers jcModifiers = treeMaker.Modifiers(Flags.PUBLIC);//public
        JCTree.JCExpression retrunType = jcVariableDecl.vartype;//方法返回类型
        Name name = getterMethodName(jcVariableDecl);// 方法名getXxx
        JCTree.JCStatement jcStatement = // retrun this.xxx
                treeMaker.Return(treeMaker.Select(treeMaker.Ident(names.fromString("this")), jcVariableDecl.name));
        List<JCTree.JCStatement> jcStatementList = List.nil();
        jcStatementList = jcStatementList.append(jcStatement);
        JCTree.JCBlock jcBlock = treeMaker.Block(0, jcStatementList);//构建代码块
        List<JCTree.JCTypeParameter> methodGenericParams = List.nil();//泛型参数列表
        List<JCTree.JCVariableDecl> parameters = List.nil();//参数列表
        List<JCTree.JCExpression> throwsClauses = List.nil();//异常抛出列表
        JCTree.JCExpression defaultValue = null;//非自定义注解类中的方法，defaultValue为null
        //最后构建getter方法
        JCTree.JCMethodDecl jcMethodDecl = treeMaker.MethodDef(jcModifiers, name, retrunType, methodGenericParams, parameters, throwsClauses, jcBlock, defaultValue);
        return jcMethodDecl;
    }
```

5、setter



```php
private JCTree.JCMethodDecl makeSetterMethod(JCTree.JCVariableDecl jcVariableDecl){
        JCTree.JCModifiers jcModifiers = treeMaker.Modifiers(Flags.PUBLIC);//public
        JCTree.JCExpression retrunType = treeMaker.TypeIdent(TypeTag.VOID);//或 treeMaker.Type(new Type.JCVoidType())
        Name name = setterMethodName(jcVariableDecl);// setXxx()
        List<JCTree.JCVariableDecl> parameters = List.nil();//参数列表
        JCTree.JCVariableDecl param = treeMaker.VarDef(
                        treeMaker.Modifiers(Flags.PARAMETER), jcVariableDecl.name, jcVariableDecl.vartype, null);
        param.pos = jcVariableDecl.pos;//设置形参这一句不能少，不然会编译报错(java.lang.AssertionError: Value of x -1)
        parameters = parameters.append(param);
        //this.xxx = xxx;  setter方法中的赋值语句
        JCTree.JCStatement jcStatement = treeMaker.Exec(treeMaker.Assign(
                        treeMaker.Select(treeMaker.Ident(names.fromString("this")), jcVariableDecl.name),
                        treeMaker.Ident(jcVariableDecl.name)));
        List<JCTree.JCStatement> jcStatementList = List.nil();
        jcStatementList = jcStatementList.append(jcStatement);
        JCTree.JCBlock jcBlock = treeMaker.Block(0, jcStatementList);//代码块
        List<JCTree.JCTypeParameter> methodGenericParams = List.nil();//泛型参数列表
        List<JCTree.JCExpression> throwsClauses = List.nil();//异常抛出列表
        JCTree.JCExpression defaultValue = null;
         //最后构建setter方法
        JCTree.JCMethodDecl jcMethodDecl = treeMaker.MethodDef(jcModifiers, name, retrunType, methodGenericParams, parameters, throwsClauses, jcBlock, defaultValue);
        return jcMethodDecl;
    }
```

6、toString



```php
private JCTree.JCMethodDecl makeToStringMethod(JCTree.JCClassDecl jcClassDecl){
        List<JCTree.JCVariableDecl> jcVariableDeclList = List.nil();
        for (JCTree jcTree : jcClassDecl.defs) {
            if (jcTree instanceof JCTree.JCVariableDecl){
                JCTree.JCVariableDecl jcVariableDecl = (JCTree.JCVariableDecl) jcTree;
                jcVariableDeclList = jcVariableDeclList.append(jcVariableDecl);
            }
        }

        List<JCTree.JCAnnotation> jcAnnotationList = List.nil();
        JCTree.JCAnnotation jcAnnotation = treeMaker.Annotation(memberAccess("java.lang.Override"), List.nil());
        jcAnnotationList = jcAnnotationList.append(jcAnnotation);
        JCTree.JCModifiers jcModifiers = treeMaker.Modifiers(Flags.PUBLIC, jcAnnotationList);
        JCTree.JCExpression retrunType = memberAccess("java.lang.String");
        Name name = names.fromString("toString");
        JCTree.JCExpression jcExpression = treeMaker.Literal(jcClassDecl.name + "{");
        for (int i = 0; i < jcVariableDeclList.size(); i++) {
            JCTree.JCVariableDecl jcVariableDecl = jcVariableDeclList.get(i);
            if (i != 0){
                jcExpression = treeMaker.Binary(JCTree.Tag.PLUS, jcExpression, treeMaker.Literal("," + jcVariableDecl.name.toString() + "="));
            }else{
                jcExpression = treeMaker.Binary(JCTree.Tag.PLUS, jcExpression, treeMaker.Literal(jcVariableDecl.name.toString() + "="));
            }
            if (jcVariableDecl.vartype.toString().contains("String")){
                jcExpression = treeMaker.Binary(JCTree.Tag.PLUS, jcExpression, treeMaker.Literal("'"));
            }
            jcExpression = treeMaker.Binary(JCTree.Tag.PLUS, jcExpression, treeMaker.Ident(jcVariableDecl.name));
            if (jcVariableDecl.vartype.toString().contains("String")){
                jcExpression = treeMaker.Binary(JCTree.Tag.PLUS, jcExpression, treeMaker.Literal("'"));
            }
        }

        jcExpression = treeMaker.Binary(JCTree.Tag.PLUS, jcExpression, treeMaker.Literal("}"));
        JCTree.JCStatement jcStatement = treeMaker.Return(jcExpression);
        List<JCTree.JCStatement> jcStatementList = List.nil();
        jcStatementList = jcStatementList.append(jcStatement);
        JCTree.JCBlock jcBlock = treeMaker.Block(0, jcStatementList);
        List<JCTree.JCTypeParameter> methodGenericParams = List.nil();//泛型参数列表
        List<JCTree.JCVariableDecl> parameters = List.nil();//参数列表
        List<JCTree.JCExpression> throwsClauses = List.nil();//异常抛出列表
        JCTree.JCExpression defaultValue = null;
        JCTree.JCMethodDecl jcMethodDecl = treeMaker.MethodDef(jcModifiers, name, retrunType, methodGenericParams, parameters, throwsClauses, jcBlock, defaultValue);
        return jcMethodDecl;
    }
```

7、hashCode



```php
private JCTree.JCMethodDecl makeHashCodeMethod(JCTree.JCClassDecl jcClassDecl){
        List<JCTree.JCVariableDecl> jcVariableDeclList = List.nil();
        for (JCTree jcTree : jcClassDecl.defs) {
            if (jcTree instanceof JCTree.JCVariableDecl){
                JCTree.JCVariableDecl jcVariableDecl = (JCTree.JCVariableDecl) jcTree;
                jcVariableDeclList = jcVariableDeclList.append(jcVariableDecl);
            }
        }
        List<JCTree.JCAnnotation> jcAnnotationList = List.nil();
        JCTree.JCAnnotation jcAnnotation = treeMaker.Annotation(memberAccess("java.lang.Override"), List.nil());
        jcAnnotationList = jcAnnotationList.append(jcAnnotation);
        JCTree.JCModifiers jcModifiers = treeMaker.Modifiers(Flags.PUBLIC, jcAnnotationList);
        Name name = names.fromString("hashCode");
        JCTree.JCExpression retrunType = treeMaker.TypeIdent(TypeTag.INT);

        List<JCTree.JCExpression> var1 = List.nil();
        List<JCTree.JCExpression> var2 = List.nil();
        for (JCTree.JCVariableDecl variableDecl:jcVariableDeclList) {
            var1 = var1.append(typeTranslator(variableDecl.vartype));
            var2 = var2.append(treeMaker.Ident(variableDecl.name));
        }
        //创建代码：Objects.hash(xxx ...)
        JCTree.JCStatement jcStatement =
                treeMaker.Return(treeMaker.Apply(var1, memberAccess("java.util.Objects.hash"), var2));
        List<JCTree.JCStatement> jcStatementList = List.nil();
        jcStatementList = jcStatementList.append(jcStatement);
        JCTree.JCBlock jcBlock = treeMaker.Block(0, jcStatementList);
        List<JCTree.JCTypeParameter> methodGenericParams = List.nil();//泛型参数列表
        List<JCTree.JCVariableDecl> parameters = List.nil();//参数列表
        List<JCTree.JCExpression> throwsClauses = List.nil();//异常抛出列表
        JCTree.JCExpression defaultValue = null;
        JCTree.JCMethodDecl jcMethodDecl = treeMaker.MethodDef(jcModifiers, name, retrunType, methodGenericParams, parameters, throwsClauses, jcBlock, defaultValue);
        return jcMethodDecl;
    }
```

8、equals



```dart
private JCTree.JCMethodDecl makeEqualsMethod(JCTree.JCClassDecl jcClassDecl){
        List<JCTree.JCVariableDecl> notStringJcVariableDeclList = List.nil();
        List<JCTree.JCVariableDecl> stringJcVariableDeclList = List.nil();
        for (JCTree jcTree : jcClassDecl.defs) {
            if (jcTree instanceof JCTree.JCVariableDecl){
                JCTree.JCVariableDecl jcVariableDecl = (JCTree.JCVariableDecl) jcTree;
                if (jcVariableDecl.vartype.toString().equals("String")){
                    stringJcVariableDeclList = stringJcVariableDeclList.append(jcVariableDecl);
                }else{
                    notStringJcVariableDeclList = notStringJcVariableDeclList.append(jcVariableDecl);
                }
            }
        }

        List<JCTree.JCAnnotation> jcAnnotationList = List.nil();
        JCTree.JCAnnotation jcAnnotation = treeMaker.Annotation(memberAccess("java.lang.Override"), List.nil());
        jcAnnotationList = jcAnnotationList.append(jcAnnotation);
        JCTree.JCModifiers jcModifiers = treeMaker.Modifiers(Flags.PUBLIC, jcAnnotationList);
        Name name = names.fromString("equals");
        JCTree.JCExpression retrunType = treeMaker.TypeIdent(TypeTag.BOOLEAN);


        List<JCTree.JCStatement> jcStatementList = List.nil();
        // if (this == o) return false;
        JCTree.JCStatement zeroth = treeMaker.If(treeMaker.Binary(JCTree.Tag.EQ, treeMaker.Ident(names.fromString("this")), treeMaker.Ident(names.fromString("o"))),
                treeMaker.Return(treeMaker.Literal(false)), null);
        jcStatementList = jcStatementList.append(zeroth);
        //if (!(o instanceof TestBean)) return false;
        JCTree.JCStatement first = treeMaker.If(treeMaker.Unary(JCTree.Tag.NOT, treeMaker.TypeTest(treeMaker.Ident(names.fromString("o")), treeMaker.Ident(jcClassDecl.name))),
                treeMaker.Return(treeMaker.Literal(false)), null);
        jcStatementList = jcStatementList.append(first);
        //TestBean testBean = (TestBean)o;
        JCTree.JCVariableDecl second = treeMaker.VarDef(
                treeMaker.Modifiers(0), names.fromString(toLowerCaseFirstOne(jcClassDecl.name.toString())), treeMaker.Ident(jcClassDecl.name),
                treeMaker.TypeCast(treeMaker.Ident(jcClassDecl.name), treeMaker.Ident(names.fromString("o"))));
        jcStatementList = jcStatementList.append(second);

        JCTree.JCExpression jcExpression = null;
        for (int i = 0; i < notStringJcVariableDeclList.size(); i++) {
            JCTree.JCExpression isEq = treeMaker.Binary(JCTree.Tag.EQ,
                    treeMaker.Ident(notStringJcVariableDeclList.get(i).name),
                    treeMaker.Select(treeMaker.Ident(names.fromString(toLowerCaseFirstOne(jcClassDecl.name.toString()))),
                            notStringJcVariableDeclList.get(i).name));
            if (jcExpression != null){
                //&& this.age == testBean.age
                jcExpression = treeMaker.Binary(JCTree.Tag.AND, jcExpression, isEq);
            }else{
                jcExpression = isEq;
            }
        }

        for (int i = 0; i < stringJcVariableDeclList.size(); i++) {
            List<JCTree.JCExpression> var1 = List.nil();
            var1 = var1.append(memberAccess("java.lang.String"));
            var1 = var1.append(memberAccess("java.lang.String"));
            List<JCTree.JCExpression> var2 = List.nil();
            var2 = var2.append(treeMaker.Ident(stringJcVariableDeclList.get(i).name));
            var2 = var2.append(treeMaker.Select(treeMaker.Ident(names.fromString(toLowerCaseFirstOne(jcClassDecl.name.toString()))),
                    stringJcVariableDeclList.get(i).name));
            JCTree.JCExpression isEq = treeMaker.Apply(var1, memberAccess("java.util.Objects.equals"), var2);
            if (jcExpression != null){
                //&& Objects.equals(this.nickName, testBean.nickName);
                jcExpression = treeMaker.Binary(JCTree.Tag.AND, jcExpression, isEq);
            }else{
                jcExpression = isEq;
            }
        }
        JCTree.JCStatement fourth = treeMaker.Return(jcExpression);//return语句
        jcStatementList = jcStatementList.append(fourth);
        JCTree.JCBlock jcBlock = treeMaker.Block(0, jcStatementList);

        List<JCTree.JCTypeParameter> methodGenericParams = List.nil();//泛型参数列表
        List<JCTree.JCVariableDecl> parameters = List.nil();//参数列表
        JCTree.JCVariableDecl param = treeMaker.VarDef(
                treeMaker.Modifiers(Flags.PARAMETER), names.fromString("o"), memberAccess("java.lang.Object"), null);
        param.pos = jcClassDecl.pos;
        parameters = parameters.append(param);//添加参数 Object o
        List<JCTree.JCExpression> throwsClauses = List.nil();//异常抛出列表
        JCTree.JCExpression defaultValue = null;
        JCTree.JCMethodDecl jcMethodDecl = treeMaker.MethodDef(jcModifiers, name, retrunType, methodGenericParams, parameters, throwsClauses, jcBlock, defaultValue);
        return jcMethodDecl;
    }
```

9、把上面的方法加入Bean中



```java
@Override
    public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
        super.visitClassDef(jcClassDecl);
        for (JCTree jcTree : jcClassDecl.defs) {
            if (jcTree instanceof JCTree.JCVariableDecl){
                JCTree.JCVariableDecl jcVariableDecl = (JCTree.JCVariableDecl) jcTree;
                jcClassDecl.defs = jcClassDecl.defs.append(makeGetterMethod(jcVariableDecl));
                jcClassDecl.defs = jcClassDecl.defs.append(makeSetterMethod(jcVariableDecl));
            }
        }
        JCTree.JCMethodDecl toString = makeToStringMethod(jcClassDecl);
        jcClassDecl.defs = jcClassDecl.defs.append(toString);
        JCTree.JCMethodDecl hashCodeMethod = makeHashCodeMethod(jcClassDecl);
        jcClassDecl.defs = jcClassDecl.defs.append(hashCodeMethod);
        JCTree.JCMethodDecl equalsMethod = makeEqualsMethod(jcClassDecl);
        jcClassDecl.defs = jcClassDecl.defs.append(equalsMethod);
    }
```

结果展示：

![img](https:////upload-images.jianshu.io/upload_images/11238893-ce54b486a48e566e.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/745/format/webp)

TestBean的java文件.jpg



![img](https:////upload-images.jianshu.io/upload_images/11238893-6ee5af1a3393e817.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

TestBean的class文件.jpg


 项目源码：[ASTDemo](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fadams-github%2FASTDemo)



> 总结：AST操作主要还是复写JCTree.Visitor中的visistXxx()的方法，获取对应的语法节点，对该语法节点进行操作；调用TreeMaker的方法，利用TreeMaker生成语法节点以及实现代码的调用。由于没有官方的文档，总体比较难的还是API的调用，以及编译出错时无法准确定位问题代码。

