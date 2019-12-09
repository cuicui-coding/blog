
### 图解Event Loop

#### 前言

事件循环（Event Loop），是每个JS开发者都会接触到的概念，但是刚接触时可能会存在各种疑惑。我是一个视觉型学习者，所以打算通过gif动图的可视化形式帮助大家理解它。

首先我们来看看，什么是事件循环，我们为什么要了解它呢？

众所周知，JavaScript是 单线程（single-threaded） 的，也就是同一时间只能运行一个任务。一般情况下这并没有什么问题，但是假如我们要运行一个耗时30秒的任务，我们就得等待30秒后才能执行下一个任务（这30秒期间，JavaScript占用了主线程，我们什么都不能做，包括页面也是卡死状态）。这都9012年了，不带这么坑爹的吧？

好在浏览器向我们提供了JS引擎不具备的特性：Web API。Web API包括DOM API、定时器、HTTP请求等特性，可以帮助我们实现异步、非阻塞的行为。

当我们调用一个函数时，函数会被放入一个叫做调用栈（call stack，也叫执行上下文栈）的地方。调用栈是JS引擎的一部分，并非浏览器特有的。调用栈是一个栈数据结构，具有后进先出的特点（Last in, first out. LIFO）。当函数执行完毕返回时，会被弹出调用栈。

![图片](images/2019-12-4/gid1.6.gif)

图例中的respond函数返回一个setTimeout函数调用，setTimeout函数是Web API提供给我们的功能：它允许我们延迟执行一个任务而不用阻塞主线程。setTimeout被调用时，我们传入的回调函数，即箭头函数 `()=>{return'hey'}`会被传递给Web API处理，然后setTimeout和respond依次执行完毕出栈。

![图片](images/2019-12-4/gif2.1.gif)

在Web API中会执行定时器，定时间隔就是我们传入setTimeout的第二个参数，也就是1000ms。计时结束后回调函数并不会立即进入调用栈执行，而是会被加入一个叫做 任务队列（Task Queue） 的地方。

![图片](images/2019-12-4/gif3.1.gif)

看到这里，有些人可能会疑惑：1000ms之后，回调竟然没有放入调用栈执行，而是被放入了任务队列，那什么时候被执行呢？不要急，既然是一个队列，那就要排排坐，吃果果。

接下来就是我们期待已久，万众瞩目的 事件循环（Event Loop） 闪亮登场的时刻了。**Event Loop的工作就是连接任务队列和调用栈**，当调用栈中的任务均执行完毕出栈，调用栈为空时，Event Loop会检查任务队列中是否存在等待执行的任务，如果存在，则取出队列中第一个任务，放入调用栈。

![图片](images/2019-12-4/gif4.1.gif)

我们的回调函数被放入调用栈中，执行完毕，返回其返回值，然后被弹出调用栈。

![图片](images/2019-12-4/gif5.1.gif)

阅读一时爽，但只有通过反复练习，将其变为自己的东西后才会一直爽。我们来做个小练习检测下学习成果，看看下面代码输出什么：

```
const foo = () => console.log('First');
const bar = () => setTimeout(() => console.log('Second'), 500);
const baz = () => console.log('Third');

bar();
foo();
baz();
```
相信大家都可以轻松给出正确答案。我们一起来看下这段代码运行时发生了什么：

![图片](images/2019-12-4/gif14.1.gif)

1. bar被调用，返回setTimeout的调用；

2. 传入setTimeout的回调被传递给Web API处理，setTimeout执行完毕出栈，bar执行完毕出栈；

3. 定时器开始运行，同时主线程中foo被调用，打印First，foo执行完毕出栈；

4. baz被调用，打印Third，baz执行完毕出栈；

5. 500ms后定时器运行完毕，回调函数被放入任务队列；

6. Event Loop检测到调用栈为空，从任务队列中取出回调函数放入调用栈；

7. 回调函数被执行，打印Second，执行完毕出栈。


#### 关于本文

译文：https://github.com/logan70/Blog/issues/25

原文：https://dev.to/lydiahallie/javascript-visualized-event-loop-3dif