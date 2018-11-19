
# You Probably Don't Need Derived State

[官网文档](https://react.css88.com/blog/2018/06/07/you-probably-dont-need-derived-state.html)

> 这是一篇React的官方博文，位置比较难找，但是感觉讲东西比较常见且比较重要，尤其是对于一些常用的错误使用方法给了一些官方的建议，所以个人进行了一个阅读。以下仅为个人理解，切勿当做标准，欢迎纠错

*topics*

* When to use derived state -- 何时使用派生状态

* [Common bugs when using derived state  --  常见bug](#bugs)

    * 1) Anti-pattern: Unconditionally copying props to state   --  无脑的复制prop到state

    * 2) Anti-pattern: Erasing state when props change    --   自身的state因为props的改变被消除了

* [Prefered solutions   --   更好的解决办法](#上面的所有问题)

* [What about memoization   --   memoization 是什么](#`memoization`是一种性能优化方案)


**需要用到派生状态 -- 不论是新的`getDerivedStateFromProps`或是以前的`componentWillRecieveProps` -- 都应该少用，大部分情况下都有更好的解决方案替代。以上两种的触发情景并不是`props`改变时才会触发，而是当父组件`render`的时候都会触发，所以很容易造成bug或者性能损耗**


#### bugs


> Anti-pattern: Unconditionally copying props to state

下面是官方给出的第一个[例子](https://react.css88.com/blog/2018/06/07/you-probably-dont-need-derived-state.html#anti-pattern-unconditionally-copying-props-to-state)。这个例子讲的是`componentWillReceiveProps` 当父组件`render`的时候，会造成子组件自己修改的`state`丢失

```jsx
class EmailInput extends Component {
  state = { email: this.props.email };

  render() {
    return <input onChange={this.handleChange} value={this.state.email} />;
  }

  handleChange = event => {
    this.setState({ email: event.target.value });
  };

  componentWillReceiveProps(nextProps) {
    // This will erase any local state updates!
    // Do not do this.
    this.setState({ email: nextProps.email });
  }
}
```

接着给出了一个不可靠的解决方案，在`shouldComponentUpdate`判断`nextProps.email = this.state.email`，来阻止一些`render`。但是这个方案不可靠的地方在于
* 有可能子组件会继承许多`props`，那需要判断的东西就很多
* 如果有一个函数，父组件在更新时可能会新建一个`func`而不是以前的函数指针，导致触发子组件的`render`。[例子](https://codesandbox.io/s/jl0w6r9w59)

结论：`shouldComponentUpdate`应该只用作性能提升，而不应该作为是否应该刷新视图的钩子

> Anti-pattern: Erasing state when props change

针对上面的例子有一个进步的解决办法，就是利用**派生状态**, 在`componentWillReceiveProps`或者`getDerivedStateFromProps`去限制视图的`render`

```jsx
class EmailInput extends Component {
  state = {
    email: this.props.email
  };

  componentWillReceiveProps(nextProps) {
    // Any time props.email changes, update state.
    if (nextProps.email !== this.props.email) {
      this.setState({
        email: nextProps.email
      });
    }
  }
  
  // ...
}
```

但是上面代码同样不完美，如果有2个人拥有同样的`email props`。但是他们的其他信息不同，这个时候本该刷新的视图缺没有刷新。[例子](https://codesandbox.io/s/mz2lnkjkrx)


上面的所有问题，都是直接把`state`和`props`混淆用起来，或者说无脑使用`派生状态`的弊端。而代替上面的做法的应该用下面更优的解决方案

* 1) 把组件变成完全受控组件

```jsx
function EmailInput(props) {
  return <input onChange={props.onChange} value={props.email} />;
}
```

这样的好处是：组件无状态，更轻量。同时组件本身不关心数据变化，只负责渲染。

* 2) 把组件变成完全不受控组件，同时拥有一个`react`中的特殊属性`key`

```jsx
class EmailInput extends Component {
  state = { email: this.props.defaultEmail };

  handleChange = event => {
    this.setState({ email: event.target.value });
  };

  render() {
    return <input onChange={this.handleChange} value={this.state.email} />;
  }
}

<EmailInput
  defaultEmail={this.props.user.email}
  key={this.props.user.id}
/>
```

父组件传来的`props`只负责作为子组件的`default`值，而内部更改全部由组件自己控制。如果需要子组件回到默认值，只需要更改`key`属性即可。如果是个表单，可以给整个`form`加一个`key`

> react的`key`属性变更会新建一个组件而不是更新之前的组件。在有复杂数据逻辑的时候这样往往会更快的渲染出一个新的（默认的）组件

当然也会存在组件的初始化花销非常昂贵，这里官方也提供了两个方案

* 1) [Reset uncontrolled component with an ID prop](https://react.css88.com/blog/2018/06/07/you-probably-dont-need-derived-state.html#alternative-1-reset-uncontrolled-component-with-an-id-prop)

在派生状态里，用一个唯一的`props`来判断

```jsx
class EmailInput extends Component {
  state = {
    email: this.props.defaultEmail,
    prevPropsUserID: this.props.userID
  };

  static getDerivedStateFromProps(props, state) {
    // Any time the current user changes,
    // Reset any parts of state that are tied to that user.
    // In this simple example, that's just the email.
    if (props.userID !== state.prevPropsUserID) {
      return {
        prevPropsUserID: props.userID,
        email: props.defaultEmail,
        // .... the rest props which need to be initialized
      };
    }
    return null;
  }

  // ...
}
```

* 2) [Reset uncontrolled component with an instance method](https://react.css88.com/blog/2018/06/07/you-probably-dont-need-derived-state.html#alternative-2-reset-uncontrolled-component-with-an-instance-method)

*官方的意思是使用场景非常少见*

```jsx
class EmailInput extends Component {
  state = {
    email: this.props.defaultEmail
  };

  resetEmailForNewUser(newEmail) {
    this.setState({ email: newEmail });
  }

  // ...
}
```


### memoize

`memoization`是一种性能优化方案，核心原理是利用闭包缓存上一次的传参，以及传参得到的结果。尤其是在`render`生命周期中使用，可以减少不必要的性能损耗

使用很简单，直接上代码

```js
// PureComponents only rerender if at least one state or prop value changes.
// Change is determined by doing a shallow comparison of state and prop keys.
class Example extends PureComponent {
  // State only needs to hold the current filter text value:
  state = {
    filterText: ""
  };

  handleChange = event => {
    this.setState({ filterText: event.target.value });
  };

  render() {
    // The render method on this PureComponent is called only if
    // props.list or state.filterText has changed.
    const filteredList = this.props.list.filter(
      item => item.text.includes(this.state.filterText)
    )

    return (
      <Fragment>
        <input onChange={this.handleChange} value={this.state.filterText} />
        <ul>{filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
      </Fragment>
    );
  }
}
```

上面的缺点是，如果filterList数据庞大渲染速度慢，并且`PureComponent`因为其他的`props`原因没能保护使组件重新渲染，这个时候这个函数消耗的性能就会明显，所以此时就是`memoize`出场的时候了

```jsx
import memoize from "memoize-one";

class Example extends Component {
  // State only needs to hold the current filter text value:
  state = { filterText: "" };

  // Re-run the filter whenever the list array or filter text changes:
  filter = memoize(
    (list, filterText) => list.filter(item => item.text.includes(filterText))
  );

  handleChange = event => {
    this.setState({ filterText: event.target.value });
  };

  render() {
    // Calculate the latest filtered list. If these arguments haven't changed
    // since the last render, `memoize-one` will reuse the last return value.
    const filteredList = this.filter(this.props.list, this.state.filterText);

    return (
      <Fragment>
        <input onChange={this.handleChange} value={this.state.filterText} />
        <ul>{filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
      </Fragment>
    );
  }
}
```

上面的代码优化以后， 当filter函数传入的参数相同时，会直接返回缓存的结果，减少了不必要的开销