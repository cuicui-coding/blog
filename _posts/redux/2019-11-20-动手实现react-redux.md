---
layout: default
title: 动手实现react-redux
---

### react-redux 是什么

react-redux 是 redux 官方 React 绑定库。它帮助我们连接UI层和数据层。本文目的不是介绍 react-redux 的使用，而是要动手实现一个简易的 react-redux，希望能够对你有所帮助。

首先思考一下，倘若不使用 react-redux，我们的 react 项目中该如何结合 redux 进行开发呢。

每个需要与 redux 结合使用的组件，我们都需要做以下几件事：

- 在组件中获取 store 中的状态

- 监听 store 中状态的改变，在状态改变时，刷新组件

- 在组件卸载时，移除对状态变化的监听。

如下:
```
import React from 'react';
import store from '../store';
import actions from '../store/actions/counter';
/**
 * reducer 是 combineReducer({counter, ...})
 * state 的结构为 
 * {
 *      counter: {number: 0},
 *      ....
 * }
 */
class Counter extends React.Component {
    constructor(props) {
        super(props);
        this.state = {
            number: store.getState().counter.number
        }
    }
    componentDidMount() {
        this.unsub = store.subscribe(() => {
            if(this.state.number === store.getState().counter.number) {
                return;
               }
            this.setState({
                number: store.getState().counter.number
            });
        });
    }
    componentWillUnmount() {
        this.unsub();
    }
    render() {
        return (
            <div>
                <p>{`number: ${this.state.number}`}</p>
                <button onClick={() => {store.dispatch(actions.add(2))}}>+</button>
                <button onClick={() => {store.dispatch(actions.minus(2))}}>-</button>
            <div>
        )
    }
}
```
如果我们的项目中有很多组件需要与 redux 结合使用，那么这些组件都需要重复写这些逻辑。显然，我们需要想办法复用这部分的逻辑，不然会显得我们很蠢。我们知道，react 中高阶组件可以实现逻辑的复用。

文中所用到的 Counter 代码在 https://github.com/YvetteLau/Blog 中的 myreact-redux/counter 中，建议先 clone 代码，当然啦，如果觉得本文不错的话，给个star鼓励。

### 逻辑复用

在 src 目录下新建一个 react-redux 文件夹，后续的文件都新建在此文件夹中。

#### 创建 connect.js 文件

文件创建在 react-redux/components 文件夹下:

我们将重复的逻辑编写 connect 中。  
```
import React, { Component } from 'react';
import store from '../../store';

export default function connect (WrappedComponent){
    return class Connect extends Component {
        constructor(props) {
            super(props);
            this.state = store.getState();
        }
        componentDidMount() {
            this.unsub = store.subscribe(() => {
                this.setState({
                    this.setState(store.getState());
                });
            });
        }
        componentWillUnmount() {
            this.unsub();
        }
        render() {
            return (
                <WrappedComponent {...this.state} {...this.props}/>
            )
        }
    }
}
```
有个小小的问题，尽管这逻辑是重复的，但是每个组件需要的数据是不一样的，不应该把所有的状态都传递给组件，因此我们希望在调用 connect 时，能够将需要的状态内容告知 connect。另外，组件中可能还需要修改状态，那么也要告诉 connect，它需要派发哪些动作，否则 connect 无法知道该绑定那些动作给你。

为此，我们新增两个参数：mapStateToProps 和 mapDispatchToProps，这两个参数负责告诉 connect 组件需要的 state 内容和将要派发的动作。

#### mapStateToProps 和 mapDispatchToProps
我们知道 mapStateToProps 和 mapDispatchToProps 的作用是什么，但是目前为止，我们还不清楚，这两个参数应该是一个什么样的格式传递给 connect 去使用。

```
import { connect } from 'react-redux';
....
//connect 的使用
export default connect(mapStateToProps, mapDispatchToProps)(Counter);
```
- mapStateToProps 告诉 connect ，组件需要绑定的状态。  

  mapStateToProps 需要从整个状态中挑选组件需要的状态，但是在调用 connect 时，我们并不能获取到 store ，不过 connect 内部是可以获取到 store 的，为此，我们将 mapStateToProps 定义为一个函数，在 connect 内部调用它，将 store 中的 state 传递给它，然后将函数返回的结果作为属性传递给组件。组件中通过 this.props.XXX 来获取。因此，mapStateToProps 的格式应该类似下面这样:

  ```
  /将 store.getState() 传递给 mapStateToProps
  mapStateToProps = state => ({
      number: state.counter.number
  });
  ```
- mapDispatchToProps 告诉 connect，组件需要绑定的动作。

  回想一下，组件中派发动作：store.dispatch({actions.add(2)})。connect 包装之后，我们仍要能派发动作，肯定是 this.props.XXX() 这样的一种格式。

  比如，计数器的增加，调用 this.props.add(2)，就是需要派发 store.dispatch({actions.add(2)})，因此 add 属性，对应的内容就是 (num) => { store.dispatch({actions.add(num)}) }。传递给组件的属性类似下面这样:
  ```
  {
    add: (num) => {
        store.dispatch(actions.add(num))
    },
    minus: (num) => {
        store.dispatch(actions.minus(num))
    }
  }
  ```
  和 mapStateToProps 一样，在调用 connect 时，我们并不能获取到 store.dispatch，因此我们也需要将 mapDispatchToProps 设计为一个函数，在 connect 内部调用，这样可以将 store.dispatch 传递给它。所以，mapStateToProps 应该是下面这样的格式:
  ```
  //将 store.dispacth 传递给 mapDispatchToProps
  mapDispatchToProps = (dispatch) => ({
      add: (num) => {
          dispatch(actions.add(num))
      },
      minus: (num) => {
          dispatch(actions.minus(num))
      }
  })
  ```
至此，我们已经搞清楚 mapStateToProps 和 mapDispatchToProps 的格式，是时候进一步改进 connect 了。

### connect 1.0 版本
```
import React, { Component } from 'react';
import store from '../../store';

export default function connect (mapStateToProps, mapDispatchToProps) {
    return function wrapWithConnect (WrappedComponent) {
        return class Connect extends Component {
            constructor(props) {
                super(props);
                this.state = mapStateToProps(store.getState());
                this.mappedDispatch = mapDispatchToProps(store.dispatch);
            }
            componentDidMount() {
                this.unsub = store.subscribe(() => {
                    const mappedState = mapStateToProps(store.getState());
                    //TODO 做一层浅比较，如果状态没有改变，则不setState
                    this.setState(mappedState);
                });
            }
            componentWillUnmount() {
                this.unsub();
            }
            render() {
                return (
                    <WrappedComponent {...this.props} {...this.state} {...this.mappedDispatch} />
                )
            }
        }
    }
}
```
我们知道，connect 是作为 react-redux 库的方法提供的，因此我们不可能直接在 connect.js 中去导入 store，这个 store 应该由使用 react-redux 的应用传入。react 中数据传递有两种：通过属性 props 或者是通过上下文对象 context，通过 connect 包装的组件在应用中分布，而 context 设计目的是为了共享那些对于一个组件树而言是“全局”的数据。

我们需要把 store 放在 context 上，这样根组件下的所有子孙组件都可以获取到 store。这部分内容，我们当然可以自己在应用中编写相应代码，不过很显然，这些代码在每个应用中都是重复的。因此我们把这部分内容也封装在 react-redux 内部。

此处，我们使用旧的 Context API 来写(鉴于我们实现的 react-redux 4.x 分支的代码，因此我们使用旧版的 context API)。

### Provider
我们需要提供一个 Provider 组件，它的功能就是接收应用传递过来的 store，将其挂在 context 上，这样它的子孙组件就都可以通过上下文对象获取到 store。

#### 新建 Provider.js 文件
文件创建在 react-redux/components 文件夹下：

```
import React, { Component } from 'react';
import PropTypes from 'prop-types';

export default class Provider extends Component {
    static childContextTypes = {
        store: PropTypes.shape({
            subscribe: PropTypes.func.isRequired,
            dispatch: PropTypes.func.isRequired,
            getState: PropTypes.func.isRequired
        }).isRequired
    }

    constructor(props) {
        super(props);
        this.store = props.store;
    }

    getChildContext() {
        return {
            store: this.store
        }
    }

    render() {
        /**
         * 早前返回的是 return Children.only(this.props.children)
         * 导致Provider只能包裹一个子组件，后来取消了此限制
         * 因此此处，我们直接返回 this.props.children
         */
        return this.props.children
    }
}
```
#### 新建一个 index.js 文件
文件创建在 react-redux 目录下:

此文件只做一件事，即将 connect 和 Provider 导出
```
import connect from './components/connect';
import Provider from './components/Provider';

export {
    connect,
    Provider
}
```
#### Provider 的使用
使用时，我们只需要引入 Provider，将 store 传递给 Provider。
```
import React, { Component } from 'react';
import { Provider } from '../react-redux';
import store from './store';
import Counter from './Counter';

export default class App extends Component {
    render() {
        return (
            <Provider store={store}>
                <Counter />
            </Provider>
        )
    }
}
```
至此，Provider 的源码和使用已经说明清楚了，不过相应的 connect 也需要做一些修改，为了通用性，我们需要从 context 上去获取 store，取代之前的导入。

### connect 2.0 版本
```
import React, { Component } from 'react';
import PropTypes from 'prop-types';

export default function connect(mapStateToProps, mapDispatchToProps) {
    return function wrapWithConnect(WrappedComponent) {
        return class Connect extends Component {
            //PropTypes.shape 这部分代码与 Provider 中重复，因此后面我们可以提取出来
            static contextTypes = {
                store: PropTypes.shape({
                    subscribe: PropTypes.func.isRequired,
                    dispatch: PropTypes.func.isRequired,
                    getState: PropTypes.func.isRequired
                }).isRequired
            }

            constructor(props, context) {
                super(props, context);
                this.store = context.store;
                //源码中是将 store.getState() 给了 this.state
                this.state = mapStateToProps(this.store.getState());
                this.mappedDispatch = mapDispatchToProps(this.store.dispatch);
            }
            componentDidMount() {
                this.unsub = this.store.subscribe(() => {
                    const mappedState = mapStateToProps(this.store.getState());
                    //TODO 做一层浅比较，如果状态没有改变，则无需 setState
                    this.setState(mappedState);
                });
            }
            componentWillUnmount() {
                this.unsub();
            }
            render() {
                return (
                    <WrappedComponent {...this.props} {...this.state} {...this.mappedDispatch} />
                )
            }
        }
    }
}
```
使用 connect 关联 Counter 与 store 中的数据。
```
import React, { Component } from 'react';
import { connect } from '../react-redux';
import actions from '../store/actions/counter';

class Counter extends Component {
    render() {
        return (
            <div>
                <p>{`number: ${this.props.number}`}</p>
                <button onClick={() => { this.props.add(2) }}>+</button>
                <button onClick={() => { this.props.minus(2) }}>-</button>
            </div>
        )
    }
}

const mapStateToProps = state => ({
    number: state.counter.number
});

const mapDispatchToProps = (dispatch) => ({
    add: (num) => {
        dispatch(actions.add(num))
    },
    minus: (num) => {
        dispatch(actions.minus(num))
    }
});


export default connect(mapStateToProps, mapDispatchToProps)(Counter);
```

store/actions/counter.js 定义如下：
```
import { INCREMENT, DECREMENT } from '../action-types';

const counter = {
    add(number) {
        return {
            type: INCREMENT,
            number
        }
    },
    minus(number) {
        return {
            type: DECREMENT,
            number
        }
    }
}
export default counter;
```
至此，我们的 react-redux 库已经可以使用了，不过很有很多细节问题待处理:

- mapDispatchToProps 的定义写起来有点麻烦，不够简洁
大家是否还记得 redux 中的 bindActionCreators，借助于此方法，我们可以允许传递 actionCreator 给 connect，然后在 connect 内部进行转换。

- connect 和 Provider 中的 store 的 PropType 规则可以提取出来，避免代码的冗余

- mapStateToProps 和 mapDispatchToProps 可以提供默认值
mapStateToProps 默认值为 state => ({}); 不关联 state；

- mapDispatchToProps 的默认值为 dispatch => ({dispatch})，将 store.dispatch 方法作为属性传递给被包装的属性。

- 目前，我们仅传递了 store.getState() 给 mapStateToProps，但是很可能在筛选过滤需要的 state 时，需要依据组件自身的属性进行处理，因此，可以将组件自身的属性也传递给 mapStateToProps，同样的原因，也将自身属性传递给 mapDispatchToProps。

### connect 3.0 版本
我们将 store 的 PropType 规则提取出来，放在 utils/storeShape.js 文件中。

浅比较的代码放在 utils/shallowEqual.js 文件中，通用的浅比较函数，直接阅读下代码。

```
export default function shallowEqual(objA, objB) {
    if (objA === objB) {
        return true
    }

    const keysA = Object.keys(objA)
    const keysB = Object.keys(objB)

    if (keysA.length !== keysB.length) {
        return false
    }

    const hasOwn = Object.prototype.hasOwnProperty
    for (let i = 0; i < keysA.length; i++) {
        if (!hasOwn.call(objB, keysA[i]) ||
            objA[keysA[i]] !== objB[keysA[i]]) {
            return false
        }
    }

    return true
}
```
修改后的connect.js
```
import React, { Component } from 'react';
import { bindActionCreators } from 'redux';
import storeShape from '../utils/storeShape';
import shallowEqual from '../utils/shallowEqual';
/**
 * mapStateToProps 默认不关联state
 * mapDispatchToProps 默认值为 dispatch => ({dispatch})，将 `store.dispatch` 方法作为属性传递给组件
 */
const defaultMapStateToProps = state => ({});
const defaultMapDispatchToProps = dispatch => ({ dispatch });

export default function connect(mapStateToProps, mapDispatchToProps) {
    if(!mapStateToProps) {
        mapStateToProps = defaultMapStateToProps;
    }
    if (!mapDispatchToProps) {
        //当 mapDispatchToProps 为 null/undefined/false...时，使用默认值
        mapDispatchToProps = defaultMapDispatchToProps;
    }
    return function wrapWithConnect(WrappedComponent) {
        return class Connect extends Component {
            static contextTypes = {
                store: storeShape
            };
            constructor(props, context) {
                super(props, context);
                this.store = context.store;
                //源码中是将 store.getState() 给了 this.state
                this.state = mapStateToProps(this.store.getState(), this.props);
                if (typeof mapDispatchToProps === 'function') {
                    this.mappedDispatch = mapDispatchToProps(this.store.dispatch, this.props);
                } else {
                    //传递了一个 actionCreator 对象过来
                    this.mappedDispatch = bindActionCreators(mapDispatchToProps, this.store.dispatch);
                }
            }
            componentDidMount() {
                this.unsub = this.store.subscribe(() => {
                    const mappedState = mapStateToProps(this.store.getState(), this.props);
                    if (shallowEqual(this.state, mappedState)) {
                        return;
                    }
                    this.setState(mappedState);
                });
            }
            componentWillUnmount() {
                this.unsub();
            }
            render() {
                return (
                    <WrappedComponent {...this.props} {...this.state} {...this.mappedDispatch} />
                )
            }
        }
    }
}
```
至此，react-redux 我们就基本实现了，不过这个代码并不完善，比如，ref 丢失的问题，组件的 props 变化时，重新计算 this.state 和 this.mappedDispatch，没有进一步进行性能优化等。你可以在此基础上进一步进行处理。

react-redux 主干分支的代码已经使用 hooks 改写，后期如果有时间，会输出一篇新版本的代码解析。

最后，使用我们自己编写的 react-redux 和 redux 编写了 Todo 的demo，功能正常，代码在 在 https://github.com/YvetteLau/Blog 中的 myreact-redux/todo 下。

### 转载文章：
[【重学React】动手实现一个react-redux](https://mp.weixin.qq.com/s/WuDhegD-bWv5WQGq--tjKw)

[一起学习造轮子（三）：从零开始写一个React-Redux](https://juejin.im/post/5b32f145f265da596a3682b4)