## 第三章 使用Promises实现同步

Promises的实现已经在JavaScript库中存在了多年。这一切都始于Promises/A+规范。这些类库库实现了他们自己的规范变体，

直到最近（确切地说是ES6），Promises规范才被JavaScript语言纳入。如章节标题那样 - 他们帮助我们应用同步原则。


在本章中，我们将首先简单介绍Promises中使用的各种术语，以便更容易理解本章的其余部分。然后，通过各种方式，

我们将使用promises来解决目前的一些问题，并在处理并发时更轻松。准备好了吗？


### Promise相关术语

在我们深入研究代码示例之前，让我们花一点时间确保我们牢牢掌握有关Promises的术语。有Promise实例，但是还有各种各样的状态

和方法。如果我们能够弄清楚Promise定义，那么后面的章节会更有意义。这些解释简短而甜蜜，所以如果您已经使用过Promises，

您可以快速浏览这些定义，以便检查您的知识。


### Promise

顾名思义，Promise是一种承诺。将Promise视为尚不存在的值的代理。Promise让我们编写更好的并发代码，因为我们知道值会

在某个时刻存在，并且我们不必编写大量的重复的状态检查代码。


### State

Promise总是处于以下三种状态之一：

• 等待：这是Promise创建后的第一个状态。它一直处于等待状态，直到它完成或被拒绝。

• 完成：该Promise值已经处理得到，并为它提供给了then()回调函数。

• 拒绝：处理Promise的值出了问题。目前没有数据。

Promise状态的一个有趣特性是它们只转换一次。他们要么从等待状态到完成，要么等待状态到拒绝。一旦他们进行了这种状态

转换，后面他们就会锁定在这种状态。


### Executor

执行程序函数负责以某种方式解析该值将处于等待状态。创建Promise后立即调用此函数。它需要两个参数：解析器函数和拒绝函数。


### Resolver

解析器是一个作为参数传递给执行器函数的函数。实际上，这非常方便，因为我们可以将解析器函数传递给另一个函数，依此类推。

调用解析器函数的位置并不重要，但是当它被调用时，Promise会进入一个完成状态。状态的这种改变将触发任何then()回调 - 我们

将很快看到这些回调。


### Rejector

在拒绝函数是相似的解析器。它是传递给executor 函数的第二个参数，可以从任何地方调用。当它被调用时，它从等待状态改变到

拒绝状态。这种状态的改变将调用错误回调函数，如果有的话，传递给then()或catch()。


### Thenable

如果对象具有接受完成回调和拒绝回调作为参数的then()方法，则该对象就是Thenable。换句话说，Promise是Thenable。

但是在某些情况下，我们可能希望实现专门的语义解析。


## 完成和拒绝Promises

如果上一节刚刚介绍了几个听起来令人困惑的新术语，那就别担心了。从本节开始，我们将看到所有这些Promises术语在实践中的应用。

在这里，我们将实现一些简单的Promise解决和拒绝。


### 完成Promises

解析器是一个函数，顾名思义，它完成了我们的Promise。这不是完成Promise的唯一方法 - 我们将在后面探索更高级的方式。

但到目前为止，这种方法是最常见的。它作为第一个参数传递给执行函数。这意味着执行者可以通过简单地调用

解析器直接完成Promise。但这不会为我们提供很多实用性，是吗？


更大程度上的常见情况是promise执行器函数设置即将发生的异步操作 - 例如拨打网络电话。然后，在这些异步操作的

回调函数中，我们可以完成这个Promise。这有点违反直觉，在我们的代码中传递一个解析函数，但是一旦我们开始使用它们

就会发现很有意义。


一个解析函数是一个相对Promise来说难懂的函数。它只能完成一次Promise。我们可以调用解析器很多次，但只在

第一个调用会改变Promise的状态。下面是一个图描绘了Promise的可能状态;它还显示了状态之间是如何变化的：

![image060.gif](image060.gif)


现在，我们来看看一些Promise代码。在这里，我们将完成一个promise，它会调用then()完成回调函数：

```javascript
//我们的承诺使用的执行函数。
//第一个参数是解析器函数，在1秒内调用以完成Promise。
function executor(esolve) {
	setTimeout(resolve, 1000);
}

//我们Promise的回调函数。
//这个简单地停止那个完整的计时器，在我们的执行程序函数运行后启动。
function fulfilled() {
	console.timeEnd('fullfillment');
}

//创建将promise执行程序并立即运行，
//然后启动一个计时器来查看调用完成函数需要多长时间
var promise = new Promise(executor);
promise.then(fullfilled);
console.time('fullfillment');
```

我们可以看到，调用解析器函数时会调用fulfilled()函数。执行程序实际上并不调用解析程序。相反，它将解析器函数传递

给另一个异步函数setTimeout()。执行程序函数本身不是我们试图纠缠的异步代码。可以将执行程序视为一种协调程序，

编排异步操作以确定何时执行Promise。


前面的示例未解析任何值。当某个操作的调用者需要确认它成功或失败时，这是一个有效的用例。相反，让我们这次尝试解析

一个值，如下所示：

```javascript
//我们的Promise使用的执行函数。
//设置延时一秒钟调用"resolve()"
//创建Promise后，返回一个字符串值 - “完成！”。
function executor(resolve) {
	setTimeout(() => {
		resolve('done!');
	}, 1000);
}

//我们Promise的履行回调接受一个值参数。
//这个值将传递到完成解析器
function fulfilled(value) {
	console.log('resolved', value);
}

//创建我们的Promise，提供执行者和完成回调函数。
var promise = new Promise(executor);
promise.then(fulfilled);
```

我们可以看到这段代码与前面的例子非常相似。区别在于我们的解析器函数实际上是在传递给setTimeout()的回调函数的闭包内

调用的。这是因为我们正在解析一个字符串值。还有一个将被解析的值参数传递给我们的fulfilled()函数。


### 拒绝promises

Promise执行器函数并不总是按计划进行，当发生这种情况时，我们需要拒绝promise。这是从等待状态转换到的另一个可能的状态。

这不是一个进入完成状态而是进入一个被拒绝的状态。这会导致执行不同的回调，与完成回调函数分开。值得庆幸的是，拒绝Promise的

机制与完成Promise非常相似。我们来看看这是如何实现的：

```javascript
//此执行函数在延时一秒后拒绝Promise
//它使用拒绝回调函数来改变状态，
//并提供拒绝的参数值到回调函数。
function executor(resolve, reject) {
	setTimeout(() => {
		reject('Failed');
	}，1000);
}

//用作拒绝回调的函数。
//它期望提供拒绝的参数值。
function rejected(reason) {
	console.error(reason);
}

//创建promise，并运行执行程序。
//使用“catch()”方法来接收拒绝回调函数。
var promise = new Promise(executor);
promise.catch(rejected);
```

这段代码看起来非常熟悉我们在上一节中看到的解析代码。我们设置了超时，而不是完成函数，我们拒绝函数。这是使用

rejector函数完成的，并作为第二个参数传递给执行程序。


我们使用catch()方法而不是then()方法来设置拒绝回调函数。我们将在本章后面看看then()方法如何用于处理完成和

拒绝回调。此示例中的拒绝回调仅将失败原因记录为错误。提供此返回值很重要。当我们完成promise时，value是常见的，

尽管不是绝对必要的。另一方面，对于拒绝，即使回调仅记录错误，也没有提供拒绝原因的可行情况。


让我们看一下另一个例子，它捕获执行器中的异常，并为拒绝的回调函数提供更有意义的错误解释：

```javascript
//此promise执行程序抛出错误，
//并调用拒绝回调函数作为结果。
new Promise(() => {
	throw new Error('Problem executing promise');
}).catch((reason) => {
	console.error(reason);
});

//此promise执行程序捕获错误，
//并带有更有用的信息
new Promise((resolv, reject) => {
	try{
		var size = this.name.length;
	} catch(error) {
		reject(error instanceof TypeError ? 'Missing "name" property' : error);
	}
}).catch((reason) => {
	console.error(reason);
});
```

前一个例子中第一个承诺的有趣之处在于它确实改变了状态，即使我们没有使用resolve()或reject()明确地改变

promise的状态。然而，最终改变promise的状态很重要; 我们将在下一节中探讨这个主题。


### 空Promises

尽管对这一事实的执行函数传递一个完成回调函数和拒绝回调函数功能，从未有任何这样的保证的承诺，将改变状态。

在这种情况下，该承诺只是挂起，并没有在完成回调也没有拒绝回调被触发。这可能看起来不像一个问题，而在事实上，

简单的promises，很容易给诊断和修复这些没有响应的promise。然而，随着我们进入更复杂的场景后，一个promise的完成回调

可以作为几个其他promise回调结果。如果一个这些承诺不能完成或拒绝，然后将整个流程分崩离析。这个场景调试是非常耗时的;

在下面的图是可以很清楚的看到这个情况：

![image061.gif](image061.gif)


在图中，我们可以看到哪个promise导致依赖的promise挂起，但审查代码来解决这个问题并不方便。现在让我们看一下导致

promise挂起的执行函数：

```javascript
//这个promise能够运行执行程序
//功能没有问题。“then()”回调函数永远不会被执行
new Promise(() => {
	console.log('executing promise');
}).then(() => {
	console.log('never called');
});

//此时，我们不知道promise有什么错了
console.log（'finished executing, promise hangs'）;
```

但是，如果有一种更安全的方式来应对这种不确定性呢？在我们的代码中，我们不需要能够无限期挂起而无需完成或拒绝的

执行函数。让我们看一下执行一个执行器包装器函数，它通过拒绝需要很长时间才能解决的承诺来充当安全网。

这将揭开不好解决的promise场景的神秘面纱：

```javascript
//promise执行函数的包装器
//在给定的超时后抛出错误。
function executorWrapper(func, timeout) {
	//这是实际调用的函数。
	//它需要解析器函数和拒绝器函数作为参数。
	return function executor(resolve, reject) {
		//设置我们的计时器。
		//当时间到达时，我们可以使用超时消息拒绝promise。
		var timer = setTimeout(() => {
			reject('Promise timed out after $​​ {timeout} MS');
		}, timeout);
		
		//调用我们原来的执行函数包装
		//我们实际上是包装了旋转变压器
		//和拒绝函数也是如此，所以当
		//执行者调用它们，清除定时器。
		func((value) => {
				clearTimeout(timer);
				resolve(value);
			}, (value) => {
				clearTimeout(timer);
				resolve(value);
			});
		};
	}

//这个promise执行者超时，超时
//错误消息传递给被拒绝的回调。
new Promise(executorWrapper((resolve，reject) => {
	setTimeout(() => {
		resolve('done');
	}, 2000);
}, 1000)).catch((reason) => {
	console.error(reason);
}）;

//这个promise自执行者以来按预期结算
//在时间到来之前调用“resolve()”。
new Promise（executorWrapper((resolve，reject)) => {
	setTimeout(() => {
		resolve(true);
	}, 500);
}, 1000)).then((value) => {
	console.log('resolved', value);
});
```

### 对promises作出反应

现在我们已经更好地理解了执行promises的机制，本节将详细介绍如何使用promises来解决特定问题。通常，这意味着当

promises得到完成或被拒绝时，我们会考虑某些目的。


我们将首先查看JavaScript解释器中的任务队列，以及这些对我们的解析回调函数意味着什么。然后，我们将考虑使用

promise的数据，处理错误，创建更好的抽象以响应promises，以及thenables。让我们开始吧。


### 处理任务队列

JavaScript的任务队列的概念在第2章-JavaScript的执行模型中被引入。它的主要职责是初始化新的执行上下文堆栈。

这是主要的任务队列。然而，还有另一种队列，这是专用于执行promises回调的。这意味着，算法承担从任一的的队列中

选择下一个任务执行，如果他们都存在时。


Promise内置了并发语义，并且有充分的理由。如果使用promise来确保最终解析某个值，那么为对其作出反应的代码

赋予高优先级是有意义的。否则，当值到达时，处理它的代码可能必须在其他任务后面等待更长的时间。

让我们编写一些代码来演示这些并发语义：

```javascript
//创建5个promise，记录它们的执行时间，以及当他们对值做出反应时
for(let i = 0; i <5; i++) {
	new Promise((resolve) => {
		console.log('execting promise');
		resolve(i);
	}).then((value) => {
		console.log('resolved', i);
	});
}

//在任何完成之前调用它
//回调，因为这个调用堆栈作业需要
//在解释器进入之前完成
// promise解析回调队列，
//当前正在调用5“then()”回调。
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


> 同样的语义也遵循被拒绝的回调。


### 使用promise的数据

到目前为止，我们已经在本章中看到了一些示例，其中解析器函数完成promise时并返回值。传递给此函数的值是最终传递给

已完成的回调函数的值。想法是让执行程序设置任何异步操作，例如setTimeout()，稍后将使用该值调用解析程序。但在

这些例子中，调用者实际上并没有等待任何值; 我们只使用setTimeout()作为示例异步操作。让我们看一下我们实际上

没有值的情况，异步网络请求需要得到它：

```javascript
//用于获取资源的通用函数
//从服务器返回一个promise。
function get(path) {
	return new Promise((resolve，reject) => {
		var request = new XMLHttpRequest();
		
		//解析时解析了promise
		//加载数据时的JSON数据。
		request.addEventListener('load', (e) => {
			resolve(JSON.parse(e.target.responseText));
		});

		//当请求出错时，
		//承诺被适当的理由拒绝。
		request.addEventListener('error', (e) => {
			reject(e.target.statusText || '未知错误');
		});


		//如果请求被中止，我们只需解决请求
		request.addEventListener('abort', resolve);
		
		request.open('get', path);
		request.send();
	});
}

//我们可以直接附加我们的“then()”处理程序
//到“get()”，因为它返回一个promise。该
//这里使用的值是一个真正的异步操作
//必须获取远程值并解析它，
//在解决之前 
get('api.json').then((value) => {
	console.log('hello', value.hello);
});
```


使用像get()这样的函数，它们不仅始终返回像promise一样的同步类型，而且还封装了一些讨厌的异步细节。在我们的

代码中处理所有地方的XMLHttpRequest对象并不令人愉快。我们还简化了可以返回的各种模式。而不是总是必须为load，

error和abort事件创建处理程序，我们只需要担心一个接口 - promise。这就是同步并发原则的全部内容。


### 错误回调

有两种方法可以对被拒绝的promise做出反应。换句话说，提供错误回调。第一种方法是使用catch()方法，该方法采用单个

回调函数。另一种方法是将被拒绝的回调函数作为then()的第二个参数传递。


该then()这个方法被用来以提供拒绝回调函数是在某些情况下表现的更好，它可能应该被使用而不是catch()。第一个

场景是编写promises和thenable对象可以互换的代码。catch()方法不是thenable必要的一部分。

在第二个场景是，当我们建立回调链时，我们将在这一章稍后探讨。


让我们看一些代码，它们比较了两种为promises提供被拒绝的回调函数的方法：

```javascript
//此承诺执行者将随机解决
//或拒绝承诺
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

//创建一个promise，然后分配错误
//通过“catch()”方法回调。
new Promise(executor).then(log).catch(error);

//创建一个promise，然后分配错误
//通过“then()”方法回调。
new Promise(executor).then(log, error);
```

我们可以看到这两种方法实际上非常相似。在代码美学方面，一方面没有真正的优势。然而，当涉及到使用thenables时，

then()方法有一个优势，我们很快就会看到。但是，由于我们实际上并没有以任何方式使用promise实例，除了添加回调之外，

实际上没有必要担心catch()和then()用于注册错误回调。


### Always响应

Promises最终总是终止于完成状态或拒绝状态。我们通常为每个状态都有不同的回调函数。但是，我们很可能希望为这

两个状态执行一些相同的操作。例如，如果使用promise的组件在promise等待时更改状态，我们将确保在完成或拒绝

promise后清除状态。


我们可以用这样的方式编写代码：完成和拒绝状态的回调每个都自己执行这些操作，或者他们每个都可以调用一些执行

清理的常用函数。这是问题的示意图：

![image065.gif](image065.gif)
 

将清理责任分配给promise是否更有意义，而不是将其分配给个别结果？这样，在解析promise时运行的回调函数关注于

它需要对值执行的操作，而拒绝回调则专注于处理错误。让我们看看是否可以使用always()方法编写一些扩展

promises的代码：

```javascript
//使用“always()”扩展promise原型方法。
//始终会调用给定的函数，
//承诺是履行还是拒绝。
Promise.prototype.always = function(func) {
	return this.then(func, func);
};

//创建随机解析的promise或被拒绝。
var promise = new Promise((resolve，reject) => {
	Math.round(Math.random()) ? 
	resolve('fullfilled')：reject('rejected');
});

//提供promise完成和拒绝回调。
promise.then((valu) => {
	console.log(value);
}, (reason) => {
	console.error(reason);
});

//这个回调总是在之后调用上面的回调。
promise.always((value) => {
	console.log('清理......');
});
```

> 请注意，顺序在这里很重要。如果我们在then()之前调用always()，那么函数仍然会一直运行，但它会在回调提供给
> then()之前运行。我们实际上可以在then()之前和之后调用always()，以便在完成或拒绝的回调之前以及之后运行代码。


### 处理其他promise

到目前为止，我们在本章中看到的大多数promise都是由执行程序函数直接完成的，或者是当值准备完成时从异步操作中调用

解析器的结果。像这样传递回调函数实际上非常灵活。例如，执行程序甚至不必执行任何工作，除了将解析器函数

存储在某处以便稍后调用它来解析promise。


当我们发现自己处于需要多个值的更复杂的同步场景时，这可能特别有用，这些值已经被传递给调用者。如果我们有处理回调

函数，我们就可以处理promise。让我们看看，在存储代码的解析函数的多个promise，使每一个promise可以在稍后处理：

```javascript
//存储一个解析器函数列表。
var resolvers = [];

//在每个执行者中创建5个新的promise,
//解析器被推到了“resolvers”数组。
//我们也可以给每一个promise回调。
for(let i = 0; i <5; i ++) {
	new Promise(() => {
		resolvers.push(resolve);
	}).then((value) => {
		console.log(`resolved $ {i + 1}`, value);
	});
}

//设置一个2s之后延时运行函数，
//当它运行时，我们遍历“解析器”数组中的每一个解析器函数，
//我们用一个返回值来调用它。
setTimeout(() => {
	for(resolver of resolvers) {
		resolver(true);
	}
}, 2000);
```

正如这个例子所表明的那样，我们不必在executor函数本身内解处理任何问题。事实上，我们甚至不需要在创建和设置

执行程序和完成函数之后显式引用promise实例。解析器函数已存储在某处，它包含对promise的引用。


### 类Promise对象

Promise类是一种原生的JavaScript类型。但是，我们并不总是需要创建新的promise实例来实现相同的同步的行为

动作。我们可以使用静态Promise.resolve()方法来解析这些对象。让我们看看如何使用此方法：

```javascript
//“Promise.resolve()”方法可以解决问题
//对象 这是一个带有“then（）”方法的对象
//作为执行者。这个遗嘱执行人将
//随机解决或拒绝承诺。
Promise.resolve({then: (resolve, reject) => {
	Math.round(Math.random()) ? resolve('fulfilled'): reject('rejected');

	//这个方法返回一个promise，所以我们能够
	//设置我们已完成和被拒绝的回调
	//通常。
}}).then((value) => {
	console.log('resolved', value);
}, (reason) => {
	console.error('reason', reason);
});
```

我们将在本章的最后一节中重新讨论Promise.resolve()方法，以了解更多用例。


### 建立回调链

我们在本章中检查的每种promise方法都会返回promise。这允许我们在返回值上再次调用这些方法，从而

产生then().then()调用的链，依此类推。链式promise具有挑战性的一个方面是promise方法返回的实例是新实例。

也就是说，我们将在本节中探讨promise一定程度的不变性。


随着我们的应用程序变得越来越大，并发性挑战随之增长。这意味着我们需要考虑更好的方法来利用原生同步语法，

例如promises。正如JavaScript中的任何其他原始值一样，我们可以将它们从函数传递给函数。我们必须以同样的方式

处理promises - 传递它们，并建立在回调函数链上。


### Promises只改变状态一次

Promise诞生于等待状态，并且它们结束于已解决或被拒绝的状态。一旦promise转变为其中一种状态，它们就会陷入这种状态。

这有两个有趣的副作用。首先，多次尝试解决或拒绝promise将被忽略。换句话说，解决器和拒绝器是幂等的 - 只有第一次调用

对promise有影响。让我们看看这代码如何执行：

```javascript
//此执行程序函数尝试解析
//promise两次，但完成的回调是
//只调用一次
new Promise((resolve, reject) => {
	resolve('fulfilled');
	resolve('fulfilled');
}).then((value) => {
	console.log('then', value);
});

//这个执行器函数试图拒绝
//promise两次，但被拒绝的回调是
//只调用一次
new Promise((resolve, reject) => {
	reject('rejected');
	reject('rejected');
}).catch((reason) => {
	console.error('reason');
});
```


promise仅改变状态一次的另一个含义是promise可以在添加完成或拒绝回调之前处理。竞争条件，例如这个，是并发编程的残酷

现实。通常，回调函数会在创建时添加到promise中。由于JavaScript是运行即完成的，因此在添加回调之前，不会处理promise

解析回调的任务队列。但是，如果promise立即在执行中处理怎么办？如果将回调添加到另一个JavaScript执行上下文中的

promise中会怎样？让我们看看是否可以用一些代码来更好地说明这些情况：

```javascript
//此执行程序函数立即解析promise。添加“then()”回调时，
//promise已经解决了。但仍然会使用已解析的值调用回调。
new Promise((resolve，reject) => {
	resolve('done');
	console.log('executor', 'resolved');
}).then((value) => {
	console.log('then', value);
});

//创建一个立即解决的新promise执行函数。
var promise = new Promise((resolve，reject) => {
	resolve('done');
	console.log('executor', '已解决');
});

//这个回调是自promise立即运行后已经解决了。
promise.then((value) => {
	console.log('then 1', value);
});

//此回调未添加到另一个的promise中
//解决之后的第二个问题。它仍然被称为
//立即使用已解析的值。
setTimeout(() => {
	promise.then((value) => {
		console.log('then 2', value);
	});
}, 1000);
```

此代码说明了promises的一个非常重要的属性。无论何时将执行回调添加到promise中，无论是处于暂时挂起状态还是执行状态，使用

promise的代码都不会更改。从表面上看，这似乎不是什么大不了的事。但是这种类型的竞争条件检查需要更多的并发代码来

保护我们自己。相反，Promise原生语法为我们处理这个问题，我们可以开始将异步值视为基本类型。


### 不可改变的promises

promises并非真正不可改变。它们改变状态，then()方法将回调函数添加到promise。但是，有一些不可改变的promises特征值得

在这里讨论，因为它们会在某些情况下影响我们的promise代码。


从技术上讲，then()方法实际上并没有改变promise对象。它创建了所谓的promise能力，它是一个引用promise的内部JavaScript

记录，以及我们添加的功能。因此，它不是JavaScript术语中的真正语法。

这是一张图，说明当我们链接两个或更多then()一起调用时会发生什么：

![image069.gif](image069.gif)


我们可以看到，then()方法不会返回与上下文一起调用的相同实例。相反，then()创建一个新的promise实例并返回它。

让我们看看一些代码，以便更仔细地检查当我们使用then()将promises链接在一起时会发生什么：

```javascript
//创建一个立即解决的promise，
//并且存储在“promise1”中。
var promise1 = new Promise((resolve, reject) => {
	resolve('fulfilled');
});

//使用“promise1”的“then()”方法创建一个
//新的promise实例，存储在“promise2”中。
var promise2 = promise1.then((value) => {
	console.log('then 1', value);
	//→然后1完成
});

//为“promise2”创建一个“then()”回调。这实际上
//创建第三个promise实例，但我们不用它做任何事情。
promise2.then((value) => {
	console.log('then 2', value);
	//→然后2未定义
});

//确保“promise1”和“promise2”实际上是不同的对象
console.log('equal', promise1 === promise2);
//→等于false
```

我们可以清楚地看到这两个创建promise的实例在这个例子中是独立的promise对象。值得指出的是第二个

promise执行前时，一定是它执行了第一个promise。但是，我们可以看到的是该值不会传递到第二个promise。

我们将在下一节中解决此问题。


### 多少个then()回调，多少个promises对象

正如我们在上一节中看到的那样，使用then()创建的promise将绑定到它们的创建者。也就是说，当第一个promise完成时，

绑定它的promise也会完成，依此类推。但是，我们也发现了一个小问题。已解析的值不会使其传递到第一个回调函数。这样做的

原因是为响应promise解析而运行的每个回调都是第一个回调的返回值被送入第二个回调，依此类推。我们的第一个回调

将值作为参数的原因是因为这在promise机制中显然会发生的。


我们来看看另一个promise链示例。这一次，我们将显式返回回调函数中的值：

```javascript
//创建一个随机解析或被拒绝的新promis。
new Promise((resolve, reject) => {
	Math.round(Math.random()) ?
	resolve('fulfilled') : reject('rejected');
}).then((value) => {
	//在解决原始promise时调用返回值，
	//以防另一个promise链接到这一个。
	console.log('then 1', value); 
	return value;
}).catch((reason) => {
	//被称为第二个promise的链接
	//什么时候被拒绝了
	console.error('catch 1', reason);
}).then((value) => {
	//链接到第三个promise，得到了
	//按预期值，并为任何返回值
	//下游promise回调使用。
	console.log('then 2', value);
	return value;
}).catch((reason) => {
	//这绝不会被调用 - 拒绝回调不会
	//通过promise链扩散。
	console.error('catch 2', reason);
});
```

这看起来有希望。现在我们可以看到已解析的值通过promise链。有一个问题 - 拒绝不是累积的。相反，只有链中的第一个promise

才被拒绝。其余的promise只是解决，而不是拒绝。这意味着最后一个catch()回调永远不会运行。


当我们以这种方式将promise链接在一起时，我们的执行回调函数需要能够处理错误条件。例如，已解析的值可能具有error属性，

可以检查其具体细节。


### promises传递

在本节中，我们讲讲promise视为原始值的想法。我们经常用原始值做的事情是将它们作为参数传递给函数，并从函数中返回

它们。promise和其他原生语法之间的关键区别在于我们如何使用它们。其他值现在一直都存在，而promise的值到最后才存在。

因此，我们需要通过回调函数定义一些操作过程，以便在值到达时执行。


promises的好处是用于提供这些回调函数的接口很小且一致。当我们可以将值与将作用于它的代码耦合时，我们不需要动态

地发明同步机制。这些单元可以像任何其他值一样在我们的应用程序中移动，并且并发语义是不引人注目的。这是一个

promise相互传递的几个函数的示例：

![image071.gif](image071.gif)


在这个函数调用堆栈结束时，我们有一个promise对象，反思了几个promise的解析。整个决议链是由第一个promise解决

而开始的。比如何遍历promise链的机制​​更重要的是所有这些函数都可以自由使用这个promise传递的值而不影响其他函数。


在这里有两个并发原则。首先，我们将通过执行异步操作来保存该值仅一次; 每个回调函数都可以自由使用此解析值。

其次，我们在抽象同步机制方面做得很好。换句话说，代码并不觉得它带有重复模板并发代码。让我们看看传递promise的代码

实际上是什么样的：

```javascript
//较小的功能可以组成更大的函数，简单实用
function compose(...funcs) {
	return function(value) {
		var result = value;
		
		for(let func of funcs) {
			result = func(value);
		}
		return result;
	};
}

//接受promise或已解决的值。如果这是一个promise，
//它添加了一个“then()”回调并返回一个新的promise。
//否则，它会执行“更新”并返回值
function updateFirstName(value) {
	if (value instanceof Promise) {
		return value.then(updateFirstName);
	}

	console.log('first name', value.first); 
	return value;
}

//与上面的函数一样工作，除了它
//执行不同的UI“更新”。
function updateLastName(value) {
	if (value instanceof Promise) {
		return value.then(updateLastName);
	} 

	console.log('last name', value.last); 
	return value;
}

//与上面的函数一样工作，除了它
//执行不同的UI“更新”。
function updateAge(value) {
	if (value instanceof Promise) {
		return value.then(updateAge);
	}

	console.log('age', value.age);
	return value;
}

//延时一秒钟之后
//使用数据对象解析的promise对象
var promise = new Promise((resolve, reject) => {
	setTimeout(() => {
		resolve({
			first: 'John',
			last: 'Smith',
			age: 37
		});
	});
}, 1000);

//我们组装一个“update()”函数来更新
//各种UI组件。
var update = compose(
	updateFirstName，
	updateLastName，
	updateAge
);

//使用promise调用我们的更新函数。
update(promise);
```

这里的关键函数是我们的更新函数 - updateFirstName()，updateLastName()和updateAge()。他们非常灵活，接受一个

promise或promise返回值。如果这些函数中的任何一个将promise作为参数，它们会通过添加then()回调函数来返回新的promise。

请注意，它添加了相同的功能。updateFirstName()将添加updateFirstName()作为回调。当回调触发时，它将与此次用于

更新UI的普通对象一起使用。因此，promise检查失败，我们可以继续更新UI。


promise检查每个函数都需要三行，这并不是非常突兀的。最终结果是易于阅读的灵活代码。顺序无关紧要; 我们可以用不同的顺序包装

我们的update()函数，并且UI组件都将以相同的方式更新。我们可以将普通对象直接传递给update()，一切都会起作用。看起来

不像并发代码的并发代码是我们在这里取得的重大胜利。


### 同步多个promises

在本章前面，我们已经探究了单个promise实例，它解析了一个值，触发了回调，并可能传递给其他promises处理。在本节中，

我们将介绍几种静态Promise方法，它们可以帮助我们在需要同步多个promise值的情况下处理。


首先，我们将处理我们开发的组件需要同步访问多个异步资源的情况。然后，我们将看一下不常见的情况，即异步操作在

处理之前由于UI中发生的事件而变得无关紧要。


### 等待promises

在我们等待处理多个promise的情况下，也许是将多个数据源转换为UI组件可以使用的东西，我们可以使用Promise.all()方法。

它将promise实例的集合作为输入，并返回一个新的promise实例。仅当解析了所有输入的promise时，才会解析此新实例。


then()函数是我们为Promise提供的创建新promise的回调。给出一组解析值作为输入。这些值对应于索引位置方面的

输入promise。这是一个非常强大的同步机制，它可以帮助我们实现同步并发原则，因为它隐藏了所有的记录。


我们有一个回调，而不是几个回调，每个回调都需要协调它们所绑定的promise状态，它具有我们需要的所有已解析数据。

这是一个展示如何同步多个promise的示例：

```javascript
//用于发送“GET”HTTP请求的实用程序，
//并返回使用已解析的响应解析的promise。
function get(path) {
	return new Promise((resolve, reject) => {
		var request = new XMLHttpRequest();
		//解析时解析了promise
		//加载数据时的JSON数据。
		request.addEventListener('load', (e) => {
			resolve(JSON.parse(e.target.responseText));
		});

		//当请求出错时，
		//promise被适当的理由拒绝。
		request.addEventListener('error', (e) => {
			reject(e.target.statusText || '未知错误');
		});

		//如果请求被中止，我们只需处理请求 
		request.addEventListener('abort', resolve);
		
		request.open('get', path);
		request.send();
	});
}


//我们的请求promises。
var requests = [];

//发出5个API请求，并将相应的5个
//promise放在“requests”数组中。
for(let i = 0; i <5; i++) {
	requests.push(get('api.json'));
}

//使用“Promise.all()”让我们传入一个数组promises
//当所有promise解决时，返回一个已经解决的新promise
//我们的回调得到一个数组对应于promises的已解析值。
Promise.all(requests).then((values) => {
	console.log('first', values.map(x => x[0])); 
	console.log('second', values.map(x => x[1]));
});
```


### 取消promises

我们在本书中到目前为止看到的XHR请求具有中止请求的处理程序。这是因为我们可以手动中止请求并阻止任何回调函数运行。

需要此功能的典型场景是用户单击取消按钮，或导航到应用程序的其他部分，从而使请求变得多余。


如果我们是要在抽象promise上更上一层楼，在同样的原则也适用。而一些可能发生的并发操作的执行呈现的promise毫无意义。

该promise和XHR请求的过程中之间的差异，是前者没有abort()方法。我们要做的最后一件事是在我们的promise回调中

开始引入不必要的取消逻辑。


这是Promise.race()方法可以帮助我们的地方。顾名思义，该方法返回一个新的promise，它由第一个要解析的输入promise

决定。这可能听起来不多，但实现Promise.race()的逻辑并不容易。它是同步原则，隐藏了应用程序代码中的并发复杂性。

我们来看看这个方法是怎么做的可以帮助我们处理因用户互动而取消的promise：

```javascript
//用于取消数据请求的解析器​​函数。
var cancelResolver;

//一个简单的“常量”值，用于解决取消promise
var CANCELED = {};

//我们的UI组件
var buttonLoad = document.querySelector('button.load'),
	buttonCancel = document.querySelector('button.cancel');

//请求数据，返回一个promise。
function getDataPromise() {
	//创建取消承诺。遗嘱执行人分配
	//“解析”函数为“cancelResolver”，所以
	//稍后可以调用它。
	var cancelPromise = new Promise((resolve) => {
		cancelResolver = resolve;
	});

	//我们想要的实际数据 这通常是
	//一个HTTP请求，但我们在这里模拟一个
	//使用setTimeout()简洁。
	var dataPromise = new Promise((resolve) => {
		setTimeout(() => {
			resolve({hello: 'world'});
		}, 3000);
	});

	//“Promise.race()”方法返回一个新的promise，
	//并且无论输入承诺是什么，它都可以解决
	//先解决 
	return Promise.race([cancelPromise, dataPromise]);
}

//单击取消按钮时，我们使用
//“cancelResolver()”函数来解决
//取消承诺
buttonCancel.addEventListener('click', () => {
	cancelResolver('CANCELLED');
});

//单击加载按钮时，我们发出请求
//使用“getDataPromise()”获取数据。
buttonLoad.addEventListener('click', () => {
	buttonLoad.disabled = true;
	getDataPromise().then(value) => {
		buttonLoad.disabled = false;
		//承诺得到了解决，但那是因为
		//用户取消了请求。所以我们退出
		//这里通过返回CANCELED“常量”。
		//否则，我们有数据可以使用。
		if (Object.is(value, CANCELED)) {
			return value;
		}
	});
});

console.log('loaded data', value);
```

> 作为练习，尝试想象一个更复杂的场景，其中dataPromise是由Promise.all()创建的承诺。我们的cancelResolver()

> 函数可以一次无缝地取消许多复杂的异步操作。


### 没有执行函数的promises

在最后一节中，我们将介绍Promise.resolve()和Promise.reject()方法。我们已经本章前面看到Promise.resolve()如何

处理可用对象。它还可以直接处理值或其他promise。当我们实现一个可能同步和异步的函数时，这些方法会派上用场。这不是

我们想要使用并具有模糊并发语义函数的情况。


例如，这是一个同步和异步的函数，导致混淆，几乎肯定会在以后出现错误：

```javascript
//返回“value”的示例函数
//缓存，或异步“获取”它。
function getData(value) {
	//如果它存在于缓存中，我们返回这个值
	var index = getData.cache.indexOf(value);
	if(index > -1) {
		return getData.cache[index];
	}

	//否则，我们必须“取”它。这个
	//“resolve()”调用通常会在
	//网络请求回调函数 
	return new Promise((resolve) => {
		getData.cache.push(value);
		resolve(value);
	});
}

//创建缓存。
getData.cache = [];
console.log('getting foo', getData('foo'));

console.log('getting bar', getData('bar'));
//→获取foo Promise
console.log('getting foo', getData('foo'));
//→获取bar Promise
//→得到foo foo
```

我们可以看到最后一次调用返回一个缓存值，而不是一个promise。这具有直观意义，因为我们不承诺最终的价值，我们已经

拥有它！问题是我们暴露了使用我们的getData()函数的任何代码的不一致性。也就是说，调用getData()的代码需要

处理并发语义。此代码不是并发的。让我们通过引入Promise.resolve()来改变它：

```javascript
//返回“value”的示例函数
//缓存，或异步“获取”它。
function getData（value）{
	var cache = getData.cache;
	//如果这个函数没有缓存，那就试试吧
	//拒绝承诺。要有缓存。
	if(!Array.isArray(cache)) {
		return Promise.reject('missing cache');
	}

	//如果它存在于缓存中，我们返回
	//使用the解决的承诺
	//缓存的值
	var index = getData.cache.indexOf(value);
	
	if (index > -1) {
		return Promise.resolve(getData.cache[index]);
	}

	//否则，我们必须“取”它。这个
	//“resolve（）”调用通常会在
	//网络请求回调函数 
	return new Promise((resolve) => {
		getData.cache.push(value);
		resolve(value);
	});
}

//创建缓存。
getData.cache = [];

//每次调用“getData（）”都是一致的。甚至
//当使用同步值时，它们仍然存在
//得到解决的承诺。
getData('foo').then((value) => {
	console.log('getting foo', `“$ {value}”`);
}, (reason) => {
	console.error(reason);
});

getData('bar').then((value) => {
	console.log('getting bar', `“$ {value}”`);
}, (reason) => {
	console.error(reason);
});

getData('foo').then((value) => {
	console.log('getting foo', `“$ {value}”`);
}, (reason) => {
	console.error(reason);
});
```

这个更好。使用Promise.resolve()和Promise.reject()，任何使用getData()的代码都会默认获得并发，即使数据

获取操作是同步的。


### 小结

本章介绍了ES6中引入的Promise对象的大量细节，以帮助JavaScript程序员处理困扰该语言多年的同步问题。

随着异步回调的使用 - 大量的回调。这会产生一个我们想要不惜一切代价避免的回调地狱。


Promise通过实现一个足以解决任何值的通用接口来帮助我们处理同步问题。承诺总是处于三种状态之一 - 等待，完成或拒绝，

并且它们只改变一次状态。当这些状态发生改变时，将触发回调。promise有一个执行程序函数，其作用是设置使用promise的异步

操作解析器或拒绝器来改变promise的状态。


promise带来的大部分价值在于它们如何帮助我们简化复杂的场景。因为，如果我们只需处理一个运行带有已解析值回调的异步操作，

那么Promises几乎不值得。


这不是常见的情况。常见的情况是几个异步操作，每个操作都解析值;并且这些值需要同步和转换。Promise有方法允许我们这样做，

因此，我们能够更好地将同步并发原则应用于我们的代码。


在下一章中，我们将介绍另一个新引入的语法 - 生成器。与promises类似，生成器是帮助我们应用并发原则的机制 - 保存。
