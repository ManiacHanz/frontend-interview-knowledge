2 parstInt第二个参数标识进制。当第一个参数大于第二个进制的时候标识无法解析 返回NaN

3 防抖是控制高频函数的触发频率，在一定时间内重复触发就刷新定时器；节流是稀释触发频率，必须要满足到间隔才能触发一次

6 广度优先的深拷贝。把对象构造成一个树结构，使用队列来拷贝

7 ES5/6的继承区别
    ES5函数继承通过function的组合寄生模式；es6是通过class..extends..模式；
    es5由于是new，所以是先生成的子类的实例，再通过父类的构造函数`Parent.call(this)`去改变子类上的属性；而es6是先通过`super()`生成父类的实例，再用子类的构造函数修改`this`指向，所以必须写super()，如果不调用就会报引用错，因为子类实例没有this
    es6中父类的静态方法也会被继承

9 (不准确) Async /Await 理解成一个Promise的语法糖，相当于包装后变成了一个promise

15

17

20

23

26

27

28

39 BFC ： 根元素；overflow不为hidden; position不为relative; 浮动; display弹性布局或者表格布局
    作用: 减少重排；清除浮动; 去掉margin塌陷

40 在 Vue 中，子组件为何不可以修改父组件传递的 Prop： 在initProps时候，子组件改变prop在defineReactive的里面会做判断, 通过isUpdatingChildComponent变量，而这个变量在`lifeCycle.js`中进行`updateChildComponent`的时候进行修改为true，完了以后改回false

43 sort默认方法排序会把数组转成字符串，然后按照utf-16编码来进行排序

45 HTTPS 握手过程中，客户端如何验证证书的合法性
  1. 证书中的明文包括有效时期和证书所有者，所以可以确定是否和域名相同，以及是否在有效期内
  2. 从操作系统内置的ca认证机构可以分辨证书的颁布机构是否有效
  3. 从操作系统内置的CA认证机构内取出公钥对证书的签名进行解密，拿到摘要；然后浏览器再使用相同的hash算法对拿到的内容进行计算，如果两次摘要相同，则内容合法

48 call比apply性能更好 （微乎其微）

49 为什么埋点采用1*1的透明图片：
  1. 解决跨域问题
  2. 比XHR的get性能更好
  3. 可以不用关注返回响应
  4. 不会阻塞，只需要在内存里new Image

54

57 display: none (不占空间，不能点击)（场景，显示出原来这里不存在的结 构）  visibility: hidden（占据空间，不能点击）（场景：显示不会导致页面结 构发生变动，不会撑开）  opacity: 0（占据空间，可以点击）（场景：可以跟 transition 搭配）

61 JWT包括Header（头部).Payload（负载.Signature（签名）

62 redux的本意是状态变更可追溯，如果有副作用的话可能会导致出现不可追溯的问题，而且状态的变更后的比较是通过引用地址的变化来比较的。有副作用会导致整个状态变更不可控

64

65 
```js
Promise.prototype.finally = function(cb){
  let P = this.constructor
  this.then(
    value => P.resolve(cb()).then(() => value),
    reason => P.resolve(cb()).then(() => {throw reason})
  )
}
```

76 对象的键名只能是string或者是Symbol，如果是其他类型，会被隐式转换

79

85

88

89

90

91

93

94