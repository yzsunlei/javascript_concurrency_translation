## 第三章 使用Promises实现同步

Promises很多年前就在JavaScript类库中实现了。这一切都始于Promises/A+规范。这些类库的实现都有它们自己的形式，

直到最近（确切地说是ES6），Promises规范才被JavaScript语言纳入。如标题那样 - 它帮助我们应用同步原则。


在本章中，我们将首先简单介绍Promises中各种术语，以便更容易理解本章的后面部分。然后，通过各种方式，

我们将使用Promises来解决目前的一些问题，并在处理并发时更容易。准备好了吗？


### Promise相关术语

在我们深入研究代码之前，让我们花一点时间确保我们牢牢掌握Promises有关的术语。有Promise实例，但是还有各种各样的状态

和方法。如果我们能够弄清楚Promise这些定义，那么后面的章节会更有意义。这些解释简短易懂，所以如果您已经使用过Promises，

您可以快速看下这些定义，就当复习下。


#### Promise

顾名思义，Promise是一种承诺。将Promise视为尚不存在的值的代理。Promise让我们更好的编写并发代码，因为我们

知道值会在将来某个时刻存在，并且我们不必编写大量重复的状态检查代码。


#### State

Promises总是处于以下三种状态之一：

• 等待：这是Promise创建后的第一个状态。它一直处于等待状态，直到它完成或被拒绝。

• 完成：该Promise值已经处理，并能为它提供then()回调函数。

• 拒绝：处理Promise的值出了问题。现在没有数据。

Promise状态的一个有趣特性是它们只转换一次。他们要么从等待状态到完成，要么从等待状态到被拒绝。一旦他们进行了这种状态

转换，后面他们就会锁定在这种状态。


#### Executor

执行器函数负责以某种方式解析该值将处于等待状态。创建Promise后立即调用此函数。它需要两个参数：解析器函数和拒绝函数。


#### Resolver

解析器函数是一个作为参数传递给执行器函数的函数。实际上，这非常方便，因为我们可以将解析器函数传递给另一个函数，依此类推。

调用解析器函数的位置并不重要，但是当它被调用时，Promise会进入一个完成状态。状态的这种改变将触发then()回调 - 这些我们

将在后面看到。


#### Rejector

拒绝函数与解析器函数相似。它是传递给执行器函数的第二个参数，可以从任何地方调用。当它被调用时，Promise从等待状态改变到

拒绝状态。这种状态的改变将调用错误回调函数，如果有的话，会传递给then()或catch()。


#### Thenable

如果对象具有接受完成回调和拒绝回调作为参数的then()方法，则该对象就是Thenable。换句话说，Promise是Thenable。

但是在某些情况下，我们可能希望实现专门的解析语义。


### 完成和拒绝Promises

如果上一节刚刚介绍的几个术语听起来让你困惑，那别担心。从本节开始，我们将看到所有这些Promises术语在

实践中的应用。在这里，我们将展示一些简单的Promise解决和拒绝的示例。


#### 完成Promises

解析器是一个函数，顾名思义，它完成了我们的Promise。这不是完成Promise的唯一方法 - 我们将在后面探索更高级的方式。

但到目前为止，这种方法是最常见的。它作为第一个参数传递给执行器函数。这意味着执行器可以通过简单地调用

解析器直接完成Promise。但这并没有带给我们很多实用性，是吗？


更常见情况是Promise执行器函数设置即将发生的异步操作 - 例如拨打网络电话。然后，在这些异步操作的

回调函数中，我们可以完成这个Promise。在我们的代码中传递一个解析函数，刚开始可能感觉有点违反直觉，

但是一旦我们开始使用它们就会发现很有意义。


解析器函数是一个相对Promise来说比较难懂的函数。它只能完成一次Promise。我们可以调用解析器很多次，但只在

第一次调用会改变Promise的状态。下面是一个图描绘了Promise的可能状态;它还显示了状态之间是如何变化的：

![image060.gif](image060.gif)


现在，我们来看看一些Promise代码。在这里，我们将完成一个promise，它会调用then()完成回调函数：

```javascript
//我们的Promise使用的执行器函数。
//第一个参数是解析器函数，在1秒后调用完成Promise。
function executor(resolve) {
	setTimeout(resolve, 1000);
}

//我们Promise的完成回调函数。
//这个简单地在我们的执行程序函数运行后，停止那个计时器。
function fulfilled() {
	console.timeEnd('fulfillment');
}

//创建promise，并立即运行，
//然后启动一个计时器来查看调用完成函数需要多长时间
var promise = new Promise(executor);
promise.then(fulfilled);
console.time('fulfillment');
```

我们可以看到，解析器函数被调用时fulfilled()函数会被调用。执行器实际上并不调用解析器。相反，它将解析器函数传递

给另一个异步函数 - setTimeout()。执行器并不是我们试图去弄清楚的异步代码。可以将执行器视为一种协调程序，

它编排异步操作并确定何时执行Promise。


前面的示例未解析任何值。当某个操作的调用者需要确认它成功或失败时，这是一个有效的用例。相反，让我们这次尝试解析

一个值，如下所示：

```javascript
//我们的Promise使用的执行函数。
//创建Promise后，设置延时一秒钟调用"resolve()"，
//并解析返回一个字符串值 - “完成！”。
function executor(resolve) {
	setTimeout(() => {
		resolve('done!');
	}, 1000);
}

//我们Promise的完成回调接受一个值参数。
//这个值将传递到解析器
function fulfilled(value) {
	console.log('resolved', value);
}

//创建我们的Promise，提供执行程序和完成回调函数。
var promise = new Promise(executor);
promise.then(fulfilled);
```

我们可以看到这段代码与前面的例子非常相似。区别在于我们的解析器函数实际上是在传递给setTimeout()的回调函数的闭包内

调用的。这是因为我们正在解析一个字符串值。还有一个将被解析的值参数传递给我们的fulfilled()函数。


#### 拒绝promises

Promise执行器函数并不总是按期望进行，当意外发生时，我们需要拒绝promise。这是从等待状态转换到

另一个可能的状态。这不是进入一个完成状态而是进入一个被拒绝的状态。这会导致执行不同的回调，与完成回调

函数是分开的。值得庆幸的是，拒绝Promise的机制与完成Promise非常相似。我们来看看这是如何实现的：

```javascript
//此执行器在延时一秒后拒绝Promise
//它使用拒绝回调函数来改变状态，
//并传递拒绝的参数值到回调函数。
function executor(resolve, reject) {
	setTimeout(() => {
		reject('Failed');
	}, 1000);
}

//用作拒绝回调的函数。
//它接收提供拒绝的参数值。
function rejected(reason) {
	console.error(reason);
}

//创建promise，并运行执行器。
//使用“catch()”方法来接收拒绝回调函数。
var promise = new Promise(executor);
promise.catch(rejected);
```

这段代码看起来和在上一节中看到的解析代码非常相似。我们设置了超时，并且我们拒绝了它而不是完成它。这是使用

rejector函数完成的，并作为第二个参数传递给执行器。


我们使用catch()方法而不是then()方法来设置拒绝回调函数。我们将在本章后面看到then()方法如何用于

同时处理完成和拒绝回调函数。此示例中的拒绝回调函数仅将失败原因打印出来。通常情况下提供此返回值很重要。

当我们完成promise时，返回值也是常见的，尽管不是必需的。另一方面，对于拒绝函数，一般也很少有情况仅仅通过

回调函数打印拒绝原因。


让我们看一下另一个例子，它捕获执行器中抛出的异常，并为拒绝回调函数提供更有意义的报错解释：

```javascript
//此promise执行程序抛出错误，
//并调用拒绝回调函数打印原因。
new Promise(() => {
	throw new Error('Problem executing promise');
}).catch((reason) => {
	console.error(reason);
});

//此promise执行程序捕获错误，
//并调用拒绝回调函数打印更有意义的原因
new Promise((resolve, reject) => {
	try {
		var size = this.name.length;
	} catch (error) {
		reject(error instanceof TypeError ? 'Missing "name" property' : error);
	}
}).catch((reason) => {
	console.error(reason);
});
```

前一个例子中第一个Promise的有趣之处在于它确实改变了状态，即使我们没有使用resolve()或reject()明确地

改变promise的状态。然而，最终改变promise的状态是很重要的; 我们将在下一节中探讨这个话题。


#### 空Promises

尽管事实上执行器函数传递了一个完成回调函数和拒绝回调函数，但并不保证promise将改变状态。有些情况下，

promise只是挂起，并没有触发完成回调也没有触发拒绝回调。这可能并没有什么问题，事实上，简单的promises，

就容易发现和修复没有响应的promises。然而，随着我们进入更复杂的场景后，一个promise的完成回调可以作为

其他几个promise的回调结果。如果一个promises不能完成或拒绝，然后整个流程将崩溃。这种情况调试起来是非常

麻烦的;下面的图可以很清楚的看到这个情况：

![image061.gif](image061.gif)


在图中，我们可以看到哪个promise导致依赖的promise挂起，但调试代码来解决这个问题并不方便。现在让我们看

一下导致promise挂起的执行函数：

```javascript
//这个promise能够正常运行执行器函数。
//但“then()”回调函数永远不会被执行。
new Promise(() => {
	console.log('executing promise');
}).then(() => {
	console.log('never called');
});

//此时，我们并不知道promise出了什么问题
console.log('finished executing, promise hangs');
```

但是，是否有一种更安全的方式来处理这种不确定性呢？在我们的代码中，我们不需要挂起而无需完成或拒绝的

执行函数。让我们来实现一个执行器包装函数，像一个安全网那样让太长时间还没完成的promises执行拒绝回调函数。

这将揭开解决不好处理的promise场景的神秘面纱：

```javascript
//promise执行器函数的包装器，
//在给定的超时时间后抛出错误。
function executorWrapper(func, timeout) {
	//这是实际调用的函数。
	//它需要解析器函数和拒绝器函数作为参数。
	return function executor(resolve, reject) {
		//设置我们的计时器。
		//当时间到达时，我们可以使用超时消息拒绝promise。
		var timer = setTimeout(() => {
			reject('Promise timed out after $​​ {timeout} MS');
		}, timeout);
		
		//调用我们原来的执行器包装函数。
		//我们实际上也包装了完成回调函数
		//和拒绝回调函数，所以当
		//执行者调用它们时，会清除定时器。
		func((value) => {
				clearTimeout(timer);
				resolve(value);
			}, (value) => {
				clearTimeout(timer);
				reject(value);
			});
		};
	}

//这个promise执行后超时，
//超时错误消息传递给拒绝回调。
new Promise(executorWrapper((resolve, reject) => {
	setTimeout(() => {
		resolve('done');
	}, 2000);
}, 1000)).catch((reason) => {
	console.error(reason);
}）;

//这个promise执行后按预期运行，
//在定时结束之前调用“resolve()”。
new Promise(executorWrapper((resolve，reject) => {
	setTimeout(() => {
		resolve(true);
	}, 500);
}, 1000)).then((value) => {
	console.log('resolved', value);
});
```

### 对promises作出改进

既然我们已经很好地理解了promises的执行机制，本节将详细介绍如何使用promises来解决特定问题。通常，这意味着当

promises完成或被拒绝时，我们会达到我们某些目的。


我们将首先查看JavaScript解释器中的任务队列，以及这些对我们的解析回调函数的意义。然后，我们将考虑使用

promise的结果数据，处理错误，创建更好的抽象来响应promises，以及thenables。让我们开始吧。


#### 处理任务队列

JavaScript任务队列的概念在“第2章-JavaScript的执行模型”中提到过。它的主要职责是初始化新的执行上下文堆栈。

这是常见的任务队列。然而，还有另一种队列，这是专用于执行promises回调的。这意味着，如果他们都存在时，算法

会从这些队列中选择一个任务执行。


Promise自带了并发语义，并且有很好的理由。如果一个promise被用来确保某个值最终被解析，那么为对其作出响应的

代码赋予高优先级是有意义的。否则，当值到达时，处理它的代码可能还要在其他任务后面等待很长的时间才能执行。

让我们编写一些代码来演示这些并发语义：

```javascript
//创建5个promise，记录它们的执行时间，
//以及当他们对返回值做出响应的时间
for (let i = 0; i <5; i++) {
	new Promise((resolve) => {
		console.log('execting promise');
		resolve(i);
	}).then((value) => {
		console.log('resolved', i);
	});
}

//在任何promise完成回调之前，这里会先被调用，
//因为堆栈任务需要在解释器进入promise解析回调队列之前完成，
//当前5个“then()”回调将被置后。
console.log('done executing');

//→
//execting promise
//execting promise
// ...
//done executing
//resolved 1
//resolved 2
// ...
```

> 拒绝回调也遵循同样的语义。


#### 使用promise的返回数据

到目前为止，我们已经在本章中看到了一些示例，其中解析器函数完成promise后并返回值。传递给此函数的值

是最终传递给完成回调函数的值。通过让执行程序设置任何异步操作的方法，例如setTimeout()，延时传递

该值调用解析程序。但在这些例子中，调用者实际上并没有等待任何值; 我们只使用setTimeout()作为示例

异步操作。让我们看一下我们实际上没有值的情况，异步网络请求需要获取到它：

```javascript
//用于从服务器获取资源的通用函数,
//返回一个promise。
function get(path) {
	return new Promise((resolve, reject) => {
		var request = new XMLHttpRequest();
		
		//promise解析数据加载后的JSON数据。
		request.addEventListener('load', (e) => {
			resolve(JSON.parse(e.target.responseText));
		});

		//当请求出错时，promise执行拒绝回调函数。
		request.addEventListener('error', (e) => {
			reject(e.target.statusText || '未知错误');
		});


		//如果请求被中止时，我们调用完成回调函数
		request.addEventListener('abort', resolve);
		
		request.open('get', path);
		request.send();
	});
}

//我们可以直接附加我们的“then()”处理程序
//到“get()”，因为它返回一个promise。
//在解析之前，这里使用的值是一个真正的异步操作，
//因为必须发请求获取远程值。
get('api.json').then((value) => {
	console.log('hello', value.hello);
});
```


使用像get()这样的函数，它们不仅始终返回像promise一样的原生类型，而且还封装了一些让人讨厌的异步细节。

在我们的代码中处理XMLHttpRequest对象并不令人愉快。我们已经简化了可以返回的各种情况。而不是总是必须为

load，error和abort事件创建处理程序，我们只需要关心一个接口 - promise。这就是同步并发原则的全部内容。


#### 错误回调

有两种方法可以对被拒绝的promise做出处理。换句话说，提供错误回调。第一种方法是使用catch()方法，该方法

使用单一回调函数。另一种方法是将被拒绝的回调函数作为then()的第二个参数传递。


将then()方法用来处理拒绝回调函数在某些情况下表现的更好，它应该被使用替代catch()函数。第一个

场景是编写promises和thenable对象可以互换的代码。catch()方法不是thenable必要的一部分。

在第二个场景，是当我们建立回调链时，我们将在这一章后面探讨。


让我们看一些代码，它们比较了两种为promises提供拒绝回调函数的方法：

```javascript
//这个promise执行器将随机执行完成回调或拒绝回调
function executor(resolve, reject) {
	cnt++;
	Math.round(Math.random()) ? 
		resolve(`fulfilled promise ${cnt}`) :
		reject(`rejected promise ${cnt}`);
}

//让“log()”和“error()”函数作为简单回调函数
var log = console.log.bind(console),
	error = console.error.bind(console),
	cnt = 0;

//创建一个promise，然后通过“catch()”方法传入拒绝回调。
new Promise(executor).then(log).catch(error);

//创建一个promise，然后通过“then()”方法传入拒绝回调。
new Promise(executor).then(log, error);
```

我们可以看到这两种方法实际上非常相似。在代码美观上，也没有哪个有真正的优势。然而，当涉及到使用

thenables时，then()方法有一个优势，我们后面会看到。但是，由于我们实际上并没有以任何方式使用

promise实例，除了添加回调之外，实际上没有必要担心catch()和then()用于注册拒绝回调。


#### 始终响应

Promises最终总是结束于完成状态或拒绝状态。我们通常为每个状态传入不同的回调函数。但是，我们很可能希望为这

两个状态执行一些相同的操作。例如，如果使用promise的组件在promise等待时更改状态，我们要确保在完成或拒绝

promise后清除状态。


我们可以用这样的方式编写代码：完成和拒绝状态的回调每个都去执行这些操作，或者他们每个都可以调用执行一些

公用的清理函数。下面这种方式的示图：

![image065.gif](image065.gif)


将清理责任分配给promise是否更有意义，而不是将其分配给个别结果？这样，在解析promise时运行的回调函数专注于

它需要对值执行的操作，而拒绝回调则专注于处理错误。让我们看看是否可以使用always()方法编写一些扩展

promises的代码：

```javascript
//在promise原型上扩展使用“always()”方法。
//不管promise是完成还是拒绝，始终会调用给定的函数。
Promise.prototype.always = function(func) {
	return this.then(func, func);
};

//创建promise随机完成或被拒绝。
var promise = new Promise((resolve，reject) => {
	Math.round(Math.random()) ? 
	resolve('fullfilled') : reject('rejected');
});

//传递promise完成和拒绝回调。
promise.then((value) => {
	console.log(value);
}, (reason) => {
	console.error(reason);
});

//这个回调函数总是会在上面的回调执行之后调用。
promise.always((value) => {
	console.log('清理...');
});
```

> 请注意，顺序在这里很重要。如果我们在then()之前调用always()，那么函数仍然会一直运行，但它会在
> 回调提供给then()之前运行。我们实际上可以在then()之前和之后调用always()，以便在完成或拒绝回调
> 之前以及之后运行代码。


#### 处理其他promises

到目前为止，我们在本章中看到的大多数promise都是由执行程序函数直接完成的，或者是当值准备完成时从

异步操作中调用解析器的结果。像这样传递回调函数实际上非常灵活。例如，执行程序甚至不必执行任何工作，

除了将解析器函数存储在某处以便稍后调用它来解析promise。


当我们发现自己处于需要多个值的更复杂的同步场景时，这可能特别有用，这些值已经被传递给调用者。

如果我们有处理回调函数，我们就可以处理promise。让我们看看，在存储代码的解析函数的多个promises，

使每一个promise都可以在后面处理：

```javascript
//存储一系列解析器函数的列表。
var resolvers = [];

//在执行器中创建5个新的promise,
//解析器被推到了“resolvers”数组。
//我们可以给每一个promise执行回调。
for(let i = 0; i < 5; i++) {
	new Promise(() => {
		resolvers.push(resolve);
	}).then((value) => {
		console.log(`resolved ${i + 1}`, value);
	});
}

//设置一个2s之后延时运行函数，
//当它运行时，我们遍历“解析器”数组中的每一个解析器函数，
//并且传入一个返回值来调用它。
setTimeout(() => {
	for(resolver of resolvers) {
		resolver(true);
	}
}, 2000);
```

正如这个例子所表明的那样，我们不必在executor函数内处理它们。事实上，我们甚至不需要在创建和设置

执行程序和完成函数之后显式引用promise实例。解析器函数已存储在某处，它包含对promise的引用。


#### 类Promise对象

Promise类是一种原生的JavaScript类型。但是，我们并不总是需要创建新的promise实例来实现相同的

同步操作。我们可以使用静态Promise.resolve()方法来解析这些对象。让我们看看如何使用此方法：

```javascript
//“Promise.resolve()”方法可以处理thenable对象 。
//这是一个带有“then()”方法的类似于执行器的对象。
//这个执行器将随机完成或拒绝promise。
Promise.resolve({then: (resolve, reject) => {
	Math.round(Math.random()) ? resolve('fulfilled') : reject('rejected');

	//这个方法返回一个promise，所以我们能够
	//设置已完成和被拒绝的回调函数
}}).then((value) => {
	console.log('resolved', value);
}, (reason) => {
	console.error('reason', reason);
});
```

我们将在本章的最后一节中再讨论Promise.resolve()方法，以了解更多用例。


### 建立回调链

我们在本章前面介绍的每种promise方法都会返回promise。这允许我们在返回值上再次调用这些方法，从而

产生then().then()调用的链，依此类推。链式promise具有挑战性的一个方面是promise方法返回的是新实例。

也就是说，我们将在本节中探讨promise一定程度上的不变性。


随着我们的应用程序变得越来越大，并发性挑战随之增加。这意味着我们需要考虑更好的方法来利用原生同步语法，

例如promises。正如JavaScript中的任何其他原始值一样，我们可以将它们从函数传递给函数。我们必须以

同样的方式处理promises - 传递它们，并建立在回调函数链上。


#### Promises只改变状态一次

Promise诞生于等待状态，并且它们结束于已解决或被拒绝的状态。一旦promise转变为其中一种状态，

它们就会陷入这种状态。这有两个有趣的副作用。


首先，多次尝试解决或拒绝promise将被忽略。换句话说，解析器和拒绝器是幂等的 - 只有第一次调用对

promise有影响。让我们看看这代码如何执行：

```javascript
//此执行器函数尝试解析promise两次，
//但完成的回调只调用一次
new Promise((resolve, reject) => {
	resolve('fulfilled');
	resolve('fulfilled');
}).then((value) => {
	console.log('then', value);
});

//这个执行器函数尝试拒绝promise两次，
//但拒绝的回调只调用一次
new Promise((resolve, reject) => {
	reject('rejected');
	reject('rejected');
}).catch((reason) => {
	console.error('reason');
});
```


promises仅改变状态一次的另一个含义是promise可以在添加完成或拒绝回调之前处理。竞争条件，例如这个，

是并发编程的残酷现实。通常，回调函数会在创建时添加到promise中。由于JavaScript是运行即完成的，

因此在添加回调之前，不会处理promise解析回调的任务队列。但是，如果promise立即在执行中解析怎么办？

如果将回调添加到另一个JavaScript执行上下文的promise中会怎样？让我们看看是否可以用一些代码来

更好地说明这些情况：

```javascript
//此执行器函数立即解析promise。添加“then()”回调时，
//promise已经解析了。但回调函数仍然会使用已解析的值进行调用。
new Promise((resolve, reject) => {
	resolve('done');
	console.log('executor', 'resolved');
}).then((value) => {
	console.log('then', value);
});

//创建一个立即解决的新promise执行器函数。
var promise = new Promise((resolve，reject) => {
	resolve('done');
	console.log('executor', 'resolved');
});

//这个回调是promise解析后就立即执行了。
promise.then((value) => {
	console.log('then 1', value);
});

//此回调在promise解析后未添加到另一个的promise中，
//它仍然被立即调用并获得已解析的值。
setTimeout(() => {
	promise.then((value) => {
		console.log('then 2', value);
	});
}, 1000);
```

此代码说明了promises的一个非常重要的特性。无论何时将执行回调添加到promise中，无论是处于暂时

挂起状态还是解析状态，使用promise的代码都不会更改。从表面上看，这似乎不是什么大不了的事。

但是这种竞争条件检查的类型需要更多的并发代码来保护自己。相反，Promise原生语法为我们处理

这个问题，我们可以开始将异步值视为原生类型。


#### 不可改变的promises

promises并非真正不可改变。它们改变状态，then()方法将回调函数添加到promise。但是，有一些不可改变

的promises特征值得在这里讨论，因为它们会在某些情况下影响我们的promise代码。


从技术上讲，then()方法实际上并没有改变promise对象。它创建了所谓的promise能力，它是一个引用

promise的内部JavaScript记录，以及我们添加的函数。因此，它不是JavaScript语言中的真正语法。


这是一张图，说明当我们链接两个或更多then()一起调用时会发生什么：

![image069.gif](image069.gif)


我们可以看到，then()方法不会返回与上下文一起调用的相同实例。相反，then()创建一个新的promise实例

并返回它。让我们看一些代码，来进一步的说明当我们使用then()将promises链接在一起时会发生的事情：

```javascript
//创建一个立即解析的promise，
//并且存储在“promise1”中。
var promise1 = new Promise((resolve, reject) => {
	resolve('fulfilled');
});

//使用“promise1”的“then()”方法创建一个
//新的promise实例，存储在“promise2”中。
var promise2 = promise1.then((value) => {
	console.log('then 1', value);
	//→then 1 fulfilled
});

//为“promise2”创建一个“then()”回调。这实际上
//创建第三个promise实例，但我们不用它做任何事情。
promise2.then((value) => {
	console.log('then 2', value);
	//→then 2 undefined
});

//确信“promise1”和“promise2”实际上是不同的对象
console.log('equal', promise1 === promise2);
//→equal false
```

我们可以清楚地看到这两个创建promise的实例在这个例子中是独立的promise对象。值得指出的是第二个

promise执行前时，一定是它执行了第一个promise。但是，我们可以看到的是该值不会传递到第二个promise。

我们将在下一节中解决此问题。


#### 有多少个then()回调，就有多少个promises对象

正如我们在上一节中看到的那样，使用then()创建的promise将绑定到它们的创建者。也就是说，当第一个

promise完成时，绑定它的promise也会完成，依此类推。但是，我们也发现了一个小问题。已解析的值不会

使其传递到第一个回调函数。这样做的原因是为响应promise解析而运行的每个回调都是第一个回调的返回值

被送入第二个回调，依此类推。我们的第一个回调将值作为参数的原因是因为这在promise机制中显然会发生的。


我们来看看另一个promise链示例。这一次，我们将显式返回回调函数中的值：

```javascript
//创建一个新promise随机调用解析回调或拒绝回调。
new Promise((resolve, reject) => {
	Math.round(Math.random()) ?
	resolve('fulfilled') : reject('rejected');
}).then((value) => {
	//在完成原始promise时调用返回值，
	//以防另一个promise链接到这一个。
	console.log('then 1', value); 
	return value;
}).catch((reason) => {
	//链接到第二个promise，
	//当拒绝回调时执行。
	console.error('catch 1', reason);
}).then((value) => {
	//链接到第三个promise，
	//按预期得到值，并返回值给任何下个promise回调使用。
	console.log('then 2', value);
	return value;
}).catch((reason) => {
	//这里永不会被调用，
	//拒绝回调不会通过promise链传递。
	console.error('catch 2', reason);
});
```

这看起来不错。我们可以看到已解析的值通过promise链传递。有一个异常 - 拒绝回调不会向后传递。相反，

只有链中的第一个promise拒绝回调会执行。其余的promise回调只是完成，而不是拒绝。这意味着最后一个

catch()回调永远不会运行。


当我们以这种方式将promise链接在一起时，我们的执行回调函数需要能够处理错误条件。例如，已解析的值

可能具有error属性，可以检查其具体问题。


#### promises传递

在本节中，我们讲讲promise作为原始值的用法。我们经常用原始值做的事情是将它们作为参数传递给函数，

并从函数中返回它们。promise和其他原生语法之间的关键区别在于我们如何使用它们。其他值是始终都存在，

而promise的值到未来某个时间点才存在。因此，我们需要通过回调函数定义一些操作过程，当值获得时去执行。


promises的好处是用于提供这些回调函数的接口小巧且一致。当我们将值与将作用于它的代码耦合时，我们

不需要再去自主创造同步机制。这些单元可以像任何其他值一样在我们的应用程序中运用，并且并发语义

是常见的。这是几个promise函数相互传递的示图：

![image071.gif](image071.gif)


在这个函数堆栈调用结束时，我们得到一个完成几个promise的解析的promise对象。整个promise链是从第一个

promise完成而开始的。比如何遍历promise链的机制​​更重要的是所有这些函数都可以自由使用这个promise传递

的值而不影响其他函数。


在这里有两个并发原则。首先，我们通过执行异步操作仅只能处理该值一次; 每个回调函数都可以自由使用

此解析值。其次，我们在抽象同步机制方面做得很好。换句话说，代码并没有带有很多重复代码。让我们看看传递

promise的代码实际的样子：

```javascript
//简单实用的工具函数，
//将多个较小的函数组合成一个函数。
function compose(...funcs) {
	return function(value) {
		var result = value;
		
		for(let func of funcs) {
			result = func(value);
		}
		return result;
	};
}

//接受一个promise或一个完成值。
//如果这是一个promise，它添加了一个“then()”回调并返回一个新的promise。
//否则，它会执行“update”并返回值
function updateFirstName(value) {
	if (value instanceof Promise) {
		return value.then(updateFirstName);
	}

	console.log('first name', value.first); 
	return value;
}

//与上面的函数类似，
//只是它执行不同的UI“update”。
function updateLastName(value) {
	if (value instanceof Promise) {
		return value.then(updateLastName);
	} 

	console.log('last name', value.last); 
	return value;
}

//与上面的函数类似，除了它
//只是它执行不同的UI“update”。
function updateAge(value) {
	if (value instanceof Promise) {
		return value.then(updateAge);
	}

	console.log('age', value.age);
	return value;
}

//一个promise对象，
//它在延时一秒钟之后，
//携带一个数据对象完成promise。
var promise = new Promise((resolve, reject) => {
	setTimeout(() => {
		resolve({
			first: 'John',
			last: 'Smith',
			age: 37
		});
	});
}, 1000);

//我们组装一个“update()”函数来更新各种UI组件。
var update = compose(
	updateFirstName，
	updateLastName，
	updateAge
);

//使用promise调用我们的更新函数。
update(promise);
```

这里的关键函数是我们的更新函数 - updateFirstName()，updateLastName()和updateAge()。他们非常

灵活，接受一个promise或promise返回值。如果这些函数中的任何一个将promise作为参数，它们会通过

添加then()回调函数来返回新的promise。请注意，它添加了相同的函数。updateFirstName()将添加

updateFirstName()作为回调。当回调触发时，它将与此次用于更新UI的普通对象一起使用。因此，

promise如果失败，我们可以继续更新UI。


promise检查每个函数都需要三行，这并不是非常突兀的。最终结果是易读且灵活的代码。顺序无关紧要; 

我们可以用不同的顺序包装我们的update()函数，并且UI组件都将以相同的方式更新。我们可以将普通对象

直接传递给update()，一切都会同样执行。看起来不像并发代码的并发代码是我们在这里取得的重大成功。


### 同步多个promises

在本章前面，我们已经探究了单个promise实例，它解析一个值，触发回调，并可能传递给其他promises

处理。在本节中，我们将介绍几种静态Promise方法，它们可以帮助我们处理需要同步多个promise值的情况。


首先，我们将处理我们开发的组件需要同步访问多个异步资源的情况。然后，我们将看一下不常见的情况，

如异步操作在处理之前由于UI中发生的事件而变得无关紧要。


#### 等待promises

在我们等待处理多个promise的情况下，也许是将多个数据源转换后提供给一个UI组件使用，我们可以使用

Promise.all()方法。它将promise实例的集合作为输入，并返回一个新的promise实例。仅当完成了所有

输入的promise时，才会返回一个新实例。


then()函数是我们为Promise提供的创建新promise的回调。给出一组解析值作为输入。这些值对应于索引

输入promise的位置。这是一个非常强大的同步机制，它可以帮助我们实现同步并发原则，

因为它隐藏了所有的处理记录。


我们不需要几个回调，让每个回调都协调它们所绑定的promise状态，我们只需一个回调，它具有我们需要的

所有解析数据。这个示例展示如何同步多个promise：

```javascript
//用于发送“GET”HTTP请求的工具函数，
//并返回带有已解析的数据的promise。
function get(path) {
	return new Promise((resolve, reject) => {
		var request = new XMLHttpRequest();
		
		//当据加载数时，完成解析了JSON数据的promise
		request.addEventListener('load', (e) => {
			resolve(JSON.parse(e.target.responseText));
		});

		//当请求出错时，
		//promise被适当的原因拒绝。
		request.addEventListener('error', (e) => {
			reject(e.target.statusText || 'unknown error');
		});

		//如果请求被中止，我们继续完成处理请求 
		request.addEventListener('abort', resolve);
		
		request.open('get', path);
		request.send();
	});
}


//保存我们的请求promises。
var requests = [];

//发出5个API请求，并将相应的5个
//promise放在“requests”数组中。
for(let i = 0; i < 5; i++) {
	requests.push(get('api.json'));
}

//使用“Promise.all()”让我们传入一个数组promises，
//当所有promise完成时，返回一个已经完成的新promise。
//我们的回调得到一个数组对应于promises的已解析值。
Promise.all(requests).then((values) => {
	console.log('first', values.map(x => x[0])); 
	console.log('second', values.map(x => x[1]));
});
```


#### 取消promises

到目前为止，我们在本书中已看到的XHR请求具有中止请求的处理程序。这是因为我们可以手动中止请求并阻止任何

load回调函数运行。需要此功能的典型场景是用户单击取消按钮，或导航到应用程序的其他部分，从而使请求变得

毫无意义。


如果我们是要在抽象promise上更上一层楼，在同样的原则也适用。而一些可能发生的并发操作的执行让promise

变得毫无意义。promises和XHR请求的过程中之间的区别，是前者没有abort()方法。最后我们要做的

一件事是在我们的promise回调中开始引入可能并不必要的取消逻辑。


Promise.race()方法在这里可以帮助我们。顾名思义，该方法返回一个新的promise，它由第一个要解析的输入

promise决定。这可能你听的不多，但实现Promise.race()的逻辑并不容易。它实际上是同步原则，隐藏了应用

程序代码中的并发复杂性。我们来看看这个方法是怎么可以帮助我们处理因用户交互而取消的promise：

```javascript
//用于取消数据请求的解析器​​函数。
var cancelResolver;

//一个简单的“常量”值，用于处理取消promise
var CANCELED = {};

//我们的UI组件
var buttonLoad = document.querySelector('button.load'),
	buttonCancel = document.querySelector('button.cancel');

//请求数据，返回一个promise。
function getDataPromise() {
	//创建取消promise。
	//执行器传入“resolve”函数为“cancelResolver”，
	//所以它稍后可以被调用。
	var cancelPromise = new Promise((resolve) => {
		cancelResolver = resolve;
	});

	//我们实际想要的数据
	//这通常是一个HTTP请求，
	//但我们在这里使用setTimeout()简单模拟一下。
	var dataPromise = new Promise((resolve) => {
		setTimeout(() => {
			resolve({hello: 'world'});
		}, 3000);
	});

	//“Promise.race()”方法返回一个新的promise，
	//并且无论输入promise是什么，它都可以完成处理
	return Promise.race([cancelPromise, dataPromise]);
}

//单击取消按钮时，我们使用
//“cancelResolver()”函数来处理取消promise
buttonCancel.addEventListener('click', () => {
	cancelResolver(CANCELLED);
});

//单击加载按钮时，我们使用
//“getDataPromise()”发出请求获取数据。
buttonLoad.addEventListener('click', () => {
	buttonLoad.disabled = true;
	getDataPromise().then((value) => {
		buttonLoad.disabled = false;
		//promise得到了执行，但那是因为
		//用户取消了请求。所以我们这里
		//通过返回CANCELED “constant”退出。
		//否则，我们有数据可以使用。
		if (Object.is(value, CANCELED)) {
			return value;
		}
		
		console.log('loaded data', value);
	});
});
```

> 作为练习，尝试想象一个更复杂的场景，其中dataPromise是由Promise.all()创建的promise。我们的
> cancelResolver()函数可以一次取消许多复杂的异步操作。


### 没有执行器的promises

在最后一节中，我们将介绍Promise.resolve()和Promise.reject()方法。我们已经本章前面看到

Promise.resolve()如何处理thenable对象。它还可以直接处理值或其他promises。当我们实现一个可能

同步也可能异步的函数时，这些方法会派上用场。这不是我们想要使用具有模糊并发语义函数的情况。


例如，这是一个可能同步也可能异步的函数，让人感到迷惑，几乎肯定会在以后出现错误：

```javascript
//一个示例函数，它可能从缓存中返回“value”，
//也可能通过“fetchs”异步获取值。
function getData(value) {
	//如果它存在于缓存中，我们直接返回这个值
	var index = getData.cache.indexOf(value);
	if(index > -1) {
		return getData.cache[index];
	}

	//否则，我们必须通过“fetch”异步获取它。
	//这个“resolve()”调用通常是会在网络发起请求的回调函数
	return new Promise((resolve) => {
		getData.cache.push(value);
		resolve(value);
	});
}

//创建缓存。
getData.cache = [];

console.log('getting foo', getData('foo'));
//→getting foo Promise
console.log('getting bar', getData('bar'));
//→getting bar Promise
console.log('getting foo', getData('foo'));
//→getting foo foo
```

我们可以看到最后一次调用返回的是缓存值，而不是一个promise。这很直观，因为我们不需要通过promise获取

最终的值，我们已经拥有这个值！问题是我们让使用getData()函数的任何代码表现出不一致性。也就是说，

调用getData()的代码需要处理并发语义。此代码不是并发的。让我们通过引入Promise.resolve()来改变它：

```javascript
//一个示例函数，它可能从缓存中返回“value”，
//也可能通过“fetchs”异步获取值。
function getData(value) {
	var cache = getData.cache;
	//如果这个函数没有缓存，
	//那就拒绝promise。
	if(!Array.isArray(cache)) {
		return Promise.reject('missing cache');
	}

	//如果它存在于缓存中，
	//我们直接使用缓存的值返回完成的promise
	var index = getData.cache.indexOf(value);
	
	if (index > -1) {
		return Promise.resolve(getData.cache[index]);
	}

	//否则，我们必须通过“fetch”异步获取它。
	//这个“resolve()”调用通常是会在网络发起请求的回调函数
	return new Promise((resolve) => {
		getData.cache.push(value);
		resolve(value);
	});
}

//创建缓存。
getData.cache = [];

//每次调用“getData()”返回都是一致的。
//甚至当使用同步值时，
//它们仍然返回得到解完成的promise。
getData('foo').then((value) => {
	console.log('getting foo', `“${value}”`);
}, (reason) => {
	console.error(reason);
});

getData('bar').then((value) => {
	console.log('getting bar', `“${value}”`);
}, (reason) => {
	console.error(reason);
});

getData('foo').then((value) => {
	console.log('getting foo', `“$ {value}”`);
}, (reason) => {
	console.error(reason);
});
```

这样更好。使用Promise.resolve()和Promise.reject()，任何使用getData()的代码默认都是并发的，

即使数据获取操作是同步的。


### 小结

本章介绍了ES6中引入的Promise对象的大量细节内容，以帮助JavaScript程序员处理困扰该语言多年的同步

问题。大量的使用异步回调，这会产生回调地狱，因而我们要尽量避免它。


Promise通过实现一个足以解决任何值的通用接口来帮助我们处理同步问题。promise总是处于三种状态

之一 - 等待，完成或拒绝，并且它们只会改变一次状态。当这些状态发生改变时，将触发回调。promise有一个

执行器函数，其作用是设置使用promise的异步操作resolver函数或rejector来改变promise的状态。


promise带来的大部分价值在于它们如何帮助我们简化复杂的场景。因为，如果我们只需处理一个运行带有解析值

回调的异步操作，那么使用promises就不值得。这是不常见的情况。常见的情况是几个异步操作，每个操作都需要

解析返回值;并且这些值需要同步处理和转换。Promises有方法帮助我们这样做，因此，我们能够更好地将同步并发

原则应用于我们的代码。


在下一章中，我们将介绍另一个新引入的语法 - 生成器。与promises类似，生成器是帮助我们应用另一个并发原则

的机制 - 存储。
