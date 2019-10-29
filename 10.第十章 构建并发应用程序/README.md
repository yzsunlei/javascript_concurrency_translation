## 第十章 构建并发应用程序

我们现在已经涵盖了JavaScript在并发方面提供的所有主要领域。我们已经看过浏览器以及JavaScript解释器如何适应这种环境。我们已经研究了几种有助于编写并发代码的语言机制，并且我们已经学会了如何在后端编写并发JavaScript。在本章中，我们将通过构建一个简单的聊天应用程序来尝试将它们串在一起。


值得注意的是，这不是前几章所涵盖的个别没有任何实际意义的话题改编。相反，我们将更多地关注在应用程序的初始实现期间我们必须做出的并发决策，在适当的情况下调整本书中学到的早期方法。我们在代码中使用的并发语义设计比实际使用的机制要重要得多。


我们将首先简要介绍一下实现前的规划。然后，我们将查看我们正在构建的应用程序的更详细的需求。最后，我们将介绍实际的实现，分为两部分，前端和后端。


### 入门

看示例与代码片段是一个为引进给定主题的很好的途径。这或多或少是我们在本书中通过JavaScript进行并发时所做的。在第一章中，我们介绍了一些并发原则。我们应该并行化我们的代码以并发利用硬件。我们应该小心翼翼地同步并发操作。我们应该尽可能的通过惰性计算和内存分配节省CPU和内存。在整个的章节中，我们已经看到这些原则如何适用于JavaScript并发性的不同方面。他们还适用于开发的第一阶段，当我们还没有一个完整的应用程序或正在努力修复一个应用程序。


我们将从另一个角度开始讨论并发性是默认模式的想法。当并发是默认时，一切都是并发的。我们将再次讨论，为什么这是一个如此重要的系统特征。然后，我们将看看相同的原则是否适用于已经存在的应用程序。最后，我们将看看我们可能构建的应用程序类型，以及它们如何影响我们的并发方法。


### 先前的并发性

正如我们现在所知，并发很难。无论我们如何修饰或者我们的抽象有多清楚，它都与我们的大脑工作方式是完全相反的。这听起来不可能，不是吗？事实并非如此。与任何困难问题一样，正确的方法几乎总是分而治之的。在JavaScript并发的情况下，我们希望将问题分成不过是一些非常小，易于解决的问题。一个简单的方法是在我们真正坐下来编写任何代码之前仔细审查潜在的并发问题。


例如，假设我们的工作可能经常遇到并发问题，在整个代码中都是如此。这意味着我们必须花费大量时间进行前期并发设计。从开始发展的早期阶段，诸如generator和promise之类的事情是有意义的，它们使我们更接近我们的最终目标。但是其他方法，如函数式编程，map/reduce和web worker解决了更大的并发问题。这是否意味着我们希望在尚未在应用程序中实际体验的类似问题上花费大量设计时间？


另一种方法是在前期并发设计上花费更少的时间。这并不是说我们忽略了并发性;这将打败本书的整个前提下。相反，我们在假设我们还没有任何并发​​问题的情况下工作，但我们很可能会在以后使用它们。换句话说，我们继续编写默认并发的代码，而不投入尚未存在的并发问题的解决方案。我们在本书中使用的原则再次帮助我们首先解决重要问题。


例如，我们希望并行化我们的代码，以便我们可以充分利用系统上的多个CPU。考虑这个原则迫使我们提出问题 - 我们真正关心的是将8个CPU用于一个易于处理的东西吗？只需很少的努力，我们就可以构建我们的应用程序，这样我们就不会因为不真实的并发问题而陷入困境。考虑如何在开发的早期阶段促进并发性。想一想，这种实现如何使未来的并发问题难以处理，以及什么是更好的方法？在本章后面，我们的演示应用程序将以这种方式实现代码。


### 改造并发性

鉴于在前期思考并发问题花费大量时间是不明智的，我们如何在发生这些问题后立即解决呢？在某些情况下，问题可能是严重问题，导致接口无法使用。例如，如果我们尝试处理大量数据，我们可能会通过尝试分配过多内存导致浏览器选项卡崩溃，或者UI可能只是卡死。这些是需要立即关注的棘手问题，而且往往没有时间可以拖延。


我们可能会遇到的另一种情况是不太关键的，并发实现可以客观地改善用户体验，但如果我们不立即修复它，应用程序也不会失败。例如，假设我们的应用程序在初始页面加载时进行三次API调用。每个调用都等待上一个调用完成。但是，事实证明，调用之间没有实际的依赖关系;他们不需要彼此的响应数据。修复这些调用以使它们全部并行发生的风险相对较低，并且可能会延长一秒以上的时间。


这些更改在我们的应用程序中的改进是多么容易或困难的最终取决于应用程序的编写方式。如前一节所述，我们不想花很多时间考虑不存在的并发问题。相反，我们最初的重点应该是默认的促进并发。因此，当出现这些情况时，我们需要实现并发解决方案来解决实际问题，这并不困难。我们已经在并行思考，因为这就是编写代码的方式。


我们很可能会发现自己正在修复一个不介意并发的应用程序。在尝试修复需要并发解决方案的问题时，这些都很难处理。我们经常发现我们需要重构很多代码才能修复一些基本的东西。当我们在时间上很紧的时候，这会变得很艰难，但总的来说，这可能是一件好事。如果遗留应用程序开始改变为促进更好的并发性，重构一次又一次，然后我们会变得更好。这只会使下一个并发问题更容易修复，并且默认情况下它会促进良好的编码 - 并发风格。


### 应用类型

在实现的初始阶段，您可以而且应该密切关注的一件事是我们正在构建的应用程序类型。编写有助于并发的代码没有通用的方法。这样做的原因是每个应用程序都有自己独特的并发方式。在并发场景之间显然存在一些交叉，但总的来说，我们的应用程序需要自己的特殊处理是一个很好的选择。


例如，花费大量时间和精力来设计Web worker的抽象是否有意义？如果我们的应用程序几乎没有Web请求，那么考虑使API以promise响应是没有意义的。最后，如果我们没有高请求和高速率连接，我们是否真的想要考虑Node组件中的进程间通信设计？


技巧是不要忽略这些优先级较低的项目，因为一旦我们忽略了应用程序中的某些并发情况，下次当发生一些变化，我们将完全没有准备来处理这种情况。我们需要针对常见情况进行优化，而不是在并发上下文中完全忽略应用程序的这些情况。最有效的方法是深入思考我们的应用程序的性质。通过这样做，我们可以轻松地发现在我们的代码中就并发性工作的最佳候选问题被关注到。


### 需求

现在是时候将注意力转向实际构建并发应用程序了。在本节中，我们将从应用程序的总体目标开始，简要介绍我们将要构建的聊天应用程序。然后，我们将其他需求分解为“API”和“UI”两大部分。我们会暂时加入一些代码，不用担心。


### 总体目标

首先，为什么是一个聊天应用？好吧，有两个原因;第一，它不是一个真正的应用程序，而我们不会重新去造轮子;我们正在构建它来学习有关并发的JavaScript在该应用的上下文中。其次，一个聊天应用程序会有很多的模块，你已经在这本书中学习并发机制时演示了一些示例。话虽如此，它将是一个非常简单的聊天应用程序 - 我们在一章中只能讲这么多。


我们将实现的聊天概念与其他大多数熟悉的聊天应用程序相同。聊天本身带有主题，内部有用户和消息。我们将实现这些，而不会太多。即使是UI本身也是典型聊天窗口的简化版本。同样，这是努力将代码示例保持在并发上下文的相关内容中。


为了进一步简化，我们实际上不会将聊天记录保留在磁盘上;我们只会将一切都存在内存中。这样，我们可以将注意力集中在应用程序中的其他并发问题上，并且无需设置存储空间或处理磁盘空间即可轻松运行。我们还将略过聊天的其他常见功能，例如输入通知，表情符号等。它们与我们在这里尝试学习的内容无关。即使删除了所有这些功能，我们也会看到并发设计和实现的如何参与其中;更大的项目更具挑战性。


最后，这个聊天应用程序不使用身份验证，而是提供临时的使用方案，用户在这种情况下不需要注册就可以进行快速聊天。因此，聊天创建者将创建聊天，这将创建一个可与参与者共享的唯一URL。


### API

我们的聊天应用的API将使用简单的Node HTTP服务器实现。它不使用任何Web框架，只使用几个小型库。除了应用程序很简单之外，使用框架也没有任何理由以任何方式增强本章中的示例。在现实世界中，通过各种方式，使用Node Web框架来简化代码 - 本书的教训 - 包括本章 - 仍然适用。


我们聊天数据响应将是的JSON字符串。只会实现对应用程序至关重要的最基本的API接口。以下是我们需要的API接口：

• 创建新聊天

• 加入现有聊天

• 将新消息发布到现有聊天

• 获取现有聊天

很简单吧？这看似简单。由于没有过滤功能，因此需要在前端处理。这是故意的;缺少功能的API很常见，前端的并发解决方案可能就是结果。当我们开始构建UI时，我们将再次重新讨论这个话题。


> 为此示例应用程序实现的NodeJS代码还包括用于提供静态文件的处理程序。这实际上是一种方便措施，而不是对
> 生产中应该发生的事情的反思。更重要的是，您可以轻松运行此应用程序并使用它，而不是以复制生产环境中提供
> 的静态文件的方式。


### 用户界面

我们的聊天应用程序的用户界面将包含一个HTML文件和一些附带的JavaScript代码。HTML文档中有三个页面 - 只是简单的div元素，它们如下所示：

• 创建聊天：用户提供话题及其名称。

• 加入聊天：用户提供他们的姓名并重定向到聊天。

• 查看聊天：用户可以查看聊天消息并发送新消息。

这些网页的作用是不言自明的。最复杂的页面是查看聊天记录，并且这也不是太糟糕。它显示一个从任何参与者发送的所有消息的，包括我们自己的列表，以及该列表的用户。我们必须要实现一个轮询机制，以保持该页面的内容与聊天数据同步。风格方面，我们除了非常基本的布局和字体调整外没有处理很多。


最后，由于用户可能频繁的加入聊天，他们本质上是短暂的和临时性的。毕竟，如果我们不必每次创建或加入聊天时都需要输入我们的用户名，那就太好了。我们将添加把用户名保存在浏览器本地存储中的功能。


好吧，是时候该写一些代码了，准备好了吗？


### 构建API

我们将开始使用NodeJS后端实现。这是我们构建必要的API接口的地方。我们不一定要先从构建后端开始。事实上，很多时候，UI设计驱动API设计。不同的开发者有不同的方法;因为没有特别的原因，我们首先做的是后端。


我们将首先实现基本的HTTP服务和请求路由机制。然后，我们将使用协程作为处理函数。我们将贯穿本节，看看我们的每个处理函数是如何实现的。


### HTTP服务器和路由

我们不会使用Node的核心http模块来处理HTTP请求。在实际应用中，我们更有可能使用Web框架来为我们处理大量模板代码，我们可能会有一个路由组件供我们使用。我们的需求与我们在这些路由中发现的需求非常相似，所以为了简单起见，我们将在这里理清自己的需求。


我们将使用commander库来解析命令行选项，但这实际上并不那么简单。该库非常小，在我们的项目中很早就引入它意味着更容易向我们的服务器添加新的配置选项。让我们看一个示图，它显示了我们的主程序如何适应环境：

![image200.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image200.gif?raw=true)


我们主模块的工作是启动HTTP服务器并设置处理路由的执行函数。路由本身是正则表达式到处理函数的静态映射。我们可以看到，处理函数存储在一个单独的模块中。现在让我们来看看主程序：

```javacript
//我们需要的核心Node模块。
var http = require('http');

//Commander是一个“npm”包，
//对解析命令行参数非常有帮助。
var commander = require('commander');

//我们响应请求的请求处理函数。
var handlers = require('./handlers');

//routes数组包含route-handler参数。
//当一个给定的路由正则与请求URL匹配时，
//关联的处理函数就会被调用。
var routes = [
	[/^\/api\/chat\/(.+)\/message/i, handlers.sendMessage],
	[/^\/api\/chat\/(.+)\/join$/i, handlers.joinChat],
	[/^\/api\/chat\/(.+)$/i, handlers.loadChat],
	[/^\/api\/chat$/i, handlers.createChat],
	[/^\/(.+)\.js$/i, handlers.staticFile],
	[/^\/(.*)$/i, handlers.index]
];

//使用“commander”库添加命令行选项，
//并解析它们。我们只用关心“host”和“port”值。
//两个选项都有默认值。
commander
	.option('-p, --port <port>', 'The port to listen on', 8081)
	.option('-H --host <host>', 'The host to server from', 'localhost')
	.parse(process.argv);

//创建HTTP服务器。这个处理程序将迭代
//我们的“routes”数组，并测试匹配。如果找到了，
//就会使用request，response和正则表达式结果result来调用处理程序。
http.createServer((req, res) => {
	for (let route of routes) {
		let result = route[0].exec(req.url);

		if (result) {
			route[1](req, res, result);
			break;
		}
	}
}).listen(commander.port, commander.host);

console.log(`listening at http://${commander.host}${commander.port}`);
```

这是我们的处理程序路由机制的范围。我们在routes变量中定义了所有路由，随着时间的推移，我们的应用程序会发生变化，这里的路由也会随之发生对应的变化。我们也可以看到，使用commander从命令行选项获取选项非常简单。在这里添加新选项也很容易。


我们给HTTP服务器提供的请求处理函数可能永远不需要更改，因为它实际上不会满足任何请求。它只是遍历路由，直到路由正则表达式与请求URL匹配。发生这种情况时，请求将被移交给处理函数。所以，让我们把注意力转向实际的处理程序实现。


### 作为处理程序的协程

正如我们在本书前面的章节中看到的那样，在我们的前端JavaScript代码中并不需要太多介绍回调地狱。这是promises派上用场的地方，因为它们允许我们封装令人讨厌的同步语法。结果是在我们尝试实现产品功能的组件中的代码清晰可读。我们对Node HTTP请求处理程序有同样的问题吗？


在更简单的处理程序中，不，我们不会面临这一挑战。我们所要做的就是查看请求，找出应该做什么，做到这一点，然后在发送之前更新响应。在更复杂的场景中，在我们能够响应之前，我们必须在请求处理程序中执行各种异步操作。换句话说，如果我们一不小心，回调地狱是不可避免的。例如，我们的处理程序可能会访问其他Web服务以获取某些数据，它可以发出数据库查询，也可以写入磁盘。在所有这些情况下，我们需要在异步操作完成时执行回调;否则，我们永远不会完成我们的回复。


在“第9章，NodeJS高级并发”，我们看看在Node中使用Co库来实现协程。如果我们可以使用我们的请求处理函数做类似的事情呢？也就是说，使它们成为协程而不是普通的可调用函数。最终目标是生成如下所示的内容：

![image201.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image201.gif?raw=true)


在这里，我们可以看到从这些服务获得的值在我们的代码中表现为简单变量。他们不一定是服务;但是，它们可以是任何异步操作。例如，我们的聊天应用程序需要解析从UI发布的表单数据。它将使用强大的formidable库来执行此操作，这是一个异步的。解析后的表单字段将传递给回调函数。让我们将这个动作包装在一个promise中，看看它是什么样的：

```javascript
//此函数返回一个已解析的promise，
//它将解析后的表单数据作为对象。
function formFields(req) {
	return new Promise((resolve, reject) => {
		//使用“formidable”库中的“IncomingForm”类
		//来解析数据。这个“parse()”方法是异步的，
		//所以我们在回调中解析或拒绝promise。
		new formidable.IncomingForm().parse(req, (err, fields) => {
			if (err) {
				reject(err);
			} else {
				resolve(fields);
			}
		});
	})
};
```

当我们想要表单字段时，我们有一个合作的promise，这不错。但是现在，我们需要在协程的上下文中使用该函数。让我们遍历每个请求处理程序，并了解如何使用formFields()函数将promise值视为同步值。


### 创建聊天处理程序

创建聊天处理程序负责创建新聊天。它期望一个话题和一个用户。它将使用formFields()函数来解析发布到此处理程序的表单数据。在将新聊天存储在全局聊天对象中之后(请记住，此应用程序将所有内容存储在内存中)，处理程序将聊天数据作为JSON字符串进行响应。我们来看看处理程序代码：

```javascript
//“Create chat”API。这个接口
//创建一个新的聊天对象并将其存储在内存中。
exports.createChat = co.wrap(function* (req, res) {
	if (!ensureMethod(req, res, 'POST')) {
		return;
	}

	//生成“formFields()”返回的promise。
	//这暂停了这个处理程序的执行，
	//因为这是一个使用“co.wrap()”创建的协程。
	var fields = yield formFields(req);

	//新聊天的ID。
	var chatId = id();

	//用于聊天和添加用户的时间戳。
	var timestamp = new Date().getTime();

	//创建新的聊天对象并存储它。
	//该“users”数组由用户填充创建了聊天。
	//“messages”数组在默认情况下为空。
	var chat = chats[chatId] = {
		timestamp: timestamp, 
		topic: fields.topic,
		users: [{
			timestamp: timestamp,
			name: fields.user
		}],
		messages: []
	};
	
	//响应是聊天对象的JSON编码版本。
	//聊天ID将添加到响应中，
	//因为它作为键存储，而不是聊天属性。
	res.setHeader('Content-Type', 'application/json');
	res.end(JSON.stringify(Object.assign({
		id: chatId
	}, chat)));
});
```

我们可以看到createChat()函数是从这个模块导出的，因为它被主应用程序模块中的路由器使用。我们还可以看到处理函数是一个生成器，并且使用co.wrap()包装。这是因为我们希望它是一个协程而不是普通函数。对formFields()的调用说明了我们在上一节中介绍的方法。请注意，我们产生了promise，并且我们得到了已解析的值。函数会在发生这种情况时阻塞，这一点至关重要，因为它是我们能够保持代码整洁并且没有过多回调的方式。

> 每个处理程序都使用了一些工具函数。为了节省页面空间，这里没有提及这些函数。但是，它们存在于本书附带
> 的代码中，并且它们已在评论中说明。


### 加入聊天处理程序

加入聊天处理程序是用户如何加入由其他用户创建的聊天。用户首先需要与他们共享聊天的URL。然后，他们可以向此接口提供他们的名字并发送请求，该接口将编码过的聊天ID作为URL的一部分。此处理程序的工作是将新用户推送到聊天的用户数组。我们现在来看看处理程序代码：

```javascript
//此接口允许用户加入现有的
//与他们共享的聊天(URL)。
exports.joinChat = co.wrap(function* (req, res, id) {
	if(!ensureMethod(req, res, 'POST')) {
		return;
	}

	//从内存加载聊天 - “chats”对象
	var chat = chats[id[1]];
	if(!ensureFound(req, res, chat)) {
		return;
	}

	//生成获取解析后的表单字段。
	//这个函数是使用“co.wrap()”创建的协程。
	var fields = yield formFields(req);
	chat.timestamp = new Date().getTime();

	//将新用户添加到聊天中。
	chat.users.push({
		timestamp: chat.timestamp,
		name: fields.user
	});
	
	//使用JSON编码的聊天字符串进行响应。
	//我们需要单独添加ID，因为它不是聊天属性。
	res.setHeader('Content-Type', 'application/json');
	res.end(JSON.stringify(Object.assign({
		id: id[1]
	}, chat)));
});
```

我们可能会注意到这个处理程序和创建聊天处理程序之间有很多相似之处。我们检查正确的HTTP方法，返回一个JSON响应，并将处理函数包装为协程，以便我们可以以完全避免回调函数的方式解析表单。主要区别在于我们更新现有聊天，而不是创建新聊天。


> 我们将新user对象推送到users数组的代码将被视为存储聊天。在实际的应用程序中，这意味着以某种方式将数据写入
> 磁盘 - 可能是对数据库的调用。这意味着发出异步请求。幸运的是，我们可以效仿与我们的表单解析使用相同的
> 技术 - 让它返回一个promise并利用已经存在的协程。


### 加载聊天处理程序

加载聊天处理程序的工作正如其名 - 使用URL中查找到的ID加载给定的聊天，并使用此聊天的JSON字符串进行响应。这是执行此操作的代码：

```javascript
//此接口加载聊天。这个函数
//没有被包装成一个协程，因为
//没有要等待的异步操作。
exports.loadChat = function(req, res, id) {

//查找聊天，使用URL中的“id”作为键。
var chat = chats[id[1]];

if(!ensureFound(req, res, chat)) {
	return;
}

//使用JSON编码的字符串版本进行响应聊天。
res.setHeader('Content-Type', 'application/json');
	res.end(JSON.stringify(chat));
};
```

这个函数没有co.wrap()调用，也没有生成器。这是因为它不需要。并不是说将这个函数作为一个包装成协程的生成器是不好的，这只是浪费。


> 这实际上是我们开发人员的一个例子，他们有意识地决定在不合理的情况下避免并发。这可能会改变这个处理程序的
> 道路，如果确实如此，我们将有工作要做。然而，权衡的是我们现在拥有更少的代码，而且运行速度更快。这对于
> 阅读它的人来说是有益的，因为它看起来不像是异步函数，因此不应该这样对待它。


### 发送消息处理程序

我们需要实现的最后一个主要API接口是发送消息。这就是给定聊天中的任何用户能够发布可供所有其他聊天参与者使用的消息的方式。这类似于加入聊天处理程序，除了我们将一个新的消息对象推送到messages数组。我们来看看处理程序代码;现在这个模式应该开始变得熟悉了：

```javascript
//这个处理程序将新消息发布到给定的聊天。
//它也是一个协程函数，因为它需要等待要完成的异步操作。
exports.sendMessage = co.wrap(function* (req, res, id) {
	if(!ensureMethod(req, res, 'POST')) {
		return;
	}

	//加载聊天并确保找到它。
	var chat = chats[id[1]];
	if (!ensureFound(req, res, chat)) {
		return;
	}

	//通过从“formFields()”返回生成的promis，
	//得到解析后的表单字段。
	var fields = yield formFields(req);
	chat.timestamp = new Date().getTime();

	//将新消息对象推送到“messages”属性。
	chat.messages.push({
		timestamp: chat.timestamp,
		user: fields.user,
		message: fields.message
	});
	
	res.setHeader('Content-Type', 'application/json');
	res.end(JSON.stringify(chat));
});
```

加入聊天时也适用同样的方法。修改聊天对象可能是实际应用程序中的异步操作，现在，我们的协程处理程序模式都已设置为我们在时机成熟时进行此更改。这是这些协程处理程序的关键，可以轻松地向处理程序添加新的异步操作，而不是非常困难。


### 静态处理程序

构成我们的聊天应用程序的最后一组处理程序是静态内容处理程序。它们的作用是将静态文件从文件系统提供给浏览器，例如index.html文档和JavaScript源代码。通常情况下，这是在Node应用程序之外处理的，但我们会将它们包含在此处，因为有时候更容易包含：

```javascript
//用于提供静态文件的辅助函数。
function serveFile(req, res, file) {

	//创建一个流来读取文件。
	var stream = fs.createReadStream(file);

	//当没有更多输入时结束响应。
	stream.on('end', () => {
		res.end();
	});

	//将输入文件传递给HTTP响应，
	//这是一个可写的流。
	stream.pipe(res);
}

//将请求的路径作为静态文件提供。
exports.staticFile = function(req, res) {
	serveFile(req, res, path.join(dirname, req.url));
};

//默认情况下，我们要提供“index.html”文件。
exports.index = function index(req, res) {
	res.setHeader('ContentType', 'text/html');
	serveFile(req, res, path.join(dirname, 'index.html'));
};
```

### 构建UI

我们现在有了目标API;是时候开始为我们的聊天应用创建用户界面了。我们将开始考虑与我们刚刚构建的API进行通信，然后一个个实现。接下来，我们将构建实际的HTML，我们需要渲染此应用程序需要使用的三个页面通。从这里，我们将走上也许是在前端最有挑战性的部分 - 构建DOM事件处理程序和交互。最后，我们会看到我们是否可以通过混入一个web worker来提高应用程序的响应。


### 与API通信

我们UI中的API通信方式本质上是并发的 - 它们通过网络连接发送和接收数据。因此，我们的应用程序体系结构的最有意思的是，我们需要花费时间尽可能地将同步机制与系统的其余部分隐藏起来。要与我们的API通信，我们将使用XMLHttpRequest类的实例。但是，正如我们在本书前面的章节中所看到的，这个类可以导致我们走向回调地狱。


我们知道，解决方案是使用promise来支持所有API数据的一致接口。这并不意味着我们需要一遍又一遍地抽象XMLHttpRequest类。我们创建了一个简单的工具函数来处理我们的并发封装，然后我们创建几个较小的特定于相应API接口的函数。下面的示图展示了这个想法：

![image206.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image206.gif?raw=true)


这种与异步API接口通信的方法可以很好地扩展，因为添加新功能只需添加一个小函数。所有同步语法都封装在一个api()函数中。让我们看看这代码：

```javascript
//用于向API接口发送HTTP请求的通用函数。
//“method”是HTTP方法，“path”是请求路径，
//“data”是可选的请求参数。
function api(method, path, data) {
	//返回一个被调用的promise，
	//解析API响应或失败拒绝。
	return new Promise((resolve, reject) => {
		var request = new XMLHttpRequest();
		
		//使用解析的JSON对象(通常是聊天)完成promise。
		request.addEventListener('load', (e) => {
			resolve(JSON.parse(e.target.responseText));
		});

		//当API出现问题时拒绝promise。
		request.addEventListener('error', (e) => {
			reject(e.target.statusText || 'unknown error');
		});

		request.addEventListener('abort', resolve); 
		request.open(method, path);

		//如果没有“data”，我们可以简单地“send()”请求。
		//否则，我们必须创建一个新“FormData”实例
		//来编码请求的表单数据。
		if (Object.is(data, undefined)) {
			request.send();
		} else {
			var form = new FormData();
			Object.keys(data).forEach((key) => {
				form.append(key, data[key]);
			});

			request.send(form);
		}
	});
}
```

这个函数很容易使用并且支持我们所有API的使用场景。较小的API函数，我们很快就会实现，可以简单地返回通过这个API()函数返回的promise。除此之外没有必要做得太复杂。


然而，这里有另一件事需要考虑。如果我们从本应用程序的中重新调用，那么API就没有任何过滤功能。这是关于UI的一个问题，因为我们不打算要重新渲染整个聊天对象。消息可以频繁发布，如果我们重新渲染了大量的信息，有一个很好的机会，该屏幕会在为我们呈现DOM元素时闪烁。所以，我们显然需要过滤浏览器中的聊天消息和用户;但这应该在哪里发生？


让我们在并发的背景下考虑这一点。假设我们决定在直接操作DOM的组件中执行过滤。这在某种意义上是好的，因为它意味着我们可以使用相同的数据来使用多个独立的组件，然后以不同的方式过滤它。当数据转换接近DOM时进行任何调整的并发性也很困难。例如，我们的应用程序不需要灵活性。只有一个组件可以呈现过滤后的数据。但是，它可能会受益于并发性。下图说明了另一种方法，我们实现的API功能执行过滤：

![image207.gif](https://github.com/yzsunlei/javascript_concurrency_translation/blob/master/images/image207.gif?raw=true)


使用这种方法，API函数与DOM分离得足够好。如果需要，我们可以稍后引入并发。现在让我们看一些特定的API函数，以及我们可以根据需要附加到给定API调用的过滤机制：

```javascript
//过滤“chat”对象以仅包含新用户和新消息。
//也就是说，带有较新“timestamp”数据还不是我们上次检查时间戳。
function filterChat(chat) {
	Object.assign(chat, {

		//将过滤后的数组分配给
		//对应的“chat”属性。
		users: chat.users.filter(
			user => user.timestamp > timestamp
		),
		messages: chat.messages.filter(
			message => message.timestamp > timestamp
		)
	});

	//重置“timestamp”，以便我们下次可以寻找更新的时间戳数据。
	//返回我们修改过的chat实例
	timestamp = chat.timestamp;
	return chat;
}

//使用给定的“topic”和“user”创建聊天。
//使用创建的聊天数据来解析返回的promise。
function createChat(topic, user) {
	return api('post', 'api/chat', {
		topic: topic,
		user: user
	});
}

//将给定的“user”加入给定的聊天“id”。
//返回加入了聊天数据的promise。
function joinChat(id, user) {
	return api('post', `api/chat/${id}/join`, {
		user: user
	}).then(filterChat);
}

//加载给定的聊天“id”。
//返回使用筛选的聊天数据的promise。
function loadChat(id) {
	return api('get', `api/chat/${id}`).then(filterChat);
};

//从给定的“user”发布“message”到给定的聊天“id”。
//返回使用过滤聊天数据的promise。
function sendMessage(id, user, message) {
	return api('post', `api/chat/${id}/message`, {
		user: user,
		message: message
	}).then(filterChat);
}
```

该filterChat()函数是简单的就够了。它只是修改给定的聊天对象以仅包含新用户和消息。新消息是那些时间戳大于此处使用的时间戳变量的消息。过滤完成后，时间戳是以聊天的更新时间戳属性。如果没有任何更改，这可能是相同的值，但如果某些内容发生更改，则会更新此值，以便不返回重复值。


我们可以看到，在我们的特定API函数中，filterChat()函数作为解析器传递给promise。所以我们在这里保留了一定程度的灵活性。例如，如果不同的组件需要以不同方式过滤聊天，我们可以引入一个使用相同方法的新函数，并添加一个相应过滤的不同的promise解析器函数。


### 实现HTML

我们的UI需要一些HTML才能呈现。聊天应用程序非常简单，只需一个HTML页面即可完成。我们可以将DOM结构组织成三个div元素，每个元素代表我们的页面。每个页面上的元素本身都很简单，因为在开发的这个阶段没有很多模块。我们的首要任务是功能 - 构建可用的功能。同时，我们应该考虑并发设计。这些项目绝对比构建弹性体系结构更有意义，而不是考虑小部件和虚拟DOM渲染库。这些是重要的考虑因素，但它们比错误的并发设计更容易解决。


我们来看看我们UI界面的HTML源代码。为这些元素定义了一些CSS样式。但是，它们是微不足道的，不在此处。例如，hide类用于切换给定页面的可见性。默认情况下，一切都是隐藏的。由我们的事件处理程序来处理这些元素的显示 - 我们将在下面介绍这些内容：

```html
<div id="create" class="hide">
	<h1>Create Chat</h1>
	<p>
		<label for="topic">Topic：</label>
		<input name="topic" id="topic" autofocus/>
	</p>
	<p>
		<label for="create-user">Your Name：</label>
		<input name="create-user" id="create-user"/>
	</p>
	<button>Create</button>
</div>

<div id="join" class="hide">
	<h1>Join Chat</h1>
	<p>
		<label for="join-user">Your Name：</label>
		<input name="join-user" id="join-user" autofocus/>
	</p>
	<button>Join</button>
</div>

<div id="view" class="hide">
	<h1></h1>
	<div>
		<div>
			<ul id="messages"></ul>
			<input placeholder="message" autofocus />
		</div>
		<ul id="users"></ul>
	</div>
</div>
```


### DOM事件和交互

我们现在有一些API通信机制和DOM元素。让我们将注意力转向应用程序的事件处理程序，以及它们如何与DOM交互。我们要解决的涉及最多的DOM操作行动就是描述聊天。也就是说，显示参与聊天的消息和用户。我们从这里开始吧。我们将实现一个drawChat()函数，因为它可能会在多个地方使用：

```javascript
//更新DOM中给定的“chat”。
function drawChat(chat) {
	
	//我们的主要DOM组件。
	//“$users”是聊天中的用户列表。
	//“$messages”就聊天中的消息列表。
	//“$view”是两个列表的容器元素。
	var $users = document.getElementById('users'),
		$messages = document.getElementById('messages'),
		$view = document.getElementById('view');

	//更新文档标题以反映聊天“topic”，
	//通过删除“隐藏”类显示聊天容器，
	//并用粗体标题更新聊标题。
	document.querySelector('title').textContent = chat.topic;
	$view.classList.remove('hide');
	$view.querySelector('h1').textContent = chat.topic;

	//迭代消息，
	//关于过滤或类似的事情不做任何假设。
	for (var messages of chat.messages) {
		
		//构造我们消息的用户部分需要的DOM元素。
		var $user = document.createElement('li'),
			$strong = document.createElement('strong'),
			$em = document.createElement('em');

		//组装DOM结构...
		$user.appendChild($strong);
		$user.appendChild($em);
		$user.classList.add('user');

		//添加内容 - 用户名和消息发布的时间。
		$strong.textContent = message.user + '';
		$em.textContent = new Date(message.timestamp).toLocaleString();

		//消息本身...
		var $message = document.createElement('li');
		$message.textContent = message.message;

		//附加用户部分和消息部分到DOM。
		$messages.appendChild($user);
		$messages.appendChild($message);
	}

	//在聊天中迭代用户，
	//对数据不做任何假设，只显示它。
	for (var users of chat.users) {
		var $user = document.createElement('li');
		$user.textContent = user.name;
		$users.appendChild($user);
	}

	//确保用户可以看到新呈现的内容。
	$messages.scrollTop = $messages.scrollHeight;

	//返回聊天，以便可以使用此函数
	//作为promise解析链中的解析器。
	return chat;
}
```

关于drawChat()函数有两个重要的注意事项。首先，这里没有完成聊天过滤。它假定任何消息和用户都是新的，它只是将它们附加到DOM。其次，我们实际上在渲染DOM之后返回聊天对象。起初这似乎是不必要的，但我们实际上将使用此函数作为promise解析器。这意味着如果我们想要向then()链添加更多的解析器，我们必须通过返回它来传递数据。


让我们看看load事件以突出强调前一点。聊天完成后，我们需要执行更多的工作。为此，我们可以使用另一个then()调用链接下一个函数：

```javascript
//页面加载时... 
window.addEventListener('load', (e) => {

	//“chatId”来自页面URL。
	//“user”可能已存在于localStorage中。
	var chatId = location.pathname.slice(1),
		user = localStorage.getItem('user'),
		$create = document.getElementById('create'),
		$join = document.getElementById('join');

	//如果URL中没有聊天ID，
	//则显示创建聊天提示，
	//如果在localStorage中找到，填充用户输入。
	if (!chatId) {
		$create.classList.remove('hide');
		if (user) {
			document.getElementById('create-user').value = user;
		}

		return;
	}

	//如果在localStorage中找不到用户名，
	//我们显示允许它们加入的提示，
	//通过在加入聊天之前输入他们的名字。
	if (!user) {
		$join.classList.remove('hide');
		return;
	}

	//我们加载聊天，使用drawChat()渲染它，
	//启动聊天轮询过程。
	api.postMessage({
		action: 'loadChat',
		chatId: chatId
	}).then(drawChat).then((chat) => {

		//如果用户不是聊天的一部分，我们加入它。
		//这会发生在用户名缓存在localStorage中的时候。
		//如果用户创建聊天，然后加载它们，
		//它们将属于这个聊天。
		if (chat.users.map(u => u.name).indexOf(user) < 0) {
			api.postMessage({
				action: 'joinChat',
				chatId: chatId,
				user: user
			}).then(drawChat).then(() => {
				poll(chatId);
			});
		} else {
			poll(chatId);
		};
	});
});
```

首次加载页面时会调用此处理程序，它首先需要根据当前URL检查是否要加载聊天。如果有，那么我们进行API调用以使用drawChat()作为解析器来加载聊天。但是，我们还需要执行一些额外的功能，并将其添加到链中的下一个then()解析器中。它的工作是确保用户实际上是聊天的一部分，为此，它需要我们刚刚从API加载的聊天，它是从drawChat()传递的。在我们进行进一步的API调用以将用户添加到聊天之后，如有必要，我们启动轮询机制。这就是我们如何使用新消息和新用户加入聊天来保持UI的最新状态：

```javascript
//开始对给定聊天“id”的API进行轮询。
function poll(chatId) {
	setInterval(() => {
		api.postMessage({
			action: 'loadChat',
			chatId: chatId
		}).then(drawChat);
	}, 3000);
}
```

您可能已经注意到我们正在使用类似web worker的很奇怪的调用 - api.postMessage()。这是因为它是一个Web worker，这是我们接下来要介绍的内容。


> 为了节省空间，我们省去了与创建聊天，加入聊天和发送消息相关的其他三个DOM事件处理程序。与我们刚才
> 介绍的加载处理程序相比，它们在并发性方面没有什么不同。


### 添加API worker

早些时候，当我们实现API通信功能时，我们认为从并发角度来看，过滤组件与API相结合而不是DOM更有意义。现在是时候从这个决定中受益，并将我们的API代码封装在Web worker中。我们想要这样做的主要原因是filterChat()函数有可能锁定响应性。换句话说，对于较大的聊天，这将花费更长的时间来完成，并且文本输入将停止响应用户输入。例如，在我们尝试渲染更新列表时，没有理由阻止用户发送消息。


首先，我们需要扩展worker API以使postMessage()返回一个promise。这就跟我们在“第7章，抽象并发”所做的那样。看看下面的代码：

```javascript
//这将生成唯一ID。
//我们需要它们将Web worker执行的任务
//映射到更大的创建它们的操作上。
function* genID() {
	var id = 0;
	while (true) {
		yield id++;
	}
}

//创建全局“id”生成器。
var id = genID();

//这个对象包含promises的解析器函数，
//结果从worker那里回来，我们在这里查看，基于ID。
var resolvers = {};
var rejectors = {};

//保存“postMessage()”的原始实现，
//这样我们可以稍后在我们的自定义“postMessage()”中调用它执行。
var postMessage = Worker.prototype.postMessage;

//用我们的自定义实现替换“postMessage()”。
Worker.prototype.postMessage = function(data) {
	return new Promise((resolve, reject)=> {

		//用于将Web worker响应和解析器函数绑定在一起的ID。
		var msgId = id.next().value;

		//保存解析器以便以后在Web worker消息回调可以使用。
		resolvers[msgId] = resolver;

		rejectors[msgId] = reject;
		
		//运行原始的“Worker.postMessage()”实现，
		//负责实际上将消息发布到worker线程。
		postMessage.call(this, Object.assign({
			msgId: msgId
		}, data));
	});
};

//开始我们的worker...
var api = new Worker('ui-api.js');

//当worker响应时，解析由“postMessage()”返回的promise。
api.addEventListener('message', (e) => {

	//如果数据处于错误状态，那么我们想要拒绝函数，
	//并且使用那个错误消息调用它。
	//否则，使用worker返回的数据的调用普通解析器函数。
	var source = e.data.error ? rejectors : resolvers,
		callback = source[e.data.msgId],
		data = e.data.error ? e.data.error : e.data;
		
	callback(data);
	
	//不需要了，删除它们。
	delete resolvers[e.data.msgId];
	delete rejectors[e.data.msgId];
});
```

有一个小细节，我们在“第7章，抽象并发”并没有涵盖它，拒绝promise的这种技术。例如，如果API调用由于某种原因失败，我们必须确保主线程中等待worker的promise也被拒绝;否则，奇怪的错误将开始出现。


现在，我们需要对我们的ui-api.js模块进行补充，其中定义了所有API函数以适应它在Web worker中运行的事实。我们只需要添加以下事件处理程序：

```javascript
//监听来自主线程的消息。
addEventListener('message', (e) => {

	//通用的promise解析器函数。
	//它的是使用“postMessage()”将数据发布回主线程。
	//它也返回了数据，以便可以进一步使用promise解析链。
	function resolve(data) {
		postMessage(Object.assign({
			msgId: e.data.msgId
		}, data));

		return data;
	}

	//通用拒绝函数将数据发回到主线程。
	//这里的区别在于它将数据标记为错误。
	//这允许promise在最后被拒绝。
	function reject(error) {
		postMessage({
			msgId: e.data.msgId,
			error: error.toString()
		});

		return error;
	}

	//此开关决定在“action”消息属性上调用哪个函数。
	//“resolve()”函数作为解析器传递给每个返回的promise。
	switch (e.data.action) {
		case 'createChat': 
			createChat(e.data.topic, e.data.user)
				.then(resolve, reject);
			break;
		case 'joinChat': 
			joinChat(e.data.chatId, e.data.user)
				.then(resolve, reject);
			break;
		case'loadChat':
			loadChat(e.data.chatId)
				.then(resolve, reject);
			break;
		case'sendMessage':
			sendMessage(
				e.data.chatId,
				e.data.user,
				e.data.message
			).then(resolve, reject);
			break;
	}
});
```

这个消息事件处理程序是我们能够与主线程通信的方式。该行动财产是我们现在能够确定要调用的API端点。所以现在，每当我们对聊天消息执行任何昂贵的过滤时，它都在一个单独的线程中。


引入这个worker的另一个结果是它将API功能封装成一个有凝聚力的整体。现在可以将API Web worker组件视为整个较大UI中的较小应用程序。


### 增强和改进

这就是我们对聊天应用程序开发覆盖的范围。我们没有遍历每一段代码，但这就是为什么代码可以作为本书的配套提供，可以完整地查看它。前面几节的重点是过一遍编写并发JavaScript代码的过程。我们没有利用本章之前章节中的每一个示例，这将破坏并发的整个目的，以修复导致次优用户体验的问题。


聊天应用程序示例的重点是促进并发性。这意味着可以在需要时实现并发代码，而不是为了实现并发代码。后者并没有使我们的应用程序比现在更好，也没有让我们更好地解决以后发生的并发问题。


我们将贯穿本章可能值得我们的聊天应用考虑的几个方面。你，作为读者，也鼓励一起开发聊天应用程序代码，并看看遵循这些点是否是适用的。你将如何去支持他们？我们是否需要改变我们的设计？这一点是在我们的JavaScript应用程序中并发性的设计不是一次性发生的，而是随着我们的应用程序不断发展设计任务也一起改变的。


### 集群API

在“第9章，NodeJS高级并发”，向您介绍了NodeJS中的cluster模块。这直接地扩展了HTTP服务器的请求处理能力。此模块通过将Node进程分配到多个子进程来进行工作。因为他们每个人都是自己的进程，所以他们有自己的事件循环。此外，不需要额外的通信同步代码。


我们并不需要花费太多精力将这些集群功能添加到我们的app.js模块中。但问题是 - 我们在什么时候决定集群是值得的？我们要等到实际出现性能问题，或者我们只是自动打开它？这些是事先很难知道的事情。实际情况是，它取决于我们的请求处理程序如何获得CPU密集型。这些变化通常是由于新功能被添加到软件中而产生的。


我们的聊天应用程序是否需要集群？也许，总有一天。但是处理程序确实没有执行任何工作。这总是可以改变的。也许我们可以继续实现集群功能，但也添加一个让我们可以关闭的选项。


### 清理聊天记录

我们的聊天应用程序没有任何持久存储;它将所有聊天数据保存在内存中。这对我们的特定用例来说很好，因为它适用于想要启动临时聊天的用户，以便他们可以与人共享链接而不是必须经过注册过程。这里的问题是，在聊天不再被使用之后，它的数据仍然占用内存。最终，这对我们的Node进程来说是致命的。


如果我们决定实现清理服务，其工作是定期迭代聊天数据并且在给定时间内未被修改的聊天将被删除，该怎么办？这将只保留内存中的活动聊天。


### 异步入口

我们早期决定在大多数请求处理程序中使用协程。这些处理程序使用的唯一异步操作是表单解析行为。但是，在任何给定的处理程序中，这仍然是唯一的异步操作的可能性很小。特别是随着我们的应用程序的成长，我们将开始依赖更多的核心NodeJS功能，这意味着我们将要在promises中包含更多的异步回调式代码。我们可能也会开始依赖我们自己的或第三方软件的外部服务。


我们是否可以将我们的异步架构更进一步，并为那些希望扩展系统的人提供这些处理程序的入口？例如，如果请求是创建聊天请求，请在创建已提供的聊天扩展之前向任何请求发送请求。像这样的事情是一项艰巨的任务，并且容易出错。但是对于具有许多模块的大型系统，所有这些都是异步的，最好将异步入口标准化到系统中。


### 谁在打字？

我们在聊天应用程序中遗漏了给定用户的输入状态。这是一种机制，通知聊天的所有其他成员特定用户正在键入消息并且它几乎存在于所有现代聊天系统中。


鉴于我们目前的设计，我们需要实现这样的功能吗？轮询机制是否足以应对这种不断变化的状态？数据模型是否必须改变很多，这样的改变是否会导致我们的请求处理程序出现问题？


### 离开聊天

我们的聊天应用程序中缺少的另一个功能是删除不再参与聊天的用户。例如，对于其他聊天参与者来说，在聊天中看到的用户不是真的这有意义吗？是否需要监听卸载事件并实现新的离开聊天API接口，或者还有更多内容吗？


### 轮询超时

我们刚刚构建的聊天应用程序几乎没有错误处理。特别值得一提的一个案例就是在超时时杀死轮询机制。通过这个，我们谈谈防止客户端重复失败的请求尝试。假设服务器已关闭，或者由于引入了错误，处理程序只是失败了;我们是否希望轮询机制无限运行下去？我们不希望它这样做，并且可能有一些事情可以做到。


例如，我们需要取消轮询开始时调用setInterval()时设置的定时器。同样，我们需要一种方法来跟踪连续失败尝试的次数，因此我们知道何时关闭它。


### 小结

希望这个简单的聊天应用程序的示例让您对端到端设计并发JavaScript应用程序所涉及的内容有了新的认识。本书首先概述了并发性，特别是在JavaScript应用程序的上下文中，因为它与其他编程语言环境不同。然后，我们介绍了一些指导原则，以帮助我们一步步实现。


本章我们对各种语言和环境并发机制进行严格审查实际上只是为达到目的的手段。我们的结局 - JavaScript程序员和架构师 - 是一个没有并发问题的应用程序。这是一个广泛的声明，但最终，我们在Web应用程序中遇到的许多问题都是并发设计不充分的直接结果。


所以使用这些原则。使用JavaScript中提供的强大并发功能。结合这两点来开发超出用户期望的优秀应用程序。当我们编写代码默认就是并发时，许多JavaScript编程挑战就会消失。
