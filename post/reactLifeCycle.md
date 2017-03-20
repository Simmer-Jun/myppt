title: 浅谈React lifecycle和优化实践
speaker: Simmer
url: 可以设置链接
transition: slide2
files: <link rel="stylesheet" href="/css/a.css"></style><script src="/js/b.js"></script>


[slide]
# 浅谈React lifecycle及优化实践(上)
<p style="text-align: center !important;">By Simmer  React源码版本: V15.0.0 stable </p>


[slide]
## 先来看一个栗子
下面是Wap端封装的一个Lazyload组件。
```jsx
var Lazyload = React.createClass({
   // 忽略
  componentDidMount:function(){
      this.reloadLazy();
  },

  componentWillReceiveProps: function() {
      this.reloadLazy();
  },
  reloadLazy:function(){
    $(this.refs.img).lazyload({
        effect : "fadeIn",
        threshold : 200,
        error: this.handleError,
     });
  },
	render:function(){
		return(
      <div className={classNames("m-lazyload " + this.props.className, {
        error: this.state.error,
      })}>
      <img ref="img" data-original={this.props.src} onLoad={this.handleLoad}/>
      </div>
		);
	}
});

module.exports = Lazyload;

```
[slide]
## 当谈到React时你会想起什么？
 * 1、Virtual DOM {:&.rollIn}
 * 2、一切皆为Component
 * 3、生命周期
 * 4、Diff 算法
 * 5、有限状态机
 * 6、Just View library
 * 7、JSX
 * 8、Redux/Flux/Reflux...
 * 9、React-Router
 * etc...
 

[slide]
## 目录
 * render的玄机－Babel转译JSX {:&.rollIn}
 * 组成React世界的子元素－ReactNode
 * 保证渲染的有条不紊－React Internal Component
 * 创建React组件
 * ReactCompositeComponent－你好，自定义组件！

[slide]
<div class="title">render的玄机－Babel转译JSX</div>
我们来看看Babel转译JSX之后的代码:
 * Befor:![JSX](/img/jsx1.jpeg)
 
[slide]
 <div class="title">render的玄机－Babel转译JSX</div>
 我们来看看Babel转译JSX之后的代码:
 * After:![JSX](/img/babel1.jpeg) 

[slide]
<div class="title">render的玄机－Babel转译JSX</div>
## 先来思考一个问题
为什么，所有自定义的React组件在最外层包上一层标签？


[slide]
<div class="title">组成React世界的子元素－ReactNode</div>
## 打开React世界的大门
一切皆为React Node，React Node分为2种:
 * `plan text` {String || Number || null} {:&.rollIn}
 * `React Element` {Object}. Vitual DOM就是React在内存中维护的一整套与DOM相对应的一系列对象，这些对象就是React Element.
 
 [slide]
 <div class="title">组成React世界的子元素－ReactNode</div>
 ## 来聊聊React.createElement
 ```js
 ReactElement.createElement = function(type, config, children) {

   var props = {};
   var key = null; // for React diff 算法
   // 将传入的config进行处理
   //省略
   return ReactElement(
     type,// typeof type {String || Number 
     key, // || function==> normal function or constructors}
     ref, // reference 引用
     self, // config.__self === undefined ? null : config.__self
     source, //config.__source === undefined ? null : config.__source
     ReactCurrentOwner.current, // ReactCompositeComponent接口对象
     props // props.children指向child node {Array}
   );
 }
 ```
 [slide]
 <div class="title">组成React世界的子元素－ReactNode</div>
 ## 语法糖－React.createFactory 
```js
ReactElement.createFactory = function(type) {
  var factory = ReactElement.createElement.bind(null, type);
  // Expose the type on the factory and the prototype so that it can be
  // easily accessed on elements. E.g. `<Foo />.type === Foo`.
  // This should not be named `constructor` since this may not be the function
  // that created the element, and it may not even be a constructor.
  // Legacy hook TODO: Warn if this is accessed
  factory.type = type;
  return factory;
};
```
[slide]
<div class="title">组成React世界的子元素－ReactNode</div>
## 总结一下React的世界
 * 一切皆为React Node，可以大致将React Node分为2种进行讨论 {:&.rollIn}
 * 不通过ReactElement.createElement创建的child,即没有嵌套标签的纯文字或数字，一般不会独立出现，需要作为其它组件的child来存在：
 ```js
 React.createElement('p', null, "Just a text.");
 ```
 * 通过`ReactElement.createElement`创建出来的child {React Element}，这种React Element又可以分为2种：
 * 嵌套的标签为与Normal DOM(h1、p、input)等对应标签的React Element. 
 ```js
 ReactElement.createElement('h1', {property1Name: property1Value[,property2Name: property2Value...]}[, child1, child2...]);
 ``` 
 * 自定义标签组件 
 ```js
 ReactElement.createElement(Nav, {property1Name: property1Value[,property2Name: property2Value...]}[, child1, child2...]);
 ```
 * 自定义组件其实就是返回了一个ReactElement 这个Element的type指向的是自定义组件的构造器|纯函数


[slide]
<div class="title">保证渲染的有条不紊－React Internal Component</div>
### 接下来聊聊 React Internal Component
针对不同类型的React Node, React内部都定义了相关的"Internal Component"去管理这些Node的 mount、Update、unMount.So 他们的大致类型又有哪些呢？
 * `ReactEmptyComponent` ==> 管理空的ReactNode {:&.rollIn}
 * `ReactDOMTextComponent` ==> 管理纯文字的ReactNode
 * `ReactDOMComponent` ==> 管理与浏览器标准DOM相对应的ReactNode 
 * `ReactCompositeComponent` ==> 管理自定义ReactNode


一些细节：
* 对于字符串在V15.0.0 之前处理空字符串是将它外层包裹上一层span标签之后则是当作纯文本插入父元素内部，而之后则是作为一个inlineText存在  {:&.rollIn}

[slide]
<div class="title">保证渲染的有条不紊－React Internal Component</div>
在了解instantiateReactComponent之前，先来看看React Node与对应的React Component之间的关系。
![ReactComponent](/img/ReactComponent.png)
 
[slide]
<div class="title">保证渲染的有条不紊－React Internal Component</div>
## 源码分析 - instantiateReactComponent.js
 * `ReactDOM.render`就是一个挂载点. {:&.rollIn}
  ```js
  ReactDOM.render(ReactElement, DOMElement[,callback]);
  ```
前面我们说过，ReactElement的类型有很多种，那么React又是如何去确定不同的ReactElement类型进而实例化不同的`Internal Component`去负责ReactElement的渲染、更新及销毁的呢？
 * 相信你大概猜到了，一定有一个类似的方法会根据不同的ReactElement类型去生成对应的`Internal Component`.
 * 没错，你猜对了！它就是`instantiateReactComponent`


[slide]
<div class="title">保证渲染的有条不紊－React Internal Component</div>
## instantiateReactComponent
 ```js
 /**
  * Given a ReactNode, create an instance that will actually be mounted.
  *
  * @param {ReactNode} node
  * @param {boolean} shouldHaveDebugID
  * @return {object} A new instance of the element's constructor.
  * @protected
  */
 function instantiateReactComponent(node, shouldHaveDebugID) {
   var instance;

   if (node === null || node === false) {
     instance = ReactEmptyComponent.create(instantiateReactComponent);// 创建empty组件
   } else if (typeof node === 'object') { // node为ReactElement
       var element = node;                // 标签组件或者自定义组件
       var type = element.type;           // "div"{String} or "Nav" {Function}

       // Special case string values
       if (typeof element.type === 'string') {// 标签组件
         instance = ReactHostComponent.createInternalComponent(element); // ReactDOMComponent
       } else if (isInternalComponentType(element.type)) {// 是否是内部标示的React 组件
         
         instance = new element.type(element);

         // We renamed this. Allow the old name for compat. :(
         if (!instance.getHostNode) { // 旧版本兼容处理
           instance.getHostNode = instance.getNativeNode;
         }
       } else {
         instance = new ReactCompositeComponentWrapper(element); // 创建compositeComponent 实例
       }
   } else if (typeof node === 'string' || typeof node === 'number') {
     instance = ReactHostComponent.createInstanceForText(node); // 创建reactTextComponent 实例
   } else {
     // not handler
   }

   // These two fields are used by the DOM and ART diffing algorithms
   // respectively. Instead of using expandos on components, we should be
   // storing the state needed by the diffing algorithms elsewhere.
   instance._mountIndex = 0;
   instance._mountImage = null;


   return instance;
 }
 ```

[slide]
<div class="title">创建React组件</div>
## 何为自定义组件？
React 作为一个view Library，所见即所得。从传统的输入输出来看看待自定义组件：
```jsx

return (
  ...组件模版          输入数据
       v                v
       v                v
    <Login data={this.props.xxx} /> ==> 输出代表DOM结构的ReactElement
  ...
  );
```

```jsx
//输入
const data = {
  src: "http:// xxx.xx.xx.png",
  userName: "Sunny"
}
```
```jsx
// template
const Login = (props) => {
  return (
    <div className="m-loigin">
      <img src={props.userPic} alt="userPicture"/>
      <p>Welcome, {props.userName}!</p>
    </div>);
}
```
```jsx
//输出React.createElement('div', {className: "m-login"},..)
<div class="m-login">
  <img src="http:// xxx.xx.xx.png" alt="userPicture" />
  <p>Welcome, sunny!</p>
</div>
```
[slide]
<div class="title">创建React组件！</div>
## ReactElement Type{Function || string}
考虑以下代码:

```jsx
const Nav = (props) => {
  return (<div>I'm Nav</div>);
}

const App1 = (props) => {
  return (
    <div>
      <Nav />
    </div>
    );
}
// return React.createElement('div', null, React.createElement(Nav, null, null));

const App2 = (props) => {
  return (
    <Nav>
      <div>I'm another Nav</div>
    </Nav>
    );
}
// return React.createElement(Nav, null, React.createElement('div', null, "I'm anther Nav"))
```


[slide]
<div class="title">创建React组件</div>
## 梳理一下思路
* 自定义组件接收输入，自身渲染接收到的数据后，返回一个能够代表相对应的Virtual DOM结构的ReactElement {:&.rollIn}
* 自定义组件除非在组件返回的jsx内部显示调用`{this.props.children}`否则嵌套在自定义组件内部的ReactElement不会被真正的渲染.
* `ReactCompositeComponent`是负责管理我们自定义组件的Component，它是一个构造函数，原型链上包含了对于自定义组件从mount(首次渲染)、update(更新)、unMount(卸载)方法，在这些方法中调用可能存在生命周期方法。
* 在了解`ReactCompositeComponent`之前，我们先来看看有哪几种定义自定义组件的方法：

[slide]
<div class="title">创建React组件</div>
按照创建的方式ES5VsES6:

```jsx
//ES5
var App = React.createClass({ // impureComponent
 render: function(){
   return (
     <div>Hello {this.props.name}</div>
   );
 }
});
var App = function(props){ // stateLessComponent
 return (<div>Hello {props.name}</div>);
}
//ES6
class App extends React.Component { // impureComponent
 constructor(props){
   super(props);
 }
 render(){
   return (<div>Hello {this.props.name}</div>);
 }
}
class App extends React.PureComponent{ // pureComponent
 constructor(props){
   super(props);
 }
 render(){
   return (<div>Hello {this.props.name}</div>);
 }
}
const App = (props) => (<div>Hello {props.name}</div>); // stateLessComponent
```
[slide]
<div class="title">创建React组件</div> 
### 按照创建的组件类型分类：stateLessComponent|pureComponent|impureComponent
---
|stateLessComponent| pureComponent | impureComponent
:-------|:------:|-------:|--------
是否有生命周期方法 |否 | 是 | 是
支持ES5语法 | 是 | 否(只能通过React官方提供的mixin实现) | 是
能否接收Props | 能 | 能 | 能
是否有state | 否 | 是 | 是
特点 | 纯函数，无脑接收数据进行渲染 | 包含生命周期，组件已自定义`shouldComponentUpdate`方法进行浅比较 | 拥有完整可定制化的生命周期，组合性更强
是否需要作为构造器使用 | 否 | 是，在ReactCompositeComponent内部实例化 | 是，在ReactCompositeComponent内部实例化



[slide]
<div class="title">创建React组件</div>
## 回顾 `React.createClass` Vs `React.Component` Vs `React.PureComponent`Vs `React.stateLessComponent`
 * 揭秘ES5与ES6创建React组件方式不同的本质 {:&.rollIn}
 * 以及为何ES6需要我们手动绑定各种事件处理函数

[slide]
<div class="title">创建React组件</div>
## 先来看看React.Component
```js
/**
 * Base class helpers for the updating state of a component.
 */
function ReactComponent(props, context, updater) {
  this.props = props; // props
  this.context = context; // 潜逃组件内部共享的对象 一般很少使用，不利于组件复用
  this.refs = emptyObject;
  // var updateQueue = transaction.getUpdateQueue();// 得到更新队列 ReactCompositeComponent.js
  // return new Component(publicProps, publicContext, updateQueue);
  this.updater = updater || ReactNoopUpdateQueue; // 在render的时候注入
}
// 标示这是react用来标识组件与纯函数的区别
ReactComponent.prototype.isReactComponent = {}; // React组件标识
ReactComponent.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState);
  if (callback) {// 放入更新回调队列
    this.updater.enqueueCallback(this, callback, 'setState');
  }
};
// 强制更新
ReactComponent.prototype.forceUpdate = function(callback) {
  // 设置internalInstance._forceUpdate = true
  this.updater.enqueueForceUpdate(this);
  if (callback) { // 加入更新回调队列
    this.updater.enqueueCallback(this, callback, 'forceUpdate');
  }
};
```

[slide]
<div class="title">创建React组件</div>
## ReactPureComponent
```js
/**
 * Base class helpers for the updating state of a component.
 */
function ReactPureComponent(props, context, updater) {
  // Duplicated from ReactComponent.
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer. 依赖注入更新队列
  this.updater = updater || ReactNoopUpdateQueue;
}

function ComponentDummy() {}
ComponentDummy.prototype = ReactComponent.prototype;
ReactPureComponent.prototype = new ComponentDummy(); // 继承React.Component
ReactPureComponent.prototype.constructor = ReactPureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(ReactPureComponent.prototype, ReactComponent.prototype);
ReactPureComponent.prototype.isPureReactComponent = true;
 // 标识 在生命周期shouldComponentUpdate中有用到
```
[slide]
<div class="title">创建React组件</div>
## createClass揭秘
```js
createClass: function(spec) {
  // To keep our warnings more understandable, we'll use a little hack here to
  // ensure that Constructor.name !== 'Constructor'. This makes sure we don't
  // unnecessarily identify a class without displayName as 'Constructor'.
  var Constructor = identity(function(props, context, updater) {

    // if (__DEV__) {
    //   warning(
    //     this instanceof Constructor,
    //     'Something is calling a React component directly. Use a factory or ' +
    //     'JSX instead. See: https://fb.me/react-legacyfactory'
    //   );
    // }

    // 将this.__reactAutoBindPairs的方法绑定到实例上
    if (this.__reactAutoBindPairs.length) {
      bindAutoBindMethods(this);
    }

    this.props = props; // 设置props context 
    this.context = context;
    this.refs = emptyObject;
    this.updater = updater || ReactNoopUpdateQueue;

    this.state = null;

    // 由于Constructor初始化的过程不对外暴露
    // 只有通过原型上的getInitialState钩子方法来获取初始化state
    // 但是ES6语法创建的则不同
    var initialState = this.getInitialState ? this.getInitialState() : null;
    this.state = initialState;
  });
  // 原型链继承
  Constructor.prototype = new ReactClassComponent(); // 继承自ReactClass
  Constructor.prototype.constructor = Constructor;
  Constructor.prototype.__reactAutoBindPairs = [];// 自动绑定到inst上的handler

  // 注入mixin
  injectedMixins.forEach(
    mixSpecIntoComponent.bind(null, Constructor)
  );

  // 将生命周期函数混入 Constructor.prototype
  // 将绑定的各种handler 统一放到__reactAutoBindPairs中
  mixSpecIntoComponent(Constructor, spec);

  // 在构造器生成中就初始化了defaultProps 
  // 这也就是为何在整个生命周期中，多次render同一个组件
  // getDefaultProps方法仅仅只调用一次的原因
  if (Constructor.getDefaultProps) {
    Constructor.defaultProps = Constructor.getDefaultProps();
  }

  //减少原型链查找的时间
  for (var methodName in ReactClassInterface) {
    if (!Constructor.prototype[methodName]) {
      Constructor.prototype[methodName] = null;
    }
  }
   // 返回构造函数   React.createElement(Constructor, config, child)
  return Constructor;
},
```
[slide]
<div class="title">创建React组件</div>
### createClass中的ReactClassComponent
ReactClassComponent其实就是一个通过`Object.assign`扩展原型链的空函数
```js
var ReactClassComponent = function() {};
Object.assign(
  ReactClassComponent.prototype,
  ReactComponent.prototype,
  ReactClassMixin
);

//ReactClassMixin
/**
 * Add more to the ReactClass base class. These are all legacy features and
 * therefore not already part of the modern ReactComponent.
 */
var ReactClassMixin = {

  /**
   * TODO: This will be deprecated because state should always keep a consistent
   * type signature and the only use case for this, is to avoid that.
   */
  replaceState: function(newState, callback) {
    this.updater.enqueueReplaceState(this, newState);
    if (callback) {
      this.updater.enqueueCallback(this, callback, 'replaceState');
    }
  },
  isMounted: function() {
    return this.updater.isMounted(this);
  },
};
// 定义废弃属性的get Wraning
if ("development" !== 'production') {
  var deprecatedAPIs = {
    getDOMNode: ['getDOMNode', 'Use ReactDOM.findDOMNode(component) instead.'],
    isMounted: ['isMounted', 'Instead, make sure to clean up subscriptions and pending requests in ' + 'componentWillUnmount to prevent memory leaks.'],
    replaceProps: ['replaceProps', 'Instead, call render again at the top level.'],
    replaceState: ['replaceState', 'Refactor your code to use setState instead (see ' + 'https://github.com/facebook/react/issues/3236).'],
    setProps: ['setProps', 'Instead, call render again at the top level.']
  };
  var defineDeprecationWarning = function (methodName, info) {
    if (canDefineProperty) {
      Object.defineProperty(ReactComponent.prototype, methodName, {
        get: function () {
          "development" !== 'production' ? warning(false, '%s(...) is deprecated in plain JavaScript React classes. %s', info[0], info[1]) : undefined;
          return undefined;
        }
      });
    }
  };
  for (var fnName in deprecatedAPIs) {
    if (deprecatedAPIs.hasOwnProperty(fnName)) {
      defineDeprecationWarning(fnName, deprecatedAPIs[fnName]);
    }
  }
}

```

[slide]
<div class="title">创建React组件</div>
### StatelessComponent
在ReactCompositeComponent中定于了StatelessComponent类
```js
// inst = new StatelessComponent(Component)
// 为了保证ReactCompositeComponent接口调用的一致性
function StatelessComponent(Component) {
}
StatelessComponent.prototype.render = function() {
  var Component = ReactInstanceMap.get(this)._currentElement.type;// 拿到当前的stateless函数
  var element = Component(this.props, this.context, this.updater);// 调用函数传入props context updater
  return element;
};
```

[slide]
<div class="title">创建React组件</div>
通过React.createClass和extends React.Component创建组件
---
|React.createClass| extends React.Component 
:-------|:------:|-------:|--------
是否支持自动绑定 |是 | 否(需要在constructor中手动bind各种handler) 
可定义getDefaultProps方法 | 可以(只执行一次) | 否(通过Constructor.defaultProps = {your props}实现)
可定义getInitialState方法 | 可以(多次执行) | 否(通过Constructor中 this.state = {your defaultProps}实现)
是否有replaceState方法 | 是(通过createClass创建类特有方法,会在未来遗弃) | 否


[slide]
<div class="title">创建React组件</div>
### 小总结
 * `React.Component`其实就是一个在原型上定义了setState、forceUpdate方法和isReactComponent标识的函数对象  {:&.rollIn}
 * `React.createClass`在内部使得返回的构造函数继承React.Component，及添加了2个将会被遗弃的方法(replaceState|isMounted)并为我们提供了`this`的自定绑定
 * `React.PureComponent`其实就是React.Component的一个子类，原型上定义了`isPureReactComponent`属性方便后面的ReactCompositeComponent识别该自定义组件类型
 * `React.StatelessComponent` inst = new StatelessComponent(component)为了统一ReactCompositeComponent管理inst强行定义的一个类，这个类原型上只有一个`render`方法，返回的是组件中return的的ReactElement

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## 思考一下？
在React中负责渲染不同的ReactElement的React Internal Component 之间是如何通信的？
```jsx
class App extends React.Component {
  //ReactElement {React.createElement(App...)} ==> ReactElement {React.createElement('div',)}
  componentWillMount(){}
  render(){
    <div className="wraper">
      <Nav data={this.props.navData} />
      <Body data={this.props.bodyData} />
    </div>
  }
  componentDidMount(){} ...etc
}
//负责PureContainer和Nav组件渲染的React "Internal" Component组件如何进行通信？
const PureContainer = (props) => {
  return (
    <Nav data={this.props.data} />
    );
}
```
[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## ReactReconciler React组件协调器
```js
var ReactReconciler = {
  //渲染组件
  mountComponent : function(arguments) {...},
  //获取host node
  getHostNode : function(internalInstance) {...},
  // 卸载组件
  unmountComponent : function(arguments) {...},
  // Update a component using a new element.
  receiveComponent : function(internalInstance, nextElement...) {
  //ref change ?
  internalInstance.receiveComponent(nextElement, Transaction, Context...)
  },
  // 更新dirty change  其实就是调用ReactCompositeComponent的performUpdateIfNecessary
  performUpdateIfNecessary: function(internalInstance, transaction, updateBatchNumber){...},
}
```
[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
协调器作为不同类型的React组件之间沟通的桥梁，所包含的方法几乎是所有的组件类型共有的。
  * 类似ReactCompositeComponent，ReactDOMComponent和ReactTextComponent组件都有`mountComponent`、`unmountComponent`、`receiveComponent`方法 {:&.rollIn}

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
##一张图理清ReactReconciler和其它组件之间的关系
![React协调器](/img/ReactReconciler.png)


[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## 来看看ReactCompositeComponent构造函数：
```js
/**
 * Base constructor for all composite component.
 *
 * @param {ReactElement} element
 * @final
 * @internal
 */
construct: function(element) { // 传入ReactElement 
  this._currentElement = element;// _currentElement保存代表当前自定义组件的ReactElement
  this._rootNodeID = 0;          // 调用React.createElement(constructor, null)返回的React Element 
  this._compositeType = null;
  this._instance = null; 
  this._hostParent = null;
  this._hostContainerInfo = null;

  // See ReactUpdateQueue
  this._updateBatchNumber = null; 
  this._pendingElement = null; // 等待更新的ReactElement
  this._pendingStateQueue = null; // 等待更新的 state队列 {Array}
  this._pendingReplaceState = false; // for this.replaceState API React在将来的版本中将会被遗弃
  this._pendingForceUpdate = false; // this.forceUpdate API

  this._renderedNodeType = null;
  this._renderedComponent = null;
  this._context = null;
  this._mountOrder = 0;
  this._topLevelWrapper = null;

  // See ReactUpdates and ReactUpdateQueue.
  this._pendingCallbacks = null;

  // ComponentWillUnmount shall only be called once
  this._calledComponentWillUnmount = false;

}
```

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## 回顾
下面将会对compositeComponent划分为3个阶段讲解：
 * `mountComponent`:组件初次渲染阶段 {:&.rollIn}
 * `updateComponent`: 组件更新阶段
 * `unMountComponent`: 组件销毁阶段
 
[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## step 1 - mountComponent 
mountComponent:组件初次渲染时，调用的组件生命周期方法为
  * `getDefaultProps`(如果是组件第一次挂载) ES6则是通过 App.defaultProps = {obj} 来实现效果类似ES5 {:&.rollIn}
  * `getInitialState` 组件初始化 对ES6则是在constructor中通过this.state={initialStateObj}来完成state的初始化。
  * `componentWillMount` 在这里设置setState将不会触发rerender!!
  * `render` 渲染React Node 返回ReactElement ES5&&ES6必须定义!
  * `componentDidUpdate` 渲染完毕之后React将会调用，由于递归的原因，父组件的该方法一定比子组件调用晚

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## 在上代码之前先看看整体流程图

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## step 1 - compositeComponent.mountComponent
```js
mountComponent: function(
  transaction, // React事务
  hostParent, 
  hostContainerInfo,
  context 
) {
  this._context = context; 
  this._mountOrder = nextMountID++; // 渲染id
  this._hostParent = hostParent;
  this._hostContainerInfo = hostContainerInfo;

  var publicProps = this._currentElement.props; // 拿到当前React element 接收到的props
  var publicContext = this._processContext(context);
    // currentElement.type constructor => component
  var Component = this._currentElement.type; 

  var updateQueue = transaction.getUpdateQueue();// 得到更新队列

  // Initialize the public class
  var doConstruct = shouldConstruct(Component); //这里判断是stateLess组件还是pure/Impure组件
  var inst = this._constructComponent(// 拿到对应的实例 inst = new constructor(props,...etc)
    doConstruct,                     // 这里运行getInitialState
    publicProps,                 //如果是stateLessComponent返回的inst为ReactElement
    publicContext,
    updateQueue
  );
  var renderedElement;   // render函数返回的ReactElement

  // Support functional components  
  // 存函数组件，并非通过React.createClass创建
  if (!doConstruct && (inst == null || inst.render == null)) {
    renderedElement = inst; // 无状态组件 
    inst = new StatelessComponent(Component);// 获取真实的渲染实例
    this._compositeType = CompositeTypes.StatelessFunctional; // 设置compositeType
  } else {
    if (isPureComponent(Component)) {// component.prototype.isPureComponent ?
      this._compositeType = CompositeTypes.PureClass;
    } else {
      this._compositeType = CompositeTypes.ImpureClass;
    }
  }
  // 这里的inst 就是我们在组件中的this
  inst.props = publicProps; // 设置实例的props  React.createElement(App, {key:value})
  inst.context = publicContext;
  inst.refs = emptyObject;
  inst.updater = updateQueue; // 更新队列 依赖注入

  this._instance = inst; // 保存指向实例的引用
  
  var initialState = inst.state; // 初始化state
  if (initialState === undefined) {
    inst.state = initialState = null;
  }

  this._pendingStateQueue = null; // state队列 {Array}
  this._pendingReplaceState = false;  // for replaceState API 
  this._pendingForceUpdate = false; // invoked by this.forceUpdate

  var markup;
  markup = this.performInitialMount(renderedElement, hostParent, hostContainerInfo, transaction, context);

  // 如果实例上存在componentDidMount方法 则加入回调队列队列
  if (inst.componentDidMount) {
  transaction.getReactMountReady().enqueue(inst.componentDidMount, inst);
  }
  return markup;
},

```

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## 你可能会疑惑
你可能会疑惑，React究竟是在何处处理传入的props和defaultProps的，答案就是在`React.createElement`中
```js
// ReactElement.js
// Resolve default props
// type = React.createElement(type, config, child)
if (type && type.defaultProps) {
  var defaultProps = type.defaultProps;
  for (propName in defaultProps) {
    if (props[propName] === undefined) {
      props[propName] = defaultProps[propName];
    }
  }
}
```

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## step 2 - updateComponent
现在我们来看看当组件更新的时候运行的生命周期函数
 * `componentWillReceiveProps(nextProps, nextState, context)` 可以setState当组件接收新的props的时候调用 {:&.rollIn}
 * `shouldComponentUpdate(nextProps, nextState, context)` 根据这个函数的返回值来决定是否执行接下来的更新 不可以setState!!!
 * `componentWillUpdate(nextPros, nextState, context)` 在ren方法之前执行，到这里的话就已经证明了React一定会进行rerender了! 注意：这里还是不能进行setState!!!
 * `render` rerender 可以setState但是不推荐,render 函数最好只负责渲染数据不负责其它业务逻辑
 * `componentDidUpdate(prevProps, prevState, context)` 真实DOM更新完毕后调用 可以setState但是不推荐

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## 在上代码之前先看看整体流程图

[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
3个函数receiveComponent、updateComponent、performUpdateIfNecessary，他们之间的关系是：
```jsx
              ReactReconciler.receiveComponent
 receiveComponent( <============ performUpdateIfNecessary(transaction)
    nextElement,                   v
    transaction,                   v
    nextContext)                   v 
             v                     v
             v                     v
            updateComponent: function(
              transaction,
              prevParentElement,
              nextParentElement,
              prevUnmaskedContext,
              nextUnmaskedContext
            )
```
[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## ReactCompositeComponent update
```js
// nextElement 是新的 React.createElement(Constructor, config, child)
receiveComponent: function(nextElement, transaction, nextContext) {
  var prevElement = this._currentElement; // 先前的ReactElement
  var prevContext = this._context; // 上下文

  this._pendingElement = null; // _pendingElement 置为空

  this.updateComponent(//调用updateComponent
    transaction,
    prevElement,
    nextElement,
    prevContext,
    nextContext
  );
},
/**
 * If any of `_pendingElement`, `_pendingStateQueue`, or `_pendingForceUpdate`
 * is set, update the component.
 *
 * @param {ReactReconcileTransaction} transaction
 * @internal
 */
performUpdateIfNecessary: function(transaction) {
  if (this._pendingElement != null) {
    ReactReconciler.receiveComponent(
      this,
      this._pendingElement,
      transaction,
      this._context
    );
  } else if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
    this.updateComponent(
      transaction,
      this._currentElement,
      this._currentElement,
      this._context,
      this._context
    );
  } else {
    this._updateBatchNumber = null;
  }
},
/**
 * Perform an update to a mounted component. The componentWillReceiveProps and
 * shouldComponentUpdate methods are called, then (assuming the update isn't
 * skipped) the remaining update lifecycle methods are called and the DOM
 * representation is updated.
 *
 * By default, this implements React's rendering and reconciliation algorithm.
 * Sophisticated clients may wish to override this.
 *
 * @param {ReactReconcileTransaction} transaction
 * @param {ReactElement} prevParentElement
 * @param {ReactElement} nextParentElement
 * @internal
 * @overridable
 */
updateComponent: function(
  transaction, // React事务
  prevParentElement,
  nextParentElement,
  prevUnmaskedContext,
  nextUnmaskedContext
) {
  var inst = this._instance; //保存原先的实例
  invariant(
    inst != null,
    'Attempted to update component `%s` that has already been unmounted ' +
    '(or failed to mount).',
    this.getName() || 'ReactCompositeComponent'
  );

  var willReceive = false;
  var nextContext;

  // Determine if the context has changed or not
  if (this._context === nextUnmaskedContext) {
    nextContext = inst.context;
  } else {
    nextContext = this._processContext(nextUnmaskedContext);
    willReceive = true;
  }
  // React.createElement(constructor, props)
  var prevProps = prevParentElement.props; // 旧的props this.props
  var nextProps = nextParentElement.props; //新的props 

  // Not a simple state update but a props update
  if (prevParentElement !== nextParentElement) {//2次render返回的Element不一样
    willReceive = true;                        // 肯定是接收到了新的props
  }
  // An update here will schedule an update but immediately set
  // _pendingStateQueue which will ensure that any state updates gets
  // immediately reconciled instead of waiting for the next batch.
  if (willReceive && inst.componentWillReceiveProps) {
     //调用实例上的componentWillReceiveProps方法
      inst.componentWillReceiveProps(nextProps, nextContext); 
  }

  // 合并state 注意⚠ 这个操作是在componentWillReceiveProps之后
  // shouldUpdate之前进行的 ！！！很重要
  var nextState = this._processPendingState(nextProps, nextContext);
  var shouldUpdate = true; // 默认shouldComponentUpdate

  if (!this._pendingForceUpdate) {// 没有forceUpdata 强制更新 需要调用可能存在的shouldComponentUpdate
    if (inst.shouldComponentUpdate) {
      // 调用shouldComponentUpdate
      shouldUpdate = inst.shouldComponentUpdate(nextProps, nextState, nextContext);
    } else {// pureComponent 实现了React的props和state的浅比较
            // see https://facebook.github.io/react/docs/react-api.html#react.purecomponent
      if (this._compositeType === CompositeTypes.PureClass) {
        shouldUpdate =
          !shallowEqual(prevProps, nextProps) ||
          !shallowEqual(inst.state, nextState);
      }
    }
  }
  // 清除批量更新
  this._updateBatchNumber = null;
  if (shouldUpdate) {// 需要更新
    this._pendingForceUpdate = false; //将_pendingForceUpdate置为false
    // Will set `this.props`, `this.state` and `this.context`.
    this._performComponentUpdate(// 组件更新
      nextParentElement, //new React Element
      nextProps,
      nextState,
      nextContext,
      transaction, //React 事务
      nextUnmaskedContext
    );
  } else {// 就算是在shouldUpdate里面返回false 仍旧将新的props及state更新到inst实例上
    // If it's determined that a component should not update, we still want
    // to set props and state but we shortcut the rest of the update.
    this._currentElement = nextParentElement;
    this._context = nextUnmaskedContext;
    inst.props = nextProps;
    inst.state = nextState;
    inst.context = nextContext;
  }
},
  
```
[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## step 3 - unmountComponent 再见！自定义组件
组件销毁进入销毁期间，注意：可以使用ReactDOM.umMountComponentAt(DOM Container)API来手动销毁组件
 * `componentWillUnmount` 组件销毁之前调用，这里做一些组件清理的工作(例如清除定时器等)，清除自定义实现的相关事件等
  
[slide]
<div class="title">ReactCompositeComponent－你好，自定义组件！</div>
## step 3 - unmountComponent 再见！自定义组件
```js
unmountComponent: function(safely) {
  if (!this._renderedComponent) { // 负责管理 render返回的ReactElement 的component
    return;
  }

  var inst = this._instance;

  if (inst.componentWillUnmount && !inst._calledComponentWillUnmount) {
    inst._calledComponentWillUnmount = true; // setFlag

    if (safely) {
      var name = this.getName() + '.componentWillUnmount()';
      ReactErrorUtils.invokeGuardedCallback(name, inst.componentWillUnmount.bind(inst));
    } else {
      //调用 componentWillUnmount
      inst.componentWillUnmount();
    }
  }

  if (this._renderedComponent) {// 递归销毁组件
    ReactReconciler.unmountComponent(this._renderedComponent, safely);
    this._renderedNodeType = null;
    this._renderedComponent = null;
    this._instance = null;
  }

  // Reset pending fields
  // Even if this component is scheduled for another update in ReactUpdates,
  // it would still be ignored because these fields are reset.
  this._pendingStateQueue = null;
  this._pendingReplaceState = false;
  this._pendingForceUpdate = false;
  this._pendingCallbacks = null;
  this._pendingElement = null;

  // These fields do not really need to be reset since this object is no
  // longer accessible.
  this._context = null;
  this._rootNodeID = 0;
  this._topLevelWrapper = null;

  // Delete the reference from the instance to this internal representation
  // which allow the internals to be properly cleaned up even if the user
  // leaks a reference to the public instance.
  ReactInstanceMap.remove(inst);

  // Some existing components rely on inst.props even after they've been
  // destroyed (in event handlers).
  // TODO: inst.props = null;
  // TODO: inst.state = null;
  // TODO: inst.context = null;
},
```
[slide]
## 总结
 * 组件返回一个代表组件DOM结构的ReactElement，只不过Element.type ==> function(构造函数或者无状态组件函数) {:&.rollIn}
 * stateLess、pureComponent、ImpureComponent之间的区别
 * ReactCompositeComponent 负责自定义组件的mount、update、unmount(递归进行)
 * ReactReconciler 协调器让不同类型组件之间进行通行
 * ES5VsES6创建自定义组件区别及为什么(生命周期函数、绑定事件)

[slide]
## 解答一开始提出的代码问题
```jsx
var Lazyload = React.createClass({
   // 忽略
  componentDidMount:function(){
      this.reloadLazy();
  },

  shouldComponentUpdate: function(nextProps, nextState) {
      return if(this.props.src !== nextProps.src);
  },
  reloadLazy:function(){
    $(this.refs.img).lazyload({
        effect : "fadeIn",
        threshold : 200,
        error: this.handleError,
     });
  },
	render:function(){
		return(
      <div className={classNames("m-lazyload " + this.props.className, {
        error: this.state.error,
      })}>
      <img ref="img" data-original={this.props.src} onLoad={this.handleLoad}/>
      </div>
		);
	},
  componentDidUpdate: function(prevProps, prevState){
    this.reloadLazy();
  }
});

module.exports = Lazyload;

```
[slide]
## 写在后面
React自定义组件生命周期源码到这里分析的差不多了，下一次的分享我将会为大家解答以下疑惑:
* 1 多次ReactDOM.render生命周期函数的运行疑惑 {:&.rollIn}
* 2 amazing setState， setState是如何实现异步机制的？
* 3 transaction的魅力，无处不在的React transaction是如何帮助React有条不紊的渲染更新等操作的
* 4 开发实践，在开发中知道这些知识，我们能够做一些怎样的力所能及的优化？


[slide]
## Thx

