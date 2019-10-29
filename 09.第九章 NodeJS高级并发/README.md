## 第九章 NodeJS高级并发

在“第8章，NodeJS的Evented IO”，您了解了NodeJS应用程序的核心并发机制 - IO事件循环。在本章中，我们将深入探讨一些更高级的主题，它们与事件循环互补，与事件循环相反。


让我们开始讨论使用Co库在Node中实现协程。接下来，我们将介绍如何创建子进程以及与这些进程之间的通信。在此之后，我们将深入研究Node的内置功能，以创建一个进程集群，每个进程集群都有自己的事件循环。我们以了解如何在大规模的Node服务器集群中创建集群来结束本章。


### 使用Co中的协程

在“第4章，使用Generators实现惰性计算”中，我们已经看到了使用generator在前端实现协程的一种方法。在本节中，我们将使用Co库(`https://github.com/tj/co`)来实现协程。该库也是依赖于generator和promise。


我们首先介绍Co的一般前提，然后，我们将编写一些使用promises等待异步值的代码。然后，我们将研究将已解析值从promise转移到我们的协程函数，异步依赖关系以及创建协程工具函数的机制。


### 生成promises

Co库的核心是使用co()函数来创建协程。事实上，它的基本用法看起来比我们在本书前面创建的协程函数很相似。这是它的示例：

```javascript
co(function* () {
	// TODO：co-routine amazeballs。
});
```

Co库与我们早期的协程实现之间的另一个相似之处是值通过yield语句传递。但是，此协程使用promises传入值，而不是调用返回的函数来传递值。效果是相同的 - 异步值被传递到同步代码中，如下图所示：

![image175.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image175.gif?raw=true)


异步值实际上来自一个promise。已解析的值进入协程。我们将深入探讨其工作的机制。即使我们没有生成promise，比如说我们生成了一个字符串，Co库也会把它包装成我们的promise。但是，这样做会破坏在同步代码中使用异步值的目的。


> 当程序员找到像Co这样的工具来封装凌乱的同步语法时，不能低估它对我们的价值。我们在协程中的代码是同步和可维护的。


### 等待生成值

由co()函数创建的协程与ES7异步函数非常相似。async关键字标记的函数，这意味着它使用异步值。与promise一起使用的await关键字会暂停函数的执行，直到值完成解析。如果这感觉很像generator的作用，这是因为它正是generator所做的。这是ES7语法的示例：

```javascript
//这是ES7语法，其中函数标记为“async”。
//然后，“await”调用暂停执行，直到他们的操作解决完成。
(async function() {
	var result;
	result = await Promise.resolve('hello');
	console.log('async result', `"${result}"`);
	//→async result "hello"

	result = await Promise.resolve('world');
	console.log('async result', `"$ {result}"`);
	//→async result "world"
}());
```

在这个例子中，promises立即被完成，因此没有必要暂停执行。但是，即使promise解析了网络请求，它也会等待请求需要几秒钟。我们将在下一节中更深入地解决promise问题。鉴于这是ES7语法，如果我们今天可以使用相同的方法，那就太好了。以下是我们用Co实现同样的事情的示例：

```javascript
//我们需要“co()”函数。
var co = require('co');

// ES7和“co()”之间的差异是微妙的，
//整体结构是一样的。
//该函数是一个生成器，我们通过yield生成器暂停执行。
co(function* () {
	var result;

	result = yield Promise.resolve('hello');
	console.log('co result', `"${result}"`);
	//→co result "hello"

	result = yield Promise.resolve('world');
	console.log('co result', `"${result}"`);
	//→co result "world"
});
```

Co库向ES7方向发展应该不足为奇了;Co作者的动作很快。


### 解析值

在一个给定的Co协程中至少有两个地方可以解析promise。首先，我们将传递给co()的生成器函数中产生一个或多个promise。如果在这个函数中没有任何promise，那么使用Co就没有多大意义了。调用co()时的返回的是另一个promise，这很酷，因为它意味着协程可以将其他协程作为依赖项。我们将更加深入地探究这个方法。现在，让我们看一下解析promise，以及它是如何完成的。下图是协程的promise解析顺序的说明：

![image179.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image179.gif?raw=true)


promise的解析顺序与它们的名称相同。例如，第一个promise会导致协程中的代码暂停执行，直到它的值被解析为止。然后，在等待第二个promise时再次暂停执行。从co()返回的最终promise将使用生成器函数的返回值进行解析。我们现在来看一些代码：

```javascript
var co = require('co');

co(function* () {

	//这里产生的promise直到1秒后才得到解析。
	//这是yield声明的时候返回已解析的值。
	var first = yield new Promise((resolve, reject) => {
		setTimeout(() => {
			resolve(['First1', 'First2', 'First3']);
		}, 1000);
	});

	//这里有同样的方法，
	//除了在“second”变量获得它的值之前让我们等待2秒。
	var second = yield new Promise((resolve, reject) => {
		setTimeout(() => {
			resolve(['Second1', 'Second2', 'Second3']);
		}, 2000);
	});

	//此处解析了“first”和“second”两个问题，
	//所以我们可以使用它们来映射一个新数组。
	return first.map((v, i) => [v, second[i]]);
}).then((value) => {
	console.log('zipped', value);
	//→
	// [
	// ['First1', 'Second1' ],
	// ['First2', 'Second2' ],
	// ['First3', 'Second3' ]
	// ]
});
```

正如我们所看到的，生成器的返回值最终作为已解析的promise值。回想一下，从generator返回的对象，类似yield返回的value和done属性。Co知道使用value属性解析promise 。


### 异步依赖

当一个动作依赖于协程中稍后的异步值时，用Co制作的协程真的很不错。否则，将只是回调和状态的混乱而不是按正确的顺序分配。在解析值之前，从不会调用依赖操作。下图是该方法的说明：

![image180.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image180.gif?raw=true)


现在，让我们写一些代码有两个异步操作，其中第二个操作取决于第一个的结果。即使使用promise，这也很棘手：

```javascript
var co = require('co');

//一个简单的用户集合。
var users = [
	{name: 'User1'},
	{name: 'User2'},
	{name: 'User3'},
	{name: 'User4'}
];

co(function* () {

	//“userID”值是异步的，并且是执行
	//暂停在这个yield语句中直到promise解析完成。
	var userID = yield new Promise((resolve, reject) => {
		setTimeout(() => {
			resolve(1);
		}, 1000);
	});


	//此时，我们有一个“userID”值。这个
	//嵌套的协同例程将基于这个ID查找用户。
	//我们这样嵌套协同程序是因为
	//“co()”返回一个promise。
	var user = yield co(function* (id) {
		let user = yield new Promise((resolve, reject) => {
			setTimeout(() => {
				resolve(users[id]);
			}, 1000);
		});

		//“co()”promise用“user”值解析完成。
		return user;
	}, userID);

});


console.log(user);
//→{name: 'User2'}
```

我们在这个例子中使用了一个嵌套的协程，但它可能是任何需要参数并返回一个promise的函数。这个例子，如果没有别的，用于突显并发环境中的promises的多功能性。


### 包装协程

我们将看到的最后一个Co示例使用wrap()工具函数将普通协程函数转换为可重用的函数，我们可以反复调用它。顾名思义，协程只是包含在一个函数中。当我们将参数传递给协程时，这尤其有用。我们来看看构建的代码示例的修改版本：

```javascript
var co = require('co');

//一个简单的用户集合。
var users = [
	{name: 'User1'},
	{name: 'User2'},
	{name: 'User3'},
	{name: 'User4'}
];

//无论什么时候调用，“getUser()”函数将创建一个新的协程，
//同时转发任何参数。
var getUser = co.wrap(function* (id) {
	let user = yield new Promise((resolve, reject) => {
		setTimeout(() => {
			resolve(users[id]);
		}, 1000);
	});
	
	//“co()”promise使用“user”值完成解析。
	return user;
});

co(function* () {
	
	//“userID”值是异步的，并且执行暂停在
	//这个yield语句中直到promise解析完成。
	var userID = yield new Promise((resolve, reject) => {
		setTimeout(() => {
			resolve(1);
		}, 1000);
	});

	//我们有一个函数，现在可以在其他地方使用，
	//而不是嵌套的协程。
	var user = yield getUser(userID);
	
	console.log(user);
	//→{name：'User2'}
});
```

因此，我们使用co.wrap()来创建可重用的协程函数，而不是嵌套协程。也就是说，它会在每次调用时创建一个新的协程，并传递函数获取的所有参数。实际上没有比这更多的东西，但收益是显而易见的并且值得的。我们有一些可以跨组件共享的东西，而不是嵌套的协程函数。


### 子进程

我们知道NodeJS使用一个事件IO循环作为其主要的并发机制。这是基于我们的应用程序执行大量IO和非常少的CPU密集型工作的假设。对于我们代码中的大多数处理程序，这可能是正确的。但是，总有一个特殊的边缘情况需要比平常更多的CPU时间。


在本节中，我们将讨论处理程序如何阻止IO循环，以及为什么只需要一个糟糕的处理程序来破坏其他人的体验。然后，我们将通过分配新的Node子进程来研究解决此限制的方法。我们还将研究如何生成其他非Node进程以获取我们需要的数据。


### 阻止事件循环

在“第8章，NodeJS中的Evented IO”中，我们看到了一个示例，演示了一个处理程序在执行开销大的CPU操作时如何阻止整个IO事件循环。我们在此重申这一点，以突出问题的全部范围。它不仅仅是我们阻止的一个处理程序，而是所有处理程序。这可能是数百，或者可能是数千，取决于应用程序，以及如何使用它。


由于我们不是在硬件级别并行处理请求，这是多线程方法的情况 - 只需要一个开销大的处理程序来阻止所有处理程序。如果有一个请求能够导致这个开销大的处理程序运行，那么我们可能会收到多个这样开销大的请求，使我们的应用处于停滞状态。让我们看一个处理程序，它阻止其后的所有其他处理程序：

```javascript
//消耗一些CPU周期......
//源自http://adambom.github.io/parallel.js/
function work(n) {
	var i = 0;
	while (++i < n * n) {}
	return i;
}

//将一些函数添加到事件循环队列中。
process.nextTick(() => {
	var promises = [];

	//在“promises”数组中创建500个promise。
	//他们每一项都在1s后完成解析。
	for(let i = 0; i < 500; i++) {
		promises.push(new Promise((resolve) => {
			setTimeout(resolve, 1000);
		}));
	}

	//当它们全部完成解析后，
	//记录下来我们已经完成了处理。
	Promise.all(promises).then(() => {
		console.log('handled requests');
	});
});

//这需要比1s更长的时间，
//来解析所有被添加到队列中的promise。
//所以这个处理程序阻止了500个用户请求，直到它完成.. 
process.nextTick(() => {
	console.log('hogging the CPU ...');
	work(100000);
});
```

第一次调用process.nextTick()通过调度函数在1s后运行来模拟实际的客户端请求。所有这些导致一个单一的promise得到解析; 这记录了所有请求都已处理完毕的事实。process.nextTick()的下一次调用开销很大的，完全阻止了这500个请求。这绝对不适合我们的应用。我们在NodeJS中运行CPU密集型代码的场景的唯一方法是打破单进程方法。接下来介绍该话题。


### 交叉进程

我们已经达到了在我们的应用程序中的目的，在那里根本没有办法解决它。我们有一些相对开销大的处理请求。我们需要在硬件层面使用并行性。在Node中，这意味着只有一件事 - 交叉子进程处理主进程外的CPU密集型工作，以便正常的请求可以不间断地继续。这里有张示图展示了这个策略的样子：

![image181.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image181.gif?raw=true)


现在，让我们编写一些使用child_process.fork()函数生成新Node进程的代码，当我们需要处理CPU耗尽的请求时。首先，这里是主模块：

```javascript
//我们需要“child_process”来分叉新的节点进程
var child_process = require('child_process');

//分叉我们的worker进程。
var child = child_process.fork(`${dirname} / child`);

//子进程响应数据时发出此事件。
child.on('message', (message) => {
	//显示结果并杀死子进程
	//因为我们已经完成它了。
	console.log('work result', message);
	child.kill();
});

//向子进程发送消息。
//我们在这一端发送一个数字，
//然后是“child_process”确保它作为一个数字到达在另一端。
child.send(100000);
console.log('work sent ...');

//由于开销大的计算正在发生在另一个进程，
//普通请求像平常一样流过事件循环。
process.nextTick(() => {
	console.log('look ma, no blocking!');
});
```

现在我们面对的唯一开销实际上是生成新的进程，这在对比实际我们必须要执行的工作中简直是小巫见大巫。我们可以清楚地看到主IO循环没有被阻止，因为在主进程没有占用CPU。子进程，在另一方面，占用CPU，但这是可以的，因为它可能发生在不同的核心上。这是我们的子进程代码：

```javascript
//消耗一些CPU周期......
//源自http://adambom.github.io/parallel.js/
function work(n) {
	var i = 0;
	while(++i < n * n) {}
	return i;
}

//当父进程发送消息时触发“message”事件，
//然后我们响应执行开销大的CPU操作结果。
process.on('message', (message) => {
	process.send(work(message));
});
```
 

### 生成外部进程

有时，我们的Node应用程序需要与非Node进程的其他程序进行通信。这些可能是我们编写的其他应用程序，但使用不同的平台或基础系统命令。我们可以生成这些类型的进程并与它们进行对话，但它们与分叉另一个节点进程的工作方式不同。这个示图说明这些差异：

![image182.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image182.gif?raw=true)


如果我们是如此倾向，我们可以使用spawn()来创建子节点进程，但这在某些情况下并不是很好。例如，我们没有得到fork()为我们自动设置的消息传递基础结构。但是，最好的通信办法取决于我们想要实现的目标，而且大多数时候，我们实际上并不需要消息传递。


让我们看一些产生进程并读取该进程输出的代码：

```javascript
//我们所需的模块...
var child_process = require('child_process');
var os = require('os');

//生成我们的子进程 - “ls”系统命令。
//命令行标志作为一个数组传递。
var child = child_process.spawn('ls', [
	'-lha',
	__dirname
]);

//我们的输出累加器
//初始化为一个空字符串。
var output = '';

//在进程到达时添加输出。
child.stdout.on('data', (data) => {
	output += data;
});

//我们已经完成了子进程的输出，
//所以记录输出并将其删除。
child.stdout.on('end', () => {
	output = output.split(os.EOL);
	console.log(output.slice(1, output.length - 2));
	child.kill();
});
```
 

> 我们生成的ls命令在Windows系统上并不存在。我这里没有其他有关的方法 - 这是一个事实。


### 进程间通信

在我们刚看到的示例中，生成了子进程，并且我们的主进程收集了输出，从而终止了进程; 但是，当我们编写服务器和其他类型的长时间运行的程序时呢？在这种情况下，我们可能不希望不断产生和杀死子进程。相反，最好是让进程与主程序一起保持活动并继续向其发送消息，如下图所示：

![image186.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image186.gif?raw=true)


即使worker正在同步处理请求，它仍然可以作为我们主要应用程序的优势，因为没有任何东西阻止它提供请求。例如，请求不需要任何繁重的CPU运算才可以继续提供快速响应。现在让我们看看代码示例：

```javascript
var child_process = require('child_process');

//分叉我们的“worker”进程并创建一个“resolvers”对象
//用于存储我们的promise解析器。
var worker = child_process.fork(`${dirname} / worker`),
	resolvers = {};

//当worker响应消息时，
//将消息输出到适当的解析器。
worker.on('message', (message) => {
	resolver[message.id](message.output);
	delete resolvers[message.id];
});

//IDs用于映射来自worker进程的响应
//到promise解析器函数。
function* genID() {
	var id = 0;

	while(true) {
		yield id++;
	}
}

var id = genID();

//这个函数将给定的“input”发送给worker，
//并返回一个promise。promise通过worker的返回值解析。
function send(input) {
	return new Promise((resolve, reject) => {
		var messageID = id.next().value;

		//在“resolvers”映射中存储解析器函数
		resolvers[messageID] = resolve;

		//将“messageID”和“input”发送给worker
		worker.send({
			id: messageID,
			input: input
		});
	});
}

var array;

//构建一个数字数组以发送给worker
//单独处理。
array = new Array(100)
	.fill(null)
	.map((v, i) => (i + 1) * 100);

//将“array”中的每个数字发送到工作进程
//作为一条消息。当每个promise得到解析时，我们可以迭代结果。
var first = Promise.all(array.map(send)).then((results) => {
	console.log('first result', results.reduce((r, v) => r + v));
	//→first result 3383500000
});


//使用较小的数字创建一个较小的数组 - 
//它应该比以前花费更少的时间来处理这个数组。
array = new Array(50)
	.fill(null)
	.map((v, i) => (i + 1) * 10);

//处理第二个数组，记录迭代的结果。
var second = Promise.all(array.map(send))
	.then((results) => {
		console.log('second result', results.reduce((r, v) => r + v));
		//→第二个结果4292500
	});

//当两个数组都完成处理时，我们需要
//杀死worker进程以退出我们的程序。
Promise.all([first, second]).then(() => {
	worker.kill();
});
```

现在让我们看看我们从主模块派生的worker模块：

```javascript
//消耗一些CPU周期...
//源自http://adambom.github.io/parallel.js/
function work(n) {
	var i = 0;
	while (++i < n * n) {}
	return i;
}

//使用调用“work()”的结果和消息ID响应主进程。
process.on('message', (message) => {
	process.send({
		id: message.id,
		output: work(message.input)
	});
});
```

我们创建的数组中的每个数字都传递给执行CPU繁重工作的worker进程。结果将传递回主进程，并用于解析promise。这种技术是非常类似于我们在“第7章，抽象并发”中使用的web worker的promise方式。


我们在这里尝试计算两个结果 - 一个用于first数组，一个用于second数组。第一个具有比第二个更多的数组项，并且数字更大。这意味着这需要更长的时间来计算，事实上，它确实需要更长的时间。但是，如果我们运行此代码，在第一个数组完成之前，我们看不到第二个数组的输出。


这是因为尽管需要较少的CPU时间，但第二个作业仍然被阻止，因为保留了发送给workers的消息的顺序。换句话说，来自第一个数组的所有100条消息甚至在第二个数组上开始之前都会被处理。乍一看，这似乎是一件坏事，因为它实际上并没有为我们解决任何问题。嗯，这并不是真的。


唯一被阻止的是到达worker进程的排队消息。由于worker忙于CPU，因此无法在到达时立即处理消息。但是，此worker程序的目的是从需要它的Web请求处理程序中删除繁重的处理。并非每个请求处理程序都有这种类型的重负载，你觉得呢？它们可以继续正常运行，因为进程中没有任何东西会占用CPU。


但是，当我们的应用程序继续变得越来越大和越来越复杂，因为这些增加的功能以及与其他功能交互的方式。我们需要一种更好的方法来处理开销大的请求处理程序，因为我们将拥有更多这些功能。这是我们将在下一节中介绍的内容。


### 进程集群

在上一节中，我们介绍了NodeJS中的子进程创建。当请求处理程序开始使用越来越多的CPU时，这是Web应用程序的必要措施，因为它可以阻止系统中的其他每个处理程序。在本节中，我们将基于这个方法，维护一个通用进程池，而不是分叉一个通用worker进程，它能够处理任何请求。


我们首先重申手动管理这些过程所带来的挑战，这些过程可以帮助我们在Node中实现并发方案。然后，我们将看看Node的内置进程集群功能。


### 进程管理面临的挑战

在我们的应用程序中手动编排进程的一个明显问题是并发代码就在那里，明白的讲，与我们的其他应用程序代码混合在一起。在实现Web workers时，我们实际上遇到了本书前面完全相同的问题。如果没有封装同步和worker的通用管理，我们的代码主要由并发模板组成。一旦发生这种情况，很难将并发机制从对我们的产品特有的至关重要的功能代码分离。


Web worker的一个解决方案是创建一个worker线程池并将它们隐藏在统一的API之后。这样，我们需要并行执行的函数代码可以这样做，而不会乱放我们的编辑器并发同步语法。


事实证明，NodeJS解决了大多数系统上利用可用的硬件并行性的问题，这与我们对Web worker所做的类似。接下来，我们将深入探讨其工作原理。


### 抽象进程池

我们可以使用child_process模块手动派生我们的Node进程来启用真正的并行性。这在执行可能阻塞主进程的CPU密集型工作时非常重要。因此，主IO事件循环可以为传入的请求提供服务。我们可以提高并行度，而不仅仅是单个worker进程，但这需要很多手动同步逻辑。


所述cluster模块需要很少的配置代码，但worker进程和主进程之间的实际通信处理对我们的代码是完全透明的。换句话说，看起来我们只是运行一个Node进程来处理我们传入的Web请求，但实际上，有几个克隆进程在处理它们。由cluster模块将这些请求分发给worker节点，默认情况下，这使用轮循方法，这对于大多数情况来说已经足够了。


在Windows上，默认不是轮循法。我们可以手动更改我们想要使用的方法，但轮循法可以保持简单和平衡。唯一的挑战是我们的请求处理程序运行成本比大多数要高得多。然后，我们可以最终分发请求到一个重载的worker进程。在对此模块进行故障排除时，这只是需要注意的事项。


这里的示图，显示了与主节点进程相关的worker节点进程：

![image187.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image187.gif?raw=true)


主进程在集群方案中有两个职责。首先，它需要与worker进程建立通信渠道。其次，它需要接受传入的连接并将它们分发给worker进程。这实际上很难绘制，因此未在图中表示。在我尝试进一步解释之前，让我们先看一些代码：

```javascript
//我们需要的模块... 
var http = require('http');
var cluster = require('cluster');
var os = require('os');

//消耗一些CPU周期...
//源自http://adambom.github.io/parallel.js/
function work(n) {
	var i = 0;
	while(++i < n * n) {}
	return i;
}

//检查这是哪种类型的进程。
//它也是主进程还是worker进程。
if (cluster.isMaster) {

	//进入“worker”的并行级别。
	var workers = os.cpus().length;

	//分叉我们的worker流程。
	for(let i = 0; i < workers; i++) {
		cluster.fork();
	}

	console.log('listening at http://localhost:8081'); 
	console.log(`worker processes: ${workers}`);

	//如果这个进程不是主进程，那就是worker进程。
	//所以我们对每个其他worker创建了相同的HTTP服务器。
} else {
	http.createServer((req, res) => {
		res.setHeader('Content-Type', 'text/plain');
		res.end(`worker ${cluster.worker.id}: ${work(100000)}`);
	}).listen(8081);
}
```

这种并行化我们的请求处理程序的方法真正令人高兴的是并发代码是不显眼的。总共有大约10行。一眼望去，我们可以很容易地看到这些代码在做什么。如果我们想要看到这个应用程序的运行，我们可以打开多个浏览器窗口并同时指向它们。由于请求处理程序在CPU周期方面开销很大，我们应该能够看到每个页面都响应计算的值以及计算它的worker ID。如果我们没有分叉这些worker进程，那么我们可能仍在等待我们的每个浏览器选项卡加载。


唯一有点棘手的部分是我们实际创建HTTP服务器的部分。因为每个worker程序都运行相同的代码，所以在同一台计算机上使用相同的主机和端口 - 这是怎么回事？嗯，这实际上并不是发生了什么。net模块，HTTP模块使用的低级别的网络库，实际上是群集感知。这意味着当我们要求net模块监听套接字以获取传入请求时，它首先检查它是否是worker节点。如果是，则它实际上共享主进程使用的相同套接字句柄。这很不错。将请求分发到worker进程并实际切换请求需要很多丑陋的逻辑，所有这些都由cluster模块为我们处理。


### 服务器集群

通过进程管理启用并行性来扩展运行我们的NodeJS应用程序的单个机器。这是充分利用我们的物理硬件或虚拟硬件的好方法 - 它们都需要花钱。但是，只扩展一台机器存在固有的局限性 - 到目前为止是这样的。在我们扩展问题的某些方面的门槛上，我们会碰壁。在此之前，我们需要考虑将Node应用程序扩展到多台机器。


在本节中，我们将介绍Web请求代理到其他计算机的方法，而不是在它们到达的计算机上处​​理它们。然后，我们将介绍如何实现微服务，以及它们如何帮助构建一个健全的应用程序架构。最后，我们将实现一些针对我们的应用程序量身定制的负载均衡代码; 以及它如何处理请求。


### 代理请求

NodeJS中的请求代理正如其名。请求到达服务器，由Node进程处理。但是，这里没有完成请求 - 它代理到另一台机器。所以问题是，为什么要费心的去做代理呢？为什么不直接去实际响应的目标机器完成我们的请求？


这个想法的问题是Node应用程序通常响应来自浏览器的HTTP请求。这意味着我们通常需要一个进入后端的入口点。另一方面，我们不一定希望这个单一入口点是单个Node服务器。当我们的应用程序变大时，这会受到限制。相反，我们希望能够传播我们的应用程序或按照他们的说法横向扩展它。代理服务器删除地理限制;我们的应用程序的不同部分可以部署在世界的不同区域，同一数据中心的不同部分，甚至是不同的虚拟机上。关键是我们可以灵活地改变应用程序组件的驻留位置，以及如何配置它们而不影响应用程序的其他部分。


通过代理分发Web请求的另一个很不错的方面是我们可以实际编程我们的代理处理程序来修改请求和响应。因此，虽然我们的代理所依赖的各个服务可以实现我们的应用程序的一个特定方面，但代理可以实现适用于每个应用程序的通用部分请求。以下是代理服务器和实际满足每个请求的API端点的示图：

![image188.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image188.gif?raw=true)


### 促进微服务

根据我们正在构建的应用程序的类型，我们的API可以是一个整体的服务，或者可以是组装的几个微服务。在一个方面，整体式的API往往是更适合于构建一个没有大量功能和数据的小应用。在另一方面，较大的应用程序的APIs往往会变得很复杂的，从这点来看它不可能很好的维护，因为有许多地区交织着一个另一个。如果它们拆分成微服务，它更容易将它们部署到适合自己需求的特定环境，并有一个专门的团队专注于一个服务，这样会很好。


> 微服务架构是一个很大的话题，显然远远超出了本书的范围。这里的重点是微服务启用 - 机制比设计更多。


我们将使用node-http-proxy(`https://github.com/nodejitsu/node-http-proxy`)模块来实现我们的代理服务器。这不是核心Node模块，因此我们的应用程序需要将其作为npm依赖项。让我们看一下代理请求到相应服务的基本示例：

> 此示例启动三个Web服务器，每个服务器在不同的端口上运行。

```javascript
//我们需要的模块...
var http = require('http'),
	httpProxy = require('http-proxy');


//“proxy”服务器是我们发送请求到其他主机的方式。
var proxy = httpProxy.createProxyServer();
http.createServer((req, res) => {

	//如果请求是针对站点根目录的，
	//我们返回一些带有一些链接的HTML'。
	if (req.url === '/') {
		res.setHeader('Content-Type', 'text/html');
		res.end(`
			<html>
				<body>
					<p><a href="hello">您好</a></p>
					<p><a href="world">世界</a></p>
				</body>
			</html>
		`);

	//如果URL是“hello”或“world”，我们使用“proxy.web()”
	//代理请求到相应的微服务。
	} else if (req.url === '/hello') {
		proxy.web(req, res, {
			target: 'http://localhost:8082'
		});
	} else if (req.url === '/world') {
		proxy.web(req, res, {
			target: 'http://localhost:8083'
		});
	} else {
		res.statusCode = 404; 
		res.end();
	}

}).listen(8081);

console.log('listening as http://localhost:8081');
```

这两个服务hello和world实际上并没有列在这里，因为他们所做的只是为任何请求返回一行纯文本。他们分别监听在端口8082和8083上。该http-proxy模块很容易让我们简单地使用最少的逻辑将请求转发到相应的服务。


### 了解负载均衡

在本章的前面部分，我们研究了流程集群。这里我们使用cluster模块创建进程池，每个进程都能够处理来自客户端的请求。主进程在此方案中充当代理，默认情况下，以轮循法将请求分发给worker进程。我们可以使用http-proxy模块做类似的事情，但这方法比轮循法更天真。


例如，假设我们有两个运行相同微服务的实例。好吧，这些服务中的一个可能比另一个更加繁忙，这会使服务失去平衡，因为繁忙的节点将继续接收请求，即使它无法立即连接上。持续保持请求直到服务可以处理它们是有意义的。首先，我们将实现一个需要随机一段时间才能完成的服务：

```javascript
var http = require('http');

//将端口作为命令行参数，
//所以我们可以运行多个实例服务。
var port = process.argv[2];

//消耗一些CPU周期......
//源自http://adambom.github.io/parallel.js/
function work() {

	var i = 0,
		min = 10000,
		max = 100000,
		n = Math.floor(Math.random() * (max - min)) + min;
	while (++i < n * n) {}
	return i;
}

//阻止CPU随机时间后响应一个纯文本。
http.createServer((req, res) => {
	res.setHeader('Content-Type', 'text/plain');
	res.end(work().toString());
}).listen(port);

console.log(`listening at http://localhost:${port}`);
```

现在我们可以启动这些进程的两个实例，监听不同的端口。实际上，这些将在两台不同的机器上运行，但我们现在只是检测这个方法。现在我们将实现代理服务器，该服务器需要确定给定请求所针对的服务worker：

```javascript
var http = require('http'),
	httpProxy = require('http-proxy');

var proxy = httpProxy.createProxyServer();

//这些是服务目标。他们有一个“host”属性
//和一个“budy”属性。最初他们是
//不繁忙的，因为我们没有发送任何工作。
var targets = [
	{
		host: 'http://localhost:8082', busy: false
	},
	{
		host: 'http://localhost:8083', busy: false
	}
];

//当我们的目标都很忙时，
//所有请求都在这里排队。
var queue = [];

//通过代理请求到不忙的目标来处理请求队列。
function processQueue() {

	//迭代消息队列。
	for (let i = 0; i < queue.length; i++) {

		//迭代目标。
		for (let target of targets) {
			
			//如果目标正忙，跳过它。
			if (target.busy) {
				continue;
			}

			//将目标标记为繁忙 - 从此处开始，
			//目标不接受任何请求直到它移除了标记。
			target.busy = true;

			//从队列中获取当前项。
			let item = queue.splice(i, 1)[0];

			//标记响应，当请求当它返回时以便
			//我们知道是哪个服务worker。
			item.res.setHeader('X-Target', i);

			//发送代理请求并退出循环。
			proxy.web(item.req, item.res, {
				target: target.host
			});

			break;
		}
	}
}

//由http-proxy模块发出事件
//当来自服务worker响应时。
proxy.on('proxyRes', function(proxyRes, req, res) {

	//这是我们之前设置的值，
	//即索引“target”数组。
	var target = res.getHeader('X-Target');

	//我们使用此索引取消标记。
	//现在它会接收新的代理请求。
	targets[target].busy = false;

	//客户端不需要此内部
	//信息，所以删除它。
	res.removeHeader('X-target');

	//当服务worker变得可用，
	//再处理队列，以防pending请求。
	processQueue();
});

http.createServer((req, res) => {

	//所有传入的请求都被推送到队列中。
	queue.push({
		req: req,
		res: res
	});

	//重新处理队列，如果所有服务worker都很繁忙，
	//就将请求留在那里。
	processQueue();
}).listen(8081);

console.log('listening at http://localhost:8081');
```

关于此代理的工作方式，需要注意的点是请求仅代理尚未处理请求的服务。这是知情部分 - 代理知道服务器何时可用，因为它响应最后一个忙于处理的请求。当我们知道哪些服务器正忙时，我们不会因为更多工作​​而使它们超载。


### 小结

在本章中，我们将事件循环视为NodeJS中的并发机制。我们首先使用Co库实现协程。从那里，我们学习了启动新进程，包括分叉另一个Node进程和生成其他非Node进程之间的区别。然后，我们看了看另一种使用cluster模块管理并发的方法，这使得并行流程中的Web请求尽可能透明。最后，我们以使用node-http-proxy模块在机器级别并行化我们的Web请求结束了本章。


这适用于JavaScript并发话题。我们已经在浏览器和Node中涵盖了很多方面。但是，如何将这些方法和组件组合在一起形成并发应用程序？在本书的最后一章，我们将看看并发应用程序的实现。
