[Primary source where these questions were raised](https://medium.freecodecamp.org/before-you-bury-yourself-in-packages-learn-the-node-js-runtime-itself-f9031fbd8b69)

## 1.- What is the relationship between Node.js and V8? Can Node work without V8?

V8 is the Javascript engine inside of Node.js that parses and runs your JavaScript. The same V8 engine is used inside of Chrome to run JavaScript in the Chrome browser. Google open-sourced the V8 engine and the builders of Node.js used it to run JavaScript in Node.js.

The current Node.js binary cannot work without V8. It would have no JavaScript engine and thus no ability to run code which would obviously render it non-functional. Node.js was not designed to run with any other JavaScript engine and, in fact, all the native code bindings that come with Node.js (such as the `fs` module or the `net` module) all rely on the specific V8 interface between C++ and JavaScript.

There is an effort by Microsoft to allow the Chakra JavaScript engine (that's the engine in Edge) to be used with Node.js. They build a V8 shim on top of Chakra so that the Node.js binary code that expects to be talking to V8 can continue to do what it was doing, but actually end up talking to the Chakra engine underneath. From what I've read this is particularly targeted at Microsoft platforms that already have the Chakra engine and do not have the V8 engine running on them, though presumably you could use it on Windows too.

Also there are different new JavaScript engines including Rhino, JavaScriptCore, and SpiderMonkey. These engines follow the ECMAScript Standards. ECMAScript defines the standard for the scripting language. JavaScript is based on ECMAScript standards. These standards define how the language should work and what features it should have.

[Source](https://stackoverflow.com/questions/42616120/what-is-the-relationship-between-node-js-and-v8)
[Source](https://medium.freecodecamp.org/understanding-the-core-of-nodejs-the-powerful-chrome-v8-engine-79e7eb8af964)

## 2.- How come when you declare a global variable in any Node.js file it’s not really global to all modules?

If you have module1 that defines a top-level variable g:
```
// module1.js
var g = 42;
```
And you have module2 that requires module1 and try to access the variable g, you would get g is not defined.

Why? If you do the same in a browser you can access top-level defined variables in all scripts included after their definition.

That's because every Node.js file gets its own IIFE [(Immediately Invoked Function Expression)](https://developer.mozilla.org/en-US/docs/Glossary/IIFE) behind the scenes. All variables declared in a Node.js file are scoped to that IIFE.

[Source](https://medium.com/@samerbuna/you-dont-know-node-6515a658a1ed)

## 3.- When exporting the API of a Node module, why can we sometimes use exports and other times we have to use module.exports?

You can always use `module.exports` to export the API of your modules. You can also use exports except for one case:
```
module.exports.g = ...  // Ok
exports.g = ...         // Ok
module.exports = ...    // Ok
exports = ...           // Not Ok
```
Why?

`exports` is just a reference or alias to `module.exports`. When you change `exports` you are changing that reference and no longer using the official API (which is `module.exports`). You would just get a local variable in the module scope.

[Source](https://medium.com/@samerbuna/you-dont-know-node-6515a658a1ed)

## 4.- Can we require local files without using relative paths?

When the directory structure of your Node.js application (not library!) has some depth, you end up with a lot of annoying relative paths in your require calls like:
```
var Article = require('../../../models/article');
```
Those suck for maintenance and they're ugly. Ideally, I'd like to have the same basepath from which I require() all my modules. Like any other language environment out there. I'd like the require() calls to be first-and-foremost relative to my application entry point file, in my case `app.js`.

There are only solutions here that work cross-platform, because 42% of Node.js users use Windows as their desktop environment (source).

### 0. The Container
* Learn all about Dependency Injection and Inversion of Control containers. Example implementation using Electrolyte here: [github/branneman/nodejs-app-boilerplate)](http://github/branneman/nodejs-app-boilerplate)

* Create an entry-point file like this:
```
const IoC = require('electrolyte');
IoC.use(IoC.dir('app'));
IoC.use(IoC.node_modules());
IoC.create('server').then(app => app());
```

* You can now define your modules like this:
```
module.exports = factory;
module.exports['@require'] = [
    'lib/read',
    'lib/http-helpers',
    'lib/render-view'
];
function factory(read, { sendHTML, push }, render) { /* ... */ }
```
More detailed example module: [app/areas/homepage/index.js](https://github.com/branneman/nodejs-app-boilerplate/blob/master/app/areas/homepage/index.js)

* Profit

### 1. The Symlink
Stolen from: [focusaurus](https://github.com/focusaurus) / [express_code_structure](https://github.com/focusaurus/express_code_structure) # [the-app-symlink-trick](https://github.com/focusaurus/express_code_structure#the-app-symlink-trick)

* Create a symlink under `node_modules` to your app directory:
```
Linux: ln -nsf node_modules app
Windows: mklink /D app node_modules
```

* Now you can require local modules like this from anywhere:
```
var Article = require('models/article');
```
Note: you can not have a symlink like this inside a Git repo, since Git does not handle symlinks cross-platform. If you can live with a post-clone git-hook and/or the instruction for the next developer to create a symlink, then sure.

Alternatively, you can create the symlink on the npm postinstall hook, as described by [scharf](https://github.com/scharf) in this [awesome comment](https://gist.github.com/branneman/8048520#comment-1412502). Put this inside your package.json:
```
"scripts": {
    "postinstall" : "node -e \"var s='../src',d='node_modules/src',fs=require('fs');fs.exists(d,function(e){e||fs.symlinkSync(s,d,'dir')});\""
  }
```

### 2. The Global
* In your app.js:
```
global.__base = __dirname + '/';
```

* In your very/far/away/module.js:
```
var Article = require(__base + 'app/models/article');
```

### 3. The Module
* Install some module:
```
npm install app-module-path --save
```

* In your app.js, before any require() calls:
```
require('app-module-path').addPath(__dirname + '/app');
```
* In your very/far/away/module.js:
```
var Article = require('models/article');
```

### 4. The Environment
Set the `NODE_PATH` environment variable to the absolute path of your application, ending with the directory you want your modules relative to (in my case .).

There are 2 ways of achieving the following require() statement from anywhere in your application:
```
var Article = require('app/models/article');
```


#### 4.1. Up-front
Before running your node app, first run:

Linux: `export NODE_PATH=.`
Windows: `set NODE_PATH=.`

Setting a variable like this with export or set will remain in your environment as long as your current shell is open. To have it globally available in any shell, set it in your userprofile and reload your environment.

#### 4.2. Only while executing node
This solution will not affect your environment other than what node preceives. It does change your application start command.

Start your application like this from now on:
Linux: `NODE_PATH=. node app`
Windows: `cmd.exe /C "set NODE_PATH=.&& node app"`

(On Windows this command will not work if you put a space in between the path and the &&. Crazy shit.)

### 5. The Start-up Script
Effectively, this solution also uses the environment (as in 4.2), it just abstracts it away.

With one of these solutions (5.1 & 5.2) you can start your application like this from now on:
Linux: `./app (also for Windows PowerShell)`
Windows: `app`

An advantage of this solution is that if you want to force your node app to always be started with v8 parameters like `--harmony` or `--use_strict`, you can easily add them in the start-up script as well.

#### 5.1. Node.js
Example implementation:
[https://gist.github.com/branneman/8775568](https://gist.github.com/branneman/8775568)

#### 5.2. OS-specific start-up scripts
Linux, create app.sh in your project root:
```
#!/bin/sh
NODE_PATH=. node app.js
```

Windows, create app.bat in your project root:
```
@echo off
cmd.exe /C "set NODE_PATH=.&& node app.js"
```

### 6. The Hack
Courtesy of [@joelabair](https://github.com/joelabair). Effectively also the same as 4.2, but without the need to specify the `NODE_PATH` outside your application, making it more fool proof. However, since this relies on a private Node.js core method, this is also a hack that might stop working on the previous or next version of node.

This code needs to be placed in your app.js, before any require() calls:
```
process.env.NODE_PATH = __dirname;
require('module').Module._initPaths();
```

### 7. The Wrapper
Courtesy of [@a-ignatov-parc](https://github.com/a-ignatov-parc). Another simple solution which increases obviousness, simply wrap the require() function with one relative to the path of the application's entry point file.

Place this code in your app.js, again before any require() calls:
```
global.rootRequire = function(name) {
    return require(__dirname + '/' + name);
}
```
You can then require modules like this:
```
var Article = rootRequire('app/models/article');
```

Another option is to always use the initial `require()` function, basically the same trick without a wrapper. Node.js creates a new scoped `require()` function for every new module, but there's always a reference to the initial global one. Unlike most other solutions this is actually a documented feature. It can be used like this:
```
var Article = require.main.require('app/models/article');
```

[Source](https://gist.github.com/branneman/8048520)

## 5.- Can different versions of the same package be used in the same application?
Yes, they can be used. There are some packages that take of that issue but from my point of view is easier to use built in package manager features. For example, in Yarn it can be done by:
```
yarn add <alias-package>@npm:<package>
```

This will install a package under a custom alias. Aliasing, allows multiple versions of the same dependency to be installed, each referenced via the alias-package name given. For example, `yarn add my-foo@npm:foo` will install the package `foo` (at the latest version) in your dependencies under the specified alias `my-foo`. Also, `yarn add my-foo@npm:foo@1.0.1` allows a specific version of foo to be installed.

[Source](https://yarnpkg.com/lang/en/docs/cli/add/#toc-yarn-add-alias)

## 6.- What is the Event Loop? Is it part of V8?

The event loop is what allows Node.js to perform non-blocking I/O operations — despite the fact that JavaScript is single-threaded — by offloading operations to the system kernel whenever possible.

Since most modern kernels are multi-threaded, they can handle multiple operations executing in the background. When one of these operations completes, the kernel tells Node.js so that the appropriate callback may be added to the poll queue to eventually be executed.
In Node.js, the responsibility of gathering events from the operating system or monitoring other sources of events is handled by [libuv](https://github.com/libuv/libuv), and the user can register callbacks to be invoked when an event occurs. The event-loop usually keeps running forever. So it is not responsibility of V8 engine.

Event Loop phases:
  * timers: this phase executes callbacks scheduled by setTimeout() and setInterval().
  * pending callbacks: executes I/O callbacks deferred to the next loop iteration.
  * idle, prepare: only used internally.
  * poll: retrieve new I/O events; execute I/O related callbacks (almost all with the exception of close callbacks, the ones scheduled by timers, and setImmediate()); node will block here when appropriate.
  * check: setImmediate() callbacks are invoked here.
  * close callbacks: some close callbacks, e.g. socket.on('close', ...).

Between each run of the event loop, Node.js checks if it is waiting for any asynchronous I/O or timers and shuts down cleanly if there are not any.

[Source](https://asafdav2.github.io/2017/how-well-do-you-know-node-js-answers-part-1/)
[Source](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

## 7.- What is the Call Stack? Is it part of V8?
The call stack is the basic mechanism for JavaScript code execution. When we call a function, we push the function parameters and the return address to the stack. This allows to runtime to know where to continue code execution once the function ends. In Node.js, the Call Stack is handled by V8.

The Call Stack is a data structure which records basically where in the program we are. If we step into a function, we put it on the top of the stack. If we return from a function, we pop off the top of the stack. That’s all the stack can do.

Let’s see an example. Take a look at the following code:
```
function multiply(x, y) {
    return x * y;
}

function printSquare(x) {
    var s = multiply(x, x);
    console.log(s);
}

printSquare(5);
```

When the engine starts executing this code, the Call Stack will be empty. Afterwards, the steps will be the following:
![Call Stack Diagram](https://cdn-images-1.medium.com/max/1600/1*Yp1KOt_UJ47HChmS9y7KXw.png "Call Stack Diagram")

Each entry in the Call Stack is called a Stack Frame.


[Source 1](https://asafdav2.github.io/2017/how-well-do-you-know-node-js-answers-part-1/)
[Source 2](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf)

## 8.- What is the difference between setImmediate and process.nextTick?
Use setImmediate if you want to queue the function behind whatever I/O event callbacks that are already in the event queue. Use process.nextTick to effectively queue the function at the head of the event queue so that it executes immediately after the current function completes.

So in a case where you're trying to break up a long running, CPU-bound job using recursion, you would now want to use setImmediate rather than process.nextTick to queue the next iteration as otherwise any I/O event callbacks wouldn't get the chance to run between iterations.
[Source](https://stackoverflow.com/questions/15349733/setimmediate-vs-nexttick#15349865)

* process.nextTick()
The callback of a process.nextTick() is placed at the head of the event queue and is completely processed before I/O or timer callbacks but still after execution of the current execution context. It is used when we need to postpone emitting an event until after the caller has had the chance to register an event listener for this event. Technically, is not part of the event loop even though is part of the asynchronous API.

* setImmediate()
setImmediate is similiar to setInterval/setTimeOut in that it has a cancelImmediate() in the same way as canceInterval/cancelTimeout, but it lacks a time as a second argument.

Actually in a sense it is closer to process.nextTick. The difference with nextTick is that setImmediate's callback is queued afer I/O callbacks, while nextTick's callback is queued to execute before I/O callbacks.

NOTE: the names are counter-intuitive since in reality the setImmediate callback is actually executed after the process.nextTick callback.

Executing the following code:
```
setTimeout(function() {
    console.log("TIMEOUT");
}, 0);

setImmediate(function() {
    console.log("IMMEDIATE");
});

process.nextTick(function() {
    console.log("NEXTTICK");
});
```
the output will be:
```
NEXTTICK
IMMEDIATE
TIMEOUT
```

[Source](http://becausejavascript.com/node-js-process-nexttick-vs-setimmediate/)
[Source](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

## 9.- How do you make an asynchronous function return a value?

You could return a promise resolving to that value, for example return Promise.resolve(true).

[Source](https://asafdav2.github.io/2017/how-well-do-you-know-node-js-answers-part-1/)

## 10.- Can callbacks be used with promises or is it one way or the other?


What Node module is implemented by most other Node modules?
What are the major differences between spawn, exec, and fork?
How does the cluster module work? How is it different than using a load balancer?
What are the --harmony-* flags?
How can you read and inspect the memory usage of a Node.js process?
What will Node do when both the call stack and the event loop queue are empty?
What are V8 object and function templates?
What is libuv and how does Node.js use it?
How can you make Node’s REPL always use JavaScript strict mode?
What is process.argv? What type of data does it hold?
How can we do one final operation before a Node process exits? Can that operation be done asynchronously?
What are some of the built-in dot commands that you can use in Node’s REPL?
Besides V8 and libuv, what other external dependencies does Node have?
What’s the problem with the process uncaughtException event? How is it different than the exit event?
What does the _ mean inside of Node’s REPL?
Do Node buffers use V8 memory? Can they be resized?
What’s the difference between Buffer.alloc and Buffer.allocUnsafe?
How is the slice method on buffers different from that on arrays?
What is the string_decoder module useful for? How is it different than casting buffers to strings?
What are the 5 major steps that the require function does?
How can you check for the existence of a local module?
What is the main property in package.json useful for?
What are circular modular dependencies in Node and how can they be avoided?
What are the 3 file extensions that will be automatically tried by the require function?
When creating an http server and writing a response for a request, why is the end() function required?
When is it ok to use the file system *Sync methods?
How can you print only one level of a deeply nested object?
What is the node-gyp package used for?
The objects exports, require, and module are all globally available in every module but they are different in every module. How?
If you execute a node script file that has the single line: console.log(arguments);, what exactly will node print?
How can a module be both requirable by other modules and executable directly using the node command?
What’s an example of a built-in stream in Node that is both readable and writable?
What happens when the line cluster.fork() gets executed in a Node script?
What’s the difference between using event emitters and using simple callback functions to allow for asynchronous handling of code?
What is the console.time function useful for?
What’s the difference between the Paused and the Flowing modes of readable streams?
What does the --inspect argument do for the node command?
How can you read data from a connected socket?
The require function always caches the module it requires. What can you do if you need to execute the code in a required module many times?
When working with streams, when do you use the pipe function and when do you use events? Can those two methods be combined?
