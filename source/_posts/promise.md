---
title: 手写一个promise
---

#### 从做题开始

```javascript
const foo = () => (new Promise((resolve, reject) => {
  const p = new Promise((resolve, reject) => {
    console.log(0);
    setTimeout(() => {
      console.log(1);
      resolve(2);
    }, 0)
    resolve(3);
  });

  resolve(4);

  p.then((value) => {
    console.log(value);
    // throw new Error('something wrong')
  });

  return p
}));

foo()
  .then(() => new Promise(resolve => resolve(5)))
  .then(null, err => console.log(err))
  .then(value => console.log(value))

const anotherPromise = Promise.resolve(100)
setTimeout(() => {
  anotherPromise.then(value => console.log(value))
}, 1000)
```

#### Promise/A+规范

[Promise/A+规范](https://github.com/promises-aplus/promises-spec)

#### 代码实现之基本篇

```javascript
const PENDING = 'pending'
const REJECTED = 'rejected'
const RESOLVED = 'resolved'

const resolvePromise = (returnedValue, resolve, reject) => {
  if (returnedValue instanceof Promise) {
    returnedValue.then(resolve, reject)
  } else {
    resolve(returnedValue)
  }
}

class Promise {
  constructor(executor) {
    this.value = undefined
    this.status = PENDING
    this.onResolvedQueue = []
    this.onRejectedQueue = []

    const resolve = (value) => {
      if (this.status === PENDING) {
        process.nextTick(() => {
          this.status = RESOLVED
          this.value = value
          this.onResolvedQueue.forEach(cb => cb(value))
        })
      }
    }

    const reject = (err) => {
      if (this.status === PENDING) {
        process.nextTick(() => {
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

  static resolve(value) {
    return new Promise((resolve) => {
      resolve(value)
    })
  }

  static reject(err) {
    return new Promise((resolve, reject) => {
      reject(err)
    })
  }

  then(onResolved, onRejected) {
    onResolved = typeof onResolved === 'function' ? onResolved : value => value
    onRejected = typeof onRejected === 'function' ? onRejected : err => {
      throw err
    }

    let newPromise

    if (this.status === PENDING) {
      newPromise = new Promise((resolve, reject) => {
        this.onResolvedQueue.push(value => {
          try {
            const returnedValue = onResolved(value)
            resolvePromise(returnedValue)
          } catch (err) {
            reject(err)
          }
        })
        this.onRejectedQueue.push(err => {
          try {
            const returnedValue = onRejected(err)
            resolvePromise(returnedValue)
          } catch (err) {
            reject(err)
          }
        })
      })
    }

    if (this.status === RESOLVED) {
      newPromise = new Promise((resolve, reject) => {
        process.nextTick(() => {
          try {
            const returnedValue = onResolved(this.value)
            resolvePromise(returnedValue)
          } catch (err) {
            reject(err)
          }
        })
      })
    }

    if (this.status === REJECTED) {
      newPromise = new Promise((resolve, reject) => {
        process.nextTick(() => {
          try {
            const returnedValue = onRejected(this.value)
            resolvePromise(returnedValue)
          } catch (err) {
            reject(err)
          }
        })
      })
    }

    return newPromise
  }

  catch (onRejected) {
    return this.then(null, onRejected)
  }
}
```

#### 代码实现之完全篇

```javascript
const PENDING = 'pending'
const REJECTED = 'rejected'
const RESOLVED  = 'resolved'

const isObjOrFunction = value => typeof value === 'object' && value !== null || typeof value === 'function'

const resolvePromise = (promise, x, resolve, reject) => {
  if (promise === x) {
    return reject(
      new TypeError('a promise is resolved with a thenable that participates in a circular thenable chain')
    )
  }

  if (isObjOrFunction(x)) {
    let then

    // If retrieving the property x.then results in a thrown exception err,
    // reject promise with err as the reason.
    try {
      then = x.then
    } catch (err) {
      return reject(err)
    }

    // If x is a thenable, it attempts to make promise adopt the state of x,
    // under the assumption that x behaves at least somewhat like a promise.
    // Otherwise, it fulfills promise with the value x.
    if (typeof then === 'function') {
      let hasCalled = false

      try {
        const _resolvePromise = (y) => {
          if (!hasCalled) {
            hasCalled = true
            return resolvePromise(promise, y, resolve, reject)
          }
        }
        const _rejectPromise = (err) => {
          if (!hasCalled) {
            hasCalled = true
            return reject(err)
          }
        }

        then.call(x, _resolvePromise, _rejectPromise)
      } catch (err) {
        if (!hasCalled) {
          return reject(err)
        }
      }
    } else {
      resolve(x)
    }
  } else {
    resolve(x)
  }
}

class Promise {
  constructor(executor) {
    this.value = undefined
    this.status = PENDING
    this.onResolvedQueue = []
    this.onRejectedQueue = []

    const resolve = (value) => {
      if (this.status === PENDING) {
        process.nextTick(() => {
          this.status = RESOLVED
          this.value = value
          this.onResolvedQueue.forEach(cb => cb(value))
        })
      }
    }

    const reject = (err) => {
      if (this.status === PENDING) {
        process.nextTick(() => {
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

    let newPromise

    if (this.status === PENDING) {
      newPromise = new Promise((resolve, reject) => {
        this.onResolvedQueue.push(value => {
          try {
            const returnedValue = onResolved(value)
            resolvePromise(newPromise, returnedValue, resolve, reject)
          } catch (err) {
            reject(err)
          }
        })
        this.onRejectedQueue.push(err => {
          try {
            const returnedValue = onRejected(err)
            resolvePromise(newPromise, returnedValue, resolve, reject)
          } catch (err) {
            reject(err)
          }
        })
      })
    }

    if (this.status === RESOLVED) {
      newPromise = new Promise((resolve, reject) => {
        process.nextTick(() => {
          try {
            const returnedValue = onResolved(this.value)
            resolvePromise(newPromise, returnedValue, resolve, reject)
          } catch (err) {
            reject(err)
          }
        })
      })
    }

    if (this.status === REJECTED) {
      newPromise = new Promise((resolve, reject) => {
        process.nextTick(() => {
          try {
            const returnedValue = onRejected(this.value)
            resolvePromise(newPromise, returnedValue, resolve, reject)
          } catch (err) {
            reject(err)
          }
        })
      })
    }

    return newPromise
  }

  catch (onRejected) {
    return this.then(null, onRejected)
  }
}

```
