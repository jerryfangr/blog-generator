---
title: Promise原理解析(下)
date: 2021-02-14 18:41:23
tags: ['学习笔记', 'web', 'javascript']
---



## 开头

**[上篇](/2021/02/01/Promise原理解析(上)/)对promise的核心构造和then方法进行了实现**
**本篇解析实现promise常见的原型和类方法**

<br>



## 实现(默认采用[上篇](/2021/02/01/Promise原理解析(上)/)的代码结构)

### catch(原型方法)

* **特性**

  1. 获取链式中的错误，类似增加一个失败回调

* **结构**

  ```typescript  
  export default class XPromise {
      /**
       * * 获取链式中传递的错误
       * @param onRejected 错误触发函数
       * @returns promise
       */
      catch(onRejected: Function): XPromise {
          return this.then(undefined, onRejected);
      }
  }
  ```


### finally(原型方法)

* **特性**

  1. 无论上一步执行情况如何，都将执行finally的回调

* **结构**

  ```typescript  
  export default class XPromise {
      /**
       * * 论上一步是成功还是失败，都将触发finally回调
       * @param callback 触发的回调函数
       * @returns promise
       */
      finally(callback: Function): XPromise {
          return this.then(callback, callback);
      }
  }
  ```

### resolve(类方法)

* **特性**

  1. 接收一个参数，当参数类型不为promise，默认返回一个状态为fulfilled的promise，then的数值即为该参数
  2. 接收一个参数，当参数类型为promise，则状态将由该参数决定

* **结构**

  ```typescript
  export default class XPromise {
      /**
       * * 返回一个状态为fulfilled或其他的promise
       * @param value 数值
       * @returns promise
       */
      static resolve(value: any): XPromise {
          return new XPromise((resolve, reject) => {
              if (value instanceof XPromise) {
                  value.then(resolve, reject);
              } else {
                  resolve(value);
              }
          });
      }
  }
  ```

### reject(类方法)

* **特性**

  1. 返回一个状态为

* **结构**

  ```typescript
  export default class XPromise {
      /**
       * * 返回一个状态为rejected的promise
       * @param reason 数值
       * @returns promise
       */
      static resolve(reason: any): XPromise {
          return new XPromise((resolve, reject) => {
              reject(reason);
          });
      }
  }
  ```

### race(类方法)

* **特性**

  1. 接收一组promise，其中第一个执行成功或失败的promise决定该race的状态

* **结构**

  ```typescript
  export default class XPromise {
      /**
       * * 接收一组promise，其中第一个执行成功或失败的promise决定该race的状态
       * @param promises promise任务列表
       * @returns promise
       */
      static race(promises: XPromise[]): XPromise {
          return new XPromise((resolve, reject) => {
              promises.forEach(promise => {
                  promise.then(
                      value => { resolve(value); },
                      reason => { reject(reason); }
                  )
              });
          });
      }
  }
  ```

### all(类方法)

* **特性**

  1. 接收一组promise，所有promise执行成功则为成功状态，任意一个失败的都为失败

* **结构**

  ```typescript
  export default class XPromise {
      /**
       * * 当所有执行都成功, 才算成功,否则失败
       * @param promises promise任务列表
       * @returns promise
       */
      static all(promises: XPromise[]): XPromise {
          // 存放所有结果
          const values = new Array(promises.length);
          // 存放完成计数
          let resolvedCount = 0;
          return new XPromise((resolve, reject) => {
              promises.forEach((promise, index) => {
                  XPromise.resolve(promise).then(
                      value => {
                          values[index] = value;
                          resolvedCount++;
                          
                          if (resolvedCount === promises.length) {
                              resolve(values);
                          }
                      },
                      reason => {
                          reject(reason);
                      }
                  )
              });
          });
      }
  }
  ```

<br>



## 完整代码(typerscript)

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
    private _thenCallback: {
        [k: string]: Function
    };

    /**
     * * 构造方法
     * @param excutor 需要执行的异步或同步函数
     */
    constructor(excutor: (resolve: Function, reject: Function) => void) {
        // 初始化状态
        this._state = STATE.PENGDING;
        this._data = undefined;
        this._thenCallback = {};
        const self = this;

        // 修改状态，触发回调
        function changeStatus(state: STATE, data: any) {
            if (self._state === STATE.PENGDING) {
                self._state = state;
                self._data = data;

                if (state === STATE.FULFILLED) {
                    self._thenCallback.success?.(data);
                } else {
                    self._thenCallback.fail?.(data);
                }
            }
        }

        // 修改状态，触发成功的回调
        function resolve(data: any) {
            changeStatus(STATE.FULFILLED, data);
        }

        // 修改状态，触发失败的回调
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

    /**
     * * 获取链式中传递的错误
     * @param onRejected 错误触发函数
     * @returns promise
     */
    catch(onRejected: Function): XPromise {
        return this.then(undefined, onRejected);
    }

    /**
     * * 无论上一步是成功还是失败，都将触发finally回调
     * @param callback 触发函数
     * @returns promise
     */
    finally(callback: Function): XPromise {
        return this.then(callback, callback);
    }

    /**
     * * 返回一个状态为fulfilled或其他的promise
     * @param value 数值
     * @returns promise
     */
    static resolve(value: any): XPromise {
        // 返回一个成功或失败的 promise
        return new XPromise((resolve, reject) => {
            if (value instanceof XPromise) {
                value.then(resolve, reject);
            } else {
                resolve(value);
            }
        });
    }

    /**
     * XPromise函数对象的 reject 方法
     * @param {any} reason reject的值
     * @return {XPromise} 返回指定reason的XPromise, 状态为失败
     */
    static reject(reason: any): XPromise {
        // 返回一个失败的 XPromise
        return new XPromise((resolve, reject) => {
            reject(reason);
        });
    }

    /**
     * * 接收一组promise，其中第一个执行成功或失败的promise决定该race的状态
     * @param promises promise任务列表
     * @returns promise
     */
    static race(promises: XPromise[]): XPromise {
        return new XPromise((resolve, reject) => {
            promises.forEach(promise => {
                promise.then(
                    value => { resolve(value); },
                    reason => { reject(reason); }
                )
            });
        });
    }

    /**
     * 当所有执行都成功, 才算成功,否则失败
     * @param {Array} xpromises XPromise数组
     * @return {XPromise} 返回一个XPromise
     */
    static all(promises: XPromise[]): XPromise {
        // 存放所有结果
        const values = new Array(promises.length);
        // 存放完成计数
        let resolvedCount = 0;
        return new XPromise((resolve, reject) => {
            promises.forEach((promise, index) => {
                XPromise.resolve(promise).then(
                    value => {
                        values[index] = value;
                        resolvedCount++;

                        if (resolvedCount === promises.length) {
                            resolve(values);
                        }
                    },
                    reason => {
                        reject(reason);
                    }
                )
            });
        });
    }
}
```



## 结尾

本篇对Promise的一些常见方法进行了实现。

可以看出，Promsie的构造和then方法才是核心，其余都可以通过这两种方式去轻松实现