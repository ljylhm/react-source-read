在`beginWork`阶段会生成一个“全新”的`Fiber`节点，而在这个过程中，还会产生一些`effect`，这些`effect`标记了我们应该如何处理这些节点，例如`Placement`，`Update`，`Deletion`，`Passive`等；不过我们并不会对其真实的`DOM`结构进行处理，在`completeWork`过程中，我们`创建`或者`更新`每个`Fiber`对应的真实`DOM节点`，`Fiber`上的`stateNode`属性代表了真实节点，也会对产生的`effect`进行进一步的处理。<br />所以在`completeWork`过程中，我们需要完成如下工作

1. `DOM节点`的创建以及更新
2. `effect`收集构成`effectList`
3. 错误处理
<a name="PswSi"></a>
# 后序遍历
在`completeWork`的遍历过程中，都是采用`后序遍历`的顺序来处理节点，如图所示<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/29455494/1669948702738-2839b2d3-63ae-475a-97e2-1524e708d202.png#averageHue=%23fcfcfb&clientId=u5289debd-7a0a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=324&id=ue5dbdab5&margin=%5Bobject%20Object%5D&name=image.png&originHeight=648&originWidth=780&originalType=binary&ratio=1&rotation=0&showTitle=false&size=89916&status=done&style=none&taskId=u4e8b2ac1-7eab-4ff1-a582-955710fdea7&title=&width=390)<br />如图所示，我们的`completeWork`的得到的顺序如下
```tsx

	如图结构 completeWork对应的顺序如下

  grandsonJack -- childBilly -- childMary -- parent

```
当一个节点向下遍历的时候，如果它还有子节点就会依然执行子节点的`beginWork`，如果这时候发现没有子节点了，就会执行该节点的`completeWork`工作，执行完以后会依次遍历该节点的`兄弟节点`执行`beginWork`，当该层级的所有兄弟节点都被遍历完成以后，会会返回到父级元素执行`completeWork`
<a name="CtsY4"></a>
# 入口
你可以在[这里](https://github.com/facebook/react/blob/17.0.2/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L800)看到`completeWork`入口的源代码，
```tsx
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  // 根据节点不同的tag来对应不同的处理逻辑
  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      return null;
    case ClassComponent: {
      // ...省略
      return null;
    }
    case HostRoot: {
      // ...省略
      updateHostContainer(workInProgress);
      return null;
    }
    case HostComponent: {
      // ...省略
      return null;
    }
  // ...省略

```
和`beginWork`的流程类似，根据节点不同的`tag`来对应不同的处理逻辑，接下来的篇章我们以原生节点`HostComponent`进行详解
<a name="tBu0i"></a>
# HostComponet
对于节点的处理分成两个部分

- 创建节点
- 更新节点
<a name="qtiOv"></a>
## 更新节点
```tsx
  // 代表是更新阶段
  if (current !== null && workInProgress.stateNode != null) {
    // 更新的work操作
    updateHostComponent(
      current,
      workInProgress,
      type,
      newProps,
      rootContainerInstance,
    );
    if (current.ref !== workInProgress.ref) {
      markRef(workInProgress);
    }
  }
```
如果`current`不为空并且`stateNode`也不会空，代表存在原有的`DOM`节点，我们可以走`更新节点`的流程，更新节点的入口方法是`updateHostComponent`
<a name="btRdk"></a>
### updateHostComponent
```tsx
 updateHostComponent = function(
    current: Fiber,
    workInProgress: Fiber,
    type: Type,
    newProps: Props,
    rootContainerInstance: Container,
  ) {
    // 如果current不为空，代表原有的节点存在 
    // 更新原有节点
    // 1. 如果原有的props相等 代表节点无需改变
    // 2. 否则我们需要通过对比得出不同的属性并挂载到节点的updateQueue上

    const oldProps = current.memoizedProps;
    if (oldProps === newProps) {
      return;
    }

    // instance是当前节点对应的真实DOM节点
    const instance: Instance = workInProgress.stateNode;
    // 获取当前节点所在的上下文
    const currentHostContext = getHostContext();

    // 通过prepareUpdate方法来比较出不同的props
    // 并将得到的updatePayload集合挂载到当前节点的更新队列上
    // prepareUpdate -> 
    const updatePayload = prepareUpdate(
      instance,
      type,
      oldProps,
      newProps,
      rootContainerInstance,
      currentHostContext,
    );
    workInProgress.updateQueue = (updatePayload: any);
    if (updatePayload) {
      markUpdate(workInProgress);
    }
  };
```
可以看到关键的方法是`prepareUpdate`，通过该方法我们可以得到一个待更新的集合`updatePayload`，而`prepareUpdate`真正执行的方法是`diffProperties`，而在真正调用`prepareUpdate`方法前我们需要判断该节点是否能够复用，这里的复用主要是指的`DOM节点`的复用；而在得到`updatePayload`后，我们需要将其挂载到节点的`updateQueue`属性交由后续处理
<a name="B2ngV"></a>
### diffProperties
上面提到`prepareUpdate`方法实际执行的方法是`diffProperties`,
<a name="qjkyp"></a>
#### 入参和返回
我们先看`入参`和`返回值`
```tsx

	function diffProperties(
    domElement: Element,
    tag: string,
    lastRawProps: Object,
    nextRawProps: Object,
    rootContainerElement: Element | Document,
 ): null | Array<mixed> {
  	// ...省略
    let updatePayload: null | Array<any> = null;
    // ...省略
    return updatePayload;
 }

 // domElement 当前Fiber对应的元素节点
 // tag 这里的tag不是指Fiber的tag 而是指Fiber的type
 // lastRawProp 当前Fiber的props
 // nextRawProps 新传入的props
 // rootContainerElement 根节点的实例

```
`diffProperties`的入参分别为

1. `domElement` 当前`Fiber`对应的元素节点
2. `tag` 这里的`tag`不是指`Fiber`的`tag`而是指`Fiber`的`type`，例如类型为`div`的节点，`tag`就为`div`的字符串
3. `lastRawProp` 当前`Fiber`的`props`
4. `nextRawProps` 新传入的`props`
5. `rootContainerElement` 节点的实例

返回值为`updatePayload`，上面也提到过，可以将其理解为收纳`更新对比`的容器，本质是一个数组，索引为`偶数`代表`key`为`奇数`代表`value`<br />通过这个我们可以看到`diffProperties`可的主要功能

- 根据当前节点类型，对比新老`props`，得到一个`更新对比`的容器
<a name="t6zI9"></a>
#### 处理props
首先根据节点类型的不同我们需要对`props`进行处理
```tsx

  // 根据hostComponent的不同 对不同类型的props进行处理
	switch (tag) {
    case 'input':
      lastProps = ReactDOMInputGetHostProps(domElement, lastRawProps);
      nextProps = ReactDOMInputGetHostProps(domElement, nextRawProps);
      updatePayload = [];
      break;
    case 'option':
      lastProps = ReactDOMOptionGetHostProps(domElement, lastRawProps);
      nextProps = ReactDOMOptionGetHostProps(domElement, nextRawProps);
      updatePayload = [];
      break;
    case 'select':
      lastProps = ReactDOMSelectGetHostProps(domElement, lastRawProps);
      nextProps = ReactDOMSelectGetHostProps(domElement, nextRawProps);
      updatePayload = [];
      break;
    case 'textarea':
      lastProps = ReactDOMTextareaGetHostProps(domElement, lastRawProps);
      nextProps = ReactDOMTextareaGetHostProps(domElement, nextRawProps);
      updatePayload = [];
      break;
    default:
      lastProps = lastRawProps;
      nextProps = nextRawProps;
      // 对function进行一些处理
      if (
        typeof lastProps.onClick !== 'function' &&
        typeof nextProps.onClick === 'function'
      ) {
        // TODO: This cast may not be sound for SVG, MathML or custom elements.
        trapClickOnNonInteractiveElement(((domElement: any): HTMLElement));
      }
      break;
  }

```
<a name="rTTVv"></a>
#### 两次遍历
在`diffProperties`中通过两层遍历来比对得到不同属性的集合，我们这里为了方便，将当前存在的`lastProps`命名为 「 老字典 」，传进来的`nextProps`命名为「 新字典 」，可能有下面3种假设

1. 「 老字典 」存在但是「 新字典 」不存在的`propKey`，我们需要删掉
2. 「 老字典 」存在并且「 新字典 」也存在的`propKey`，我们需要更新
3. 「 老字典 」不存在但是「 新字典 」中存在的`propKey`，我们需要新增
<a name="UqN08"></a>
#### 第一次遍历
第一次遍历的目的是为了找出「 老字典 」中存在但是「 新字典 」中并不存在的`propKey`，对应`假设1`，并将其置为`null` 放入更新对比的容器`updatePayload`中
```tsx
for (propKey in lastProps) {
    // 「中止遍历的条件」
    // 1. 如果「 新字典 」中含有对应key
    // 2. propKey并不是自身的属性，而是对应原型链上的属性
    // 3. 「 老字典 」对应key的value并不存在
    if (
      nextProps.hasOwnProperty(propKey) ||
      !lastProps.hasOwnProperty(propKey) ||
      lastProps[propKey] == null
    ) {
      continue;
    }

    // 经过中止判断，我们可以知道当前
    // 1. 对应的propKey在「 新字典 」上是不存在的，这些属性需要被删除

    // 对于key === style的处理
    // 我们知道style的props为一个对象 
    // 新建一个容器 styleUpdates用来收纳style其中属性的不同
    if (propKey === STYLE) {
      const lastStyle = lastProps[propKey];
      for (styleName in lastStyle) {
        if (lastStyle.hasOwnProperty(styleName)) {
          if (!styleUpdates) {
            styleUpdates = {};
          }
          styleUpdates[styleName] = '';
        }
      }
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML || propKey === CHILDREN) {
      // Noop. This is handled by the clear text mechanism.
    } else if (
      propKey === SUPPRESS_CONTENT_EDITABLE_WARNING ||
      propKey === SUPPRESS_HYDRATION_WARNING
    ) {
      // Noop
    } else if (propKey === AUTOFOCUS) {
      // Noop. It doesn't work on updates anyway.
    } else if (registrationNameDependencies.hasOwnProperty(propKey)) {
      if (!updatePayload) {
        updatePayload = [];
      }
    } else {
      (updatePayload = updatePayload || []).push(propKey, null);
    }
  }

```
首先我们看`中断循环`的条件

1. 如果「 新字典 」中含有对应`propKey`有
2. `propKey`并不是自身的属性，而是对应原型链上的属性
3. 「 老字典 」对应`propKey`的`value`并不存在

如果通过`中断循环`的条件，我们可以知道

- 当前对应的`propKey`在「 新字典 」上是不存在的，这些属性需要被删除

在删除这里对于不同的属性也有不同的处理

1. `style`属性需要单独处理
2. `dangerouslySetInnerHTML`或者`children`等属性需要被单独处理
3. 除此以外的其他属性统一处理（将对应属性置为`null`）

这里我们对`style`属性的处理需要单独说明，我们知道`style`属性的`value`为一个对象，这里新建一个容器`styleUpdates`用来收纳`style`其中属性的不同，`styleUpdates`是一个`Object`，在当前的第一层遍历中，因为是`删除处理`，所以我们这里遍历`styleUpdates`，对其中的属性置为`空字符串`
<a name="lLz9P"></a>
#### 第二次遍历
```tsx

	for (propKey in nextProps) {
    const nextProp = nextProps[propKey];
    const lastProp = lastProps != null ? lastProps[propKey] : undefined;
    // 「中止遍历的条件」
    // 1. 「 新字典 」中propKey并不在自身属性中，而是对应原型链上的属性
    // 2. 新老prop对应的value是相等的
    // 3. 新老prop对应的value都没有有效值
    if (
      !nextProps.hasOwnProperty(propKey) ||
      nextProp === lastProp ||
      (nextProp == null && lastProp == null)
    ) {
      continue;
    }

    // 1. 进入这里说明nextProp和lastProp至少有一个是有效的
    if (propKey === STYLE) {
      // 处理propKey为style时的特殊逻辑
      // 这时候对应有两个style合集 「老style」「新style」
      if (lastProp) {
        // Unset styles on `lastProp` but not on `nextProp`.
        
        // 第一层遍历「老style」集合
        // 这一层遍历的目的是将「老style」中但「新style」中不存在的key删掉
        for (styleName in lastProp) {
          if (
            lastProp.hasOwnProperty(styleName) &&
            (!nextProp || !nextProp.hasOwnProperty(styleName))
          ) {
            if (!styleUpdates) {
              styleUpdates = {};
            }
            styleUpdates[styleName] = '';
          }
        }
        // 第二层遍历 更新和新增「新style」集合中的key
        // 更新手机style集合的容器
        // Update styles that changed since `lastProp`.
        for (styleName in nextProp) {
          if (
            nextProp.hasOwnProperty(styleName) &&
            lastProp[styleName] !== nextProp[styleName]
          ) {
            if (!styleUpdates) {
              styleUpdates = {};
            }
            styleUpdates[styleName] = nextProp[styleName];
          }
        }
      } else {
        // 走到这里说明没有对应的「老style」合集
        //「新style」合集是存在的
        // 代表这是一个新增操作 最后得到的结果是将nextProp置入更新容器中;
        if (!styleUpdates) {
          if (!updatePayload) {
            updatePayload = [];
          }
          updatePayload.push(propKey, styleUpdates);
        }
        styleUpdates = nextProp;
      }
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
      // 对React元素的属性中“dangerouslySetInnerHTML”进行处理
      const nextHtml = nextProp ? nextProp[HTML] : undefined;
      const lastHtml = lastProp ? lastProp[HTML] : undefined;
      if (nextHtml != null) {
        if (lastHtml !== nextHtml) {
          (updatePayload = updatePayload || []).push(propKey, nextHtml);
        }
      } else {
        // TODO: It might be too late to clear this if we have children
        // inserted already.
      }
    } else if (propKey === CHILDREN) {
      //
      if (typeof nextProp === 'string' || typeof nextProp === 'number') {
        (updatePayload = updatePayload || []).push(propKey, '' + nextProp);
      }
    } else if (
      propKey === SUPPRESS_CONTENT_EDITABLE_WARNING ||
      propKey === SUPPRESS_HYDRATION_WARNING
    ) {
      // Noop
    } else if (registrationNameDependencies.hasOwnProperty(propKey)) {
      if (nextProp != null) {
        ensureListeningTo(rootContainerElement, propKey, domElement);
      }
      if (!updatePayload && lastProp !== nextProp) {
        updatePayload = [];
      }
    } else if (
      typeof nextProp === 'object' &&
      nextProp !== null &&
      nextProp.$$typeof === REACT_OPAQUE_ID_TYPE
    ) {
      nextProp.toString();
    } else {
      // 针对特殊条件处理完以后
      // 普通属性的处理直接覆盖「 updatePayload 」对应的key即可
      (updatePayload = updatePayload || []).push(propKey, nextProp);
    }
  }
```
第二次遍历的目的是为了找出`假设2`和`假设3`，这次遍历的主体是「 新字典 」，每一次的遍历我们都会拿到对应的`propKey`对应的值，为了方便我们将其定义为「 新属性 」和「 老属性 」<br />首先我们依然先来看`中断循环`的条件即我们并不需要`更新`或者`新增`的属性

1. 「 新字典 」中`propKey`并不在自身属性中，而是对应原型链上的属性
2. 「 新属性 」和「 老属性 」是相等的
3. 「 新属性 」和「 老属性 」都没有有效值

如果通过`中断循环`的条件，我们可以知道

- 说明「 新属性 」和「 老属性 」至少有一个是有效的

对于不同的属性值我们也有不同的处理方法，对`style`属性的处理依然独特，我们先来理解对`style`的处理，`style`属性依然会有两次遍历，
<a name="B1qg5"></a>
##### style的第一次遍历
对于`style`来说，依然会有老值和旧值，这里我们将其命名为「 老style 」和「 新style 」，我们依然会有如此3种假设

1. 「 老style  」存在但是「 新style 」不存在的`styleKey`，我们需要删掉
2. 「 老style 」存在并且「 新style 」也存在的`styleKey`，我们需要更新
3. 「 老style 」不存在但是「 新style  」中存在的`styleKey`，我们需要新增

与`属性`的第一次遍历类似，目的依然是找出不存在的`styleKey`并将其置为`空字符串`
<a name="ZjpVp"></a>
##### style的第二次遍历
第二次遍历中我们需要完成`假设2`和`假设3`，遍历的主体是「 新style 」集合中的`key`，我们只需要满足一个条件便会执行

- `lastProp[styleName] !== nextProp[styleName]`

这里条件包含两层意义

1. 老`styleName`并不存在
2. 新`style`值和老`style`值并不相等

对应我们的`更新`和`新增`，而不管`更新`和`新增`，我们的直接覆写`key`即可
<a name="r1TAu"></a>
##### style的最后处理
```tsx

	// 针对「style」属性的特殊处理 放到了最后
  if (styleUpdates) {
    if (__DEV__) {
      validateShorthandPropertyCollisionInDev(styleUpdates, nextProps[STYLE]);
    }
    (updatePayload = updatePayload || []).push(STYLE, styleUpdates);
  }

```
对于搜集到的`style`的处理则是放到了最后，直接将`styleUpdates`推入`updatePayload`容器中
<a name="eEqEG"></a>
##### 其他属性
对于`其他属性`的处理则相对“平淡”了很多，基本满足如下条件

- `特殊属性` 对于需要特殊处理的属性，判断条件后采用覆写的方式
- `普通属性` 对于普通的属性，直接采用覆写的方式
<a name="bCr4x"></a>
#### 总结
得到最后的`updatePayload`后将其返回给上一层级的`prepareUpdate`，`prepareUpdate`又将其返回给上一层级的`updateHostComponent`，在这一层级将`updatePayload`挂载到节点的更新队列`updateQueue`上
<a name="EjboV"></a>
## 创建节点
```tsx

		// 以下皆是简化过的代码
    // 创建一个对应的实例
    // 这里我们对应一个新的DOM节点
    const instance = createInstance(
      type,
      newProps,
      rootContainerInstance,
      currentHostContext,
      workInProgress,
    );
    
    // 将该节点下所有的子节点 append到该节点下
    appendAllChildren(instance, workInProgress, false, false);

    // 将新节点赋值到stateNode属性上
    workInProgress.stateNode = instance;

    // 实例化DOM节点属性
    if (
      finalizeInitialChildren(
        instance,
        type,
        newProps,
        rootContainerInstance,
        currentHostContext,
      )
    ) {
      markUpdate(workInProgress);
    }

```
当我们知道一个节点并没有与之对应的`current`节点和`stateNode`时，我们可以判定他是需要重新被创建的，这时进入`创建流程`，整个`创建流程`做了如下几件事情

1. `createInstance` 创建一个新的`DOM`实例
2. `添加子元素` 将该节点所有的子元素加到该节点下
3. `赋值stateNode` 将创建好的实例赋值到该节点下
4. `实例化DOM节点属性`
<a name="JowR6"></a>
### createInstance
利用`createInstance`方法创建一个`DOM`实例，而该方法实际是调用了`createElement`方法来完成，我们来看`createElement`方法的实现
```tsx

	export function createElement(
  type: string,
  props: Object,
  rootContainerElement: Element | Document,
  parentNamespace: string,
): Element {
  let isCustomComponentTag;


  // props 节点上个各种属性
  // type Fiber.type 比如<div>div节点</div>就是“div” <Input />即是“Input”
  // rootContainerElement 根节点对应的元素
  // parentNamespace 父级命名空间

  // 为了简化代码 删除了一些错误提示 & 原生注释
  // 虽然React大部分情况下运行在web环境，但是也有很多场景是特殊的
  // 比如RN，小程序环境
  // 因为这些特殊的环境，所以React的宿主环境并不是统一的，是根据对应的宿主环境来定制的

  // 获取当前宿主环境对象
  const ownerDocument: Document = getOwnerDocumentFromRootContainer(
    rootContainerElement,
  );
  let domElement: Element;
  // 获取不同父级的命名空间
  let namespaceURI = parentNamespace;
  if (namespaceURI === HTML_NAMESPACE) {
    namespaceURI = getIntrinsicNamespace(type);
  }
  // 根据不同的命名空间来调用不同的方法
  if (namespaceURI === HTML_NAMESPACE) {
    // 如果类型为script的处理方式
    if (type === 'script') {
      const div = ownerDocument.createElement('div');
      div.innerHTML = '<script><' + '/script>'; // eslint-disable-line
      const firstChild = ((div.firstChild: any): HTMLScriptElement);
      domElement = div.removeChild(firstChild);
    } else if (typeof props.is === 'string') {
      domElement = ownerDocument.createElement(type, {is: props.is});
    } else {
      domElement = ownerDocument.createElement(type);
      if (type === 'select') {
        const node = ((domElement: any): HTMLSelectElement);
        if (props.multiple) {
          node.multiple = true;
        } else if (props.size) {
          node.size = props.size;
        }
      }
    }
  } else {
    domElement = ownerDocument.createElementNS(namespaceURI, type);
  }
  return domElement;
}


```
首先我们来看`createElement`方法的入参

1. `type` `Fiber.type` 比如`div`节点就是“div” `Input`节点即是“Input”
2. `props` 新的`props`参数
3. `rootContainerElement` 根节点对应的元素
4. `parentNamespace` 父级命名空间

不同`渲染器`对应不同的创建元素的方式，虽然`React`大部分情况下运行在`web`环境，但是也有很多场景是特殊的，比如RN，小程序环境；因为这些特殊的环境，所以`React`的宿主环境并不是统一的，是根据对应的宿主环境来定制的，所以我们首先需要根据当渲染器环境获得相对应的`ownerDocument`，然后根据不同的`命名空间`来创建不同的元素
<a name="QSWML"></a>
### appendAllChildren
承接上一步，当我们为该`Fiber`节点创建了一个对应的实例后，我们已经知道`completeWork`的顺序是后续遍历，这是代表该`Fiber`节点的所有子节点已经被完成了，所以我们现在要将所有子节点加到该`Fiber`节点下。
```tsx

	// 将子元素全部添加到父级元素下面
  appendAllChildren = function(
    parent: Instance,
    workInProgress: Fiber,
    needsVisibilityToggle: boolean,
    isHidden: boolean,
  ) {
    let node = workInProgress.child;

    // 深度优先遍历 后续遍历

    // 假设「 根节点 」就为「 workInProgress 」

    // 遍历的规则
    // 1. 首先定义一个正在被处理的节点「 workingNode 」
    // 2. 当遇到类型为「 HostComponent 」或者「 HostText 」节点时，直
    //    接添加利用appendChild添加到父元素下
    // 3. 如果当前节点不为「 HostComponent 」或者 「 HostText 」，
    //	  并且还有孩子节点，将遍历孩子节点
    // 4. 如果没有孩子节点，遍历兄弟节点
    // 5. 当兄弟节点遍历完成，向上找到最近未被处理完成的层级
          
    while (node !== null) {
      if (node.tag === HostComponent || node.tag === HostText) {
        appendInitialChild(parent, node.stateNode);
      } else if (enableFundamentalAPI && node.tag === FundamentalComponent) {
        appendInitialChild(parent, node.stateNode.instance);
      } else if (node.tag === HostPortal) {
     
      } else if (node.child !== null) {
        // 如果当前节点不为「 HostComponent 」或者 「 HostText 」，并且还有孩子节点，
        // 重新赋值当前节点
        node.child.return = node;
        node = node.child;
        continue;
      }
      // 终止条件，当node === workInProgres遍历已经重新回到了根节点
      if (node === workInProgress) {
        return;
      }
      // 通过此方法向上找到最近的未被递归完的层级节点
      while (node.sibling === null) {
        if (node.return === null || node.return === workInProgress) {
          return;
        }
        node = node.return;
      }
      // 遍历同一层级的所有节点 遍历兄弟节点
      node.sibling.return = node.return;
      node = node.sibling;
    }
  };
	
```
`appendAllChildren`的方法代码很少，可是理解起来却没有那么容易，其也是通过`后续遍历`的方式来遍历处理每一个节点，不过因为各个节点的类型并不相同，所以处理的方式并不完全相同<br />因为是`深度优先遍历`，所以我们这里定义一个「 根节点 」为当前需要被处理的`Fiber`节点`workInProgress`，遍历的规则如下

1. 首先定义一个正在被处理的节点「 workingNode 」
2. 当遇到类型为「 HostComponent 」或者「 HostText 」节点时，直接添加利用`appendChild`方法添加到父元素下
3. 如果当前节点不为「 HostComponent 」或者 「 HostText 」，并且还有孩子节点，将遍历孩子节点
4. 如果没有孩子节点，遍历兄弟节点
5. 当兄弟节点遍历完成，向上找到最近未被处理完成的层级，然后依次该过程
6. 如果遇到「 根节点 」时，代表已经处理完成，整个过程结束

单看文字会感觉很绕，我们来看一个场景，假设有如下代码
```typescript

	function Container() {
    return 
      <div>这里是一个容器组件</div>
      </App>
  }

  function App() {
    return <div>
      <span>文字内容</span>
      <div>这里是App组件</div>
    </div>
  }


```
对应如下图结构<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/29455494/1670207404859-6f3fc380-3862-41ac-b128-afd4a467c509.png#averageHue=%23f8f8f8&clientId=u0dcd8666-3d11-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=389&id=u110017ab&margin=%5Bobject%20Object%5D&name=image.png&originHeight=778&originWidth=984&originalType=binary&ratio=1&rotation=0&showTitle=false&size=53301&status=done&style=none&taskId=u9d75d3b8-a96c-45bd-a121-b2da8ac09b5&title=&width=492)<br />解析的过程如下

1. 首先定义「 Container 」为根节点，待处理的节点为「 workingNode 」
2. 首先来到最上层的节点「 Container 」，检测到是 「 FunctionComponent 」并且发现还有子节点时，对应「 规则3 」，继续遍历子节点
3. 这时来到第一个子节点「 div1 」，检测到是 「 HostComponent 」，对应「 规则3 」，将其直接`append`到父节点中
4. 发现该节点还有兄弟节点，这时来到兄弟节点「 App 」，检测到是 「 FunctionComponent 」并且发现还有子节点时，对应「 规则3 」，继续遍历子节点
5. 这时来到第一个子节点「 span 」，检测到是 「 HostComponent 」，对应「 规则3 」，将其直接`append`到父节点中
6. 发现该节点还有兄弟节点，这时来到兄弟节点「 div2 」，检测到是 「 HostComponent 」，对应「 规则3 」，将其直接`append`到父节点中
7. 此时发现已经没有兄弟节点了，向上找到最近未被处理完成的层级，对应「 规则5 」
8. 来到父节点「 App 」，发现没有兄弟节点，代表该层级也已经处理完成，同样对应「 规则5 」，继续向上寻找
9. 来到根节点「 Container 」，此时「 workingNode 」=== 「 Container 」，和根节点相同，对应规则「  规则6 」，遍历过程结束

在`appendAllChildren`的过程结束以后，我们将对应的实例赋值给当前的节点即`workInProgress`
```tsx

	workInProgress.stateNode = instance;

```
<a name="FqPFE"></a>
### finalizeInitialChildren
最后我们需要对节点的属性进行再一次处理，属于`ReactDom`的范畴，这里并不多做详细说明
<a name="L2nv1"></a>
#### 总结
创建一个新节点的步骤大致需要三步

1. 创建一个`DOM`对应的实例
2. 因为是`后序遍历`，子节点已经被处理完成，依次将子节点`append`到该节点下
3. 初始化节点的属性
<a name="US0wM"></a>
## effect处理
在`beginWork`和`completeWork`阶段，都会对节点打上不同的`effect`，我们知道他们将会在`commit`阶段得到处理，可是这里有一个问题，在`commit`阶段难道我们又通过遍历的方式来找到这些包含`effectTag`吗，当然不是，在`completeWork`的工作中，我们会对所有含有`effectTag`的节点进行归纳收纳，构成一条链表`effectList`<br />在每个节点中都会存在两个属性来标记自己的`effect`链表

1. `firstEffect` 代表该节点下孩子节点「 effect 」链表的第一个「 头节点 」
2. `lastEffect` 代表该节点下孩子节点「 effect 」链表的最后一个「 尾节点 」

每个节点的「 effectList 」都包含了子节点中所有的`effectTag`<br />假设我们有个正在被处理的节点「 workingNode 」，在串联起来操作中，我们需要做两件事

1. `添加EffectList到父节点` 这一步是串联起所有子孙的`effectList`
2. `添加自己到父节点的EffectList` 再将自己添加到父元素的`effectList`中
<a name="iNPfT"></a>
### 添加EffectList到父节点
```tsx

	// 1. 这里的作用是为了将父节点和该节点的孩子节点「 effect 」链表建立连接
  // 为空的逻辑 该节点的子节点变成头节点
  if (returnFiber.firstEffect === null) {
    returnFiber.firstEffect = completedWork.firstEffect;
  }
  // 如果子孙链表的最后一个effect不为空
  if (completedWork.lastEffect !== null) {
    if (returnFiber.lastEffect !== null) {
      // 拼接上去
      returnFiber.lastEffect.nextEffect = completedWork.firstEffect;
    }
    // 赋值最后一个effect为父节点的尾节点
    returnFiber.lastEffect = completedWork.lastEffect;
  }
	
```
<a name="CTdeN"></a>
### 添加自己到父节点的EffectList
```tsx

  const effectTag = completedWork.effectTag;

  // PerformedWork与React DevTool有关 大于他的都视作有更改的节点
	if (effectTag > PerformedWork) {
    if (returnFiber.lastEffect !== null) {
      // 将该节点push到父节点effectList后面
      returnFiber.lastEffect.nextEffect = completedWork;
    } else {
      // 如果没有lastEffect 说明也没有firstEffect
      // 重新赋值父节点的头指针为firstEffect
      returnFiber.firstEffect = completedWork;
    }
      // 因为拼接到最后，所以该节点成为了最后一个链表节点
    returnFiber.lastEffect = completedWork;
  }


```
<a name="BxyTg"></a>
### 总结
由此跟随`completeWork`层层向上处理，会在`rootFiber`上形成一条带有所有`effectTag`节点的`effectList`
```tsx

												nextEffect         nextEffect
	rootFiber.firstEffect -----------> fiber -----------> fiber

```
