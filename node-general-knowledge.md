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
```javascript
// module1.js
var g = 42;
```
And you have module2 that requires module1 and try to access the variable g, you would get g is not defined.

Why? If you do the same in a browser you can access top-level defined variables in all scripts included after their definition.

That's because every Node.js file gets its own IIFE [(Immediately Invoked Function Expression)](https://developer.mozilla.org/en-US/docs/Glossary/IIFE) behind the scenes. All variables declared in a Node.js file are scoped to that IIFE.

[Source](https://medium.com/@samerbuna/you-dont-know-node-6515a658a1ed)

## 3.- When exporting the API of a Node module, why can we sometimes use exports and other times we have to use module.exports?

You can always use `module.exports` to export the API of your modules. You can also use exports except for one case:
```javascript
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
```javascript
var Article = require('../../../models/article');
```
Those suck for maintenance and they're ugly. Ideally, I'd like to have the same basepath from which I require() all my modules. Like any other language environment out there. I'd like the require() calls to be first-and-foremost relative to my application entry point file, in my case `app.js`.

There are only solutions here that work cross-platform, because 42% of Node.js users use Windows as their desktop environment (source).

### 0. The Container
* Learn all about Dependency Injection and Inversion of Control containers. Example implementation using Electrolyte here: [github/branneman/nodejs-app-boilerplate)](http://github/branneman/nodejs-app-boilerplate)

* Create an entry-point file like this:
```javascript
const IoC = require('electrolyte');
IoC.use(IoC.dir('app'));
IoC.use(IoC.node_modules());
IoC.create('server').then(app => app());
```

* You can now define your modules like this:
```javascript
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
```console
Linux: ln -nsf node_modules app
Windows: mklink /D app node_modules
```

* Now you can require local modules like this from anywhere:
```js
var Article = require('models/article');
```
Note: you can not have a symlink like this inside a Git repo, since Git does not handle symlinks cross-platform. If you can live with a post-clone git-hook and/or the instruction for the next developer to create a symlink, then sure.

Alternatively, you can create the symlink on the npm postinstall hook, as described by [scharf](https://github.com/scharf) in this [awesome comment](https://gist.github.com/branneman/8048520#comment-1412502). Put this inside your package.json:
```js
"scripts": {
    "postinstall" : "node -e \"var s='../src',d='node_modules/src',fs=require('fs');fs.exists(d,function(e){e||fs.symlinkSync(s,d,'dir')});\""
  }
```

### 2. The Global
* In your app.js:
```js
global.__base = __dirname + '/';
```

* In your very/far/away/module.js:
```js
var Article = require(__base + 'app/models/article');
```

### 3. The Module
* Install some module:
```console
npm install app-module-path --save
```

* In your app.js, before any require() calls:
```js
require('app-module-path').addPath(__dirname + '/app');
```
* In your very/far/away/module.js:
```js
var Article = require('models/article');
```

### 4. The Environment
Set the `NODE_PATH` environment variable to the absolute path of your application, ending with the directory you want your modules relative to (in my case .).

There are 2 ways of achieving the following require() statement from anywhere in your application:
```js
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
```bash
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
```js
process.env.NODE_PATH = __dirname;
require('module').Module._initPaths();
```

### 7. The Wrapper
Courtesy of [@a-ignatov-parc](https://github.com/a-ignatov-parc). Another simple solution which increases obviousness, simply wrap the require() function with one relative to the path of the application's entry point file.

Place this code in your app.js, again before any require() calls:
```js
global.rootRequire = function(name) {
    return require(__dirname + '/' + name);
}
```
You can then require modules like this:
```js
var Article = rootRequire('app/models/article');
```

Another option is to always use the initial `require()` function, basically the same trick without a wrapper. Node.js creates a new scoped `require()` function for every new module, but there's always a reference to the initial global one. Unlike most other solutions this is actually a documented feature. It can be used like this:
```js
var Article = require.main.require('app/models/article');
```

[Source](https://gist.github.com/branneman/8048520)

## 5.- Can different versions of the same package be used in the same application?
Yes, they can be used. There are some packages that take of that issue but from my point of view is easier to use built in package manager features. For example, in Yarn it can be done by:
```console
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

[Source 1](https://asafdav2.github.io/2017/how-well-do-you-know-node-js-answers-part-1/)
[Source 2](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

## 7.- What is the Call Stack? Is it part of V8?
The call stack is the basic mechanism for JavaScript code execution. When we call a function, we push the function parameters and the return address to the stack. This allows to runtime to know where to continue code execution once the function ends. In Node.js, the Call Stack is handled by V8.

The Call Stack is a data structure which records basically where in the program we are. If we step into a function, we put it on the top of the stack. If we return from a function, we pop off the top of the stack. That’s all the stack can do.

Let’s see an example. Take a look at the following code:
```js
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

* process.nextTick()
The callback of a process.nextTick() is placed at the head of the event queue and is completely processed before I/O or timer callbacks but still after execution of the current execution context. It is used when we need to postpone emitting an event until after the caller has had the chance to register an event listener for this event. Technically, is not part of the event loop even though is part of the asynchronous API.

* setImmediate()
setImmediate is similiar to setInterval/setTimeOut in that it has a cancelImmediate() in the same way as canceInterval/cancelTimeout, but it lacks a time as a second argument.

Actually in a sense it is closer to process.nextTick. The difference with nextTick is that setImmediate's callback is queued afer I/O callbacks, while nextTick's callback is queued to execute before I/O callbacks.

NOTE: the names are counter-intuitive since in reality the setImmediate callback is actually executed after the process.nextTick callback.

Executing the following code:
```js
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
```console
NEXTTICK
IMMEDIATE
TIMEOUT
```

[Source 1](https://stackoverflow.com/questions/15349733/setimmediate-vs-nexttick#15349865)
[Source 2](http://becausejavascript.com/node-js-process-nexttick-vs-setimmediate/)
[Source 3](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

## 9.- How do you make an asynchronous function return a value?

You could return a promise resolving to that value, for example:
```js
return Promise.resolve(true)
```

[Source](https://asafdav2.github.io/2017/how-well-do-you-know-node-js-answers-part-1/)

## 10.- Can callbacks be used with promises or is it one way or the other?
Callbacks are not interchangeable with Promises. This means that callback-based APIs cannot be used as Promises. The main difference with callback-based APIs is it does not return a value, it just executes the callback with the result. A Promise-based API, on the other hand, immediately returns a Promise that wraps the asynchronous operation, and then the caller uses the returned Promise object and calls .then() and .catch() on it to declare what will happen when the operations has finished.

You can use callbacks as promises by _promisifying_ them. Since Node.js v.8 there is the built-in module `util` that allows to convert callback functions into promises.
```js
import request from 'request';
import { promisify } from 'util';

const requestPromise = promisify(request);
```

To create own's _promisify_ method the following code can be used:
```js
const pkg = require('pkg'); 

export class Foo {
   getProperty(value, cb){
     pkg.doSomething(value, cb);
   }
  
   getProperty(value){
     return new Promise((resolve, reject) => {
       this.getProperty(value, (err,val) => err ? reject(err) : resolve(val));
     });
   }
  
}
```
And used like this:
```js
const foo = new Foo();

// callback
foo.getProperty('bar', (err, val) => {
 // logic 
});

// promise 
foo.getProperty('bar').then(val => {
 // logic 
});
```


## 11.- What Node module is implemented by most other Node modules?

// TO-DO


## 12.- What are the major differences between spawn, exec, and fork?

They are all part of the `child_process` module. Also all of them are asynchronous and calling them will return an object which is an instance of the `ChildProcess` class.

* `spawn` is a command used to run system commands, so it will run in its own process but won't execute any further code within the node process. No new V8 instance is created unless you execute a new node command. Only one copy of the node module is active on the processor. The `spawn` method returns a streaming interface for I/O operations (`stdout & stderr`). So it is great for handling applications that produce large amounts of data or for working with data as it reads in.
```js
const spawn = require('child_process').spawn;
const fs = require('fs');
function resize(req, resp) {
    const args = [
        "-", // use stdin
        "-resize", "640x", // resize width to 640
        "-resize", "x360<", // resize height if it's smaller than 360
        "-gravity", "center", // sets the offset to the center
        "-crop", "640x360+0+0", // crop
        "-" // output to stdout
    ];
    const streamIn = fs.createReadStream('./path/to/an/image');
    const proc = spawn('convert', args);
    streamIn.pipe(proc.stdin);
    proc.stdout.pipe(resp);
}
```

* `exec` is a command that will spawn a subshell and execute the command in that shell (mapped to /bin/sh (Linux) / cmd.exe (Win)) and buffer generated data. It allows to execute more than one command in that shell. It should be used when any shell functionality is needed such pipes, redirects, backgrounding, etc.
```js
const exec = require('child_process').exec;
exec('for i in $( ls -LR ); do echo item: $i; done', (e, stdout, stderr)=> {
    if (e instanceof Error) {
        console.error(e);
        throw e;
    }
    console.log('stdout ', stdout);
    console.log('stderr ', stderr);
});
```

* `fork` is a special instance of `spawn` that runs a fresh instance of the V8 engine, it creates child processes. It is mostly used to create a worker pool so you can take advantage of all the cores of the machine. The `fork` method returns an object with a built-in communication channel (IPC) in addition to the methods of a normal ChildProcess instance. It is useful to unload long running tasks such computational ones so Node's main process doesn't get blocked for it.
```js
//parent.js
const cp = require('child_process');
const n = cp.fork(`${__dirname}/sub.js`);
n.on('message', (m) => {
  console.log('PARENT got message:', m);
});
n.send({ hello: 'world' });
//sub.js
process.on('message', (m) => {
  console.log('CHILD got message:', m);
});
process.send({ foo: 'bar' });
```

[Source1](https://dzone.com/articles/understanding-execfile-spawn-exec-and-fork-in-node)
[Source2](https://stackoverflow.com/questions/17861362/node-js-child-process-difference-between-spawn-fork)


## 13.- How does the cluster module work? How is it different than using a load balancer?

Node cluster core module has a master process listening on a port and then distributes the connections to different workers being all run in the same host. A round-robin algorithm is used to distribute the load between workers. This is the default approach since v.6.0.0. In practice however, distribution tends to be very unbalanced due to operating system scheduler vagaries. Loads have been observed where over 70% of all connections ended up in just two processes, out of a total of eight. Also the master process will work almost as much as the worker processes with less request rate than other solutions such load balancing.

A load balancer such the likes of Nginx, in contrast, is used to distribute incoming connections across multiple hosts. 

[Source1](https://medium.com/@fermads/node-js-process-load-balancing-comparing-cluster-iptables-and-nginx-6746aaf38272)


## 14.- What are the --harmony-* flags?

Are flags inside Node.js that allow to enable staged features only, i.e. completed features that have not been considered stable yet.


## 15.- How can you read and inspect the memory usage of a Node.js process?

Using `process.memoryUsage()`. `memoryUsage` returns an object with various information: `rss`, `heapTotal`, `heapUsed` (and external):
```js
// Response
rss 25.63 MB
heapTotal 5.49 MB
heapUsed 3.6 MB
external 0.01 MB
```


## 16.- What will Node do when both the call stack and the event loop queue are empty?

It will simply exit.

When you run a Node program, Node will automatically start the event loop and when that event loop is idle and has nothing else to do, the process will exit.

To keep a Node process running, you need to place something somewhere in event queues. For example, when you start a timer or an HTTP server you are basically telling the event loop to keep running and checking on these events.


## 17.- What are V8 object and function templates?

A template is a blueprint for JavaScript functions and objects in a context. You can use a template to wrap C++ functions and data structures within JavaScript objects so that they can be manipulated by JavaScript scripts. For example, Google Chrome uses templates to wrap C++ DOM nodes as JavaScript objects and to install functions in the global namespace. You can create a set of templates and then use the same ones for every new context you make. You can have as many templates as you require. However you can only have one instance of any template in any given context.

In JavaScript there is a strong duality between functions and objects. To create a new type of object in Java or C++ you would typically define a new class. In JavaScript you create a new function instead, and create instances using the function as a constructor. The layout and functionality of a JavaScript object is closely tied to the function that constructed it. This is reflected in the way V8 templates work. There are two types of templates:

 - Function templates

A function template is the blueprint for a single function. You create a JavaScript instance of the template by calling the template’s `GetFunction` method from within the context in which you wish to instantiate the JavaScript function. You can also associate a C++ callback with a function template which is called when the JavaScript function instance is invoked.

 - Object templates

Each function template has an associated object template. This is used to configure objects created with this function as their constructor. You can associate two types of C++ callbacks with object templates:

 * accessor callbacks are invoked when a specific object property is accessed by a script
 * interceptor callbacks are invoked when any object property is accessed by a script

[Source1](https://medium.com/@hyperandroid/javascript-native-wrappers-in-v8-part-i-67851a3a797a)
[Source2](https://v8.dev/docs/embed)


## 18.- What are circular modular dependencies in Node and how can they be avoided?
 
Let’s now try to answer the important question about circular dependency in Node: What happens when module 1 requires module 2, and module 2 requires module 1?

To find out, let’s create two files module1.js and module2.js under lib/ and have them require each other: 
```js
// lib/module1.js
exports.a = 1;

require("./module2");

exports.b = 2;
exports.c = 3;
```

```js
// lib/module2.js

const Module1 = require("./module1");
console.log("Module1 is partially loaded here", Module1);
```

When we execute module1.js, we see the following:
```bash
$ node lib/module1.js0
Module1 is partially loaded here { a: 1 }
```

We required `module2` before `module1` was fully loaded and since `module2` required `module1` while it wasn’t fully loaded, what we get from the exports object at that point are all the properties exported prior to the circular dependency. Only the a property was reported because both b and c were exported after module2 required and printed module1.

Node keeps this really simple. During the loading of a module, it builds the exports object. You can require the module before it’s done loading and you’ll just get a partial exports object with whatever was defined so far.

To avoid them there are different approaches:
- If module A and module2 are always used together, merge them in same module might be a good idea.
- Using composite pattern to break the dependency.
- In database models, having model A and model B, wanting to reference model B in model A (to join operations), exporting several A and B properties (the ones not depending on other modules) before using the `require` function.
```js
function ModuleA() {
}

module.exports = ModuleA;

var moduleB = require('./b');

ModuleA.hello = function () {
  console.log('hello!');
};
```


## 19.- What is libuv and how does Node.js use it?

Event loop is message dispatcher that waits for and dispatches events or messages in a program. It works by making a request to some internal or external event provider (which generally blocks the request until an event has arrived), and then it calls the relevant event handler (dispatches the event). The event loop may be used in conjunction with a reactor if the event provider follows the file interface which can be selected or polled. The event loop almost always operates asynchronously with the message originator.

V8 can accept event loop as an argument when you are creating V8 Environment. But before setting up an event loop to V8 we need to implement it first…

To accommodate the single-threaded event loop, Node.js uses the [libuv](https://libuv.org/) library. Without libuv Node.js is just a synchronous JavaScript\C++ execution. It is a C library.

The I/O (or event) loop is the central part of libuv. It establishes the content for all I/O operations, and it’s meant to be tied to a single thread. One can run multiple event loops as long as each runs in a different thread.
Which, in turn, is to use a fixed-sized thread pool that handles the execution of some of the non-blocking asynchronous I/O operations in parallel. The main thread call functions post tasks to the shared task queue, which threads in the thread pool pull and execute.

libuv doesn't use thread for asynchronous tasks, but for those that aren't asynchronous by nature. As an example, it doesn't use threads to deal with sockets, it uses threads to make synchronous fs calls asynchronous.

So, basically, we can include libuv sources into NodeJS and create V8 Environment with libuv default event loop in there. Here is an implementation of it.

```cpp
Environment* CreateEnvironment(Isolate* isolate, uv_loop_t* loop, Handle<Context> context, int argc, const char* const* argv, int exec_argc, const char* const* exec_argv) {
  HandleScope handle_scope(isolate);

  Context::Scope context_scope(context);
  Environment* env = Environment::New(context, loop);

  isolate->SetAutorunMicrotasks(false);

  uv_check_init(env->event_loop(), env->immediate_check_handle());
  uv_unref(reinterpret_cast<uv_handle_t*>(env->immediate_check_handle()));
  uv_idle_init(env->event_loop(), env->immediate_idle_handle());
  uv_prepare_init(env->event_loop(), env->idle_prepare_handle());
  uv_check_init(env->event_loop(), env->idle_check_handle());
  uv_unref(reinterpret_cast<uv_handle_t*>(env->idle_prepare_handle()));
  uv_unref(reinterpret_cast<uv_handle_t*>(env->idle_check_handle()));

  // Register handle cleanups
  env->RegisterHandleCleanup(reinterpret_cast<uv_handle_t*>(env->immediate_check_handle()), HandleCleanup, nullptr);
  env->RegisterHandleCleanup(reinterpret_cast<uv_handle_t*>(env->immediate_idle_handle()), HandleCleanup, nullptr);
  env->RegisterHandleCleanup(reinterpret_cast<uv_handle_t*>(env->idle_prepare_handle()), HandleCleanup, nullptr);
  env->RegisterHandleCleanup(reinterpret_cast<uv_handle_t*>(env->idle_check_handle()), HandleCleanup, nullptr);

  if (v8_is_profiling) {
    StartProfilerIdleNotifier(env);
  }

  Local<FunctionTemplate> process_template = FunctionTemplate::New(isolate);
  process_template->SetClassName(FIXED_ONE_BYTE_STRING(isolate, "process"));

  Local<Object> process_object = process_template->GetFunction()->NewInstance();
  env->set_process_object(process_object);

  SetupProcessObject(env, argc, argv, exec_argc, exec_argv);
  LoadAsyncWrapperInfo(env);

  return env;
}
```

[Source1](https://www.freecodecamp.org/news/node-js-what-when-where-why-how-ab8424886e2/)
[Source2](https://stackoverflow.com/questions/46753611/in-node-js-what-is-libuv-and-does-it-use-all-core#46754428)
[Source3](https://blog.ghaiklor.com/how-nodejs-works-bfe09efc80ca)


## 20.- How can you make Node’s REPL always use JavaScript strict mode?

You can run Node.js with the `--use_strict` flag which will open the REPL in strict mode.

```bash
node --use_strict index.js
```


## 21. What is process.argv? What type of data does it hold?

The `process.argv` property returns an array containing the command line arguments passed when the Node.js process was launched. The first element will be `process.execPath`. See `process.argv0` if access to the original value of `argv[0]` is needed. The second element will be the path to the JavaScript file being executed. The remaining elements will be any additional command line arguments.

For example, assuming the following script for process-args.js:
```javascript
// print process.argv
process.argv.forEach((val, index) => {
  console.log(`${index}: ${val}`);
});
```

Launching the Node.js process as:
```bash
$ node process-args.js one two=three four
```

Would generate the output:
```bash
0: /usr/local/bin/node
1: /Users/mjr/work/node/process-args.js
2: one
3: two=three
4: four
```
[Source](https://nodejs.org/api/all.html#process_process_argv)


## 22. How can we do one final operation before a Node process exits? Can that operation be done asynchronously?

By registering a handler for `process.on('exit')`:
```javascript
process.on('exit', (code) => {
  console.log(`About to exit with code: ${code}`);
});
```

Listener functions must only perform synchronous operations. The Node.js process will exit immediately after calling the `exit` event listeners causing any additional work still queued in the event loop to be abandoned. In the following example, for instance, the timeout will never occur:
```javascript
process.on('exit', (code) => {
  setTimeout(() => {
    console.log('This will not run');
  }, 0);
});
```

The `beforeExit` event is emitted when Node.js empties its event loop and has no additional work to schedule. Normally, the Node.js process will exit when there is no work scheduled, but a listener registered on the `beforeExit` event can make asynchronous calls, and thereby cause the Node.js process to continue.

[Source1](https://nodejs.org/api/process.html#process_event_exit)
[Source2](https://nodejs.org/api/process.html#process_event_beforeexit)


## 23. Besides V8 and libuv, what other external dependencies does Node have?

The following are all separate libraries that a Node process can use:

- [http-parser](https://github.com/joyent/http-parser/): a lightweight C library which handles HTTP parsing
- [c-areas](http://c-ares.haxx.se/docs.html): used for some asynchronous DNS requests
- [OpenSSL](https://www.openssl.org/docs/): used extensively in both the `tls` and `crypto` modules
- [zlib](http://www.zlib.net/manual.html): used for fast compression and decompression

All of them are external to Node. They have their own source code. They also have their own license. Node just uses them.

[Source](https://medium.com/edge-coders/you-dont-know-node-6515a658a1ed)


## 24. What are some of the built-in dot commands that you can use in Node’s REPL?

The following dot commands can be used:

`.break` - When in the process of inputting a multi-line expression, entering the `.break` command (or pressing the `<ctrl>-C` key combination) will abort further input or processing of that expression.
`.clear` - Resets the REPL ‘context’ to an empty object and clears any multi-line expression currently being input.
`.exit` - Close the I/O stream, causing the REPL to exit.
`.help` - Show this list of special commands.
`.save` - Save the current REPL session to a file: > `.save ./file/to/save.js`
`.load` - Load a file into the current REPL session. > `.load ./file/to/load.js`
`.editor` - Enter editor mode (`<ctrl>-D` to finish, `<ctrl>-C` to cancel)

[Source](https://asafdav2.github.io/2017/how-well-do-you-know-node-js-answers-part-1/)


## 25. What’s the problem with the process uncaughtException event? How is it different than the exit event?

`uncaughtException` is a crude mechanism for exception handling intended to be used only as a last resort. The event should not be used as an equivalent to `On Error Resume Next`. Unhandled exceptions inherently mean that an application is in an undefined state. Attempting to resume application code without properly recovering from the exception can cause additional unforeseen and unpredictable issues.

Exceptions thrown from within the event handler will not be caught. Instead the process will exit with a non-zero exit code and the stack trace will be printed. This is to avoid infinite recursion.

Attempting to resume normally after an uncaught exception can be similar to pulling out the power cord when upgrading a computer. Nine out of ten times, nothing happens. But the tenth time, the system becomes corrupted.

The correct use of `uncaughtException` is to perform synchronous cleanup of allocated resources (e.g. file descriptors, handles, etc) before shutting down the process. It is not safe to resume normal operation after `uncaughtException`.

To restart a crashed application in a more reliable way, whether `uncaughtException` is emitted or not, an external monitor should be employed in a separate process to detect application failures and recover or restart as needed.

The exit event is emitted when the Node.js process is about the exit as a result of either:
- The `process.exit()` method being called explicitly
- The event loop has no additional work to perform

[Source](https://nodejs.org/api/process.html#process_warning_using_uncaughtexception_correctly)

## 26. What does the _ mean inside of Node’s REPL?

Node’s REPL always sets _ to the result of the last expression.


Do Node buffers use V8 memory? Can they be resized?
What’s the difference between Buffer.alloc and Buffer.allocUnsafe?
How is the slice method on buffers different from that on arrays?
What is the string_decoder module useful for? How is it different than casting buffers to strings?
What are the 5 major steps that the require function does?
How can you check for the existence of a local module?
What is the main property in package.json useful for?
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
