title: 浅谈React lifecycle和优化实践
speaker: Simmer
url: 可以设置链接
transition: slide2
files: <link rel="stylesheet" href="/css/a.css"></style><script src="/js/b.js"></script>


[slide]
# 浅谈React lifecycle及优化实践(上)
<p style="text-align: center !important;">By Simmer  React源码版本: V15.0.0 stable </p>


[slide]
<div class="title">React render-越少越快越好！</div>
为何要阻止rerender?
render 运行component的render方法，接着运行diff算法对比新旧virtual DOM
 * 新旧virtual DOM不一致进行更新 {:&.rollIn}
 * 新旧virtual DOM一致，不进行更新
实际上，render方法应该保持纯净，仅仅负责渲染，没有副作用。所以，可以说，数据一致则返回的virtual DOM一定不变。也就是说我们完全可以在rerender之前就阻止render的进一步发生。也就是在`shouldComponentUpdate`方法里根据传来的nextProps、nextState等进行判断

[slide]
<div class="title">React render-越少越快越好！</div>
React更新的时机有2点：
 * 其一：组件接收到了新的`props`，则从`componentWillReceiveProps`方法开始执行组件自定义生命周期方法 {:&.rollIn}
 * 其二：组件包含自定义状态即`state`，state发生变化进而导致从`shouldComponentUpdate`方法开始执行组件自定义更新生命周期方法
 * 所以我们就可以从这2点进行优化。

[slide]
<div class="title">React render-越少越快越好！</div>
## 优化点一： 组件接收到了新的`props`
  * 好好利用`shouldComponentUpdate` {:&.rollIn}
  ```jsx
  shouldComponentUpdate: function(nextProps, nextState, context){
    if(nextProps.data.itemId !== this.props.data.itemId){
      return false;
    }
    return true;
  }
  // 如果保证组件本身并没有很深的数据，则直接使用 React.PureComponent无需重复定义shouldComponentUpdate
  ```
 * 对于哪些只负责渲染数据的组件，何不使用`stateLessComponent`?
 ```jsx
 function item(props){
   return (
     <div class="m-product">
       <h1>{props.header}</h1>
       <p>{props.desc}</p>
       ...
     </div>
     );
 }
 ```
[slide]
<div class="title">React render-越少越快越好！</div>
## 优化点2 新的state来了 

## 优化
慎重使用forceUpdata方法

## 思考
React将状态机的概念进行了抽象，搬迁到了前端，view的不同的状态下代表着不同的状态，正是由这些生命周期方法才能使得组件能够在这些不同的状态之间进行变化，从而满足我们的业务逻辑需求。
后续的深入思考点包括但不限于：
 * 何时应该使用setState?
----
 * React 组件之间的通行方式都有哪些？
----
 * 如何理解React的事件绑定机制？
----
 * React的不足之处有哪些？
----
 * pure component 概念
 * 在React中如何和DOM打交道？
 * JSX语法
 * 与Redux/Reflex/Flux等框架的结合


数据拼装

  *服务器返回的数据需要经过前端进行拼装和处理
 

