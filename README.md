## 前言
前端中的库很多，开发这些库的作者会尽可能的覆盖到大家在业务中千奇百怪的需求，但是总有无法预料到的，所以优秀的库就需要提供一种机制，让开发者可以干预插件中间的一些环节，从而完成自己的一些需求。  

本文将从`axios`、`vuex`和`redux`的实现来教你怎么编写属于自己的插件机制。  

* 对于新手来说：  
本文能让你搞明白神秘的插件和拦截器到底是什么东西。

* 对于老手来说：  
在你写的开源框架中也加入拦截器或者插件机制，让它变得更加强大吧！  


## axios

首先我们模拟一个简单的axios，
```js
const axios = config => {
  if (config.error) {
    return Promise.reject({
      error: 'error in axios',
    });
  } else {
    return Promise.resolve({
      ...config,
      result: config.result,
    });
  }
};
```

如果传入的config中有error参数，就返回一个rejected的promise，反之则返回resolved的promise。  

先简单看一下axios官方提供的拦截器示例：

```js
axios.interceptors.request.use(function (config) {
    // 在发送请求之前做些什么
    return config;
  }, function (error) {
    // 对请求错误做些什么
    return Promise.reject(error);
  });

// 添加响应拦截器
axios.interceptors.response.use(function (response) {
    // 对响应数据做点什么
    return response;
  }, function (error) {
    // 对响应错误做点什么
    return Promise.reject(error);
  });
```

可以看出，不管是request还是response的拦截求，都会接受两个函数作为参数，一个是用来处理正常流程，一个是处理失败流程，这让人想到了什么？  

没错，`promise.then`接受的同样也是这两个参数。  

axios内部正是利用了promise的这个机制，把use传入的每一对函数作为一个`intercetpor`
```js
// 把
axios.interceptors.response.use(func1, func2)

// 注册为
const interceptor = {
    resolved: func1,
    rejected: func2
}
```

接下来简单实现一下：  

```js

// 先构造一个对象 存放拦截器
axios.interceptors = {
  request: [],
  response: [],
};

// 注册请求拦截器
axios.useRequestInterceptor = (resolved, rejected) => {
  axios.interceptors.request.push({ resolved, rejected });
};

// 注册响应拦截器
axios.useResponseInterceptor = (resolved, rejected) => {
  axios.interceptors.response.push({ resolved, rejected });
};

// 运行拦截器
axios.run = config => {
  const chain = [
    {
      resolved: axios,
      rejected: undefined,
    },
  ];

  // 把请求拦截器往数组头部推
  axios.interceptors.request.forEach(interceptor => {
    chain.unshift(interceptor);
  });

  // 把响应拦截器往数组尾部推
  axios.interceptors.response.forEach(interceptor => {
    chain.push(interceptor);
  });

  // 把config也包装成一个promise
  let promise = Promise.resolve(config);

  // 暴力while循环解忧愁 
  // 利用promise.then的能力递归执行所有的拦截器
  while (chain.length) {
    const { resolved, rejected } = chain.shift();
    promise = promise.then(resolved, rejected);
  }

  // 最后暴露给用户的就是响应拦截器处理过后的promise
  return promise;
};
```

从`axios.run`这个函数看运行时的机制，首先构造一个`chain`作为promise链，并且把正常的请求也就是我们的请求参数axios也构造为一个拦截器的结构，接下来  

* 把request的interceptor给unshift到函数顶部
* 把response的interceptor给push到函数尾部

以这样一段调用代码为例：
```js
axios.useRequestInterceptor(resolved1, rejected1); // requestInterceptor1 

axios.useRequestInterceptor(resolved2, rejected2); // requestInterceptor2 

axios.useResponseInterceptor(resolved1, rejected1); // responseInterceptor1 

axios.useResponseInterceptor(resolved2, rejected2); // responseInterceptor2
```
这样子构造出来的promise链就是这样的`chain`结构：

```js
[
    requestInterceptor2，
    requestInterceptor1，
    axios,
    responseInterceptor1,
    responseInterceptor2
]
```

至于为什么requestInterceptor的顺序是反过来的，仔细看看代码就知道 XD。

有了这个`chain`之后，只需要一句简短的代码：

```js
 let promise = Promise.resolve(config);

  while (chain.length) {
    const { resolved, rejected } = chain.shift();
    promise = promise.then(resolved, rejected);
  }

  return promise;
```

promise就会把这个链从上而下的执行了。  

以这样的一段测试代码为例：
```js
axios.useRequestInterceptor(config => {
  return {
    ...config,
    extraParams1: 'extraParams1',
  };
});

axios.useRequestInterceptor(config => {
  return {
    ...config,
    extraParams2: 'extraParams2',
  };
});

axios.useResponseInterceptor(
  resp => {
    const {
      extraParams1,
      extraParams2,
      result: { code, message },
    } = resp;
    return `${extraParams1} ${extraParams2} ${message}`;
  },
  error => {
    console.log('error', error)
  },
);
```

### 成功的调用
在成功的调用下输出 `result1:  extraParams1 extraParams2 message1`  
```js
(async function() {
  const result = await axios.run({
    message: 'message1',
  });
  console.log('result1: ', result);
})();
```

### 失败的调用
```js
(async function() {
  const result = await axios.run({
    error: true,
  });
  console.log('result3: ', result);
})();
```
在失败的调用下，则进入响应拦截器的rejected分支：  

首先打印出拦截器定义的错误日志：  
`error { error: 'error in axios' }`   

然后由于失败的拦截器
```js
error => {
  console.log('error', error)
},
```
没有返回任何东西，打印出`result3:  undefined`  

可以看出，axios的拦截器是非常灵活的，可以在请求阶段任意的修改config，也可以在响应阶段对response做各种处理，这也是因为用户对于请求数据的需求就是非常灵活的，没有必要干涉用户的自由度。  


## vuex
vuex提供了一个api用来在action被调用前后插入一些逻辑：  

https://vuex.vuejs.org/zh/api/#subscribeaction

```js
store.subscribeAction({
  before: (action, state) => {
    console.log(`before action ${action.type}`)
  },
  after: (action, state) => {
    console.log(`after action ${action.type}`)
  }
})
```

其实这有点像AOP（面向切面编程）的编程思想。  

在调用`store.dispatch({ type: 'add' })`的时候，会在执行前后打印出日志
```js
before action add
add
after action add
```  

来简单实现一下：  

```js
import { Actions, ActionSubscribers, ActionSubscriber, ActionArguments } from './vuex.type';

class Vuex {
  state = {};

  action = {};

  _actionSubscribers = [];

  constructor({ state, action }) {
    this.state = state;
    this.action = action;
    this._actionSubscribers = [];
  }

  dispatch(action) {
    // action前置监听器
    this._actionSubscribers
      .forEach(sub => sub.before(action, this.state));

    const { type, payload } = action;
    
    // 执行action
    this.action[type](this.state, payload).then(() => {
       // action后置监听器
      this._actionSubscribers
        .forEach(sub => sub.after(action, this.state));
    });
  }

  subscribeAction(subscriber) {
    // 把监听者推进数组
    this._actionSubscribers.push(subscriber);
  }
}

const store = new Vuex({
  state: {
    count: 0,
  },
  action: {
    async add(state, payload) {
      state.count += payload;
    },
  },
});

store.subscribeAction({
  before: (action, state) => {
    console.log(`before action ${action.type}, before count is ${state.count}`);
  },
  after: (action, state) => {
    console.log(`after action ${action.type},  after count is ${state.count}`);
  },
});

store.dispatch({
  type: 'add',
  payload: 2,
});
```

此时控制台会打印如下内容：
```js
before action add, before count is 0
after action add, after count is 2
```

轻松实现了日志功能。

当然Vuex在实现插件功能的时候，选择性的将 type payload 和 state暴露给外部，而不再提供进一步的修改能力，这也是框架内部的一种权衡，当然我们可以对state进行直接修改，但是不可避免的会得到Vuex内部的警告，因为在Vuex中，所有state的修改都应该通过mutations来进行，但是Vuex没有选择把commit也暴露出来，这也约束了插件的能力。  

## redux

想要理解redux中的中间件机制，需要先理解一个方法：`compose`
```js
function compose(...funcs: Function[]) {
  return funcs.reduce((a, b) => (...args: any) => a(b(...args)))
}
```  

简单理解的话，就是`compose(fn1, fn2, fn3) (...args) = > fn1(fn2(fn3(...args)))`  
它是一种高阶聚合函数，相当于把fn3先执行，然后把结果传给fn2再执行，再把结果交给fn1去执行。  

有了这个前置知识，就可以很轻易的实现redux的中间件机制了。  

虽然redux源码里写的很少，各种高阶函数各种柯里化，但是抽丝剥茧以后，redux中间件的机制可以用一句话来解释：  

*把dispatch这个方法不端用高阶函数包装，最后返回一个强化过后的dispatch*，  

以logMiddleware为例，这个middleware接受原始的redux dispatch，返回的是
```js
const typeLogMiddleware = (dispatch) => {
    // 返回的其实还是一个结构相同的dispatch，接受的参数也相同
    // 只是把原始的dispatch包在里面了而已。
    return ({type, ...args}) => {
        console.log(`type is ${type}`)
        return dispatch({type, ...args})
    }
}
```  

有了这个思路，就来实现这个mini-redux吧：  

```js
function compose(...funcs) {
    return funcs.reduce((a, b) => (...args) => a(b(...args)));
}

function createStore(reducer, middlewares) {
    let currentState;
    
    function dispatch(action) {
        currentState = reducer(currentState, action);
    }
    
    function getState() {
        return currentState;
    }
    // 初始化一个随意的dispatch，要求外部在type匹配不到的时候返回初始状态
    // 在这个dispatch后 currentState就有值了。
    dispatch({ type: 'INIT' });  
    
    let enhancedDispatch = dispatch;
    // 如果第二个参数传入了middlewares
    if (middlewares) {
        // 用compose把middlewares包装成一个函数
        // 让dis
        enhancedDispatch = compose(...middlewares)(dispatch);
    }  
    
    return {
        dispatch: enhancedDispatch,
        getState,
    };
}

```

接着写两个中间件
```js
// 使用

const otherDummyMiddleware = (dispatch) => {
    // 返回一个新的dispatch
    return (action) => {
        console.log(`type in dummy is ${type}`)
        return dispatch(action)
    }
}

// 这个dispatch其实是otherDummyMiddleware执行后返回otherDummyDispatch
const typeLogMiddleware = (dispatch) => {
    // 返回一个新的dispatch
    return ({type, ...args}) => {
        console.log(`type is ${type}`)
        return dispatch({type, ...args})
    }
}

// 中间件从右往左执行。
const counterStore = createStore(counterReducer, [typeLogMiddleware, otherDummyMiddleware])

console.log(counterStore.getState().count)
counterStore.dispatch({type: 'add', payload: 2})
console.log(counterStore.getState().count)

// 输出：
// 0
// type is add
// type in dummy is add
// 2
```


## 总结
1. `axios` 把用户注册的每个拦截器构造成一个promise.then所接受的参数，在运行时把所有的拦截器按照一个promise链的形式以此执行。
 - 在发送到服务端之前，config已经是请求拦截器处理过后的结果
 - 服务器响应结果后，response会经过响应拦截器，最后用户拿到的就是处理过后的结果了。

2. vuex的实现最为简单，就是提供了两个回调函数，vuex内部在合适的时机去调用（我个人感觉大部分的库提供这样的机制也足够了）。
3. redux的源码里写的最复杂最绕，它的中间件机制本质上就是用高阶函数不断的把dispatch包装再包装，形成套娃。本文实现的已经是精简了n倍以后的结果了，不过复杂的实现也是为了很多权衡和考量，Dan对于闭包和高阶函数的运用已经炉火纯青了，只是外人去看源码有点头秃...    

希望看了这篇文章的你，能对于前端库中的插件机制有进一步的了解，进而更优秀的使用或者写出更好的开源框架。

本文所写的代码都整理在这个仓库里了：  
https://github.com/sl1673495/tiny-middlewares  

代码是使用ts编写的，js版本的代码在js文件夹内，各位可以按自己的需求来看。
