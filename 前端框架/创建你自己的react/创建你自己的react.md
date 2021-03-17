<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [1.(Didact)一个 DIY 教程:创建你自己的 react](#1didact一个diy教程创建你自己的react)
  - [1.1 引言](#11-引言)
- [2.渲染 dom 元素](#2渲染dom元素)
  - [2.1 什么是 DOM](#21-什么是dom)
  - [2.2 Didact 元素](#22-didact元素)
  - [2.3 渲染 dom 元素](#23-渲染dom元素)
  - [2.4 渲染 DOM 文本节点](#24-渲染dom文本节点)
  - [2.5 总结](#25-总结)
- [3.JSX 和创建元素](#3jsx和创建元素)
  - [3.1 JSX](#31-jsx)
  - [3.2 总结](#32总结)
- [4.虚拟 DOM 和调和过程](#4虚拟dom和调和过程)
  - [4.1 虚拟 DOM 和调和过程](#41-虚拟dom和调和过程)
  - [4.2 实例(instance)](#42-实例instance)
  - [4.3 重构](#43-重构)
  - [4.4 复用 dom 节点](#44-复用dom节点)
  - [4.5 子元素的调和](#45-子元素的调和)
  - [4.6 删除 Dom 节点](#46-删除dom节点)
  - [4.7 总结](#47-总结)
- [5.组件和状态(state)](#5组件和状态state)
  - [5.1 回顾](#51-回顾)
  - [5.2 组件类](#52-组件类)
- [6.Fiber:增量调和](#6fiber增量调和)
  - [6.1 为什么使用 Fiber](#61-为什么使用fiber)
  - [6.2 调度微任务(micro-task)](#62-调度微任务micro-task)
  - [6.3 fiber 数据结构](#63-fiber数据结构)
  - [6.4 Didact 的调用层次](#64-didact的调用层次)
  - [6.5 之前的代码](#65-之前的代码)

<!-- /code_chunk_output -->

#1.(Didact)一个 DIY 教程:创建你自己的 react

[更新]这个系列从老的 react 架构写起，你可以跳过前面直接到，使用新的 fiber 架构重写的：[文章](https://engineering.hexacta.com/didact-fiber-incremental-reconciliation-b2fe028dcaec)

[更新 2]听 Dan 的没错，我是认真的 ☺

> 这篇深入 fiber 架构的文章真的很棒。
> — @dan_abramov

## 1.1 引言

很久以前，当学数据结构和算法时，我有个作业就是实现自己的数组，链表，队列，和栈（用 Modula-2 语言）。那之后，我再也没有过要自己来实现链表的需求。总会有库让我不需要自己重造轮子。

所以，那个作业还有意义吗？当然，我从中学到了很多，知道如何合理使用各种数据结构，并知道根据场景合理选用它们。

这个系列文章以及对应的（[仓库](https://github.com/hexacta/didact/)）的目的也是一样，不过要实现的是一个，我们比链表使用更多的东西：React

> 我好奇如果不考虑性能和设备兼容性，POSIX(可移植操作系统接口)核心可以实现得多么小而简单。
> — @ID_AA_Carmack

&emsp;&emsp;&emsp;&emsp;&ensp;&nbsp;我对 react 也这么好奇

幸运的是,如果不考虑性能，调试，平台兼容性等等，react 的主要 3，4 个特性重写并不难。事实上，它们很简单，甚至只要不足 200 行代码

这就是我们接下来要做的事，用不到 200 行代码写一个有一样的 API,能跑的 React。因为这个库的说教性(didactic)特点，我们打算就称之为 Didact

用 Didact 写的应用如下：

```js
const stories = [
  { name: 'Didact introduction', url: 'http://bit.ly/2pX7HNn' },
  { name: 'Rendering DOM elements ', url: 'http://bit.ly/2qCOejH' },
  { name: 'Element creation and JSX', url: 'http://bit.ly/2qGbw8S' },
  { name: 'Instances and reconciliation', url: 'http://bit.ly/2q4A746' },
  { name: 'Components and state', url: 'http://bit.ly/2rE16nh' }
]

class App extends Didact.Component {
  render() {
    return (
      <div>
        <h1>Didact Stories</h1>
        <ul>
          {this.props.stories.map((story) => {
            return <Story name={story.name} url={story.url} />
          })}
        </ul>
      </div>
    )
  }
}

class Story extends Didact.Component {
  constructor(props) {
    super(props)
    this.state = { likes: Math.ceil(Math.random() * 100) }
  }
  like() {
    this.setState({
      likes: this.state.likes + 1
    })
  }
  render() {
    const { name, url } = this.props
    const { likes } = this.state
    const likesElement = <span />
    return (
      <li>
        <button onClick={(e) => this.like()}>
          {likes}
          <b>❤️</b>
        </button>
        <a href={url}>{name}</a>
      </li>
    )
  }
}

Didact.render(<App stories={stories} />, document.getElementById('root'))
```

这就是我们在这个系列文章里要使用的例子。效果如下
![pic](./img/demo.gif)

我们将会从下面几点来一步步添加 Didact 的功能：

- [渲染 dom 元素](#2渲染dom元素)
- [JSX 和创建元素](#3jsx和创建元素)
- [虚拟 DOM 和调和过程](#4虚拟dom和调和过程)
- [组件和状态(state)](#5组件和状态state)
- [Fiber:增量调和](#6fiber增量调和)

这个系列暂时不讲的地方：

- Functional components
- Context（上下文）
- 生命周期方法
- ref 属性
- 通过 key 的调和过程（这里只讲根据子节点原顺序的调和）
- 其他渲染引擎 （只支持 DOM）
- 旧浏览器支持
  > 你可以从[react 实现笔记](https://reactjs.org/docs/implementation-notes.html)，[Paul O’Shannessy](https://medium.com/@zpao)的这个 youtube[演讲视频](https://www.youtube.com/watch?v=_MAD4Oly9yg),或者 react[仓库地址](https://github.com/facebook/react),找到更多关于如何实现 react 的细节.
  > #2.渲染 dom 元素
  > ##2.1 什么是 DOM
  > 开始之前，让我们回想一下，我们经常使用的 DOM API

```js
// Get an element by id
const domRoot = document.getElementById('root')
// Create a new element given a tag name
const domInput = document.createElement('input')
// Set properties
domInput['type'] = 'text'
domInput['value'] = 'Hi world'
domInput['className'] = 'my-class'
// Listen to events
domInput.addEventListener('change', (e) => alert(e.target.value))
// Create a text node
const domText = document.createTextNode('')
// Set text node content
domText['nodeValue'] = 'Foo'
// Append an element
domRoot.appendChild(domInput)
// Append a text node (same as previous)
domRoot.appendChild(domText)
```

注意到我们设置元素的属性而不是特性[属性和特性的区别](https://stackoverflow.com/questions/6003819/what-is-the-difference-between-properties-and-attributes-in-html)，只有合法的属性才可以设置。
##2.2 Didact 元素
我们用 js 对象来描述渲染过程，这些 js 对象我们称之为 Didact 元素.这些元素有 2 个属性，type 和 props。type 可以是一个字符串或者方法。在后面讲到组件之前，我们先用字符串。props 是一个可以为空的对象（不过不能为 null）。props 可能有 children 属性,这个 children 属性是一个 Didact 元素的数组。

> 我们将多次使用 Didact 元素，目前我们先称之为元素。不要和 html 元素混淆，在变量命名的时候，我们称它们为 DOM 元素或者 dom([preact](https://github.com/developit/preact)就是这么做的)

一个元素就像下面这样：

```js
const element = {
  type: 'div',
  props: {
    id: 'container',
    children: [
      { type: 'input', props: { value: 'foo', type: 'text' } },
      { type: 'a', props: { href: '/bar' } },
      { type: 'span', props: {} }
    ]
  }
}
```

对应描述下面的 dom：

```html
<div id="container">
  <input value="foo" type="text" />
  <a href="/bar"></a>
  <span></span>
</div>
```

Didact 元素和[react 元素](https://reactjs.org/blog/2014/10/14/introducing-react-elements.html)很像，但是不像 react 那样，你可能使用 JSX 或者 createElement，创建元素就和创建 js 对象一样.Didatc 我们也这么做，不过在后面章节我们再加上 create 元素的代码

##2.3 渲染 dom 元素
下一步是渲染一个元素以及它的 children 到 dom 里。我们将写一个 render 方法(对应于 react 的[ReactDOM.render](https://reactjs.org/docs/react-dom.html#render))，它接受一个元素和一个 dom 容器。然后根据元素的定义生成 dom 树,附加到容器里。

```js
function render(element, parentDom) {
  const { type, props } = element
  const dom = document.createElement(type)
  const childElements = props.children || []
  childElements.forEach((childElement) => render(childElement, dom))
  parentDom.appendChild(dom)
}
```

我们仍然没有对其添加属性和事件监听。现在让我们使用 object.keys 来遍历 props 属性，设置对应的值：

```js
function render(element, parentDom) {
  const { type, props } = element
  const dom = document.createElement(type)

  const isListener = (name) => name.startsWith('on')
  Object.keys(props)
    .filter(isListener)
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2)
      dom.addEventListener(eventType, props[name])
    })

  const isAttribute = (name) => !isListener(name) && name != 'children'
  Object.keys(props)
    .filter(isAttribute)
    .forEach((name) => {
      dom[name] = props[name]
    })

  const childElements = props.children || []
  childElements.forEach((childElement) => render(childElement, dom))

  parentDom.appendChild(dom)
}
```

##2.4 渲染 DOM 文本节点
现在 render 函数不支持的就是文本节点，首先我们定义文本元素什么样子，比如，在 react 中描述\<span>Foo\</span>：

```js
const reactElement = {
  type: 'span',
  props: {
    children: ['Foo']
  }
}
```

注意到子节点，只是一个字符串，并不是其他元素对象。这就让我们的 Didact 元素定义不合适了：children 元素应该是一个数组，数组里的元素都有 type 和 props 属性。如果我们遵守这个规则，后面将减少不必要的 if 判断.所以，Didact 文本元素应该有一个“TEXT ELEMENT”的类型，并且有在对应的节点有文本的值。比如：

```js
const textElement = {
  type: 'span',
  props: {
    children: [
      {
        type: 'TEXT ELEMENT',
        props: { nodeValue: 'Foo' }
      }
    ]
  }
}
```

现在我们来定义文本元素应该如何渲染。不同的是，文本元素不使用 createElement 方法，而用 createTextNode 代替。节点值就和其他属性一样被设置上去。

```js
function render(element, parentDom) {
  const { type, props } = element

  // Create DOM element
  const isTextElement = type === 'TEXT ELEMENT'
  const dom = isTextElement ? document.createTextNode('') : document.createElement(type)

  // Add event listeners
  const isListener = (name) => name.startsWith('on')
  Object.keys(props)
    .filter(isListener)
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2)
      dom.addEventListener(eventType, props[name])
    })

  // Set properties
  const isAttribute = (name) => !isListener(name) && name != 'children'
  Object.keys(props)
    .filter(isAttribute)
    .forEach((name) => {
      dom[name] = props[name]
    })

  // Render children
  const childElements = props.children || []
  childElements.forEach((childElement) => render(childElement, dom))

  // Append to parent
  parentDom.appendChild(dom)
}
```

## 2.5 总结

我们现在创建了一个可以渲染元素以及子元素的 render 方法。后面我们需要实现如何创建元素。我们将在下节讲到如何使 JSX 和 Didact 很好地融合。

# 3.JSX 和创建元素

## 3.1 JSX

我们之前讲到了[Didact 元素](#2渲染dom元素),讲到如何渲染到 DOM，用一种很繁琐的方式.这一节我们来看看如何使用 JSX 简化创建元素的过程。

JSX 提供了一些创建元素的语法糖，不用使用下面的代码：

```js
const element = {
  type: 'div',
  props: {
    id: 'container',
    children: [
      { type: 'input', props: { value: 'foo', type: 'text' } },
      {
        type: 'a',
        props: {
          href: '/bar',
          children: [{ type: 'TEXT ELEMENT', props: { nodeValue: 'bar' } }]
        }
      },
      {
        type: 'span',
        props: {
          onClick: (e) => alert('Hi'),
          children: [{ type: 'TEXT ELEMENT', props: { nodeValue: 'click me' } }]
        }
      }
    ]
  }
}
```

我们现在可以这么写：

```js
const element = (
  <div id='container'>
    <input value='foo' type='text' />
    <a href='/bar'>bar</a>
    <span onClick={(e) => alert('Hi')}>click me</span>
  </div>
)
```

如果你不熟悉 JSX 的话，你可能怀疑下面的代码是否是合法的 js--它确实不是。要让浏览器理解它，上面的代码必须使用预处理工具处理。比如 babel.babel 会把上面的代码转成下面这样：

```js
const element = createElement(
  'div',
  { id: 'container' },
  createElement('input', { value: 'foo', type: 'text' }),
  createElement('a', { href: '/bar' }, 'bar'),
  createElement('span', { onClick: (e) => alert('Hi') }, 'click me')
)
```

支持 JSX 我们只要在 Didact 里添加一个 createElement 方法。其他事的交给预处理器去做。这个方法的第一个参数是元素的类型 type,第二个是含有 props 属性的对象，剩下的参数都是子节点 children。createElement 方法需要创建一个对象，并把第二个参数上所有的值赋给它，把第二个参数后面的所有参数放到一个数组，并设置到 children 属性上，最后返回一个有 type 和 props 的对象。用代码实现很容易：

```js
function createElement(type, config, ...args) {
  const props = Object.assign({}, config)
  const hasChildren = args.length > 0
  props.children = hasChildren ? [].concat(...args) : []
  return { type, props }
}
```

同样，这个方法对文本元素不适用。文本的子元素是作为字符串传给 createElement 方法的。但是我们的 Didact 需要文本元素一样有 type 和 props 属性。所以我们要把不是 didact 元素的参数都转成一个'文本元素'

```js
const TEXT_ELEMENT = 'TEXT ELEMENT'

function createElement(type, config, ...args) {
  const props = Object.assign({}, config)
  const hasChildren = args.length > 0
  const rawChildren = hasChildren ? [].concat(...args) : []
  props.children = rawChildren.filter((c) => c != null && c !== false).map((c) => (c instanceof Object ? c : createTextElement(c)))
  return { type, props }
}

function createTextElement(value) {
  return createElement(TEXT_ELEMENT, { nodeValue: value })
}
```

我同样从 children 列表里过滤了 null，undefined,false 参数。我们不需要把它们加到 props.children 上因为我们根本不会去渲染它们。
##3.2 总结
到这里我们并没有为 Didact 加特殊的功能.但是我们有了更好的开发体验，因为我们可以使用 JSX 来定义元素。我已经更新了[codepen](https://codepen.io/pomber/pen/xdmoWE?editors=0010)上的代码。因为 codepen 用 babel 转译 JSX,所以以/\*_ @jsx createElement _/开头的注释都是为了让 babel 知道使用哪个函数。

你同样可以查看[github 提交](https://github.com/hexacta/didact/commit/15010f8e7b8b54841d1e2dd9eacf7b3c06b1a24b)

下面我们将介绍 Didact 用来更新 dom 的虚拟 dom 和所谓的调和算法.

#4.虚拟 DOM 和调和过程
到目前为止，我们基于 JSX 的描述方式实现了 dom 元素的创建机制。这里开始，我们专注于怎么更新 DOM.

在下面介绍 setState 之前，我们之前更新 DOM 的方式只有再次调用 render()方法，传入不同的元素。比如：我们要渲染一个时钟组件，代码是这样的：

```js
const rootDom = document.getElementById('root')

function tick() {
  const time = new Date().toLocaleTimeString()
  const clockElement = <h1>{time}</h1>
  render(clockElement, rootDom)
}

tick()
setInterval(tick, 1000)
```

我们现在的 render 方法还做不到这个。它不会为每个 tick 更新之前同一个的 div,相反它会新添一个新的 div.第一种解决办法是每一次更新,替换掉 div.在 render 方法的最下面，我们检查父元素是否有子元素，如果有，我们就用新元素生产的 dom 替换它：

```js
function render(element, parentDom) {
  // ...
  // Create dom from element
  // ...

  // Append or replace dom
  if (!parentDom.lastChild) {
    parentDom.appendChild(dom)
  } else {
    parentDom.replaceChild(dom, parentDom.lastChild)
  }
}
```

在这个小列子里，这个办法很有效。但在复杂情况下，这种重复创建所有子节点的方式并不可取。所以我们需要一种方式，来对比当前和之前的元素树之间的区别。最后只更新不同的地方。
##4.1 虚拟 DOM 和调和过程
React 把这种 diff 过程称之为[调和过程](https://reactjs.org/docs/reconciliation.html)，我们现在也这么称呼它。首先我们要保存之前的渲染树，从而可以和新的树对比。换句话说，我们将实现自己的 DOM,虚拟 dom.

这种虚拟 dom 的‘节点’应该是什么样的呢？首先考虑使用我们的 Didact 元素。它们已经有一个 props.children 属性，我们可以根据它来创建树。但是这依然有两个问题,一个是为了是调和过程容易些，我们必须为每个虚拟 dom 保存一个对真实 dom 的引用，并且我们更希望元素都不可变(imumutable).第二个问问题是后面我们要支持组件，组件有自己的状态(state),我们的元素还不能处理那种。
##4.2 实例(instance)
所以我们要介绍一个新的名词：实例。实例代表的已经渲染到 DOM 中的元素。它其实是一个有着，element,dom,chilInstances 属性的 JS 普通对象。childInstances 是有着该元素所以子元素实例的数组。

> 注意我们这里提到的实例和[Dan Abramov](https://medium.com/@dan_abramov)在[react 组件，元素和实列](https://medium.com/@dan_abramov/react-components-elements-and-instances-90800811f8ca)这篇文章提到实例不是一个东西。他指的是 React 调用继承于 React.component 的那些类的构造函数所获得的‘公共实例’(public instances)。我们会在以后把公共实例加上。

每一个 DOM 节点都有一个相应的实例。调和算法的一个目标就是尽量避免创建和删除实例。创建删除实例意味着我们在修改 DOM，所以重复利用实例就是越少地修改 dom 树。

##4.3 重构
我们来重写 render 方法，保留同样健壮的调和算法，但添加一个实例化方法来根据给定的元素生成一个实例（包括其子元素）

```js
let rootInstance = null

function render(element, container) {
  const prevInstance = rootInstance
  const nextInstance = reconcile(container, prevInstance, element)
  rootInstance = nextInstance
}

function reconcile(parentDom, instance, element) {
  if (instance == null) {
    const newInstance = instantiate(element)
    parentDom.appendChild(newInstance.dom)
    return newInstance
  } else {
    const newInstance = instantiate(element)
    parentDom.replaceChild(newInstance.dom, instance.dom)
    return newInstance
  }
}

function instantiate(element) {
  const { type, props } = element

  // Create DOM element
  const isTextElement = type === 'TEXT ELEMENT'
  const dom = isTextElement ? document.createTextNode('') : document.createElement(type)

  // Add event listeners
  const isListener = (name) => name.startsWith('on')
  Object.keys(props)
    .filter(isListener)
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2)
      dom.addEventListener(eventType, props[name])
    })

  // Set properties
  const isAttribute = (name) => !isListener(name) && name != 'children'
  Object.keys(props)
    .filter(isAttribute)
    .forEach((name) => {
      dom[name] = props[name]
    })

  // Instantiate and append children
  const childElements = props.children || []
  const childInstances = childElements.map(instantiate)
  const childDoms = childInstances.map((childInstance) => childInstance.dom)
  childDoms.forEach((childDom) => dom.appendChild(childDom))

  const instance = { dom, element, childInstances }
  return instance
}
```

这段代码和之前一样，不过我们对上一次调用 render 方法保存了实例，我们也把调和方法和实例化方法分开了。

为了复用 dom 节点而不需要重新创建 dom 节点，我们需要一种更新 dom 属性（className,style,onClick 等等）的方法。所以，我们将把目前用来设置属性的那部分代码抽出来，作为一个更新属性的更通用的方法。

```js
function instantiate(element) {
  const { type, props } = element

  // Create DOM element
  const isTextElement = type === 'TEXT ELEMENT'
  const dom = isTextElement ? document.createTextNode('') : document.createElement(type)

  updateDomProperties(dom, [], props)

  // Instantiate and append children
  const childElements = props.children || []
  const childInstances = childElements.map(instantiate)
  const childDoms = childInstances.map((childInstance) => childInstance.dom)
  childDoms.forEach((childDom) => dom.appendChild(childDom))

  const instance = { dom, element, childInstances }
  return instance
}

function updateDomProperties(dom, prevProps, nextProps) {
  const isEvent = (name) => name.startsWith('on')
  const isAttribute = (name) => !isEvent(name) && name != 'children'

  // Remove event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2)
      dom.removeEventListener(eventType, prevProps[name])
    })
  // Remove attributes
  Object.keys(prevProps)
    .filter(isAttribute)
    .forEach((name) => {
      dom[name] = null
    })

  // Set attributes
  Object.keys(nextProps)
    .filter(isAttribute)
    .forEach((name) => {
      dom[name] = nextProps[name]
    })

  // Add event listeners
  Object.keys(nextProps)
    .filter(isEvent)
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2)
      dom.addEventListener(eventType, nextProps[name])
    })
}
```

updateDomProperties 方法删除所有旧属性，然后添加上新的属性。如果属性没有变，它还是照做一遍删除添加属性。所以这个方法会做很多无谓的更新，为了简单，目前我们先这样写。

##4.4 复用 dom 节点
我们说过调和算法会尽量复用 dom 节点.现在我们为调和(reconcile)方法添加一个校验，检查是否之前渲染的元素和现在渲染的元素有一样的类型(type)，如果类型一致，我们将重用它(更新旧元素的属性来匹配新元素)

```js
function reconcile(parentDom, instance, element) {
  if (instance == null) {
    // Create instance
    const newInstance = instantiate(element)
    parentDom.appendChild(newInstance.dom)
    return newInstance
  } else if (instance.element.type === element.type) {
    // Update instance
    updateDomProperties(instance.dom, instance.element.props, element.props)
    instance.element = element
    return instance
  } else {
    // Replace instance
    const newInstance = instantiate(element)
    parentDom.replaceChild(newInstance.dom, instance.dom)
    return newInstance
  }
}
```

##4.5 子元素的调和
现在调和算法少了重要的一步，忽略了子元素。[子元素调和](https://reactjs.org/docs/reconciliation.html#recursing-on-children)是 react 的关键。它需要元素上一个额外的 key 属性来匹配之前和现在渲染树上的子元素.我们将实现一个该算法的简单版。这个算法只会匹配子元素数组同一位置的子元素。它的弊端就是当两次渲染时改变了子元素的排序，我们将不能复用 dom 节点。

实现这个简单版，我们将匹配之前的子实例 instance.childInstances 和元素子元素 element.props.children，并一个个的递归调用调和方法（reconcile）。我们也保存所有 reconcile 返回的实例来更新 childInstances。

```js
function reconcile(parentDom, instance, element) {
  if (instance == null) {
    // Create instance
    const newInstance = instantiate(element)
    parentDom.appendChild(newInstance.dom)
    return newInstance
  } else if (instance.element.type === element.type) {
    // Update instance
    updateDomProperties(instance.dom, instance.element.props, element.props)
    instance.childInstances = reconcileChildren(instance, element)
    instance.element = element
    return instance
  } else {
    // Replace instance
    const newInstance = instantiate(element)
    parentDom.replaceChild(newInstance.dom, instance.dom)
    return newInstance
  }
}

function reconcileChildren(instance, element) {
  const dom = instance.dom
  const childInstances = instance.childInstances
  const nextChildElements = element.props.children || []
  const newChildInstances = []
  const count = Math.max(childInstances.length, nextChildElements.length)
  for (let i = 0; i < count; i++) {
    const childInstance = childInstances[i]
    const childElement = nextChildElements[i]
    const newChildInstance = reconcile(dom, childInstance, childElement)
    newChildInstances.push(newChildInstance)
  }
  return newChildInstances
}
```

##4.6 删除 Dom 节点
如果 nextChildElements 数组比 childInstances 数组长度长，reconcileChildren 将为所有子元素调用 reconcile 方法，并传入一个 undefined 实例。这没什么问题，因为我们的 reconcile 方法里 if (instance == null)语句已经处理了并创建新的实例。但是另一种情况呢？如果 childInstances 数组比 nextChildElements 数组长呢，因为 element 是 undefined,这将导致 element.type 报错。

这是我们并没有考虑到的，如果我们是从 dom 中删除一个元素情况。所以，我们要做两件事，在 reconcile 方法中检查 element == null 的情况并在 reconcileChildren 方法里过滤下 childInstances

```js
function reconcile(parentDom, instance, element) {
  if (instance == null) {
    // Create instance
    const newInstance = instantiate(element)
    parentDom.appendChild(newInstance.dom)
    return newInstance
  } else if (element == null) {
    // Remove instance
    parentDom.removeChild(instance.dom)
    return null
  } else if (instance.element.type === element.type) {
    // Update instance
    updateDomProperties(instance.dom, instance.element.props, element.props)
    instance.childInstances = reconcileChildren(instance, element)
    instance.element = element
    return instance
  } else {
    // Replace instance
    const newInstance = instantiate(element)
    parentDom.replaceChild(newInstance.dom, instance.dom)
    return newInstance
  }
}

function reconcileChildren(instance, element) {
  const dom = instance.dom
  const childInstances = instance.childInstances
  const nextChildElements = element.props.children || []
  const newChildInstances = []
  const count = Math.max(childInstances.length, nextChildElements.length)
  for (let i = 0; i < count; i++) {
    const childInstance = childInstances[i]
    const childElement = nextChildElements[i]
    const newChildInstance = reconcile(dom, childInstance, childElement)
    newChildInstances.push(newChildInstance)
  }
  return newChildInstances.filter((instance) => instance != null)
}
```

##4.7 总结
这一章我们增强了 Didact 使其支持更新 dom.我们也通过重用 dom 节点避免大范围 dom 树的变更，使 didact 性能更好。另外也使管理一些 dom 内部的状态更方便，比如滚动位置和焦点。

这里我更新了[codepen](https://codepen.io/pomber/pen/WjLqYW?editors=0010),在每个状态改变时调用 render 方法，你可以在 devtools 里查看我们是否重建 dom 节点。

![pic](./img/demo2.gif)

因为我们是在根节点调用 render 方法，调和算法是作用在整个树上。下面我们将介绍组件，组件将允许我们只把调和算法作用于其子树上。

#5.组件和状态(state)
##5.1 回顾
我们上一章的代码有几个问题：

- 每一次变更触发整个虚拟树的调和算法
- 状态是全局的
- 当状态变更时，我们需要显示地调用 render 方法

组件解决了这些问题，我们可以：

- 为 jsx 定义我们自己的‘标签’
- 生命周期的钩子（我们这章不讲这个）

##5.2 组件类
首先我们要提供一个供组件继承的 Component 的基类。我们还需要提供一个含 props 参数的构造方法，一个 setState 方法，setState 接收一个 partialState 参数来更新组件状态：

```js
class Component {
  constructor(props) {
    this.props = props
    this.state = this.state || {}
  }

  setState(partialState) {
    this.state = Object.assign({}, this.state, partialState)
  }
}
```

我们的应用里将和其他元素类型(div 或者 span)一样继承这个类再这样使用：\<Mycomponent/>。注意到我们的[createElement](https://gist.github.com/pomber/2bf987785b1dea8c48baff04e453b07f)方法不需要改变任何东西，createElement 会把组件类作为元素的 type，并正常的处理 props 属性。我们真正需要的是一个根据所给元素来创建组件实例(我们称之为公共实例)的方法。

```js
function createPublicInstance(element, internalInstance) {
  const { type, props } = element
  const publicInstance = new type(props)
  publicInstance.__internalInstance = internalInstance
  return publicInstance
}
```

除了创建公共实例外，我们保留了对触发组件实例化的内部实例(从虚拟 dom)引用，我们需要当公共实例状态发生变化时，能够只更新该实例的子树。

```js
class Component {
  constructor(props) {
    this.props = props
    this.state = this.state || {}
  }

  setState(partialState) {
    this.state = Object.assign({}, this.state, partialState)
    updateInstance(this.__internalInstance)
  }
}

function updateInstance(internalInstance) {
  const parentDom = internalInstance.dom.parentNode
  const element = internalInstance.element
  reconcile(parentDom, internalInstance, element)
}
```

我们也需要更新实例化方法。对组件而言，我们需要创建公共实例，然后调用组件的 render 方法来获取之后要再次传给实例化方法的子元素：

```js
function instantiate(element) {
  const { type, props } = element
  const isDomElement = typeof type === 'string'

  if (isDomElement) {
    // Instantiate DOM element
    const isTextElement = type === TEXT_ELEMENT
    const dom = isTextElement ? document.createTextNode('') : document.createElement(type)

    updateDomProperties(dom, [], props)

    const childElements = props.children || []
    const childInstances = childElements.map(instantiate)
    const childDoms = childInstances.map((childInstance) => childInstance.dom)
    childDoms.forEach((childDom) => dom.appendChild(childDom))

    const instance = { dom, element, childInstances }
    return instance
  } else {
    // Instantiate component element
    const instance = {}
    const publicInstance = createPublicInstance(element, instance)
    const childElement = publicInstance.render()
    const childInstance = instantiate(childElement)
    const dom = childInstance.dom

    Object.assign(instance, { dom, element, childInstance, publicInstance })
    return instance
  }
}
```

组件的内部实例和 dom 元素的内部实例不同，组件内部实例只能有一个子元素(从 render 函数返回)，所以组件内部只有 childInstance 属性，而 dom 元素有 childInstances 数组。另外，组件内部实例需要有对公共实例的引用，这样在调和期间，才可以调用 render 方法。

唯一缺失的是处理组件实例调和，所以我们将为调和算法添加一些处理。如果组件实例只能有一个子元素，我们就不需要处理子元素的调和，我们只需要更新公共实例的 props 属性，重新渲染子元素并调和算法它：

```js
function reconcile(parentDom, instance, element) {
  if (instance == null) {
    // Create instance
    const newInstance = instantiate(element)
    parentDom.appendChild(newInstance.dom)
    return newInstance
  } else if (element == null) {
    // Remove instance
    parentDom.removeChild(instance.dom)
    return null
  } else if (instance.element.type !== element.type) {
    // Replace instance
    const newInstance = instantiate(element)
    parentDom.replaceChild(newInstance.dom, instance.dom)
    return newInstance
  } else if (typeof element.type === 'string') {
    // Update dom instance
    updateDomProperties(instance.dom, instance.element.props, element.props)
    instance.childInstances = reconcileChildren(instance, element)
    instance.element = element
    return instance
  } else {
    //Update composite instance
    instance.publicInstance.props = element.props
    const childElement = instance.publicInstance.render()
    const oldChildInstance = instance.childInstance
    const childInstance = reconcile(parentDom, oldChildInstance, childElement)
    instance.dom = childInstance.dom
    instance.childInstance = childInstance
    instance.element = element
    return instance
  }
}
```

这就是全部代码了，我们现在支持组件，我更新了[codepen](https://codepen.io/pomber/pen/RVqBrx),我们的应用代码就像下面这样：

```js
const stories = [
  { name: 'Didact introduction', url: 'http://bit.ly/2pX7HNn' },
  { name: 'Rendering DOM elements ', url: 'http://bit.ly/2qCOejH' },
  { name: 'Element creation and JSX', url: 'http://bit.ly/2qGbw8S' },
  { name: 'Instances and reconciliation', url: 'http://bit.ly/2q4A746' },
  { name: 'Components and state', url: 'http://bit.ly/2rE16nh' }
]

class App extends Didact.Component {
  render() {
    return (
      <div>
        <h1>Didact Stories</h1>
        <ul>
          {this.props.stories.map((story) => {
            return <Story name={story.name} url={story.url} />
          })}
        </ul>
      </div>
    )
  }
}

class Story extends Didact.Component {
  constructor(props) {
    super(props)
    this.state = { likes: Math.ceil(Math.random() * 100) }
  }
  like() {
    this.setState({
      likes: this.state.likes + 1
    })
  }
  render() {
    const { name, url } = this.props
    const { likes } = this.state
    const likesElement = <span />
    return (
      <li>
        <button onClick={(e) => this.like()}>
          {likes}
          <b>❤️</b>
        </button>
        <a href={url}>{name}</a>
      </li>
    )
  }
}

Didact.render(<App stories={stories} />, document.getElementById('root'))
```

使用组件使我们可以创建自己的'JSX 标签'，封装组件状态，并且只在子树上进行调和算法

![demo3](./img/demo3.gif)
最后的[codepen](https://codepen.io/pomber/pen/RVqBrx)使用这个系列的所有代码。

# 6.Fiber:增量调和

> 我们正在写一个 react 复制品来理解 react 内部运行机制，我们称之为 didact.为了简洁代码，我们只专注于主要的功能。首先我们讲到了怎么渲染元素并如何使 jsx 生效。我们写了调和算法来只重新渲染两次更新之间的发生的变化。然后我们添加了组件类和 setState()

现在 React16 已经发布了，因为内部架构重构，所以大部分 react 代码都进行了重写。

这意味着一些我们之前不能通过旧的架构实现的功能现在可以完美实现了。

同时也意味着我们之前这个系列所写的代码全部没用了 😁。

这篇文章里，我们将使用 react 的 fiber 架构重新实现 didact.我们将模仿源码里的架构，变量，函数名。跳过一些我们不需要的公共 API：

- Didact.createElement()
- Didact.render() (只支持 dom rendering)
- Didact.createElement(有 setState(),但没有 context 和生命周期方法)

如果你想直接看代码，可以移步更新的后的[codepen 示例](https://codepen.io/pomber/pen/veVOdd)和[git 仓库](https://github.com/pomber/didact)

首先解释我们为什么要重写代码

## 6.1 为什么使用 Fiber

> 这里并不会覆盖 fiber 的方方面面，如果你想知道更多，请看[资源列表](https://github.com/koba04/react-fiber-resources)

当浏览器主线程在忙着运行一些代码的时候，重要的短时任务却不得不等待主线程空闲下来才能得到执行。

为了说明这个问题，我写了个小[demo](https://pomber.github.io/incremental-rendering-demo/react-sync.html)(在手机上看效果明显).主线程必须每 16ms 有一刻空闲才能保证行星转起来。如果主线程卡在其他事情上，比如说 200ms,你就会发现动画失帧，行星卡住直到主线程空闲。

是什么导致主线程繁忙而不能分配一点时间来确保动画顺滑和界面可相应？

记得之前实现的[调和代码](https://engineering.hexacta.com/didact-instances-reconciliation-and-virtual-dom-9316d650f1d0)?一旦开始调和就不会停止。如果主线程这个时候需要干别的事，它就得等着。而且，**因为调和过程基于太多的递归调用，很难让它可停止**。所以，我们将采用一种新的数据结构，它让我们可以使用循环代替递归调用。

> 理解 React 如何遍历 fiber Tree 却不使用递归得花上一点时间。。。。

## 6.2 调度微任务(micro-task)

我们需要把任务拆分成小块任务，运行这些小任务需要一小段时间，让主线程先做高优先级的任务，然后回来完成这些未完成的任务。

我们将使用一个方法[requestIdleCallback](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback),该方法把一个回调方法推到队列中，下次等浏览器空闲的时候就会执行。同时它还包含一个 deadline 参数，表示我们剩下多少时间来运行代码。

主要工作都在 performUnitOfWork 方法里，我们将在这方法里写新的调和方法。这个方法将执行一段任务，然后返回下次需要继续执行的任务信息。

为了追踪这些任务我们将使用 fiber

## 6.3 fiber 数据结构

我们将为每个要渲染的组件创建一个 fiber,nextUnitOfWork 指向下一个我们将处理的 fiber.performUnitOfWork 会处理当前的 fiber,并会在所有工作结束后返回一个新的 fiber.不要担心，我会在后面详细解释。

一个 fiber 数据结构是什么样的？

其实它就是一个普通的 javascript 的对象。

我们将使用 parent,child,和 sibling 属性来创建一颗 fiber 树。用它来描绘组件树。

stateNode 是指向组件实例的引用。它可能是 dom 元素或者是用户自己定义的组件类

比如：

![fiber1](./img/fiber1.png)

上面的例子可以看到，我们将支持 3 种不同的组件：

- b,p,i 的 fiber 代表**主组件(host components)**,我们将用 HOST_COMPONENT tag 来标识它们，它们 fiber 的 type 是字符串(就是 html 元素的 tag)。props 就是这些元素的属性和事件监听

- Foo fiber 表示一个**class 组件**，它的 tag 标识是 CLASS_COMPONENT。它的 type 是指向用户继承 Didact.Component 定义的类。

- div fiber 代表**主根节点(host root)**,它和主组件很像，因为它有一个 stateNode 的 DOM 元素。但作为树的根节点，它要被特殊对待。它的 tag 标识是 HOST_ROOT.需要注意的是，这个 fiber 的 stateNode 就是传给 Didact.render()的 DOM 节点

另一个很重要的属性是 alternate。我们需要这个属性是因为大多数情况下，我们将有两个 fiber 树。**一个表示我们已经渲染到 dom 的元素，我们称之为现有的树或者旧树。另一个是我们将要进行更新的树(调用 setState 或者 Didact.render()时),我们称之为进行中的树**

进行中的树和旧树之间不共享任何 fiber 节点。一旦我们创建完成进行中的树并完成了 dom 变更，进行中的树就变成了旧树。

所以我们使用 alternate 来关联进行的树和旧树对应的 fiber 节点。一个 fiber 和它的 alternate 共享一样的 tag,type 和 stateNode.有时候，当我们渲染全新的内容时，fibers 就会都没有 alternate 属性。

最后，我们需要 effects 列表和 effectTag。当我们发现进行中的树需要更新 DOM 时，我们就把 effectTag 设置成 PLACEMENT，UPDATE，DELETION。为了更方便地一次性提交所有的 dom 修改，我们保存一个包含所有 fiber 的列表(包括 fiber 的子树)，每个 fiber 有一个 effects 属性，下面列了所有 effectTag。

貌似一次性讲了太多概念了，不要担心，我们接下来就会代码实现 fiber 树

## 6.4 Didact 的调用层次

为了理清代码逻辑，我们先来看一张图：

![codeFlow](./img/codeFlow.png)

我们将从 render()和 setState()开始，先顺着由此到 commitAllWork()结束的这条路线

## 6.5 之前的代码

我说过我们将重写之前的大部分代码，那开始先来看下哪些不需要重写

在[JSX 和创建元素](#3jsx和创建元素)里我们写了创建元素 createElement()的[代码](https://gist.github.com/pomber/2bf987785b1dea8c48baff04e453b07f)，这个方法用来处理转换好的 jsx.这部分我们不需要改，我们将保留相同的元素。如果你不了解什么是元素，请看前面的章节。

在[实例，虚拟 DOM 和调和过程](#4虚拟dom和调和过程)里，我们写了 updateDomProperties()方法来更新 dom 节点的属性，另外 createDomElement()方法我们也抽出来了，你可以在 dom-uitls.js 这个[gist](https://gist.github.com/pomber/c63bd22dbfa6c4af86ba2cae0a863064)里找到这两个方法。

在[组件和状态](#5组件和状态state)里，我们写了 Component 基类，现在我们来改一下，setState()调用 scheduleUpdate(),createInstance()保存指向实例上 fiber 的一个引用

```js
class Component {
  constructor(props) {
    this.props = props || {}
    this.state = this.state || {}
  }

  setState(partialState) {
    scheduleUpdate(this, partialState)
  }
}

function createInstance(fiber) {
  const instance = new fiber.type(fiber.props)
  instance.__fiber = fiber
  return instance
}
```

仅仅以这段代码开始，我们将把剩下的部分重头写一遍

![flow1](./img/flow1.png)

除了 Component 类和 createElement()方法，我们还有 2 个公共方法:render()和 setState(),我们刚刚已经看到 setState()只是调用 scheduleUpdate()方法。

render()方法和 scheduleUpdate()类似，它们接收一个更新任务并把任务推进队列：

```js
// Fiber tags
const HOST_COMPONENT = 'host'
const CLASS_COMPONENT = 'class'
const HOST_ROOT = 'root'

// Global state
const updateQueue = []
let nextUnitOfWork = null
let pendingCommit = null

function render(elements, containerDom) {
  updateQueue.push({
    from: HOST_ROOT,
    dom: containerDom,
    newProps: { children: elements }
  })
  requestIdleCallback(performWork)
}

function scheduleUpdate(instance, partialState) {
  updateQueue.push({
    from: CLASS_COMPONENT,
    instance: instance,
    partialState: partialState
  })
  requestIdleCallback(performWork)
}
```

我们将使用 updateQueue 数组来保存待进行的变更任务，每调用 render()或 scheduleUpdate()就会推一个新的 update 对象进 updateQueue 队列，每个 update 信息都是不同的，我们将在后面的 resetNextUnitOfWork()方法里看到具体细节。

update 被推进队列之后，就触发一个对 performWork()的延时调用。

![flow2](./img/flow2.png)

```js
const ENOUGH_TIME = 1 // milliseconds

function performWork(deadline) {
  workLoop(deadline)
  if (nextUnitOfWork || updateQueue.length > 0) {
    requestIdleCallback(performWork)
  }
}

function workLoop(deadline) {
  if (!nextUnitOfWork) {
    resetNextUnitOfWork()
  }
  while (nextUnitOfWork && deadline.timeRemaining() > ENOUGH_TIME) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
  }
  if (pendingCommit) {
    commitAllWork(pendingCommit)
  }
}
```

这里就是我们之前提到的使用 performUnitOfWork()模式的地方

requestIdleCallback()调用目标方法并传入一个 deadline 参数。performWork()接收 deadline 参数并把它传给 workLoop()方法。workLoop()返回后，performWork()检查是否还有剩余的任务，如果有的话，就延时调用自己。

workLoop()是监控时间的方法。如果 deadline 太小了，就会跳出工作循环并更新 nextUnitOfWork,便于下次回来还能继续更新。

> 我们使用 ENOUGH_TIME(1ms 的常量，react 里也是这么设置的)来检查 deadline.timeRemaining()的剩余时间是否够执行一个单元的任务。如果 performUnitOfWork()花费时间比这多，这就超出了 deadline 的限制。deadline 只是浏览器的建议时间，所以超过几毫秒也不是那么严重

performUnitOfWork()将会创建对应更新的**进行中的树**，并且找出对应到 dom 的相应变更。这些都是增量做的，一次一个 fiber.

performUnitOfWork()完成当前更新的所有任务后，它会返回 null 并把 dom 待更新的内容保存在 pendingCommit 中。最后，commitAllWork()从 pendingCommit 拿到 effects 并更新 dom.

注意，commitAllWork()是在循环外调用的。performUnitOfWork()并不会去更新 dom,所以把它们分开是没问题的。从另一个角度来说，commitAllWork()会变更 dom，所以为了避免不稳定的 UI,应该一次性完成。

我们还没说 nextUnitOfWork 哪来的.

![flow3](./img/flow2.png)

接收一个 update 对象并把它转变成 nextUnitOfWork 的方法就是 resetNextUnitOfWork()

```js
function resetNextUnitOfWork() {
  const update = updateQueue.shift()
  if (!update) {
    return
  }

  // Copy the setState parameter from the update payload to the corresponding fiber
  if (update.partialState) {
    update.instance.__fiber.partialState = update.partialState
  }

  const root = update.from == HOST_ROOT ? update.dom._rootContainerFiber : getRoot(update.instance.__fiber)

  nextUnitOfWork = {
    tag: HOST_ROOT,
    stateNode: update.dom || root.stateNode,
    props: update.newProps || root.props,
    alternate: root
  }
}

function getRoot(fiber) {
  let node = fiber
  while (node.parent) {
    node = node.parent
  }
  return node
}
```

resetNextUnitOfWork()首先从队列中取出第一个 update 对象。

如果 update 上有 partialState，我们就把它保存在组件实例的 fiber 上。然后我们在调用组件的 render()方法时就可以用了。

然后我们寻找老的 fiber 树的根节点。如果 update 来自第一次调用 render()方法，就没有根 fiber。所以跟 fiber 就是 null。如果是后续的 render 调用，我们就会在 DOM 节点的\_rootContainerFiber 属性上找到根 fiber。但如果更新是来自于 setState()，我们就只能通过向上查找 fiber 实例的父母节点，知道某个节点没有父母，那它就是根节点。

然后，我们把新的 fiber 赋给 nextUnitOfWork,**这个 fiber 就是进行中的树的根节点**

如果我们没有旧的根节点，stateNode 就会作为 DOM 节点传给 render()方法.update 对象上的 newProps 就作为 props。update 的 children 属性上就有传给 render 的另一个参数 elements。alternate 是 null.

如果有旧的根节点，stateNode 就是之前根节点上的 DOM 节点，如果 newProps 非 null 的话，newProps 还是作为 props，或者我们从之前的旧根节点上拷贝 props，alternate 就是旧根节点。

我们现在有了进行中树的根节点，下面我们来创建剩余部分。

![flow4](./img/flow4.png)

```js
function performUnitOfWork(wipFiber) {
  beginWork(wipFiber)
  if (wipFiber.child) {
    return wipFiber.child
  }

  // No child, we call completeWork until we find a sibling
  let uow = wipFiber
  while (uow) {
    completeWork(uow)
    if (uow.sibling) {
      // Sibling needs to beginWork
      return uow.sibling
    }
    uow = uow.parent
  }
}
```

performUnitOfWork()遍历进行中的树。

我们调用 beginWork() -来创建 fiber 的一个子节点-然后返回第一个子节点，使它成为 nextUnitOfWork。

如果没有子节点，我们调用 completeWork()并把兄弟节点作为 nextUnitOfWork。

如果没有兄弟节点，我们不断调用 completeWork(),向上遍历父母节点，直到找到有兄弟节点的节点(这个兄弟节点会成为 nextUnitOfWork)，这个过程可能会一直到根节点。

多次调用 performUnitOfWork()会向下为每个 fiber 的第一个子 fiber 创建子节点，直到它找到一个 fiber 没有子节点。然后向右移到兄弟节点做同样的事，然后向上到叔伯节点，重复一样的事。(为了加深理解，我们可以在[fiber-debugger](https://fiber-debugger.surge.sh)上渲染几个组件看看)

![flow5](./img/flow5.png)

```js
function beginWork(wipFiber) {
  if (wipFiber.tag == CLASS_COMPONENT) {
    updateClassComponent(wipFiber)
  } else {
    updateHostComponent(wipFiber)
  }
}

function updateHostComponent(wipFiber) {
  if (!wipFiber.stateNode) {
    wipFiber.stateNode = createDomElement(wipFiber)
  }
  const newChildElements = wipFiber.props.children
  reconcileChildrenArray(wipFiber, newChildElements)
}

function updateClassComponent(wipFiber) {
  let instance = wipFiber.stateNode
  if (instance == null) {
    // Call class constructor
    instance = wipFiber.stateNode = createInstance(wipFiber)
  } else if (wipFiber.props == instance.props && !wipFiber.partialState) {
    // No need to render, clone children from last time
    cloneChildFibers(wipFiber)
    return
  }

  instance.props = wipFiber.props
  instance.state = Object.assign({}, instance.state, wipFiber.partialState)
  wipFiber.partialState = null

  const newChildElements = wipFiber.stateNode.render()
  reconcileChildrenArray(wipFiber, newChildElements)
}
```

beginWork()做两件事：

- 如果没有 stateNode,创建一个
- 获得 children 组件并把它们传给 reconcileChildrenArray()方法

因为这两个都要知道组件的类型，我们把方法拆成两个：updateHostComponent()和 updateClassComponent()

updateHostComponent()处理 host 组件以及根组件。如果需要的话它创建一个新的 DOM 节点(单一的节点，没有子节点，也不插入 Dom).**然后使用 fiber 的 props 上的子元素作为参数，调用 reconcileChildrenArray().**

updateClassComponent()处理类组件实例，如果需要，调用类组件的构造函数创建实例。**它也会更新实例的 props 和 state，然后调用 render()得到新的子节点**

updateClassComponent()也会检测是否有必要调用 render().这是个简单版的 shouldComponentUpdate().如果发现不需要 re-render,我们就把子树直接考给进行中的树，跳过调和。

现在我们有了 newChildElements,我们来为进行中的树创建子 fibers

![flow6](./img/flow6.png)

这里是库的核心，这里就是随着进行中的树增长，我们确定在提交阶段 dom 做哪些修改。

```js
// Effect tags
const PLACEMENT = 1
const DELETION = 2
const UPDATE = 3

function arrify(val) {
  return val == null ? [] : Array.isArray(val) ? val : [val]
}

function reconcileChildrenArray(wipFiber, newChildElements) {
  const elements = arrify(newChildElements)

  let index = 0
  let oldFiber = wipFiber.alternate ? wipFiber.alternate.child : null
  let newFiber = null
  while (index < elements.length || oldFiber != null) {
    const prevFiber = newFiber
    const element = index < elements.length && elements[index]
    const sameType = oldFiber && element && element.type == oldFiber.type

    if (sameType) {
      newFiber = {
        type: oldFiber.type,
        tag: oldFiber.tag,
        stateNode: oldFiber.stateNode,
        props: element.props,
        parent: wipFiber,
        alternate: oldFiber,
        partialState: oldFiber.partialState,
        effectTag: UPDATE
      }
    }

    if (element && !sameType) {
      newFiber = {
        type: element.type,
        tag: typeof element.type === 'string' ? HOST_COMPONENT : CLASS_COMPONENT,
        props: element.props,
        parent: wipFiber,
        effectTag: PLACEMENT
      }
    }

    if (oldFiber && !sameType) {
      oldFiber.effectTag = DELETION
      wipFiber.effects = wipFiber.effects || []
      wipFiber.effects.push(oldFiber)
    }

    if (oldFiber) {
      oldFiber = oldFiber.sibling
    }

    if (index == 0) {
      wipFiber.child = newFiber
    } else if (prevFiber && element) {
      prevFiber.sibling = newFiber
    }

    index++
  }
}
```

首先，我们知道 newChildElements 是个数组(不同于之前的调和算法，这里总是数组，这意味着我们可以在组件的 render()方法里返回数组)

然后，我们对比旧 fiber 树的子节点和新的元素(我们对比 fiber 和元素)。旧 fiber 树的子节点就是 wip.alternate 的子节点。新元素就是 wipFiber.props.children 或者 wipFiber.stateNode.render()返回的。

我们的调和算法通过匹配第一个旧 fiber(wipFiber.alternate.child)和第一个子元素(elements[0]),第二个旧 fiber(wipFiber.alternate.child.sibling)和第二个子元素(elements[1])，以此类推，每一个旧 fiber-元素对：

- 如果旧 fiber 和元素有一样的 type,好消息，我们可以保留之前的 stateNode.我们基于旧的创建一个新的 fiber。我们添加 UPDATE 的 effectTag。然后我们把新的 fiber 添加到进行中的树。

- 如果我们有一个元素和旧 fiber 的 type 不同，或者我们没有一个旧 fiber(因为我们新的子元素比旧的子元素多)，我们根据元素中的信息创建一个新的 fiber。需要注意的是这个新的 fiber 将没有 alternate 和 stateNode(我们将在 beginWork 里创建 stateNode)。这个 fiber 的 effectTag 就是 PLACEMENT。

- 如果旧 fiber 和元素有不同的 type 或者没有对应这个旧 fiber 的元素(因为我们旧的子元素比新的子元素多),我们给这个旧 fiber 标记 DELETION。介于这个 fiber 不是进行中的树的一部分，我们需要把它加到 wipFiber.effects,这样我们才不会失去跟踪。

> 和 React 不同的是我们没有用 keys 来做调和，这样如果子元素移动了位置我们就不知道了

![flow7](./img/flow7.png)

updateClassComponent()有一种特例情况。我们直接跳过调和，把旧的 fiber 子树拷贝到进行中的树中。

```js
function cloneChildFibers(parentFiber) {
  const oldFiber = parentFiber.alternate
  if (!oldFiber.child) {
    return
  }

  let oldChild = oldFiber.child
  let prevChild = null
  while (oldChild) {
    const newChild = {
      type: oldChild.type,
      tag: oldChild.tag,
      stateNode: oldChild.stateNode,
      props: oldChild.props,
      partialState: oldChild.partialState,
      alternate: oldChild,
      parent: parentFiber
    }
    if (prevChild) {
      prevChild.sibling = newChild
    } else {
      parentFiber.child = newChild
    }
    prevChild = newChild
    oldChild = oldChild.sibling
  }
}
```

cloneChildFibers() 会拷贝 wipFiber.alternate 上的每个子节点，然后把它们挂载到进行树上。因为我们确定知道没有任何改动，所以我们没必要添加 effectTag。

![flow8](./img/flow8.png)
