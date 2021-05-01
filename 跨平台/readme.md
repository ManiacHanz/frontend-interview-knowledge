* APP内嵌H5⻚⾯如何和APP本⾝进⾏通信 
* 微信⼩程序和传统h5⻚⾯相⽐哪个性能更好⼀些，为什么
* H5的开发和PC端的开发有什么本质的不同 
* 如何针对H5⻚⾯进⾏远程调试？ 
* 如何进⾏⼿机的⻚⾯或者应⽤的抓包


* 移动端300ms延时的原因? 如何处理?

300ms延时是为了移动端浏览器检测双击事件，尤其是双击放大/缩小事件

fastclick处理是在检测到touchend触发以后添加event.preventDefault(). 然后在document.createEvent去手动创建一个事件，并通过eventTarget.dispatchEvent去触发绑定在目标元素上的click函数