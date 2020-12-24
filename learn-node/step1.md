# “Hello World”
好了，“废话”不多说了，马上开始我们第一个Node.js应用：“Hello World”。

打开你最喜欢的编辑器，创建一个helloworld.js文件。我们要做就是向STDOUT输出“Hello World”，如下是实现该功能的代码：

<pre class="file" data-filename="helloworld.js" data-target="insert">
console.log("Hello World");
</pre>

通过Node.js来执行：

`node helloworld.js`{{execute interrupt}}

正常的话，就会在终端输出Hello World 。

好吧，我承认这个应用是有点无趣，那么下面我们就来点“干货”。

# 一个完整的基于Node.js的web应用

## 用例
我们来把目标设定得简单点，不过也要够实际才行：

用户可以通过浏览器使用我们的应用。

当用户请求 http://domain/start 时，可以看到一个欢迎页面，页面上有一个文件上传的表单。

用户可以选择一个图片并提交表单，随后文件将被上传到 http://domain/upload ，该页面完成上传后会把图片显示在页面上。

进一步地说，在完成这一目标的过程中，我们不仅仅需要基础的代码而不管代码是否优雅。我们还要对此进行抽象，来寻找一种适合构建更为复杂的Node.js应用的方式。

## 应用不同模块分析

我们来分解一下这个应用，为了实现上文的用例，我们需要实现哪些部分呢？

* 我们需要提供Web页面，因此需要一个HTTP服务器
* 对于不同的请求，根据请求的URL，我们的服务器需要给予不同的响应，因此我们需要一个路由，用于把请求对应到请求处理程序（request handler）
* 当请求被服务器接收并通过路由传递之后，需要可以对其进行处理，因此我们需要最终的请求处理程序
* 路由还应该能处理POST数据，并且把数据封装成更友好的格式传递给请求处理入程序，因此需要请求数据处理功能
* 我们不仅仅要处理URL对应的请求，还要把内容显示出来，这意味着我们需要一些视图逻辑供请求处理程序使用，以便将内容发送给用户的浏览器
* 最后，用户需要上传图片，所以我们需要上传处理功能来处理这方面的细节

我们先来想想，使用PHP的话我们会怎么构建这个结构。一般来说我们会用一个Apache HTTP服务器并配上mod_php5模块。
从这个角度看，整个“接收HTTP请求并提供Web页面”的需求根本不需要PHP来处理。

不过对Node.js来说，概念完全不一样了。使用Node.js时，我们不仅仅在实现一个应用，同时还实现了整个HTTP服务器。事实上，我们的Web应用以及对应的Web服务器基本上是一样的。

听起来好像有一大堆活要做，但随后我们会逐渐意识到，对Node.js来说这并不是什么麻烦的事。

现在我们就来开始实现之路，先从第一个部分--HTTP服务器着手。

# 构建应用的模块

## 一个基础的HTTP服务器

当我准备开始写我的第一个“真正的”Node.js应用的时候，我不但不知道怎么写Node.js代码，也不知道怎么组织这些代码。 
我应该把所有东西都放进一个文件里吗？网上有很多教程都会教你把所有的逻辑都放进一个用Node.js写的基础HTTP服务器里。但是如果我想加入更多的内容，同时还想保持代码的可读性呢？

实际上，只要把不同功能的代码放入不同的模块中，保持代码分离还是相当简单的。

这种方法允许你拥有一个干净的主文件（main file），你可以用Node.js执行它；同时你可以拥有干净的模块，它们可以被主文件和其他的模块调用。

那么，现在我们来创建一个用于启动我们的应用的主文件，和一个保存着我们的HTTP服务器代码的模块。

在我的印象里，把主文件叫做index.js或多或少是个标准格式。把服务器模块放进叫server.js的文件里则很好理解。

让我们先从服务器模块开始。在你的项目的根目录下创建一个叫server.js的文件，并写入以下代码：

<pre class="file" data-filename="server.js" data-target="insert">
var http = require("http");

http.createServer(function(request, response) {
  response.writeHead(200, {"Content-Type": "text/plain"});
  response.write("Hello World");
  response.end();
}).listen(8888);
</pre>

搞定！你刚刚完成了一个可以工作的HTTP服务器。为了证明这一点，我们来运行并且测试这段代码。首先，用Node.js执行你的脚本：

`node server.js`{{execute interrupt}}

接下来，打开浏览器访问 <https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/> ，你会看到一个写着“Hello World”的网页。

这很有趣，不是吗？让我们先来谈谈HTTP服务器的问题，把如何组织项目的事情先放一边吧，你觉得如何？我保证之后我们会解决那个问题的。

## 分析HTTP服务器

那么接下来，让我们分析一下这个HTTP服务器的构成。

第一行请求（require）Node.js自带的 http 模块，并且把它赋值给 http 变量。

接下来我们调用http模块提供的函数： createServer 。这个函数会返回一个对象，这个对象有一个叫做 listen 的方法，这个方法有一个数值参数，指定这个HTTP服务器监听的端口号。

咱们暂时先不管 http.createServer 的括号里的那个函数定义。

我们本来可以用这样的代码来启动服务器并侦听8888端口：

```
var http = require("http");

var server = http.createServer();
server.listen(8888);
```

这段代码只会启动一个侦听8888端口的服务器，它不做任何别的事情，甚至连请求都不会应答。

最有趣（而且，如果你之前习惯使用一个更加保守的语言，比如PHP，它还很奇怪）的部分是 createServer() 的第一个参数，一个函数定义。

实际上，这个函数定义是 createServer() 的第一个也是唯一一个参数。因为在JavaScript中，函数和其他变量一样都是可以被传递的。

## 进行函数传递

举例来说，你可以这样做：

<pre class="file" data-filename="demo.js" data-target="insert">
function say(word) {
  console.log(word);
}

function execute(someFunction, value) {
  someFunction(value);
}

execute(say, "Hello");
</pre>

请仔细阅读这段代码！在这里，我们把 say 函数作为execute函数的第一个变量进行了传递。这里传递的不是 say 的返回值，而是 say 本身！

这样一来， say 就变成了execute 中的本地变量 someFunction ，execute可以通过调用 someFunction() （带括号的形式）来使用 say 函数。

当然，因为 say 有一个变量， execute 在调用 someFunction 时可以传递这样一个变量。

`node demo.js`{{execute interrupt}}

我们可以，就像刚才那样，用它的名字把一个函数作为变量传递。但是我们不一定要绕这个“先定义，再传递”的圈子，我们可以直接在另一个函数的括号中定义和传递这个函数：

<pre class="file" data-filename="demo.js" data-target="replace">
function execute(someFunction, value) {
  someFunction(value);
}

execute(function(word){ console.log(word) }, "Hello");
</pre>


我们在 execute 接受第一个参数的地方直接定义了我们准备传递给 execute 的函数。

用这种方式，我们甚至不用给这个函数起名字，这也是为什么它被叫做 匿名函数 。

`node demo.js`{{execute interrupt}}

这是我们和我所认为的“进阶”JavaScript的第一次亲密接触，不过我们还是得循序渐进。现在，我们先接受这一点：在JavaScript中，一个函数可以作为另一个函数接收一个参数。我们可以先定义一个函数，然后传递，也可以在传递参数的地方直接定义函数。

# 函数传递是如何让HTTP服务器工作的

带着这些知识，我们再来看看我们简约而不简单的HTTP服务器：

