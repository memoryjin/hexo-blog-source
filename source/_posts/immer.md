---
title: immer讨论分享
---

##### 从简单的例子说起

从一个简单的例子说起，考虑下面这个例子：

```javascript
// case 1
const state = {
  name: 'wq',
  age: 18
}
const nextState = {
  name: 'wq',
  age: 20
}

// case 2
const state = {
  name: 'wq',
  detail: {
    age: 18,
    sex: 'male'
  }
}
const nextState = {
  name: 'wq',
  detail: {
    age: 20,
    sex: 'male'
  }
}

// case 3
const state = {
  name: 'wq',
  detail: {
    age: 18,
    children: ['yezi', 'xiaosi']
  }
}

const nextState = {
  name: 'wq',
  detail: {
    age: 18,
    children: ['yezi', 'xiaosi', 'xiaohei']
  }
}
```

对于case1，这个简单，我们通常可以直接如下

```javascript
const nextState = {...state, age: 20}
```

对于case2，好吧，这个好像麻烦了点，但还是可以实现如下

```javascript
const nextState = {...state, detail: {...state.detail, age: 20}}
```

对于case3，额...这时候估计你就懒得再来什么同构结构了，一般会比较粗暴的采用如下方法

```javascript
const nextState = deepClone(state)
nextState.detail.children.push('xiaohei')
```

从上面三个例子可以看出，对于层级不深的数据，通过ES6的扩展运算符可以比较优雅的获取想要的state，但一旦层级超过3级，这时候再用扩展运算符来写就有点力不从心了。为了解决这个问题，上面我们采用的是"深克隆"来实现。但深克隆是最简单的实现却未必是最好的实现，因为当state中的数据结构比较复杂，比较臃肿的时候深克隆带来的性能开销是比较大的。

##### 数据的不可变性——immutable

为了解决深克隆性能导致的性能问题，facebook在2014年推出了`immutable.js`，只不过当时的react的风头实在太盛，完全盖过了immutable的光芒。mobx的作者则在2018年2月推出了`immer.js`。

`immutable.js`使用的是另一套数据结构的API，首先将原生对象转换成immutable对象，并且每次操作将返回一个新的immutable对象。以上文的case3为例，借助immutable.js我们可以实现如下：

```javascript
import { fromJS } from 'immutable'

const nextState = fromJS(state)
	.updateIn(['detail', 'children'], list => list.push('xiaohei'))
	.toJS()
```

`immer.js`则不然，使用immer时我们操作的仍然是原生的js对象：

```javascript
import produce from 'immer'

const nextState = produce(state, draftState => {
  draftState.detail.children.push('xiaohei')
})
```

相比`immutable.js`，`immer.js`没有引入新的API，对于新手更友好，但也正因为没有新的API，所以在对数据的操作上immer不如immutable灵活强大。在性能上，二者的速度基本相当，但由于immutable.js最后都要执行toJS的操作，而这个过程是比较耗时的，所以总的来说immer.js的性能略优于immutable.js吧。

##### Immer.js原理浅析

让我们先抛开immer，看看下面这个数据结构：

```javascript
const state = {
  basic: {
    name: 'wq',
    age: '18'
  },
  detail: {
    sex: 'female',
    address: 'www.baidu.com',
    children: ['yezi', 'xiaohua']
  }
}

const nextState = {
  basic: {
    name: 'wq',
    age: '18'
  },
  detail: {
    sex: 'female',
    address: 'www.baidu.com',
    children: ['yezi', 'xiaohua', 'xiaomo']
  }
}
```

通过deepClone，我们可以在不改变`state`的基础上得到`nextState`，而所谓deepClone，本质上就是**递归地shallowCopy**，而这也正是导致deepClone性能不好的根源所在！观察上面这个例子，其实我们要改的是detail中的信息，basic的信息并未发生变化。既然性能瓶颈是由于递归的shallowCopy导致的，那么我们是否可以这么想，**如果我们可以提前知道哪些属性将会发生改变，然后只对将会发生改变的那些属性进行递归的shallowCopy（深复制），对于未发生改变的属性我们直接取原始值，这样是否可以大大提高性能？**答案是肯定的，但问题是你如何"提前"知道变化的属性？？当对对象的属性进行赋值的时候在JS语言层面到底会发生什么？？而这其中是否有方法让我们"提前"知道到底是哪个属性发生变化了？？请大家思考两分钟......















答案是**`Object.defineProperty`**和**`Proxy`**。对于前者，当对一个对象进行写操作时本质上会触发Object.defineProperty中的set函数，对于后者，Proxy 用于修改某些操作的默认行为，等同于在语言层面做出修改，可以理解成，在目标对象之前架设一层“拦截”，外界对该对象的访问，都必须先通过这层拦截，因此提供了一种机制，可以对外界的访问进行过滤和改写。更多的关于`Proxy`的知识可以查看[阮一峰老师的ES6入门](http://es6.ruanyifeng.com/#docs/proxy) 。下面我们以`Proxy`为例详细分析下`immer.js`的基本实现原理：

1、首先我们定义两个`Map`类型的对象和`Proxy`中的过滤及改写外界访问的objectTraps

```javascript
// Maps baseState objects to proxies
const proxies = new Map()
// Maps baseState objects to their copies
const copies = new Map()

const objectTraps = {
  get (target, prop) {
    return getOrCreateProxy(target[prop])
  },
  set (target, prop, val) {
    const current = getOrCreateProxy(target[prop])
    const newVal = getOrCreateProxy(val)
    if (current !== newVal) {
      const copy = getOrCreateCopy(target)
      copy[prop] = newVal
    }
    return true
  },
  deleteProperty (target, prop) {
    const copy = getOrCreateCopy(target)
    delete copy[prop]
    return true
  }
}
```

其中**`getOrCreateProxy`**和**`getOrCreateCopy`**具体如下：

```javascript
const getOrCreateProxy = baseState => {
  if (typeof baseState === 'object' && baseState !== null || Array.isArray(baseState)) {
    if (proxies.has(baseState)) {
      return proxies.get(baseState)
    }
    const proxy = new Proxy(baseState, objectTraps)
    proxies.set(baseState, proxy)
    return proxy
  } else {
    return baseState
  }
}

const getOrCreateCopy = baseState => {
  if (copies.has(baseState)) {
    return copies.get(baseState)
  }
  const copy = Array.isArray(baseState) ? baseState.slice() : {...baseState}
  copies.set(baseState, copy)
  return copy
}
```

到此为止，我们把对对象的读写操作全部转移到了它的copy对象上，从而保证了初始对象的`immutable`。接下来我们要做的是如何通过初始对象和这些拷贝对象构建出我们修改过后的新对象，为此，我们定义了下面的`finalize`方法：

```javascript
const finalize = baseState => {
  if (typeof baseState === 'object' && baseState !== null || Array.isArray(baseState)) {
    if (!hasChanges(baseState)) {
      return baseState
    }
    const copy = getOrCreateCopy(baseState)
    Object.keys(copy).forEach(prop => {
      copy[prop] = finalize(copy[prop])
    })
    return copy
  } else {
    return baseState
  }
}

const hasChanges = baseState => {
  if (!proxies.has(baseState)) {
    return false
  }
  if (copies.has(baseState)) {
    return true
  }
  return Object.values(baseState).some(value => {
    return typeof value === 'object' && value !== null
    	|| Array.isArray(baseState)
    	&& hasChanges(value)
  })
}
```

我们通过递归的思路来完成最终新对象的“构建”，到此为止，我们的`produce`函数其实也就出来了，完整代码如下：

```javascript
function produce (baseState, thunk) {
  // Maps baseState objects to proxies
  const proxies = new Map()
  // Maps baseState objects to their copies
  const copies = new Map()

  const objectTraps = {
    get (target, prop) {
      return getOrCreateProxy(target[prop])
    },
    set (target, prop, val) {
      const current = getOrCreateProxy(target[prop])
      const newVal = getOrCreateProxy(val)
      if (current !== newVal) {
        const copy = getOrCreateCopy(target)
        copy[prop] = newVal
      }
      return true
    },
    deleteProperty (target, prop) {
      const copy = getOrCreateCopy(target)
      delete copy[prop]
      return true
    }
  }
  
  const getOrCreateProxy = baseState => {
    if (typeof baseState === 'object' && baseState !== null || Array.isArray(baseState)) {
      if (proxies.has(baseState)) {
        return proxies.get(baseState)
      }
      const proxy = new Proxy(baseState, objectTraps)
      proxies.set(baseState, proxy)
      return proxy
    } else {
      return baseState
    }
  }

  const getOrCreateCopy = baseState => {
    if (copies.has(baseState)) {
      return copies.get(baseState)
    }
    const copy = Array.isArray(baseState) ? baseState.slice() : {...baseState}
    copies.set(baseState, copy)
    return copy
  }
  
  const finalize = baseState => {
    if (typeof baseState === 'object' && baseState !== null || Array.isArray(baseState)) {
      if (!hasChanges(baseState)) {
        return baseState
      }
      const copy = getOrCreateCopy(baseState)
      Object.keys(copy).forEach(prop => {
        copy[prop] = finalize(copy[prop])
      })
      return copy
    } else {
      return baseState
    }
  }

  const hasChanges = baseState => {
    if (!proxies.has(baseState)) {
      return false
    }
    if (copies.has(baseState)) {
      return true
    }
    return Object.values(baseState).some(value => {
      return typeof value === 'object' && value !== null
        || Array.isArray(baseState)
        && hasChanges(value)
    })
  }
  
  // create proxy for root
  const rootProxy = getOrCreateProxy(baseState)
  // execute the thunk
  thunk(rootProxy)
  // and finalize the modified proxy
  return finalize(baseState)
}

```

接下来我们回到本节开篇的小例子来验证上述的produce方法是否生效

```javascript
const state = {
  basic: {
    name: 'wq',
    age: '18'
  },
  detail: {
    sex: 'female',
    address: 'www.baidu.com',
    children: ['yezi', 'xiaohua']
  }
}

const nextState = {
  basic: {
    name: 'wq',
    age: '18'
  },
  detail: {
    sex: 'female',
    address: 'www.baidu.com',
    children: ['yezi', 'xiaohua', 'xiaomo']
  }
}

const target = produce(state, draftState => {
  draftState.detail.children.push('xiaomo')
})
```

在控制台上打印`state`和`target`:

![](https://img.alicdn.com/tfs/TB1w8hWaH2pK1RjSZFsXXaNlXXa-610-438.png)

由此可见我们定义的produce方法跟我们的预期相符，功能正确。但是真的如此吗？？我们再看一个稍微复杂点的例子：

```javascript
const state = {
  a: 1,
  b: 2,
  list: [10, 20, 50],
  person: {
    basicInfo: {
      name: 'wq',
      age: 18
    },
    detailInfo: {
      sex: 'female',
      chldren: [
        {
          name: 'xiaosan'
        },
        {
          name: 'lisi'
        }
      ]
    }
  }
}

const nextState = {
  b: 2,
  list: [10, 20, 50],
  person: {
    basicInfo: {
      name: 'wq',
      age: 18
    },
    detailInfo: {
      sex: 'female',
      chldren: [
        {
          name: 'xiaosan'
        },
        {
          name: '李四'
        },
        {
          name: '王五'
        },
        {
          name: '李六'
        }
      ]
    }
  }
}
```

类似的，借助produce方法，我们可以通过如下操作得到nextState:

```javascript
const target = produce(state, draftState => {
  delete draftState.a
  draftState.person.detailInfo.children[1].name = '李四'
  draftState.person.detailInfo.children.push({
    name: '王五'
  })
  draftState.person.detailInfo.children.push({
    name: '李六'
  })
})
```

我们在控制台上打印出target看看

![](https://img.alicdn.com/tfs/TB1weSlaNTpK1RjSZFGXXcHqFXa-609-284.png)

很诡异的一幕出现了，我们对children数组Push了两次，结果只有第二次的生效了，并且还被Proxy包裹了一层。哪里出了问题？？这个问题感兴趣的同学可以自己去推敲研究，正确完整的代码会在文末贴出，但如果同学们可以根据这份残缺的代码找出错误并修复，那么其实际意义肯定是远大于所谓的研读源码的。思路是没问题的，但其间确实犯了几个不小的错误，同学们，find out them and fix them!!!

















------

```javascript
const IMMER_PROXY = Symbol('immer-proxy')

const isPlainObject = value => {
  if (value === null || typeof value !== "object") {
    return false
  }
  const proto = Object.getPrototypeOf(value)
  return proto === Object.prototype || proto === null
}

const isProxy = value => !!value && !!value[IMMER_PROXY]

function produce (baseState, thunk) {
  // Maps baseState objects to proxies
  const proxies = new Map()
  // Maps baseState objects to their copies
  const copies = new Map()

  const objectTraps = {
    get (target, prop) {
      if (prop === IMMER_PROXY) {
        return target
      }
      return getOrCreateProxy(getCurrentSource(target)[prop])
    },
    set (target, prop, val) {
      const current = getOrCreateProxy(getCurrentSource(target)[prop])
      const newVal = getOrCreateProxy(val)
      if (current !== newVal) {
        const copy = getOrCreateCopy(target)
        copy[prop] = isProxy(newVal)
          ? newVal[IMMER_PROXY]
          : newVal
      }
      return true
    },
    deleteProperty (target, prop) {
      const copy = getOrCreateCopy(target)
      delete copy[prop]
      return true
    }
  }
  
  const getOrCreateProxy = baseState => {
    if (isPlainObject(baseState) || Array.isArray(baseState)) {
      // avoid double wrapping
      if (isProxy(baseState)) {
        return baseState
      }

      if (proxies.has(baseState)) {
        return proxies.get(baseState)
      }

      const proxy = new Proxy(baseState, objectTraps)
      proxies.set(baseState, proxy)
      return proxy
    } else {
      return baseState
    }
  }

  const getOrCreateCopy = baseState => {
    if (copies.has(baseState)) {
      return copies.get(baseState)
    }
    const copy = Array.isArray(baseState) ? baseState.slice() : {...baseState}
    copies.set(baseState, copy)
    return copy
  }

  const getCurrentSource = baseState => {
    return copies.get(baseState) || baseState
  }
  
  const finalize = baseState => {
    if (isPlainObject(baseState) || Array.isArray(baseState)) {
      if (!hasChanges(baseState)) {
        return baseState
      }
      const copy = getOrCreateCopy(baseState)
      Object.keys(copy).forEach(prop => {
        copy[prop] = finalize(copy[prop])
      })
      return copy
    } else {
      return baseState
    }
  }

  const hasChanges = baseState => {
    if (!proxies.has(baseState)) {
      return false
    }
    if (copies.has(baseState)) {
      return true
    }
    return Object.values(baseState).some(value => {
      return isPlainObject(value)
        || Array.isArray(value)
        && hasChanges(value)
    })
  }
  
  // create proxy for root
  const rootProxy = getOrCreateProxy(baseState)
  // execute the thunk
  thunk(rootProxy)
  // and finalize the modified proxy
  return finalize(baseState)
}
```

