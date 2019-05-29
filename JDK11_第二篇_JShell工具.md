---
title: JDK11 | 第二篇 : JShell 工具
date: 2019-05-29 22:47:40:040
tags: [Java] 
published: true
feature: https://user-gold-cdn.xitu.io/2019/5/29/16b02e962ac9b254?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文转载自 [https://juejin.im/post/5cee4ffd5188257abf7d8461](https://juejin.im/post/5cee4ffd5188257abf7d8461) 

## 一、简介

Java Shell工具是JDK1.9出现的工具， Java Shell工具（JShell）是一个用于学习Java编程语言和Java代码原型的交互式工具。JShell是一个Read-Evaluate-Print循环（REPL），它在输入时评估声明，语句和表达式，并立即显示结果。该工具从命令行运行。

## 二、为什么要使用JShell ？

使用JShell，您可以一次输入一个程序元素，立即查看结果，并根据需要进行调整。
Java程序开发通常涉及以下过程：

	* 写一个完整的程序。
	* 编译它并修复任何错误。
	* 运行程序。
	* 弄清楚它有什么问题。
	* 编辑它。
	* 重复这个过程。

JShell可帮助您在开发程序时尝试代码并轻松探索选项。您可以测试单个语句，尝试不同的方法变体，并在JShell会话中试验不熟悉的API。JShell不替换IDE。在开发程序时，将代码粘贴到JShell中进行试用，然后将JShell中的工作代码粘贴到程序编辑器或IDE中。

## 三、JShell的使用

### 1. 启动和退出

使用JShell需要配置好java的环境变量。

启动：
```
jshell
```

要以详细模式启动JShell，请使用以下-v选项：

```
jshell -v
```

退出：

```
/exit
```

### 2. 运行代码片段

使用详细选项启动JShell以获得最大可用反馈量：
```
jshell -v
|  欢迎使用 JShell -- 版本 11.0.2
|  要大致了解该版本, 请键入: /help intro
```

在提示符处输入以下示例语句，并查看显示的输出：

```
jshell> int x = 45
x ==> 45
|  已创建 变量 x : int
```

首先，显示结果。将其读作：变量x的值为45.因为您处于详细模式，所以还会显示所发生情况的描述。

注意：如果未输入分号，则会自动将终止分号添加到完整代码段的末尾。

当输入的表达式没有命名变量时，会创建一个临时变量，以便稍后可以引用该值。以下示例显示表达式和方法结果的临时值。该示例还显示了...> 在代码段需要多行输入完成时使用的continuation prompt（）：
```
jshell> String twice(String s) {
   ...>   return s + s;
   ...> }
|  已创建 方法 twice(String)

jshell> twice("Oecan")
$4 ==> "OecanOecan"
|  已创建暂存变量 $4 : String
```

### 3. 改变定义

在试验代码时，您可能会发现变量，方法或类的定义没有按照您希望的方式执行。通过输入新的定义可以轻松更改定义，该定义将覆盖先前的定义。
要更改变量，方法或类的定义，只需输入新定义即可。例如，twice在定义该方法尝试片段得到在下面的示例中的新定义：
```
jshell> String twice(String s) {
   ...>   return "Twice: " + s;
   ...> }
|  已修改 方法 twice(String)
|    更新已覆盖 方法 twice(String)

jshell> twice("thing")
$6 ==> "Twice: thing"
|  已创建暂存变量 $6 : String
```

还可以改变变量的类型定义。以下示例显示x从String更改int为：

```
jshell> int x = 45
x ==> 45
|  已创建 变量 x : int

jshell> String x
x ==> null
|  已替换 变量 x : String
|    更新已覆盖 变量 x : int
```

### 4. 查看默认导入和使用自动补全功能

默认情况下，JShell提供了一些常用包的导入，我们可以使用 **import** 语句导入必要的包或是从指定的路径的包，来运行我们的代码片段。我们可以输入以下命令列出所有导入的包：
```
jshell> /imports 
|    import java.io.*
|    import java.math.*
|    import java.net.*
|    import java.nio.file.*
|    import java.util.*
|    import java.util.concurrent.*
|    import java.util.function.*
|    import java.util.prefs.*
|    import java.util.regex.*
|    import java.util.stream.*
```

### 5. 自动补全的功能

当我们想输入System类时，根据前面说的自动补全，只需要输入Sys然后按下 Tab 键，则自动补全， 然后再输入“.o”，则会自动补全方法， 在补全“System.out.”后按下 Tab 键，接下来就会列出当前类的所有的 public 方法的列表：
```
jshell> System
签名:
java.lang.System

<再次按 Tab 可查看文档>

jshell> System.out.
append(        checkError()   close()        equals(        flush()        format(        getClass()     
hashCode()     notify()       notifyAll()    print(         printf(        println(       toString()     
wait(          write(
```

### 6. 列出到目前为止当前 session 里所有有效的代码片段

```
jshell> /list 

   2 : 2+2
   4 : twice("Oecan")
   5 : String twice(String s) {
         return "Twice: " + s;
       }
   6 : twice("thing")
   8 : String x;
```

### 7. 列出到目前为止当前 session 里所有方法

```
jshell> /methods 
|    String twice(String)
```

### 8. 使用外部代码编辑器来编写 Java 代码

现在，我想对twice方法做一些改动，如果这时有外部代码编辑器的话，做起来会很容易。在 JShell 中可以启用JShell Edit Pad 编辑器，需要输入如下命令，来修改上面的方法：

![](https://user-gold-cdn.xitu.io/2019/5/29/16b02e962ac9b254?imageView2/0/w/1280/h/960/ignore-error/1)

代码修改完成以后，先点击“Accept”按钮，再点击“Exit”按钮，则退出编辑器，在 JShell 命令行中提示方法已经修改。

### 8. 从外部加载源代码

如果在外部已经有写好的 Java 文件，可以使用/open 命令导入到 JShell 环境中，例如现在有一个Test.java文件：
```
void say(String name) {
     System.out.println("hello " + name);
}
``````
jshell> /open /Users/Documents/java11/Test.java

jshell> /methods
|    String twice(String)
|    void say(String)

jshell> say("zhangsan")
hello zhangsan
```

JShell工具的更多使用方法，请参照官方示例：[docs.oracle.com/javase/9/js…](https://link.juejin.im?target=https%3A%2F%2Fdocs.oracle.com%2Fjavase%2F9%2Fjshell%2F)

欢迎扫码或微信搜索公众号《程序员果果》关注我，关注有惊喜~
![](https://user-gold-cdn.xitu.io/2019/4/11/16a09fab998c3fb5?imageView2/0/w/1280/h/960/ignore-error/1)

