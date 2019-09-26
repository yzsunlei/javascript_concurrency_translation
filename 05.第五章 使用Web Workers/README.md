## 使用Web Workers

Web workers在Web浏览器中实现了真正的并发。他们花了很多时间改进，现在有很好的浏览器支持。在

Web workers之前，我们的JavaScript代码局限于CPU，我们的执行环境在页面首次加载时启动。

Web workers发展起来 - web应用程序越来越强大。他们也开始需要更多的计算能力。与此同时，多核CPU

现在很常见 - 即使是在一些低端设备上。


在本章中，我们将介绍Web workers的思想，以及它们如何与我们努力在应用中实现的并发性原则产生关联。

然后，将通过示例学习如何使用Web worker，以便在本书的后面部分，我们可以开始将并发性与我们已经探索

过的其他一些想法联系起来，例如promises和generators。


### 什么是Web workers？

在深入研究实现示例之前，本节将简要介绍Web workers的概念。搞清楚Web workers如何与引擎下的其他

系统协作的。Web workers是操作系统线程 - 我们可以调度事件的对象，它们以真正的并发范式来执行

我们的JavaScript代码。


#### OS线程

从本质上讲，Web workers只不过是操作系统级线程。线程有点像进程，除了它们需要更少的开销，因为它们

与创建它们的进程共享内存地址。由于为Web workers提供支持的线程处于操作系统级别，因此受系统及其

进程调度程序的管理。实际上，这正是我们想要的 - 让内核清楚我们的JavaScript代码应该什么时候运行，

这样才能充分地利用CPU。


下面的示图展示了浏览器如何将其Web workers映射到OS线程，以及这些线程如何映射到CPU上：

![image090.gif](./images/image090.gif)


在日常活动结束时，操作系统最好能放下其他任务来负责它擅长的 - 处理物理硬件上的软件任务调度。

在传统的多线程编程环境中，代码更接近操作系统内核。Web workers不是这种情况。虽然底层机制是

一个线程，但是暴露的编程接口看起来更像是你可能在DOM中查找的东西。


#### 事件对象

Web workers实现了熟悉的事件对象接口。这使得Web workers的行为类似于我们使用的其他组件，

例如DOM元素或XHR请求。Web workers触发事件，这就是我们在主线程中从他们那里接收数据的方式。

我们也可以向Web workers发送数据，这使用一个简单的方法调用。


当我们将数据传递给Web workers时，我们实际上会触发另一个事件；只有这时候，它位于Web workers的

执行上下文中，而不是在主页面的执行上下文。没有更多的事情要处理：数据输入，数据输出。没有互斥结构

或任何此类结构。这实际上是一件好事，因为作为平台的Web浏览器已经有许多模块。想象一下，如果我们投入

很复杂的多线程模型而不是一个简单的基于事件对象的方法。我们每天已经有足够多的bugs需要处理。


以下是关于Web worker排布的样子，相对于生成这些Web workers的主线程：

![image091.gif](./images/image091.gif)


#### 真正的并发性

Web workers是在我们的架构中实现并发性原则的方法。我们知道，Web workers是操作系统线程，这意味

着在它们内部运行的JavaScript代码可能在与主线程中的某些DOM事件处理程序代码相同的实例上运行。

能够做这样的事情已经在很长一段时间成为JavaScript程序员的目标了。在Web workers之前，真正的

并发性是不可能的。我们所做的最好的就是模拟它，给用户一种许多事情同时发生的的假象。


但是，始终在同一CPU内核上运行是存在问题的。我们从根本上限制了在给定时间窗口内可以执行多少次计算。

当引入真正的并发性时，此限制会被打破，因为可以运行计算的时间窗口会随着添加的CPU而增加。


话虽这么说，对于我们的应用程序所做的大多数事情，单线程模型工作的也很好。现在的机器都很强大。我们

可以在很短的时间内完成很多工作。当我们临近峰值时会出现问题。这些可能是一些事件中断了我们代码处理

进程。我们的应用程序不断被要求做得更多 - 更多功能，更多数据。


我们可以更好地利用在我们面前的硬件的方法就是web workers所关心的。Web workers，如果使用得当，

它不一定是我们在项目中永远不会使用的不可逾越的新东西，因为它的概念超出我们之前的理解。


### workers的种类

在开发并发JavaScript应用程序中，我们可能会见到三种类型的Web workers。在本节中，我们将比较这三种类型，

以便可以了解在给定的上下文中哪种类型的workers更有用。
 

#### 专用workers

专用workers可能是最常见的workers类型。它们被作为是Web worker的默认类型。当我们的页面创建

一个新的Web worker时，它专门用于页面的执行上下文而不是其他内容。当我们的页面销毁时，页面创建的所有

专用workers也会销毁。


页面与其创建的任何专用worker之间的通信路径非常简单。该页面将消息发送给workers，workers又

将消息发回页面。这些消息的顺序取决于我们尝试使用Web worker解决的问题。我们将在本书中深入研究这些

消息传递模式。

> 术语主线程和页面在本书中是同义词。主线程是典型的执行上下文，我们可以在这里操作页面并监听输入。
> Web worker上下文基本相同，但只能访问较少的Web组件。我们将很快讨论这些限制。


以下是页面与专用workers通信的描述：

![image095.gif](./images/image095.gif)


正如我们所看到的那样，专用workers是专注的。它们仅用来服务创建它们的页面。他们不直接与

其他Web workers通信，也无法与任何其他页面进行通信。


#### 子workers

子workers与专用workers非常相似。主要区别在于它们是由专门的Web worker创建的，而不是

由主线程创建的。例如，如果专用workers的任务可以从并发执行中受益，则可以生成子workers

并协调子workers之间的任务执行。


除了拥有不同的创建者之外，子workers还具有一些与专用workers相同的特征。子workers不直接

与主线程中运行的JavaScript通信。由创建它们的worker来协调他们的通信。以下有张示图，说明

子workers如何按照约定来运行的：

![image096.gif](./images/image096.gif)


#### 共享workers

第三类Web worker被称为一个共享worker。共享workers被如此命名是因为多个页面可以共享

这种类型worker的同一个实例。在该页面可以访问一个给定的共享workers实例由同源策略所限制，

这意味着，如果一个页面跟这个worker不同域，该worker是不被允许与此页面通信的。


共享workers解决的问题与专用workers解决的问题不同。将专用workers视为没有副作用的函数。

你将数据传递给它们并获得不同的返回数据。将共享workers视为遵循单例模式的应用程序对象。

它们是在不同上下文之间共享状态的方法。因此，例如，我们不会仅仅为了处理数字而创建一个共享worker; 

我们可以使用一个专用worker。


当内存中的应用程序数据来自同一应用程序的其他页面时，我们使用共享workers就有意义了。想想用户

在新选项卡中打开链接。这将创建一个新的上下文。这也意味着我们的JavaScript组件需要经历获取页面

所需的所有数据，执行所有初始化步骤等过程。这造成重复和浪费。为什么不通过在不同的浏览上下文之间

共享的方式来保存这些资源呢？以下有个示图说明来自同一应用程序的多个页面与共享workers实例通信：

![image097.gif](./images/image097.gif)


实际上还有第四种类型称为服务workers。这些是共享worker，其中包含与缓存网络资源和脱机功能相关

的其他功能。服务workers仍处于规范的早期阶段，但他们看起来很有意义。如果服务workers成为

可行的Web技术，我们今天了解的关于共享workers的任何内容都将适用于服务workers。


这里要考虑的另一个重要因素是服务workers的复杂性。主线程和服务worker之间的通信机制涉及

使用端口。同样，在共享workers中运行的代码需要确保它通过正确的端口进行通信。我们将在本章后面

更深入地介绍共享workers的通信。


### Web workers环境

Web worker环境与我们的代码通常运行的JavaScript环境不同。在本节中，我们将指出主线程的

JavaScript环境与Web worker线程之间的主要区别。


#### 什么是可用的，什么不是？

对Web workers的一个常见误解是，它们与默认的JavaScript执行上下文完全不同。确实，他们是不同的，

但没有那么不同以至于没有可比性。也许，正是由于这个原因，JavaScript开发人员在可能的时候回避使用

Web worker是有益的。


明显的差距是DOM - 它在Web worker执行环境中不存在。它不存在是规范起草者有意识决定的。通过避免DOM

集成到工作线程中，浏览器提供商可以避免许多潜在的边缘情况。我们都非常重视浏览器的稳定性，或者至少

我们应该重视。从Web worker那里获取DOM访问权限真的很方便吗？我们将在本书接下来的几章中看到，workers

擅长许多其他任务，这些任务最终有助于成功实现并发原则。


由于我们的Web worker代码没有DOM访问权限，因此我们不太可能自找麻烦。它实际上迫使我们去思考为什么

我们要使用Web workers。我们实际上可能退后一步，重新思考我们的方法。除了DOM之外，我们日常使用的

大部分东西都存在，这正是我们所期望的。这包括在Web workers中使用我们喜欢的类库。

> 有关Web worker执行环境中缺少功能的更详细分类，请参阅此页面
> `https：//developer.mozilla.org/en-US/docs/Web/API/Worker/Functions_`
> `and_classes_available_to_workers`。


#### 加载脚本

我们绝不会将整个应用程序编写在一个JavaScript文件中。相反，我们通过将源代码划分为文件的方式来

便于模块化，从逻辑上可以将设计分解为我们想映射的内容。同样，我们可能不希望有由数千行代码组成

的Web workers。幸运的是，Web worker提供了一种机制，允许我们将代码导入到我们的Web worker中。


第一种场景是将我们自己的代码导入到一个Web worker上下文。我们很可能有许多低级别的工具方法是

专门针对我们的应用程序。有很大可能，我们就需要在两个环境使用这些工具：一个普通的脚本环境和一个

worker线程。我们想要保持代码的模块化，并希望代码以相同的方式作用于Web workers环境，就像它会

在任何其他环境下运行。


第二种场景是在Web workers中加载第三方库。这与将我们自己的模块加载到Web workers中的原理相同 - 我们

的代码可以在任何上下文中使用，但有一些例外，例如DOM代码。让我们看一个创建Web worker并加载lodash

库的示例。首先，我们将启动Web worker：

```javascript
//加载Web worker脚本，
//然后启动Web worker线程。
var worker = new Worker('worker.js');
```

接下来，我们将使用loadScripts()函数将lodash库导入我们的库：

```javascript
//导入lodash库，
//让全局“_”变量在Web worker上下文中可用。
importScripts('lodash.min.js');

//我们现在可以在Web worker中使用库。
console.log('in worker', _.at([1, 2, 3], 0, 2));
//→in worker[1,3]
```

在开始使用脚本之前，我们不需要担心等待脚本加载 - importScripts()是一个阻塞的操作。


### 与Web workers通信

前面的示例创建了一个Web worker，它确实在自己的线程中运行。但是，这对我们没有多大帮助，因为我们需要

能够与我们创造的workers通信。在本节中，我们将介绍从Web workers发送和接收消息所涉及的基本机制，

包括如何序列化这些消息。


#### 发布消息

当我们想要将数据传递给Web worker时，我们使用postMessage()方法。顾名思义，此方法将给定的消息发送

给worker。如果在worker中设置了任何消息事件处理程序，它们将响应此调用。让我们看一个将字符串发送

给worker的基本示例：

```javascript
//启动Web worker线程。
var worker = new Worker('worker.js');

//向Web worker发送消息，
//触发“message”事件处理程序。
worker.postMessage('hello world');
```

现在让我们看看worker通过为消息对象设置事件处理程序来查看此响应消息：

```javascript
//为任何“message”设置事件监听器
//调度给该worker的事件。
addEventListener('message', (e) => {

	//可以通过事件对象的“data”属性访问发送的数据
	console.log(e.type, `"${e.data}"`);
	//→message “hello world”
});
```

> addEventListener()函数是在全局专用Web workers环境调用的。
> 我们可以将其视为Web workers的窗口对象。


#### 消息序列化

从主线程传递到worker线程的消息数据要经过序列化转换。当此序列化数据到达worker线程时，它被

反序列化，并且数据可用作JavaScript基本类型。当worker线程想要将数据发送回主线程时，使用同样

的过程。


毋庸置疑，这是一个多余的步骤，给我们可能已经过度工作的应用程序增加了开销。因此，必须考虑在

线程之间来回传递数据，因为从CPU成本方面来说这不是轻松的操作。在本书的Web worker代码示例中，

我们将消息序列化视为我们并发决策过程中的关键因素。


所以问题是 - 为什么要这么长？如果我们在JavaScript代码中使用的worker只是线程，我们应该在技术上

能够使用相同的对象，因为这些线程使用相同的内存地址段。当线程共享资源（例如内存中的对象）时，

可能会发生具有挑战性的资源抢占情况。例如，如果一个worker锁定一个对象而另一个worker试图使用它，

则这会发生错误。我们必须实现逻辑来优雅地等待对象变得可用，并且我们必须在worker中实现逻辑来释放

锁定的资源。


简而言之，这是一个容易出错的令人头痛的问题，如果没有这个问题我们会好得多。值得庆幸的是，在仅序列化

消息的线程之间没有共享资源。这意味着我们在实际传递给worker的东西方面受到限制。经验上是传递可以编码

为JSON字符串的东西通常是安全的。请记住，worker必须从此序列化字符串重建对象，因此函数或类实例的字符串

表示根本将不起作用。让我们通过一个例子来看看它是如何工作的。首先，看一个简单的worker记录它收到的消息：

```javascript
//简单显示收到的消息。
addEventListener('message', (e) => {
	console.log('message', e.data);
});
```

现在让我们看看使用postMessage()可以序列化哪种类型的数据并发送给这个worker：

```javascript
//启动Web worker
var worker = new Worker('worker.js');

//发送一个普通对象。
worker.postMessage({hello: 'world'});
//→消息{hello：“world”}

//发送一个数组。
worker.postMessage([1, 2, 3]);
//→消息[1,2,3]

//试图发送一个函数，结果抛出错误
worker.postMessage(setTimeout);
//→未捕获的DataCloneError
```

我们可以看到，当我们尝试将函数传递给postMessage()时会出现一些问题。这种数据类型一旦到达worker线程

就无法重建，因此，postMessage()只能抛出异常。这些类型的限制可能看起来过于局限，但它们确实消除了

许多可能出现的并发问题。


#### 接收来自workers的消息

如果没有将数据传回主线程的能力，workers对我们来说就没什么用了。在某些时候，workers执行的任务需要

显示在UI中。我们可能还记得，worker实例是事件对象。这意味着我们可以监听消息事件，并在workers发回

数据时做出相应的响应。可以将此视为向workers发送数据的反向。workers通过向主线程发送消息将主线程

视为另一个workers线程，而主线程则侦听消息。我们在上一节中探讨的序列化限制在这里也是一样的。


让我们看一下将消息发送回主线程的一些worker代码：

```javascript
//2秒后，将一些数据发回给
//使用“postMessage()”函数的主线程
setTimeout(() => {
	postMessage('hello world');
}, 2000);
```

我们可以看到，这个worker启动了，2秒后，将一个字符串发送回主线程。现在，让我们看看如何在主

JavaScript环境中处理这些传入的消息：

```javascript
//启动一个worker线程。
var worker = new Worker('worker.js');

//为“message”对象添加一个事件侦听器，
//注意“data”属性包含实际的消息数据，
//与发送消息给workers的方式相同。
worker.addEventListener('message', (e) => {
	console.log('from worker', `“$ {e.data}”`);
});
```

> 您可能已经注意到我们没有显式终止任何worker线程。这没关系。当浏览上下文终止时，所有活动工作
> 线程都将终止。我们也可以使用terminate()方法显式的终止worker，这将显式停止线程而无需等待任何
> 现有代码执行完成。但是，很少去显式终止worker。一旦创建，workers通常在页面整个生命周期内存活。
> 生成worker不是免费的，它会产生开销，所以如果可能的话，我们应该只做一次。


### 共享应用状态

在本节中，我们将介绍共享workers。首先，我们将了解多个浏览上下文如何访问内存中的相同数据对象。

然后，我们将介绍如何获取远程资源，以及如何通知多个浏览上下文有关新数据的返回。最后，我们将了解如何

利用共享workers来允许浏览上下文之间的直接消息传递。


> 考虑下本节用于实验编码的高级特性。浏览器对共享workers的支持目前还不是很好（只有Firefox和Chrome）。
> Web worker仍处于W3C的候选推荐阶段。一旦他们成为推荐并为共享workers提供了更好的浏览器支持，
> 我们就可以使用它们了。对于额外的意义，当服务workers规范成熟，共享Worker能力将更加重要。


#### 共享内存

到目前为止我们已经看到了Web workers的序列化机制，因为我们不能直接从多个线程引用同一个对象。但是，

共享worker的内存空间不仅限于一个页面，这意味着我们可以通过各种消息传递方法间接访问内存中的这些对象。

实际上，这是一个展示我们如何使用端口传递消息的好机会。让我们来看看吧。


端口的概念对于共享worker是很必要的。没有它们，就没有管理机制来控制来自共享worker的消息的流入和流出。

例如，假设我们有三个页面使用相同的共享worker，那么我们必须创建三个端口来与该workers通信。将端口视为

workers通往外部世界的入口。这是一个小的间接过程。


这是一个基本的共享worker，让我们了解设置这些类型的workers所涉及的内容：

```javascript
//这是连接到worker的页面之间的共享状态数据
var connections = 0;

//侦听连接到此worker的页面，
//我们可以设置消息端口。
addEventListener('connect', (e) => {
	//“source”属性代表由连接到这个worker页面创建的消息端口，
	//我们实际上要通过调用“start()”建立连接。
	e.source.start();
});

//我们将消息发回页面，数据是更新的连接数。
e.source.postMessage(++connections);
```

一旦页面与此worker连接，就会触发一个connect事件。该connect事件具有一个source属性，这是消息端口。

我们必须通过调用start()来告诉这个worker已准备开始与它通信。请注意，我们必须在端口上调用postMessage()，

而不是在全局上下文中调用。worker怎么知道要将消息发送到哪个页面？该端口充当worker和页面之间的代理，

如下图所示：

![image109.gif](./images/image109.gif)


现在让我们看看如何在多个页面中使用这个共享worker：

```javascript
//启动共享worker。
var worker = new SharedWorker('worker.js');

//设置“message”事件处理程序。
//通过连接共享worker，我们实际上是在创建一个消息
//发送到消息传递端口。
worker.port.addEventListener('message', (e) => {
	console.log('connections made', e.data);
});

//启动消息传递端口，
//表明我们是准备开始发送和接收消息。
worker.port.start();
```

这个共享worker和专用worker之间只有两个主要区别。它们如下：

• 我们有一个port对象，我们可以通过发布消息和附加事件监听器来与worker通信。

• 我们告诉worker我们已准备好通过调用端口上的start()方法来启动通信，就像worker一样。

将这两个start()调用视为共享worker与其客户端之间的握手。


#### 获取资源

前面的示例让我们了解了来自同一应用程序的不同页面如何共享数据，从而无需在加载页面时分配两次完全相同

的结构。让我们以这个方法为基础，使用共享worker来获取远程资源，以便与任何依赖它的页面共享返回的结果。

这是worker代码：

```javascript
//我们保存连接页面的端口，
//以便我们可以广播消息。
var ports = [];

//从API获取资源。
function fetch() {
	var request = new XMLHttpRequest();
	
	//当接口响应时，我们只需解析JSON字符串一次，
	//然后将它广播到所有端口。
	request.addEventListener('load', (e) => {
		var resp = JSON.parse(e.target.responseText);
		for (let port of ports) {
			port.postMessage(resp);
		}
	});

	request.open('get', 'api.json');
	request.send();
}

//当一个页面连接到这个worker时，
//我们保存到“ports”数组，
//以便worker可以持续跟踪它。
addEventListener('connect', (e) => {
	ports.push(e.source);
	e.source.start();
});

//现在我们可以“poll”API，并广播结果到所有页面。
setInterval(fetch, 1000);
```

我们只是在ports数组中存储对它的引用，而不是在页面连接到worker时响应端口。这就是我们如何跟踪连接

到worker页面的方式，这很重要，因为并非所有消息都遵循命令响应模式。在这种情况下，我们希望将更新的

API资源广播到正在监听它的所有页面。一个常见的情况是在同一个应用程序，如果有许多浏览器选项卡打开

查看同一个页面，我们可以使用相同的数据。


例如，如果API资源是一个很大的JSON数组需要被解析，如果三个不同的浏览器选项卡解析完全相同的数据，

则会很浪费资源。另一个好处是我们不会轮询API 3次，如果每个页面都运行自己的轮询代码就会是这种情况。

当它在共享worker上下文中时，它只发生一次，并且数据被分发到连接的页面。这对后端的负担也较少，

因为总体而言，发起的请求要少得多。我们现在来看看这个worker使用的代码：

```javascript
//启动共享worker
var worker = new SharedWorker('worker.js');

//监听“message”事件，
//并打印从worker发回的任何数据。
worker.port.addEventListener('message', (e) => {
	console.log('from worker', e.data);
});

//通知共享worker我们已经准备好了开始接收消息
worker.port.start();
```

#### 在页面间进行通信

到目前为止，我们已经处理过以共享worker中的数据为中心的数据资源。也就是说，它来自于一个集中的地方，

比如作为一个API，随后页面通过连接worker来读取数据。我们实际上没有从页面修改任何的数据。

例如，我们甚至没有连接到后端，连接共享worker的页面也没有产生任何数据。现在其他页面都需要知道

这些改变。


但是，让我们说用户切换到其中一个页面并进行一些调整。我们必须支持双向更新。让我们来看看如何

使用共享worker来实现这些功能：

```javascript
//保存所有连接页面的端口。
var ports = [];

addEventListener('connect', (e) => {

	//收到的消息数据被分发给任何连接到此worker的页面。
	//页面代码逻辑决定如何处理数据。
	e.source.addEventListener('message', (e) => {
		for (let port of ports) {
			port.postMessage(e.data);
		}
	});
});

//保存连接页面的端口引用，
//使用“start()”方法开始通信。
ports.push(e.source);
e.source.start();
```

这个worker就像是一颗卫星; 它只是将收到的所有内容传输到已连接的端口。这就是我们所需要的，为什么

还需要更多？我们来看看连接到这个worker的页面代码：

```javascript
//启动共享worker，
//并保存我们正在使用的UI元素的引用。
var worker = new SharedWorker('worker.js'); 
var input = document.querySelector('input');

//每当输入值改变时，发送输入值数据
//到worker以供其他需要的页面使用。
input.addEventListener('input', (e) => {
	worker.port.postMessage(e.target.value);
});

//当我们收到输入数据时，更新我们文字输入框的值，
//也就是说，除非值已经更新。
worker.port.addEventListener('message', (e) => {
	if (e.data !== input.value) {
		input.value = e.data;
	}
});

//启动worker开始通信。
worker.port.start();
```

有趣！现在，如果我们继续打开两个或更多浏览器选项卡，我们对输入值的任何更改都将立即反映在其他页面中。

这个设计的优点在于它的表现一致; 无论哪个页面执行更新，任何其他页面都会收到更新的数据。换句话说，

这些页面承担着数据生产者和数据消费者的双重角色。

> 您可能已经注意到，最后一个示例中的worker向所有端口发送消息，包括发送消息的端口。我们肯定不想这样做。
> 为避免向发送方发送消息，我们需要以某种方式排除for..of循环中的发送端口。

> 这实际上并不容易，因为消息事件对象没有与一起发送端口的识别信息。我们可以建立端口标识符并使​​消息包含ID。
> 这里需要有很多工作，好处并不是那么好。这里的并发设计 - 只是简单地检查页面代码，该消息实际上与页面相关。


### 通过子workers执行子任务

我们在本章中创建的所有workers - 专用workers和共享workers - 都是由主线程生成的。在本节中，我们将讨论

子workers。它们与专用worker相似，只是创建者不同。例如，子worker不能直接与主线程交互，只能通过产生

子workers的代理进行交互。


我们将看看将较大的任务划分为较小的任务，并且我们还将看看围绕子worker的一些挑战。


#### 将工作分为任务

我们的Web worker的工作是以这样的方式执行任务，即主线程可以继续服务于一些事情，例如DOM事件，

而不会中断。对于Web worker线程来说，某些任务很简单。它们接受输入，计算结果，并将结果作为输出

返回。但是，如果任务很复杂，该怎么办？如果它涉及许多较小的分散步骤，需要我们将较大的任务

分解为较小的任务，该怎么办？


像这些任务，通过将它们分解为更小的子任务是有意义的，这样我们就可以进一步利用所有可用的CPU。

然而，将任务分解为较小的任务本身会导致严重的性能损失。如果任务分解放在主线程中，我们的用户

体验可能会受到影响。我们在这里使用的一种技术涉及启动一个Web worker，其工作是将任务分解

为更小的步骤，并为每个步骤启动子worker。


让我们创建一个在数组中搜索指定项的worker，如果该项存在则返回true。如果输入数组很大，

我们会将它分成几个较小的数组，每个数组都是并行搜索的。这些并行搜索任务将作为子worker

创建。首先，我们来看看子worker：

```javascript
//侦听传入的消息。
addEventListener('message', (e) => {
	
	//将结果发回给worker。
	//我们在输入数组上调用“indexOf()”，寻找“search”数据。
	postMessage({
		result: e.data.array.indexOf(e.data.search) > -1
	});
});
```

所以，我们现在有一个子worker可以获取一个数组的块并返回一个结果。这很简单。现在，对于棘手的部分，

让我们实现将输入数组划分为较小输入的worker，然后将其输入子worker。

```javascript
addEventListener('message', (e) => {

	//我们将要分成4个较小块的数组。
	var array = e.data.array;

	//大致计算数组四分之一的大小，
	//这将是我们的块大小。
	var size = Math.floor(0.25 * array.length);

	//我们正在寻找的搜索数据。
	var search = e.data.search;

	//用于在下面的“while”循环将数组分成块。
	var index = 0;

	//一旦被切片，我们的块就会去执行。
	var chunks = [];

	//我们需要保存对子worker的引用，
	//这样我们可以终止它们。
	var workers = [];

	//这用于统计从子workers返回的结果数
	var results = 0;

	//将数组拆分为按比例大小的块。
	while (index < array.length) {
		chunks.push(array.slice(index，index + size));
		index += size;
	}

	//如果还有剩下的（第5块），
	//把它放到它之前的块中。
	if (chunks.length> 4) {
		chunks[3] = chunks[3].concat(chunks[4]);
		chunks = chunks.slice(0, 4);
	}

	for (let chunk of chunks) {
		
		//启动我们的子worker并在“workers”中保存它的引用。
		let worker = new Worker('sub-worker.js');
		workers.push(worker);

		//当子worker有返回结果时。
		worker.addEventListener('message', (e) => {
			results++;

			//结果是“truthy”，我们可以发送一个响应给主线程。
			//否则，我们检查是否全部子workers都返回了。
			//如果是这样，我们可以发送一个false返回值。
			//然后，终止所有子workers。
			if (e.data.result) {
				postMessage({
					search: search,
					result: true
				});
				
				workers.forEach(x => x.terminate());
			} else if (results === 4) {
				postMessage({
					search: search,
					result: false
				});

				workers.forEach(x => x.terminate());
			}
		});

		//为worker提供一大块数组进行搜索。
		worker.postMessage({
			array: chunk,
			search: search
		});
	}
});
```

这种方法的优点是，一旦我们得到了正确的结果，我们就可以终止所有现有的子worker。因此，如果我们执行

一个特别大的数据集，就可以避免让一个或多个子worker在后台进行不必要的运算。


我们在这里采用的方法是将输入数组切成四个比例（25％）的块。这样，我们将并发级别限制为四级。在下一章中，

我们将进一步讨论细分任务和技巧，以确定要使用的并发级别。


现在，让我们通过编写一些代码在页面上使用这个worker以完成示例：

```javascript
//启动worker...
var worker = new Worker('worker.js');

//生成一些输入数据，一个数字0 - 1041数组。
var input = new Array(1041).fill(true).map((v, i) => i);

//当worker返回时，显示我们搜索的结果。
worker.addEventListener('message', (e) => {
	console.log(`${e.data.search}存在？`, e.data.result);
});

//搜索一个存在的项。
worker.postMessage({
	array: input,
	search: 449
});
//→449存在？真

//搜索一个不存在的项。
worker.postMessage({
	array: input,
	search: 1045
});
//→1045存在？假
```

我们能够与worker通信，传递输入数组和数据进行搜索。结果传递给主线程，它们包含搜索词，因此我们能够通过

发送给worker线程的原始消息对输出进行协调。然而，这里有一些困难需要克服。虽然这非常有用，能够细分任务

以更好地利用多核CPU，但涉及到很多复杂性。一旦我们得到每个子worker的结果，我们就必须进行协调。


如果这个简单的例子可以变得像它一样复杂，那么想象一下大型应用程序的上下文中的类似代码。我们可以从

两个角度解决这些并发问题。首先是关于并发性的前期设计挑战。这将在下一个章节解决。然后，还有是同步

挑战，我们如何避免回调地狱？这个主题比较深入，将在“第7章，抽象并发”讨论。


#### 提醒一下

虽然前面的示例是一种强大的并发技术，可以提供很大的性能提升，但还有一些缺点需要注意。因此，在深入

涉及子worker的实现之前，请考虑其中的一些挑战以及必须做出的权衡。


子workers没有一个父页面来直接通信。这是一个复杂的设计，因为即使一个来自子worker简单响应也需要

子worker通过代理从而在运行的JavaScript主线程进行创建。而这样做得到的是一堆让人困惑的通信过程。

换句话说，它很容易导致复杂化的设计，因为要通过比实际上需要的更多组件来完成。所以，在决定使用子

workers作为设计选项之前，让我们看看是否可以只依赖于专用worker来实现。


第二个问题是，由于Web worker仍然是候选推荐的W3C规范，并非所有浏览器都能一致的实现Web worker的

所有功能。共享workers和子workers是我们可能遇到跨浏览器问题的两个部分。另一方面，专用workers具有

很好的浏览器支持，并且在大部分浏览器中表现一致。再一次说明，从简单的专用worker设计开始，如果这不

满足需要，再考虑引入共享workers和子workers。


### Web workers中的错误处理

本章中的所有代码都假设我们的worker程序中运行的代码不会出错。显然，我们的workers会遇到异常被抛出的情况，

或者是我们在开发过程中编写有bug的代码 - 这是我们作为程序员所必须面临的事实。但是，如果没有适当的

错误事件处理程序，Web worker可能很难调试。我们可以采取的另一种方法是显式发回一条消息，标识自己已经

出错。我们将在本节中介绍两个错误处理话题。


#### 错误条件检查

假设我们的主应用程序代码向worker线程发送消息，并期望得到一些返回结果。如果出现问题，那么等待数据的

代码需要知道该怎么办？一种可能性是仍然发送主线程期望的消息; 只是它有一个字段表示操作错误的状态。

下图让我们了解下它是怎么样的：

![image115.gif](image115.gif)


现在让我们看一下实现这种方法的代码。首先，worker确定消息返回成功或错误状态：

```javascript
//当消息返回时，检查提供的消息数据是否是一个数组。
//如果不是，返回一个设置了“error”属性的数据。
//否则，计算并返回结果。
addEventListener('message', (e) => {
	if(!Array.isArray(e.data)) {
		postMessage({
			error: 'expecting an array'
		});
	} else {
		postMessage({
			error: e.data[0]
		});
	}
});
```

该worker总是会通过发送一个消息进行响应，但它并不总是返回一个计算结果。首先，它会检查，以确保该输入

值是可以接受的。如果没有得到期望的数据，它发送一个附加错误状态的消息。否则，它正常的发送返回结果。

现在，让我们编写一些代码来使用这个worker：

```javascript
//启动worker
var worker = new Worker('worker.js');

//监听来自worker的消息。
//如果收到错误，我们会记录错误信息。
//否则，我们记录成功的结果。
worker.addEventListener('message', (e) => {
	if (e.data.error) {
		console.error(e.data.error);
	} else {
		console.log('result', e.data.result);
	}
});

worker.postMessage([3, 2, 1]);
//→result 3

worker.postMessage({});
//→expecting an array
```

#### 异常处理

即使我们在上一个示例中明确检查了workers程序中的错误情况，也可能会抛出异常。从我们的主应用程序线程

的角度来看，我们需要处理这些未捕获类型的错误。如果没有适当的错误处理机制，我们的Web workers将悄然

无声地失败。有时候，workers甚至都不加载 - 遇到这种悄无声息的代码调试。


我们来看一个侦听Web worker error事件的示例。这是一个Web worker尝试访问不存在的属性：

```javascript
//当一个消息数组返回时，
//发送一个包含的“name”属性输入数据作为响应，
//如果数据没有定义怎么办？
addEventListener('message', (e) => {
	postMessage(e.data.name);
});
```

这里没有错误处理代码。我们所做的只是通过读取name属性并将其发回来作为响应消息。让我们看一下使用

这个worker的一些代码，以及它如何响应这个worker中引发的异常：

```javascript
//启动我们的worker
var worker = new Worker('worker.js');

//监听从worker发回的消息，
//并打印结果数据。
worker.addEventListener('message', (e) => {
	console.log('result', `“${e.data}”`);
});

//监听从worker发回的错误，
//并打印错误消息。
worker.addEventListener('error', (e) => {
	console.error(e.message);
});

worker.postMessage(null);
//→Uncaught TypeError：Cannot read property “name” of null

worker.postMessage({name: 'JavaScript'});
//→result “JavaScript”
```

在这里，我们可以看到的是第一个发布消息的worker导致异常被抛出。然而，此异常被封装在worker内部，

它不是抛出在我们的主线程。如果我们在主线程监听error事件，我们就可以做出相应的响应。在这里，

我们只是打印错误消息。然而，在其他情况下，我们可能需要采取更复杂的纠正措施，例如释放资源或

发送一个不同的消息给worker。


### 小结

在本章中，我们介绍了使用Web worker并发执行的概念。在Web worker之前，我们的JavaScript无法利用

当今硬件上的多核CPU。


我们首先对Web worker进行了大致的概述。它们是操作系统级的线程。从JavaScript的角度来看，它们是

可以发送消息和监听message事件的事件对象。Web worker主要分为三种 - 专用workers，共享workers和

子workers。


然后，学习了如何通过发送消息和监听事件来与Web worker进行通信。并且了解到，在消息中传递的内容方面

存在限制。这是因为所有消息数据都在目标线程中被序列化和重建。


我们以如何处理Web worker中的错误和异常来结束本章。在下一章中，我们将讨论并行化的实际应用 - 我们

应该使用并行执行的任务类型，以及实现它的最佳方法。
