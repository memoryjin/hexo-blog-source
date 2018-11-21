---
title: 简易版Promise的实现
---

##### 从一道题开始

```javascript
const wait = time => new Promise(resolve => setTimeout(resolve, time))

wait(200)
  .then(() => new Promise(res => res('foo')))
  .then(a => a)
  .then(b => console.log(b))
  .then(() => null)
  .then(c => console.log(c))
  .then(() => {throw new Error('foo');})
  .then(
    d => console.log(`d: ${ d }`),
    e => console.log(e))
  .then(f => console.log(`f: ${ f }`))
  .catch(e => console.log(e))
  .then(() => { throw new Error('bar'); })
  .then(g => console.log(`g: ${ g }`))
```

大家说说看，控制台上将输出什么？



正确答案是：**foo -> null -> Error: full -> undefined -> Uncaught (in promise) Error: bar**



好了，接下来步入正题，看看Promise的实现机制吧。

##### Promise/A+规范

在开始实现之前，大家可以先查阅下[Promise/A+的规范](https://github.com/promises-aplus/promises-spec)。

其中，几个重要的术语要注意下：

1. "promise" is an object or function with a `then` method whose behavior conforms to this specification.
2. "thenable" is an object or function that defines a `then` method.
3. "value" is any legal JavaScript value (including `undefined`, a thenable, or a promise).
4. "exception" is a value that is thrown using the `throw` statement.
5. "reason" is a value that indicates why a promise was rejected.

除此之外，对于`Promise`中的then方法，规范有非常详细的要求限制，大家可以仔细阅读。这里就不罗列那好大一堆的东西了，直接开始撸代码吧！

##### 简易版Promise的实现

```javascript
const PENDING = 'pending'
const REJECTED = 'rejected'
const RESOLVED  = 'resolved'

class Promise {
  constructor(executor) {
    this.value = undefined
    this.status = PENDING
    this.onResolvedQueue = []
    this.onRejectedQueue = []

    const resolve = (value) => {
      // 注册在then里头的回调要异步执行，这里用setTimeout模拟异步
      if (this.status === PENDING) {
        setTimeout(() => {
          this.status = RESOLVED
          this.value = value
          this.onResolvedQueue.forEach(cb => cb(value))
        })
      }
    }

    const reject = (err) => {
      // 注册在then里头的回调要异步执行，这里用setTimeout模拟异步
      if (this.status === PENDING) {
        setTimeout(() => {
          this.status = REJECTED
          this.value = err
          this.onRejectedQueue.forEach(cb => cb(err))
        })
      }
    }

    try {
      executor(resolve, reject)
    } catch (err) {
      reject(err)
    }
  }

  static resolve (value) {
    return new Promise((resolve) => {
      resolve(value)
    })
  }

  static reject (err) {
    return new Promise((resolve, reject) => {
      reject(err)
    })
  }

  then (onResolved, onRejected) {
    onResolved = typeof onResolved === 'function' ? onResolved : value => value
    onRejected = typeof onRejected === 'function' ? onRejected : err => { throw err }

    if (this.status === PENDING) {
      return new Promise((resolve, reject) => {
        this.onResolvedQueue.push(value => {
          try {
            const returnedValue = onResolved(value)
            if (returnedValue instanceof Promise) {
              returnedValue.then(resolve, reject)
            } else {
              resolve(returnedValue)
            }
          } catch (err) {
            reject(err)
          }
        })
        this.onRejectedQueue.push(err => {
          try {
            const returnedValue = onRejected(err)
            if (returnedValue instanceof Promise) {
              returnedValue.then(resolve, reject)
            } else {
              resolve(returnedValue)
            }
          } catch (err) {
            reject(err)
          }
        })
      })
    }

    if (this.status === RESOLVED) {
      return new Promise((resolve, reject) => {
        // 注册在then里头的回调要异步执行，这里用setTimeout模拟异步
        setTimeout(() => {
          try {
            const returnedValue = onResolved(this.value)
            if (returnedValue instanceof Promise) {
              returnedValue.then(resolve, reject)
            } else {
              resolve(returnedValue)
            }
          } catch (err) {
            reject(err)
          }
        })
      })
    }

    if (this.status === REJECTED) {
      // 注册在then里头的回调要异步执行，这里用setTimeout模拟异步
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          try {
            const returnedValue = onRejected(this.value)
            if (returnedValue instanceof Promise) {
              returnedValue.then(resolve, reject)
            } else {
              resolve(returnedValue)
            }
          } catch (err) {
            reject(err)
          }
        })
      })
    }
  }

  catch (onRejected) {
    return this.then(null, onRejected)
  }
}
```

如上，一个简易版的`Promise`就实现了，但如果大家直接拿上面的代码去跑[Promise的测试用例](https://github.com/promises-aplus/promises-tests)的话，会很尴尬的发现……其实跑不通o(╯□╰)o。这主要是因为官方考虑到当前的第三库或多或少都有自己的Promise实现，为了让这些不同实现的Promise可以链式调用，官方做了一系列的假设和约束。感兴趣的同学可以去仔细阅读[这部分内容](https://github.com/promises-aplus/promises-spec#the-promise-resolution-procedure)。从学习`Promise`的角度出发，上面的实现个人感觉已经足够了，倘若把`[[Resolve]](promise, x)`这块内容也实现，从原理的可读性上看反而变差了。