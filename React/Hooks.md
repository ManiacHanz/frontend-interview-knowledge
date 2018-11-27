
# Hooks 

*写这篇的时候`react`的`version`是`16.6`，纯属个人兴趣，参阅官网写下`hooks`的简单理解和用法*

```js
npm i react@next --save
npm i react-dom@next --save
```

简单通俗但不一定准确的说，`Hooks`提供`react`在`函数式组件`里增加可以自己管理的`state`，而且可以在`生命周期`的钩子中做一定的处理的功能。

#### Hello world

* useState

`useState`hooks的提供的一个核心方法，接收一个参数作为初始的`state`（可以是任何类型的值），返回的一个长度为2的数组，第一位是这个`state`的变量名，第二个是修改这个变量的函数，函数内可以接收当前的`state`作为参数。
[](./assets/useState.png "useState(0)的截图")

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