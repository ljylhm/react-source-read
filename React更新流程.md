<a name="AyZgB"></a>
## 更新流程
我们来重新回顾一下在`scheduler`中提到的更新流程
```javascript

	触发更新 ----- scheduler调度 ----- reconciler协调 ----- render渲染 

```
当我们发起一次更新时，会经历以下三个阶段

1. 任务被注册到`scheduler`进行调度
2. 调度完成，选择一个任务进入协调阶段，如果协调阶段被“打断”，重新1步骤
3. 当上面两个阶段都已经完成，进入`render`渲染阶段，此阶段不可暂停
<a name="iq2Q3"></a>
## 发起更新
在`react`中有很多种方式可以发起更新，例如

- useState
- useReducer
- this.setState
- this.forceUpdate
- ......

我们选择一种方式来看一看**发起更新**到**进入调度**前都做了什么
<a name="GQaUc"></a>
## 进入调度前
```typescript

  const initialState = {count: 0};

  function reducer(state, action) {
    switch (action.type) {
      case 'increment':
        return {count: state.count + 1};
      case 'decrement':
        return {count: state.count - 1};
      default:
        throw new Error();
    }
  }
  
  function Counter() {
    const [state, dispatch] = useReducer(reducer, initialState);
    return (
      <>
        Count: {state.count}
        <button onClick={() => dispatch({type: 'decrement'})}>-</button>
        <button onClick={() => dispatch({type: 'increment'})}>+</button>
      </>
    );
  }

```
`useReducer`是我们常用的方法，最后发起更新的常用函数，我们来看看在它的内部发生了什么<br />通过上一节我们知道在`renderWithHooks`函数中，会根据`current`是否存在为`hook`分配不同的执行函数（你可以[点击](https://www.yuque.com/junyuan-ruxka/foof2g/wr4su9#D4tZf)这里查看），而在`useReducer`中，挂载阶段分配得到的函数为`MountReducer`
<a name="aDLdJ"></a>
### MountReducer
```javascript

    function mountReducer<S, I, A>(
      reducer: (S, A) => S,
      initialArg: I,
      init?: I => S,
    ): [S, Dispatch<A>] {
      const hook = mountWorkInProgressHook();
      let initialState;
      // 初始初始值
      // 判断第三个参数是否存在，存在直接执行
      // 赋予初始值函数返回值
      if (init !== undefined) {
        initialState = init(initialArg);
      } else {
      // 如果第三个参数不存在，初始值即为第二个参数
        initialState = ((initialArg: any): S);
      }
      hook.memoizedState = hook.baseState = initialState;
      // 创建一个队列对象
      // pending 待执行更新对象队列
      // dispatch 分发函数
      // lastRenderedReducer 保留的reducer执行器
      // 上一次更新留下的值
      const queue = (hook.queue = {
        pending: null,
        dispatch: null,
        lastRenderedReducer: reducer,
        lastRenderedState: (initialState: any),
      });
      const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(
        null,
        currentlyRenderingFiber,
        queue,
      ): any));
      // 返回初始值
      return [hook.memoizedState, dispatch];
    }


```
在`MountReducer`的过程中，会创建一个`queue`的对象，它依次有如下属性

- `pending`待执行的更新队列
- `dispatch`更新时候调用的函数，最后`dispatchAction`函数接管了这个对象
- `lastRenderedState`上一次更新后留下来的值
- `lastRenderedReducer`保存初始化的`reducer`

最后返回的更新函数中返回了新的`dispatch`函数由`dispatchAction`函数包装而成，我们接下来看看在`dispatchAction`中都做了什么
<a name="LmuaN"></a>
### dispatchAction
```typescript

function dispatchAction<S, A>(
    fiber: Fiber,
    queue: UpdateQueue<S, A>,
    action: A,
) {  

    // 获取当前事件的时间 标记时间时间
    const eventTime = requestEventTime();
    // suspense相关暂时不用管
    const suspenseConfig = requestCurrentSuspenseConfig();
    // 获取当前更新对应的优先级
    const lane = requestUpdateLane(fiber, suspenseConfig);

    // 创建一个更新对象
    const update: Update<S, A> = {
        eventTime,             // 事件时间
        lane,                  // 事件优先级
        suspenseConfig,        // suspense不用管
        action,                // 执行的操作
        eagerReducer: null,    // 前置的reducer（用于更早生成值）
        eagerState: null,      // 前置的state（用户更早生成值）
        next: (null: any),     // 下一个更新的指针
    };

    // hook对象的queue对象有一个pending指针
    // 所有的更新对象构成一个循环链表
    // pending指针代表着循环链表的尾指针
    // 下面这段操作代表着将新的update对象放入循环指针的尾部
    // 并将新的update对象赋值为pending尾指针对象
    const pending = queue.pending;
    if (pending === null) {
        update.next = update;
    } else {
        update.next = pending.next;
        pending.next = update;
    }
    queue.pending = update;

    const alternate = fiber.alternate;
    // 这里做了判断 如果是在渲染阶段更新
    // 打上正在渲染阶段更新的标记并且退出流程 交由其他流程处理
    // 这里先暂时不管
    if (
        fiber === currentlyRenderingFiber ||
        (alternate !== null && alternate === currentlyRenderingFiber)
    ) {
        didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
    } else {

        // 如果当前Fiber的lanes为空 代表队列为空
        // 队列为空表示
        // 1. 当前对象为第一个更新对象
        // 2. 我们可以确定初始值为当前更新的基础值
        if (
            fiber.lanes === NoLanes &&
            (alternate === null || alternate.lanes === NoLanes)
        ) {
            
            const lastRenderedReducer = queue.lastRenderedReducer;
            if (lastRenderedReducer !== null) {
                let prevDispatcher;
                try {
                    const currentState: S = (queue.lastRenderedState: any);
                    const eagerState = lastRenderedReducer(currentState, action);
                    
                    update.eagerReducer = lastRenderedReducer;
                    update.eagerState = eagerState;
                    // 如果state并未改变返回
                    // 调度前流程结束
                    if (is(eagerState, currentState)) {
                        return;
                    }
                } catch (error) {
                   
                } finally {

                }
            }
        }
        // 进入对Fiber的调度流程
        scheduleUpdateOnFiber(fiber, lane, eventTime);
    }
}
```
`dispatchAction`函数是触发更新的函数，我们把触发函数的过程标记为`update`,触发n次就会生成n个更新对象，一个`update`对象有如下的属性

- `eventTime`标记事件开始时间
- 开始`lane`标记事件优先级，事件优先级通过`requestUpdateLane`拿到，这个函数很关键，一会儿会讲到
- `suspenseConfig`与suspense相关，暂时不用管
- `eagerReducer`预先`reducer`,可以理解为预先求解
- `eagerState`预先`state`,可以理解为预先求解

在`dispatchAction`内部，通过循环链表的方式来组织这些`update`,`hook`对象`queue`的`pending`为循环链表的尾节点。<br />总结一下`dispatchAction`这个函数做了哪些事情了

- 生成`update`对象并标记当前对象的种种属性
- 将新生成的`update`对象加入到循环列表的队尾，并将尾节点`pending`标记为新生成`update`对象
- 进行渲染时触发更新判断，如果有打上标记交由外部流程处理
- 交由调度`scheduleUpdateOnFiber`函数开启调度流程
<a name="ic1kc"></a>
### requestUpdateLane
在讲到`scheduleUpdateOnFiber`之前，我们先讲一个很重要的函数`requestUpdateLane`，在`dispatchAction`方法体中我们看到`requestUpdateLane`这个方法，这个方法非常关键，用来确定本次更新的一个优先级，决定本次更新将在何时被执行，代码量并不大，但理解起来比较麻烦。
```typescript

		export function requestUpdateLane(
      fiber: Fiber,
      suspenseConfig: SuspenseConfig | null,
    ): Lane {
      // Special cases
    	  
      // 1. 拿到当前fiber的mode，在创建root的时候会根据不同tag设置三个状态
      //    ConcurrentMode并发模式 BlockingMode过渡模式 StrictMode严格模式
      // 2. 如果没有BlockingMode，则是经典模式，直接赋予同步lane
      // 3. 如果没有ConcurrentMode，则是过渡模式，是否为立即执行
      // 4. 如果有ConcurrentMode，代表并发模式
    
      const mode = fiber.mode;
      if ((mode & BlockingMode) === NoMode) {
        return (SyncLane: Lane);
      } else if ((mode & ConcurrentMode) === NoMode) {
        return getCurrentPriorityLevel() === ImmediateSchedulerPriority
          ? (SyncLane: Lane)
          : (SyncBatchedLane: Lane);
      } else if (
        !deferRenderPhaseUpdateToNextBatch &&
        (executionContext & RenderContext) !== NoContext &&
        workInProgressRootRenderLanes !== NoLanes
      ) 
        // workInProgressRootRenderLanes代表当前更新的
        // 如果是渲染阶段的话更新，选取当前渲染lanes的最高优先级
        // 合并进行一次批处理
        
        // 如果是渲染阶段的话取workInProgressRoot中最高的
        return pickArbitraryLane(workInProgressRootRenderLanes);
      }
    
      // The algorithm for assigning an update to a lane should be stable for all
      // currentEventWipLanes 是指代的当前渲染树上所有包含的lanes
      if (currentEventWipLanes === NoLanes) {
        currentEventWipLanes = workInProgressRootIncludedLanes;
      }
      
      // 获取当前执行事件的优先级
      const schedulerPriority = getCurrentPriorityLevel();
    
      let lane;
      if (
        (executionContext & DiscreteEventContext) !== NoContext &&
        schedulerPriority === UserBlockingSchedulerPriority
      ) {
        // 一些边缘化处理
        lane = findUpdateLane(InputDiscreteLanePriority, currentEventWipLanes);
      } else {
        // 将现在的优先级转化成lanePriority
        // 通过findUpdateLane函数获得当前最高的优先级
        const schedulerLanePriority = schedulerPriorityToLanePriority(
          schedulerPriority,
        );
        lane = findUpdateLane(schedulerLanePriority, currentEventWipLanes);
      }
      return lane;
    }


```
虽然删除了一些代码，但是`requestUpdateLane`的代码依旧不多，主要是为了获得当前更新的优先级，而在获取当前更新优先级的过程中与已存在的更新进行对比，给予更新一个合理的`lane`，这里需要注意几个问题

1. `currentEventWipLanes`是干嘛的，从代码里面看到`currentEventWipLanes`是被赋予了`workInProgressRootIncludedLanes`，而`workInProgressRootIncludedLanes`则是代表了当前渲染树上正在等待的赛道集合，所以`currentEventWipLanes`可以理解成待执行的更新任务，而这些更新任务合成了一条新的赛道作为基准赛道，更新的优先级都需要基于当前这条已有的赛道
2. 对于`findUpdateLane`函数的理解，上面我们说过更新的优先级需要基于`currentEventWipLanes`当前已有的赛道，通过两个参数`lanePriority`和`currentEventWipLanes`找到当前更新优先级是否在已有的赛道中已存在，如果已存在，被依次降级处理；举个简单的例子，就好像球场看比赛，你被分派了一张第一排的位置，但是过去发现第一排的位置已经全被人坐了，于是你降级处理去了第二排，发现第二排有位置，于是你占据了第二排的位置，如果有人继续往后依次处理，你可以[点击](https://github.com/facebook/react/blob/17.0.2/packages/react-reconciler/src/ReactFiberLane.js#L484)这里查看`findUpdateLane`实现

我们来总结以下`requestUpdateLane`这个函数做了什么

- 顶层设计，根据当前`mode`来判断更新应该采用什么赛道，在创建`root`的时候就会确定
- 如果是在渲染阶段更新，返回当前渲染赛道中的优先级最高赛道，这次更新将会和下次更新一起进行批处理
- 利用`currentEventWipLanes`和`findUpdateLane`找到当前更新在更新赛道的位置
<a name="D5Etv"></a>
### scheduleUpdateOnFiber
你可以在[这里](https://github.com/facebook/react/blob/17.0.2/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L528)看到`scheduleUpdateOnFiber`的代码，在`scheduleUpdateOnFiber`内有两个很重要的函数`markUpdateLaneFromFiberToRoot`和`markRootUpdated`我们着重分析这两个函数
<a name="sddvl"></a>
### markUpdateLaneFromFiberToRoot
```typescript
    
    
    function markUpdateLaneFromFiberToRoot(
      sourceFiber: Fiber,
      lane: Lane,
    ): FiberRoot | null {
      // 从当前Fiber开始依次向父级Fiber递归，一直到root节点结束
      // 递归到新节点的过程中更新当前Fiber对应的lanes和childLanes
      // 每个Fiber对应的alternate也同步更新
      sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
      let alternate = sourceFiber.alternate;
      if (alternate !== null) {
        alternate.lanes = mergeLanes(alternate.lanes, lane);
      }
      let node = sourceFiber;
      let parent = sourceFiber.return;
      while (parent !== null) {
        parent.childLanes = mergeLanes(parent.childLanes, lane);
        alternate = parent.alternate;
        if (alternate !== null) {
          alternate.childLanes = mergeLanes(alternate.childLanes, lane);
        } else {
        }
        node = parent;
        parent = parent.return;
      }
      if (node.tag === HostRoot) {
        const root: FiberRoot = node.stateNode;
        return root;
      } else {
        return null;
      }
    }
    

```
通过分析`requestUpdateLane`我们已经得到了当前更新所对应的`lane`,我们首先需要将该更新的`lane`在沿途到根节点的所有Fiber中依次添加进去，所以在这个函数中我们做了以下几件事情

- 从当前`Fiber`开始依次向父级`Fiber`递归，一直到`root`节点结束
- 递归到新节点的过程中更新当前`Fiber`对应的`lanes`和`childLanes`
- 每个`Fiber`对应的`alternate`也同步更新
<a name="l9Rjd"></a>
### markRootUpdated
```javascript
export function markRootUpdated(
  root: FiberRoot,
  updateLane: Lane,
  eventTime: number,
) {
  root.pendingLanes |= updateLane;

  // ...suspended相关忽略
  
  const eventTimes = root.eventTimes;
  const index = laneToIndex(updateLane);
  eventTimes[index] = eventTime;
}
```
这段代码也非常简单，通过`markUpdateLaneFromFiberToRoot`得到的根节点，在这个根节点上的`pendingLanes`添加本次更新的赛道,这里需要注意的是过期时间的处理，我们知道`lane`是一个31位的赛道，而`eventTimes`是一个数组，可以看成一个长度为31的数组，正好映射了赛道中对应的每一个`lane`的事件时间，这里得到当前`upDateLane`在31位中的顺序也非常的巧妙，通过`31 - clz32(lanes)`方法来获取对应的顺序，`clz32`是`Math`提供的原生API，它可以告诉你二进制中最左边的1还余有多少个0，我们通过31减去剩余的0便可以知道当前`upDateLane`在31位中的顺序，并在`eventTimes`进行标记
<a name="LRpVx"></a>
### 任务饥饿问题
通过上面许多的步骤我们终于来到了调度前的最后一步，简单回顾一下，我们上面最重要的事情是创建了一个`upDate`对象，围绕这个更新对象，我们获取当前更新的优先级`lane`,并将该`lane`沿途一直标记到根节点，并标记根节点对应的更新，调度前的准备似乎已经完成，我们将在这里开始一个新的调度，但是在此之前我们还需要了解另外一个问题——`任务饥饿问题`<br />假设一个场景，如果不断有高优先级任务插队执行，那么低优先级任务便一直得不到执行，那这种情况就叫做`任务饥饿问题`，所以在调度之前，我们还需要在当前待执行的任务中找到最紧迫的任务，从而调度这个优先级更高的任务
```javascript

		
		export function markStarvedLanesAsExpired(
      root: FiberRoot,
      currentTime: number,
    ): void {

      // 这里我们只讨论pendingLanes
      const pendingLanes = root.pendingLanes;
      const suspendedLanes = root.suspendedLanes;
      const pingedLanes = root.pingedLanes;
      // 根节点存储的过期时间
      const expirationTimes = root.expirationTimes;
    
      let lanes = pendingLanes;
      while (lanes > 0) {
        // 获取最低优先级的索引
        const index = pickArbitraryLaneIndex(lanes);
        // 将二进制表示为1的二进制数移动索引位 用来表示当来lane
        const lane = 1 << index;
    
        // 拿到对应的过期时间数组中的过期时间
        const expirationTime = expirationTimes[index];
        // 找到没有过期时间的挂起通道。如果没有暂停，或如果它是ping的，
        // 假设它是CPU绑定的。计算新的过期时间使用当前时间。
        if (expirationTime === NoTimestamp) {
          if (
            (lane & suspendedLanes) === NoLanes ||
            (lane & pingedLanes) !== NoLanes
          ) {
            expirationTimes[index] = computeExpirationTime(lane, currentTime);
          }
        } else if (expirationTime <= currentTime) {
          // This lane expired
          // 当前lane已经过期，将其加入过期lanes中
          root.expiredLanes |= lane;
        }
        // 删除对应的lane
        lanes &= ~lane;
      }
    }


```
我们在这里只分析`pendingLanes`，我们首先从根节点拿到`pendingLanes`以及过期时间对应的数组`expirationTimes`，我们来分析一下步骤

1. 通过`pickArbitraryLaneIndex`拿到当前最左边为1的`lane`,也就是优先级最低的`lane`,`pickArbitraryLaneIndex`函数其实就是通过上文我们说的`31 - clz32(lanes)`方法来实现的，拿到对应的索引
2. 给定一个“1”，在Javascript中，整数都是以32位二进制来表示，就好像这样`0000 0000 0000 0000 0000 0000 0000 0001`所以直接左移`index`位，得到对应`lane`的表示
3. 判断当前映射的`expirationTimes`数组中对应索引的过期时间是否过期，利用`&`+`取反`来删除这个`lane`,接着继续循环，并将过期的`lane`加入到根节点的`expiredLanes`中
<a name="pblYh"></a>
### getNextLanes
在完成饥饿问题后，我们开始调读前的最后一步，这时我们还是需要从`pendingLanes`以及`expiredLanes`中本次更新的终极人选，而这个任务则是通过`getNextLanes`函数来完成的，我们可以这样理解

- `getNextLanes`在通过以上一系列操作得到的素材中选出一道优先级最高的任务

因为`getNextLanes`代码很长，我们通过切割代码的方式来一段一段的查看
```javascript

 // 如果没有剩余任务，直接退出了
 const pendingLanes = root.pendingLanes;
 if (pendingLanes === NoLanes) {
    return_highestLanePriority = NoLanePriority;
    return NoLanes;
 }


```
判断`pendingLanes`是否存在，如果不存在说明没有剩余任务了，直接退出
```javascript

  // 判断过期任务，如果有的话，以过期任务为准
	if (expiredLanes !== NoLanes) {
    nextLanes = expiredLanes;
    // 已经过期了，就需要把渲染优先级设置为同步，来让更新立即执行
    nextLanePriority = return_highestLanePriority = SyncLanePriority;
  }

```
我们通过上面饥饿问题得知，如果有过期的任务，会被推入`expiredLanes`,这里判断如果存在，以过期任务为准
```javascript

		const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
	  if (nonIdlePendingLanes !== NoLanes) {
      // 未被阻塞的lanes，它等于有优先级的lanes中除去被挂起的lanes
      // & ~ 相当于删除
      const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;

      // 如果有任务被阻塞了
      if (nonIdleUnblockedLanes !== NoLanes) {

        // 那么从这些被阻塞的任务中挑出最重要的
        nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
        nextLanePriority = return_highestLanePriority;
      } else {

        // 如果没有任务被阻塞，从正在处理的lanes中找到优先级最高的
        const nonIdlePingedLanes = nonIdlePendingLanes & pingedLanes;
        if (nonIdlePingedLanes !== NoLanes) {
          nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
          nextLanePriority = return_highestLanePriority;
        }
      }
    } else {

      // The only remaining work is Idle.
      // 剩下的任务是闲置的优先级不高的任务。unblockedLanes是未被阻塞的闲置任务
      const unblockedLanes = pendingLanes & ~suspendedLanes;
      if (unblockedLanes !== NoLanes) {

        // 从这些未被阻塞的闲置任务中挑出最重要的
        nextLanes = getHighestPriorityLanes(unblockedLanes);
        nextLanePriority = return_highestLanePriority;
      } else {
        if (pingedLanes !== NoLanes) {
          nextLanes = getHighestPriorityLanes(pingedLanes);
          nextLanePriority = return_highestLanePriority;
        }
      }
    }

```
这里的逻辑简单说来就是去找到`pendingLanes`中未闲置的任务，如果有未闲置的任务优先找出他们，`NonIdleLanes`指代的是包含所有非闲置任务的赛道集合，事实上它长这个样子`0b0000111111111111111111111111111`,暂时不用考虑`suspendedLanes`和`pingedLanes`,总结一下

- 如果有非闲置的任务从里面找出优先级最高的任务
- 如果都是闲置的任务从里面找出优先级最高的任务
```javascript

    if (
        wipLanes !== NoLanes &&
        wipLanes !== nextLanes &&
        // If we already suspended with a delay, then interrupting is fine. Don't
        // bother waiting until the root is complete.
        (wipLanes & suspendedLanes) === NoLanes
      ) {
        getHighestPriorityLanes(wipLanes);
        const wipLanePriority = return_highestLanePriority;
        if (nextLanePriority <= wipLanePriority) {
          return wipLanes;
        } else {
          return_highestLanePriority = nextLanePriority;
        }
    }


```
这里的`wipLanes`其实就是`workInProgressRootRenderLanes`，不过这里有个前提

- 当前`root`与`workInProgressRoot`是同一个根节点

将`wipLanes`和上面获得各自的最高优先级`lane`进行对比,获取最高的优先级并返回
<a name="AUwpp"></a>
### ensureRootIsScheduled
在上面我们讲到了`任务饥饿问题`和`getNextLanes`,现在我们正式进入`ensureRootIsScheduled`函数中查看，经过上面两个问题，`ensureRootIsScheduled`剩余的代码已经变得非常简单了
```javascript
    

	  // 通过上面个getNextLanes得到执行方法的优先级
    const newCallbackPriority = returnNextLanesPriority();
      // 新任务没有任何渲染优先级，退出
      if (nextLanes === NoLanes) {
        // Special case: There's nothing to work on.
        // 如果还有任务取消掉
        if (existingCallbackNode !== null) {
          cancelCallback(existingCallbackNode);
          root.callbackNode = null;
          root.callbackPriority = NoLanePriority;
        }
        return;
      }
    
      // Check if there's an existing task. We may be able to reuse it.
      // 检查之前是否有执行任务
      // 将当前任务的优先级和根节点已存在的优先级进行比较
      if (existingCallbackNode !== null) {
        const existingCallbackPriority = root.callbackPriority;
        // 如果相同的话 说明是同一优先级直接返回
        if (existingCallbackPriority === newCallbackPriority) {
          // The priority hasn't changed. We can reuse the existing task. Exit.
          return;
        }
        // The priority changed. Cancel the existing callback. We'll schedule a new
        // one below.
        // 如果不是 取消当前的任务
        cancelCallback(existingCallbackNode);
      }
    
      // Schedule a new callback.
      // 判断当前任务属于哪一个优先级
      // 不同的优先级利用scheduler注册不同的任务
      let newCallbackNode;
      if (newCallbackPriority === SyncLanePriority) {
        // Special case: Sync React callbacks are scheduled on a special
        // internal queue
        newCallbackNode = scheduleSyncCallback(
          performSyncWorkOnRoot.bind(null, root),
        );
      } else if (newCallbackPriority === SyncBatchedLanePriority) {
        newCallbackNode = scheduleCallback(
          ImmediateSchedulerPriority,
          performSyncWorkOnRoot.bind(null, root),
        );
      } else {
        const schedulerPriorityLevel = lanePriorityToSchedulerPriority(
          newCallbackPriority,
        );
        newCallbackNode = scheduleCallback(
          schedulerPriorityLevel,
          performConcurrentWorkOnRoot.bind(null, root),
        );
      }

	    // 重置根节点的任务和方法
      root.callbackPriority = newCallbackPriority;
      root.callbackNode = newCallbackNode;
```
在剩余的代码中我们看到通过`getNextLanes`我们拿到了当前需要更新的优先级，进行了如下判断

- 如果`root`节点的`callbackPriority`和当前的相同，说明是同一级的更新，进行批处理即可
- 如果不相同，取消正在执行的`callbackNode`，当前任务被抢占
- 根据不同`newCallbackPriority`通过调度器注册不同的方法，这里主要是区分`performSyncWorkOnRoot`和`performConcurrentWorkOnRoot`
<a name="Oapki"></a>
## 调度以后
在此代表执行到了对应`hook`的更新阶段<br />在`hooks`概览里面（你可以[点击](https://www.yuque.com/junyuan-ruxka/foof2g/wr4su9#EEJbu)这里查看）我们介绍了在更新阶段中通过`updateWorkInProgressHook`我们会获得当前正在渲染的`Fiber`的`hook`即`workInProgressHook`和当前树`current`对应的`hook`<br />我们看看更新的代码是什么样的
<a name="vs0o9"></a>
### updateReducer
```javascript
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  // reducer可能被随时改变 因为reducer本职其实就是一个方法
  // 1. 更新当前最新的reducer
  queue.lastRenderedReducer = reducer;

  // current Tree对应的Fiber对象
  const current: Hook = (currentHook: any);

  // The last rebase update that is NOT part of the base state.
  // 从current Hook中拿到上次更新尚未被执行的更新
  let baseQueue = current.baseQueue;


  // The last pending update that hasn't been processed yet.
  // 这次更新待执行的更新对象
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
 
    // 将这次更新链表合并上次未执行的链表 合并成一个新的链表
    if (baseQueue !== null) {
      // Merge the pending queue and the base queue.
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
   
    // 将新构成的链表的尾节点赋值给 current Hook的baseQueue
    // 将该workInProgress Hook的待更新更列重置
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }


  // 开始计算阶段
  if (baseQueue !== null) {
    // We have a queue to process.
    const first = baseQueue.next;
    // 为了保证数据更新与用户触发的交互一致
    // baseState为上次到第一个尚未更新的值
    let newState = current.baseState;
    
    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;
    // 利用while循环遍历当前链表 suspense相关暂时不用管
    // 重新创建一个更新链表 预先定义两个结点 头结点newBaseQueueFirst 尾节点newBaseQueueLast
    // 如果判断不是当前更新的赛道批中 说明该更新在此次更新中依然不会被执行
 

    do {
      const suspenseConfig = update.suspenseConfig;
      const updateLane = update.lane;
      const updateEventTime = update.eventTime;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // Priority is insufficient. Skip this update. If this is the first
        // skipped update, the previous update/state is the new base
        // update/state.
        // 克隆出一个新的更新对象
        const clone: Update<S, A> = {
          eventTime: updateEventTime,
          lane: updateLane,
          suspenseConfig: suspenseConfig,
          action: update.action,
          eagerReducer: update.eagerReducer,
          eagerState: update.eagerState,
          next: (null: any),
        };
        // 呼应上面我们说的为了保证数据更新与用户触发的交互一致
        // 我们的newBaseState为链表中第一个未更新的更新对象前保存的值
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        // Update the remaining priority in the queue.
        // TODO: Don't need to accumulate this. Instead, we can remove
        // renderLanes from the original lanes.
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        // 标记跳过的赛道
        markSkippedUpdateLanes(updateLane);
      } else {
        // This update does have sufficient priority.
        // 如果当前更新对象在更新赛道里面 同样克隆一个对象不过将其lane设为NoLane
        if (newBaseQueueLast !== null) {
          const clone: Update<S, A> = {
            eventTime: updateEventTime,
            // This update is going to be committed so we never want uncommit
            // it. Using NoLane works because 0 is a subset of all bitmasks, so
            // this will never be skipped by the check above.
            lane: NoLane,
            suspenseConfig: update.suspenseConfig,
            action: update.action,
            eagerReducer: update.eagerReducer,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        // 在之前分析dispatchAction中说过
        // 如果触发一个更新的时候,如果改Fiber的更新队列为空
        // 说明当前是第一个更新对象 所以我们可以预先计算这个更新的值
        // 因为第一个更新依赖的上一次更新是确定的即Fiber.lastRenderedState
        // Process this update.
        // 因为reducer可能会改变 所以先判断一下
        if (update.eagerReducer === reducer) {
          // If this update was processed eagerly, and its reducer matches the
          // current reducer, we can use the eagerly computed state.
          newState = ((update.eagerState: any): S);
        } else {
          // 这里是真正执行更新的地方
          const action = update.action;
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    // 如果没有newBaseQueueLast 
    // 对应一种情况 说明此次更新都全部被执行了
    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      // 如果没有 将尾节点和头节点连结构成新的链表
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }


    // Mark that the fiber performed work, but only if the new state is
    // different from the current state.
    if (!is(newState, hook.memoizedState)) {
      // 标记该Fiber节点发生了改变
      markWorkInProgressReceivedUpdate();
    }

    // 重新为hook的基础属性赋值
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;

    queue.lastRenderedState = newState;
  }


  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```
代码看起来有很多，中间有些代码看起来比较绕，不知道为什么，我们首先理解这两个问题

- 如果保证更新状态与用户交互的一致性

打个比方如果当前有个更新队列如下<br />`A -> B -> C -> D`<br />用户的交互的顺序是按链表循序，所以最后的结果用户希望看到的是`abcd`,但是在假设本次更新中，<br />更新了`A`和`C`，所以此时值变为`ac`,第二次更新发生，此时更新的是`B`和`D`,如果是以之前更新后`ac`作为基础值的话，最后我们得到的是`acbd`，因为`B`和`D`的更新在后面，会造成用户看到的结果和自己交互的结果不一致的问题，那是如何保证交互的一致性的了,还记得我们之前提到过的

- **baseState**

这里的`baseState`代表着

- 链表中第一个未更新的更新对象前保存的值

拿上面的例子解释一下，在第一次更新中我们得到了

- `ac`更新后的值
- 余下剩余的链表`B -> D`

但我们知道这样其实是错的，所以我们真实得到是

- `a`更新后的值
- 余下剩余的链表`B -> C -> D`

我们保存了第一个未被更新的节点前的值以及余下所有的队列，我们这时再来看第二次更新基于当前`baseState`的`a`,依次更新得到后的值也会`abcd`与我们之前预期的一致，这里需要注意的是

- `c`节点的`lane`值会被设置为`NoLane`,在下一次更新中始终为被执行的对象

了解这个问题以后我们再看代码并总结一下`updateReducer`做了哪些事情了

1. 赋值新的`reducer`
2. 拿到上一次未更新的更新队列`baseQueue`，与当前待更新的队列`queue.pending`合并成一个新的队列
3. 创建一个新的`newBaseQueue`队列来存储这次更新后跳过的更新对象
4. 遍历新的队列，如果不符合当前的`renderLane`,克隆出一个新的更新对象并加入`newBaseQueue`，如果符合执行更新方法，并更新的状态
5. 根据新的属性重置`hook`的属性
