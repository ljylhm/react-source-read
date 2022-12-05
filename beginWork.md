我们已经知道了一个节点的解析分成了两个部分`beginWork`和`completeWork`，而`beginWork`在遍历节点的过程中采用了`先序遍历`，当本节点的`beginWork`过程结束后，会寻找自己的子节点是否存在，存在的话继续重复`beginWork`的过程，一直到节点的`child === null`时停止<br />`beginWork`最后会返回一个“新”的`Fiber`节点，这里的“新”并不意味着全新，事实上在整个更新的过程中，`react`会不停地比对该节点是否可以复用，所以可以得到返回的两个形态

- 创建过程中通过`JSX`返回的`element`创建新的`Fiber`节点
- 更新过程中通过`Diff`算法得到该节点是否可以复用，并标记其改变

在整个`reconcile`的过程中，我们并不会对`DOM`节点进行操作，一直到`commit`阶段才会进行真实的`DOM`操作，在对比的过程中如果遇到有修改或者新增的操作，我们对该节点打上对应的`effectTag`,在随后交由`completeWork`处理
<a name="DYodi"></a>
## 先序遍历
在`beginWork`的遍历过程中，都是采用先序遍历的顺序来处理节点，例如如下的节点
```tsx

	const App = () => {

    return (<div id="parent">
        <div id="child-billy">
          <div id="grandson-jack"></div>
        </div>
        <div id="child-mary"></div>
    </div>)
    
  }

```
对应的`Fiber`树结构<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/29455494/1669628742683-2aced0ef-fe16-4bd9-ad2f-156d157c94f9.png#averageHue=%23fcfcfb&clientId=ubd76052d-1cae-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=324&id=u3fac0488&margin=%5Bobject%20Object%5D&name=image.png&originHeight=648&originWidth=780&originalType=binary&ratio=1&rotation=0&showTitle=false&size=127479&status=done&style=none&taskId=u750fc396-f9e4-4c95-9b97-612c74f0ff3&title=&width=390)<br />如果所示的结构中，执行`beginWork`的顺序为
```tsx

	如图结构 beginWork对应的顺序如下

  parent -- childBilly -- grandsonJack -- childMary

```
在遍历`beginWork`的过程中，会查询该节点是否有子节点，如果有子节点会继续遍历全部子节点，一直到所有的子节点都遍历完，开始`completeWork`
<a name="Lv0yL"></a>
## effectTag
虽然在`beginWork`的过程中的，我们并不会对有改动的`Fiber`节点进行`Dom`操作，但是会对需要更新的节点打上对应的标记，在后期交由`completeWork`进一步处理，在`commit`阶段会对将所有的改动合并到真实的`Dom`树中；而在`beginWork`阶段打上的标记就是`effectTag`,你可以点击[这里](https://github.com/facebook/react/blob/17.0.2/packages/react-reconciler/src/ReactHookEffectTags.js)查看所有的`effectTag`
```tsx

	// 列举几个常用的effectTag

	1. NoEffect 一般作为effectTag的初始值，或者用于effectTag的比较判断，表示NoWork
  2. Placement 向树中插入新的节点
  3. Update 在树中更新新的节点
  4. Deletion 卸载节点时打上的标记
	5. Passive 副作用关联的标记 常用于useEffect,useLayEffect等hook
	.......
```
你可以点击[这里](https://zhuanlan.zhihu.com/p/64196683)了解更多关于`effectTag`的详解
<a name="c7a7K"></a>
## 判断能否复用
在解析节点之前并且在`更新阶段`的话，需要判断节点是否可以复用
```tsx

  // 判断是否是更新阶段
	current !== null 代表更新阶段 反之代表创建阶段

```
在更新阶段判断能否服用的代码如下
```tsx
if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    // 如果oldProps与newProps并不相等
    // 标记全局变量didReceiveUpdate为true 代表这次是会更新的
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false;
    
      // 这个fiber节点没有任何等待处理的工作，进入begin阶段之前跳出，同时需要做一些记录工作，
      // 主要是把数据推入栈
      // ....设置全局上下文的操作 忽略

      // bailout节点进行复用
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    } else {
      // 特殊逻辑 先不管...
    }
}

```
首先我们会定义一个全局变量`didReceiveUpdate`，在判断进入`更新阶段`以后，会经历如下几个阶段的判断

1. 判断新旧`props`是否相等，如果`oldProps`与`newProps`并不相等，标记全局变量`didReceiveUpdate`为`true` 代表会进入正式更新阶段
2. 如果新旧`props`相等，进入下一个阶段，判断`该节点是否存在本次更新的优先级`，如果不存在进入`bailout`阶段，在`Diff算法`中介绍了`bailoutOnAlreadyFinishedWork`方法，你可以点击[这里](https://www.yuque.com/junyuan-ruxka/foof2g/gcabgw#mBhML)查看该函数的解析；如果存在更新，进入正式的更新阶段

而根据不同的节点，我们会进入不同的处理方法
```tsx

	switch (workInProgress.tag) {
    case IndeterminateComponent: {
      // .....
    }
    case LazyComponent: {
      // .....
    }
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case ClassComponent: {
      // .....
    }
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      return updateHostText(current, workInProgress);
     // .... 省略
  }

```
根据节点的不同类型来确定是否应该采用什么的方法，因为我们使用`函数组件`的场景更多，接下来我们通过`FunctionComponent`来解析接下来的流程
<a name="H0LyM"></a>
## updateFunctionComponent
`函数组件`在最后进入的是`updateFunctionComponent`方法
```tsx

function updateFunctionComponent(
  current,
  workInProgress,
  Component,
  nextProps: any,
  renderLanes,
) {
	// 关于传参
  // current workInProgress映射的当前节点
  // workInProgress 经过处理得到正在被处理的节点
  // Component 来源于workInProgress.type 函数组件对应的函数本身为一个function
  // nextProps pendingProps 新的props
  // renderLanes 这次渲染的优先级

  let context;
  // 省略一些逻辑....

  let nextChildren;
  // 获取当前上下文
  prepareToReadContext(workInProgress, renderLanes);
  // 渲染hooK
  nextChildren = renderWithHooks(
      current,
      workInProgress,
      Component,
      nextProps,
      context,
      renderLanes,
  );

  // bailout复用逻辑
  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }

  // React DevTools reads this flag.
  workInProgress.effectTag |= PerformedWork;
  // diff算法核心
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}


```
首先来看该方法的入参

- `current` workInProgress映射的当前节点
- `workInProgress` 经过处理得到正在被处理的节点
- `Component` 来源于`workInProgress.type`函数组件对应的函数本身为一个`function`
- `nextProps` `pendingProps` 新的`props`
- `renderLanes` 本次渲染的优先级

在执行方法里面有个`renderWithHooks`里，`renderWithHooks`方法在[这里](https://www.yuque.com/junyuan-ruxka/foof2g/wr4su9#D4tZf)有简单介绍过，它主要处理的功能是执行`Component`，并处理中间遇到的种种情况
```tsx

	const Show = () => {
    return <div>展示组件</div>
  }

  // renderWithHooks源码
  let children = Component(props, secondArg);
  // 这里等同于 let children = Show(props, secondArg)


```
如图假设我们有一个`Show`组件，`Componet`就指代的`Show`这个函数，它的执行过程就是运行`Show`这个函数，在运行过程中，函数组件中的`hooks`会被一一触发，最后返回一个新的子元素`children`
<a name="bTT2H"></a>
## 关于didReceiveUpdate
我们已经知道`didReceiveUpdate`来标记是否能够复用的标记，在函数组件`Componet`运行的过程中，每一个`hook`处理过程中依然会对这个标志位发起标记
<a name="BgkYU"></a>
## 复用逻辑
```tsx

  // 代表是更新阶段 & 并不需要更新
  // didReceiveUpdate在Component()运行后可能会改变
	if (current !== null && !didReceiveUpdate) {
    // 复用hook
    bailoutHooks(current, workInProgress, renderLanes);
    // 复用节点
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }

```
上面说到，在`Componet`运行的过程中，`didReceiveUpdate`标志位会被修改，所以运行后我们判断能否服用的逻辑便是

- 当前是否是`更新阶段` & `didReceiveUpdate === false`

如果并不要复用`hook`并且复用节点（进入到这个阶段，代表这个节点时完全可以复用的，其剩余的子元素也一并复用），在`Diff算法`中对`bailoutOnAlreadyFinishedWork`做了介绍，这是一个非常常见的方法，你可以点击[这里](https://www.yuque.com/junyuan-ruxka/foof2g/gcabgw#mBhML)查看
<a name="aazBw"></a>
## 进入Diff
如果并不能复用，我们需要进入到`beginWork`最重要的阶段——通过`Diff`计算出新的`children`与原有节点的差异，并得出一个全新的节点，将该节点的子元素交由`beginWork`继续处理，你可以在[这里](https://www.yuque.com/junyuan-ruxka/foof2g/gcabgw)看到`Diff`算法的解析

