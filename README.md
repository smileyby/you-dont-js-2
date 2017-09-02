第二章 词法作用域
===============

在第1章中，我们将“作用域”定义为一套规则，这套规则用来管理引擎如何在当前作用域以及嵌套的子作用于中根据实际标识符名称进行变量查找。

作用域共有两种主要的工作模式。第一种是最为普遍的，被大多数编程语言所采用的此法作用域，我们会对这种作用域进行深入讨论。另一种叫做动态作用域，仍有一些编程语言在使用（比如Bash脚本，Perl中的一些模式等）。

## 2.1

第1章介绍过，大部分标准语言编辑器的第一个工作阶段叫做词法化（也叫单词化）。回忆一下，词法化的过程对源代码中的字符进行检查，如果是有状态的解析过程，还会赋予单词语义。

这个概念是理解词法作用域及其名称的来历基础。

简单地说，词法作用域就是定义在词法阶段的作用域。换句话说，词法作用域是由你在写代码时将变量和块作用域写在哪里来决定的，因此当词法分析器处理代码会保持作用域不变（大部分情况下是这样）。

**后面会介绍一些欺骗词法作用域的方法，这些方法在词法分析器处理过后依然可以携该作用域，但是这种机制可能会有点难以理解。事实上，让词法作用域更具词法关系保持书写时的自然关系不变，是一个非常好的最佳实践。**

考虑一下代码：

```js

function foo(a) {
	var b = a * 2;
	function bar(c) {
		console.log( a, b, c );
	}
	
	bar( b * 3 );
}

foo( 2 ); // 2, 4, 12

```

在这个例子中有三个逐级嵌套的作用域。为了帮助理解，可以将他们想象成几个逐级包含的气泡。

![](http://oqpmmru7y.bkt.clouddn.com/qiantaozuoyongyu.png)

1. 包含着整个全局作用域，其中只有一个标识符：foo
2. 包含着foo所创建的作用域，其中有三个标识符：a、bar和b
3. 包含着bar所创建的作用域，其中只有一个标识符：c

作用域气泡由其对应的作用域块代码写在这里决定，他们是逐级包含的。下一章会讨论不同类型的作用域，但现在是要假设每一个函数都会创建一个新的作用域气泡就好了。

bar的气泡被完全包含在foo所创建的气泡中，唯一的原因是哪里就是我们希望定义的函数bar的位置。

注意，这里所说的气泡是严格包含的。我们并不是在讨论文氏图，这种可以跨越边界的气泡。换句话说，没有任何函数的气泡可以（部分地）同事出现在连个外部作用域气泡中，就如同没有任何函数可以部分地同事出现在两个父级函数中一样。

### 查找

作用域气泡的结构和互相之间的位置关系给引擎提供了足够的位置信息，引擎用这些信息来查找标识符位置。

在上一个代码片段中，引擎执行的console.log(...)声明，并查找a、b和c三个变量的引用。它首先从最内部的作用域，也就是bar(...)函数的作用域气泡开始查找。引擎无法找到a，因此会去上一级到所嵌套的foo(...)的作用域中继续查找。在这里找到了a，因此引擎使用了这个引用。对b来说也是一样。而对c来说，引擎在bar(...)中就找到了它。

如果a，c都存在于bar(...)和foo(...)的内部，console.log(...)就可以直接使用bar(...)中的变量，而无需到外面的foo(...)中查找。

作用域差债会在找到第一个匹配的标识符时停止。在多层的嵌套作用域中可以定义同名的标识符，这叫做“遮蔽效应”（内部的标识符“遮蔽”了外部的标识符）。抛开遮蔽效应，作用域查找始终从运行时所处的最内部作用域开始，逐级向外或者向上进行，知道意见第一个匹配的标识符为止。

**全局变量会自动生成为全局对象（比如浏览器中的window对象）的属性，因此可以不直接通过全局对象的词法名称，而是间接地通过对全局对象属性的引用来对其进行访问。`window.a` 通过这种技术可以访问那些被同名变量所遮蔽的全局变量。但非全局变量如果被遮蔽了，无论如何都无法被访问到。**

无论函数在哪里被调用到，也无论它如何被调用，它的词法作用域都只由函数被声明时所处的位置决定。

词法作用域查找只会查找一级标识符，比如a，b，和c。如果代码中引用了foo.bar.baz，词法作用域只会试图查找foo标识符，找到这个变量后，对象属性访问规则会分别接管bar和baz属性的访问。

## 2.2欺骗词法

如果词法作用域完全由写代码期间函数所声明的位置来定义，怎样才能在运行时来“修改”（也就是说欺骗）词法作用域呢？

JavaScript中有两种机制来实现这个目的。社区普遍认为在代码中使用这两种机制并不是什么好注意。但是关于他们的争论通常会忽略掉最重要的点：欺骗词法作用域会导致性能下降。

在详细解释性能问题之前，先来看看这两种机制分别是什么原理。

### 2.2.1 eval

JavaScript中的eval(...)函数可以接受一个字符串为参数，并将其中的内容视为好像在书写时就存在于程序中这个位置的代码。换句话说，可以在你写程序的代码中用程序生成代码运行，就好像代码是写在那个位置的一样。

根据这个原理来理解eval(...),它是如何通过代码欺骗假装成书写时（也就是词法期）代码就在那，来实现修改词法作用域环境的，这个原理就变得清晰易懂了。

在执行eval(...)之后的代码时，引擎并不“知道”或“在意”前面的代码是以动态形式插入进来，并对词法作用域的环境进行修改的。引擎只会如往常进行词法作用域查找。

考虑一下代码：

```js

function foo(str, a) {
	eval( str ); // 欺骗！
	console.log( a, b );
}

var b = 2;

foo( "var b = 3;", 1 ); // 1, 3

```

eval(...)调用中的"var b = 3;"这段代码会被当作本来就在哪里一样来处理。由于那段代码生命了一个新的变量b，因此它对已经存在的foo(...)的词法作用域进行了修改。事实上，和前面提到的原理一样，这段代码实际上在foo(...)内部创建了一个变量b，并遮蔽了外部（全局变量）作用域中的同名变量。

当console.log(...)被执行时，会在foo(...)的内部同时找到a和b，但永远无法找到外部的b。因此会输出“1,3”而不是正常情况下会输出的“1,2”。

**在这个例子中，为了展示的方便和简洁，我们传递进去的“代码”字符串是固定不变的。而实际情况中，可以非常容易地根据程序逻辑动态地将字符拼接在一起之后再传递进去。eval(...)通常被用来执行动态创建的代码，因为像例子中这样动态地执行一段固定字符所组成的代码，并没有比直接将代码写在哪里更有好处。**

默认情况下，如果eval(...)中所执行的代码包含有一个或多个声明（无论是变量还是函数），就会对eval(...)所处的词法作用域进行修改。技术上，通过一些技巧（已经超出我们的讨论范围）可以间接调用eval(...)来使其运行在全局作用域中，并对全局作用域进行修改。但无论何种情况，eval(...)都可以在运行期书写期的词法作用域。

**在严格模式下的程序，eval(...)在运行时有其自己的词法作用域，意味着其中的声明无法修改所在的作用域。**

```js

function foo(str) {
	"use strict";
	eval( str );
	console.log( a ); // ReferenceError: a is not defined
}

foo( "var a = 2" );

```

JavaScript中还有其他一些功能效果和eval(...)很相似。setTimeOut(...)和setInterval(...)的第一个参数可以是字符串，字符串的内容可以被解释为一段动态生成的函数代码。这些功能已经过时且并不被提倡。不要使用它们！

new Function(...)函数的行为也很类似，最后一个参数可以接受代码字符串，并将其转化为动态生成的函数（前面的参数是这个新生成的函数的形参）。这种构建函数的语法比eval(...)略微安全一些，但也要尽量避免使用。

在程序中动态生成的代码的使用场景非常罕见，因此它所带来的好处无法抵消性能上的损失。

### 2.2.2 with

JavaScript中另一个难以掌握（并且现在也不推荐使用）的用来欺骗词法作用域的功能是with关键字。可以有很多方法来解释with，在这里我们选择从这个角度来解释他：它如何同被它所影响的词法作用域进行交互。

with通常被当做重复引用同一个对象中的多个属性的快捷方式，可以不需要重复引用对象本身。

比如：

```js

var obj = {
	a: 1,
	b: 2,
	c: 3
};

// 单调乏味的重复"obj"
obj.a = 2;
obj.b = 3;
obj.c = 4;

// 简单的快捷方式
with (obj) {
	a = 3;
	b = 4;
	c = 5;
}

```

但实际上这不仅仅是为了方便地访问对象属性。考虑如下代码：

```js

function foo(obj) {
	with (obj) {
		a = 2;
	}
}

var o1 = {
	a: 3
};

var o2 = {
	b: 3
};

foo( o1 );
console.log( o1.a ); // 2

foo( o2 );
console.log(o2.a); // undefined
console.log( a ); // 2---不好，a泄露到全局作用域上了！

```

这个例子中创建了o1和o2两个对象。其中一个具有a属性，另外一个没有。foo(...)函数接受一个obj参数，该参数是一个对象引用，并对这个对象引用执行了with(obj){...}。在with内部，我们写的代码看起来只是对变量a进行简单的词法引用，实际上就是一个LHS引用，并将2赋值给它。

在我们将o1传递进去，a=2赋值操作找到了o1.a并将2赋值给它，这在后面的console.log(o1.a)中可以提现。而当o2传递进去，o2并没有a属性，因此不会创建这个属性，o2.a保持undefined。

但是可以注意到一个奇怪的副作用，实际上a=2赋值操作创建了一个全局的变量a。这是怎么回事？

with可以将一个没有会有多个属性的对象处理为一个完全隔离的词法作用域，因此这个对象的属性也会被创建为定义在这组用于中的词法标识符。

**尽管with块可以将一个对象处理为词法作用域，但是这个块内部正常的var声明并不会被限制在这个块的作用域中，而是被添加到with所处的函数作用域中。**

eval(...)函数如果接受了含有一个或多个声明的代码，就会修改其所处的词法作用域，而with声明实际上是更具你床底给它的对象凭空创建了一个全新的词法作用域。

可以这样理解，当我们传递给o1给with，with所声明的作用域o1，而这个作用域中包含一个同o1.a属性相符的标识符。但当我们将o2作为作用域时，其中没有a标识符，因此进行了正常的LHS标识符查找。

o2的作用域、foo(...)的作用域和全局作用域都没有找到标识符a，因此当a=2执行时，自动创建了一个全局变量（因为是非严格模式）。

with这种将对象及其属性放进一个作用域并同时分配给标识符的行为很让人费解。但为了说明我们所看到的现象，这是我能给出的最直白的解释了。

**另外一个不推荐使用eval(...)和with的原因是会被严格模式所影响（限制）。with被完全禁止，而在保留核心功能的前提下，间接或非安全地使用eval(...)也被禁止。**

### 2.2.3 性能

eval(...)和with会在运行时修改或创建新的作用域，以此来欺骗其他在书写是定义的词法作用域。

你可能会问，那又怎样呢？如果他们能够实现更复杂的功能，并且代码更具扩展性，难道不是非常好的功能吗？答案是否定的。

JavaScript引擎会在便器阶段进行数项的性能优化。其中有些优化依赖于能够更具代码词法进行静态分析，并预先确定所有变量和函数的定义位置，才能在执行过程中快速找到标识符。

但如果引擎在代码中发现了eval(...)或with，它只能简单地假设关于标识符位置的判断都是无效的，因为无法在词法分析阶段明确知道eval(...)会接收到什么代码，这些代码会如何对作用域进行修改，也无法知道传递给with用来创建新词法作用域的对象的内容到底是什么。

最悲观的情况是如果出现了eval(...)或with，所有的优化可能都是无意义的，因此做简单的做法就是完全不做任何优化。

如果代码中大量使用eval(...)或with,那么运行起来一定变得非常慢。无论引擎多聪明，试图讲这些悲观情况的副作用限制在最小范围内，也无法避免如果没有这些优化，代码运行得更慢这个事实。

## 2.3小结

词法作用域意味着作用域是由书写代码时函数声明的位置来决定的。编译的词法分析阶段基本能够知道全部标识符在哪里以及是如何声明的，从而能够预测在执行过程中如何对他们进行查找。

JavaScript中的两个机制可以“欺骗”词法作用域：eval(...)和with。前者可以对一段包含一个或多个声明的“代码”字符串进行盐酸，并借此来修改已经存在的词法作用域（在运行时）。后者本质上是通过将一个对象的引用当作作用域来处理，将对象的属性当作作用域的标识符来处理，从而创建一个新的词法作用域（同样是在运行时）。

这两个机制的副作用是引擎在编译时对作用域查找进行优化，因为引擎只能谨慎地认为这样的优化是无效的。使用着其中任何一个机制都将导致代码运行变慢。不要使用它们。





