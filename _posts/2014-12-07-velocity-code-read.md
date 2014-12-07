---
layout: post
title: "Velocity源码阅读"
description: ""
category: code
tags: [velocity]
---
{% include JB/setup %}

> Velocity 是一个基于Java的模板引擎。它高效、简洁，且有很好的可扩展性，适合在Web项目中做为MVC模型中的视图，也可以在内容生成、数据转换等应用中独立使用。本文将以源码阅读的方式，浅析velocity的实现机制，并通过几个小例子来阐述velocity中主要使用的技术点。

### 1.下载源码
Velocity是Apache的顶级项目。可以从 http://velocity.apache.org/ 上获取最新源码，最新版本是1.7。为了契合日常的工作，这里使用1.6.3版本做为本文基准版本，可以从 http://archive.apache.org/dist/velocity/engine/ 获取Older releases版本。

下载：http://archive.apache.org/dist/velocity/engine/1.6.3/velocity-1.6.3.tar.gz

原始的velocity使用ant构建工程，如果要使用maven可以单独下载对应的pom文件，重命名后放入工程目录。
http://archive.apache.org/dist/velocity/engine/1.6.3/velocity-1.6.3.pom

下载完成后，解压，放入pom.xml，然后 ```mvn eclipse:eclipse -DdownloadSources=true```，可以愉快的看源码了~

### 2.代码结构
看代码前，先来了解一下velocity的工程目录结构：

| 目录 | 说明 |
|-
| build/ | 存放一些ant构建脚本，比如可以使用```ant javadocs```生成javadoc |
| convert/ | WebMacro到Velocity的转换程序，WebMacro是老牌模板引擎，Velocity脱胎于它，参见 http://velocity.apache.org/engine/releases/velocity-1.4/differences.html 了解它们的不同。|
| docs/ | Velocity开发文档，快速指南等 |
| docs/api/ | Velocity Javadocs |
| examples/ | 如何使用Velocity的几个例子 |
| lib/ | velocity依赖的包 |
| lib/test/ | 单元测试依赖的包 |
| src/ | 源码包 |
| test/ | 单测 |
| xdocs/ | 从xml构建docs站点的源文件 |
| LICENSE | Apache License Version 2.0 ，可作为开源或商业软件再发布 |
| velocity-1.6.3.jar | velocity jar |
| velocity-1.6.3-dep.jar | velocity 将依赖打入的standalone jar |

#### 2.1.Hello World !
> 在开始Hello World前，可以思考一下，如果是自己来设计一个模板引擎需要哪些步骤？
> 
> 嗯，应该需要一个上下文变量Context，用于存放一些会渲染到模板中的变量；然后通过文件或是常量字符串来获取模板；模板里面掏几个占位符，用一些特殊的标记隔离出来(比如#name#)；最后解析模板中的占位符，将Context中的变量的实际值替换进去，然后输出merge后的结果完成模板渲染。(之前自定义短信模板这么做的，很轻巧；邮件内容较多，则使用velocity。)

思考过后，我们来run一个examples目录下的例子，找到应用程序的入口点，然后进行代码阅读，看看跟我们想的是否一样。
将examples/下的第一个例子app_example1/的样例文件(主要是Example.java、example.vm、velocity.properties)copy到src里。

Example

```
public class Example {
	public Example(String templateFile) {
		try {
			/*
			 * Velocity的初始化可以使用Velocity类通过RuntimeSingleton类的单例模式初始化，或者使用
			 * VelocityEngine类创建并初始化。两者用在不同的场景。
			 */
			Velocity.init("velocity.properties");

			/*
			 * 构建模板上下文，放入一些将会渲染的变量。
			 */
			VelocityContext context = new VelocityContext();
			context.put("list", getNames());

			/*
			 * 从文件获取模板：此过程会在调用Template.process()过程中生成抽象语法树AST。这个貌似比想象的要复杂一点，
			 * 稍后来讨论AST相关内容。
			 */
			Template template = null;
			try {
				template = Velocity.getTemplate(templateFile);
			} catch (ResourceNotFoundException rnfe) {
				System.out.println("Example : error : cannot find template "
						+ templateFile);
			} catch (ParseErrorException pee) {
				System.out.println("Example : Syntax error in template "
						+ templateFile + ":" + pee);
			}

			/*
			 * 模板执行merge动作，将context的变量替换到template中，此过程使用到java的Introspection机制，实现了一套反射功能。
			 */
			BufferedWriter writer = writer = new BufferedWriter(
					new OutputStreamWriter(System.out));
			if (template != null)
				template.merge(context, writer);

			/*
			 * flush and cleanup，放finally比较好，虽然只是测试代码
			 */
			writer.flush();
			writer.close();
		} 
		...
	}

	...
}
```

通过阅读和执行Example例子，我们看到velocity运转时的几个关键步骤：

1.引擎初始化。

2.获取模板文件并解析生成AST。

3.构建context并将其merge到模板中输出。


接下来将就这三个重要步骤做进一步的阅读和分析。

### 3.初始化过程

RuntimeInstance.init()

```
public synchronized void init() throws Exception {
		...
				// 初始化配置，先读org/apache/velocity/runtime/defaults/velocity.properties，然后再合并应用特殊配置。
				initializeProperties();

				// 初始化日志，可通过properties文件配置使用何种日志组件。
				initializeLog();

				// 初始化资源管理类，对模板资源的管理、缓存等。
				initializeResourceManager();

				// 初始化velocity指令配置：org/apache/velocity/runtime/defaults/directive.properties。加载8个预定义指令，比如Foreach，以及userdirective配置的用户指令。
				initializeDirectives();

				// 初始化5种事件处理器，包括空值处理、异常处理等。详见org.apache.velocity.app.event包。
				initializeEventHandlers();

				// 初始化解析器池。
				initializeParserPool();

				// 初始化反射处理类。
				initializeIntrospection();
				
				// 初始化宏工厂，我们日常使用的macros.vm。
				vmFactory.initVelocimacro();
		...
	}
```

初始化过程涉及到比较多的点，本文将不展开分析，着重分析其中两个点：解析器和反射，也正是关键的运行步骤中的2、3。

### 4.抽象语法树AST  
Velocity是通过JavaCC和JJTree生成抽象语法树的。说到JavaCC我们先来了解它相关的几个概念。

> BNF -- 复杂语言的语法通常都是使用 BNF（巴科斯-诺尔范式，Backus-Naur Form）表示法或者其“近亲”― EBNF（扩展的 BNF）描述的。

```
BNF 规定是推导规则(产生式)的集合，写为：
<符号> ::= <使用符号的表达式>

这里的 <符号> 是非终结符，而表达式由一个符号序列，或用指示选择的竖杠 '|' 分隔的多个符号序列构成，每个符号序列整体都是左端的符号的一种可能的替代。从未在左端出现的符号叫做终结符。
```
 
> JavaCC 是一个用于生成解析器的工具，它可以将一份语法定义（以.jj为后缀的文件）转化成Java代码用于检查一本文本是否符合这一份语法定义。

> JJTree 是JavaCC提供的一个工具，JJTree可以将一份语法定义（以.jjt为后缀的文件，语法和.jj文件基本相同）转化成Java 代码，这段代码可以检查一份输入是否符合这一份语法定义。并且最后还会生成一颗抽象语法树提供给使用者来遍历。Velocity就将其模板语法定义成了一个jjt文件，然后根据这一份jjt文件生成了velocity模板的解析器。

#### 4.1.JavaCC Examples
简单了解了JavaCC的相关概念之后，我们来看JavaCC自带的例子如何构建一个AST。

从http://javacc.java.net/下载JavaCC工具(内含JJTree工具)。本文使用5.0版本。

解压后，进入到javacc-5.0/examples/SimpleExamples目录，先看一个纯JavaCC的简单例子。

```
// 参数配置段：Simple1中列举了JavaCC配置的默认值。
options {
  ... 
}

// 解析器段：这里可以编写任意Java代码，只要保证PARSER_BEGIN、PARSER_END的参数与Java类名一致。
PARSER_BEGIN(Simple1)

/** Simple brace matcher. */
public class Simple1 {

  /** Main entry point. */
  public static void main(String args[]) throws ParseException {
    Simple1 parser = new Simple1(System.in);
    parser.Input();
  }

}

PARSER_END(Simple1)

// 产生式段
/** Root production. */
void Input() :
{}
{
  MatchedBraces() ("\n"|"\r")* <EOF>
}

/** Brace matching production. */
void MatchedBraces() :
{}
{
  "{" [ MatchedBraces() ] "}"
}
```

如上一个jj文件包含几个段落的代码，参数配置、解析器、产生式，常用的还有SKIP(跳词)、TOKEN(关键字)等，更多可参见 https://javacc.java.net/doc/javaccgrm.html。

正如上面介绍的BNF范式，jj文件中的产生式即是遵循它来实现，只是语法上更贴近Java的方法；
冒号左侧的"符号"即是方法名，冒号右侧的"使用符号的表达式"分为两部分：第一个{}是声明，用于声明变量和一些初始化的动作，如果没有可以忽略，使用方式参见Simple3例子；第二个{}是语法部分和动作部分，语法部分表示左侧可被替换的项，动作部分参见Simple3例子，可以是返回值或者一组动作。

Simple1的Input方法的语法部分 ```MatchedBraces() ("\n"|"\r")* <EOF>``` 的含义是：可被MatchedBraces这个产生式替代，随后可有任意个回车换行符号，最后以文件结束符结尾(<EOF>是javacc系统定义的TOKEN，表示文件结束符号)。

MatchedBraces方法的语法部分 ```"{" [ MatchedBraces() ] "}"``` 的含义是：一个以{开始，并配对的以}结束的串，期间可以有任意个递归的MatchedBraces方法。

这份解析器可以解析如 ```"{ }", "{ { { { { } } } } }"``` 这样的串；
不能解析 ```"{ { { {", "{ } { }", "{ } }", "{ { } { } }", "{ }", "{x}"``` 这样的串。

#### 4.2.JJTree Examples
接下来，我们通过编译运行的方式看一个结合了JJTree的例子。
进入到javacc-5.0/examples/JJTreeExamples目录，先来手工编译一把eg1.jjt。(本身例子自带ant编译文件，这里了解下编译过程)。

```
jjtree eg1.jjt // 生成Node文件及jj文件
javacc eg1.jj // 生成Eg1.java及相关解析文件
javac *.java // 编译Java源文件
```

```
java Eg1 // 运行例子，输入待解析的文本
1+1; // 回车生成一个AST

Start
 Expression
  AdditiveExpression
   MultiplicativeExpression
    UnaryExpression
     Integer
   MultiplicativeExpression
    UnaryExpression
     Integer
```

eg1例子实现了4则运算，先乘除后加减，如果有括号则先解析括号中的。如上如果是一个简单加法运算，它生成的AST将会包含空节点：MultiplicativeExpression(乘除)、UnaryExpression(括号)，除此之外就是一颗树了。大家可以输入更为复杂的4则运算表达式以观察生成的树的构成。

例子中通过SimpleNode的dump方法输出了Tree。SimpleNode是JJTree中表示一个节点的父类。

```
SimpleNode n = t.Start();
n.dump("");

...

/** Main production. */
SimpleNode Start() : {}
{
  Expression() ";"
  { return jjtThis; }
}
```

#### 4.2.从源码构建Velocity的解析器
看完上面的JJTree的简单例子我们就大体上清楚velocity解析器的构建方式。velocity的解析器需要JavaCC 3.2版本以上。

velocity的jjt文件在src/parser/Parser.jjt，产生的jj文件及其他生成的文件在org.apache.velocity.runtime.parser包下。在其子包node下，有jjt文件生成的一组继承自SimpleNode的节点。

```
jjtree Parser.jjt // 生成Parser.jj、一组Node的Java文件骨架(需要自行实现Node的功能性代码)等
javacc Parser.jj // 生成解析器
```

最终生成的解析器会在Template.process()方法中调用。
要想完整的理解Parser.jjt文件，还需要进一步阅读javacc-5.0/doc/JJTree.html文档，了解jjtree相关的扩展语法。

### 5.模板渲染
velocity中为了将context中的复杂对象merge到模板中渲染呈现，实现了一套自省机制。可以通过Enterprise Architect来对org.apache.velocity.util.introspection包下的代码做逆向工程，生成UML图以便阅读代码结构。如下图是去除枝节后的类图结构：

![image](/postimg/velocity1.jpg)

图1 velocity自省类图

通过类图我们来梳理一下其中的关系。
Uberspect是主要的自省/反射接口，提供方法如下：

```
init()：初始化
getIterator()：支持迭代 #foreach
getMethod()：支持方法调用 $foo.bar( $woogie )
getPropertyGet()：支持获取属性值 $bar.woogie
getPropertySet()：支持设置属性值 #set($foo.bar = "geir")
```

Uberspect的实现UberspectImpl，该实现使用Introspector完成自省功能。Introspector扩展自基类IntrospectorBase，增添日志记录。
IntrospectorBase内部维护了一个IntrospectCache，用于缓存已经完成自省的类和方法信息。
IntrospectorCacheImpl内通过一个HashMap维护一个class与其对应的类型信息，类型信息用一个ClassMap表示。
一个ClassMap内部维护了一个MethodCache，用于缓存该类已经解析出得方法信息。MethodMap表示一个方法信息。

了解完类图之后，我们来看一个例子，通过这个例子可以更好的理解模板渲染过程。

#### 5.1.velocity的AST
在org.apache.velocity.runtime.parser包下找到Parser.java(本类即上面通过jjtree&javacc构建的解析器入口文件)。
给Parser.java类加个main方法，以便将解析生成的AST dump输出出来。

```
public static void main(String[] args) {
	String temp = "我是一名来自$company的$person.business('short'),我叫$person.name";
	CharStream stream = new VelocityCharStream(new ByteArrayInputStream(temp.getBytes()), 0, 0);
	Parser t = new Parser(stream);
	try {
		SimpleNode n = t.process();
		n.dump("");
	} catch (Exception e) {
		e.printStackTrace();
	}
}
```

输出的AST：

```
*.node.ASTprocess@68a75974[id=0,info=0,invalid=false,children=6,tokens=[我是一名来自],	[$company],	[的],	[$person],	[.],	[business],	[(],	['short'],	[)],	[,我叫],	[$person],	[.],	[name],	[e]]
	*.node.ASTText@42e20459[id=19,info=0,invalid=false,children=0,tokens=[我是一名来自]]
	*.node.ASTReference@48b915d[id=16,info=0,invalid=false,children=0,tokens=[$company]]
	*.node.ASTText@66f472ff[id=19,info=0,invalid=false,children=0,tokens=[的]]
	*.node.ASTReference@3aa9f827[id=16,info=0,invalid=false,children=1,tokens=[$person],	[.],	[business],	[(],	['short'],	[)]]
		*.node.ASTMethod@6ce2e687[id=15,info=0,invalid=false,children=2,tokens=[business],	[(],	['short'],	[)]]
			*.node.ASTIdentifier@248ce0ea[id=8,info=0,invalid=false,children=0,tokens=[business]]
			*.node.ASTStringLiteral@1d023565[id=7,info=0,invalid=false,children=0,tokens=['short']]
	*.node.ASTText@7bff88c3[id=19,info=0,invalid=false,children=0,tokens=[,我叫]]
	*.node.ASTReference@456bf9ce[id=16,info=0,invalid=false,children=1,tokens=[$person],	[.],	[name]]
		*.node.ASTIdentifier@33dd66fd[id=8,info=0,invalid=false,children=0,tokens=[name]]
```

可以看到，temp模板中的纯文本被解析为ASTText节点，变量被解析为ASTReference节点，如果是方法则被解析为ASTMethod(方法参数解析为各种类型节点，比如ASTStringLiteral)，如果是属性解析为ASTIdentifier。

#### 5.2.merge过程
Template.merge方法中调用了AST的根节点(ASTprocess)的render方法 ```( (SimpleNode) data ).render( ica, writer);``` 。此调用将迭代处理各个子节点render方法。如果是ASTReference类型的节点则在render方法中会调用execute方法执行反射替换相关处理。

以ASTMethod为例，来看一下execute方法的反射过程。

```
public class ASTMethod extends SimpleNode {

public Object execute(Object o, InternalContextAdapter context)
        throws MethodInvocationException
    {
        // 获取方法
        VelMethod method = null;

        Object [] params = new Object[paramCount];

        try
        {
            // 获得参数类型
            final Class[] paramClasses = paramCount > 0 ? new Class[paramCount] : ArrayUtils.EMPTY_CLASS_ARRAY;

            for (int j = 0; j < paramCount; j++)
            {
                params[j] = jjtGetChild(j + 1).value(context);

                if (params[j] != null)
                {
                    paramClasses[j] = params[j].getClass();
                }
            }

            // 尝试从缓存中获取方法
            MethodCacheKey mck = new MethodCacheKey(methodName, paramClasses);
            IntrospectionCacheData icd =  context.icacheGet( mck );
            if ( icd != null && (o != null && icd.contextData == o.getClass()) )
            {
                method = (VelMethod) icd.thingy;
            }
            else
            {
                // 缓存获取不到则使用自省机制获取，并写入缓存
                method = rsvc.getUberspect().getMethod(o, methodName, params, new Info(getTemplateName(), getLine(), getColumn()));

                if ((method != null) && (o != null))
                {
                    icd = new IntrospectionCacheData();
                    icd.contextData = o.getClass();
                    icd.thingy = method;

                    context.icachePut( mck, icd );
                }
            }
			...
        }
        ...
        
        try
        {
            // 通过反射调用方法，获得返回值
            Object obj = method.invoke(o, params);

            if (obj == null)
            {
                if( method.getReturnType() == Void.TYPE)
                {
                    return "";
                }
            }

            return obj;
        }
        ...
    }
}
```

ASTMethod的主要执行过程：首先从IntrospectionCache中获取方法，如果没有缓存过，则通过Uberspect反射出方法，并执行方法返回结果。
最终每个节点返回的结果会拼接成文本渲染输出。

### 6.小结
本文通过源码阅读、样例分析的方式，介绍了velocity如何使用Java提供的JavaCC、JJTree和自省工具达到模板渲染的目的。其中使用的工具和方法希望对读者在其他源码分析过程中有所帮助。

Java为了能够实现动态语言原生就支持的特性比如：反射、闭包等，需要做很多事情，弄的很复杂的样子；在某些场景下，更适合用动态语言来弥补Java的短板，比如模板、DSL等都可以通过在Java中方便的集成Groovy等脚本语言来支持。








