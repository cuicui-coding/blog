```
import React from './react'
import ReactDOM from './react-dom';

// const ele = (
//   <div className='active' title='123'>
//     hello,<span>react</span>
//   </div>
// )

function Home(){
  return (
    <div className='active' title='123'>
      hello,<span>react</span>
    </div>
  )
}
const title = 'active';
console.log(<Home name={title} />)

// 核心：组件化开发
/**
 * 两个问题：
 * 1.为什么ReactDOM.render()必须引入React?
 * 2.组件：函数组件 类组件
 */

// ReactDOM.render(ele, document.querySelector('#root'))

/*
createElement(tag, attrs, child1, child2...)

const ele = React.createElement("div", {
  className: "active",
  title: "123"
}, "hello,", React.createElement("span", null, "react"));
*/
```
`console.log(<Home name={title} />)`插件babel-plugin-transform-react-jsx会判断Home是个组件还是JSX，然后解析处理，得到的Home组件的tag属性是个Home()方法
```
{attrs: {…}, childrens: Array(0), tag: ƒ}
tag: ƒ Home()
attrs: {name: "active"}
childrens: []
__proto__: Object
```




```
// react-dom/index.js 执行console.log(comp)
Component {props: {…}, state: {…}, constructor: ƒ, render: ƒ}
props: {name: "active"}
state: {}
constructor: ƒ Home()
render: ƒ ()
__proto__: Object
```

```
// react-dom/index.js

import Component from '../react/component'

const ReactDOM = {
  render
}

function render(vnode, container) {
  // console.log('vnode:', vnode)
  return container.appendChild(_render(vnode))
  
}
function createComponent(comp, props){
  let inst;
  if(comp.prototype && comp.prototype.render){
    // 如果是类定义的组件 则创建实例 返回
    inst = new comp(props)
  }else{
    // 如果是函数组件 将函数组件扩展类组件 方便后面统一管理
    inst = new Component(props);
    inst.constructor = comp;
    // 定义render函数 返回JSX对象
    inst.render = function(){
      return this.constructor(props);
    }
  }
  return inst;
}
export function renderComponent(comp){
  let base;

  // comp.render()返回JSX转化为的vnode对象
  const renderer = comp.render();
  base = _render(renderer);
  comp.base = base;

}
function setComponentProps(comp, props){

  // 设置组件的属性
  comp.props = props;
  // 渲染组件
  renderComponent(comp)
}

function _render(vnode){
  if (vnode === undefined || vnode === null || typeof vnode === 'boolean') vnode='';

  if(typeof vnode === 'number') vnode = String(vnode)

  // 如果是vnode是字符串
  if (typeof vnode === 'string') {
    // 创建文本节点
    return document.createTextNode(vnode)
  }

  // 如果是tag是函数，则渲染组件 react函数就是一个组件
  if(typeof vnode.tag === 'function'){
    // 1.创建组件
    const comp = createComponent(vnode.tag, vnode.attrs);
    // console.log(comp)
    
    // 2.设置组件的属性
    setComponentProps(comp, vnode.attrs);
    
    // 3.组件渲染的节点对象返回
    return comp.base;
  }
  // 否则就是虚拟DOM对象
  const { tag, attrs } = vnode
  // 创建节点对象
  const dom = document.createElement(tag)

  if (attrs) {
    // 有属性 key: className = 'active' title = '123'
    Object.keys(attrs).forEach(key => {
      const value = attrs[key]
      setAttribute(dom, key, value)
    })
  }

  // 递归渲染子节点
  if(vnode.childrens){
    vnode.childrens.forEach(child => render(child, dom))
  }
  
  return dom;
}

// 设置属性
function setAttribute(dom, key, value) {
  // 将属性名className转化成class
  if (key === 'className') {
    key = 'class'
  }
  // 如果是事件 onClick onBlur
  if (/on\w+/.test(key)) {
    // 转小写
    key.toLowerCase()
    dom[key] = value
  } else if (key === 'style') {
    if (!value || typeof value === 'string') {
      dom.style.cssText = value || ''
    } else if (value && typeof value === 'object') {
      // {width: 20}
      for (let k in value) {
        if (typeof value[k] === 'number') {
          dom.style[k] = value[k] + 'px'
        } else {
          dom.style[k] = value[k]
        }
      }
    }
  } else {
    // 其他属性 title='' id='app'
    if (key in dom) {
      dom[key] = value || ''
    }

    if (value) {
      // 更新值
      dom.setAttribute(key, value)
    } else {
      dom.removeAttribute(key)
    }
  }
}

export default ReactDOM

```

```
// react/component.js

class Component{
  constructor(props = {}){
    this.props = props;
  }
}

export default Component;
```
```
//react/index.js

import Component from './component'

function createElement(tag, attrs, ...childrens) {
  return {
    tag,
    attrs,
    childrens
  }
}

export default { createElement, Component }

```