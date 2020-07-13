this 的指向是在函数被调用的时候确定的，也就是执行上下文被创建的时候确定的

# this 绑定规则

## 默认绑定

**this 指向全局变量，严格模式下绑定到 undefined**

* 非严格模式下

由于 foo 直接使用没有带任何修饰的函数引用进行调用，因此使用的是默认绑定，this 指向全局变量

```js
function foo() { 
    console.log( this.a ); 
} 

var a = 2;
foo();    // 2
```

* 严格模式

严格模式下不能将全局对象用于默认绑定，this 会被绑定到 undefined

```js
function foo() {
    "use strict";

    console.log(this.a);
}

var a = 2;
foo();     // TypeError: this is undefined
```

**注意：在严格模式下运行函数 foo(), 是不会影响到默认绑定的**

```js
function foo() {
    console.log(this.a);
}

var a = 2;
(function() {
    "use strict";
    foo();    // 2
})()
```

## 隐形绑定

当函数引用存在上下文对象时，**隐式绑定会把函数调用中的 this 绑定到这个上下文对象中**

```js
function foo() {
    console.log(this.a);
}

var a = 10;

var obj2 = {
    a: 2,
    b: this.a,
    foo: foo
};

obj2.b;         // 10，因为单独的 {} 不会形成新的作用域，没有作用域的限制，这里的 this.a 仍然处于全局作用域之中，所以此时的 this 指向 window
obj2.foo();     // 2，foo 不是独立调用，而是被对象 obj2 所拥有，因此他的 this 指向了 obj2
```

**注意：对象属性引用链中只有上一层或者说最后一层在调用位置中起作用**

```js
function foo() {
    console.log(this.a);
}

var obj2 = {
    a: 42,
    foo: foo
}

var obj1 = {
    a: 2,
    obj2: obj2
}

obj1.obj2.foo();    // 2
```

### 隐式丢失

**被隐式绑定的函数丢失绑定对象，也就是将 this 默认绑定到了全局对象或者 undefined 上面**

下例中，**bar 实际上，引用的是 foo 函数本身**，是独立调用，所以 bar() 实际是一个不带任何修饰的函数调用，因此应用了默认绑定

```js
function foo() {
    console.log(this.a);
}

var obj = {
    a: 2,
    foo: foo
}

var bar = obj.foo;
var a = "oops, global"; // a 是全局对象的属性
bar();       // "oops, global"
```

**参数传递其实也是一种隐式赋值**，相当于将 foo 函数直接传入了 doFoo 中，因此结果和上个例子一样

```js
function foo() {
    console.log(this.a);
}

function doFoo(fn) {
    fn();
}


var obj = {
    a: 2,
    foo: foo
};

var a = "oops, global"; // a 是全局对象的属性
doFoo(obj.foo);
```

还有些**内置函数**，例如 setTimeout，它的实现原理类似于以下伪代码，**也是隐式赋值**

```js
function setTimeout(fn, delay) {
    // 等待 delay 
    fn();
}
```

所以 setTimeout 上的 this 也会隐形丢失

```js
function foo() {
    console.log(this.a);
}

var obj = {
    a: 2,
    foo: foo
};

var a = 'oops, global';
setTimeout(obj.foo, 100);    // oops, global
```

还有些 js 库事件处理器，会将回调函数的 this 绑定到 DOM 元素上


## 显示绑定

### 硬绑定

下例中，将 bar 函数中的 foo 的 this 强制绑定到了 obj 上，所有之后无论怎么调用 bar，它总会在 obj 的基础上调用 foo

```js
function foo() {
    console.log(this.a);
}

var obj = {
    a: 2
};

var bar = function() {
    foo.call(obj);
};

bar();   // 2
setTimeout(bar, 100);    // 2

bar.call(window);     // 2
```

### API 调用的上下文

某些回调函数使用指定的 this

```js
function fool(el) {
    console.log(el, this.id);
}

var obj = {
    id: 'awesome'
};

[1, 2, 3].forEach(foo, obj);
// 1 awesome 2 awesome 3 awesome
```

## new 绑定

```js
function Foo() {
    this.a = 2;
}

var foo = new Foo();
foo.a;   // 2
```

## 判断 this（优先级从上到下）

* 函数是否在 new 中调用（new 绑定）？如果是的话 this 绑定的是新创建的对象。

`var bar = new foo();`

* 函数是否通过 call、apply（显式绑定）或者硬绑定调用？如果是的话，this 绑定的是指定的对象。

`var bar = foo.call(obj2)`

* 函数是否在某个上下文对象中调用（隐式绑定）？如果是的话，this 绑定的是那个上下文对象。

`var bar = obj1.foo()`

* 如果都不是的话，使用默认绑定。如果在严格模式下，就绑定到 undefined，否则绑定到全局对象。

`var bar = foo()`

## 箭头函数

箭头函数不使用 this 的四种标准，而是根据外层（函数或者全局）作用于来决定 this

下例中，箭头函数的 this 取决于外层函数 foo 的 this

foo() 内部创建的箭头函数会捕获调用时 foo() 的 this，由于 foo() 的 this 绑定到 obj1，bar 引用箭头函数的 this 也会绑定到 obj1，箭头函数的绑定无法被修改（new 也不行）

```js
function foo() {
    return (a) => {
        console.log(this.a);
    }
}

var obj1 = {
    a: 2
};

var obj2 = {
    a: 3
};

var bar = foo.call(obj1);
bar.call(obj2);    // 2
```

```js
function foo() { 
    setTimeout(() => { 
        // 这里的 this 在词法上继承自 foo() 
        console.log( this.a ); 
    },100); 
} 
var obj = { 
    a: 2 
}; 
foo.call( obj ); // 2
```

## 其他

* 把 null 或者 undefined 作为 this 的绑定对象传入 call、apply 或者 bind

这些值会被忽略，实际应用的是默认绑定

```js
function foo() {
    console.log(this.a);
}

var a = 2;
foo.call(null);    // 2
```

使用 bind 预设参数

```js
function foo(a, b) {
    console.log(a, b);
}

var bar = foo.bind(null, 1);
bar(2);    // 1, 2
```

* 创建一个函数，间接引用，这种情况下，调用这个函数会默认绑定

```js
function foo() {
    console.log(this.a);
}

var a = 2;
var o = {a: 3, foo: foo};
var p = {a: 4};

o.foo();   // 3
(p.foo() = o.foo());    // 2
```

赋值表达式 `p.foo() = o.foo()` 的返回值是目标函数的引用，因此调用位置是 foo() 而不是 p.foo() 或者 o.foo()，所以采用的就是默认绑定

* 软绑定

默认绑定一个全局对象和 undefined 以外的值，即可以实现硬绑定，同时保留隐式绑定或者显示绑定修改 this 的能力

```js
if (!Function.prototype.softBind) { 
    Function.prototype.softBind = function(obj) { 
        var fn = this; 
        // 捕获所有 curried 参数
        var curried = [].slice.call( arguments, 1 ); 
        var bound = function() { 
        return fn.apply( 
            (!this || this === (window || global)) ? 
            obj : this,
            curried.concat.apply( curried, arguments ) 
            ); 
        }; 
        bound.prototype = Object.create( fn.prototype ); 
        return bound; 
    }; 
}
```

先检查调用时的 this，如果 this 绑定到全局变量或者 undefined，那就把默认对象 obj 绑定到 this，否则不会修改 this。