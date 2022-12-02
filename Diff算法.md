我们已经知道`react`是一个着重在运行时的框架，而尤其重要的在更新的过程优化更新比对的效率，而其`Diff`算法将其比对的效率大大提高
<a name="G4ZQw"></a>
## 原则
`diff`的比对发生在`reconcile`过程，我们已经知道，在`react`更新过程中，会最多有两棵树同时存在，`diff`的作用可以理解成在创建`workInProgressTree`的过程中，是否可以复用以前的节点。<br />为了降低复杂度，`diff`会遵循以下几个原则

- 只做同级比较，如果一个节点更新过程中跨越了层级，将不会尝试复用
- 类型比较，如果两个节点类型不一致，即判定不可复用，会重建该节点下所有子孙节点
- 可以通过`key`关键字来显示告诉`react`哪些节点发生了变动
<a name="Wz5V2"></a>
## 比较的对象
首先我们需要有一个大概的印象，即节点的形态有以下几种

- **currentFiber**
- **workInProgressFiber**
- **reactElement**

有时候我们需要进行`currentFiber`与`currentFiber`之间的比较，例如比较`props`是否发生变化，有时候需要进行`currentFiber`与`reactElement`之间的比较，例如比较`render`新生成的元素和`currentFiber`之间是否存在复用关系，但是比较的目的最终是生成一个新的`workInProgressFiber`节点，下面我们将详细讲解中间的不同的情况
<a name="lmYvT"></a>
## 组件的比较
我们现在有这样一段代码
```tsx
  
  const Student = ({ name }) => {
    return <div>
          	<p>{ name }</p>
         </div>
  }

  <Student name="Billy">


```
假设我们有这么一个`Student`组件，这个组件会有一个`name`的`props`，在更新阶段`name`会出现两种情况；

1. `name`值发生变化
2. `name`值并没有变化

假设`1`成立，那么会进入下个阶段`比较子节点是否相同`，这里我们着重讨论`2`的情况，假设`name`的值并没有发生变化，那么又会有以下两种情况

1. 子节点并没有发生更新事件
2. 子节点这时候有更新事件

通过判断`updateLanes`在不在本次更新事件中`renderLanes`可以判断子节点有没有发生更新，如果发生更新事件，那我们并不能直接复用这个组件节点，依然要进入`比较子节点是否相同`的流程；而只有当子节点没有更新事件时，我们可以完全复用当前组件节点，进入`bailoutOnAlreadyFinishedWork`流程
<a name="mBhML"></a>
### bailoutOnAlreadyFinishedWork
这是复用节点的关键方法，很多复用的地方都会使用到它，它最主要的工作就是

- 判断是否可以复用当前节点，并返回下一个需要处理的节点
```tsx

	function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {

  if (current !== null) {
    // Reuse previous dependencies
    workInProgress.dependencies = current.dependencies;
  }

  // 性能相关，省略.....

  // 标记跳过当前节点所在的lanes
  markSkippedUpdateLanes(workInProgress.lanes);

  // Check if the children have any pending work.
  // 如果子元素没有待处理的任务 返回null
  // 这里返回null，表示workInProgressTree中的一个分支在beginWork结束
  // 这个分支进入completeWork流程
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    return null;
  } else {
    // 如果不是 克隆这一个Fiber下的所有子节点
    // 并返回当前节点的子节点 重新进入beginWork流程
    cloneChildFibers(current, workInProgress);
    return workInProgress.child;
  }
}

```
我们可以看到该方法其实很简单，它会进行如下判断

- 判断子元素是否有待处理的任务

如果子元素有处理的任务，返回`null`，这里返回`null`，表示`workInProgressTree`中的一个分支在`beginWork`阶段结束，这个分支进入`completeWork`流程<br />如果子元素还有处理的任务，克隆这一个`Fiber`节点下的所有子节点，重新进入`beginWork`来依次流程子节点
<a name="Eg4r7"></a>
## 获得新元素的过程
在前段我们知道了`Diff`比较的对象有两种

1. `currentFiber`与`currentFiber`之间的比较
2. `currentFiber`与`reactElement`之间的比较

而在`组件的比较`中我们介绍了`1`的情况，接下来介绍`2`发生的情况，发生`2`的上一个步骤是我们需要获得新元素的组成，拿`Function`组件举例，在`renderWithHooks`中可以看到有一段代码
```tsx

	let children = Component(props, secondArg);

```
这里的`Component`指`Function`组件本身，即函数，`children`则是指经过运行得到新的元素`reactElement`<br />好了，我们现在拿到了新的元素`reactElement`可以与当前存在的`currentFiber`进行比较
<a name="b5Btp"></a>
## 单节点比较
`react`将返回的节点不同类型进行了不同的处理，但是根据新元素的大类基本可以分为两种

- 单节点类型
- 多节点类型

`react`中通过判断新类型是不是`object`来判断是否是单多节点
```tsx

	const isObject = typeof newChild === 'object' && newChild !== null;

```
这节主要分析`单节点`类型，在`react`中，针对`单节点`也有不同的处理，比如普通的`REACT_ELEMENT_TYPE`，`REACT_PORTAL_TYPE`，`REACT_LAZY_TYPE`这里我们主要介绍`REACT_ELEMENT_TYPE`
```tsx

	 function reconcileSingleElement(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    element: ReactElement,
    lanes: Lanes,
  ): Fiber {
    const key = element.key;
    let child = currentFirstChild;
    // 我们现在拿到了一个新的元素，通过在外层的判断我们知道当前节点是一个单节点
    // 所以我们遍历之前currentFiber的child，
    // 从第一个子节点开始，向后遍历，搜寻是否能找到对应的节点
    while (child !== null) {
      // 如果key相同进入匹配逻辑
      if (child.key === key) {
        switch (child.tag) {
          case Fragment: {
            if (element.type === REACT_FRAGMENT_TYPE) {
              // 如果是Fragment的标签 做特殊处理
              deleteRemainingChildren(returnFiber, child.sibling);
              const existing = useFiber(child, element.props.children);
              existing.return = returnFiber;
              if (__DEV__) {
                existing._debugSource = element._source;
                existing._debugOwner = element._owner;
              }
              return existing;
            }
            break;
          }
          // ... 中间为实验性内容 暂时忽略
          default: {
            if (
              child.elementType === element.type
            ) {
              // 如果type相同 说明这个节点完全可以复用
              // 删除该节点的所有兄弟节点
              deleteRemainingChildren(returnFiber, child.sibling);
              // 复用该节点
              const existing = useFiber(child, element.props);
              // ref的逻辑单独处理
              existing.ref = coerceRef(returnFiber, child, element);
              existing.return = returnFiber;
              return existing;
            }
            break;
          }
        }
        // 如果匹配不到，将该节点加入delete集合，并打上删除标签
        deleteRemainingChildren(returnFiber, child);
        break;
      } else {
        // 如果key不相同，删除该节点
        deleteChild(returnFiber, child);
      }
      // 如果没有匹配到 继续遍历兄弟节点
      child = child.sibling;
    }

    // 这时已经遍历完成 没有找到对应的节点
    // 没有找到对应节点 节点都被删除 通过新的元素重新生成一个新的节点
    // 根据type不同利用不同的方法创建不同的节点
    if (element.type === REACT_FRAGMENT_TYPE) {
      const created = createFiberFromFragment(
        element.props.children,
        returnFiber.mode,
        lanes,
        element.key,
      );
      created.return = returnFiber;
      return created;
    } else {
      const created = createFiberFromElement(element, returnFiber.mode, lanes);
      created.ref = coerceRef(returnFiber, currentFirstChild, element);
      created.return = returnFiber;
      return created;
    }
  }

```
首先，通过判断我们已经知晓`newChild`是一个单节点元素，所以我们的核心就是遍历`currentFiber`的子节点，搜索是否有和`newChild`相等的节点并且复用，判断的规则如下

- `key`相同
- 新元素的`type`和原有节点的`elementType`相等，即类型相同

如果在遍历中，一旦满足这两个条件，复用该节点并立即返回，如果没有满足，变直接打上删除的标记，如果遍历完依然没有找到合适的节点，说明子节点中并没有任何节点可以服用，这时利用`newChild`重新生成一个新的`Fiber`节点并返回
<a name="hg1j8"></a>
## 对文本节点处理
`react`对文本节点进行了单独的处理，事实上`react`根据如下代码来判断是否是文本节点
```tsx

	typeof newChild === 'string' || typeof newChild === 'number'

```
```tsx

	function reconcileSingleTextNode(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    textContent: string,
    lanes: Lanes,
  ): Fiber {
    // There's no need to check for keys on text nodes since we don't have a
    // way to define them.
    // 译: 没有必要去检查文本节点的key，因为我们从来没有定义过他们
    if (currentFirstChild !== null && currentFirstChild.tag === HostText) {
      // We already have an existing node so let's just update it and delete
      // the rest.
      // 如果第一个子节点是文本节点 我们直接复用
      deleteRemainingChildren(returnFiber, currentFirstChild.sibling);
      const existing = useFiber(currentFirstChild, textContent);
      existing.return = returnFiber;
      return existing;
    }
    // The existing first child is not a text node so we need to create one
    // and delete the existing ones.
    // 如果不是的话 直接创建一个文本节点
    deleteRemainingChildren(returnFiber, currentFirstChild);
    const created = createFiberFromText(textContent, returnFiber.mode, lanes);
    created.return = returnFiber;
    return created;
  }

```
根据代码可以看到对于`文本节点`的判断非常简单，我们无法使用`key`来判断，因为文本节点并没有显示的`key`，它的整个判断逻辑如下

- 判断父节点的第一个子节点是否为`文本节点`

这里其实就类似一个“换壳”的操作，当匹配为`文本节点`的时候，我们只需要替换`文本节点`的内容即可，所以以判断以后有如下操作

- 如果是`文本节点`，复用并将新的内容传入
- 如果不是，删除父节点下所有的子节点并创建一个新的`文本节点`并返回
<a name="ha7aq"></a>
## 对多节点处理
在更多的情况下，我们返回的更多的是含有多个元素的集合，多节点比起单节点来说更复杂一点，首先我们先来假设我们有一个如下的节点
```tsx

  // 假设有如下的一个更新节点
	a -> b -> c -> d

  // 增加
  a -> b -> c -> d -> e

  // 删除
  a -> b -> c

  // 更新
  d -> a -> b -> c
	
```
在更新的过程中他可能会出现以下几种场景

- 节点增加
- 节点删除
- 节点更新，节点更新可能为`位置移动`，也有可能为`元素替换`，例如从`div`变成了`li`
<a name="EKN8q"></a>
### 两层循环
为了找出各种不一样的场景，`react`使用两层循环来一层一层剥开不同的场景，首先我们需要确定我们比较的对象，在[比较的对象](#Wz5V2)中知道，我们需要弄明白当前比较的两个对象

- `oldFiber`当前存在`Fiber元素`组成的链表
- `newChild`构成的元素数组

而我们循环遍历对比的对象便是如此两个对象`oldFiber`和`newChild`，这两个对象后面会经常用到
<a name="UBySC"></a>
#### 输入与输出
首先我们需要明确我们经过了比较以后需要得到的对象，即是根据`oldFiber`和`newChild`判断生成一个新的`Fiber`链表，在遍历前我们需要了解几个常用的变量
```tsx

	  // 最后返回新Fiber链表的头指针
		let resultingFirstChild: Fiber | null = null;
    // 新Fiber链表的尾节点
    let previousNewFiber: Fiber | null = null;

    // 当前对应的老节点
    let oldFiber = currentFirstChild;
    // ** 这个参数比较重要 可以理解成新生成链表过程中在原有链表中顺序最大的索引
    // ** 后面会做讲解
    let lastPlacedIndex = 0;
    // 当前遍历的索引
    let newIdx = 0;
    // 与当前对应的老节点的下一个老节点
    let nextOldFiber = null;

```
<a name="mW0OD"></a>
#### 第一层循环
第一层循环的主要目的是为了找到不同的`key`，我们可以理解同时遍历`oldFiber`和`newChild`，<br />依次遍历过程中，如果找到他们之间有不同的`key`，立马退出第一层遍历，代码如下
```tsx


	 // 第一次循环依次比对，遇到不相同的节点立马结束
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
  

      // 1.增加
      // a -> b -> c -> d
      // a -> b -> c -> d -> e

      // 2.删除
      // a -> b -> c -> d
      // a -> b -> c

      // 3.替换
      // a -> b -> c -> d
      // a -> b -> d -> c

      if (oldFiber.index > newIdx) {
        nextOldFiber = oldFiber;
        oldFiber = null;
      } else {
        nextOldFiber = oldFiber.sibling;
      }
      // updateSlot 
      // 如果类型不同 会返回null代表并不能复用
      // 如果key不同 也会返回null代表不能复用
      const newFiber = updateSlot(
        returnFiber,
        oldFiber,
        newChildren[newIdx],
        lanes,
      );
      // 如果当newFiber === null说明该节点不可复用 这个时候中断循环 第一次循环结束
      // 这个时候我们找到了链表在那个节点发生中断
      if (newFiber === null) {
        if (oldFiber === null) {
          oldFiber = nextOldFiber;
        }
        break;
      }
      
      // 性能相关 省略...
      
      // 进行到这里的前提条件是 newFiber是复用老Fiber生成的
      // 在第一阶段的目标是找到差异的节点
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
      oldFiber = nextOldFiber;
    }

```
通过代码我们可以看到我们以`newChild`数组做为基础开始遍历，利用`updateSlot`函数来返回一个新的节点`newFiber`，（你可以点击[这里](https://github.com/facebook/react/blob/12adaffef7105e2714f82651ea51936c563fe15c/packages/react-reconciler/src/ReactChildFiber.new.js#L566)看到`updateSlot`函数的实现），这里会有两种情况

- 如果`newFiber`为`null`，说明`key`并不相同，退出遍历并保留当前遍历到的节点
- 如果`newFiber`不为`null`，说明`key`相同，这个时候继续向下操作

通过`placeChild`得到返回新的`lastPlacedIndex`索引，最后和`previousNewFiber`串联起来构成新的链表
<a name="SjvkE"></a>
#### 第一层循环结果处理
我们需要对第一次循环得到的结果进行处理，有些场景已经可以得到处理了，如下所示
```tsx

	// 假设我们有一个列表
  a -> b -> c -> d -> e
  // 1. 情景一
  a -> b -> c -> d
  // 删除操作 这时候更新操作得到的新链表
  // newChild顺利遍历完，说明在此之前的节点都是可以复用的
  // 我们只需要删除oldFiber中余下的节点即可

  // 这里还有一个场景需要注意 如果链表并没有更新
  // 依然会走到这里来（因为没有余下的节点，所以并没有实质删除）
  // 假设我们有一个列表
  a -> b -> c -> d
  // 更新后的链表
  a -> b -> c -> d

  // --code--
	if (newIdx === newChildren.length) {
      // We've reached the end of the new children. We can delete the rest.
      deleteRemainingChildren(returnFiber, oldFiber);
      return resultingFirstChild;
  }
	

```
针对删除场景的判断的非常简单，我们只需要知道当前遍历的索引是否到达最后一个，如果是说明`newChild`中的每一个节点都可以被复用，这时候我们只需要删除`oldFiber`中余下的节点即可。<br />这里有一个场景我们需要注意

- 链表并没有更新

依然会走到这里来（因为没有余下的节点，所以并没有实质删除），返回的是一个相同的链表
```tsx

  	// 假设我们有一个列表
    a -> b -> c -> d -> e
    // 1. 情景二
    a -> b -> c -> d -> e -> f

    // 新增操作
    // 如果oldFiber === null 说明oldFiber链表已经被完全遍历
    // 说明老元素的长度<=新元素的长度
    // 这个时候执行的是新增的操作 将剩余的新元素执行新增操作
		if (oldFiber === null) {
      for (; newIdx < newChildren.length; newIdx++) {
        const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
        if (newFiber === null) {
          continue;
        }
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        if (previousNewFiber === null) {
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
      return resultingFirstChild;
    }


```
针对新增的逻辑也比较简单，通过判断`oldFiber`是否为`null`来判断`oldFiber`链表是否已经被完全遍历，也同时说明`oldFiber.length <= newChildren.length`，这时候遍历`newChildren`剩余的元素，通过他们创建新的元素并`push`到新链表中，并返回新的链表即可。
<a name="M0t9L"></a>
#### 第二层循环
首先在第一层循环中，我们已经处理了很多场景并且在遍历过程中找到对应不同`key`的`oldFiber`，但是依然有一种场景我们需要在第二层循环中处理

- 更新操作
```tsx

		// 依然假设我们有一个列表
    a -> b -> c -> d -> e
    // 1. 情景三 更新操作
    a -> e -> b -> c -> d

    // 这是一个更新操作
    // 1. 在第一层循环中我们找到了不同的key -> e并跳出循环
    // 2. 首先我们需要对oldFiber余下的节点进行一个map 这样提高搜索时间效率
    // --code--
    const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

    // 3. 通过upDateFromMap得到一个新的节点 upDateFromMap通过「2」中生成的map查找
    // 是否有匹配的节点，如果有复用它，如果没有重新创建一个 
    // 利用placeChild函数对该节点进行处理并更新当前的lastPlacedIndex
    // placeChild是Diff的关键 
		// lastPlacedIndex之前我们说过是newChild上搜索过的节点在老节点上最大的索引
    // 在比对中最重要的原则 
		// ** 小于这个索引 标记移动 ** 即 oldIndex < lastPlacedIndex 标记该节点为移动
    // ** 大于这个索引 保持不动 ** 并返回oldIndex为lastPlacedIndex

	  // 第一次遍历后
    // 原有
    b -> c -> d -> e
    // 现在
    e -> b -> c -> d
    // 因为a节点并没有变化，所以先将oldFiber剩余的节点生成map，然后进行第二轮遍历

    // 现在的lastPlacedIndex === 1
    // 第一次比对
    // 判断e是否在原链表上，
	  // e在原链表的索引为4，即oldIndex = 4
    // oldIndex > lastPlacedIndex 节点保持不变
    // lastPlacedIndex标记为4

    // 第二次比对
    // 判断b是否在原链表上，
	  // b在原链表的索引为1，即oldIndex = 1
    // oldIndex < lastPlacedIndex 节点标记移动
    // lastPlacedIndex保持不变为4

    // 第三次比对
    // 判断c是否在原链表上，
	  // b在原链表的索引为2，即oldIndex = 2
    // oldIndex < lastPlacedIndex 节点标记移动
    // lastPlacedIndex保持不变为4

	  // 第四次比对
    // 判断d是否在原链表上，
	  // d在原链表的索引为3，即oldIndex = 3
    // oldIndex < lastPlacedIndex 节点标记移动
    // lastPlacedIndex保持不变为4
    
```
通过以上代码我们可以知道`Diff`更新的三个重要原则

1. 小于这个索引，标记移动 即`oldIndex` < `lastPlacedIndex` 标记该节点为移动
2. 大于这个索引，保持不动 并返回`oldIndex`为`lastPlacedIndex`
3. 如果是创建的节点，并没有`oldIndex`返回当前的`lastPlacedIndex`
<a name="ntRPH"></a>
#### placeChild
在「第二层循环」中，我们介绍了`Diff`的原则，而`placeChild`函数实现了它<br />它做了如下来几件事情

1. 判断当前`newFiber`是否有对应的`alternate`
2. 如果有`alternate`代表是更新操作，如果没有代表创建操作
3. 如果是创建，给新节点打上`Placement`标签，相当于插入的操作
4. 如果是更新根据「原则一」「原则二」来判断是否是移动还是保持不变
```javascript
function placeChild(
    newFiber: Fiber,
    lastPlacedIndex: number,
    newIndex: number,
  ): number {
    // newIndex为新元素在数组中的索引
    // 新Fiber的索引赋值为这个索引
    newFiber.index = newIndex;
    // 性能相关 忽略....
    const current = newFiber.alternate;
    // 如果current为null 代表是新增的节点
    // 如果current不为null 代表是复用的节点
    if (current !== null) {
      const oldIndex = current.index;
      if (oldIndex < lastPlacedIndex) {
        // This is a move.
        // 第二次循环，判定是否移动元素
        newFiber.effectTag = Placement;
        return lastPlacedIndex;
      } else {
        // 第一次循环几乎都会走到这里来
        return oldIndex;
      }
    } else {
      newFiber.effectTag = Placement;
      return lastPlacedIndex;
    }
  }
```
