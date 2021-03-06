# 处理POST请求

考虑这样一个简单的例子：我们显示一个文本区（textarea）供用户输入内容，然后通过POST请求提交给服务器。最后，服务器接受到请求，通过处理程序将输入的内容展示到浏览器中。

/start请求处理程序用于生成带文本区的表单，因此，我们将requestHandlers.js修改为如下形式：



<pre class="file" data-filename="requestHandlers.js" data-target="replace">

function start(response) {
  console.log("Request handler 'start' was called.");

  var body = '&lt;html&gt;'+
    '&lt;head&gt;'+
    '&lt;meta http-equiv="Content-Type" content="text/html; '+
    'charset=UTF-8" /&gt;'+
    '&lt;/head&gt;'+
    '&lt;body&gt;'+
    '&lt;form action="/upload" method="post"&gt;'+
    '&lt;textarea name="text" rows="20" cols="60"&gt;&lt;/textarea&gt;'+
    '&lt;input type="submit" value="Submit text" /&gt;'+
    '&lt;/form&gt;'+
    '&lt;/body&gt;'+
    '&lt;/html&gt;';

    response.writeHead(200, {"Content-Type": "text/html"});
    response.write(body);
    response.end();
}

function upload(response) {
  console.log("Request handler 'upload' was called.");
  response.writeHead(200, {"Content-Type": "text/plain"});
  response.write("Hello Upload");
  response.end();
}

exports.start = start;
exports.upload = upload;

</pre>


好了，现在我们的应用已经很完善了，都可以获得威比奖（Webby Awards）了，哈哈。（译者注：威比奖是由国际数字艺术与科学学院主办的评选全球最佳网站的奖项，具体参见详细说明）

重启应用:

`node index.js`{{execute interrupt}}

通过在浏览器中访问
<https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/start>

就可以看到简单的表单了!

你可能会说：这种直接将视觉元素放在请求处理程序中的方式太丑陋了。说的没错，但是，我并不想在本书中介绍诸如MVC之类的模式，因为这对于你了解JavaScript或者Node.js环境来说没多大关系。

余下的篇幅，我们来探讨一个更有趣的问题： 当用户提交表单时，触发/upload请求处理程序处理POST请求的问题。

现在，我们已经是新手中的专家了，很自然会想到采用异步回调来实现非阻塞地处理POST请求的数据。

这里采用非阻塞方式处理是明智的，因为POST请求一般都比较“重” —— 用户可能会输入大量的内容。用阻塞的方式处理大数据量的请求必然会导致用户操作的阻塞。

为了使整个过程非阻塞，Node.js会将POST数据拆分成很多小的数据块，然后通过触发特定的事件，将这些小数据块传递给回调函数。这里的特定的事件有data事件（表示新的小数据块到达了）以及end事件（表示所有的数据都已经接收完毕）。

我们需要告诉Node.js当这些事件触发的时候，回调哪些函数。怎么告诉呢？ 我们通过在request对象上注册监听器（listener） 来实现。这里的request对象是每次接收到HTTP请求时候，都会把该对象传递给onRequest回调函数。

如下所示：

```
request.addListener("data", function(chunk) {
  // called when a new chunk of data was received
});

request.addListener("end", function() {
  // called when all chunks of data have been received
});
```
问题来了，这部分逻辑写在哪里呢？ 我们现在只是在服务器中获取到了request对象 —— 我们并没有像之前response对象那样，把 request 对象传递给请求路由和请求处理程序。

在我看来，获取所有来自请求的数据，然后将这些数据给应用层处理，应该是HTTP服务器要做的事情。因此，我建议，我们直接在服务器中处理POST数据，然后将最终的数据传递给请求路由和请求处理器，让他们来进行进一步的处理。

因此，实现思路就是： 将data和end事件的回调函数直接放在服务器中，在data事件回调中收集所有的POST数据，当接收到所有数据，触发end事件后，其回调函数调用请求路由，并将数据传递给它，然后，请求路由再将该数据传递给请求处理程序。

还等什么，马上来实现。先从server.js开始：

<pre class="file" data-filename="server.js" data-target="replace">
var http = require("http");
var url = require("url");

function start(route, handle) {
  function onRequest(request, response) {
    var postData = "";
    var pathname = url.parse(request.url).pathname;
    console.log("Request for " + pathname + " received.");

    request.setEncoding("utf8");

    request.addListener("data", function(postDataChunk) {
      postData += postDataChunk;
      console.log("Received POST data chunk '"+
      postDataChunk + "'.");
    });

    request.addListener("end", function() {
      route(handle, pathname, response, postData);
    });

  }

  http.createServer(onRequest).listen(8888);
  console.log("Server has started.");
}

exports.start = start;
</pre>

上述代码做了三件事情： 
首先，我们设置了接收数据的编码格式为UTF-8，

然后注册了“data”事件的监听器，用于收集每次接收到的新数据块，并将其赋值给postData 变量，

最后，我们将请求路由的调用移到end事件处理程序中，以确保它只会当所有数据接收完毕后才触发，并且只触发一次。我们同时还把POST数据传递给请求路由，因为这些数据，请求处理程序会用到。

上述代码在每个数据块到达的时候输出了日志，这对于最终生产环境来说，是很不好的（数据量可能会很大，还记得吧？），但是，在开发阶段是很有用的，有助于让我们看到发生了什么。

我建议可以尝试下，尝试着去输入一小段文本，以及大段内容，当大段内容的时候，就会发现data事件会触发多次。

再来点酷的。我们接下来在/upload页面，展示用户输入的内容。要实现该功能，我们需要将postData传递给请求处理程序，修改router.js为如下形式：

<pre class="file" data-filename="router.js" data-target="replace">

function route(handle, pathname, response, postData) {
  console.log("About to route a request for " + pathname);
  if (typeof handle[pathname] === 'function') {
    handle[pathname](response, postData);
  } else {
    console.log("No request handler found for " + pathname);
    response.writeHead(404, {"Content-Type": "text/plain"});
    response.write("404 Not found");
    response.end();
  }
}

exports.route = route;

</pre>

然后，在requestHandlers.js中，我们将数据包含在对upload请求的响应中：

<pre class="file" data-filename="requestHandlers.js" data-target="replace">

function start(response, postData) {
  console.log("Request handler 'start' was called.");

  var body = '&lt;html&gt;'+
    '&lt;head&gt;'+
    '&lt;meta http-equiv="Content-Type" content="text/html; '+
    'charset=UTF-8" /&gt;'+
    '&lt;/head&gt;'+
    '&lt;body&gt;'+
    '&lt;form action="/upload" method="post"&gt;'+
    '&lt;textarea name="text" rows="20" cols="60"&gt;&lt;/textarea&gt;'+
    '&lt;input type="submit" value="Submit text" /&gt;'+
    '&lt;/form&gt;'+
    '&lt;/body&gt;'+
    '&lt;/html&gt;';

    response.writeHead(200, {"Content-Type": "text/html"});
    response.write(body);
    response.end();
}

function upload(response, postData) {
  console.log("Request handler 'upload' was called.");
  response.writeHead(200, {"Content-Type": "text/plain"});
  response.write("You've sent: " + postData);
  response.end();
}

exports.start = start;
exports.upload = upload;

</pre>

好了，我们现在可以接收POST数据并在请求处理程序中处理该数据了。

我们最后要做的是： 当前我们是把请求的整个消息体传递给了请求路由和请求处理程序。我们应该只把POST数据中，我们感兴趣的部分传递给请求路由和请求处理程序。在我们这个例子中，我们感兴趣的其实只是text字段。

我们可以使用此前介绍过的querystring模块来实现：

<pre class="file" data-filename="requestHandlers.js" data-target="replace">

var querystring = require("querystring");

function start(response, postData) {
  console.log("Request handler 'start' was called.");

  var body = '&lt;html&gt;'+
    '&lt;head&gt;'+
    '&lt;meta http-equiv="Content-Type" content="text/html; '+
    'charset=UTF-8" /&gt;'+
    '&lt;/head&gt;'+
    '&lt;body&gt;'+
    '&lt;form action="/upload" method="post"&gt;'+
    '&lt;textarea name="text" rows="20" cols="60"&gt;&lt;/textarea&gt;'+
    '&lt;input type="submit" value="Submit text" /&gt;'+
    '&lt;/form&gt;'+
    '&lt;/body&gt;'+
    '&lt;/html&gt;';

    response.writeHead(200, {"Content-Type": "text/html"});
    response.write(body);
    response.end();
}

function upload(response, postData) {
  console.log("Request handler 'upload' was called.");
  response.writeHead(200, {"Content-Type": "text/plain"});
  response.write("You've sent the text: "+
  querystring.parse(postData).text);
  response.end();
}

exports.start = start;
exports.upload = upload;

</pre>

好了，以上就是关于处理POST数据的全部内容。

## 处理文件上传

最后，我们来实现我们最终的用例：允许用户上传图片，并将该图片在浏览器中显示出来。

回到90年代，这个用例完全可以满足用于IPO的商业模型了，如今，我们通过它能学到这样两件事情： 如何安装外部Node.js模块，以及如何将它们应用到我们的应用中。

这里我们要用到的外部模块是Felix Geisendörfer开发的node-formidable模块。它对解析上传的文件数据做了很好的抽象。 其实说白了，处理文件上传“就是”处理POST数据 —— 但是，麻烦的是在具体的处理细节，所以，这里采用现成的方案更合适点。

使用该模块，首先需要安装该模块。Node.js有它自己的包管理器，叫NPM。它可以让安装Node.js的外部模块变得非常方便。通过如下一条命令就可以完成该模块的安装：

`npm init -y`{{execute interrupt}}

`npm install formidable`{{execute interrupt}}

如果终端输出如下内容：

```
npm info build Success: formidable@1.0.9
npm ok

```

就说明模块已经安装成功了。

现在我们就可以用formidable模块了——使用外部模块与内部模块类似，用require语句将其引入即可：

```
var formidable = require("formidable");

```

这里该模块做的就是将通过HTTP POST请求提交的表单，在Node.js中可以被解析。我们要做的就是创建一个新的IncomingForm，它是对提交表单的抽象表示，之后，就可以用它解析request对象，获取表单中需要的数据字段。

node-formidable官方的例子展示了这两部分是如何融合在一起工作的：

```
var formidable = require('formidable'),
    http = require('http'),
    util = require('util');

http.createServer(function(req, res) {
  if (req.url == '/upload' && req.method.toLowerCase() == 'post') {
    // parse a file upload
    var form = new formidable.IncomingForm();
    form.parse(req, function(err, fields, files) {
      res.writeHead(200, {'content-type': 'text/plain'});
      res.write('received upload:\n\n');
      res.end(util.inspect({fields: fields, files: files}));
    });
    return;
  }

  // show a file upload form
  res.writeHead(200, {'content-type': 'text/html'});
  res.end(
    '&lt;form action="/upload" enctype="multipart/form-data" '+
    'method="post"&gt;'+
    '&lt;input type="text" name="title"&gt;&lt;br&gt;'+
    '&lt;input type="file" name="upload" multiple="multiple"&gt;&lt;br&gt;'+
    '&lt;input type="submit" value="Upload"&gt;'+
    '&lt;/form&gt;'
  );
}).listen(8888);
```

如果我们将上述代码，保存到一个文件中，并通过node来执行，就可以进行简单的表单提交了，包括文件上传。然后，可以看到通过调用form.parse传递给回调函数的files对象的内容，如下所示：

```
received upload:

{ fields: { title: 'Hello World' },
  files:
   { upload:
      { size: 1558,
        path: '/tmp/1c747974a27a6292743669e91f29350b',
        name: 'us-flag.png',
        type: 'image/png',
        lastModifiedDate: Tue, 21 Jun 2011 07:02:41 GMT,
        _writeStream: [Object],
        length: [Getter],
        filename: [Getter],
        mime: [Getter] } } }
```

为了实现我们的功能，我们需要将上述代码应用到我们的应用中，另外，我们还要考虑如何将上传文件的内容（保存在/tmp目录中）显示到浏览器中。

我们先来解决后面那个问题： 对于保存在本地硬盘中的文件，如何才能在浏览器中看到呢？

显然，我们需要将该文件读取到我们的服务器中，使用一个叫fs的模块。

我们来添加/showURL的请求处理程序，该处理程序直接硬编码将文件/tmp/test.png内容展示到浏览器中。当然了，首先需要将该图片保存到这个位置才行。

将requestHandlers.js修改为如下形式：

<pre class="file" data-filename="requestHandlers.js" data-target="replace">

var querystring = require("querystring"),
    fs = require("fs");

function start(response, postData) {
  console.log("Request handler 'start' was called.");

  var body = '&lt;html&gt;'+
    '&lt;head&gt;'+
    '&lt;meta http-equiv="Content-Type" '+
    'content="text/html; charset=UTF-8" /&gt;'+
    '&lt;/head&gt;'+
    '&lt;body&gt;'+
    '&lt;form action="/upload" method="post"&gt;'+
    '&lt;textarea name="text" rows="20" cols="60"&gt;&lt;/textarea&gt;'+
    '&lt;input type="submit" value="Submit text" /&gt;'+
    '&lt;/form&gt;'+
    '&lt;/body&gt;'+
    '&lt;/html&gt;';

    response.writeHead(200, {"Content-Type": "text/html"});
    response.write(body);
    response.end();
}

function upload(response, postData) {
  console.log("Request handler 'upload' was called.");
  response.writeHead(200, {"Content-Type": "text/plain"});
  response.write("You've sent the text: "+
  querystring.parse(postData).text);
  response.end();
}

function show(response, postData) {
  console.log("Request handler 'show' was called.");
  fs.readFile("/tmp/test.png", "binary", function(error, file) {
    if(error) {
      response.writeHead(500, {"Content-Type": "text/plain"});
      response.write(error + "\n");
      response.end();
    } else {
      response.writeHead(200, {"Content-Type": "image/png"});
      response.write(file, "binary");
      response.end();
    }
  });
}

exports.start = start;
exports.upload = upload;
exports.show = show;

</pre>

我们还需要将这新的请求处理程序，添加到index.js中的路由映射表中：

<pre class="file" data-filename="index.js" data-target="replace">

var server = require("./server");
var router = require("./router");
var requestHandlers = require("./requestHandlers");

var handle = {}
handle["/"] = requestHandlers.start;
handle["/start"] = requestHandlers.start;
handle["/upload"] = requestHandlers.upload;
handle["/show"] = requestHandlers.show;

server.start(router.route, handle);

</pre>

重启服务器之后，

`node index.js`{{execute interrupt}}

通过访问

<https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/show>

就可以看到保存在/tmp/test.png的图片了。

好，最后我们要的就是：

* 在/start表单中添加一个文件上传元素
* 将node-formidable整合到我们的upload请求处理程序中，用于将上传的图片保存到/tmp/test.png
* 将上传的图片内嵌到/uploadURL输出的HTML中

第一项很简单。只需要在HTML表单中，添加一个multipart/form-data的编码类型，移除此前的文本区，添加一个文件上传组件，并将提交按钮的文案改为“Upload file”即可。 如下requestHandler.js所示：

<pre class="file" data-filename="requestHandlers.js" data-target="replace">

var querystring = require("querystring"),
    fs = require("fs");

function start(response, postData) {
  console.log("Request handler 'start' was called.");

  var body = '&lt;html&gt;'+
    '&lt;head&gt;'+
    '&lt;meta http-equiv="Content-Type" '+
    'content="text/html; charset=UTF-8" /&gt;'+
    '&lt;/head&gt;'+
    '&lt;body&gt;'+
    '&lt;form action="/upload" enctype="multipart/form-data" '+
    'method="post"&gt;'+
    '&lt;input type="file" name="upload"&gt;'+
    '&lt;input type="submit" value="Upload file" /&gt;'+
    '&lt;/form&gt;'+
    '&lt;/body&gt;'+
    '&lt;/html&gt;';

    response.writeHead(200, {"Content-Type": "text/html"});
    response.write(body);
    response.end();
}

function upload(response, postData) {
  console.log("Request handler 'upload' was called.");
  response.writeHead(200, {"Content-Type": "text/plain"});
  response.write("You've sent the text: "+
  querystring.parse(postData).text);
  response.end();
}

function show(response, postData) {
  console.log("Request handler 'show' was called.");
  fs.readFile("/tmp/test.png", "binary", function(error, file) {
    if(error) {
      response.writeHead(500, {"Content-Type": "text/plain"});
      response.write(error + "\n");
      response.end();
    } else {
      response.writeHead(200, {"Content-Type": "image/png"});
      response.write(file, "binary");
      response.end();
    }
  });
}

exports.start = start;
exports.upload = upload;
exports.show = show;

</pre>

很好。下一步相对比较复杂。这里有这样一个问题： 我们需要在upload处理程序中对上传的文件进行处理，这样的话，我们就需要将request对象传递给node-formidable的form.parse函数。

但是，我们有的只是response对象和postData数组。看样子，我们只能不得不将request对象从服务器开始一路通过请求路由，再传递给请求处理程序。 或许还有更好的方案，但是，不管怎么说，目前这样做可以满足我们的需求。

到这里，我们可以将postData从服务器以及请求处理程序中移除了 —— 一方面，对于我们处理文件上传来说已经不需要了，另外一方面，它甚至可能会引发这样一个问题： 我们已经“消耗”了request对象中的数据，这意味着，对于form.parse来说，当它想要获取数据的时候就什么也获取不到了。（因为Node.js不会对数据做缓存）

我们从server.js开始 —— 移除对postData的处理以及request.setEncoding （这部分node-formidable自身会处理），转而采用将request对象传递给请求路由的方式：

<pre class="file" data-filename="server.js" data-target="replace">
var http = require("http");
var url = require("url");

function start(route, handle) {
  function onRequest(request, response) {
    var pathname = url.parse(request.url).pathname;
    console.log("Request for " + pathname + " received.");
    route(handle, pathname, response, request);
  }

  http.createServer(onRequest).listen(8888);
  console.log("Server has started.");
}

exports.start = start;

</pre>

接下来是 router.js —— 我们不再需要传递postData了，这次要传递request对象：

<pre class="file" data-filename="router.js" data-target="replace">
function route(handle, pathname, response, request) {
  console.log("About to route a request for " + pathname);
  if (typeof handle[pathname] === 'function') {
    handle[pathname](response, request);
  } else {
    console.log("No request handler found for " + pathname);
    response.writeHead(404, {"Content-Type": "text/html"});
    response.write("404 Not found");
    response.end();
  }
}

exports.route = route;
</pre>

现在，request对象就可以在我们的upload请求处理程序中使用了。node-formidable会处理将上传的文件保存到本地/tmp目录中，而我们需要做的是确保该文件保存成/tmp/test.png。 没错，我们保持简单，并假设只允许上传PNG图片。

这里采用fs.renameSync(path1,path2)来实现。要注意的是，正如其名，该方法是同步执行的， 也就是说，如果该重命名的操作很耗时的话会阻塞。 这块我们先不考虑。

接下来，我们把处理文件上传以及重命名的操作放到一起，如下requestHandlers.js所示：

<pre class="file" data-filename="requestHandlers.js" data-target="replace">

var querystring = require("querystring"),
    fs = require("fs"),
    formidable = require("formidable");

function start(response) {
  console.log("Request handler 'start' was called.");

  var body = '&lt;html&gt;'+
    '&lt;head&gt;'+
    '&lt;meta http-equiv="Content-Type" content="text/html; '+
    'charset=UTF-8" /&gt;'+
    '&lt;/head&gt;'+
    '&lt;body&gt;'+
    '&lt;form action="/upload" enctype="multipart/form-data" '+
    'method="post"&gt;'+
    '&lt;input type="file" name="upload" multiple="multiple"&gt;'+
    '&lt;input type="submit" value="Upload file" /&gt;'+
    '&lt;/form&gt;'+
    '&lt;/body&gt;'+
    '&lt;/html&gt;';

    response.writeHead(200, {"Content-Type": "text/html"});
    response.write(body);
    response.end();
}

function upload(response, request) {
  console.log("Request handler 'upload' was called.");

  var form = new formidable.IncomingForm();
  console.log("about to parse");
  form.parse(request, function(error, fields, files) {
    console.log("parsing done");
    fs.renameSync(files.upload.path, "/tmp/test.png");
    response.writeHead(200, {"Content-Type": "text/html"});
    response.write("received image:&lt;br/&gt;");
    response.write("&lt;img src='/show' /&gt;");
    response.end();
  });
}

function show(response) {
  console.log("Request handler 'show' was called.");
  fs.readFile("/tmp/test.png", "binary", function(error, file) {
    if(error) {
      response.writeHead(500, {"Content-Type": "text/plain"});
      response.write(error + "\n");
      response.end();
    } else {
      response.writeHead(200, {"Content-Type": "image/png"});
      response.write(file, "binary");
      response.end();
    }
  });
}

exports.start = start;
exports.upload = upload;
exports.show = show;

</pre>

好了，重启服务器，

`node index`{{execute interrupt}}

我们应用所有的功能就可以用了。选择一张本地图片，将其上传到服务器，然后浏览器就会显示该图片。

<https://[[HOST_SUBDOMAIN]]-8888-[[KATACODA_HOST]].environments.katacoda.com/start>

# 总结与展望

恭喜，我们的任务已经完成了！我们开发完了一个Node.js的web应用，应用虽小，但却“五脏俱全”。 期间，我们介绍了很多技术点：服务端JavaScript、函数式编程、阻塞与非阻塞、回调、事件、内部和外部模块等等。

当然了，还有许多本书没有介绍到的： 如何操作数据库、如何进行单元测试、如何开发Node.js的外部模块以及一些简单的诸如如何获取GET请求之类的方法。

但本教程毕竟只是一本给初学者的教程 —— 不可能覆盖到所有的内容。