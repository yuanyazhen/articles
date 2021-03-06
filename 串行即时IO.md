在有些场景下，一个handle下可能会有大量的即时消息，如果每个即时消息会产生一个异步io，如果需要保证io的顺序，那么如何将这些io串行化？如下。

```js
// 获得一个可以串行io的闭包
function promiseIO () {
   let promise = null;
   return function (io) {
       if(!promise) {
           promise = io();
           return promise;
       } else {
           let prevPromise = promise;
           promise = (async function () { 
           // 将旧的promise与新的promise compose成一个新的promise
               await prevPromise;
               return io();
           })()
       }
       return promise;
   }
}

// 测试用io生成器
function ioFaker (ms) {
    return _ => new Promise (res => setTimeout(_ => {res(ms);console.log(ms + 'io')}, ms));
}

// 获得几个测试io
let _1sIO = ioFaker(1000);
let _2sIO = ioFaker(2000);
let _3sIO = ioFaker(3000);
let _4sIO = ioFaker(4000);
let IOs = [_4sIO, _3sIO, _2sIO, _1sIO];
let newPromiseIO = promiseIO();

// 不定期的handle请求，并且对于每个请求形成的异步操作按顺序执行
async function IOHandler (io) {
    await newPromiseIO(io);
}

// 不定期串行handle
for (let i = 0 ; i < 4 ; i ++) {
    IOHandler (IOs[i])
}
```