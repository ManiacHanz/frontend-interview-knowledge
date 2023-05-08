* vue的源码是否看过，说⼀下⽐较有收获的⼏个点

* APP内嵌H5⻚⾯如何和APP本⾝进⾏通信 
* 微信⼩程序和传统h5⻚⾯相⽐哪个性能更好⼀些，为什么
* H5的开发和PC端的开发有什么本质的不同 
* 如何针对H5⻚⾯进⾏远程调试？ 
* 如何进⾏⼿机的⻚⾯或者应⽤的抓包
* https和http  1.0 1.1 2.0
* 说⼀下从浏览器输⼊⽹址到⻚⾯渲染中间发⽣了什么
* 说下你知道的HTTP状态码并说出它们的出现场景
* 说下你知道的设计模式及应⽤场景
* 说⼀下从浏览器输⼊⽹址到⻚⾯渲染中间发⽣了什么


1.如何确保项⽬按时交付 2.如何安排开发和管理的时间分配 3.如何体现项⽬价值

* 如何进⾏前端性能优化


* 为什么选择react / vue, 两者区别

react模板需要静态编译的更少，jsx更贴近js语法，数据更新主要通过setState触发。diff主要是从root到下的深度遍历，需要的在fiber保存的数据更多。以及有alternate来保存

vue有template模板的概念，及对应的compiler编译器。数据更新更准确，通过Proxy确定响应式数据，及相应的副作用函数，配合模板编译器更准确


* 构建流程/cli/⼯作流/git 的⼯作

构建流程： 服务器拉取代码，安装依赖，打包，启动服务。
          jenkins可以设定环境变量，打包脚本等等
          可以有webhook。通过设定仓库的特定事件，调用服务端接口，执行操作。
          启动服务可以是nginx跑起的服务

cli

工作流

git的工作  add commit push reset revert rebase cherry-pick 


* 项目里的webpack配置等等 用的插件等等 详细看下 react高阶组件

* react源码看了能学到什么
   - 链表的数据结构 和数组的对比
   - 用位运算表示状态
   - 双缓存策略
   - 深度优先遍历，修改树的状态


* react setState 实现

* react 怎么根据 state 和 props 的变化去更新 dom 树的

* react fiber算法原理及实现, 大概看下属性的意义

* react事件系统

* concurrent mode

* 小程序账号科普 openId unionId

* mongo aggregate看一下




* node child_progress cluster forker
* vue 主要是想要问 keep-alive 和 actived


* 瀑布屏如何实现， 无限滚动如何实现
[无线滚动的简单实现](https://zhuanlan.zhihu.com/p/26022258)
[无线滚动的高度优化](https://zhuanlan.zhihu.com/p/34585166)
无限滚动的大概原理：
   需要根据数据计算出一个足够高的容器，用来滚动。其次需要有一个比视口更高的列表，用来装载需要展示的数据。原理是计算两个，一个是滚动条的滚动位置，用来改变列表的Y轴位置，使它们始终在视口中；第二个是根据滚动条的位置和每一条数据的高度计算需要展示的数据内容，完成展示数据的切换

* 你在项⽬中的任务是什么? 最难的模块是什么？如何实现

* cdn

* requestAnimationFrame与setTimeout 区别

* 进程和线程的区别，Chrome 中有哪些常⻅进程，如果我有⼀个耗时很⻓的同步计算任务，如何让 JS 代码达到多线程并发执⾏的效果？


* web-component

* 简述公司node架构中容灾的实现

* 微前端的意义在哪⾥ 为什么是微前端⽽不是iframe

* 手写各种函数
   [掘金](https://juejin.cn/post/6844903856489365518)

* Generator再看下

* DVA官网对原理的解释看下

* webpack-dev-server以及Hot module的实现
https://github.com/879479119/879479119.github.io/issues/5
https://juejin.cn/post/6844904008432222215

* 问题1.一个项目全是 html 文件，现在要改成 vue 实现，你会怎么做？

* 问题2，现在pc网页，要调用一个应用程序，类似百度网盘这样，你会怎么做？

* 问题3,你能说说 react 跟 vue 有什么区别吗

* 问题4，你研究过的设计模式说几种，并说说使用场景

* 问题5，npm 私服怎么搭建，说说流程？

* 问题6，npm 包发布需要注意哪些？

http无状态的原因？
HTTP是不保存状态的协议，既无状态协议，协议本身对于请求或响应之间的通信状态不进行保存，因此连接双方不能知晓对方当前的身份和状态。
HTTP有无连接的特性，即每次连接只能处理一个请求，收到响应后立即断开连接。

* 网页性能优化的各个方面的总结
[掘金](https://juejin.cn/post/6844903655330562062)


* 知识点map
[掘金](https://juejin.cn/post/6844903830887366670)

* 事件流
事件流 描述的是页面中事件接收的顺序
冒泡的说法： 关键词 *具体的元素* *向上传播* *不太具体的元素 -- 文档*

* 继承的区别

* 广度优先实现深拷贝

* Async / Await实现原理


* Promise.all 的源码手写
* taro 的编译流程 -> webpack的编译流程
* 微前端的理解 以及qiankun
* 提升nodejs水平
* 写下令牌桶的抓错