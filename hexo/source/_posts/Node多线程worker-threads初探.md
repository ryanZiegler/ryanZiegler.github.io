---
title: Node多线程worker_threads初探
date: 2019-09-26 20:31:01
tags:
---
最近在面试求职的时候有被问到 Node 有没有办法实现多线程,我一拍脑袋,害,这个我会:

"利用 cluster 模式通过 fork() 实例化多个 node 服务实例, 比如一个8核服务器,就可以启动8个独立实例,相互之间并不影响,十分稳定.最典型的工具就是 pm2."

"恩恩,你说的是 cluster 多进程模式,有没有多线程方式呢?"

"啊...这个...那就通过 child_process 去 fork/spawn 一个子进程..."

"这也是调起了一个子进程,并不是真正的多线程,有关注过 Node 的一些新特性/api 吗?"

"饿...这个(此时的我是当真不知道 worker_threads...."

然后结局很明显,在欢声笑语中打出GG...

不甘心的我去[Node中文文档](http://nodejs.cn/api/worker_threads.html)查阅了相关资料,果不其然,我还是太年轻了:)

<!-- more -->

害! 为啥 Node中文文档居然英文API,于是我学习的同时顺便翻译了一哈.有兴趣学习或者了解 worker_threads 模块的同学可以点击[这里]([https://iiamego.com/2019/09/19/%E8%AF%91-worker-threads-%E5%B7%A5%E4%BD%9C%E7%BA%BF%E7%A8%8B/](https://iiamego.com/2019/09/19/译-worker-threads-工作线程/))查阅~~(谷歌翻译+作者人肉翻译,如有错谬,还望海涵)~~

ps: Node不是单线程的么? 实际上并不是, Node 单线程是指 v8 引擎执行 js 时是单线程的,就好像浏览器一个Tab进程中,就有GUI渲染线程, JS引擎线程,事件触发线程,定时器触发线程,异步请求http线程. [Node.js 异步原理](https://segmentfault.com/a/1190000019111942#articleHeader0)其实是通过 libuv 的线程池完成异步调用的;当你启动一个Node服务时,打开任务管理器,你会发现 Node 任务的线程数为7(主线程,编译/优化线程,分析器线程,垃圾回收线程等)

#### worker_threads模块内的对象和类

- isMainThread: true表示为主线程, false表示为 worker 线程
- parentPort: 主线程为null, worker 线程表示为父进程的 MessagePort 类型对象
- SHARE_ENV: 指示主线程和 worker线程应共享的环境变量
- threadId: 当前线程的整数标识符,唯一
- workerData: 创建 worker 线程的初始化数据
- MessageChannel 类: 类的实例表示异步双向通信通道,即 MessagePort 实例
- MessagePort 类: 类的实例表示异步双向通信通道的一端
- Worker 类: 表示一个独立的 JavaScript 执行线程

#### 如何创建一个子线程

首先确认自己 Node 版本环境支持 worker_threads 模块.如不支持,可以通过nvm下个最新的 Node.

笔者在 Node.js 开发版本v12.6.0上测试该模块. 首先引入模块:

```javascript
// main.js
const { Worker, isMainThread, parentPort, workerData, MessageChannel } = require('worker_threads');

if (isMainThread) {
    console.log('我是主线程', isMainThread);
    const worker = new Worker(__filename);
} else {
    console.log('我不是主线程', isMainThread);
}

=> 我是主线程 true
=> 我不是主线程 false
```

例子通过 new Worker 生成子线程重新执行了 main.js ,执行完毕, worker子线程自动销毁.

`__filename` 你可以写成你所要执行的具体 `worker.js` 所在的路径.

除了通过 new Worker 去加载执行 js 文件,有没有办法直接执行 js 代码呢, 如下所示:

```javascript
let code = `
for(let i = 0;i < 5;i++) {
    console.log('worker线程执行中:', i);
}
`

let worker = new Worker(code, { eval: true });
console.log('主线程执行完毕');

=> 主线程执行完毕
=> worker线程执行中: 0
=> worker线程执行中: 1
=> worker线程执行中: 2
=> worker线程执行中: 3
=> worker线程执行中: 4
```

> 如果通过 port 设置了 port.on 监听事件, 除非手动 terminate 终结, 否则线程不会自动中断(或者和我一样使用 port.once 即监听一次) 

#### 线程初始化数据

即通过`workerData`完成完成线程数据初始化

```javascript
const data = {
    name: 'ego同学',
    age: 23,
    sex: 'male',
    addr: '深圳南山',
    arr: [{ skill: 'coding'}, { hobby: 'basketball' }]
}

if (isMainThread) {
    const worker = new Worker(__filename, { workerData: data });
} else {
    workerData.age = 16;
    workerData.arr[0].skill = 'sleep';
    console.log(data);
    console.log(workerData);
}

=>
{
  name: 'ego同学',
  age: 23,
  sex: 'male',
  addr: '深圳南山',
  arr: [ { skill: 'coding' }, { hobby: 'basketball' } ]
}
{
  name: 'ego同学',
  age: 16,
  sex: 'male',
  addr: '深圳南山',
  arr: [ { skill: 'sleep' }, { hobby: 'basketball' } ]
}
```

#### 线程间的通信

##### 父子线程通信

```javascript
if (isMainThread) {
    const worker = new Worker(__filename);

    worker.postMessage({name: 'ego同学'});
    worker.once('message', (message) => {
        console.log('主线程接收信息:', message);
    });
} else {
    parentPort.once('message', (obj) => {
        console.log('子线程接收信息:', obj);
        parentPort.postMessage(obj.name);
    })
}

=> 子线程接收信息: { name: 'ego同学' }
=> 主线程接收信息: ego同学
```

`parentPort`是生成 worker 线程时自动创建的`MessagePort`实例,用于与父进程进行通信.

##### 子线程间通信

```javascript
//main.js
const path = require('path');

const { port1, port2 } = new MessageChannel();
if (isMainThread) {
    const worker1 = new Worker(__filename);
    const worker2 = new Worker(path.join(__dirname, 'worker.js'));

    worker1.postMessage({ port1 }, [ port1 ]);
    worker2.postMessage({ port2 }, [ port2 ]);
} else {
	parentPort.once('message', ({ port1 }) => {
        console.log('子线程1收到port1', port1);
        port1.once('message', (msg) => {
            console.log('子线程1收到', msg);
        })

        port1.postMessage('port1 向 port2 发消息啦');
    })
}

// worker.js
const { parentPort } = require('worker_threads');

parentPort.once('message', ({ port2 }) => {
    console.log('子线程2收到port2');

    port2.once('message', (msg) => {
        console.log('子线程2收到', msg);
    })

    port2.postMessage('这里是port2, over!');
})

=>
子线程1收到port1
子线程2收到port2
子线程1收到 这里是port2, over!
子线程2收到 port1 向 port2 发消息啦
```

简单来说就是父线程将`MessageChannel`类生成的`MessagePort`对象实例分别发送到子线程中,两个子线程即可通过 `port1`,`port2`进行通信.

菜徐坤疑问:

worker线程实例可不可以通过`workerData`传递到另一个worker线程里直接使用呢?试一下:

```javascript
// main.js
const worker1 = new Worker(__filename);
const worker2 = new Worker(path.join(__dirname, 'worker.js'), { workerData: worker1 });

// worker.js
const { workerData } = require('worker_threads');
console.log(workerData);

=>
internal/worker.js:144
    this[kPort].postMessage({
                ^

DOMException [DataCloneError]: (name) => {
    if (name === eventName && eventEmitter.listenerCount(eventName) === 0) {
      port.ref();
 ...<omitted>... } could not be cloned.
```

抛出数据克隆错误 !

OK,我们来看看 [Node worker_thread模块源码](https://github.com/nodejs/node/blob/master/lib/internal/worker.js) 如下:

```javascript
// node/lib/internal/worker.js

...
const url = options.eval ? null : pathToFileURL(filename);
// Set up the C++ handle for the worker, as well as some internal wiring.
// 为工作程序设置C ++句柄以及一些内部连线。
this[kHandle] = new WorkerImpl(url, options.execArgv);

...

this[kPort] = this[kHandle].messagePort;
this[kPort].on('message', (data) => this[kOnMessage](data));

...

const { port1, port2 } = new MessageChannel();
this[kPublicPort] = port1;
this[kPublicPort].on('message', (message) => this.emit('message', message));
setupPortReferencing(this[kPublicPort], this, 'message');
this[kPort].postMessage({
	type: messageTypes.LOAD_SCRIPT,
    filename,
    doEval: !!options.eval,
    cwdCounter: cwdCounter || workerIo.sharedCwdCounter,
    workerData: options.workerData,
    publicPort: port2,
    manifestSrc: getOptionValue('--experimental-policy') ?
      require('internal/process/policy').src :
      null,
    hasStdin: !!options.stdin
}, [port2]);
// Actually start the new thread now that everything is in place.
// 现在，一切就绪，实际上开始新线程。
this[kHandle].startThread();
```

`workerData`是通过 `port.postMessagePort(value[, transferList])` 克隆副本传输给目标线程,即`workerData`通过[结构化克隆算法](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)进行复制:

> It builds up a clone by recursing through the input object while maintaining a map of previously visited references in order to avoid infinitely traversing cycles.
>
> 通过递归遍历输入对象而建立一个副本，同时保持先前访问的引用的映射，以避免无限遍历循环。

该算法不复制函数、错误、属性描述符或原型链,可以包含循环引用和类型化数组.

#### 线程共享内存

`port.postMessage(value[, transferList])`

`transferList`可以是`ArrayBuffer`和`MessagePort`对象的列表, 传递`ArrayBuffer`后,访问权限被修改,归属于消息接收方,它们将不再可用于频道的发送方 !!!

线程间通过 clone 第一个参数来互相传递消息, 那如果我不想到处 clone 到处传递数据呢, 有办法解决吗? 答案是有的.

`cluster` 和 `child_process` 时通常使用 `SharedArrayBuffer` 来实现需要多进程共享的内存, 同样的 `value`可以包含`SharedArrayBuffer`实例,从而可以在任一线程访问这些实例 !

```javascript
const { Worker, isMainThread, parentPort, MessageChannel, threadId } = require('worker_threads');

if (isMainThread) {
    const worker1 = new Worker(__filename);
    const worker2 = new Worker(__filename);
    
    const { port1, port2 } = new MessageChannel();
    const sharedUint8Array = new Uint8Array(new SharedArrayBuffer(4));
	// 输出一下sharedUint8Array
    console.log(sharedUint8Array);
    worker1.postMessage({ uPort: port1, data: sharedUint8Array }, [ port1 ]);
    worker2.postMessage({ uPort: port2, data: sharedUint8Array }, [ port2 ]);

    worker2.once('message', (message) => {
        console.log(`${message}, 查看共享内存:${sharedUint8Array}`);
    });
} else {
    parentPort.once('message', ({ uPort, data }) => {
        uPort.postMessage(`我是${threadId}号线程`);
        uPort.on('message', (msg) => {
            console.log(`${threadId}号收到:${msg}`);
            if (threadId === 2) {
                data[1] = 2;
                parentPort.postMessage('2号线程修改了共享内存!!!');
            }
            console.log(`${threadId}号查看共享内存:${data}`);
        })
    })
}

=>
Uint8Array [ 0, 0, 0, 0 ]
2号收到:我是1号线程
2号线程修改了共享内存!!!, 查看共享内存:0,2,0,0
1号收到:我是2号线程
2号查看共享内存:0,2,0,0
1号查看共享内存:0,2,0,0
```

通过共享内存,  我们在一个线程中修改它, 意味所有线程中中进行了修改, 意味着数据的传递修改无需多次序列化clone, 是不是方便很多呢.

如果不满足共享一个 Buffer 数组, 通常数据都是以对象的形式来存储传递的, 我们可以[创建类似的结构](https://stackoverflow.com/questions/51053222/nodejs-worker-threads-shared-object-store)来达到我们的目的.

#### 创建线程池

worker 工作线程一般有两种使用方法:

- 实行创建任务线程, 任务执行完毕后销毁释放
- 预先创建线程池, 通过调度规划执行任务

官方推荐线程池的使用方法, 毕竟一次次的创建销毁 worker 需要占用不小的开销, 我们可以根据实际业务情况来选择自己的使用方式.

下面我们来实现一个简单的 worker_thread 线程池:

```javascript
// main.js
const path = require('path');
const { Worker } = require('worker_threads');

class WorkerPool {
    _workers = [];                    // 线程引用数组
    _activeWorkers = [];              // 激活的线程数组
    _queue = [];                      // 任务队列

    constructor(workerPath, numOfThreads) {
        this.workerPath = workerPath;
        this.numOfThreads = numOfThreads;
        this.init();
    }

    // 初始化多线程
    init() {
        if (this.numOfThreads < 1) {
            throw new Error('线程池最小线程数应为1');
        }

        for (let i = 0;i < this.numOfThreads; i++) {
            const worker = new Worker(this.workerPath);

            this._workers[i] = worker;
            this._activeWorkers[i] = false;
        }
    }

    // 结束线程池中所有线程
    destroy() {
        for (let i = 0; i < this.numOfThreads; i++) {
            if (this._activeWorkers[i]) {
                throw new Error(`${i}号线程仍在工作中...`);
            }
            this._workers[i].terminate();
        }
    }

    // 检查是否有空闲worker
    checkWorkers() {
        for (let i = 0; i < this.numOfThreads; i++) {
            if (!this._activeWorkers[i]) {
                return i;
            }
        }
          
        return -1;
    }

    run(getData) {
        return new Promise((resolve, reject) => {
            const restWorkerId = this.checkWorkers();

            const queueItem = {
                getData,
                callback: (error, result) => {
                    if (error) {
                        return reject(error);
                    }
                    return resolve(result);
                }
            }
            
            // 线程池已满,将任务加入任务队列
            if (restWorkerId === -1) {
                this._queue.push(queueItem);
                return null;
            }
            
            // 空闲线程执行任务
            this.runWorker(restWorkerId, queueItem);
        })
    }

    async runWorker(workerId, queueItem) {
        const worker = this._workers[workerId];
        this._activeWorkers[workerId] = true;

        // 线程结果回调
        const messageCallback = (result) => {
            queueItem.callback(null, result);
            cleanUp();
        };
           
        // 线程错误回调
        const errorCallback = (error) => {
            queueItem.callback(error);
            cleanUp();
        };

        // 任务结束消除旧监听器,若还有待完成任务,继续完成
        const cleanUp = () => {
            worker.removeAllListeners('message');
            worker.removeAllListeners('error');

            this._activeWorkers[workerId] = false;

            if (!this._queue.length) {
                return null;
            }

            this.runWorker(workerId, this._queue.shift());
        }

        // 线程创建监听结果/错误回调
        worker.once('message', messageCallback);
        worker.once('error', errorCallback);
        // 向子线程传递初始data
        worker.postMessage(queueItem.getData);
    }
}
```

创建一个`workerPool`类,在构造函数里传入要执行的js文件路径和要启动的线程池数,然后通过`init()`方法初始化多线程,并将它们的引用存储在`_workers`数组里,初始状态默认都为`false`不活跃存储在`_activeWorkers`数组中.

`run`方法分配执行任务,返回`Promise`调用任务的回调去`resolve/reject`,使用空闲线程`runWorker`执行任务,如果暂时没有空闲线程,就把任务`push`进`_queue`任务队列等待执行.

`runWorker`使用空闲线程指定任务,定义好结果回调和`error`回调,通过设置子线程的监听事件传递回调结果,把子线程的计算结果传递出来

然后我们创建一个 `worker.js `在里面写CPU密集耗时操作:

```javascript
// worker.js
const { isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
    throw new Error('Its not a worker');
}
    
const doCalcs = (data) => {
    const collection = [];
    
    for (let i = 0; i < 10; i++) {
        collection[i] = Math.round(Math.random() * 1000);
    }
    
    return collection.sort((a, b) => { return a - b });
};
    
parentPort.on('message', (data) => {
    const result = doCalcs(data);
    parentPort.postMessage(result);
});
```

~~这里就简单写了点排序,方便输出...~~

然后通过`new WorkerPool`生成实例执行任务:

```javascript
onst pool = new WorkerPool(path.join(__dirname, 'worker.js'), 4);

const items = [...new Array(10)].fill(null);

Promise.all(items.map(async (_, i) => {
	const res = await pool.run();

    console.log(`任务${i}完成结果:`, res);
})).then(() => {
    console.log('所有任务完成 !');
    // 销毁线程池
    pool.destroy();
});

=>
任务1完成结果: [
   45,  96, 197, 314,
  606, 631, 648, 648,
  658, 874
]
任务4完成结果: [
   68,  86, 124, 330,
  330, 469, 533, 766,
  772, 900
]
任务5完成结果: [
  107, 344, 370, 499,
  504, 627, 750, 840,
  873, 972
]
任务6完成结果: [
  218, 257, 282, 284,
  500, 607, 699, 723,
  739, 826
]
任务7完成结果: [
   31,  98, 141, 190,
  428, 507, 685, 686,
  794, 945
]
任务8完成结果: [
   27, 100, 188, 245,
  471, 497, 514, 620,
  645, 993
]
任务9完成结果: [
  193, 336, 407, 455,
  478, 534, 564, 651,
  755, 963
]
任务2完成结果: [
  319, 337, 398, 549,
  587, 659, 670, 781,
  792, 843
]
任务3完成结果: [
  173, 188, 273, 406,
  445, 450, 582, 678,
  727, 882
]
任务0完成结果: [
   38,  76, 134, 239,
  439, 468, 568, 696,
  910, 923
]
所有任务完成 !
```

#### 结语

`worker_threads`模块提供了真正的单进程多线程使用方法,我们可以将CPU密集的任务交给线程去解决,等有了结果后通过`MessageChannel`跨线程通信/或者使用共享内存.

当然,上面这些例子都是最简单最基本的使用方式,真正运用到生产中根据不同的业务复杂度,`worker_threads`可能会有各种花里胡哨的运用和实现.

#### 参考文章

这些文章在我学习过程给予我非常大的帮助,有些作者已经在实际生产业务中有所实践,他们的学习过程和经验非常值得我们学习~有兴趣的同学可以选读一些加以学习,有所收获!

[Node.js 真·多线程 Worker Threads 初探](https://blog.csdn.net/weixin_33691598/article/details/91392543)

[Node.js 多线程完全指南总结](https://www.jb51.net/article/158538.htm)

[nodejs-worker-threads-shared-object-store](https://stackoverflow.com/questions/51053222/nodejs-worker-threads-shared-object-sto)

[真-Node多线程](https://juejin.im/post/5c63b5676fb9a049ac79a798#heading-4)

[node 线程池技术让文档编译起飞](https://cloud.tencent.com/developer/article/1494064)

[Node.js 异步原理-线程池-libuv](https://segmentfault.com/a/1190000019111942)