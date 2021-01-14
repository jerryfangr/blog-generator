---
title: async与await
date: 2021-01-14 17:02:57
tags: ['学习笔记', 'web', 'javascript']
---



## 介绍

**如果说Promise的出现是为了解决异步又长又臭的回调问题，使之更加灵活。**

**那async与await的出现，就是为了让你感觉不到异步与同步书写的区别，是目前异步的终极解决方案**
<br>



## 使用

### 在正式使用前，先看一个Generator的示例

* **假设有这么一个异步函数，用于获取数据**

```javascript
function fetchData(number) {
    return new Promise((resolve, reject) => {
        number++;

        setTimeout(() => {
            resolve({
                number: number,
                data: 'data-' + number
            })
        }, 200);
    })
}
```

* **现在构建一个Generator用于获取第四轮的数据**

```javascript
function *genGet(startNumber) {
    console.log('1');
    
    let data = yield fetchData(startNumber);
    console.log('2');
    
    data = yield fetchData(data.number);
    console.log('3');
    
    data = yield fetchData(data.number);
    console.log('END fetch: ', data);
}
```

* **Generator执行**

```javascript
function runGen(genInstance, value) {
    let result = genInstance.next(value);
    
    if (!result.done) {
        setTimeout(() => {
            if (result.value instanceof Promise) {
                result.value.then(data => {
                    runGen(genInstance, data);
                })
            } else {
                runGen(genInstance, result.value);
            }
        }, 0);
    }
}

runGen(genGet(1));
```

* **执行结果**

```
1
2
3
END fetch:  {number: 4, data: "data-4"}
```

<br>

### 现在把上面的换成async与await

* **构建一个async用于获取第四轮的数据**

```javascript
function fetchData(number) {
    return new Promise((resolve, reject) => {
        number++;

        setTimeout(() => {
            resolve({
                number: number,
                data: 'data-' + number
            })
        }, 200);
    })
}

function async asyncGet(startNumber) {
    console.log('1');
    
    let data = await fetchData(startNumber);
    console.log('2');
    
    data = await fetchData(data.number);
    console.log('3');
    
    data = await fetchData(data.number);
    console.log('END fetch: ', data);
}

asyncGet(1);
```

* **执行结果**

```
1
2
3
END fetch:  {number: 4, data: "data-4"}
```

<br>

### 对比Generator 与 async

那么现在你是否意识到了双方的相似性，async可以理解为一个带自执行co的Generator 的语法糖。

<br>

### 正式创建一个基本的async

* **创建**

```javascript
async function asyncFun1() {
}


async function asyncFun2() {
    return 1;
}

async function asyncFun3() {
    return new Promise((resolve, reject) => {
        resolve(111)
    });
}

console.log(asyncFun1());
console.log(asyncFun2());
console.log(asyncFun3());
```

* **执行结果**

```javascript
Promise {<fulfilled>: undefined}
Promise {<fulfilled>: 1}
Promise {<pending>}
    [[PromiseState]]: "fulfilled"
    [[PromiseResult]]: 111
```

* **特性**

async的返回必然是一个promise。

如果返回值为非Promise类型，则返回一个resolve状态，数值为返回值的promise。

如果返回值为Promise类型，则返回该promise。

<br>

###  await获取数据(有且只能在async函数内使用)

* **创建**

```javascript
function syncFun() {
    return 1;
}

function asyncFun() {
    let data;
    setTimeout(() => {
        data = 2;
    }, 200);
    
    return data;
}

function asyncPromiseFun() {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(3);
        }, 200);
    })
}

function asyncPromiseFun2() {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            reject(new Error('code 4'));
        }, 200);
    })
}

async function asyncTest() {
    let data1 = await syncFun();
    let data2 = await asyncFun();
    let data3 = await asyncPromiseFun();
    
    console.log('data1: ', data1);
    console.log('data2: ', data2);
    console.log('data3: ', data3);
    try {
        let data4 = await asyncPromiseFun2();
    } catch(error) {
        console.log(error)
    }
}

asyncTest();
```

* **执行结果**

```javascript
data1:  1
data2:  undefined
data3:  3
Error: code 4
```

* **特性**

await 右边为非promise的数值，直接返回该数值。

await 右边为promise，直接返回该promise的resolve值或throw reject值。

<br>



## 结尾

异步的解决方案，从 回调 到 promise 到 async&await，已经趋于同步了。

显然如今async 与await这样的语法糖比像[node-fibers](https://github.com/laverdet/node-fibers)这样的要更直观，更符合书写逻辑了。

