### Vue3.0有哪些地方优化

* 源码优化
   - monorepo。增加代码解耦
   - 改用typescript
* 性能优化
   - 减少无用api，更好支持tree-shaking
   - 编译优化。增加静态动态标记，减少diff
   - 响应式api优化，从Object.defineProperty改到Proxy，深对象是通过getter动态递归绑定响应式，也就是访问到了对象才把内部对象改成响应式
* API优化
   - 增加compisition Api


### Vue3.0 diff

Vue 3的diff逻辑主要在`patch`方法中。概括来说，这里面有主要的是通过shapeFlag判断vnode类型，然后处理对应的process过程。
以组件来说，会首先判断`shouldUpdateComponent`，这里是根据vnode上的一些信息，props、children、dirs、transition等等来决定。如果需要更新就把对应的这些属性更新。
如果是处理普通元素，主要会落到两点，更新属性和子节点。属性通过dom操作就可以更新，子节点就分为三种类型：空、文本节点、数组节点。这里面只有新旧节点都是数组节点的情况才会进行diff。diff的过程有一部分和2.0一样，就是同样需要分别从头部指针进行遍历，找到可复用的元素，如果是相同的节点，就递归调用patch去更新。当进行完这两步以后又会有三种情况：1. 假如已经遍历完所有旧节点但是没有遍历完新节点，就代表需要新增节点；2. 假如已经遍历完所有新节点但是没有遍历完旧节点，就代表需要删除节点；3. 都没有遍历完，就代表有比较复杂的操作，比如移动子元素，这里涉及到判断 哪些是可移动的，如何移动代价最小。vue3.0会首先以新序列的元素简历索引图`keyToNewIndexMap`，key是元素的key，value是元素的索引。然后建立一个数组，长度为新子序列变化的这部分长度，每个位置初始值为0，用于存放新子序列和旧子序列元素索引的映射关系`newIndexToOldIndexMap`。然后遍历旧序列，如果索引里没有，就是需要删除的元素，如果有，就是可以复用的需要移动的元素，就存在这个映射数组里面（这个数组里面的下标，就是当前元素在新序列里面的下标，值就是在旧元素里的下标）。如果遍历完了映射数组里有值为0，那这个元素就是新增元素。遍历过程中，vue用`maxNewIndexSoFar`来记录上次求值得到的在新序列里面的索引，如果这个值不是一直递增，则说明有元素需要移动，此时设置标识`move`为true，标识有节点需要移动，进入移动步骤。所谓移动，vue3.0中采用的是求解最长递归子序列的算法，也就是说，如果在倒序遍历的过程中，如果元素的索引在这个最长递归子序列中，则代表不需要移动，如果不在，则表示当前节点需要移动

流程图如下

patch方法调用 -> processElement对比元素 -> 文本、空白、数组节点 -> diff数组节点 -> 头部指针遍历 -> 尾部指针遍历 -> 新增或删除元素结束/ 未知元素比较 -> 简历新节点索引map -> 遍历旧节点，建立新旧索引关系，并判断有没有移动 -> 有移动则使用最长递增子序列算法，求得索引关系的数组里的最长递增子序列的下标数组 -> 如果新节点的下标在最长递增子序列中则更新即可，不在就需要移动元素


### Vue3 Composition Api

主流程： 判断组件是否有setup函数 -> 有就创建渲染上下文代理 -> 创建setup函数上下文 -> 执行setup函数 -> 处理结果变成响应式对象 -> 

有就创建渲染上下文代理： createSetupContext 返回组件实例的属性attr，插槽slot，以及事件触发方法emit

### Vue3响应式原理

*vue2.0* 通过Object.defineProperty建立响应式数据, 在getter中收集依赖，在setter中触发依赖。依赖的收集和触发主要是通过Dep类，每监听一个响应式数据的时候会新建一个Dep实例，在这个实例中收集这个响应式数据对应的Watcher实例，watcher是~~在compiler编译模板的时候，实例化的（1.0是以模板编译）~~ mountComponent时候实例化的，这里包含了根据nodetype在数据变化后如何处理的回调。Dep负责收集这些watcher，并且在setter中通知到所有watcher


#### 收集依赖
Vue3.0 使用reactive 把数据变成响应式。核心就是使用Proxy api。劫持了主要以下几个操作
* 访问对象属性会触发 get 函数；
* 设置对象属性会触发 set 函数；
* 删除对象属性会触发 deleteProperty 函数；
* in 操作符会触发 has 函数；
* 通过 Object.getOwnPropertyNames 访问对象属性名会触发 ownKeys 函数。

同时在getter中使用`track`函数进行依赖收集。这里涉及到vue3的依赖图。

设定target是原始数据，也就是要被proxy的那个数据。最外层是全局targetMap作为原始数据对象的Map，它的键是target，值是depsMap。 depsMap的键是target数据里的key，值是dep集合。dep集合里存的是数据更新后的副作用函数

也就是通过target对象找到target对象的key，在找到对应的副作用函数集

#### 派发通知
Vue3 也是在setter里派发通知，使用`trigger`函数派发

trigger的原理就是上面说的，通过target拿到depsMap，然后再通过key拿到所有的effect集合。这里面有一点就是trigger是新建里一个effects数组，把所有的effect全保存下来再触发的