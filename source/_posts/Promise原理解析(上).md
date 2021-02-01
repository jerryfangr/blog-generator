---
title: Promise原理解析(上)
date: 2021-02-01 20:23:47
tags: ['学习笔记', 'web', 'javascript']
---



## 开头

**Promise作为ES6标准中提出的一种新的异步解决方案，给前端带来了深远的影响。**
**如果你对浏览器的宏微任务有了解，那么应该知道Promise.then会产生微任务。**
**本篇尝试解析还原一个Promise的构造和then方法**

<br>



## 实现(为避免重名以下用XPromise代替Promise)

### 构造方法

* **特性**
  1. 三大状态 pending、fulfilled、rejected
  2. 一个实例有且只能更改一次状态
  3. 接收一个执行函数作为参数
  4. 该执行函数同步执行
  5. 如执行函数异步执行，且在then执行之后才修改状态，则应提前记录then的回调

* **结构**

  ```typescript
  enum STATE {
      PENGDING,
      FULFILLED,
      REJECTED,
  }
  
  export default class XPromise {
      // 状态
      private _state: STATE;
      // 数据
      private _data: any;
      // then的回调
      private _thenCallback?: Function;
  
      /**
       * * 构造方法
     * @param excutor 需要执行的异步或同步函数
       */
      constructor(excutor: (resolve: Function, reject: Function) => void) {
          // 初始化状态
          this._state = STATE.PENGDING;
      	this._data = undefined;
          this._thenCallback = undefined;
          const self = this;
  
          // 修改状态，触发回调
          function changeStatus(state: STATE, data: any) {
              if (self._state === STATE.PENGDING) {
                  self._state = state;
                  self._data = data;
                  this.__thenCallback?.(data);
  
                  let logMessage = state === STATE.FULFILLED ? 'resolve data: ' : 'reject reason: ';
                  console.log(logMessage, data);
              }
          }
  
          // 修改为成功状态，触发成功的回调
          function resolve(data: any) {
              changeStatus(STATE.FULFILLED, data);
          }
  
          // 修改为失败状态，触发失败的回调
          function reject(reason: any) {
              changeStatus(STATE.REJECTED, reason);
          }
  
          // 执行函数
          try {
              excutor(resolve, reject);
          } catch (error) {
              reject(error)
          }
      }
  }
  ```

* **测试**

  ```typescript
  // 测试resolve回调，预期下面顺序执行
  console.log('------ test1 ------');
  new XPromise((resolve, reject) => {
      console.log('t1 excutor start');
      resolve('data');
      console.log('t1 excutor end');
  });
  
  // 测试状态只能更改一次，预期下面resolve不会执行
  console.log('------ test2 ------');
  new XPromise((resolve, reject) => {
      console.log('t2 excutor start');
      reject(new Error('T2 error').message);
      resolve('data');
      console.log('t2 excutor end');
  })
  
  // 测试excutor出错，预期reject输出error
  console.log('------ test3 ------');
  new XPromise((resolve, reject) => {
      console.log('t3 excutor start');
      throw new Error('T3 error');
      resolve('data');
      console.log('t3 excutor end');
  })
  
  // 测试异步回调，预期resolve 1秒后执行
  console.log('------ test4 ------');
  new XPromise((resolve, reject) => {
      console.log('t4 excutor start');
      setTimeout(() => {
          resolve('data');
      }, 1000);
      console.log('t4 excutor end');
  })
  ```

* **测试结果**

  ```bash
  ------ test1 ------
  t1 excutor start
  resolve data:  data
  t1 excutor end
  ------ test2 ------
  t2 excutor start
  reject reason:  T2 error
  t2 excutor end
  ------ test3 ------
  t3 excutor start
  reject reason:  Error: T3 error
  ------ test4 ------
  t4 excutor start
  t4 excutor end
  resolve data:  data
  ```

<br>

### then

* **特性**

  1. then返回一个新的promise实例。
  2. then中传入的回调函数都会异步执行。
  3. then接收两个参数: 成功与失败的回调函数，具体执行哪个，取决于上一步的执行结果。
  4. then可以无限链式，并且后一个than运行的回调由上一个than的回调运行是否成功决定。
  5. then的回调返回如果一个promise实例时，下一个than的执行将由该实例决定。
  6. then可以不传入任何参数，默认向下传递数值。
  7. then中的reject或运行出错将一直向后面的链式传递，直至有有对失败处理的回调。

* **结构**

  ```typescript
  export default class XPromise {
      // ...
      constructor() {
          // ...
      }
  
      /**
       * * then用于定义执行成功与失败的回调
       * @param onResolved 执行成功的回调
       * @param onRejected 执行失败的回调
       * @returns promise
       */
      then(onResolved?: Function, onRejected?: Function): XPromise {
          // 默认数据传递
          onResolved = typeof onResolved === 'function' ? onResolved : value => value;
          // 默认错误传递
          onRejected = typeof onRejected === 'function' ? onRejected : reason => { throw reason };
  
          const self = this;
  
          // 返回一个新的promise
          return new XPromise((resolve, reject) => {
  
              /**
               * * then 处理回调执行的结果，决定链式下一个then的执行
               * * 执行成功，且返回值不为promise，链式下一个then执行成功回调
               * * 执行成功，且返回值为promise，该promise决定下一个then的执行
               * * 执行出错，链式下一个then执行失败回调
               * @param excutor 成功或失败的触发函数
               */
              function handle(excutor: Function): void {
                  try {
                      let value = excutor(self._data);
                      if (value instanceof XPromise) {
                          value.then(resolve, reject);
                      } else {
                          resolve(value)
                      }
                  } catch (error) {
                      reject(error)
                  }
              }
  
              // 上一步的执行函数未执行完毕，记录要触发的回调
              if (self._state === STATE.PENGDING) {
                  this._thenCallback = {
                      success: () => { handle(onResolved) },
                      fail: () => { handle(onRejected) },
                  }
                  // 上一步的执行函数已执行，且为完成状态，异步执行触发函数
              } else if (self._state === STATE.FULFILLED) {
                  setTimeout(() => { handle(onResolved)}, 0);
                  // 上一步的执行函数错误或为异常状态，异步执行触发函数
              } else {
                  setTimeout(() => { handle(onRejected) }, 0);
              }
          })
      }
  }
  ```

* **测试**

  * then返回一个新的promise实例。

    ```typescript
    console.log('------- test 1 start -------');

    let t1 = new XPromise(() =>{});
    let t2 = t1.then();
    console.log('is promise: ', t2 instanceof XPromise, '\nis same promise: ', t1 === t2);

    console.log('------- test 1 end -------');
    ```

    ```bash
    ------- test 1 start -------  
    is promise:  true 
    is same promise:  false
    ------- test 1 end -------
    ```

  * then中传入的回调函数都会异步执行。

    ```typescript
    console.log('------- test 2 start -------');

    new XPromise((resolve, reject) => {
        resolve();
    }).then(data => {
        console.log('then resolved');
    })

    console.log('------- test 2 end -------');
    ```

    ```bash
    ------- test 2 start -------
    ------- test 2 end -------
    then resolved
    ```

  * then接收两个参数: 成功与失败的回调函数，具体执行哪个，取决于上一步的执行结果。

    ```typescript
    console.log('------- test 3-1 start -------');
    new XPromise((resolve) => {
        resolve();
    }).then(data => {
        console.log('then resolve');
        console.log('------- test 3-1 end -------');
    }, error => {
        console.log('then rejected');
    })
    
    setTimeout(() => {
        console.log('------- test 3-2 start -------');
        new XPromise((resolve, reject) => {
            reject();
        }).then(data => {
            console.log('then resolve');
        }, error => {
            console.log('then rejected');
            console.log('------- test 3-2 end -------');
        })
    }, 10);
    ```

    ```bash
    ------- test 3-1 start -------
    then resolve
    ------- test 3-1 end -------
    ------- test 3-2 start -------
    then rejected
    ------- test 3-2 end -------
    ```
  
  * then可以无限链式，并且后一个than运行的回调由上一个than的回调运行是否成功决定。
  
    ```typescript
    console.log('------- test 4 start -------');
    new XPromise((resolve) => {
        resolve();
    }).then(() => {
        console.log('then 1 resolved');
        throw new Error('T4 Error');
    }, error => {
        console.log('then 1 rejected');
    }).then(() => {
        console.log('then 2 resolved');
    }, error => {
        console.log('then 2 rejected');
    }).then(() => {
        console.log('then 3 resolved');
        console.log('------- test 4 end -------');
    }, error => {
        console.log('then 3 rejected');
    })
    ```

    ```bash
    ------- test 4 start -------
    then 1 resolved
    then 2 rejected
    then 3 resolved
    ------- test 4 end -------
    ```

  * then的回调返回一个promise实例时，下一个than的执行将由该实例决定。
  
    ```typescript
    console.log('------- test 5 start -------');
    new XPromise((resolve) => {
        resolve();
    }).then(() => {
        return new XPromise((resolve, reject) => {
            setTimeout(() => {
                reject()
            }, 2000);
        })
    }).then(undefined, error => {
        console.log('then 2 reject');
        console.log('------- test 5 end -------');
    })
    ```
    
    ```bash
    ------- test 5 start -------
    then 2 reject
    ------- test 5 end -------
    ```
  
  * then可以不传入任何参数，默认向下传递数值。
  
    ```typescript
    console.log('------- test 6 start -------');
    new XPromise((resolve) => {
        resolve('|data|');
    })
    .then()
    .then()
    .then(data => {
        console.log('last then data: ', data);
        console.log('------- test 6 end -------');
    })
    ```
    
    ```bash
    ------- test 6 start -------
    last then data:  |data|
    ------- test 6 end -------
    ```
  
  * then中的reject或运行出错将一直向后面的链式传递错误，直至自定义了失败的回调。
  
    ```typescript
    console.log('------- test 7 start -------');
    new XPromise((resolve) => {
        resolve('|data|');
    }).then(() => {
        throw new Error('T7 Error')
    }).then().then(data => {
        console.log('resolve data: ', data);
    }, error => {
        console.log('reject error', error);
        console.log('------- test 7 end -------');
    })
    ```
  
    ```bash
    ------- test 7 start -------
    reject error Error: T7 Error
    ------- test 7 end -------
    ```

<br>



## 结尾

本篇主要实现了promise相对核心的构造和then方法的简易版。

从中可以看出，promise对异步的处理本质上还是连续多个回调。