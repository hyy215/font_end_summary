本文仅简单介绍洋葱模型原理

### 经典示例

```js
const Koa = require('koa');
const app = new Koa();

app.use(async (ctx, next) => {
  console.log('1');
  await next();
  console.log('1-1');
});

app.use(async (ctx, next) => {
  console.log('2');
  await next();
  console.log('2-2');
});

app.use(async (ctx, next) => {
  console.log('3');
  await next();
  console.log('3-2');
});

app.listen(3000);
```

#### 输出结果

访问 `127.0.0.1:3000`，结果如下

```js
1
2
3
3-2
2-2
1-1
```


![](https://user-gold-cdn.xitu.io/2020/3/23/171051c3d31d8bac?w=1484&h=619&f=png&s=29499)

这就是经典的 `koa` 洋葱模型


![](https://user-gold-cdn.xitu.io/2020/3/23/171051c92da0c0d8?w=478&h=435&f=png&s=136596)

<br>

### 那么为什么呢？

#### 参考源码，构造一个 `App` 的类

```js
class App {
    constructor() {
        // 定义中间件数组
        this.middleware = [];
    }

    use(fn) {
        if (fn && typeof fn !== "function") throw new Error('入参必须是函数');
        // 入参 fn 都传入到 middleware 中间件数组中
        this.middleware.push(fn);
    }

    listen(...arg) {
        /**
        * 源码，this.callbakck() 作为请求处理函数，本处省略该过程
        * const server = http.createServer(this.callback());
        * return server.listen(...args);
        */
        this.callback();
    }

    callback() {
        const fn = compose(this.middleware);
        return this.handleRequest(fn);
    }

    handleRequest(fnMiddleware) {
        return fnMiddleware()
            .then(() => { console.log('over'); })
            .catch((err) => { console.log(err); });
    }
}
```

* `App` 定义了 `this.middleware` 这样一个数组，用来放 `app.use(fn)` 传进来的中间件方法 `fn`
* `app.listen(3000)` 实际是创建一个 `HTTP` 服务器（本文为了简化，省略了该步骤）。`this.callback()` 就是 `http.createServer()` 的回调函数，用来处理 `http` 请求（ps. 也可以通过 `app.callback()` 方法来改写）
* `this.callback()` 呢，实际是在执行 `compose(this.middleware)` 函数返回的结果

#### 接下来我们看 `compose` 这个函数

```js
// 简化 koa-compose 源码下的 compose 方法
function compose(middleware) {
    return function (context, next) {
        return dispatch(0)
        function dispatch(i) {
            let fn = middleware[i]
            if (i === middleware.length) fn = next
            if (!fn) return Promise.resolve()
            try {
                return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
}
```

`compose(this.middleware)` 返回了一个可遍历执行中间件方法 `this.middleware` 的函数 `fn`

通过本文开头的例子，我们再来了解下 `fn` 的执行过程

```js
// dispatch(0)
Promise.resolve((async (ctx, next) => {
    console.log('1');
    await next();
    console.log('1-1');
})(context, dispatch.bind(null, 1)));

// dispatch(0) 中的入参 next 为 dispatch.bind(null, 1)
// 所以 next() 相当于执行 dispatch(1)
Promise.resolve((async (ctx, next) => {
    console.log('2');
    await next();
    console.log('2-1');
})(context, dispatch.bind(null, 2)));

// dispatch(1) 中的入参 next 为 dispatch.bind(null, 2)
// 所以 next() 相当于执行 dispatch(2)
Promise.resolve((async (ctx, next) => {
    console.log('3');
    await next();
    console.log('3-1');
})(context, dispatch.bind(null, 3)));

// 由于 this.middleware[3] 不存在，所以直接返回了一个 Promise.resolve();
```

所以根据代码的执行顺序，就是 `1, 2, 3, 3-1, 2-1, 1-1`

#### 错误捕获

可以看到在 `fnMiddleware()` 执行后还跟有 `then` 及 `catch` 函数，说明在中间件执行过程中，任意一个中间发生错误，都会被最后这个 `catch` 捕获到

