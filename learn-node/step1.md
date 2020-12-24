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

<pre class="file" data-filename="helloworld.js" data-target="insert">
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