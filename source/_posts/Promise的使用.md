---
title: Promise的使用
date: 2020-12-02 00:27:48
tags: ['javascript']
categories: [学习笔记, javascript]
---



# 开篇

## 介绍

Promise 的字面直译是承诺。
MDN上的解释是，Promise是一个对象，它代表了一个异步操作的最终完成或者失败。
从使用上来看，Promise是对异步回调的一种新的解决方案，让异步更加灵活，还可以摆脱令人僵硬的三角结构。



## 示例

### 没有Promise时的面对连续的异步

```js
asyncFunction1((data1) =>{
    asyncFunction2((data2) => {
        asyncFunction3((data3) => {
            asyncFunction4((data4) => {
                asyncFunction5((data5) => {
                    // 瞧瞧这优美的三角结构，这可是只有纯粹的异步才能独享的moment
                    console.log(data5);
                })
            })
        })
    })
})
```



### 有Promise时的面对连续的异步

```js
let p = new Promise((resolve) => {
    asyncFunction1((data1) => { resolve(data1); });
})
.then((data1) => {
    asyncFunction2((data2) => { return data2; });
})

// 中途随意出去做了点事
console.log('do something')

p.then((data2) => {
    asyncFunction3((data3) => { return data3; });
})
.then((data3) => {
    asyncFunction4((data4) => { return data4; });
})
.then((data4) => {
    asyncFunction5((data5) => { return data5; });
})
.then((data5) => {
    // 优美三角的陨落
    console.log(data5);
})
```

<br>

# 使用

## 创建

### new Promise(excutor: Function)

创建Promise时传入的函数，将在Promise的构造函数中同步执行。

```js
function doSomething() {
    // ....
}

console.log(1)
new Promise((resolve, reject) => {
    try {
       let result =  doSomething();
        console.log(2)
        resolve(result); // 完成
    } catch(error) {
        reject(error)
    }
});
console.log(3)
```

上面的执行结果为：

```bash
1
2
3
```

<br>



## 实例(原型)方法

### then(onFulfilled: Function, onRejected: Function) 

then接收两个函数，操作成功和失败的回调函数。
该回调函数会以异步的方式执行。

```js
console.log(1)
new Promise((resolve, reject) => {
    try {
        console.log(2)
        resolve(3); // 完成
    } catch(error) {
        reject(error)
    }
})
.then(data => {
    console.log(data)
})

console.log(4)
```

上面的执行结果为：

```bash
1
2
4
3
```

then实际会返回一个新的Promise实例，所以then可以无限链式下去。
在then的链式中，下一个then的调用由上一个then的结果决定。

```js
new Promise((resolve, reject) => {
	resolve(); 
})
.then(() => {
    throw new Error('第一个then的成功回调')
})
.then(() => {
    console.log('第二个then的成功回调')
}, () => {
    console.log('第二个then的失败回调')
})
.then(() => {
    console.log('第三个then的成功回调')
}, () => {
    console.log('第三个then的失败回调')
})
```

上面的执行结果为：

```bash
第二个then的失败回调
第三个then的成功回调
```

<br>

### catch(onRejected: Function) 

如果then的调用链中都没有指定onRejected函数，那么Promise会补一个onRejected继续传递错误的回调。
该默认onRejected错会调用下一个then的onRejected从而把错误下传，直至遇到自定义的onRejected。
catch实质上就是在最后接一个有onRejected的then。

```js
new Promise((resolve, reject) => {
	resolve(); 
})
.then(() => {
    throw new Error('第一个then的成功回调')
})
.then(() => {
    console.log('第二个then的成功回调')
})
.then(() => {
    console.log('第三个then的成功回调')
})
.catch((error) => {
    console.log(error);
})
.then(() => {
    console.log('第四个then的成功回调')
})
```

上面的执行结果为：

```bash
Error: 第一个then的成功回调
第四个then的成功回调
```

<br>



## 类方法

### all(promises: iterable) 

all 接收一个promise的iterable类型（Array，Map，Set都属于ES6的iterable类型），返回一个Promise。
all接收多个Promise实例，所有实例都成功执行，则执行成功回调，任意一个出错执行失败的回调。

```js
let p1 = new Promise((resolve) => {
	setTimeout(() => { 
        console.log(1)
        resolve();
    })
});
let p2 = new Promise((resolve) => {
	setTimeout(() => { 
        console.log(2)
        resolve();
    })
});

Promise.all([p1, p2])
.then(() => {
    console.log(3)
})
```

上面的执行结果为：

```bash
1
2
3
```

<br>

### race(promises: iterable) 类方法

race与all一样接收一个promise的iterable类型，返回一个Promise。
all接收多个Promise实例，任何一个先执行完成promise的回调决定 race的回调。

```js
let p1 = new Promise((resolve) => {
	setTimeout(() => { 
        console.log(1)
        resolve();
    }, 100)
});
let p2 = new Promise((resolve) => {
	setTimeout(() => { 
        console.log(2)
        resolve();
    }, 200)
});

Promise.race([p1, p2])
.then(() => {
    console.log(3)
})
```

上面的执行结果为：

```bash
1
3
2
```

