受控组件

表单组件有value props(单选按钮和复选按钮对应的是 checked props)，同时事件处理器更新value值。
```
<input value={someValue} onChange={handleChange} />
```
思考input的value props应该来自哪里？
通常组件会把input的value props值保存在自身state中，当然也可以来自另一个组件的state,或者放在store,比如Redux。
```
class Form extends Component {
  constructor() {
    super();
    this.state = {
      name: '',
    };
  }

  handleNameChange = (event) => {
    this.setState({ name: event.target.value });
  };

  render() {
    return (
      <div>
        <input
          type="text"
          value={this.state.name}
          onChange={this.handleNameChange}
        />
      </div>
    );
  }
}
```
<input>或<select>都要绑定一个change事件;每当表单的状态发生变化,都会被写入组件的state中,这种组件在React中被称为受控组件;在受控组件中,组件渲染出的状态与它的value或者checked prop向对应.react通过这种方式消除了组件的局部状态,使应用的整个状态可控.react官方同样推荐使用受控表单组件,总结下React受控组件更新state的流程:

- 1.可以通过初始state中设置表单的默认值;
- 2.每当表单的值发生变化时,调用onChange事件处理器;
- 3.事件处理器通过合成事件对象e拿到改变后的状态,并更新应用的state.
- 4.setState触发视图的重新渲染,完成表单组件值得更新

> react中数据是单向流动的.从示例中,我们能看出来表单的数据来源于组件的state,并通过props传入,这也称为单向数据绑定.然后,我们又通过onChange事件处理器将新的表单数据写回到state,完成了双向数据绑定.

非受控组件

非受控组件不同于受控组件数据是react组件处理的，是通过dom进行处理，将真实数据保存在DOM中，此时将input通过ref暴露给组件，可以通过this.input进行访问DOM获得表单值，而不是为每个状态更新编写一个事件处理程序。

在React的生命周期中，表单元素上的value属性将会覆盖DOM中的值。使用非受控组件时，通常你希望React可以为其制定初始值，它仅会被渲染一次,在后续的渲染时并不起作用。解决这个问题的办法是你可以指定一个defaultValue属性而不是value。


```
class Form extends Component {
  handleSubmitClick = () => {
    const name = this._name.value;
    // do something with `name`
  }

  render() {
    return (
      <div>
        <input type="text" ref={input => this._name = input} defaultValue="Bob"/>
        <button onClick={this.handleSubmitClick}>Sign up</button>
      </div>
    );
  }
}
```
[官网文章](https://goshakkk.name/controlled-vs-uncontrolled-inputs-react/)
[](https://segmentfault.com/a/1190000012404114?utm_source=tag-newest)

