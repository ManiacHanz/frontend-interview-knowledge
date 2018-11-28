# 渲染属性

[官方文档](https://react.css88.com/docs/render-props.html)

> 渲染属性，主要用于使用一个值为函数的prop在React组件之间的代码共享

使用方法，

```jsx
class Cat extends React.Component {
  render() {
    const mouse = this.props.mouse;
    return (
      <img src="/cat.jpg" style={{ position: 'absolute', left: mouse.x, top: mouse.y }} />
    );
  }
}

class Mouse extends React.Component {
  constructor(props) {
    super(props);
    this.handleMouseMove = this.handleMouseMove.bind(this);
    this.state = { x: 0, y: 0 };
  }

  handleMouseMove(event) {
    this.setState({
      x: event.clientX,
      y: event.clientY
    });
  }

  render() {
    return (
      <div style={{ height: '100%' }} onMouseMove={this.handleMouseMove}>

        {/*
          Instead of providing a static representation of what <Mouse> renders,
          use the `render` prop to dynamically determine what to render.
        */}
        {this.props.render(this.state)}
      </div>
    );
  }
}

// a
class MouseTracker extends React.Component {
  render() {
    return (
      <div>
        <h1>Move the mouse around!</h1>
        <Mouse render={mouse => (
          <Cat mouse={mouse} />
        )}/>
      </div>
    );
  }
}

// b
class MouseTracker extends React.Component {
  render() {
    return (
      <div>
        <h1>Move the mouse around!</h1>
        <Mouse children={state => (
          <Cat mouse={state} />
        )}/>
      </div>
    );
  }
}

// c 
class MouseTracker extends React.Component {
  render() {
    return (
      <div>
        <h1>Move the mouse around!</h1>
        <Mouse>
            {
                state => <Cat mouse={state} />
            }
        </Mouse>
      </div>
    );
  }
}
```

*上面的a,b,c三种写法达到的效果是一样的，说明所谓的`render prop`并不是一定要以`render`作为属性名，也可以直接用`children`的写法，也就是说，其实也是和`HOC`,`children`一样的目的*

这个例子满足了当我想把`Mouse`组件的功能重用在不同的组件上，不用去写重复的代码，只需要继承决定需要渲染什么。如果说以`a`方法写起来还有点晦涩，那么我们把上面的改一点点，如下


```jsx
class Mouse extends React.Component {
  //...

  render() {
    return (
      <div style={{ height: '100%' }} onMouseMove={this.handleMouseMove}>
        {this.props.children(this.state)}
      </div>
    );
  }
}
```

那么一旦到了上面的代码是不是看起来非常明确了呢（感觉突然就没有什么需要写的了），只是在使用`Mouse`组件的时候中间用一个函数来返回`children`组件就可以了

```jsx
// 以前
<Mouse>
    <div></div>
</Mouse>

// 现在
<Mouse>
    {
        renderProps => (
            <div>{renderProps}</div>
        )
    }
</Mouse>
```


####  当然还是有一点需要注意，就是使用`React.PureComponent`的时候

`render`方法里创建函数，每次返回的`render prop`会返回一个新的函数，使得`PureComponent`“失效”。绕过这个方法的原理就是在`render`里面用一个函数指针，代替每次创建一个新方法

```jsx
class MouseTracker extends React.Component {
  constructor(props) {
    super(props);

    // This binding ensures that `this.renderTheCat` always refers
    // to the *same* function when we use it in render.
    this.renderTheCat = this.renderTheCat.bind(this);
  }

  renderTheCat(mouse) {
    return <Cat mouse={mouse} />;
  }

  render() {
    return (
      <div>
        <h1>Move the mouse around!</h1>
        <Mouse render={this.renderTheCat} />
      </div>
    );
  }
}
```


##### 完