### React Hooks异步操作的二三事

### 前言
React Hooks 是 React 16.8 的新功能，可以在不编写 class 的情况下使用状态等功能，从而使得函数式组件从无状态的变化为有状态的。React 的类型包 @types/react 中也同步把 React.SFC (Stateless Functional Component) 改为了 React.FC (Functional Component)。

通过这一升级，原先 class 写法的组件也就完全可以被函数式组件替代。虽然是否要把老项目中所有类组件全部改为函数式组件因人而异，但新写的组件还是值得尝试的，因为代码量的确减少了很多，尤其是重复的代码（例如 componentDidMount + componentDidUpdate + componentWillUnmount = useEffect）。

从 16.8 发布（今年2月）至今也有大半年了，但本人水平有限，尤其在 useEffect 和异步任务搭配使用的时候经常踩到一些坑。特作本文，权当记录，供遇到同样问题的同僚借鉴参考。我会讲到三个项目中非常常见的问题：

- 如何在组件加载时发起异步任务

- 如何在组件交互时发起异步任务

- 其他陷阱

### 关键点
- 使用 useEffect 发起异步任务，第二个参数使用空数组可实现组件加载时执行方法体，返回值函数在组件卸载时执行一次，用来清理一些东西，例如计时器。

- 使用 AbortController 或者某些库自带的信号量 ( axios.CancelToken) 来控制中止请求，更加优雅地退出。

- 当需要在其他地方（例如点击处理函数中）设定计时器，在 useEffect 返回值中清理时，使用局部变量或者 useRef 来记录这个 timer。不要使用 useState。

- 组件中出现 setTimeout 等闭包时，尽量在闭包内部引用 ref 而不是 state，否则容易出现读取到旧值的情况。

- useState 返回的更新状态方法是异步的，要在下次重绘才能获取新值。不要试图在更改状态之后立马获取状态。

### 如何在组件加载时发起异步任务
这类需求非常常见，典型的例子是在列表组件加载时发送请求到后端，获取列表后展现。

发送请求也属于 React 定义的副作用之一，因此应当使用 useEffect 来编写。基本语法我就不再过多说明，代码如下：
```
const SOME_API = '/api/get/value';

export const MyComponent: React.FC<{}> = () => {
    const [loading, setLoading] = useState(true);
    const [value, setValue] = useState(0);

    useEffect(() => {
        (async () => {
            const res = await fetch(SOME_API);
            const data = await res.json();
            setValue(data.value);
            setLoading(false);
        })();
    }, []);

    return (
        <>
        {loading ? (
            <h2>Loading...</h2>
        ) : (
            <h2>value is {value}</h2>
        )}
        </>
    );
}

```

如上是一个基础的带 Loading 功能的组件，会发送异步请求到后端获取一个值并显示到页面上。如果以示例的标准来说已经足够，但要实际运用到项目中，还不得不考虑几个问题。

#### 如果在响应回来之前组件被销毁了会怎样？
React 会报一个 Warning

> Warning: Can't perform a React state update on an unmounted component. This is a no-op, but it indicates a memory leak in your application. To fix, cancel all subscriptions and asynchronous tasks in a useEffect cleanup http://function.in Notification

大意是说在一个组件卸载了之后不应该再修改它的状态。虽然不影响运行，但作为完美主义者代表的程序员群体是无法容忍这种情况发生的，那么如何解决呢？

问题的核心在于，在组件卸载后依然调用了 setValue(data.value) 和 setLoading(false) 来更改状态。因此一个简单的办法是标记一下组件有没有被卸载，可以利用 useEffect 的返回值。

和这个例子相比，还有很多其他的场景更难辨别和分析，比如，你在 useEffect 里调用了某个函数(函数定义在 useEffect 外面)，但是却 没有 在依赖项里添加这个useEffect函数，那么代码很可能有bug了。相信我，每次在我忽略了这条规则之后，我都会后悔当初为什么没有遵守规则。

后面只是列举了这个例子，暂时先忽略这条规则：// eslint-disable-next-line react-hooks/exhaustive-deps

```
// 省略组件其他内容，只列出 diff
useEffect(() => {
    let isUnmounted = false;
    (async () => {
        const res = await fetch(SOME_API);
        const data = await res.json();
        if (!isUnmounted) {
            setValue(data.value);
            setLoading(false);
        }
    })();

    return () => {
        isUnmounted = true;
    }
}, []);
```
#### 有没有更加优雅的解法？
上述做法是在收到响应时进行判断，即无论如何需要等响应完成，略显被动。一个更加主动的方式是探知到卸载时直接中断请求，自然也不必再等待响应了。这种主动方案需要用到 AbortController。

AbortController 是一个浏览器的实验接口，它可以返回一个信号量(singal)，从而中止发送的请求。这个接口的兼容性不错，除了 IE 之外全都兼容（如 Chrome, Edge, FF 和绝大部分移动浏览器，包括 Safari）。
```
useEffect(() => {
    let isUnmounted = false;
    const abortController = new AbortController(); // 创建
    (async () => {
        const res = await fetch(SOME_API, {
            singal: abortController.singal, // 当做信号量传入
        });
        const data = await res.json();
        if (!isUnmounted) {
            setValue(data.value);
            setLoading(false);
        }
    })();

    return () => {
        isUnmounted = true;
        abortController.abort(); // 在组件卸载时中断
    }
}, []);
```

singal 的实现依赖于实际发送请求使用的方法，如上述例子的 fetch 方法接受 singal 属性。如果使用的是 axios，它的内部已经包含了 axios.CancelToken，可以直接使用，例子在这里。

### 如何在组件交互时发起异步任务
另一种常见的需求是要在组件交互（比如点击某个按钮）时发送请求或者开启计时器，待收到响应后修改数据进而影响页面。这里和上面一节（组件加载时）最大的差异在于 React Hooks 只能在组件级别编写，不能在方法（ dealClick）或者控制逻辑（ if, for 等）内部编写，所以不能在点击的响应函数中再去调用 useEffect。但我们依然要利用 useEffect 的返回函数来做清理工作。

以计时器为例，假设我们想做一个组件，点击按钮后开启一个计时器(5s)，计时器结束后修改状态。但如果在计时未到就销毁组件时，我们想停止这个计时器，避免内存泄露。用代码实现的话，会发现开启计时器和清理计时器会在不同的地方，因此就必须记录这个 timer。看如下的例子：

如果把timer作为内部变量来存储，会有什么问题？
```
import React, { useState, useEffect } from 'react';

export default function MyComponent() {
    const [value, setValue] = useState(0);

    let timer=null
    console.log('timer初始值:',timer)

    useEffect(() => {
      console.log('调用useEffect方法：', 'timer=', timer,' value=',value)
        // timer 需要在点击时建立，因此这里只做清理使用
        return () => {
            console.log('in useEffect return: ', timer); // <- 这个timer跟你点击的时间间隔有关，且都是第一次重渲染的timer的值。
            window.clearTimeout(timer);
        }
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, []);

    function dealClick() {
        timer = window.setTimeout(() => {
            setValue(value + 100);
        }, 5000);
        console.log('点击之后timer值为: ',timer)
    }

    return (
        <>
            <span>Value is {value}</span>
            <button onClick={dealClick}>Click Me!</button>
        </>
    );
}
```
既然要记录 timer，自然是用一个内部变量来存储即可（暂不考虑连续点击按钮导致多个 timer 出现，假设只点一次，卸载组件时能取到第一次timer的值）。

因为useEffect依赖了timer, 会警告提示`React Hook useEffect has a missing dependency: 'timer'. Either include it or remove the dependency array  react-hooks/exhaustive-deps`，因此我们避免出现警告，添加注释代码，` // eslint-disable-next-line react-hooks/exhaustive-deps`。


如果多次点击带来的问题

- useEffect的第二个参数是[]空数组，那么组件只会在挂载阶段执行，返回的回调函数中，timer就是挂载之后取到的最新timer值，也就是第一次重绘组件之前，timer的最新值，后续多次更新重绘组件，都是第一次重绘之前的值。因为闭包的原理。
- 对于函数组件，每次重绘组件，就会从头重新执行一次, timer就会被重新赋值为null。（如果放在类组件的实例变量timer，无论挂载还是更新，timer都只是唯一的一个）。
- 如果点击之后，且在5s之内，也就是说组件卸载之后, setTimeout的回调函数中执行setValue设置状态，就会警告不要在卸载组件之后修改状态值的提示，如果点击5s之后，setTimeout已经执行完毕，再卸载组件就不会报错。

```
timer初始值: null
调用useEffect方法： timer= null  value= 0
点击之后timer值为:  7
timer初始值: null
点击之后timer值为:  9
timer初始值: null
点击之后timer值为:  11
in useEffect return:  7
(卸载组件后，setTimeout回调修改状态，导致报错，如果卸载组件之前，setTimeout就已经执行完毕就不会报错)
Warning: Can't perform a React state update on an unmounted component. This is a no-op, but it indicates a memory leak in your application. To fix, cancel all subscriptions and asynchronous tasks in a useEffect cleanup function.
    in MyComponent (at App.js:11)
```
第一种写法代码是把 timer 作为组件内的局部变量使用。在初次渲染组件时， useEffect 返回的闭包函数中指向了这个局部变量 timer。在 dealClick 中设置计时器时返回值依旧写给了这个局部变量（即读和写都是同一个变量），因此在后续卸载时，虽然组件重新运行导致出现一个新的局部变量 timer，但这不影响闭包内老的 timer，所以还是取的老的timer值。

分析useEffect第二个参数，如果不设置，就会每次render都会执行，如果设置为[],就会在mount阶段执行一次，如果设置为[value], 每次render时，只会value值变化，useEffect就会重新执行。
```
useEffect(() => {
      console.log('调用useEffect方法：', 'timer=', timer,' value=',value)
        // timer 需要在点击时建立，因此这里只做清理使用
        return () => {
            console.log('in useEffect return: ', timer); // <- 这个timer跟你点击的时间间隔有关，且都是第一次重渲染的timer的值。
            window.clearTimeout(timer);
        }
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [value]);
```

会出现两种情况
- 更新之后，不点击按钮，再卸载组件，timer是null。因为渲染时，函数组件重新执行，timer被赋值为null。

- 更新之后，点击按钮，再卸载组件, timer就是这次点击的值，能成功取消定时器。
```
timer初始值: null
index.js:10 调用useEffect方法： timer= null  value= 0
index.js:23 点击之后timer值为:  7
index.js:7 timer初始值: null
index.js:13 in useEffect return:  7
index.js:10 调用useEffect方法： timer= null  value= 100
index.js:23 点击之后timer值为:  9
index.js:13 in useEffect return:  9
```

这里需要注意的是，如果把 timer 升级为状态(state)，看看会出现问题。考虑如下代码：

```
import React, { useState, useEffect } from 'react';

export default function MyComponent  () {
    const [value, setValue] = useState(0);
    const [timer, setTimer] = useState(null); // 把 timer 升级为状态

    useEffect(() => {
       console.log('调用useEffect方法: ', timer)
        // timer 需要在点击时建立，因此这里只做清理使用
        return () => {
            console.log('in useEffect return: ', timer); // <- 正确的值
            window.clearTimeout(timer);
        }
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, []);

    function dealClick() {
      
        const tmp = window.setTimeout(() => {
            setValue(value + 100);
        }, 5000);

        setTimer(tmp);
        console.log('点击之后timer值为：', timer)
    }

    return (
        <>
            <span>Value is {value}</span>
            <button onClick={dealClick}>Click Me!</button>
        </>
    );
}

```

有关语义上 timer 到底算不算作组件的状态我们先抛开不谈，仅就代码层面来看。利用 useState 来记住 timer 状态，利用 setTimer 去更改状态，看似合理。但实际运行下来，由于 useEffect的第二个参数是[], 只在挂载阶段运行，返回的清理函数中，得到的 timer 永远是初始值，即 null。

发现的问题
- setTimer修改状态是异步操作，timer并未立即更新
- useEffect只是在第一次挂载时执行，更新的时候，不执行，卸载的时候，执行返回的函数。

```
调用useEffect方法： null
点击之后timer值为： null
点击之后timer值为： 7
点击之后timer值为： 9
in useEffect return:  null
(点击超过5s之后，卸载组件就不会报错)
Warning: Can't perform a React state update on an unmounted component. This is a no-op, but it indicates a memory leak in your application. To fix, cancel all subscriptions and asynchronous tasks in a useEffect cleanup function.
    in MyComponent (at App.js:11)
```

为什么两种写法会有差异呢？

其核心在于写入的变量和读取的变量是否是同一个变量。

第二种写法， timer 是一个 useState 的返回值，并不是一个简单的变量。从 React Hooks 的源码来看，它返回的是 [hook.memorizedState,dispatch]，对应我们接的值和变更方法。当调用 setTimer 和 setValue 时，分别触发两次重绘，使得 hook.memorizedState 指向了 newState（注意：不是修改，而是重新指向）。但 useEffect 返回闭包中的 timer 依然指向旧的状态，从而得不到新的值。（即读的是旧值，但写的是新值，不是同一个）

如果觉得阅读 Hooks 源码有困难，可以从另一个角度去理解：虽然 React 在 16.8 推出了 Hooks，但实际上只是加强了函数式组件的写法，使之拥有状态，用来作为类组件的一种替代，但 React 状态的内部机制没有变化。在 React 中 setState 内部是通过 merge 操作将新状态和老状态合并后，重新返回一个新的状态对象。不论 Hooks 写法如何，这条原理没有变化。现在闭包内指向了旧的状态对象，而 setTimer 和 setValue 重新生成并指向了新的状态对象，并不影响闭包，导致了闭包读不到新的状态。

但是我们修改useEffect的第二个参数：

```
import React, { useState, useEffect } from 'react';

export default function MyComponent  () {
    const [value, setValue] = useState(0);
    const [timer, setTimer] = useState(null); // 把 timer 升级为状态

    useEffect(() => {
        console.log('调用useEffect方法：','timer=', timer, 'value=',value)
        // timer 需要在点击时建立，因此这里只做清理使用
        return () => {
            console.log('in useEffect return: ', timer); // <- 正确的值
            window.clearTimeout(timer);
        }
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [value]); // 第一个参数改为value，每次value变化，useEffect就会重新执行，timer就会取最新的值

    function dealClick() {
      
        const tmp = window.setTimeout(() => {
            setValue(value + 100);
        }, 5000);

        setTimer(tmp);
        console.log('点击之后timer值为：', timer)
    }

    return (
        <>
            <span>Value is {value}</span>
            <button onClick={dealClick}>Click Me!</button>
        </>
    );
}

```
我们把useEffect的第二个参数由[]改为[value]，每次value变化，useEffect就会重新执行, 那返回的回调函数就拿到最新的timer值。
```
调用useEffect方法： timer= null value= 0
点击之后timer值为： null
in useEffect return:  null
调用useEffect方法： timer= 7 value= 100
点击之后timer值为： 7
in useEffect return:  7
调用useEffect方法： timer= 9 value= 200
(等5s之后，点击卸载组件，就不会报错，卸载修改状态)
index.js:11 in useEffect return:  9
```

我们注意到 React 还提供给我们一个 useRef， 它的定义是

> useRef 返回一个可变的 ref 对象，其 current 属性被初始化为传入的参数（initialValue）。返回的 ref 对象在组件的整个生命周期内保持不变。

ref 对象可以确保在整个生命周期中值不变，且同步更新，是因为 ref 的返回值始终只有一个实例，所有读写都指向它自己。所以也可以用来解决这里的问题。

```
import React, { useState, useEffect, useRef } from 'react';

export const MyComponent: React.FC<{}> = () => {
    const [value, setValue] = useState(0);
    const timer = useRef(0);

    useEffect(() => {
        // timer 需要在点击时建立，因此这里只做清理使用
        return () => {
            window.clearTimeout(timer.current);
        }
    }, []);

    function dealClick() {
        timer.current = window.setTimeout(() => {
            setValue(100);
        }, 5000);
    }

    return (
        <>
            <span>Value is {value}</span>
            <button onClick={dealClick}>Click Me!</button>
        </>
    );
}
```
事实上我们后面会看到， useRef 和异步任务配合更加安全稳妥。

### 其他陷阱
#### 修改状态是异步的
这个其实比较基础了。

```
import React, { useState } from 'react';

export const MyComponent: React.FC<{}> = () => {
    const [value, setValue] = useState(0);

    function dealClick() {
        setValue(100);
        console.log(value); // <- 0
    }

    return (
        <span>Value is {value}, AnotherValue is {anotherValue}</span>
    );
}
```
useState 返回的修改函数是异步的，调用后并不会直接生效，因此立马读取 value 获取到的是旧值（ 0）。

React 这样设计的目的是为了性能考虑，争取把所有状态改变后只重绘一次就能解决更新问题，而不是改一次重绘一次，也是很容易理解的。

#### 在 timeout 中读不到其他状态的新值
```
import React, { useState, useEffect } from 'react';

export const MyComponent: React.FC<{}> = () => {
    const [value, setValue] = useState(0);
    const [anotherValue, setAnotherValue] = useState(0);

    useEffect(() => {
        window.setTimeout(() => {
            console.log('setAnotherValue', value) // <- 0
            setAnotherValue(value);
        }, 1000);
        setValue(100);
    }, []);

    return (
        <span>Value is {value}, AnotherValue is {anotherValue}</span>
    );
}
```
这个问题和上面使用 useState 去记录 timer 类似，在生成 timeout 闭包时，value 的值是 0。虽然之后通过 setValue 修改了状态，但 React 内部已经指向了新的变量，而旧的变量仍被闭包引用，所以闭包拿到的依然是旧的初始值，也就是 0。

要修正这个问题，也依然是使用 useRef，如下：

```
import React, { useState, useEffect, useRef } from 'react';

export const MyComponent: React.FC<{}> = () => {
    const [value, setValue] = useState(0);
    const [anotherValue, setAnotherValue] = useState(0);
    const valueRef = useRef(value);
    valueRef.current = value;

    useEffect(() => {
        window.setTimeout(() => {
            console.log('setAnotherValue', valueRef.current) // <- 100
            setAnotherValue(valueRef.current);
        }, 1000);
        setValue(100);
    }, []);

    return (
        <span>Value is {value}, AnotherValue is {anotherValue}</span>
    );
}
```
#### 还是 timeout 的问题
假设我们要实现一个按钮，默认显示 false。当点击后更改为 true，但两秒后变回 false（ true 和 false 可以互换）。考虑如下代码：

```
import React, { useState } from 'react';

export const MyComponent: React.FC<{}> = () => {
    const [flag, setFlag] = useState(false);

    function dealClick() {
        setFlag(!flag);

        setTimeout(() => {
            setFlag(!flag);
        }, 2000);
    }

    return (
        <button onClick={dealClick}>{flag ? "true" : "false"}</button>
    );
}
```
我们会发现点击时能够正常切换，但是两秒后并不会变回来。究其原因，依然在于 useState 的更新是重新指向新值，但 timeout 的闭包依然指向了旧值。所以在例子中， flag 一直是 false，虽然后续 setFlag(!flag)，但依然没有影响到 timeout 里面的 flag。

解决方法有二。

第一个还是利用 useRef
```
import React, { useState, useRef } from 'react';

export const MyComponent: React.FC<{}> = () => {
    const [flag, setFlag] = useState(false);
    const flagRef = useRef(flag);
    flagRef.current = flag;

    function dealClick() {
        setFlag(!flagRef.current);

        setTimeout(() => {
            setFlag(!flagRef.current);
        }, 2000);
    }

    return (
        <button onClick={dealClick}>{flag ? "true" : "false"}</button>
    );
}
```
第二个是利用 setFlag 可以接收函数作为参数，并利用闭包和参数来实现
```
import React, { useState } from 'react';

export const MyComponent: React.FC<{}> = () => {
    const [flag, setFlag] = useState(false);

    function dealClick() {
        setFlag(!flag);

        setTimeout(() => {
            setFlag(flag => !flag);
        }, 2000);
    }

    return (
        <button onClick={dealClick}>{flag ? "true" : "false"}</button>
    );
}
```
当 setFlag 参数为函数类型时，这个函数的意义是告诉 React 如何从当前状态产生出新的状态（类似于 redux 的 reducer，不过是只针对一个状态的子 reducer）。既然是当前状态，因此返回值取反，就能够实现效果。

### 总结
在 Hook 中出现异步任务尤其是 timeout 的时候，我们要格外注意。useState 只能保证多次重绘之间的状态值是一样的，但不保证它们就是同一个对象，因此出现闭包引用的时候，尽量使用 useRef 而不是直接使用 state 本身，否则就容易踩坑。反之如果的确碰到了设置了新值但读取到旧值的情况，也可以往这个方向想想，可能就是这个原因所致。

### 参考文章
- [React Hooks异步操作二三事](https://mp.weixin.qq.com/s?__biz=MjM5MTA1MjAxMQ==&mid=2651234326&idx=2&sn=ee3c1fc837773eb178744499d67704c1)

- How to create React custom hooks for data fetching with useEffect

- setTimeout in React Components Using Hooks

- React - useState - why setTimeout function does not have latest state value?



