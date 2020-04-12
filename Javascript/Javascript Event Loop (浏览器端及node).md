`Javascript`  是一个单线程、非阻塞式的语言。

### 单线程

`Javascript` 是单线程的运行环境，只有一个调用栈，程序每次只能运行一段代码。

以下面这段代码为例

```js
function multiply(a, b) {
    return a * b;
}

function square(n) {
    return multiply(n, n);
}

function printSquare(n) {
    var squared = square(n);
    console.log(squared);
}

printSquare(4);
```

![](https://user-gold-cdn.xitu.io/2020/3/31/17130e0c265bb020?w=2063&h=1185&f=gif&s=1058004)

[调用栈](http://www.ruanyifeng.com/blog/2013/11/stack.html)，当进去某个函数的时候，这个函数就会放到栈里面，当离开了某个函数时，这个函数就会从栈中释放。图中可见，这是一个后进先出的概念


### 阻塞

如果所有代码都是以上述方式执行的话，就会遇到一个问题

像下面这段代码，如果其中一个网络请求特别耗时，那么其余程序就必须要等这个请求执行完了之后再执行，这就存在了阻塞

```js
var foo = $.getSync("//foo.com");
var baz = $.getSync("//baz.com");
var qux = $.getSync("//qux.com");

console.log(foo);
console.log(baz);
console.log(qux);
```

### 异步

浏览器内核是多线程的，常驻线程如下

* 渲染引擎线程：负责页面的渲染
* `JS` 引擎线程：负责 `JS` 的解析和执行
* 定时触发器线程：处理定时事件，比如 `setTimeout`, `setInterval`
* 事件触发线程：处理 `DOM` 事件（注意：[EventTarget.dispatchEvent](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/dispatchEvent) 是同步调用的, 所以其对应触发的事件也都是同步执行； `document.getElementById("button").click()` 是 `JavaScript` 初始化创建的，处理方式与 `dispatchEvent` 相同，所以当 `js` 中执行该代码，里面的回调也是同步执行，下面的例子中会有提及）
* 异步 `http` 请求线程：处理 `http` 请求

其中渲染引擎和 `JS` 引擎是不能同时进行的，防止渲染过程中，有 `JS` 操作 `DOM` 的行为，浏览器不知道该如何处理。

`Javascript` 是单线程，是因为浏览器在运行时只开启了一个 `JS` 引擎线程来解析和执行 `JS`。

但浏览器内部不是单线程的， `I/O` 操作、定时器的计时和事件监听（`click`, `keydown` `...`）等事件都是交给浏览器的其他线程去完成的，这样子就不会阻塞 `JS` 代码的执行

举例：

```js
console.log("hi");

setTimeout(function cb() {
    console.log("cb");
}, 1000);

console.log("over");
```

![](https://user-gold-cdn.xitu.io/2020/3/31/1713127a79a644d5?w=1990&h=1321&f=gif&s=1301923)

如图所示，类似 `setTimeout` 的异步回调，等定时时间到了后，会被先放到任务队列中，等主线程中的任务都完成了之后，才会被调入栈中执行。

（ps. 任务队列是个先进先出的过程）

但是呢，这个需要注意一个问题，`setTimeout` 的回调不是真的在 1s 后才执行。

```js
// 如果在 console.log("over"); 后加这段代码
for(var i = 0; i < 100000; i++) {
    for(var j = 0; j < 100000; j++) {
        // loop
    }
}
```

例如上例，当 1s 后，`cb()` 被加到任务队列后，主线程还在执行 `for` 循环，只有等 `for` 循环后执行完了之后，`cb()` 才能被推到栈中，这就造成了 `cb()` 没有在 1s 后执行。

### Microtask （微任务）

微任务呢，也不是同步任务。但是他是在所有同步任务执行完，调用栈中没有任务，就会立即执行的任务。

#### 宏任务（任务队列中的任务）与微任务的区别

微任务优先于宏任务的执行，当执行队列为空时，就会立即执行。

宏任务一个事件周期内只会执行一个宏任务，如果下个事件周期中有 `UI` 渲染或者其他事件的话，就得等其他事件都处理完后，才会去执行。

但微任务的话，就不一样，事件循环会持续调用微任务直至微任务队列中没有留存的，即使是在有更多微任务持续被加入的情况下(即在执行微任务的过程中又加入了微任务，那么新的微任务也会在这个事件周期内完成，所以使用微任务时，需要注意代码是否会进入无限处理微任务的循环中)

所以像希望回调代码保证能一定执行顺序的话，就用微任务好了

微任务常有接口有：`process.nextTick`(`Node` 独有), `Promises`, `MutationObserver`

宏任务常有接口有：`setTimeout`、`setInterval`、`setImmediate` (`Node` 独有)、`requestAnimationFrame` (浏览器独有)、`I/O`、`UI rendering` (浏览器独有)

### web worker

具体操作可参考阮一峰老师的文章：[Web Worker 使用教程](http://www.ruanyifeng.com/blog/2018/07/web-worker.html)

`Web Worker` 的作用，就是为 `JavaScript` 创造多线程环境，允许主线程创建 `Worker` 线程，将一些任务分配给后者运行。在主线程运行的同时，`Worker` 线程在后台运行，两者互不干扰。等到 `Worker` 线程完成计算任务，再把结果返回给主线程。这是一个并行的过程，不同于异步；异步最后还是会到主线程上执行，表现形式还是一个串行的方式。并行处理的好处是，一些计算密集型或高延迟的任务，被 `Worker` 线程负担了，主线程（通常负责 `UI` 交互）就会很流畅，不会被阻塞或拖慢。

`Worker` 线程一旦新建成功，就会始终运行，不会被主线程上的活动（比如用户点击按钮、提交表单）打断。这样有利于随时响应主线程的通信。但是，这也造成了 `Worker` 比较耗费资源，不应该过度使用，而且一旦使用完毕，就应该关闭。

#### worker 线程执行流程

```js
// main.js
var worker = new Worker(task.js')
worker.postMessage("worker");
```

* `worker` 线程的创建的是异步的

代码执行到 `var worker = new Worker(task.js')` 时，在内核中构造 `WebCore::JSWorker` 对象（`JSBbindings` 层）以及对应的 `WebCore::Worker` 对象（`WebCore` 模块)，根据初始化的 `url` 地址 `task.js` 发起异步加载的流程；主线程代码不会阻塞在这里等待 `worker` 线程去加载、执行指定的脚本文件，而是会立即向下继续执行后面代码。

* `postMessage` 消息交互由内核调度

`main.js` 中，在创建 `woker` 线程后，立即调用了 `postMessage` 方法传递了数据，在 `worker` 线程还没创建完成时，`main.js` 中发出的消息，会先存储在一个临时消息队列中，当异步创建 `worker` 线程完成，临时消息队列中的消息数据复制到 `woker` 对应的 `WorkerRunLoop` 的消息队列中，`worker` 线程开始处理消息。在经过一轮消息来回后，继续通信时， 这个时候因为 `worker` 线程已经创建，所以消息会直接添加到 `WorkerRunLoop` 的消息队列中；

* `worker` 线程数据通讯方式

主线程与子线程数据通信方式有多种，通信内容，可以是文本，也可以是对象。需要注意的是，这种通信是拷贝关系，即是传值而不是地址，子线程对通信内容的修改，不会影响到主线程。事实上，浏览器内部的运行机制是，先将通信内容串行化，然后把串行化后的字符串发给子线程，后者再将它还原。

![](https://user-gold-cdn.xitu.io/2020/4/3/1713e1862998ce2d?w=829&h=375&f=png&s=51223)

### 浏览器下的 事件循环

![](https://user-gold-cdn.xitu.io/2020/4/4/17140e5840fce0a4?w=2479&h=1375&f=gif&s=1861248)

由上图可以看出，这比之前的异步流程多了一个 `UI` 渲染过程

渲染过程分为：

* `Structure` - 构建 `DOM` 树的结构
* `Layout` - 确认每个 `DOM` 的大致位置（排版）
* `Paint` - 绘制每个 `DOM` 具体的内容（绘制）

同时也可以得到以下结论

**1. 如果主进程或者异步队列被阻塞了的话，事件环会永远停留在该处，也就走不到 `UI` 渲染处了。（之前也提到 `js` 线程和 `UI` 线程是互斥的）**

```js
// js 线程会一直停留在 while 语句的执行，不会再去进行 UI 渲染
while(true) {}
```

**2. 使用 `js` 操作 `DOM` 的话，不会存在先后顺序（除非存在其他计算样式的代码）。**

像下例，这是同步操作，事件环在没有执行完同步任务，是不会走到 `UI` 渲染处的
```js
// 页面不会出现闪现现象
document.getElementById("div").style.display = "block";
document.getElementById("div").style.display = "none";
```

如果一定在 `js` 代码中实现 `css` 样式变换，如何解决呢？

a. 下面这种情况也还是会被同步执行，因为 `requesAnimation`  也是实际渲染前执行的代码，所以样式还是会被合并

```js
// 错误
box.style.transform = 'translateX(1000px)'
requestAnimationFrame(() => {
  box.style.tranition = 'transform 1s ease'
  box.style.transform = 'translateX(500px)'
})
```

b. 但如果在执行 `requestAnimationFrame` 的过程中，又出现了 `requestAnimationFrame` 代码，那么后出现的 `requestAnimationFrame` 是被放到下一次渲染前再执行，所以渲染会被分成 2 帧渲染
```js
// 可以
box.style.transform = 'translateX(1000px)'
requestAnimationFrame(() => {  // 第一帧渲染
  requestAnimationFrame(() => {  // 第二帧才会被渲染
    box.style.transition = 'transform 1s ease'
    box.style.transform = 'translateX(500px)'
  })
})
```

c. 在执行的过程中，计算下其他样式，也会引起页面的重新渲染
```js
// 更简单的方法，执行过程中，计算下样式即可
box.style.transform = 'translateX(1000px)'
getComputedStyle(box) // 伪代码，只要获取一下当前的计算样式即可
box.style.transition = 'transform 1s ease'
box.style.transform = 'translateX(500px)'
```

**3. `setTimeout` 类似的异步回调，不会阻塞 `UI` 渲染**

因为异步任务一次只会执行一次，执行下一次的时候，还会再去事件环中查看是否有其他任务，如果在查看的过程中，发现需要进行 `UI` 渲染的话，就会先去执行浏览器渲染，再去执行异步回调

4. **`requestAnimationFrame` 与 `setTimeout` 的不同**

`requestAnimationFrame` 这是一个特别的异步任务，注册的方法不加入异步队列，而是加入渲染这一边的队列中，它在渲染 3 个步骤前被执行(edge / safari 则是在渲染 3 个步骤后被执行，所以这有可能导致一些浏览器兼容性的问题)。

接下来，我们看下例，

```js
// 方法 1
function callback() {
  moveBoxForwardOnePixel();   // 向右移动 1 px
  requestAnimationFrame(callback)
}
callback()

// 方法 2
function callback() {
  moveBoxForwardOnePixel();
  setTimeout(callback, 0)
}
callback()
```

结果如下，`setTimeout` 向右移动的速度比 `requestAnimationFrame` 快。

这是因为浏览器渲染有固定的频率，`requestAnimationFrame` 只在 `UI` 渲染前执行，相当于一帧只渲染一次；

而在两次渲染间隔的时间里，`setTimeout` 可能已经执行过很多次了，最后呈现在显示器上向右移动的速度也就会比较的快了，这就是为什么 `setTimeout` 的图像看起来会有跳动的原因

所以动画的话，最好是使用 `requestAnimationFrame`

![](https://user-gold-cdn.xitu.io/2020/4/3/1713ee6b97d867de?w=1431&h=526&f=gif&s=411810)

5. **理解 用户点击事件 和 `js` 的 `click()` 不同**

来看下面两例

```js
let button = document.querySelector('#button');

button.addEventListener('click', function CB1() {
  console.log('Listener 1');
  setTimeout(() => console.log('Timeout 1'))
  Promise.resolve().then(() => console.log('Promise 1'))
});

button.addEventListener('click', function CB1() {
  console.log('Listener 2');
  setTimeout(() => console.log('Timeout 2'))
  Promise.resolve().then(() => console.log('Promise 2'))
});
```

鼠标点击触发结果: 
`Listener 1  =>  Promise 1 => Listener 2 => Promise 2 => Timeout 1 => Timeout 2`

`button.click()` 结果: 
`Listener 1 => Listener 2 => Promise 1 => Promise 2 => Timeout 1 => Timeout 2`

原因：

鼠标点击：

![](https://user-gold-cdn.xitu.io/2020/4/5/171460d77f10c79e?w=1911&h=984&f=gif&s=2247050)

`js` 出发 `click()` 事件：


![](https://user-gold-cdn.xitu.io/2020/4/5/171463ff5910dbe7?w=1911&h=951&f=gif&s=2550656)

所以如果如果自动化测试中，我们用 `js` 的 `click()` 方法来模拟鼠标点击事件，就会导致和现实结果不一样的现象

再来看另外一个例子

```js
const nextClick = new Promise(resolve => {
    link.addEventListener('click', resolve, {once: true});
});

nextClick.then(event => {
    event.preventDefault();
});
```

上面这段代码，如果是鼠标点击事件的话，是没问题的，但是如果是 `js` 模拟的 `click()` 事件，`<a>` 链接跳转就会有问题

因为处理链接的话，一般有以下步骤，
* 创建一个新的事件对象 `eventObject`
* 调用每个 `click` 的事件监听回调，并传入 `eventObject`
* 检查 `eventObject` 的 `canceled` 属性，如果是 `cancel` ，那么就不会打开链接，另则反之（当调用 `event.preventDefault()` 时，该属性就会被标记成 `canceled`）

所以如果是鼠标点击的话，调用栈是空的，就会执行回调函数（微任务），不会跳转；但如果是 `click()` 事件触发的话，调用栈不为空，再执行上述步骤的时候，不会走到回调函数（微任务）。

### node event loop

`Node.js` 也是单线程的 `Event Loop`，但是它的运行机制不同于浏览器环境。

`Node` 运行机制如下：
* `V8` 引擎解析 `JavaScript` 脚本。
* 解析后的代码，调用 `Node API`。
* `libuv` 库负责 `Node API` 的执行。它将不同的任务分配给不同的线程，形成一个 `Event Loop`（事件循环），以异步的方式将任务的执行结果返回给 `V8` 引擎。
* `V8` 引擎再将结果返回给用户。

![](https://user-gold-cdn.xitu.io/2020/4/6/1714b45b44e8e523?w=800&h=316&f=png&s=56063)

#### 阶段概述

由下图可看到，一次事件循环分为多个阶段

* 每次事件循环开始，会先记录当前时间（这是为了后续减少与时间相关的系统调用次数）
* 然后判断当前循环对象是否存活（是否在等待任何异步 I/O 或计时器），如果是则继续后续的事件循环，否的话，`Node.js` 进程退出
* `timers`：本阶段执行定时时间到了的 `setTimeout()` 和 `setInterval()` 的回调函数，**何时执行则由 `poll` 阶段控制**
* `pending callbacks`：执行延迟到下一个循环迭代的 `I/O` 回调。
* `idle`, `prepare`：仅 `node` 内部使用。
* `poll`：检索新的 `I/O` 事件; 执行与 `I/O` 相关的回调（除了 `close` 、定时器和 `setImmediate()` 回调的之外的所有回调都在这个阶段执行），适当条件下， `node` 将阻塞在这里。
* `check`：执行 `setImmediate()` 回调函数。
* `close callbacks`：一些关闭的回调函数，如：`socket.on('close', ...)`。

![](https://user-gold-cdn.xitu.io/2020/4/7/17154f4254da34be?w=522&h=740&f=png&s=77269)

**详细说明**

##### timers (定时器)

这个是定时器阶段，处理 `setTimeout()` 和 `setInterval()` 的回调函数。进入这个阶段后，主线程会检查一下当前时间，是否满足定时器的条件。如果满足就执行回调函数，否则就离开这个阶段。

##### pending callbacks (挂起的回调函数)

此阶段对某些系统操作（如 `TCP` 错误类型）执行回调。例如，如果 `TCP` 套接字在尝试连接时接收到 `ECONNREFUSED`，则某些 `*nix` 的系统希望等待报告错误。这将被排队以在挂起的回调阶段执行。

##### poll (轮询)

事件循环进入到轮询阶段，如果没有达到计时时间的计时器，那么将会法发生以下两种情况之一：

* 如果轮询队列不是空，事件循环访问回调队列并 **同步** 执行他们，直到队列为空，或者达到了系统限制的最大回调次数。
* 如果轮询队列为空，还有两件事发生：
   * 如果有 `setImmediate` 回调需要执行，`poll` 阶段会停止并且进入到 `check` 阶段执行回调
   * 如果没有 `setImmediate` 回调需要执行，就会一直等待回调被加入到队列中（此时就会发生阻塞），如果有新任务加入就会立即执行回调。当然这个阶段也设置了最大超时时间，防止在这个阶段被阻塞。
  
一旦轮询队列为空，事件循环将检查是否有达到定时时间的计时器，如果有一个或者多个计时器已经准备就绪，那么事件循环就会通过 `check`、 `close` 阶段，进入下一次轮询的计时器阶段来执行这些计时器的回调

##### close callbacks (关闭的回调函数)

如果套接字或处理函数突然关闭（例如 `socket.destroy()`），则 `'close'` 事件将在这个阶段发出。否则它将通过 `process.nextTick()` 发出。

**看个例子**

下面这个例子，结社读文件过程将消耗 95ms，而执行读文件里的回调函数需要 10ms。

进入第一次事件循环后，定时器定时时间没到，挂起回调函数也没有需要执行的，那么就到了轮询阶段，此时读文件还未完成，所以是个空队列，轮询阶段就处于一个等待的状态，95ms 后文件读完，其回调就会被添加到轮询队列中并执行，当回调完成后，队列中不再有回调，就会查看最快达到定时时间的定时器，然后循环到定时器阶段，执行定时器的回调函数。

所以本例中，定时器实际延迟执行回调的时间为105ms，而非 100ms

```js
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);

// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```

#### setImmediate() 与 setTimeout()

这两个方法呢，比较相似

* `setImmediate()` 当前轮询阶段完成， 就会在 `check` 阶段执行脚本
* `setTimeout()` 在轮询阶段空闲时，且设定时间到达后执行，就会返回 `timer` 阶段执行

例如下例，因为 `setTimeout` 的第二个参数默认为 `0`。但是实际上，`Node`  做不到 `0` 毫秒，最少也需要 `1` 毫秒，根据官方文档，第二个参数的取值范围在 `1` 毫秒到 `2147483647` 毫秒之间。也就是说，`setTimeout(f, 0)` 等同于 `setTimeout(f, 1)`。

实际执行的时候，进入事件循环以后，有可能到了`1` 毫秒，也可能还没到 `1` 毫秒，取决于系统当时的状况。如果没到 `1` 毫秒，那么 `timers` 阶段就会跳过，进入 `check` 阶段，先执行 `setImmediate` 的回调函数。

```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

结果：

```js
// 有可能是
timeout
immediate

// 也有可能是
immediate
timeout
```

**`setImmediate()` 、 `setTimeout()` 哪个情况下才会确认先后执行顺序呢？**

将两个函数放到一个 `I/O` 循环内使用， `setImmediate()` 就会被优先调用，因为都已经到 `I/O` 回调中了，就说明，当前已经处于轮询阶段，而轮询阶段的下一阶段，就是 `check` 阶段，所以必须执行完 `setImmediate()`, 才会到下一轮的 `timer` 阶段

```js
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});

/**
结果：
immediate
timeout
*/
```

使用 `setImmediate()` 相对于 `setTimeout()` 的主要优势是，如果 `setImmediate()` 是在 `I/O` 周期内被调度的，那它将会在其中任何的定时器之前执行，跟这里存在多少个定时器无关

#### node 的微任务队列（process.nextTick() 与 Promise）

上述的 4 个阶段（`setTimeout`、`I/O callback`、`setImmediate()`、`close callback`）都是 `nodejs` 的宏任务。与浏览器不同的地方是，浏览器只有一个宏任务队列，而 `nodejs` 有 4 个不同的宏任务队列。

而 `nodejs` 的微任务队列主要有以下两个：

* `Next Tick Queue`：是放置 `process.nextTick(callback)` 的回调任务的
* `Other Micro Queue`：放置其他 `microtask`，比如 `Promise`等


大体解释一下 `NodeJS` 的 `Event Loop` 过程：

1. 执行全局同步代码
2. 执行 `microtask` 微任务，先执行所有 `Next Tick Queue` 中的所有任务，再执行 `Other Microtask Queue` 中的所有任务
3. 开始执行 `macrotask` 宏任务，共 6 个阶段，从第 1 个阶段开始执行相应每一个阶段 `macrotask` 中的所有任务
4. `node` 10 及其以下的版本是，每个阶段的宏任务队列中的回调都结束了之后，就会去执行微任务
    
    `Timers Queue` -> 步骤2 -> `I/O Queue` -> 步骤2 -> `Check Queue` -> 步骤2 -> `Close Callback Queue` -> 步骤2 -> `Timers Queue` ......
5. 而 `node` 11 及以上的版本，1个宏任务结束后，就会去执行微任务

例如下例：
```js
console.log('start');

setTimeout(() => {
    process.nextTick(function() {
        console.log('process 1');
    });
    console.log('timer 1');
});

setTimeout(() => {
    process.nextTick(function() {
        console.log('process 1');
    });
    console.log('timer 2');
})

var promise1 = new Promise((resolve, reject) => {
    console.log('promise 1');
    resolve();
}).then(() => {
    console.log('promise then 1');
})

process.nextTick(function() {
    console.log('process 3');
});

console.log('end');

/**
* node 10 及以下版本
* start
* promise 1
* end
* process 3
* promise then 1
* timer 1
* timer 2
* process 1
* process 1
*/

/**
* node 11 以上的版本
* start
* promise 1
* end
* process 3
* promise then 1
* timer 1
* process 1
* timer 2
* process 1
*/
```


### 参考
* [What the heck is the event loop anyway? | Philip Roberts](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
* [Jake Archibald: In The Loop - JSConf.Asia](https://www.youtube.com/watch?v=cCOL7MC4Pl0)
* [Further Adventures of the Event Loop - Erin Zimmer - JSConf](https://www.youtube.com/watch?v=u1kqx6AenYw)
* [JavaScript 运行机制详解：再谈Event Loop](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
* [带你彻底弄懂Event Loop](https://juejin.im/post/5b8f76675188255c7c653811)
* [Node.js 事件循环，定时器和 process.nextTick()](https://nodejs.org/zh-cn/docs/guides/event-loop-timers-and-nexttick/)
* [Node.js事件循环](https://www.yuque.com/baichuan/notes/aqi8yg?language=en-us)
* [Node 定时器详解](http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html)
* [深入理解js事件循环机制（Node.js篇）](http://lynnelv.github.io/js-event-loop-nodejs)
* [浏览器与Node的事件循环(Event Loop)有何区别?](https://juejin.im/post/5c337ae06fb9a049bc4cd218#heading-11)
* [【转向 Javascript 系列】深入理解 Web Worker](http://www.alloyteam.com/2015/11/deep-in-web-worker/)
* [创建自定义事件](https://zh.javascript.info/dispatch-events)
* [In depth: Microtasks and the JavaScript runtime environment](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth)
* [在 JavaScript 中通过 queueMicrotask() 使用微任务](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_DOM_API/Microtask_guide)
* [浏览器的 Event Loop](https://juejin.im/post/5c32eb726fb9a049ee809e2f)
* [JavaScript异步机制详解](https://juejin.im/post/5a6ad46ef265da3e513352c8)

感觉上面几个视频讲的挺好的，访问不了的话，可以到B站上，有同款
