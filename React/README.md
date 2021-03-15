* 对React Fiber的理解 及渲染树   --   todo


* setState的原理理解        --      todo

[文章地址](https://juejin.im/post/5b45c57c51882519790c7441)


* unstable_batchedUpdates  --  react-dom未开放的一个api
在antd的用来监听鼠标事件以及mobx用来更新渲染里面都有用到，但是查了一下没有很好的解决， -- todo


### React-router 原理

hash路由：是通过window的hashchange的事件监听Hash地址的变化达到不同渲染

history路由：是使用history接口。是通过history压栈弹栈。需要nginx把业务路由配置到index.html下面