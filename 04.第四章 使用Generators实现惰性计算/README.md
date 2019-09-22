## 使用Generators实现惰性计算

惰性计算是一种编程技术，它用于当我们希望需要使用值的时候才去计算的场景。这样，可以确保我们确实需要它。

相反的，直接都去计算，有可能计算了我们不需要的值。这通常没什么问题，但当我们的应用程序的大小和复杂性

增长到一定水平，这些计算造成的浪费就难以想象了。


Generator是一种新的原生类型引入到JavaScript中并作为ES6语言规格的一部分。Generator帮助我们在代码中

实现惰性计算技术，进一步说，帮助我们实现存储并发原则。


我们将通过对Generator的一些简单介绍来开始本章，让我们对它们的表现方式有一定了解。之后，我们将进入

更高级的惰性计算场景，并通过协程结束本章。现在让我们开始吧。


### 调用堆栈和内存分配

内存分配是任何编程语言都必不可少的。如果没有它，我们就没有所谓的数据结构，甚至没有原生类型。现在

内存虽然很便宜，一般都有足够的内存可供使用; 但这并不值得高兴。虽然今天在内存中分配更大的数据结构

更加可行，但是在10年前，当我们编程时，我们仍然必须释放分配内存。JavaScript是一种垃圾自动收集语言，

这意味着我们的代码不必显式地销毁内存中的对象。但是，垃圾收集器会导致CPU损耗。


所以这里有两个因素在起作用。我们想在这里保存两个资源，我们将尝试使用生成器来实现惰性计算。我们

不必要多余的分配内存，如果我们能避免这一点，那么我们就可以避开频繁的调用垃圾收集器。在本节中，

我将介绍一些Generator生成器概念。


#### 标记函数上下文

在一个正常的函数调用栈，一个函数返回一个值。在return语句激活一个新的执行上下文和丢弃旧的上下文，

因为返回就代表已处理完毕了。生成器函数是一个特殊的JavaScript函数语法类型，和return语句相比

他们的调用栈不那么老套。这里有张图表示了生成器函数的调用，并在开始生成值时发生的事情：

![image076.gif](../images/image076.gif)


正如return语句将值传递给调用上下文一样，yield语句也会返回一个值。但是，与普通函数不同的是，

生成器函数上下文不会被丢弃。事实上，它们被加上标记，以便在将控制权交还给生成器上下文时，它可以

从中断处继续执行获取值，直到完成为止。这个标记数据非常容易，因为它只是指向我们代码中的位置。


#### 序列而不是数组

在JavaScript中，当我们需要遍历事物，数字、字符串、对象等列表时，我们会使用数组。数组是通用的，

功能也是强大的。在惰性计算的上下文中，数组的挑战是数组本身就是数据需要分配。所以我们的数组需要

在内存中的某个位置分配元素，并且还有关于数组中元素的元数据。


如果我们在使用大数据量的对象，则与阵列相关的内存开销就很大。另外，我们需要以某种方式将这些对象

放在数组中。这是额外的步骤会增加CPU时间。另一种概念是序列。序列不是有形的JavaScript语言结构。

它们是一个抽象的概念 - 数组但没有实际分配数组。序列有助于惰性计算。由于这个原因，没有什么需要

分配内存，并且没有初始入口步骤。这是迭代数组所涉及步骤的示图：

![image077.gif](../images/image077.gif)


我们可以看到，在我们迭代这三个对象之前，我们首先必须分配一个数组，然后用这些对象填充它。让我们将这种

方法与序列的概念思想进行对比，如下图所示：

![image078.gif](../images/image078.gif)


对于序列，我们没有为我们感兴趣的迭代的对象提供明确的容器结构。与序列关联的唯一开销是指向当前项的指针。

我们可以使用生成器函数作为在JavaScript中生成序列的机制。正如我们在上一节中看到的那样，生成器在将值

返回给调用者时将其执行上下文加上标记。这是我们目前需要的最小开销。它使我们能够惰性地计算对象并将

它们作为序列进行迭代。


### 创建生成器并生成值

在本节中，将介绍生成器函数语法，并将逐步介绍生成器的值。我们还将研究可以用来迭代生成器生成

值的两种方法。


#### 生成器函数语法

生成器函数的语法几乎与普通函数相同。不同之处在于function关键字的声明后面跟一个星号。更重要的

区别是返回值，它总是返回一个生成器实例。此外，尽管创建了新对象，但不需要new关键字。下面让我们

来看看生成器函数是怎样的：

```javascript
//生成器函数使用星号来表示返回生成器实例。
//我们可以从生成器返回值，
//然而不是调用者获得该值，
//他们将永远获取生成器实例。
function* gen() {
	return 'hello world';
}

//创建生成器实例。
var generator = gen();

//让我们看看它是什么样的。
console.log('generator', generator);
//→generator Generator

//这是我们获得返回值的方式。看起来很尴尬，
//因为我们永远不会使用生成器函数只返回一个值。
console.log('return, generator.next().value);
//→return hello world
```

我们不太可能以这种方式使用生成器，但它是说明生成器函数细微差别的好方法。例如，return语句在生成器

函数中是完全有效的，然而，正如我们所看到的，它们为调用者产生了完全不同的结果。在实践中，我们更有

可能在生成器中遇到yield语句，所以让我们接下来看看它们。


#### 生成值

生成器函数的常见情况是产生值并控制返回调用者。将控制权交还给调用者是生成器的一个定义特征。

当我们生成值时，生成器会在代码中标记我们的位置。这样做是因为调用者可能会从生成器请求另一个值，

而当它发生时，生成器只是从它停止的地方开始。让我们来看一下产生几次值的生成器函数：

```javascript
//此函数按顺序生成值。
//没有容器结构，就像一个数组。
//相反，每一次调用yield语句，
//控制权交回到调用者，以及函数中的位置加上标记。
function* gen() {
	yield 'first';
	yield 'second';
	yield 'third';
}

var generator = gen();

//每次调用“next()”时，控制权都会被传回到生成器函数的执行上下文。
//然后，生成器查找标记以查找它最近产生值的位置
console.log(generator.next().value);
console.log(generator.next().value);
console.log(generator.next().value);
```

前面的代码才是序列真正的样子。我们有三个值，它们是从我们的函数中顺序产生的。它们也没有放入任何

类型的容器结构中。第一个调用`yield`传递`first`到`next()`，在它被应用的地方。其他两个值也是如此。

事实上，行为上是惰性计算的。我们有三次调用`console.log()`。`gen()`的实现将返回一组值供我们

记录。相反，当我们需要记录一个值时，我们会从生成器中获取它。这是惰性因素; 我们保留我们的能力，

直到他们真正需要，避免分配和计算。


我们之前的示例不太理想之处是我们正在重复调用`console.log()`，实际上，我们想迭代序列，为其中的

每项调用`console.log()`。让我们现在迭代一些生成器序列。


#### 迭代生成器

next()方法对于我们，已不奇怪了，生成器序列接下来的返回值。它实际返回的值由两个属性构成：生成值和

是否生成器结束。但是，我们一般不想硬编码调用next()。取而代之的是，我们想调用它反复的从生成器生成值。

下面是一个例子使用while循环，来循环遍历一个生成器：

```javascript
//基本的生成器函数产生序列值。
function* gen(){
	yield 'first';
	yield 'second';
	yield 'third';
}

//创建生成器。
var generator = gen();

//循环直到序列结束。
while(true) {
	//获取序列中的下一项。
	let item = generator.next();
	
	//有下一个值，还是结束了？
	if(item.done) {
		break;
	}

	console.log('while', item.value);
}
```

此循环将一直持续，直到yield项的done属性为true;在这一点上，我们知道没有任何东西，可以停止它。

这让我们遍历生成值的序列，而无需创建一个数组然后去迭代它。然而，在这个循环中有些重复代码，它们更多的

是在管理生成器迭代而不是实际迭代它。我们来看看另一种方法：

```javascript
//“for..of”循环消除了需要显式的调用生成器构造，
//如“next()”，“value”，“done”。
for (let item of generator) {
	console.log('for..of', item);
}
```

现在要好得多。我们将代码缩减后更加专注于手头任务。除了for..of语句之外，这段代码基本上与我们的

while循环完全相同，它知道iterable是生成器时要做什么。迭代生成器是并发JavaScript应用程序中的

常见模式，因此在这里优化代码和提升可读性将是明智的决定。


### 无限序列

一些序列是无限的，素数，斐波纳契数，奇数，等等。无限序列不限于数字组合;更抽象的概念可以被认为

是无限的。例如，一组无限重复的字符串，一个无限切换的布尔值，依此类推。在本节中，我们将探讨生成

器如何使我们能够使用无限序列。


#### 没有尽头

从内存消耗的角度来看，从无限序列中分配项是不实际的。事实上，甚至不可能分配整个序列 - 它是无限的。

内存是有限的。因此，最好是简单地完全回避整个分配问题，并使用生成器根据需要从序列中产生值。在任何

给定的时间点，我们的应用程序只会使用无限序列的一小部分。以下是无限序列中使用的内容与这些序列的

潜在大小的示意图：

![image079.gif](../images/image079.gif)


我们可以看到，在这个序列中有大量的项我们永远不会使用。让我们看看一些惰性地从无限斐波纳契数列中

产生项的生成器代码：

```javascript
//生成无限的Fibonacci序列。
function* fib() {
	var seq = [0, 1],
		next;

	//这个循环实际上并没有无限运行,
	//只当使用“next()”请求序列中的项时。
	while (true) {
		//产生序列中的下一个项。
		yield (next = seq[0] + seq[1]);
		//存储所需的状态，
		//以便计算下一次迭代中的项。
		seq[0] = seq[1];
		seq[1] = next;
	}
}

//启动生成器。这永远不会“done”生成值。
//然而，它是惰性的 - 它只是在我们需要的时候生成值。
var generator = fib();

//获取序列的前5项。
for (let i = 0; i < 5; i++) {
	console.log('item', generator.next().value);
}
```

#### 交替序列

无限序列的变化是循环序列或交替序列。到达终点时，这些类型的序列是圆形的; 他们从起点来开始。以下是两个值

之间交替的序列：

![image080.gif](../images/image080.gif)


这种类型的序列将继续无限地生成值。当我们有一组规则来确定序列的定义方式和生成的项集合时，这就变得很有用了;

然后，我们重新开始这一系列。现在，让我们看一些代码，看看如何使用生成器实现这些序列。这是一个通用的生成器函数，

我们可以用来在值之间进行交替：

```javascript
//一个通用生成器将无限迭代
//提供的参数，产生每个项。
function* alternate(...seq) {
	while (true) {
		for (let item of seq) {
			yield item;
		}
	}
}
```

这是我们第一次声明一个接受参数的生成器函数。实际上，我们使用spread运算符来迭代传递给函数的参数。与参数

不同，我们使用spread运算符创建的seq参数是一个真实数组。当我们遍历这个数组时，我们从生成器中生成每个项。

这乍一看起来似乎并不那么有用，但是这里的while循环起了真正的作用。由于while循环永远不会退出，for循环

将自己重复。也就是说，它会交替出现。这否定了明确的需要标记代码（我们到达了序列的末尾吗？我们如何重置

计数器并回到开头？等等）让我们看看这个生成器函数是如何工作的：

```javascript
//通过提供的参数，创建一个交替的生成器。
var alternator = alternate(true, false); 

console.log('true/false', alternator.next().value);
console.log('true/false', alternator.next().value);
console.log('true/false', alternator.next().value); 
console.log('true/false', alternator.next().value);
//→
// true/false true
// true/false false
// true/false true
// true/false false
```

酷。因此，只要我们继续获取值，alternator将继续生成true/false值。这里的主要好处是我们不需要知道关于

下一个值，alternator为我们负责完成。让我们看看这个用不同的序列迭代的生成器函数：

```javascript
//使用新值创建新的生成器实例
//来迭代每个项。
alternator = alternator('one', 'two', 'three');

//从无限序列中获取前10个项。
for (let i = 0; i < 10; i++) {
	console.log('one/two/three', `"${alternator.next().value}"`);
}

//→
//one/two/three "one"
//one/two/three "two"
//one/two/three "three"
//one/two/three "one"
//one/two/three "two"
//one/two/three "three"
//one/two/three "one"
//one/two/three "two"
//one/two/three "three"
//one/two/three "one"
```

正如我们所看到的，alternate()函数在传递给它的任何参数之间交替生成项。


### 传递到其他生成器

我们已经看到了yield语句如何能够暂停一个生成器函数执行上下文，并生成一个值返回到当前调用上下文。

在yield语句上有一个变化，它允许我们传递到其他generator函数。另一种技术涉及到创建一个组合生成器，它由

几个生成器交织在一起。在本节中，我们将探讨这些方法。


#### 选择一个策略

传递到其他生成器使我们的函数能够在运行时决定将控制从一个生成器切换到另一个生成器。换句话说，它允许

基于策略选择更合适的生成器函数。这有一张图表示一个生成器函数，决定并传递到其他某个生成器函数：

![image081.gif](../images/image081.gif)


我们在整个应用程序会使用这里的三个专用生成器。也就是说，他们每一个都有自己独有的方式。也许，他们有

自己特定类型的输入。然而，这些生成器只是对它们给出的输入做出假设。它可能不是在用最好的工具在执行任务，

所以，我们必须要弄清楚其中的这些生成器再使用。我们希望避免在所有的地方执行这些决策选择的代码。如果

我们能够封装所有这些成为一个通用的生成器，能处理通常的一些情况，这将会很不错。


假设我们有以下生成器函数，它们同样适用在我们的应用程序中：

```javascript
//映射对象集合到特定的属性名称的生成器。
function* iteratePropertyValues(collection, property) {
	for (let object of collection) {
		yield object[property];
	}
}

//生成给定对象的每个值的生成器。
function* iterateObjectValues(collection) {
	for (let key of Object.keys(collection)) {
		yield collection[key];
	}
}

//生成给定数组中每个项的生成器。
function* iterateArrayElements(collection) {
	for (let element of collection) {
		yield element;
	}
}
```

这些函数简洁小巧，易于使用。麻烦的是这些函数中的每一个都会对传入的集合做出判断。它是一个对象

数组，每个对象都有一个特定的属性吗？它是一个字符串数组？它是一个对象而不是一个数组？由于这些

生成器函数在我们的代码中通常用于类似的目的，我们可以实现一个更通用的迭代器，它的工作是确定要

使用的最适合的生成器函数，然后再用它。让我们看看这个函数是什么样的：

```javascript
//这个生成器传递到其他生成器。
//但首先，它执行一些逻辑来确定最好的生成器函数。
function* iterateNames(collection) {
	//我们正在处理数组吗？
	if (Array.isArray(collection)) {
		
		//这是一个启发式，我们检查第一个
		//数组的元素。基于此，我们
		//对剩余元素做出假设。
		let first = collection[0];

		//这是我们推崇其他更专业的生成器，
		//基于我们从第一个数组元素发现的内容
		if (first.hasOwnProperty('name')) {
			yield* iteratePropertyValues(collection, 'name');
		} else if(first.hasOwnProperty('customerName')) {
			yield* iteratePropertyValues(collection, 'customerName');
		} else {
			yield* iterateArrayElements(collection);
		}
	} else {
		yield* iterateObjectValues(collection);
	}
}
```

可以将iterateNames()函数看作其他三个生成器中的任何一个的简单代理。它根据输入，并在一个集合上做出

选择。我们本可以实现一个大型生成器函数，但这将使我们无法直接使用想要使用较小生成器的用例。如果我们

想用它们来组合新功能特性怎么办？或者另一个复合生成器需要用吗？保持生成器功能小而专注是一个好主意。

该yield* 语法允许我们将控制权移交给更合适的生成器。


现在，让我们看看这个通用生成器函数如何通过传递到最适合处理数据的生成器来使用：

```javascript
var colection;

//迭代一串字符串名称。
collection = ['First', 'Second', 'Third'];

for (let iterateNames(collection)) {
	console.log('array element', `“${name}”`);
}

//迭代一个对象，其中使用值
//来命名的 - 这里的键不相关。
collection = {
	first: 'First',
	second: 'Second',
	third: 'Third'
};

for (let name of iterateNames(collection)) {
	console.log('object value', `"${name}"`);
}

//在集合中迭代每个对象的“name”属性。
collection = [
	{name: 'First'},
	{name: 'Second'},
	{name: 'Third'}
];

for (let name of iterateNames(collection) {
	console.log('property value', `"${name}"`);
}
```


#### 交错生成器

当生成器传递到另一个生成器时，控制器不会返回第一个生成器，直到第二个生成器全部完成。在前面的例子中，

我们的生成器只是寻找一个更好的生成器来完成工作。但是，有时我们会有两个或更多数据源需要一起使用。

因此，而不是将控制权交给一个生成器，然后传递到另一个等等，我们会在各种来源之间交替，轮流处理数据。


这是一个示图，说明了交错多个数据源以创建单个数据源的生成器的方法：

![image082.gif](../images/image082.gif)


我们的方法是循环数据源，而不是清空一个源，然后清空另一个源，依此类推。这样的生成器将要处理的，并不是一个

大型集合，而是两个或更多集合。使用这种生成器技术，我们实际上可以将多个数据源视为一个大数据源，但无需为

大型结构分配内存。我们来看下面的代码示例：

```javascript
'use strict';

//将输入数组转换为生成每个值的生成器的实用程序函数。
//如果它不是数组，假定它已经是一个生成器并且传递给它。
function* toGen(array) {
	if (Array.isArray(array)) {
		for (let item of array) {
			yield item;
		}
	} else {
		yield* array;
	}
}

//交错给定的数据源（数组或生成器）到一个生成器源。
function* weave(...sources) {
	//这控制“while”循环。
	//只要有一个产生数据的来源，
	//while循环仍然有效。
	var yielding = true;

	//我们必须确保每一个sources是一个生成器。
	var generators = sources.map(source => toGen(source));
		
	//启动主交错循环。它就是这样通过每个来源，
	//从每个源产生一个项，然后重新开始，
	//直到每一个来源是空的。
	while(yield) {
		yielding = false;
		for (let origin of generator) {
			let next = source.next();
			
			//只要我们产生数据，“yield”值就是true，
			//而且“while”循环继续。
			//当每个来源“done”都是true，
			//“yielding”变量保持为false，
			//那么“while循环退出.
			if (!next.done) {
				yielding = true; 
				yiedl next.value;
			}
		}
	}
}

//一个通过迭代给定的源生成值的基本过滤器，
//并且产生项未被禁用。
function* enabled(source) {
	for (let item of source) {
		if (!item.disabled) {
			yield item;
		}
	}
}

//这些是我们要交错的两个数据源传入一个生成器，
//然后可以由另一个生成器过滤。
var enrolled = [
	{name: 'First'},
	{name: 'Sencond'},
	{name: 'Third', disabled: true}
];

var pending = [
	{name: 'Fourth'},
	{name: 'Fifth'},
	{name: 'Sixth', disabled: true}
];

//创建生成器，从两个数据源生成用户对象。
var users = enabled(weave(registered, pending));

//实际上执行交错和过滤。
for (let user of users) {
	console.log('name', `"${user.name}"`);
}
```

### 将数据传递给生成器

yield语句不只是放弃控制权返回给调用者，它也返回一个值。该值通过next()方法传递给生成器函数。这就是

我们在创建数据后将数据传递给生成器的方法。在本节中，我们将讨论生成器的两面性，以及创建反馈循环如何

能产生一些精益代码。


#### 复用生成器

有些生成器是通用的，在我们的代码中经常使用。在这种情况下，不断创建和销毁这些生成器实例是否有意义？或者

我们可以复用它们吗？例如，考虑一个主要依赖于初始条件的序列。假设我们想生成一个偶数序列。我们将从2开始，

当我们迭代这个生成器时，该值将递增。下次我们要迭代偶数时，我们必须创建一个新的生成器。


这有点浪费，因为我们所做的只是重置计数器。如果我们采用不同的方法，允许我们继续为这些类型的序列使用相同

的生成器实例，该怎么办？生成器的next()方法是此功能的可能实现方式。我们可以传递一个值，然后重置我们

的计数器。因此，每次我们需要迭代偶数时，不必创建新的生成器实例，我们可以简单地调用next()，传入的值作为

重置生成器的初始条件。


yield关键字实际上会返回一个值 - 传递到next()的参数。大多数情况下，这是未定义的，例如当生成器

在for..of循环中迭代时。然而，这就是我们在开始运行后能够将参数传递给生成器的方法。这与将参数传递给

生成器函数不同，这对于执行生成器的初始配置非常方便。传递给next()的值是当我们需要为要生成的下一个值

更改某些内容时，我们如何与生成器通信。


让我们看一下如何使用next()方法创建可重用的偶数序列生成器：

```javascript
//这个生成器将不断生成偶数。
function* genEvens() {
	
	//初始值为2.但这可以通过在传递给“next()”的input进行改变
	var value = 2,
		input;
		
	while (true) {
		//我们产生值，并获得input值。如果
		//提供input值，这将作为下一个值
		input = yield value;
		
		if (input) {
			value = input;
		} else {
			//确保下一个值是偶数。
			//处理奇数值时的情况传递给“next()”。
			value += value % 2 ? 1 : 2;
		}
	}
}

//创建“evens”生成器。
var evens = genEvens(),
	even;

//迭代偶数达到10。
while ((even = evens.next().value) <= 10) {
	console.log('even', even);
}

//→
// even 2
// even 4
// even 6
// even 8
// even 10

//重置生成器。我们不需要创建一个新的。
evens.next(999);

//在1000 - 1024之间迭代even值。
while ((even = evens.next().value) <= 1024) {
	console.log('evens from 1000', even);
}

//→
//evens from 1000 1002
//evens from 1000 1004
//evens from 1000 1006
//evens from 1000 1008
//evens from 1000 1010
//evens from 1000 1012
//evens from 1000 1014
```

> 如果你想知道为什么我们没有使用for..of循环来支持while循环，那是因为你使用for..of循环迭代生成器
> 执行此操作时，只要循环退出，生成器就会标记为已完成。因此，它将不再可用。


#### 轻量级map/reduce

我们可以用next()方法做的其他事情是将一个值映射到另一个值。例如，假设我们有一个包含七个项的集合。

要映射这些项，我们将迭代集合，将每个项传递给next()。正如我们在上一节中所见，此方法可以重置生成

器的状态，但它也可以用于提供输入数据流，就像它提供输出数据流一样。


让我们看看我们是否可以通过next()将它们传入生成器来编写一些执行此映射集合项的代码：

```javascript
//，这个生成器只要调用“next()”，将继续迭代。
//这也是期待的结果，以便它可以调用
//“iteratee()”函数就可以生成结果
function* genMapNext(iteratee) {
	var input = yield null;
	while (true) {
		input = yield iteratee(input);
	}
}

//我们想要映射的数组。
var array = ['a', 'b', 'c', 'b', 'a'];

//一个“mapper”生成器。我们传递一个iteratee函数，
//作为“genMapNext()”的参数。
var mapper = genMapNext(x => x.toUpperCase());

//我们迭代的起点 
var reduced = {};

//我们必须调用“next()”来开始生成器。
mapper.next();

//现在我们可以开始迭代数组了。
//“mapped”值来自生成器。
//我们想要映射的值通过将其传递给“next()”进入生成器。
for (let item of array) {
	let mapped = mapper.next(item).value;
	
	//我们的简化逻辑采用映射值，
	//并将其添加到“reduced”对象中，
	//计算重复键的数量。
	if (reduced.hasOwnProperty(mapped)) {
		reduced[mapped]++;
	} else {
		reduced[mapped] = 1;
	}
}

console.log('reduced', reduced);
//→reduced {A: 2, B: 2, C: 1}
```

我们可以看到，这确实是可能的。我们能够使用这种方法执行轻量级的map/reduce任务。映射生成器具有

iteratee函数，该函数应用于集合中的每一项。当我们遍历数组时，我们可以通过将这些项传递给next()

方法来将这些项提供给生成器作为一个参数。


但是，有一些关于前一种方法的东西感觉并不是最好 - 必须像这样启动生成器，并且为每次迭代显式调用

next()都会感觉很笨拙。实际上，我们不能直接应用iteratee函数，而是非得调用next()吗？在使用生成器时，

我们需要注意这些事情;特别是在将数据传递给生成器时。仅仅因为我们能够实现，并不意味着这是一个好主意。


如果我们像对待所有其他生成器一样简单地迭代生成器，mapping和reducing可能会感觉更自然。我们仍然

希望生成器为我们提供的轻量级映射，以避免内存分配。让我们尝试一种不同的方法 - 一种不需要next()的方法：

```javascript
//这个生成器是一个比“genMapNext()”更有用的映射器
//因为它不依赖于值通过“next()”进入生成器。

//相反，这个生成器接受一个iterable，
//和一个iteratee函数。iterable是iterated-over，
//以及iteratee的结果是可以生成的。
function* genMap(iterable, iteratee) {
	for (let item of iterable) {
		yield iteratee(item);
	}
}

//使用iterable的数据源创建我们的“mapped”生成器和iteratee函数。
var mapped = genMap(array, x => x.toUpperCase());
var reduced = {};

//现在我们可以简单地迭代我们的生成器而不是调用“next()”。
//每个循环迭代的工作都是执行reduction逻辑，而不是调用“next()”。
for (let item of mapped) {
	if (reduced.hasOwnProperty(item)) {
		reduced[item]++;
	} else {
		reduced[item] = 1;
	}
}

console.log('reduced', reduced);
//→reduced improved {A: 2, B: 2, C: 1}
```

这看起来像是一种改进。代码更少，生成器的流程更容易理解。不同之处在于我们将数组和iteratee函数预先

传递给生成器。然后，当我们遍历生成器时，每个项都会被惰性地映射。将此数组缩减为对象的代码也更易于

阅读。


我们刚刚实现的这个genMap()函数是通用的，他对我们很有用。在实际应用中，映射比大写转换更复杂。

更有可能的是，将有多个级别的映射。也就是说，我们映射的集合，映射它N多次。如果我们能对我们

的代码做一个良好的设计，然后，我们要以较小的迭代功能来组合生成器。


但是我们怎样才能保持这种通用和惰性呢？方法是使用几个生成器，每个生成器作为下一个生成器的输入。

这意味着，当我们的reducer代码遍历这些生成器时，只有一个项可以通过各种映射层到达代码。

让我们来实现这个：

```javascript
//此函数通过iterable组成一个生成器。
//这个方法是为每个iteratee创造生成器，
//以便每个项来自原始的可迭代，向下传递，
//通过每个iteratee，在映射下一个项之前。
function composeGenMap(...iteratees) {

	//我们正在返回一个生成器函数。
	//那样，可以使用相同的映射组合，
	//可以应用于多个迭代，而不仅仅是一个。
	return function* (iterable) {

		//为每个iteratee创建生成器传递给函数。
		//下一个生成器将前一个生成器作为“itarable”参数
		for (let iteratee of iteratees) {
			iterable = genMap(iterable, iteratee);
		}

		//简单地传递我们创建的最后一个迭代。
		yield* iterable;
	}
}

//我们的可迭代数据源 
var array = [1, 2, 3];

//使用3个iteratee函数创建“composed”映射生成器。
var composed = composeGenMap(
	x => x + 1,
	x => x * x,
	x => x - 2
);

//现在我们可以迭代组合的生成器，
//传递它到我们的迭代和惰性的映射值。
for (let item of composed(array)) {
	console.log('composed', item);
}

//→
// composed 2
// composed 7
// composed 14
```


### 协程

协程是一种允许协作式多任务处理的并发技术。这意味着如果我们应用程序的一部分需要执行一些任务，

它可以这样做，然后将控制权移交给应用程序的另一部分。想想一个子程序，或者更接近的，一个函数。

这些子程序通常依赖于其他子程序。然而，它们不仅仅是连续运行，而是相互合作。


在JavaScript中，没有内在的协程机制。生成器不是协程，但它们具有相似的属性。例如，生成器可以

暂停执行一个函数，去控制另一个执行上下文，然后重新获得控制和恢复。这让我们有些想象空间，

但是生成器只是用于生成值，它并不是是我们了解协程所必须的。在本节中，我们将介绍使用生成器在

JavaScript中实现协程的一些方法。


#### 创建协程函数

生成器为我们提供了在JavaScript中实现协同函数所需的大部分内容; 他们可以暂停并继续执行。我们

只需要在生成器周围实现一些细微的抽象，这样我们正在使用的函数实际上就像调用协程函数，而不是迭代

生成器。以下大致说明我们希望协程在调用时的行为：

![image086.gif](../images/image086.gif)


这个方法是调用协程函数从一个yield语句移动到下一个。我们可以通过传递一个参数来为协程提供输入，

然后由yield语句返回。这是很多要记住的，所以让我们在函数包装器中概述这些协程概念：

```javascript
//取自：http://syzygy.st/javascript-coroutines/
//该工具函数接受一个生成器函数，然后返回
//协程函数。任何时候协程被调用，
//它的工作都是在生成器上调用“next()”。
//
//结果是生成器函数可以无限地运行，
//只到当它命中“yield”语句时暂停。
function coroutine(func) {
	//创建生成器，并移动函数
	//在第一个“收益”声明之前。
	var gen = func();
	gen.next();

	//“val”通过“yield”语句传递给生成器函数。
	//然后从那里恢复，直到它到达另一个yield。
	return function(val) {
		gen.next(val);
	}
}
```

非常简单 - 五行代码，但它也很强大。Harold的包装器返回的函数只是将生成器推进到下一个yield语句，

如果提供了参数，则将参数提供给next()。声明工具函数是一种方法，但让我们实际使用它来实现协程函数：

```javascript
//在调用时创建一个coroutine函数，
//进入到下一个yield语句。
var coFirst = coroutine(function* () {
	var input;
	
	//输入来自yield语句，
	//而且是传递给“coFirst()”的参数值。
	input = yield;
	console.log('step1', input);
	input = yield; 
	console.log('step3', input);
});


//与上面创建的协程一样工作... 
var coSecond = coroutine(function* () {
	var input;
	input = yield;
	console.log('step2', input);
	input = yield;
	console.log('step4', input);
});

//这两个协程彼此合作，按预期输出。
//我们可以看到对每个协程的第二次调用，
//会找到上一个yield语句暂停的位置。
coFirst('the money');
coSecond('the show');
coFirst('get ready');
coSecond('go');
//→
// step1 the money
// step2 the show
// step3 get ready
// step4 go
```

当完成某项任务涉及一系列步骤时，我们通常需要标记代码，临时值等。协程不需要这些，因为函数只是暂停，

任何本地状态都保持不变。换句话说，当协程为我们隐藏这些细节时，没有必要将并发逻辑与我们的应用

程序逻辑交织在一起。


#### 处理DOM事件

我们可以使用协程的其他地方是DOM作为事件处理程序。这通过将相同的coroutine()函数作为事件侦听器

添加到多个元素来工作。让我们回想一下，对这些协程函数的每次调用都与单个生成器进行通信。这意味着我们

设置为处理DOM事件的协程将作为流传入。这几乎就像我们在迭代这些事件一样。


由于这些协程函数使用相同的生成器，因此元素可以使用此技术轻松地互相通信。DOM事件的典型方法涉及回调

函数，这些函数与元素之间共享的某种中心源进行通信并维护状态。使用协程，元素通信的状态隐含在我们的

函数代码中。让我们在DOM事件处理程序的上下文中使用我们的协程包装器：

```javascript
//与mousemove一起使用的协程函数
var onMouseMove = coroutine(function* () {
	var e;

	//这个循环无限地执行。
	//事件对象通过yield语句传入。
	while (true) {
		e = yield;
		
		//如果元素被禁用，则不执行任何操作。
		//否则，打印记录消息。
		if (e.target.disabled) {
			continue;
		}
		console.log('mousemove', e.target.textContent);
	}
});

//与点击事件一起使用的协程函数。
var onClick = coroutine(function* () {
	//保存对我们两个按钮的引用。
	//协程是有状态的，它们永远都是可用的
	var first = document.querySelector('button:first-of-type');
	var second = document.querySelector('button:last-of-type'),
	var e;
	
	while (true) {
		e = yield;
		
		//按钮被单击后禁用。
		e.target.disabled = true;
		
		//如果单击了第一个按钮，
		//则切换第二个按钮的状态。
		if(Object.is(e.target, first)) {
			second.disabled = !second.disabled;
			continue;
		}

		//如果单击了第二个按钮，
		//则切换第一个按钮的状态。
		if(Object.is(e.target, second)) {
			first.disabled = !first.disabled;
		}
	}
});

//设置事件处理程序 - 我们的协程函数。
for (let document of document.querySelectorAll('button')) {
	button.addEventListener('mousemove', onMouseMove);
	button.addEventListener('click', onClick);
}
```


#### 处理promise的值

在上一节中，我们了解了如何使用coroutine()函数来处理DOM事件。我们使用相同的coroutine()函数，

将事件视为数据流，而不是随意添加响应DOM事件的回调函数。DOM事件处理程序更容易相互协作，因为

它们共享相同的生成器上下文。


我们可以将相同的方法应用于promise的then()回调，它的工作方式与DOM协程方法类似。我们将协程

传递给then()，而不是传递常规函数。当promise解析时，协程将前进到下一个yield语句以及

已解析的值。我们来看看下面的代码：

```javascript
//一系列promise的数组。
var promises = [];

//我们的完成回调是一个协程。
//这意味着每次调用它时，都会有新的promise完成值显示在这里。
var onFulfilled = coroutine(function* () {
	var data;

	//当他们返回时继续处理已完成的promise值
	while (true) {
		data = yield;
		console.log('data', data);
	}
});

//在1到5秒之间，创建5个随机解析的promises。
for (let i = 0; i < 5; i++) {
	promises.push(new Promise((resolve, reject) => {
		setTimeout(() => {
			resolve(i);
		}, Math.floor(Math.random() * (5000 - 1000)) + 1000);
	}));
}

//将我们的完成协程附加为“then()”回调。
for (let promise of promises) {
	promise.then(onFulfilled);
}
```

这非常有用，因为它提供了静态promise方法所不具备的功能。该Promise.all()方法迫使我们等待所有的promise

完成，在处理返回promise之前。但是，在已解析的promise值彼此不相关的情况下，我们可以简单地迭代它们，

在它们按任何顺序解析时进行响应。


我们可以通过将原生函数附加到then()作为回调来类似的实现，但是，当它们完成时，我们就不会有共享上下文

给promise值来处理。另一种方法是我们可以通过将promises与协程相结合来采用声明一系列协程响应不同的

协程，具体取决于它们响应的数据类型。这些协程将在整个应用程序期间继续存在，并在创建时传递给promise。


### 小结

这一章向你介绍了生成器的概念，ES6的新结构，这让我们能够实现惰性计算。生成器帮助我们实现了并发原则，

让我们能够避免计算和内存分配的浪费。有一些与生成器关联的新语法形式。首先，是生成器函数，它总是

返回一个生成器实例。这些声明不同于常规函数。这些函数是用于生成值，依赖于yield关键字。


然后，我们探索了更高级的生成器和惰性计算话题，包括传递到其他生成器，实现map/reduce工具函数，以及将

数据传递到生成器。在本章的结尾，我们看了如何使用生成器来实现协同函数。


在下一章中，我们将介绍Web workers - 第一次看看如何在浏览器环境中使用并发性。
