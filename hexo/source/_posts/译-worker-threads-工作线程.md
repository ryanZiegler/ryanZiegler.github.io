---
title: '[译]worker_threads (工作线程)'
date: 2019-09-19 19:37:28
tags:
    - Node
    - 子线程
    - worker_threads
    - 工作线程
---
** 本文翻译自 [Node中文文档](http://nodejs.cn/api/worker_threads.html) **

**稳定性: 1 - 试验**(此特性不受语义版本控制规则的约束，在将来的任何版本中都会受到非向后兼容的更改或删除。 建议不要在生产环境中使用该特性。)

worker_threads 模块允许使用并行执行 JavaScript 的线程。首先引入模块:
`const worker = require('worker_threads');`
工作线程（线程）对于执行 CPU 密集型 JavaScript 操作非常有用。他们对于 I / O 密集型工作没有多大帮助，Node.js 的内置异步 I / O 操作比 Workers 更高效。
不同于`child_process`或者`cluster`， `worker_threads`能够共享内存。 他们能够传递`ArrayBuffer`实例或者共享`sharedArrayBuffer`实例。

<!-- more -->

```javascript
const {
  Worker, isMainThread, parentPort, workerData
} = require('worker_threads');

if (isMainThread) {
  module.exports = function parseJSAsync(script) {
    return new Promise((resolve, reject) => {
      const worker = new Worker(__filename, {
        workerData: script
      });
      worker.on('message', resolve);
      worker.on('error', reject);
      worker.on('exit', (code) => {
        if (code !== 0)
          reject(new Error(`Worker stopped with exit code ${code}`));
      });
    });
  };
} else {
  const { parse } = require('some-js-parsing-library');
  const script = workerData;
  parentPort.postMessage(parse(script));
}
```

上面的例子为每个`parse()`调用生成一个 Worker 线程.。在实际操作中， 通常使用线程池代替这些类型的任务。否则，创建这些任务的开销可能会大于创建线程的收益。
实现线程池时,，使用`AsyncResource` API 通知诊断工具(例如，为了提供异步堆栈跟踪)，以关联任务和结果。



#### worker.isMainThread

新增于: v10.5.0

- [boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type)

如果代码未在 Worker 线程内运行，则为true。

```javascript
const { Worker, isMainThread } = require('worker_threads');

if (isMainThread) {
  // This re-loads the current file inside a Worker instance.
  // 这会在Worker实例中重新加载当前文件。
  new Worker(__filename);
} else {
  console.log('Inside Worker!');
  console.log(isMainThread);  // Prints 'false'.
}
```



#### worker.moveMessagePortToContext(port, contextifiedSandbox)

新增于: v11.13.0

- `port` [MessagePort](http://nodejs.cn/api/worker_threads.html#worker_threads_class_messageport) 消息传输的端口
- `contextifiedSandbox`  [Object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)  `vm.createContext()`方法返回的一个[上下文](http://nodejs.cn/api/vm.html)对象。
- 返回: [MessagePort](http://nodejs.cn/api/worker_threads.html#worker_threads_class_messageport)

将`MessagePort`传输到不同的vm上下文。原始`port`对象将变为不可用，并且返回的`MessagePort`实例将取代它。

返回的`MessagePort`是目标上下文中的对象，并继承于全局`Object`类。同时`port.onmessage()`监听器对象也将在目标上下文中创建，并继承于全局`Object`类。

但是，创建的`MessagePort`将不再继承[EventEmitter](http://nodejs.cn/api/events.html)，并且只能使用`port.onmessage()`来接收使用它的事件。



#### worker.parentPort

新增于: v10.5.0

- [null](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Null_type) | [MessagePort](http://nodejs.cn/api/worker_threads.html#worker_threads_class_messageport)

如果该线程通过[`Worker`](http://nodejs.cn/api/worker_threads.html#worker_threads_class_worker)生成的，它将会是个`MessagePort`允许与父线程进行通信。使用`parentPort.postMessage()`发送的消息将在使用`worker.on('message')`的父线程中可用，使用`worker.postMessage()`从父线程发送的消息将在此线程中使用`parentPort.on('message')`接收。

```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
  const worker = new Worker(__filename);
  worker.once('message', (message) => {
    console.log(message);  // Prints 'Hello, world!'.
  });
  worker.postMessage('Hello, world!');
} else {
  // When a message from the parent thread is received, send it back:
  // 收到来自父线程的消息后，将其发回：
  parentPort.once('message', (message) => {
    parentPort.postMessage(message);
  });
}
```



#### worker.receiveMessageOnPort(port)

新增于:  v12.3.0

- `port` [MessagePort](http://nodejs.cn/api/worker_threads.html#worker_threads_class_messageport)
- Returns: [Objects](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object) | [undefined](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Undefined_type)

从给定的`MessagePort`接收单个消息。如果没有可用消息，则返回`undefined`，否则返回具有单个`message` 属性的对象，该消息属性包含消息有效内容，对应于`MessagePort`队列中最旧的消息。

```JavaScript
const { MessageChannel, receiveMessageOnPort } = require('worker_threads');
const { port1, port2 } = new MessageChannel();
port1.postMessage({ hello: 'world' });

console.log(receiveMessageOnPort(port2));
// Prints: { message: { hello: 'world' } }
console.log(receiveMessageOnPort(port2));
// Prints: undefined
```

使用此函数时，不会发出`message`事件，也不会调用`onmessage`监听器。



#### worker.SHARE_ENV

新增于: v11.14.0

- [symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Symbol_type)

一个特殊值，可以作为Worker构造函数的`env`选项传递，以指示当前线程和Worker线程应该共享对同一组环境变量的读写访问权限。

```JavaScript
const { Worker, SHARE_ENV } = require('worker_threads');
new Worker('process.env.SET_IN_WORKER = "foo"', { eval: true, env: SHARE_ENV })
  .on('exit', () => {
    console.log(process.env.SET_IN_WORKER);  // Prints 'foo'.
  });
```



#### worker.threadId

新增于: v10.5.0

- [integer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Number_type)

当前线程的整数标识符。在相应的工作对象（如果有）上，它可以作为`worker.threadId`使用。对于单个进程内的每个Worker实例，此值是唯一的。



#### worker.workerData

新增于: v10.5.0

一个任意JavaScript值，包含传递给此线程的Worker构造函数的数据的克隆。根据HTML结构化克隆算法，克隆数据就好像使用postMessage（）一样。

```javascript
const { Worker, isMainThread, workerData } = require('worker_threads');

if (isMainThread) {
  const worker = new Worker(__filename, { workerData: 'Hello, world!' });
} else {
  console.log(workerData);  // Prints 'Hello, world!'.
}
```



### MessageChannel 类

新增于: v10.5.0

`worker.MessageChannel`类的实例表示异步双向通信通道。`MessageChannel`没有自己的方法。新的`MessageChannel（）`生成一个具有port1和port2属性的对象，这些属性引用链接的MessagePort实例。

```javascript
const { MessageChannel } = require('worker_threads');

const { port1, port2 } = new MessageChannel();
port1.on('message', (message) => console.log('received', message));
port2.postMessage({ foo: 'bar' });
// Prints: received { foo: 'bar' } from the `port1.on('message')` listener
```



### MessagePort 类

新增于: v10.5.0

- 继承于: [EventEmitter](http://nodejs.cn/api/events.html)

`worker.MessagePort`类的实例表示异步双向通信通道的一端。它可用于在不同的Worker之间传输结构化数据，内存区域和其他`MessagePort`。

除了`MessagePorts`是EventEmitters而不是EventTargets之外，此实现与浏览器MessagePorts匹配。



#### 'close' 事件

新增于: v10.5.0

一旦通道的任一侧断开，就会发出`'close'`事件。

```javascript
const { MessageChannel } = require('worker_threads');
const { port1, port2 } = new MessageChannel();

// Prints:
//   foobar
//   closed!
port2.on('message', (message) => console.log(message));
port2.on('close', () => console.log('closed!'));

port1.postMessage('foobar');
port1.close();
```



#### 'message' 事件

新增于: v10.5.0

-  `value`[any](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Data_types) 传播属性 

为任何传入消息发出`'message'`事件，其中包含`port.postMessage（）`的克隆输入。

此事件的监听器将收到传递给`postMessage（）`的值参数的克隆，并且没有其他参数。



#### port.close()

新增于: v10.5.0

禁用在连接两侧进一步发送消息。当此`MessagePort`上不再进行任何通信时，可以调用此方法。

'close'事件将在作为通道一部分的两个`MessagePort`实例上发出。



#### port.postMessage(value[, transferList])

新增于: v10.5.0

- `value` [any](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Data_types)
- `transferList` [Object()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)

将JavaScript值发送到此频道的接收方。`value`将以与HTML结构化克隆算法兼容的方式传输。特别是，与`JSON`的显着差异是：

- `value`可能包含循环引用。
- `value`可能包含内置JS类型的实例，例如`RegExp`s，`BigInt`s，`Map`s，`Set`s等。
- `value`可以包含使用`ArrayBuffers`和`SharedArrayBuffers`的类型化数组。
- `value`可能包含`WebAssembly.Module`实例。
- `value`可能不包含除MessagePorts之外的本机（C ++支持）对象。

```javascript
const { MessageChannel } = require('worker_threads');
const { port1, port2 } = new MessageChannel();

port1.on('message', (message) => console.log(message));

const circularData = {};
circularData.foo = circularData;
// Prints: { foo: [Circular] }
port2.postMessage(circularData);
```

`transferList`可以是`ArrayBuffer`和`MessagePort`对象的列表。转移后，它们将不再可用于频道的发送方（即使它们未包含在值中）。与子进程不同，目前不支持传输诸如网络套接字之类的句柄。

如果`value`包含`SharedArrayBuffer`实例，则可以从任一线程访问这些实例。它们不能列在`transferList`中。

`value`可能仍包含不在`transferList`中的`ArrayBuffer`实例;在这种情况下，底层内存被复制而不是移动。

```javascript
const { MessageChannel } = require('worker_threads');
const { port1, port2 } = new MessageChannel();

port1.on('message', (message) => console.log(message));

const uint8Array = new Uint8Array([ 1, 2, 3, 4 ]);
// This posts a copy of `uint8Array`:
// 发送了一份uint8Array的复制体:
port2.postMessage(uint8Array);
// This does not copy data, but renders `uint8Array` unusable:
// 这不会复制数据，但会使`uint8Array`无法使用：
port2.postMessage(uint8Array, [ uint8Array.buffer ]);

// The memory for the `sharedUint8Array` will be accessible from both the
// original and the copy received by `.on('message')`:
// `sharedUint8Array`的内存可以从两个地方原文和`.on（'message'）`收到的副本访问：
const sharedUint8Array = new Uint8Array(new SharedArrayBuffer(4));
port2.postMessage(sharedUint8Array);

// This transfers a freshly created message port to the receiver.
// This can be used, for example, to create communication channels between
// multiple `Worker` threads that are children of the same parent thread.
// 这会将新创建的消息端口传输到接收方。
// 例如，这可以用于在它们之间创建多个`Worker`线程通信信道，它们是同一父线程的子节点。
const otherChannel = new MessageChannel();
port2.postMessage({ port: otherChannel.port1 }, [ otherChannel.port1 ]);
```

由于对象克隆使用结构化克隆算法，因此不会保留不可枚举的属性，属性访问器和对象原型。特别是，Buffer对象将在接收端作为普通的Uint8Arrays读取。

将立即克隆消息对象，并且可以在发布后修改消息对象而不会产生副作用。

有关此API背后的序列化和反序列化机制的更多信息，请参阅v8模块的序列化API。



#### port.ref()

新增于: v10.5.0

与`unref（）`相反。如果它是唯一的活动句柄（默认行为），则在先前的已`unref（）`端口上调用`ref（）`将不会让程序退出。如果端口是已`ref（）`，再次调用`ref（）`将无效。

如果使用`.on（'message'）`附加或删除侦听器，则端口将自动为已`ref（）`和已`unref（）`，具体取决于事件的侦听器是否存在。



#### port.start()

新增于: v10.5.0

开始在此`MessagePort`上接收消息。将此端口用作事件发射器时，一旦附加了`'message'`侦听器，就会自动调用此端口。

此方法用于与Web `MessagePort` API进行奇偶校验。在Node.js中，它仅在没有事件侦听器时忽略消息。Node.js在处理`.onmessage`方面也存在分歧。设置它将自动调用`.start（）`，但是取消设置它会让消息排队，直到设置新的处理程序或丢弃端口。



#### port.unref()

新增于: v10.5.0

如果这是事件系统中唯一的活动句柄，则在端口上调用unref（）将允许线程退出。如果端口已经是unref（），则再次调用unref（）将无效。

如果使用`.on（'message'）`附加或删除侦听器，则端口将自动为已`ref（）`和已`unref（）`，具体取决于事件的侦听器是否存在。



### Worker 类

新增于: v10.5.0

- 继承于: [EventEmitter](http://nodejs.cn/api/events.html)

`Worker`类表示一个独立的 JavaScript 执行线程。大多数 Node.js API 都可以在其中使用。

Worker环境中的显著差异如下：

- process.stdin，process.stdout 和 process.stderr 可以由父线程重定向。
- require（'worker_threads'）。isMainThread属性设置为false。
- require（'worker_threads'）。parentPort消息端口可用。
- process.exit（）不会停止整个程序，只是单个线程，而process.abort（）不可用。
- process.chdir（）和设置组或用户ID的进程方法不可用。
- process.env是父线程的环境变量的副本，除非另有说明。对一个副本的更改在其他线程中不可见，并且对于本机加载项不可见（除非将worker.SHARE_ENV作为env选项传递给Worker构造函数）。
- process.title无法修改。
- 信号不会通过process.on（'...'）传递。
- 由于调用了worker.terminate（），执行可能会在任何时候停止。
- 来自父进程的IPC通道无法访问。
- 不支持trace_events模块
- 只有满足特定条件的情况下，才能从多个线程加载本机加载项。

可以在其他Worker中创建Worker实例。

与Web Workers和集群模块一样，可以通过线程间消息传递实现双向通信。在内部，Worker有一对内置的MessagePort，它们在创建Worker时已经相互关联。虽然父端的MessagePort对象未直接暴露，但其功能通过 worker.postMessage（）和父线程的Worker对象上的worker.on（'message'）事件暴露。

要创建自定义消息传递通道（鼓励使用默认全局通道，因为它有助于分离关注点），用户可以在任一线程上创建MessageChannel对象，并通过预先存在的方式将该MessageChannel上的其中一个MessagePort传递给另一个线程频道，例如全局频道。

有关如何传递更多信息，以及通过线程关口成功传输哪种JavaScript值，请参阅port.postMessage（）。

```javascript
const assert = require('assert');
const {
  Worker, MessageChannel, MessagePort, isMainThread, parentPort
} = require('worker_threads');
if (isMainThread) {
  const worker = new Worker(__filename);
  const subChannel = new MessageChannel();
  worker.postMessage({ hereIsYourPort: subChannel.port1 }, [subChannel.port1]);
  subChannel.port2.on('message', (value) => {
    console.log('received:', value);
  });
} else {
  parentPort.once('message', (value) => {
    assert(value.hereIsYourPort instanceof MessagePort);
    value.hereIsYourPort.postMessage('the worker is sending this');
    value.hereIsYourPort.close();
  });
}
```



#### new Worker(filename[,options])

- `filename` <string>  Worker的主脚本的路径。必须是以./或../开头的绝对路径或相对路径（即相对于当前工作目录）。如果options.eval为true，则这是一个包含JavaScript代码而不是路径的字符串。
- `options` <Object>
  - `env` <Object>  如果设置，则指定Worker线程内process.env的初始值。作为一个特殊值，worker.SHARE_ENV可用于指定父线程和子线程应共享其环境变量;在这种情况下，对一个线程的process.env对象的更改也会影响另一个线程。默认值：process.env。
  - `eval`<boolean> 如果为true，则将构造函数的第一个参数解释为在工作程序启动时执行的脚本。
  - `execArgv`<string[]>  传递给worker的节点CLI选项列表。不支持V8选项（例如`--max-old-space-size`）和影响进程的选项（例如`--title`）。如果设置，则将在worker中作为process.execArgv提供。默认情况下，选项将从父线程继承。
  - `stdin` <boolean> 如果将其设置为true，则worker.stdin将提供可写流，其内容将在Worker内显示为process.stdin。默认情况下，不提供任何数据。
  - `stdout`<boolean> 如果将此值设置为true，则不会自动将worker.stdout传递给父级中的process.stdout。
  - `stderr`<boolean> 如果将其设置为true，则worker.stderr将不会自动通过管道传递给父级中的process.stderr。
  - `worderData`<any> 任何将被克隆并可用作需求的JavaScript值（'worker_threads'）。workerData。克隆将按照HTML结构化克隆算法中的描述进行，如果无法克隆对象（例如，因为它包含函数），则会引发错误。



#### 'error'事件

新增于: v10.5.0

- `exitCode` <integer>

一旦worker停止，就会发出`exit`事件。如果工作者通过调用`process.exit（）`退出，则exitCode参数将是传递的退出代码。如果worker终止，则exitCode参数将为1。



#### 'message'事件

新增于: v10.5.0

- `value`<any>  传递的值

当工作线程调用`require（'worker_threads'）.parentPort.postMessage（）`时，会发出`message`事件。有关更多详细信息，请参阅`port.on（'message'）`事件。



#### worker.postMessage(value[,transferList])

新增于: v10.5.0

- `value`<any>
- `transferList`<Object[]>

通过`require（'worker_threads'）.parentPort.on（'message'）`向工作人员发送消息。有关更多详细信息，请参阅`port.postMessage（）`。



#### worker.ref()

新增于: v10.5.0

与`unref()`相反, 在已经`unref（）`的 worker上调用`ref（）`,如果它是唯一的活动句柄，则不会让程序退出（默认行为）。如果worker是已`ref（）`，再次调用`ref（）`将无效。



#### worker.stderr

新增于: v10.5.0

- <stream.Readable>

这是一个可读的流，包含写入工作线程内的`process.stderr`的数据。如果未将`stderr：true`传递给Worker构造函数，则数据将通过管道传递给父线程的`process.stderr`流。



#### worker.stdin

新增于: v10.5.0

- <null> | <stream.Writable>

如果将`stdin：true`传递给Worker构造函数，则这是一个可写流。写入此流的数据将在工作线程中作为`process.stdin`提供。



#### worker.stdout

新增于: v10.5.0

- <stream.Readable>

这是一个可读的流，包含写入工作线程内的`process.stdout`的数据。如果未将`stdout：true`传递给Worker构造函数，则数据将通过管道传递给父线程的`process.stdout`流。



#### worker.terminate()

新增于: v10.5.0

修改于: v12.5.0 此函数现在返回Promise。传递回调是不推荐的，并且在此版本之前没用，因为Worker实际上是同步终止的。终止现在是一个完全异步的操作。

- 返回: <Promise>

尽快停止工作线程中的所有JavaScript执行。返回发出`exit`事件时满足的退出代码的`Promise`。



#### worker.threadId

新增于: v10.5.0

- <integer>

引用线程的整数标识符。在工作线程内部，它可以作为`require（'worker_threads'）.threadId`。对于单个进程内的每个Worker实例，此值是唯一的。



#### worker.unref()

新增于: v10.5.0

如果这是事件系统中唯一的活动句柄，则对worker调用`unref（）`将允许该线程退出。如果worker已经是`unref（）`，则再次调用`unref（）`将无效。


