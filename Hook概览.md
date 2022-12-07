`hook`是`react16.8`后的新特性，它无需在编写`class`的条件下帮你保持组件的状态，易化了在之前`class`下逻辑难以复用的情况
<a name="Uo6Bw"></a>
## 单链表
我们可能从开发中或多或少的知道`hook`是一个单链表的结构，事实上，每个`hook`都是一个节点,每一个节点通过`next`指针相连，如图所示![Hook单链表.png](https://cdn.nlark.com/yuque/0/2022/png/29455494/1666929303971-bfff78a0-09e0-40c4-81d7-36ae515696e8.png#clientId=u3252675a-1f5d-4&crop=0&crop=0&crop=1&crop=1&from=drop&id=ub4739f68&margin=%5Bobject%20Object%5D&name=Hook%E5%8D%95%E9%93%BE%E8%A1%A8.png&originHeight=310&originWidth=1324&originalType=binary&ratio=1&rotation=0&showTitle=false&size=31243&status=done&style=none&taskId=u63ef8de6-246c-4020-b0d5-d0219942417&title=)<br />从图中我们看出`Fiber`的`memoizedState`指向`hook`的第一个节点，所以我们得出

- `memoizedState`是`hook`链表的头指针
<a name="ZYFQm"></a>
## hook的属性
在`mountWorkInProgressHook`这个入口函数中我们可以看到一个完整`hook`对象的创建，你可以在[点击](https://github.com/facebook/react/blob/17.0.2/packages/react-reconciler/src/ReactFiberHooks.new.js#L544)这里看到源码
```typescript

    const hook: Hook = {
        memoizedState: null,
        baseState: null,
        baseQueue: null,
        queue: null,
        next: null,
    };

```
`hook`在挂载的时候会有以下5个属性

- `memoizedState`代表缓存的值，可以这样理解，`memoizedState`代表上一次更新后`hook`保留的值
- `baseState`和`memoizedState`都是和`state`相关的，细微的不同则是`baseState`是一次更新的起点
- `baseQueue`队列里存的是上次更新流程中因为优先级不够未被执行的任务
- `queue`有个`pending`队列存的是待执行的任务
- `next`指向下一个`hook`
<a name="D4tZf"></a>
## renderWithHooks
`renderWithHooks`是函数组件执行的起点，返回下一个需要被解析的节点，在`renderWithHooks`函数内部会有一个判断,`current`代表当前渲染的`workInProgress`节点在`current Tree`上对应的`Fiber`节点
```typescript

		ReactCurrentDispatcher.current = 
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;

```
从代码里面可以看到，通过判断当前`current`节点是否存在来决定当前`分发器`应该被赋予的对象

- 如果`current`为空或者`current`头指针为空都代表现在的`hook`链表处于`Mount`阶段，全局的`分发器`被赋予`Mount`阶段的对象
- 如果`current`为空或者`current`头指针都不为空代表现在的`hook`链表处于`Update`阶段，全局的`分发器`被赋予`Update`阶段的对象

![ReactCurrentDispatcher.png](https://cdn.nlark.com/yuque/0/2022/png/29455494/1667205109572-67753c25-78e1-427b-9a54-9d1ae3e461ee.png#clientId=u3252675a-1f5d-4&crop=0&crop=0&crop=1&crop=1&from=drop&id=ub2ef8db5&margin=%5Bobject%20Object%5D&name=ReactCurrentDispatcher.png&originHeight=744&originWidth=2082&originalType=binary&ratio=1&rotation=0&showTitle=false&size=133550&status=done&style=none&taskId=uf0982479-3fec-4de0-a015-1ceebf91b3a&title=)

<a name="uZchM"></a>
## mountWorkInProgressHook
这个函数在每个`hook`的挂载阶段都会被执行
```typescript

	function mountWorkInProgressHook(): Hook {
      // 生成一个新的hook节点
      const hook: Hook = {
        memoizedState: null,
    
        baseState: null,
        baseQueue: null,
        queue: null,
    
        next: null,
      };

      // 这里可以把workInProgressHook理解成上一个hook节点
      if (workInProgressHook === null) {
        // 如果没有上一个hook节点 说明该Fiber节点还暂未有hook节点
        // 将该Fiber的头指针的赋值给该节点
        currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
      } else {
        // Append to the end of the list
        // 如果有挂载到该节点的后一个指针上
        workInProgressHook = workInProgressHook.next = hook;
      }
      return workInProgressHook;
  }

```
代码很简单，上面我们说到`hook`其实就是一条单链表，我们可以这个过程理解成在新的节点挂载到单链表上的过程，它做了如下几件事

- 生成一个`hook`对象
- 判断`workInProgressHook`是否存在上一个节点是否存在，这里可以把`workInProgressHook`理解成上一个`hook`节点
- 如果没有上一个`hook`节点 说明该正在渲染的`Fiber`节点还暂未有`hook`节点，将该Fiber的头指针的赋值给该节点
- 如果有，就将新的`hook`对象挂载到该节点的后一个指针上
<a name="EEJbu"></a>
## updateWorkInProgressHook
上面说到了`hook`的挂载阶段，而在更新的阶段则是执行到了`updateWorkInProgressHook`函数
```typescript

		function updateWorkInProgressHook(): Hook {

      // 定义一个变量来接收
      let nextCurrentHook: null | Hook;
      // 判断当前是否存在Hook，如果有Hook说明是该Hook
      // 如果currentHook为null 说明该更新到节点此时currentHook被重置为null
      // 1. 如果为null，就取当前渲染节点对应的current节点取出头指针作为将要被
      if (currentHook === null) {
        const current = currentlyRenderingFiber.alternate;
        if (current !== null) {
          nextCurrentHook = current.memoizedState;
        } else {
          nextCurrentHook = null;
        }
      } else {
        nextCurrentHook = currentHook.next;
      }
    
      let nextWorkInProgressHook: null | Hook;
      // 检测当前的workInProgressHook
      // 如果没有就从当前渲染Fiber节点的头指针取出
      // 如果有直接如果当前渲染指针的下一个指针节点
      if (workInProgressHook === null) {
        nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
      } else {
        nextWorkInProgressHook = workInProgressHook.next;
      }
      
      // 这是在render阶段的更新，暂时先不管
      if (nextWorkInProgressHook !== null) {
        // There's already a work-in-progress. Reuse it.
        workInProgressHook = nextWorkInProgressHook;
        nextWorkInProgressHook = workInProgressHook.next;
    
        currentHook = nextCurrentHook;
      } else {
        // Clone from the current hook.
    
        invariant(
          nextCurrentHook !== null,
          'Rendered more hooks than during the previous render.',
        );
        currentHook = nextCurrentHook;

        // 生成一个新的hook节点
        // 将current中的属性依次赋值给这个hook对象
        const newHook: Hook = {
          memoizedState: currentHook.memoizedState,
    
          baseState: currentHook.baseState,
          baseQueue: currentHook.baseQueue,
          queue: currentHook.queue,
    
          next: null,
        };
        // 如果workInProgressHook为空 说明当前update正在初始化
        // 将当前正在渲染的头指针赋值给当前节点
        if (workInProgressHook === null) {
          currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
        } else {
          // 如果不为空
          workInProgressHook = workInProgressHook.next = newHook;
        }
      }
      return workInProgressHook;
    }


```
这段代码比起`Mount`阶段看起来复杂了很多，我们首先要理解更新的这个过程到底做了什么，因为我们当前来到了一个`Update`的阶段,在前置的条件中我们可以知道当前的`workInProgress`和对应的`current`有着映射关系，即我们在更新的过程中是要复用`current`上`hook`链表的，就不难从这里看出，我们实际是要从对应的`current`节点中`clone`出一条新的链表来赋予当前的`workInProgress`的头指针<br />![updateWorkInProgressHook.png](https://cdn.nlark.com/yuque/0/2022/png/29455494/1667208184918-087d438e-fa75-4748-9804-dc32e2e1c564.png#clientId=u3252675a-1f5d-4&crop=0&crop=0&crop=1&crop=1&from=drop&id=uf350a157&margin=%5Bobject%20Object%5D&name=updateWorkInProgressHook.png&originHeight=848&originWidth=2968&originalType=binary&ratio=1&rotation=0&showTitle=false&size=189960&status=done&style=none&taskId=u1d38ada0-6f39-408e-a804-20deb195dc1&title=)<br />如果所示，当完整的函数执行完，我们会得到两个新的参数`workInProgressHook`和`currentHook`,`workInProgressHook`和`currentHook`分别在每次更新中都会结对生成，这是为了之后再对比的时候保持顺序。

这时我们也能明白那个经典的问题
> 为什么hook不能写在条件判断语句中？

因为解析`hook`的时候是按照单链表的顺序依次解析的，而如果写在条件语句中，则会打断其顺序，我们在后期进行比对的时候`workInProgressHook`和`currentHook`的顺序并不一致，导致比对结果会出错

我们将`updateWorkInProgressHook`的过程归纳如下

- 判断当前`currentHook`是否为空，如果没有说明是初始化阶段或者该`Fiber`节点中没有`hook`存在，这时取`current`节点的头指针作为`currentHook`；如果有则直接取`currentHook`的下一个指针。
- 从对应的`currentHook`处`clone` 对应的属性生成一个新的`hook`对象，判断当前`workInProgressHook`是否为空，如果没有取`workInProgress`节点的头指针作为`workInProgressHook`；如果有则直接取`workInProgressHook`的下一个指针。


