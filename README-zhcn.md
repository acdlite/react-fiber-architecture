# React Fiber 结构

## 介绍

React Fiber 是对React核心算法的重新实现，也是React团队花了两年时间研究的结晶。

React Fiber 的目标是提升其对于动画、布局、手势等场景的适用性。它的核心功能就是**增量渲染**：一种将渲染工作分解为多个区块并将其分散到每一帧里面。

其他核心功能还包括随着程序中新的update引起的暂定、终止和继续等；以及为不同任务分配优先级；和最新的并发性。

## 关于文档

Fiber引入了一些比较新的概念，所以仅通过代码很难理解。这片文档最开始是我在React项目中实现Fiber的笔记的集合。而随着它的发展，我也逐渐意识到这些笔记对其他人也可能是比较有用的资源。

我尽量通过最简单的语言，并明确定义关键术语来避免太专业化解释。在需要的时候还会有大量的外部链接来解释。

请注意我并不是React团队中的一员，所以这篇文章也并非**官方权威文档**，但是我也找了React团队的的成员对其准确性做了审查。

这也还是一个正在进行的工作。Fiber还在持续演变，**在完成之前都有可能进行很大的改动**。设计文档也会同步在这里更新，也欢迎大家来指正。

我的目标是你读完本篇文章之后对Fiber有足够的了解，以便跟上后续的演变，甚至为React作出贡献。

## 先决条件

强烈建议在读本文之前理解一下概念：

- [React 组件](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - "Component" 是非常重要的术语，需要牢牢掌握。
- [Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - 对React reconciliation算法的高级描述。
- [React Basic Theoretical Concepts](https://github.com/reactjs/react-basic) - 对没有实现负担的的React的概念描述. 刚开始的时候可能很多东西没什么意义，但是随着阅读深入会更有意义。
- [React Design Principles](https://facebook.github.io/react/contributing/design-principles.html) -需要特别关注scheduling部分。他很好的解释了为什么会有React Fiber。

## review

请在此确认理解了之前的先觉条件。

在深入文档之前，先了解几个重要的概念。

### 什么是reconciliation？

#### reconciliation

React用来对比两颗虚拟DOM树的算法，以判断哪些DOM需要更新。

#### update

用于呈现React程序中的更改，一般是setState引发的，最终重新渲染DOM。

React的核心思想是考虑全局更新，这让开发人员可以只考虑申明式来编写代码，而不需要考虑中间过程。例如（A到B，B到C，C到A等等）。

事实上，有一点改动就更新整个app只存在于很少一部分app，而在真实的场景，这是很浪费性能的。React对此做了优化，可以保证性能的同时做到完整更新，这就是**reconciliation** 的部分功能。

**reconciliation**是虚拟DOM背后的核心算法，高级一点的描述是：渲染React程序的时候，将相应的节点树保存在内存中。然后将该节点树刷新到渲染环境中--例如浏览器，它会转化成一系列的DOM操作。当程序更新的时候（通常是setState），会重新生成一颗新的树。通过两个节点树的比较来计算这些更新需要哪些操作。

尽管Fiber是对reconciliation的重写，但是React文档中的高级算法的描述大致相同，主要在于：

- 假如不同组件类型生成不同的树，react不会区分他们，二是直接替换。
- list主要用key来做区分，这要求key必须“稳定、可靠并且唯一”。

## Reconciliation 和render

DOM只是react可以渲染的环境之一，其他的还有通过react-native实现iOS和Android视图的渲染（所以说虚拟DOM用词很不恰当）。

react之所以能做到这一点是因为react在设计的时候就已经考虑到了将Reconciliation和render分开。reconciler负责计算树的更新，而render根据这些信息来负责应用程序的更新。

这种分离意味着React DOM 和React Native 可以保证使用React Core提供的相同的reconciler而且使用不同的渲染方法。

Fiber重新实现了reconciler ， 尽管渲染方法需要重新修改以支持（并利用）新的体系结构，但它基本上与渲染无关。

## Scheduling

### Scheduling

确定何时work的过程

### work

必须执行的任何计算，work一般是指更新的结果（例如setState）。

React的 [Design Principles](https://facebook.github.io/react/contributing/design-principles.html#scheduling) 这篇文档在这个主题上非常出色，这里引用一下：

> 在当前的实现中，react递归的遍历树，并在单个任务中调用整个更新后的render函数，但是以后可能会考虑延迟一些更新，以免丢帧。
>
> 这是React 模式中常见的主题，一些流行的库实现了“push”方法，在有新数据更新的时候可用。但是React坚持使用“pull”方法，这可以将计算延迟到需要的时候。
>
> React不是一个常规的数据处理库，而是用于构建用户界面的库。而我们认为它在应用程序中唯一的作用就是计算什么相关，什么不相关。
>
> 如果有些东西不在当前界面，我们可以延迟跟这个相关的逻辑。如果数据量太大，导致可能丢帧，我们会批量更新。我们可以将用户交互（比如按钮点击引起的动画）优先于次要的后台工作（例如网络请求加载的新内容），以避免丢帧。

关键点在于：

- 在用户界面中，不必立即应用每个更新。事实上，这样很浪费性能，还会导致帧下降降低用户体验。
- 不同类型的更新有不同的优先级：动画更新优先于数据存储的更新。
- 基于“push”的程序要求程序（你，就是你）决定如何安排工作，而基于“pull”的模式（react）会更加智能，并帮你决定。

当前React并没有充分利用Scheduling的优势，一次更新会导致立刻重新渲染整个子树。所以Fiber背后的思想就是彻底革新整个核心算法以充分利用Scheduling的优势。

------

现在我们开始继续深入Fiber的实现，后面的内容会更加“技术”，请确保以上的内容你已经理解。

## 什么是Fiber

下面将讨论React Fiber体系结构的核心，Fiber是比开发人员想象的低得多的抽象层。如果你发现你理解不了，别沮丧，继续坚持下去，最终肯定能理解。（等你理解的时候可以对本篇文章提点建议）

Here we go。

-------

现在已经确定，Fiber的主要目标是利用React的Scheduling的优势，具体来说需要满足以下几点：

- 暂停工作，并回来
- 为不同类型任务分配优先级
- 重用以前完成的工作
- 不需要的时候终止任务

为了做到这一点，我们需要一种将工作分解成多个单元的方法。从某种意义上来说，这就是Fiber。Fiber是一个**最小工作单元**。

为了更近一步，我们可以回顾一下 [React components as functions of data](https://github.com/reactjs/react-basic#transformation), 通常表示为：

``` javascript 
v = f(d)
```

因此，呈现整个react程序类似于调用一个函数，该函数的主体有屌用其他函数，以此类推。所以在思考Fiber的时候，这种类比会很有帮助。

计算机通常使用调用堆栈来跟踪程序执行的方式，一个函数被调用的时候，一个新的**stack frame**被添加到堆栈中，这个**stack frame**也代表了这个函数的工作也被执行了。

在处理UI的时候，较大的问题在于如果一次性执行太多任务，会导致动画掉帧并显得断断续续。而且，如果最新的更新取代了某些工作，则会显得不太必要，因为与常规的功能相比，组件的关注点会更多一点。

较新的浏览器（和React Native）实现了有助于解决此确切问题的API：equestIdleCallback安排在空闲期间调用的低优先级函数，而requestAnimationFrame安排在下一个动画帧上调用的高优先级函数。问题在于，想要使用这些API，你需要一种将render分解为增量单位的方法，否则会一直执行到堆栈为空。

如果我们可以自定义调用堆栈的行为来优化UI展现，是不是特别棒？如果我们可以随意调用堆栈，并且手动操作堆栈，是不是更棒？

这就是React Fiber的目标，Fiber是堆栈的重新实现，也可以将其视为**虚拟堆栈** 。

重新实现堆栈的优势在于，你可以将堆栈保存在内存中，并根据需要（以及任何时候）执行他们，这对于我们的目标来说至关重要。

除了任务调度之外，手动处理堆栈还可以释放并发和错误边界等功能。这些主题在以后的章节中会介绍。

下一节中，我们更多的来研究一下Fiber的结构。

### Fiber的结构

*注意：随着我们对实现细节的更加具体化，某些事项改变的可能性也增加了，如果发现任何错误或者过时的信息，请提交PR。*

具体来说，Fiber是一个JavaScript的对象，其中包含有关组件，以及输入和输出的信息。

Fiber类似于一个堆栈框架，但也对应于一个组件的实例。

这里有一些Fiber的重要概念（包括但不限于）

#### `type` 和 `key`

在Fiber中，type和key的作用相同，就像React组件一样（事实上，创建一个元素的时候，Fiber会复制这两个字段）。

Fiber的type描述了他对一个的组件，对于复合组件，type是函数或者类组件本身，对于标准组件（例如div，span），type是string。

从概念上来讲，fiber是由堆栈框架执行的函数（例如`v = f(d)`）。

与type一起，key主要用来在reconciliation期间确定Fiber是否可重用。

#### `child` and `sibling`

这些字段指向其他Fiber，描述了Fiber的递归树结构。

child fiber对应组件的render的返回值，所以在下面代码中，`Parent`的fiber指向`Child`

```javascript
function Parent() {
  return <Child />
}
```

`sibling`字段说明了返回多个子项的情况（Fiber中的新功能）。

```javascript
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

child fiber在这种情况形成了一个一维列表，开头是第一个子链。所以在示例中，`Parent`的child是`Child1`，而`Child1`的兄弟节点是`Child2`

回到之前的函数类比, 你可以把 child fiber当成 [尾部函数](https://en.wikipedia.org/wiki/Tail_call)。

#### `return`

return fiber 是当程序处理完当前fiber之后返回的fiber。从概念上讲，它与堆栈帧的返回地址相同，也可以将其视为父fiber。

如果一个fiber具有多个子fiber，那么每个子fiber返回的都是其父fiber。因此上面的例子中`Child1`和`Child2`的 return fiber都是`Parent`。

#### `pendingProps` 和 `memoizedProps`

从概念上来讲，props是函数的参数，fiber的`pendingProps`在执行开始时设置，而`memoizedProps`在结束的时候设置。

当传入的`pendingProps`和`memoizedProps`相同的时候，表示fiber可以重新使用之前的fiber，以避免重复的工作。

#### `pendingWorkPriority（待处理的工作优先级）`

Fiber的工作优先级用数字来表示，翻阅 [ReactPriorityLevel](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js)模块可以查询每个值所代表的含义。

除NoWork为0外，数字越大表示优先级越低。例如，您可以使用以下功能来检查Fiber的优先级是否至少与给定级别一样高：

```javascript
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

*这只是一个说明示例，并非Fiber代码库的一部分*

scheduler使用优先级字段来查找下一个需要执行的单元，这个算法后面会讨论。

#### `alternate`

##### flush

fiber的flush是指将其输出渲染到屏幕上。

##### *work-in-progress*

未完成的fiber，也就是为返回的堆栈帧。

在任何时候，一个组件实例最多对应两个fiber：当前的flushed fiber 和 work-in-progress fiber。

当前fiber的备用fiber是work-in-progress fiber，反过来也是一样。

fiber的备胎由`cloneFiber`延迟创建，而且并非每次都创建一个新的对象，`cloneFiber`会尝试重用fiber的备胎（如果有的话），从而最大程度的减少资源消耗。

`alternate`字段可以理解成实现细节，但是在库中经常出现，所以在这里说明一下也是很有意义的。

#### `output`

##### *host component*

React程序中的叶节点，一般是指特定的渲染环境（例如在浏览器中就是div ，span等），在jsx中都是用小写标签名来表示。

fiber的输出一般都是函数的返回值。

每个fiber都会有output，但是output只会由`host component`在叶节点中创建，并最终输出到节点树上。

output是最终提供给渲染器的输出，以便输出到具体渲染环境（译者注：react-dom 或者 react-native）中，如何定义输出和输入是渲染器的责任。

## Future sections(未来规划)

目前只说这么多，但是这片文档还远远不够完整，以后的部分会描述在整个生命周期中使用的算法，涵盖的主题包括以下：

- scheduler如何找到下一个需要执行的工作单元
- 如何通过fiber树来跟踪和传播优先级
- scheduler怎么知道什么时候暂停或者继续
- 如何刷新工作并标记为完成
- side-effects （例如生命周期）如何运作
- coroutine是什么以及如何用于实现上下文和布局等功能。

## Related Videos

- [What's Next for React (ReactNext 2016)