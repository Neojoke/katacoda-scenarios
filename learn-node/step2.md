# 如何来进行请求的“路由”

我们要为路由提供请求的URL和其他需要的GET及POST参数，随后路由需要根据这些数据来执行相应的代码（这里“代码”对应整个应用的第三部分：一系列在接收到请求时真正工作的处理程序）。

因此，我们需要查看HTTP请求，从中提取出请求的URL以及GET/POST参数。这一功能应当属于路由还是服务器（甚至作为一个模块自身的功能）确实值得探讨，但这里暂定其为我们的HTTP服务器的功能。

我们需要的所有数据都会包含在request对象中，该对象作为onRequest()回调函数的第一个参数传递。但是为了解析这些数据，我们需要额外的Node.JS模块，它们分别是url和querystring模块。

```

                               url.parse(string).query
                                           |
           url.parse(string).pathname      |
                       |                   |
                       |                   |
                     ------ -------------------
http://localhost:8888/start?foo=bar&hello=world
                                ---       -----
                                 |          |
                                 |          |
              querystring(string)["foo"]    |
                                            |
                         querystring(string)["hello"]

```

当然我们也可以用querystring模块来解析POST请求体中的参数，稍后会有演示。

现在我们来给onRequest()函数加上一些逻辑，用来找出浏览器请求的URL路径：


<pre class="file" data-filename="server.js" data-target="insert">

var http = require("http");
var url = require("url");

function start() {
  function onRequest(request, response) {
    var pathname = url.parse(request.url).pathname;
    console.log("Request for " + pathname + " received.");
    response.writeHead(200, {"Content-Type": "text/plain"});
    response.write("Hello World");
    response.end();
  }

  http.createServer(onRequest).listen(8888);
  console.log("Server has started.");
}

exports.start = start;
</pre>

好了，我们的应用现在可以通过请求的URL路径来区别不同请求了--这使我们得以使用路由（还未完成）来将请求以URL路径为基准映射到处理程序上。

在我们所要构建的应用中，这意味着来自/start和/upload的请求可以使用不同的代码来处理。稍后我们将看到这些内容是如何整合到一起的。

现在我们可以来编写路由了，建立一个名为router.js的文件，添加以下内容：

<pre class="file" data-filename="router.js" data-target="insert">

function route(pathname) {
  console.log("About to route a request for " + pathname);
}

exports.route = route;
</pre>

如你所见，这段代码什么也没干，不过对于现在来说这是应该的。在添加更多的逻辑以前，我们先来看看如何把路由和服务器整合起来。

我们的服务器应当知道路由的存在并加以有效利用。我们当然可以通过硬编码的方式将这一依赖项绑定到服务器上，但是其它语言的编程经验告诉我们这会是一件非常痛苦的事，因此我们将使用依赖注入的方式较松散地添加路由模块（你可以读读[Martin Fowlers关于依赖注入的大作](http://martinfowler.com/articles/injection.html)来作为背景知识）。

首先，我们来扩展一下服务器的start()函数，以便将路由函数作为参数传递过去：

<pre class="file" data-filename="server.js" data-target="replace">

var http = require("http");
var url = require("url");

function start(route) {
  function onRequest(request, response) {
    var pathname = url.parse(request.url).pathname;
    console.log("Request for " + pathname + " received.");

    route(pathname);

    response.writeHead(200, {"Content-Type": "text/plain"});
    response.write("Hello World");
    response.end();
  }

  http.createServer(onRequest).listen(8888);
  console.log("Server has started.");
}

exports.start = start;
</pre>

同时，我们会相应扩展index.js，使得路由函数可以被注入到服务器中：

<pre class="file" data-filename="index.js" data-target="insert">
var server = require("./server");
var router = require("./router");

server.start(router.route);
</pre>

在这里，我们传递的函数依旧什么也没做。

如果现在启动应用（node index.js，始终记得这个命令行），

`node index.js`{{execute interrupt}}


在浏览器访问  <https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/foo> ，

随后请求一个URL，你将会看到应用输出相应的信息，这表明我们的HTTP服务器已经在使用路由模块了，并会将请求的路径传递给路由：

```
bash$ node index.js
Request for /foo received.
About to route a request for /foo
```

（以上输出已经去掉了比较烦人的/favicon.ico请求相关的部分）。


## 行为驱动执行

请允许我再次脱离主题，在这里谈一谈函数式编程。

将函数作为参数传递并不仅仅出于技术上的考量。对软件设计来说，这其实是个哲学问题。想想这样的场景：在index文件中，我们可以将router对象传递进去，服务器随后可以调用这个对象的route函数。

就像这样，我们传递一个东西，然后服务器利用这个东西来完成一些事。嗨，那个叫路由的东西，能帮我把这个路由一下吗？

但是服务器其实不需要这样的东西。它只需要把事情做完就行，其实为了把事情做完，你根本不需要东西，你需要的是动作。也就是说，你不需要名词，你需要动词。

理解了这个概念里最核心、最基本的思想转换后，我自然而然地理解了函数编程。

我是在读了Steve Yegge的大作[名词王国中的死刑](http://steve-yegge.blogspot.com/2006/03/execution-in-kingdom-of-nouns.html)之后理解函数编程。你也去读一读这本书吧，真的。这是曾给予我阅读的快乐的关于软件的书籍之一。

## 路由给真正的请求处理程序

回到正题，现在我们的HTTP服务器和请求路由模块已经如我们的期望，可以相互交流了，就像一对亲密无间的兄弟。

当然这还远远不够，路由，顾名思义，是指我们要针对不同的URL有不同的处理方式。例如处理/start的“业务逻辑”就应该和处理/upload的不同。

在现在的实现下，路由过程会在路由模块中“结束”，并且路由模块并不是真正针对请求“采取行动”的模块，否则当我们的应用程序变得更为复杂时，将无法很好地扩展。

我们暂时把作为路由目标的函数称为请求处理程序。现在我们不要急着来开发路由模块，因为如果请求处理程序没有就绪的话，再怎么完善路由模块也没有多大意义。

应用程序需要新的部件，因此加入新的模块 -- 已经无需为此感到新奇了。我们来创建一个叫做requestHandlers的模块，并对于每一个请求处理程序，添加一个占位用函数，随后将这些函数作为模块的方法导出：

<pre class="file" data-filename="requestHandlers.js" data-target="insert">
function start() {
  console.log("Request handler 'start' was called.");
}

function upload() {
  console.log("Request handler 'upload' was called.");
}

exports.start = start;
exports.upload = upload;
</pre>

这样我们就可以把请求处理程序和路由模块连接起来，让路由“有路可寻”。

在这里我们得做个决定：是将requestHandlers模块硬编码到路由里来使用，还是再添加一点依赖注入？虽然和其他模式一样，依赖注入不应该仅仅为使用而使用，但在现在这个情况下，使用依赖注入可以让路由和请求处理程序之间的耦合更加松散，也因此能让路由的重用性更高。

这意味着我们得将请求处理程序从服务器传递到路由中，但感觉上这么做更离谱了，我们得一路把这堆请求处理程序从我们的主文件传递到服务器中，再将之从服务器传递到路由。

那么我们要怎么传递这些请求处理程序呢？别看现在我们只有2个处理程序，在一个真实的应用中，请求处理程序的数量会不断增加，我们当然不想每次有一个新的URL或请求处理程序时，都要为了在路由里完成请求到处理程序的映射而反复折腾。除此之外，在路由里有一大堆if request == x then call handler y也使得系统丑陋不堪。

仔细想想，有一大堆东西，每个都要映射到一个字符串（就是请求的URL）上？似乎关联数组（associative array）能完美胜任。

不过结果有点令人失望，JavaScript没提供关联数组 -- 也可以说它提供了？事实上，在JavaScript中，真正能提供此类功能的是它的对象。

在这方面， <http://msdn.microsoft.com/en-us/magazine/cc163419.aspx> 有一个不错的介绍，我在此摘录一段：


> /在C++或C#中，当我们谈到对象，指的是类或者结构体的实例。对象根据他们实例化的模板（就是所谓的类），会拥有不同的属性和方法。但在JavaScript里对象不是这个概念。在JavaScript中，对象就是一个键/值对的集合 — 你可以把JavaScript的对象想象成一个键为字符串类型的字典。/

但如果JavaScript的对象仅仅是键/值对的集合，它又怎么会拥有方法呢？好吧，这里的值可以是字符串、数字或者……函数！

好了，最后再回到代码上来。现在我们已经确定将一系列请求处理程序通过一个对象来传递，并且需要使用松耦合的方式将这个对象注入到route()函数中。

我们先将这个对象引入到主文件index.js中：


<pre class="file" data-filename="index.js" data-target="replace">
var server = require("./server");
var router = require("./router");
var requestHandlers = require("./requestHandlers");

var handle = {}
handle["/"] = requestHandlers.start;
handle["/start"] = requestHandlers.start;
handle["/upload"] = requestHandlers.upload;

server.start(router.route, handle);
</pre>

虽然handle并不仅仅是一个“东西”（一些请求处理程序的集合），我还是建议以一个动词作为其命名，这样做可以让我们在路由中使用更流畅的表达式，稍后会有说明。

正如所见，将不同的URL映射到相同的请求处理程序上是很容易的：只要在对象中添加一个键为"/"的属性，对应requestHandlers.start即可，这样我们就可以干净简洁地配置/start和/的请求都交由start这一处理程序处理。

在完成了对象的定义后，我们把它作为额外的参数传递给服务器，为此将server.js修改如下：

<pre class="file" data-filename="server.js" data-target="replace">

var http = require("http");
var url = require("url");

function start(route, handle) {
  function onRequest(request, response) {
    var pathname = url.parse(request.url).pathname;
    console.log("Request for " + pathname + " received.");

    route(handle, pathname);

    response.writeHead(200, {"Content-Type": "text/plain"});
    response.write("Hello World");
    response.end();
  }

  http.createServer(onRequest).listen(8888);
  console.log("Server has started.");
}

exports.start = start;
</pre>

这样我们就在start()函数里添加了handle参数，并且把handle对象作为第一个参数传递给了route()回调函数。

然后我们相应地在route.js文件中修改route()函数：

<pre class="file" data-filename="router.js" data-target="replace">

function route(handle, pathname) {
  console.log("About to route a request for " + pathname);
  if (typeof handle[pathname] === 'function') {
    handle[pathname]();
  } else {
    console.log("No request handler found for " + pathname);
  }
}

exports.route = route;

</pre>

通过以上代码，我们首先检查给定的路径对应的请求处理程序是否存在，如果存在的话直接调用相应的函数。我们可以用从关联数组中获取元素一样的方式从传递的对象中获取请求处理函数，因此就有了简洁流畅的形如handle\[pathname\]();的表达式，这个感觉就像在前方中提到的那样：“嗨，请帮我处理了这个路径”。

有了这些，我们就把服务器、路由和请求处理程序在一起了。

现在我们启动应用程序

`node index.js`{{execute interrupt}}

并在浏览器中访问 <https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/start> ，

以下日志可以说明系统调用了正确的请求处理程序：

```
Server has started.
Request for /start received.
About to route a request for /start
Request handler 'start' was called.
```

并且在浏览器中打开 <https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/> 可以看到这个请求同样被start请求处理程序处理了：

```
Request for / received.
About to route a request for /
Request handler 'start' was called.
```

# 让请求处理程序作出响应

很好。不过现在要是请求处理程序能够向浏览器返回一些有意义的信息而并非全是“Hello World”，那就更好了。

这里要记住的是，浏览器发出请求后获得并显示的“Hello World”信息仍是来自于我们server.js文件中的onRequest函数。

其实“处理请求”说白了就是“对请求作出响应”，因此，我们需要让请求处理程序能够像onRequest函数那样可以和浏览器进行“对话”。

## 不好的实现方式

对于我们这样拥有PHP或者Ruby技术背景的开发者来说，最直截了当的实现方式事实上并不是非常靠谱： 看似有效，实则未必如此。

这里我指的“直截了当的实现方式”意思是：让请求处理程序通过onRequest函数直接返回（return()）他们要展示给用户的信息。

我们先就这样去实现，然后再来看为什么这不是一种很好的实现方式。

让我们从让请求处理程序返回需要在浏览器中显示的信息开始。我们需要将requestHandler.js修改为如下形式：

<pre class="file" data-filename="requestHandlers.js" data-target="replace">
function start() {
  console.log("Request handler 'start' was called.");
  return "Hello Start";
}

function upload() {
  console.log("Request handler 'upload' was called.");
  return "Hello Upload";
}

exports.start = start;
exports.upload = upload;
</pre>

好的。同样的，请求路由需要将请求处理程序返回给它的信息返回给服务器。因此，我们需要将router.js修改为如下形式：

<pre class="file" data-filename="router.js" data-target="replace">

function route(handle, pathname) {
  console.log("About to route a request for " + pathname);
  if (typeof handle[pathname] === 'function') {
    return handle[pathname]();
  } else {
    console.log("No request handler found for " + pathname);
    return "404 Not found";
  }
}

exports.route = route;

</pre>

正如上述代码所示，当请求无法路由的时候，我们也返回了一些相关的错误信息。

最后，我们需要对我们的server.js进行重构以使得它能够将请求处理程序通过请求路由返回的内容响应给浏览器，如下所示：

<pre class="file" data-filename="server.js" data-target="replace">
var http = require("http");
var url = require("url");

function start(route, handle) {
  function onRequest(request, response) {
    var pathname = url.parse(request.url).pathname;
    console.log("Request for " + pathname + " received.");

    response.writeHead(200, {"Content-Type": "text/plain"});
    var content = route(handle, pathname)
    response.write(content);
    response.end();
  }

  http.createServer(onRequest).listen(8888);
  console.log("Server has started.");
}

exports.start = start;

</pre>

如果我们运行重构后的应用，

`node index.js`{{execute interrupt}}

一切都会工作的很好：
请求 <https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/start>  ,浏览器会输出“Hello Start”，
请求 <https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/upload>  会输出“Hello Upload”,
而请求 <https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/foo>  会输出“404 Not found”。

好，那么问题在哪里呢？简单的说就是： 当未来有请求处理程序需要进行非阻塞的操作的时候，我们的应用就“挂”了。

没理解？没关系，下面就来详细解释下。