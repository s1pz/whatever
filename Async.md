##### 本文将参考[ES6文档](https://es6.ruanyifeng.com/#docs/async)一步一步实现一个`async 函数`。

首先、我们看一段关于`async`调用的代码

```
async function task() {
    let resultOne = await new Promise((resolve) => {
      setTimeout(resolve.bind(null, 1), 2000);
    })
    let resultTwo = await new Promise((resolve) => {
      setTimeout(resolve.bind(null, 2), 2000);
    })
    return resultOne + resultTwo;
  }
  const now = Date.now();
  task().then(res => {
    console.warn(`res = ${res}; duration = ${Date.now() - now}`);
  });

```

输出结果是`res = 3`, `duration 约等于 4000`，也就是说后面的`task`要等前面的`task`执行完毕才能开始运行。 而我们知道一个`function`一旦调用了、代码就会依次往下执行、如何实现等待的效果？

如果我们能将函数拆开，控制一段一段、甚至一行一行的执行就可以实现了，`Generator 函数`就是用来解决这个问题的。

---

###### `Generator 函数`简介

`Generator 函数`是`协程`在 `ES6` 的实现，最大特点就是可以交出`函数的执行权（即暂停执行）`。

整个 `Generator 函数`就是一个封装的异步任务，或者说是异步任务的容器。异步操作需要暂停的地方，都用`yield`语句注明。`Generator 函数`的使用方式可以参考代码：

```
function* helloWorldGenerator() {
  yield 'hello';
  yield 'world';
  return 'ending';
}

var hw = helloWorldGenerator();
hw.next();
// { value: 'hello', done: false }

hw.next();
// { value: 'world', done: false }

hw.next();
// { value: 'ending', done: true }

hw.next();
// { value: undefined, done: true }

```

形式上，`Generator 函数`是一个普通函数，但是有两个特征。一是，`function`关键字与函数名之间有一个`星号`；二是，函数体内部使用`yield`表达式，定义不同的内部状态（`yield`在英语里的意思就是“产出”）。这两个特征就对应的`async`和`await`。

调用 `Generator 函数`后，该函数并不执行、而且返回一个`遍历器`、然后必须调用`遍历器`对象的`next`方法，使得指针移向下一个状态。也就是说，每次调用`next`方法，内部指针就从函数头部或上一次停下来的地方开始执行，直到遇到下一个`yield`表达式（或`return`语句）为止。换言之，Generator 函数是分段执行的，`yield`表达式是暂停执行的标记，而`next`方法可以恢复执行。

---

##### 开始实现

有了 `Generator 函数`函数后、我们就可以尝试实现一个`async`了。首先第一步就是去掉`async`关键字，封装一个内部`Generator 函数`，将`async函数`内部的代码全部复制过来扔进`Generator 函数`、但是我们需要把`await`改成成`yield`、最后返回一个`Promise`。

就拿上面的`task函数`来进行举例：

```
function task() {
  function *innerTask() {
    // async函数方法体拷贝过来、把await改成 yield
    let resultOne = yield new Promise((resolve) => {
      setTimeout(resolve.bind(null, 1), 2000);
    })
    let resultTwo = yield new Promise((resolve) => {
      setTimeout(resolve.bind(null, 2), 2000);
    })
    return resultOne + resultTwo;
  }
  // 返回一个Promise
  return new Promise((resolve, reject) => {
  })
}

```

我们已经成功去掉了`async` 和`await`，剩下就是利用`Generator 函数`的`next`方法去按顺序执行任务就可以了。

```
function task() {
   function *innerTask() {
      // async函数方法体拷贝过来、把await改成 yield
      let resultOne = yield new Promise((resolve) => {
        setTimeout(resolve.bind(null, 1), 2000);
      })
      let resultTwo = yield new Promise((resolve) => {
        setTimeout(resolve.bind(null, 2), 2000);
      })
      return resultOne + resultTwo;
    }
   // 返回一个Promise
   return new Promise((resolve, reject) => {
      let g = innerTask();
      function run(next) {
        let result;
        try {
          result = next();
        } catch(e) {
          return reject(e);
        }
        if (!result.done) {
          run(() => g.next(result.value));
        } else {
          resolve(result.value);
        }
      }
      run(() => g.next(undefined));
    })
  }

  const now = Date.now();
  task().then(res => {
    console.warn(`res = ${res}; duration = ${Date.now() - now}`);
  });

```

本质上是一个递归依次调度任务执行，但是输出结果`res = [object Promise][object Promise]; duration = 1`，不符合预期。问题出在哪里了呢？

是因为我们没有等第一个任务结束了就开始了第二个任务、那么如何实现任务的挂起排队呢，很简单、只要把第二个任务注册到第一个任务的`then`方法里就可以了，所以我们只需要稍微调整下代码即可：

```
function task() {
   function *innerTask() {
      // async函数方法体拷贝过来、把await改成 yield
      let resultOne = yield new Promise((resolve) => {
        setTimeout(resolve.bind(null, 1), 2000);
      })
      let resultTwo = yield new Promise((resolve) => {
        setTimeout(resolve.bind(null, 2), 2000);
      })
      return resultOne + resultTwo;
    }
   // 返回一个Promise
   return new Promise((resolve, reject) => {
      let g = innerTask();
      function run(next) {
        let result;
        try {
          result = next();
        } catch(e) {
          return reject(e);
        }
        if (!result.done) {
          Promise.resolve(result.value).then((res) => {run(() => g.next(res))}, (e) => g.throw(e));
        } else {
          resolve(result.value);
        }
      }
      run(() => g.next(undefined));
    })
  }

  const now = Date.now();
  task().then(res => {
    console.warn(`res = ${res}; duration = ${Date.now() - now}`);
  });

```

这样一个自定义`async`函数就基本成型了。
上述代码调整的原理是利用了`promise.reslove`的特性，如果参数是 Promise 实例，那么`Promise.resolve`将不做任何修改、原封不动地返回这个实例， 如果参数是一个原始值，或者是一个不具有`then()`方法的对象，则`Promise.resolve()`方法返回一个新的 Promise 对象，状态为`resolved`。

---

##### 最后一点小猜想

为什么用了`await`后`try-catch`就能捕获`promise`内部的错误了？

本质原因是`Generator 函数`返回的遍历器对象，都有一个`throw`方法，可以在函数体外抛出错误，然后在` Generator 函数`体内捕获。

```
function *gen() {
    try {
      yield 1;
      yield 2;
    } catch(e) {
      console.log('catch error = ' + e);
    }
 }
 var g = gen();
 g.next();
 // g.throw('mistake!');
 g.next();
 g.throw('mistake!');

```

在`Generator 函数`执行一次`next`之后，直至状态变为`done`之前，我们在任何地方调用`g.throw`都会被内部的`try-catch`捕获。

结合这个特性，我们只需要在每个任务后面加上一个`catch`然后调用`g.throw`就可以触发外侧的`try-catch`了。这里补充一句：`then(func1, func2)` 本质上等价于`then(func1).catch(func2)`，后者是前者的语法糖。

最后总结一下，`async`，`await`本质上就是`Generator 函数`和`yield`的语法糖，以后看见`async`，`await`直接用`Generator `和`yield`替换即可。
