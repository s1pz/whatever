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
         });
       }
     })(callBack);
   }

```
只提供了`then`，`catch`，`finally`三个`api`， 保持和`promise`一样的使用方式。
