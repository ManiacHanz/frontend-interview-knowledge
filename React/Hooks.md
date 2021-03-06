
# Hooks 

[官方文档](https://react.css88.com/docs/hooks-effect.html)

*写这篇的时候`react`的`version`是`16.6`，纯属个人兴趣，参阅官网写下`hooks`的简单理解和用法*

```js
npm i react@next --save
npm i react-dom@next --save
```

简单通俗但不一定准确的说，`Hooks`提供`react`在`函数式组件`里增加可以自己管理的`state`，而且可以在`生命周期`的钩子中做一定的处理的功能。

#### Hello world

* useState

`useState`hooks的提供的一个核心方法，接收一个参数作为初始的`state`（可以是任何类型的值），返回的一个长度为2的数组，第一位是这个`state`的变量名，第二个是修改这个变量的函数，函数内可以接收当前的`state`作为参数。
![](./assets/useState.png "useState(0)的截图")

```jsx
import React, { Component, useState } from 'react

function Example() {
  
  const [count, setCount] = useState(0)
  
  return (
    <div>
      <p>Click {count} times</p>
      <button onClick={()=>setCount(count+1)}> click me </button>
    </div>
  )
}

class App extends Component {
  render() {
    return (
      <div className="App">
        <Example />
        
      </div>
    );
  }
}
```

不同于在`class`中用`this.state={}`声明状态的形式，用`hook`声明多个值的时候可以写多个`useState`

```jsx
function ExampleWithManyStates() {
  // 声明多个 state(状态) 变量 
  const [age, setAge] = useState(42);
  const [fruit, setFruit] = useState('banana');
  const [todos, setTodos] = useState([{ text: 'Learn Hooks' }]);
  // ...
}
```


* useEffect

`useEffect`为函数式组件添加执行`side effect`的能力。
*这个函数赋予了函数式组件生命周期，但是组件本身又不需要关注具体是哪些生命周期*

`class`例子

```jsx
class Example extends React.Component {
  // ...
  componentDidMount() {
    document.title = `You clicked ${this.state.count} times`;
  }

  componentDidUpdate() {
    document.title = `You clicked ${this.state.count} times`;
  }
  // ...
  
}
```

换成新的api以后，这里的`useEffect`基本等于上面在`componentDidMount`和`componentDidUpdate`里面的效果。同时注意，里面传入的是一个匿名函数。这里有一个小点，传递给`useEffect`的这个匿名函数在每次渲染的时候都是不同的。[官方文档](https://react.css88.com/docs/hooks-effect.html#%E8%AF%A6%E7%BB%86%E8%AF%B4%E6%98%8E)文档的这里解释的是`故意的`。这样也会保证每次都会安排一个新的`effect`来替换前一个，而不用担心旧值的影响

> 传递给 useEffect 的函数在每次渲染时都是不同的。这是故意的。实际上，这就是让我们从 effect 内部读取计数值的原因，而不用担心它是旧值。每次我们重新渲染，我们安排一个不同的 effect 来替换前一个。在某种程度上说，这使得 effect 更像是渲染结果的一部分，每个effect都属于特定的渲染。 


```jsx
import { useState, useEffect } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

当然，有时候需要在组件卸载的时候清理`effect`, `class`里用的`componenetWillUnmount`，那么新的`useEffect`呢

```jsx
import { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    // Specify how to clean up after this effect:
    return function cleanup() {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```

答案就是在`useEffect`里返回一个函数（可以是匿名），用返回值可以自然的让这个变成可选。`react`内部会在组件销毁前执行这个函数，达到清理`effect`的效果

同`useState`一样，`useEffect`也可以分离达到效果

```jsx
import { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [count, setCount] = useState(0)
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }
  // 这里负责 didmount 和 didupdate 里的
  useEffect(() => {
    document.title = `You clicked ${count} times`
  })
  // 这里负责 有 willUnmount的情况 。甚至不用关心是在 didmount 或者didupdate写漏了造成的bug
  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    // Specify how to clean up after this effect:
    return function cleanup() {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });
}

```

`useEffect`还可以传入参数，通过第二个变量来优化性能

只有当`count`改变时才会运行，如果`count`没变则不会运行此段代码

```jsx
useEffect(() => {
  document.title = `You clicked ${count} times`;
}, [count]); // Only re-run the effect if count changes
```

当然，`useEffect`会在所有渲染结束时触发，但是我们之前只需要在`componentDidMount`中请求外部数据，并在视图中更新数据，那么是不是`useEffect`不可用了？

```jsx
useEffect(() => {
  fetch('somedata').then( res => useState())
}, [])
```

只需要在第二个参数上传个空数组就行


### Hook出现原因

主要是为了解决class Component组件难以提取公共逻辑的难题

之前在多个类似的class Component组件之间提取公共逻辑只有用HOC，通过*属性代理*或者是*反向继承*。但是HOC会造成几个问题
* HOC的原理是通过一个组件包裹，把公用逻辑放在包裹的组件。这样在HOC使用多了以后，在调试器里就会有很深的结构层次，往往还需要维护额外的*displayName*
* HOC是需要通过原组件生成新组件的逻辑。但是生成新组件是通过约定的形式，还是有可能会在误操作中修改了原组件，造成组件污染
* class Component里的一件事的逻辑往往需要分开写到不同的生命周期里，造成维护代价变大
* 由于代理了组件，*静态属性*和*ref*的传递需要额外注意

所以使用hook能解决上面问题，
* hook提取的公共逻辑只是一个函数，不会渲染多余的组件
* 由于是函数，所以不会对原组件产生影响
* hook能够把同一件事的逻辑放到一起


### 常见问题

* hook为什么必须是顶层实现，且不能放在条件判断力

hook在底层源码里是通过一个链表维护，每次执行通过顺序才能知道对应的状态管理。所以不能放在条件判断里

* useEffect返回的要么是清楚副作用的函数，要么什么都不返回

所以不能在useEffect中使用async
```js
useEffect(async () => {}) // 错

useEffect(() => {
  const asyncFn = async () => {}
  asyncFn()
})        // 对
```

* 如何useEffect只在更新时运行

需要通过useRef配合，生成一个标识，来标识首次渲染完毕，然后就能让useEffect只在更新后执行