<!--
 * @Date: 2020-12-24 20:06:03
 * @LastEditors: NeoJoke
 * @LastEditTime: 2020-12-24 20:07:01
 * @FilePath: /katacoda-scenarios/learn-node/intro.md
-->
# 简单的web应用-基于事件驱动与异步非阻塞IO
## *服务端JavaScript*
JavaScript最早是运行在浏览器中，然而浏览器只是提供了一个运行上下文，它定义了使用JavaScript可以做什么，但并没有“说”太多关于JavaScript语言本身可以做什么。事实上，JavaScript是一门“完整”的语言： 它可以使用在不同的上下文中，其能力与其他同类语言相比有过之而无不及。
Node.js事实上就是另外一种上下文，它允许在后端（脱离浏览器环境）运行JavaScript代码。
要实现在后台运行JavaScript代码，代码需要先被解释然后正确的执行。Node.js的原理正是如此，它使用了Google的V8虚拟机（Google的Chrome浏览器使用的JavaScript执行环境），来解释和执行JavaScript代码。
除此之外，Node.js还有许多有用的模块，它们可以简化很多重复的劳作，比如向终端输出字符串。
因此，Node.js事实上既是一个运行时环境，同时又是一个库。