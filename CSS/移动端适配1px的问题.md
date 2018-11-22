
# 移动端适配1px的问题

*仅供参考，欢迎指正*

**首先感谢各位大佬的分享，很多概念是从手淘的[大漠](https://github.com/airen)大大的文章中学习并直接盗用过来的。[flexible的终端适配](https://github.com/amfe/article/issues/17)，以及[vw移动适配](https://www.w3cplus.com/mobile/vw-layout-in-vue.html)**

主要是利用新的`vw`特性。然后利用一系列插件解决`vw`的兼容性，以及`vw`和`px`之间的转化
另外就是小于`1px`的边框等，可以利用`border-image`属性换成图片，或者是利用`svg`保证在各个移动端屏幕表现一致

**todo...这边本来想写，但是发现没什么自己的时间经验，全从大佬文章里抄过来也没太大意思，所以暂时先不放了，内容在上面两篇文章里很详细了**