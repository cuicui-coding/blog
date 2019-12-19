```
// index.js
import React from './react'
import ReactDOM from './react-dom';

const ele = (
  <div className='active' title='123'>
    hello,<span>react</span>
  </div>
)

// // 函数组件
// function Home(){
//   return (
//     <div className='active' title='123'>
//       hello,<span>react</span>
//     </div>
//   )
// }

// 类组件
export class Home extends React.Component {
  constructor(props){
    super(props);
    this.state={
      num:0,
    }
  }
  handlerClick(){
    console.log(12)
    // 修改状态的唯一方法是调用setState
    this.setState({
      num: this.state.num+1
    })
  }
  render() {
    return (
      <div className='active' title='123'>
        hello,<span>react {this.state.num}</span>
        <button onclick={this.handlerClick.bind(this)}>摸我</button>
      </div>
    );
  }
  componentWillMount(){
    console.log('组件将要加载')
  }
  componentWillReceiveProps(props){
    console.log('props')
  }
  componentWillUpdate(){
    console.log('组件将要更新')
  }
  componentDidUpdate(){
    console.log('组件更新完成')
  }
  componentDidMount(){
    console.log('组件加载完成')
  }
}

const title = 'active';
console.log('Home comp:',<Home name={title} />)

// 核心：组件化开发
/**
 * 两个问题：
 * 1.为什么ReactDOM.render()必须引入React?
 * 2.组件：函数组件 类组件
 */

ReactDOM.render(ele, document.querySelector('#root'))

/*
createElement(tag, attrs, child1, child2...)

const ele = React.createElement("div", {
  className: "active",
  title: "123"
}, "hello,", React.createElement("span", null, "react"));
*/
```

```
// react-dom/index.js

import Component from '../react/component'
import {diff} from './diff'
const ReactDOM = {
  render
}

function render(vnode, container, dom) {
  // console.log('vnode:', vnode)
  // return container.appendChild(_render(vnode))
  
  return diff(dom, vnode, container)
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

  if(comp.base && comp.componentWillUpdate){
    comp.componentWillUpdate();
  }

  if(comp.base){
    if(comp.componentDidUpdate) comp.componentDidUpdate()
  }else if(comp.componentDidMount){
    comp.componentDidMount()
  }

  // 节点替换
  if(comp.base && comp.base.parentNode){
    comp.base.parentNode.replaceChild(base, comp.base);
  }
  comp.base = base;

}
function setComponentProps(comp, props){
  if(!comp.base){
    if(comp.componentWillMount) comp.componentWillMount();
  }else if(comp.componentWillReceiveProps){
     comp.componentWillReceiveProps();
  }

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
export function setAttribute(dom, key, value) {
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
// react-dom/diff.js

import { setAttribute } from './index'
export function diff(dom, vnode, container) {
  const ret = diffNode(dom, vnode)
  if (container) {
    container.appendChild(ret)
  }
  return ret
}

function diffNode(dom, vnode) {
  let out = dom
  if (vnode === undefined || vnode === null || typeof vnode === 'boolean')
    vnode = ''

  if (typeof vnode === 'number') vnode = String(vnode)

  // 如果是vnode是字符串
  if (typeof vnode === 'string') {
    // 如果dom,更新内容
    if (dom && dom.nodeType === 3) {
      if (dom.textContent !== vnode) {
        // 更新文本内容
        dom.textContent = vnode
      }
    } else {
      // 创建文本节点
      out = document.createTextNode(vnode)
      if (dom && dom.parentNode) {
        dom.parentNode.replaceNode(out, dom)
      }
    }
    return out
  }

  // 非文本DOM节点
  if (!dom) {
    out = document.createElement(vnode.tag)
  }
  // 比较子节点（dom节点和组件）
  if (
    (vnode.childrens && vnode.childrens.length > 0) ||
    (out.childNodes && out.childNodes.length > 0)
  ) {
    // 对比组件 或者子节点

    diffChildren(out, vnode)
  }

  diffAttribute(out, vnode)

  return out
}
function diffChildren(out, vChildren) {}
function diffAttribute(dom, vnode) {
  // 保存之前的DOM的所有属性
  const oldAttrs = {}
  // dom是原有的节点对象 vnode 虚拟DOM
  const newAttrs = vnode.attrs
  const domAttrs = dom.attributes
  ;[...domAttrs].forEach(item => {
    // console.log(item.name, item.value)
    oldAttrs[item.name] = item.value
  })
  // 比较
  // 如果原来属性跟新的属性对比，不在新的属性中，则将其移除掉（属性值为undefined）
  for (let key in oldAttrs) {
    if (!(key in newAttrs)) {
      setAttribute(dom, key, undefined)
    }
  }
  // 更新class='active'变为class='abc'
  for (const key in newAttrs) {
    if (oldAttrs[key] !== newAttrs[key]) {
      // 值不同，更新值
      setAttribute(dom, key, newAttrs[key])
    }
  }
}

```

```
// react/component.js

import {renderComponent} from '../react-dom'
class Component{
  constructor(props = {}){
    this.props = props;
    this.state = {}
  }
  setState(stateChange){
    // 对象拷贝
    Object.assign(this.state, stateChange);

    // 渲染组件
    renderComponent(this)
  }
}

export default Component;
```

```
// react/index.js

import Component from '../react/component'
import {diff} from './diff'
const ReactDOM = {
  render
}

function render(vnode, container, dom) {
  // console.log('vnode:', vnode)
  // return container.appendChild(_render(vnode))
  
  return diff(dom, vnode, container)
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

  if(comp.base && comp.componentWillUpdate){
    comp.componentWillUpdate();
  }

  if(comp.base){
    if(comp.componentDidUpdate) comp.componentDidUpdate()
  }else if(comp.componentDidMount){
    comp.componentDidMount()
  }

  // 节点替换
  if(comp.base && comp.base.parentNode){
    comp.base.parentNode.replaceChild(base, comp.base);
  }
  comp.base = base;

}
function setComponentProps(comp, props){
  if(!comp.base){
    if(comp.componentWillMount) comp.componentWillMount();
  }else if(comp.componentWillReceiveProps){
     comp.componentWillReceiveProps();
  }

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
export function setAttribute(dom, key, value) {
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