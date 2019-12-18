### 基本概念


#### useState和useEffect使用
```
import React, { useState, useEffect } from 'react';

export default () => {
    const [count, setCount] = useState(0);

    useEffect(() => {
      const I = setTimeout(()=>{
          setCount(x => x+1)
      }, 1000)
      return ()=>{
          clearTimeout(I)
      }
    }, [Math.min(count, 4)]);

    return (
        <div>{count}</div>
    );
}
```
> useEffect的第二个参数的重要性

1. useEffect的第二个参数不填，页面的count会一直+1递增，页面一直处于更新状态中，页面挂载时执行useEffect方法count加1，count变化重新渲染页面，又执行useEffect方法，count变化又渲染页面，一直循环中。

2. useEffect执行完毕后，可以返回清理函数，每次渲染都会执行。

#### 就项目迁移 LifeCycle

类组件的生命周期在React Hook函数组件的替代
```
const Input = (props) => {

    // component will receive props
    // shouldComponentUpdate 
    useEffect(() =>{
        console.log(`value changed ${props.value}`)
    }, [props.value])

    return <input />
}

export default (props) => {
    const [value, setValue] =  useState('');

    useEffect(()=>{
        console.log(`component did mount`)
        return () =>{
            console.log(`component will unmount`)
        }
    }, []) 

    return <div>
        <Input value = {value} />
        <button onClick = { () => setValue(x => Math.random())}>Clicke Me</button>
    </div>
}
```
#### useRef使用

```
import React, { useEffect, useState, useRef, forwardRef } from 'react';

const Input = (props, ref) => {

  // component will receive props
  // shouldComponentUpdate 
  useEffect(() =>{
      ref.current.value = props.initialValue
  }, [])

  return <input ref={ref} onChange={e => props.onChange(e.target.value)}/>
}

const MInput = forwardRef(Input)

export default (props) => {

  const r1 = useRef();
  const r2 = useRef();

  return <div>
      <MInput ref={r1} initialValue = {100} onChange={x=>console.log("1:"+ x)}/>
      <MInput ref={r2} initialValue = {"hello"} onChange={x=>console.log("2:"+ x)}/>
      
      <button onClick = { () => r1.current.focus()}>focus 1</button>
      <button onClick = { () => r2.current.focus()}>focus 2</button>
  </div>
}
```
这个方法是可行的，但违反了面向对象封装原则，既然ref是子组件使用的，那就应该是子组件来创建ref。父组件不应该去拿子组件的东西，而应该是子组件暴露接口给父组件用。我们设计一个对象/物体，这个物体应该有它实实在在的所有功能。子组件既然是input就应该有focus功能。

所以我们提供了另外一个hook是useImperativeHandle, 给ref注册了一个focus函数，这样就相当于子组件提供了一个接口给父组件(提供focus方法)。这就是这个hook的作用。

```
import React, {
  useEffect,
  useState,
  useRef,
  forwardRef,
  useImperativeHandle
} from 'react'

const Input = (props, ref) => {
  const refInput = useRef()

  useImperativeHandle(ref,() => ({
      focus: () => {
        refInput.current.focus()
      }
    }),
    []
  )

  // component will receive props
  // shouldComponentUpdate
  useEffect(() => {
    refInput.current.value = props.initialValue
  }, [])
  
  console.log('refInput:',refInput)
  return <input ref={refInput} onChange={e => props.onChange(e.target.value)} />
}

const MInput = forwardRef(Input)

export default props => {
  const r1 = useRef()
  const r2 = useRef()

  return (
    <div>
      <MInput
        ref={r1}
        initialValue={100}
        onChange={x => console.log('1:' + x)}
      />
      <MInput
        ref={r2}
        initialValue={'hello'}
        onChange={x => console.log('2:' + x)}
      />

      <button onClick={() => {
          console.log('r1:',r1)
          r1.current.focus()
        }
      }>focus 1</button>
      <button onClick={() => r2.current.focus()}>focus 2</button>
    </div>
  )
}

```
这个r1和r2传递下去了，但是r1打印出来，就只有r1.current.focus一个方法,而refInput.current获取的就是input的DOM节点。

#### useContext
```
import React, { useContext } from 'react'
import HelloContext from './HelloContext'

// HelloContext.Provider
// HelloContext.Consumer

const { Provider } = HelloContext

const Desendants = () => {
  const value = useContext(HelloContext)
  return <div>{value}</div>
}

const Child = () => {
  return <Desendants />
}

const Parent = () => {
  return (
    <Provider value="hello world">
      <Child />
    </Provider>
  )
}
export default Parent

```

#### useEffect

hook要写在最外层，不要写在if语句中，否则下次渲染执行if时，hook顺序错乱。

**原因：hook内部实现是靠顺序来实现，基于顺序管理判断当前hook。**


```
import React, { useState, useEffect } from 'react'

const Greetings = () => {
  return <div>hello</div>
}

export default () => {
  const [count, setCount] = useState(0)
  const [display, setDisplay] = useState(false)

  // 错误递归
  // if( count > 1){
  //   setDisplay(x => true)
  // }

  // render more hooks than previous render
  // if (count > 1) {
  //   useEffect(() => {
  //     setDisplay(x => true)
  //   }, [])
  // } else {
  //   useEffect(() => {
  //     setDisplay(x => false)
  //   }, [])
  // }

  useEffect(() => {
    if (count > 1) {
      setDisplay(x => true)
    }
  }, [count > 1])

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(x => x + 1)}>+1</button>
      {display && <Greetings />}
    </div>
  )
}

```
### useCallback && useLayoutEffect
为了解决性能问题

```
import React, { useState, useEffect, useCallback } from 'react'

export default () => {
  const [count, setCount] = useState(0)

  const inc = useCallback(() => {
    console.log('count', count)
    // setCount(x => x + 1) // 自己的count,可以递增+1
    setCount(count + 1)  // count存起来，始终得到1
  }, [])
  return (
    <div>
      <p>{count}</p>
      <button onClick={inc}>+ 1</button>
    </div>
  )
}

```
setCount(x => x + 1)和setCount(count + 1)是有区别：
- setCount(x => x + 1)是读自己的count的值，每次点击都会递增+1。
- setCount(count + 1)中count被存起来，旧的值被记住，每次点击都得到同样的值。

### useMemo
为了解决性能问题，缓存

```
export default function WithMemo() {
    const [count, setCount] = useState(1);
    const [val, setValue] = useState('');
    const expensive = useMemo(() => {
        console.log('compute');
        let sum = 0;
        for (let i = 0; i < count * 100; i++) {
            sum += i;
        }
        return sum;
    }, [count]);

    return <div>
        <h4>{count}-{expensive}</h4>
        {val}
        <div>
            <button onClick={() => setCount(count + 1)}>+c1</button>
            <input value={val} onChange={event => setValue(event.target.value)}/>
        </div>
    </div>;
}

```
在hooks出来之后，在函数组件中，react不再区分mount和update两个状态，这意味着函数组件的每一次调用都会执行其内部的所有逻辑，那么会带来较大的性能损耗。因此useMemo 和useCallback就是解决性能问题的杀手锏。

useCallback和useMemo的参数跟useEffect一致，他们之间最大的区别有是useEffect会用于处理副作用，而前两个hooks不能。

useMemo和useCallback都会在组件第一次渲染的时候执行，之后会在其依赖的变量发生改变时再次执行；并且这两个hooks都返回缓存的值，useMemo返回缓存的变量，useCallback返回缓存的函数。

#### useReducer

useState 的替代方案。它接收一个形如 (state, action) => newState 的 reducer，并返回当前的 state 以及与其配套的 dispatch 方法。以下是用 reducer 重写 useState 一节的计数器示例：
```
import React, { useState, useReducer } from 'react'

const initialState = { count: 0 }

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 }
    case 'decrement':
      return { count: state.count - 1 }
    default:
      throw new Error()
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState)
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({ type: 'decrement' })}>-1</button>
      <button onClick={() => dispatch({ type: 'increment' })}>+1</button>
    </>
  )
}
export default Counter
```
在某些场景下，useReducer 会比 useState 更适用，例如 state 逻辑较复杂且包含多个子值，或者下一个 state 依赖于之前的 state 等。并且，使用 useReducer 还能给那些会触发深更新的组件做性能优化，因为你可以向子组件传递 dispatch 而不是回调函数 。

#### 原理&源码解析

```
import React, { Component, useState, useEffect } from 'react'


const Child = (props, ref)=>{

  useState(1)
  useState(2)
  useState(3)

  useEffect((a) => {
    console.log(a)
  });
  useEffect((b) => {
    console.log(b)
  });
  useEffect((c) => {
    console.log(c)
  });

  return <span>this is span</span>

}

export class chain extends Component {
  componentDidMount(){
    const childFiberNode = this._reactInternalFiber.child;
    console.log(childFiberNode.elementType)
    console.log(childFiberNode.memoizedState)
    console.log(childFiberNode.updateQueue)


    setTimeout(()=>{
      console.log(this)
      debugger
    })
  }
  render() {
    return (
      <div>
        <Child x={1}/>
      </div>
    );
  }
}

export default chain;

```
通过this._reactInternalFiber.child得到FiberNode，FiberNode就是一棵树，每一个节点对应一个真实DOM, FiberNode有很多属性：stateNode, tag, type, updateQueue, memoizedState。

整体设计就是一个链表。

Fiber有两个更新周期，分为两个部分，phase1: render， phase2:commit.

- phase1: render
render时会产生vdom树，比较前一个版本树，产生一个更新序列。  
这个阶段对应useState

在node节点上，setData(10)改变的是FiberNode的状态, memoizedState会记住变化的值，5->10->20，是个环状链表，无数次提交setData,把变化存成一个Queue, 计算这些值，产生更新序列，牵扯到updateQueue.

所以useState发生在第一阶段。

- phase2: commit
apply每个更新序列，真实dom操作。
在apply更新序列时候，会看到updateQueue，它里面都是effects, 在函数式中称为副作用，effects它也会产生effects的变化，E1->E2->E3。

所以useEffect发生在第二个阶段。
