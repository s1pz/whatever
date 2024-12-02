##### 本文参考`jQuery`源码和[Promises/A+规范](https://promisesaplus.com/)实现一个的`promise`。

先贴上源码：

```
function myPromise(callBack) {
     //立即执行函数、防止外界访问变量
     return (function (callBack) {
       let state = "pending";
       let resolveQueue = [];
       let rejectQueue = [];
       let value = undefined;
       let isAbort = false; // 尝试注销提示报错任务
       try {
         callBack(resolve, reject);
       } catch (e) {
         if (state === "pending") {
           reject(e);
         }
       }
       return {
         then,
         catch(cb) {
           return then(undefined, cb);
         },
         finally(cb) {
           return then(
             (res) => {
               cb();
               return res;
             },
             (ex) => {
               cb();
               throw ex;
             }
           );
         },
       };
       function exec(target) {
         while (target.length) {
           let cb = target.shift();
           cb(value);
         }
       }
       function addSuccess(cb) {
         resolveQueue.push(cb);
         if (state === "fulfilled") {
           exec(resolveQueue);
         }
       }
       function addFail(cb) {
         rejectQueue.push(cb);
         if (state === "rejected") {
           exec(rejectQueue);
         }
       }
       function resolve(val) {
         if (state !== "pending") return;
         state = "fulfilled";
         value = val;
         exec(resolveQueue);
       }
       function reject(e) {
         if (state !== "pending") return;
         state = "rejected";
         value = e;
         exec(rejectQueue);
         setTimeout(() => {
           if (!isAbort) {
             console.error('Uncaught (in promise) ' + e);
           }
         }, 0)
       }
       function then(
         onFulfilled = (x) => x,
         onRejected = (ex) => {
           throw ex;
         }
       ) {
         return new myPromise((resolve, reject) => {
           function success(val) {
             setTimeout(() => {
               if (typeof val?.then === "function") {
                 val.then(success);
               } else {
                 try {
                   resolve(onFulfilled(val));
                 } catch (e) {
                   reject(e);
                 }
               }
             }, 0);
           }
           function fail(e) {
             try {
               let res = onRejected(e);
               if (typeof res?.then === "function") {
                 res.then(success);
               } else {
                 resolve(res);
               }
             } catch (e) {
               reject(e);
             }
           }
           addSuccess(success);
           addFail(fail);
           isAbort = true;
         });
       }
     })(callBack);
   }

```

只提供了`then`，`catch`，`finally`三个`api`， 保持和`promise`一样的使用方式。

##### 介绍下思路

`promise` 本质上也是一个语法糖、主要实现思路是首先、要实现链式调用，也就是`return`值必须是一个新的`promise`，将新的`promise`的状态改变方法注入到`父promise`中。然后、再就是`fulfilled`和`rejected`状态的传递。最后再就是错误的捕获，`promise`内部必须要捕获所有的错误、然后传递错误、如果没有后代接收错误、就`console.error`错误。

整体代码也就`100`行、稍微看看就懂了
