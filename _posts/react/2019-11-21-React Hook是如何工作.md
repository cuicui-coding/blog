### 深入理解：React hooks是如何工作的

#### 前言

从根本上说，Hooks是一种更简单的方式，用于封装用户界面中的有状态行为和副作用。React最先引入了Hooks，现在其他框架如Vue，Svelte都广泛实现了该功能，TNG-Hooks甚至可以为常规的JS函数提供Hooks。然而它们的函数式设计需要对Javascript里的闭包有很好的理解。

在本文，我们将使用闭包实现一个React Hooks的微型版本。这么做有两个目的，一是演示闭包的效用，二是如何使用29行易读的JS代码实现Hooks。最后我们可以很自然的得到自定义Hooks。

#### 闭包是什么？

Hooks的许多卖点之一是避免了类和高阶组件的复杂性。然而有些人觉得Hooks可能会导致另外的问题。虽然不需要再担心绑定上下文，现在我们需要担心闭包。正如Mark Dalgleish令人难忘的总结。

闭包是JS里的基本概念。众所周知对许多初学的开发者来说它们令人费解。Kyle Simpson在《你不知道的JS》中对闭包的著名定义如下：

闭包是：当一个函数在它的词法作用域之外执行的时候，仍然可以记得它的词法作用域且可以访问该作用域。

闭包很显然和词法作用域的概念紧密相关，在MDN是这么描述词法作用域的：“当函数被嵌套时，解析器解析函数的变量名的方式”。让我们来看一个实际的例子，可以更好的说明这一点：

```
  // Example 0
  function useState(initialValue) {
    var _val = initialValue //_val是useState创建的局部变量
    function state() {
      // state 是一个内部函数, 也是一个闭包
      return _val // state() 使用了_val, 该变量由父函数声明
    }
    function setState(newVal) {
      // 同样是内部函数
      _val = newVal // 给_val赋值，而不用暴露_val
    }
    return [state, setState] //将这两个函数暴露到外部
  }
  var [foo, setFoo] = useState(0) // 使用了数组解构方法
  console.log(foo()) // logs 0 - 我们给的初始值
  setFoo(1) // 在useState的作用域内给_val赋值
  console.log(foo()) // logs 1 - 尽管使用了相同的函数调用，得到的是新的初始值

```
这里，我们建立了React的 useState hook的原始版本。这里有2个内部函数，state和setState。state返回了上面定义的局部变量_val，setState将传给它的参数（即newVal）赋给该局部变量。

我们的state用getter函数实现，这并不完美，但我们将对此进行改进。这里重要的是，使用foo和setFoo，我们可以访问和操作（即所谓的“封闭”）内部变量_val。这两个函数保留了useState作用域的访问权，而这样的引用被称为闭包。放在在React和其他框架的上下文中，这看起来好像状态，实际上正是如此。

#### 在函数式组件中的使用

让我们来把刚做出的useState功能应用到常见的程序中。下面来做一个计数器组件！

```
// Example 1
function Counter() {
    const [count, setCount] = useState(0) // 和上面定义的 useState 相同
    return {
      click: () => setCount(count() + 1),
      render: () => console.log('render:', { count: count() })
    }
  }
  const C = Counter()
  C.render() // render: { count: 0 }
  C.click()
  C.render() // render: { count: 1 }
```

这里我们没有把数据渲染到DOM上，而是仅在控制台输出这些状态。我们让计数器提供一个外部API，这样我们可以直接运行脚本，而不必给它设置click事件处理函数。

虽然这种做法也可以工作起来，调用getter函数来访问状态并不是React.useState hook的实际做法。我们来改进它。

#### 不能更新状态的闭包实现
如果我们想要做得和实际的React hook一样，状态就应该是一个变量，而不是函数。如果我们简单的将_val暴露出去，而不是将它包裹在函数里面，就会出现bug：

```
// Example 0, 再来看第一个例子 - 这么做是有bug的!
function useState(initialValue) {
    var _val = initialValue
    // 不使用state()函数
    function setState(newVal) {
      _val = newVal
    }
    return [_val, setState] // 直接对外暴露_val
  }
  var [foo, setFoo] = useState(0)
  console.log(foo) // logs 0 不需要进行函数调用
  setFoo(1) // 在useState作用域内给_val赋值
  console.log(foo) // logs 0 - 糟糕!!
```
这是种闭包不能更新的问题。当我们从useState的输出中解构出foo变量时，foo的值等于对useState初始调用时的_val值，之后就不会再变了！这不是我们想要的结果，通常我们需要让组件的状态能反映出当前的状态，而且状态应该是一个变量而不是一个函数！这两个目标看起来不可兼得。

#### 模块模式的闭包实现
我们可以解决这一useState难题……通过将闭包放进另一个闭包中！（我的天！听说你喜欢闭包……）
```
// Example 2
const MyReact = (function() {
    let _val // 将我们的状态保持在模块作用域中
    return {
      render(Component) {
        const Comp = Component()
        Comp.render()
        return Comp
      },
      useState(initialValue) {
        _val = _val || initialValue // 每次运行都重新赋值
        function setState(newVal) {
          _val = newVal
        }
        return [_val, setState]
      }
    }
  })()
```
这里我们选择使用模块模式来制作我们的微型React hook。像React一样，它可以记录组件的状态（在这个例子中，它只能给每个组件记录一个状态，将状态记录在_val中）。该设计允许MyReact渲染你的函数式组件，它可以在每次组件更新时使用和它相应的闭包，对内部的_val赋值。

```
// 续Example 2 
function Counter() {
    const [count, setCount] = MyReact.useState(0)
    return {
      click: () => setCount(count + 1),
      render: () => console.log('render:', { count })
    }
  }
  let App
  App = MyReact.render(Counter) // render: { count: 0 }
  App.click()
  App = MyReact.render(Counter) // render: { count: 1 }
```
现在看起来更像React里的Hooks了。

#### 复制useEffect功能
目前为止，我们已经实现了useState，这是最基本的React Hook。下一个重要的Hook是useEffect。不像setState，useEffect是异步执行的，这意味着更容易遇到闭包问题。

我们可以扩展这个微型React模型，加入下面代码：
```
// Example 3
const MyReact = (function() {
    let _val, _deps // 在作用域内保持状态和依赖
    return {
      render(Component) {
        const Comp = Component()
        Comp.render()
        return Comp
      },
      useEffect(callback, depArray) {
        const hasNoDeps = !depArray
        const hasChangedDeps = _deps ? !depArray.every((el, i) => el === _deps[i]) : true
        if (hasNoDeps || hasChangedDeps) {
          callback()
          _deps = depArray
        }
      },
      useState(initialValue) {
        _val = _val || initialValue
        function setState(newVal) {
          _val = newVal
        }
        return [_val, setState]
      }
    }
  })()
  
  // 使用方法
  function Counter() {
    const [count, setCount] = MyReact.useState(0)
    MyReact.useEffect(() => {
      console.log('effect', count)
    }, [count])
    return {
      click: () => setCount(count + 1),
      noop: () => setCount(count),
      render: () => console.log('render', { count })
    }
  }
  let App
  App = MyReact.render(Counter)
  // effect 0
  // render {count: 0}
  App.click()
  App = MyReact.render(Counter)
  // effect 1
  // render {count: 1}
  App.noop()
  App = MyReact.render(Counter)
  // // no effect run
  // render {count: 1}
  App.click()
  App = MyReact.render(Counter)
  // effect 2
  // render {count: 2}
```
为了追踪依赖项（因为useEffect只有在依赖项发生变化才会重新运行callback），我们引入了另一个变量_deps。

#### 没有魔法，只是数组而已
我们已经很好的复制了useState和useEffect的功能，但是它们是实现得很差的单态（只允许一个状态，一个副作用，多了就会有bug）。为了让事情变得更有意思，我们需要扩展它使之可以接受任意数量的状态和副作用。幸运的是，正如Rudi Yardley所写的，React Hooks不是什么魔法，仅仅是数组而已。因此我们会使用到一个hooks数组。我们把_val和_deps全都放在同一个数组中，因为它们是互不干扰的。
```
// Example 4
const MyReact = (function() {
    let hooks = [],
      currentHook = 0 // hooks数组, 和一个iterator!
    return {
      render(Component) {
        const Comp = Component() // 运行 effects
        Comp.render()
        currentHook = 0 // 复位，为下一次render做准备
        return Comp
      },
      useEffect(callback, depArray) {
        const hasNoDeps = !depArray
        const deps = hooks[currentHook] // type: array | undefined
        const hasChangedDeps = deps ? !depArray.every((el, i) => el === deps[i]) : true
        if (hasNoDeps || hasChangedDeps) {
          callback()
          hooks[currentHook] = depArray
        }
        currentHook++ // 本hook运行结束
      },
      useState(initialValue) {
        hooks[currentHook] = hooks[currentHook] || initialValue // type: any
        const setStateHookIndex = currentHook // 给setState的闭包准备的变量!
        const setState = newState => (hooks[setStateHookIndex] = newState)
        return [hooks[currentHook++], setState]
      }
    }
  })()
```
注意我们这里使用的setStateHookIndex变量，看起来好像没有什么用，但它是用来避免setState将currentHook直接封闭进去！如果你直接使用currentHook，setState功能不会正常工作，因为currentHook在每次render后都被复位为0，之后再调用setState则每次都将修改hook数组的第一项。
```
 // Example 4 续 - 使用hook
function Counter() {
    const [count, setCount] = MyReact.useState(0)
    const [text, setText] = MyReact.useState('foo') // 第二个 state hook!
    MyReact.useEffect(() => {
      console.log('effect', count, text)
    }, [count, text])
    return {
      click: () => setCount(count + 1),
      type: txt => setText(txt),
      noop: () => setCount(count),
      render: () => console.log('render', { count, text })
    }
  }
  let App
  App = MyReact.render(Counter)
  // effect 0 foo
  // render {count: 0, text: 'foo'}
  App.click()
  App = MyReact.render(Counter)
  // effect 1 foo
  // render {count: 1, text: 'foo'}
  App.type('bar')
  App = MyReact.render(Counter)
  // effect 1 bar
  // render {count: 1, text: 'bar'}
  App.noop()
  App = MyReact.render(Counter)
  // // no effect run
  // render {count: 1, text: 'bar'}
  App.click()
  App = MyReact.render(Counter)
  // effect 2 bar
  // render {count: 2, text: 'bar'}
```
所以基本的思路是使用数组存放hook的状态和依赖，调用每个hook只需增加索引号操作相应的数组项，当组件render完毕后复位索引。

还可以很容易的实现自定义hooks：
```
// Example 4, revisited
function Component() {
    const [text, setText] = useSplitURL('www.netlify.com')
    return {
      type: txt => setText(txt),
      render: () => console.log({ text })
    }
  }
  function useSplitURL(str) {
    const [text, setText] = MyReact.useState(str)
    const masked = text.split('.')
    return [masked, setText]
  }
  let App
  App = MyReact.render(Component)
  // { text: [ 'www', 'netlify', 'com' ] }
  App.type('www.reactjs.org')
  App = MyReact.render(Component)
  // { text: [ 'www', 'reactjs', 'org' ] }}
```
这就是hooks的实际原理，自定义hooks只需简单的利用框架提供的原语，不管是React里的还是我们这里制作的微型hook版本都是如此。

#### 使用hook的法则
现在你可以很容易理解使用Hooks的第一个法则：只在最顶层调用Hooks。因为我们使用currentHook变量，需要根据调用次序对React的依赖建模。你可以对照着我们的代码实现，去阅读Hooks法则的解释，就可以完全理解所有内容。

第二条法则，“仅在React函数中调用Hooks”。使用我们这一实现方法，这条法则不是必须遵守的，但是明确的界定代码的哪一部分依赖于有状态的逻辑是相当好的实践方式。（这也可以让我们更容易编写工具来确保遵守第一个法则。你不会在无意中包裹有状态的函数，在循环和条件语句中当成一般的函数包裹它们。遵守第二条法则有助于遵守第一条法则）

#### 结论
到这里，我们已经把最初的例子扩展的很远了。你可以尝试使用一行代码实现useRef，或者让render函数接受JSX并挂载到DOM上，或者实现无数种其他重要的细节，在这28行的hook版本里我们忽略掉了。希望现在你已经收获一些在上下文中使用闭包的经验，并且在头脑中有一个有用的模型，可以解释React Hooks是如何工作的。

我要感谢Dan Abramov和Divya Sasidharan审阅这篇文章的草稿，根据他们宝贵的反馈我改进了本文。剩下出现的所有错误都是我的。

#### 转载文章
[深入理解：React hooks是如何工作的？](https://zhuanlan.zhihu.com/p/81528320)
[Deep dive: How do React hooks really work?](https://www.netlify.com/blog/2019/03/11/deep-dive-how-do-react-hooks-really-work/)