
分析JSX转化为JS对象（虚拟DOM）
```
const ele = (
  <div className='active' title='123'>
    hello,<span>react</span>
  </div>
)
/*
createElement(tag, attrs, child1, child2...)

const ele = React.createElement("div", {
  className: "active",
  title: "123"
}, "hello,", React.createElement("span", null, "react"));
*/
```
在babel中 https://babeljs.io/repl, 编译JSX转化为VDOM。

实现render方法：
```
// index.js
import React from './react'
import ReactDOM from './react-dom';

const ele = (
  <div className='active' title='123'>
    hello,<span>react</span>
  </div>
)

console.log('ele:', ele)

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
// react/index.js
const React = {
  createElement
}

function createElement(tag, attrs, ...childrens){
  return {
    tag,
    attrs,
    childrens
  }
}

export default React;
```
```
// react-dom/index.js
const ReactDOM = {
  render
}

function render(vnode, container) {
  // console.log('vnode:', vnode)

  if (vnode === undefined) return

  // 如果是vnode是字符串
  if (typeof vnode === 'string') {
    // 创建文本节点
    const textNode = document.createTextNode(vnode)
    return container.appendChild(textNode)
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
  vnode.childrens.forEach(child => render(child, dom))
  
  return container.appendChild(dom)
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