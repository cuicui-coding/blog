### 从0实现React
#### 实现功能
```
下载nodejs
下载脚手架：npm i create-react-app -g
创建项目：create-react-app react-test
```
0. 火热的0配置的打包工具parcel
  安装babel插件，将JSX语法转换成JS对象（虚拟DOM）
  ```
  cnpm i babel-core babel-preset-env babel-plugin-transform-react-jsx --save-dev
  ```
1. 封装JSX和理解虚拟DOM
2. 组件和生命周期
3. diff算法
   **diff算法**
   diff算法？what? 什么玩意
   
   如何减少DOM更新，我们需要找到渲染前后真正变化的部分，只更新这一部分，而对比变化，找到需要更新部分的算法称之为**diff算法**

  **对比策略**

  在前面我们实现了_render方法,它能将虚拟DOM转换成真正的DOM

  但是我们我需要改进它，不要让它傻乎乎地重新渲染整个DOM树，而是找出真正变化的部分进行替换

  这部分很多类似React框架实现方式都不太一样，有的框架会选择保存上次渲染的虚拟DOM，然后对比虚拟DOM前后的变化，得到一系列更新的数据，然后再将这些更新应用到真正的DOM上。

  **我们会选择直接对比虚拟DOM与真实DOM，这样就不需要额外保存上一次渲染的虚拟DOM，并且能够一边对比一边更新，这也是我们选择的方式。**

  不管是DOM还是虚拟DOM，它们的结构都是一棵树，完全对比两颗树变化的算法时间复杂度是O(n^3),但是考虑很少会跨层级移动DOM,所以我们只需要对比同一层级的变化。

  总而言之，我们的diff算法有两个原则
  - 对比当前真实DOM和虚拟DOM,在对比过程中直接更新正式DOM
  - 只对比同一层级的变化



4. 异步的setState

```
import React from 'react';
import ReactDOM from 'react-dom';

const ele=(
  <div title="hello">
    <h3>hello, react</h3>
  </div>
)

// 
// const ele = React.createElement("div", {
//   title: "hello"
// }, React.createElement("h3", null, "hello, react"));

// jsx:javasript+xml 虚拟DOM 语法糖  （虚拟DOM->真实DOM）
ReactDOM.render(ele, document.querySelector('#root'))
```
使用parcel构建工具
```
{
  "scripts": {
    "start": "parcel index.html"
  },
  "dependencies": {},
  "devDependencies": {
    "babel-core": "^6.26.3",
    "babel-plugin-transform-react-jsx": "^6.24.1",
    "babel-preset-env": "^1.7.0",
    "parcel-bundler": "^1.12.4"
  }
}

```
JSX转化成React.createElememt的方法，使用babel-plugin-transform-react-jsx插件
```
{
  "presets": ["env"],
  "plugins": [
    ["transform-react-jsx", {
      "prama":"React.createElememt"
    }]
  ]
}
```