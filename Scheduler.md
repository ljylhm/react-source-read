<a name="f69hX"></a>
## 概览
在`react16`以前，`react`有以下急需解决的问题

- 一次更新带来的长时任务会滞后浏览器的渲染任务，从而导致页面出现卡顿的情况

虽然`react`团队已经将**diff算法**的优先级降到了O(n)，但对于频繁且复杂的更新任务时，依然会显得有心无力<br />所以在`react`团队带来了全新的调度机制，也就是我们下面要讲到的`scheduler`调度器。
<a name="RC08K"></a>
## 更新流程
```typescript

		触发更新 ----- scheduler调度 ----- reconciler协调 ----- render渲染 

```
当我们发起一次更新时，会经历以下三个阶段

1. 任务被注册到`scheduler`进行调度
2. 调度完成，选择一个任务进入协调阶段，如果协调阶段被“打断”，重新1步骤
3. 当上面两个阶段都已经完成，进入`render`渲染阶段，此阶段不可暂停

在整个任务过程中`scheduler`就好像地铁的调度员，决定了哪一趟车先走，哪一趟车后走，哪一趟车第一个走，哪一趟车最后一个走，而这一切都是让用户和浏览器的交互变得更加流畅。
<a name="T6ajN"></a>
## 时间切片
我们首先来理解时间切片的概念，上面说到`react`遇到了很大的困境
> 一次更新带来的长时任务会滞后浏览器的渲染任务，从而导致页面出现卡顿的情况

如果一次用户带来的长任务是不可避免的，那一次任务我们是否可以分成几次来做了，当然是可以的，在此之前我们先来看在一帧的时间范围内，会做些什么事情了
<a name="xDQsG"></a>
### 执行时机
![](https://cdn.nlark.com/yuque/0/2022/png/29455494/1666789316019-498c6367-ee5d-41e7-b3f6-7926c41651ca.png#clientId=u62371778-64a6-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=245&id=u9eba843b&margin=%5Bobject%20Object%5D&originHeight=651&originWidth=2000&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uee105ec5-fa34-4fe2-88fa-6b2b682d51b&title=&width=753)<br />概览一下，一帧中大概要以下这些事情
```typescript

一个task(宏任务) -- 队列中全部job(微任务) -- requestAnimationFrame -- 浏览器重排/重绘 -- requestIdleCallback

```
如果我们要不影响浏览器的渲染，我们有两个选择

- 在一帧渲染后的空闲时间内执行**JS**，看上去原生提供的`requestIdleCallback`非常适合
- 为了不影响当前帧的渲染，注册宏任务，放入下帧执行

`react`采用的是第二种方式，看起来第一种方式很不错，为什么没有采用了，因为`requestIdleCallback`虽然看起来很美好，但却藏着几个无法忽视的缺点，而第二种方式其实对`requestIdleCallback`的polyfill。

-   兼容性一般
- `requestIdleCallback`并不是每帧都会触发，FPS只有20ms，正常情况下渲染一帧时长控制在16.67ms (1s / 60 = 16.67ms)。该时间是高于页面流畅的诉求

你可以[点击](https://github.com/facebook/react/blob/17.0.2/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L31)这里看到`scheduler`中如何实现<br />通过代码可以到在`scheduler`中其实是使用了`MessageChannel`原生API来实现的，如果`MessageChannel`并不支持，会降级到`setTimeout`来执行；`MessageChannel`与`setTimeout`都执行的宏任务，可是`setTimeout`在启动的开始会有4ms的浪费，这在一帧中属于极大的开销，所以只能属于降级处理。
```typescript

		const channel = new MessageChannel();
    const port = channel.port2;
    channel.port1.onmessage = performWorkUntilDeadline;


```
<a name="zuFxH"></a>
### 执行时长
执行时机说到为了不影响当前帧的渲染，我们会将真正的执行操作通过`MessageChannel`放到一个事件循环中，虽然这样可以避免当前帧的渲染卡顿，如果是过长的任务依然会阻塞下一帧的渲染，所以`scheduler`给每一个任务都分配了固定的执行时间，当执行时间结束的时候，需要交还控制权给浏览器
```typescript
    forceFrameRate = function(fps) {
      if (fps < 0 || fps > 125) {
        console['error'](
          'forceFrameRate takes a positive int between 0 and 125, ' +
            'forcing frame rates higher than 125 fps is not supported',
        );
        return;
      }
      if (fps > 0) {
        yieldInterval = Math.floor(1000 / fps);
      } else {
        // reset the framerate
        yieldInterval = 5;
      }
    };
```
通过代码可以看到`scheduler`会根据当前设备的刷新率来确定一帧执行的时间，默认的时间为`5ms`，如果刷新率太高或者太低都会进入默认时长
<a name="LHdTx"></a>
### 如何交替控制权
关于`scheduler`如何与浏览器交替控制权，在我们讲完`scheduler`的整个流程最后来解答，你也可以点击这里直接跳到[最后](#xuscg)查看
<a name="BJdZt"></a>
## 队列
假设现在有很多任务在`scheduler`中，那么这些任务会分别存在在两个队列中，分别是

- **timerQueue**
- **taskQueue**

`timerQueue`存在的是延迟执行的任务，这里可以理解成任务优先级并不高的任务，`taskQueue`存在的是即将执行的任务，这里可以理解成优先级很高的任务，需要马上去执行。
<a name="Ec5ZA"></a>
### 小顶堆
在`scheduler`中使用`小顶堆`的数据结构来实现`优先级队列`<br />你可以[点击](https://github.com/facebook/react/blob/main/packages/scheduler/src/SchedulerMinHeap.js)查看`scheduler`中实现的代码<br />我们这里在这里只需要记住以下几点和方法就可

- **小顶堆的左右节点都小于它**
- **push**
- **pop**
- **peek**
- **siftUp**
- **siftDown**

<a name="X9DKl"></a>
## 优先级的概念
我们必须明确的是，在`scheduler`中优先级的概念和`React`中的优先级概念并不相同，两者进行交换的时候需要转换为对方的优先级
```typescript

	export const NoPriority = 0; // 无任何优先级
  export const ImmediatePriority = 1; // 立即执行，优先级最高，Sync模式采用这种优先级进行调度
  export const UserBlockingPriority = 2; // 用户阻塞，用户操作引起的调度任务采用该优先级调度
  export const NormalPriority = 3; // 默认的优先级
  export const LowPriority = 4; // 低优先级
  export const IdlePriority = 5; // 优先级最低，闲置的任务


```
<a name="tT7vI"></a>
## 流程
在整个`scheduler`的过程中，我们将其划分为两个阶段

- 调度阶段
- 执行阶段

`调度阶段`我们需要从现存的任务中找到优先级最高的任务，并放入`执行阶段`，在`执行阶段`中我们需要对当前任务进行控制，具体表现为

- 任务过长时暂停和恢复该任务
- 有更高优先级来时替换该任务
<a name="jZ1am"></a>
### 调度阶段
<a name="XBeJ7"></a>
#### ScheduleCallback
`scheduleCallback`是`react`和`scheduler`进行交互的重要方法,也是进入`scheduler`的入口函数，`react`中的任务通过该方法进入到`scheduler`中，在后面的流程中开启调度过程。
```typescript
// priorityLevel 传入的优先级
// callback 执行的操作
// options-delay 延迟执行的时间

function unstable_scheduleCallback(
  priorityLevel: PriorityLevel,
  callback: Callback,
  options?: {delay: number},
): Task {

  // 获取当前时间
  var currentTime = getCurrentTime();
  
  // 通过options中delay参数来确定是否任务开始时间
  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }
  
  // ===== 根据优先级定义过期时间 =====
  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  var expirationTime = startTime + timeout;
  // ===== end =====
  
  // 定义一个任务
  var newTask: Task = {
    id: taskIdCounter++,      // 任务id taskIdCounter为一个自增变量
    callback,                 // 执行的任务 传入回调函数
    priorityLevel,            // 任务优先级
    startTime,                // 开始时间
    expirationTime,           // 过期时间
    sortIndex: -1,            // 排序的索引
  };
  
  // 性能检测相关 不管
  if (enableProfiling) {
    newTask.isQueued = false;
  }
  
  // 如果任务并不是现在执行的任务 放入timerQueue队列中
  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    // 如果taskQueue中并没有待执行的任务而新任务刚好是timerQueue中的第一个
    // 开启一个调度流程
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // 如果有正在被调度的延时任务，取消它，因为当前注册的任务为更优先级的任务
			if (isHostTimeoutScheduled) {
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // 开启一个延时任务调度
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 如果当前任务为即刻任务
    newTask.sortIndex = expirationTime;
    // 首先推入到taskQueue中
    push(taskQueue, newTask);
    // 如果当前没有正在被调度的即刻任务 & 也没有正在执行的任务 立即执行
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```
通过代码我们可以看到，`scheduleCallback`相当于一个注册方法，是调度一个事件的起点，它做了如下的几件事情

1. 通过配置定义此次事件的开始时间`startTime`
2. 根据事件的不同优先级分配设定过期时间生成`expirationTime`
3. 创建一个任务对象`newTask`
4. 判断当前任务是否延迟进行，如果延迟执行，放入`timerQueue`并判断`timerQueue`是否为空，如果为空，利用延时器调度该任务；如果不延迟执行，判断当前是否有正在进行的任务，如果没有，利用`requestHostCallback`开启调度
<a name="k0Qkn"></a>
#### requestHostTimeout
在`scheduleCallback`中我们看到如果进入延时任务的判断逻辑中，会进入`requestHostTimeOut`函数中
```typescript

  requestHostTimeout = function(callback, ms) {
      taskTimeoutID = setTimeout(() => {
        callback(getCurrentTime());
      }, ms);
  };

```
可以看到`requestHostTimeOut`其实就是对`setTimeout`的封装，并传入当前事件进入`callback`，总结下来`requestHostTimeOut`的最主要作用

1. 注册延时队列中优先级最高的任务
2. 该任务会在定时器到期后进入`taskQueue`中
<a name="R4XHp"></a>
#### handleTimeOut
```typescript

	function handleTimeout(currentTime) {
      // 此次表明timerQueue的调度已经完成
      // 标志位 标记这次延时任务的调度已经完成
      isHostTimeoutScheduled = false;
      // 取出timerQueue中优先级最高的任务并放入taskQueue中
      // 每次进行一次任务调度前 都会判断timerQueue中是否有已经快过期的任务
      advanceTimers(currentTime);
      // 如果当前没有任务调度
      if (!isHostCallbackScheduled) {
        // 判断当前即刻任务队列中是否有任务
        if (peek(taskQueue) !== null) {
          isHostCallbackScheduled = true;
          // 立即开始一次任务调度
          requestHostCallback(flushWork);
        } else {
          // 如果当前有即刻任务正在被调度 & 顺便开启一个延时任务的调度
          const firstTimer = peek(timerQueue);
          // 如果有利用requestHostTimeout注册
          if (firstTimer !== null) {
            requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
          }
        }
      }
   }

```
从代码中可以看出`handleTimeout`是对调度结束的延时任务进行了后续处理，它主要做了以下几件事

1. 标记此次调度结束，并利用`advanceTimers`找到`timerQueue`中优先级最高的任务并放入`taskQueue`中
2. 如果当前没有即刻任务正在被调度，判断当前是否有优先级最高的任务，如果有则立即开启即刻任务的调度执行`requestHostCallback(flushWork)`，如果没有就从`timerQueue`中判断是否有待被调度的任务，如果有进入到延时任务的调度中

不难看到`handleTimeout`对延时任务的处理最终进入到`requestHostCallback(flushWork)`的流程中，在介绍`requestHostCallback`方法之前，我们需要先认识一下`advanceTimers`方法
<a name="Cc3Rj"></a>
#### `advanceTimers`
```typescript

	function advanceTimers(currentTime) {
      // 选出timerQueue中优先级最高的任务并且添加到taskQueue中
      let timer = peek(timerQueue);
      while (timer !== null) {
        // 判断这个任务的回调函数是否为空，如果为空直接取消该任务
        // 不能接受一个没有回调函数的任务
        if (timer.callback === null) {
          pop(timerQueue);
        } else if (timer.startTime <= currentTime) {
          // 如果开始时间小于当前时间 说明任务已经过期需要立刻被执行
          // 将其从timerQueue中取出，放入taskQueue中
          pop(timerQueue);
          timer.sortIndex = timer.expirationTime;
          push(taskQueue, timer);
        } else {
          // 是个延时任务，还没有到执行时间，继续pending
          return;
        }
        // 如果不满足条件，继续循环
        timer = peek(timerQueue);
      }
  }


```
通过代码我们可以看到`advanceTimers`的主要目的是为了在`timerQueue`找到优先级最高并且能用的任务
<a name="BGvFw"></a>
#### `requestHostCallback`
在分析`handleTimeout`这个方法的时候，会发现调用完`advanceTimers`函数后会调用`requestHostCallback`这个方法，所以`requestHostCallback`方法到底有什么作用了

```typescript

    // 设置任务
    requestHostCallback = function(callback) {
        // 设置scheduledHostCallback为将要执行的函数
        // *注意 这里的callback其实就是指代的flushWork 因为使用的时候requestHostCallback(flushWork)
        scheduledHostCallback = callback;
        // 开启新一轮的消息队列循环
        if (!isMessageLoopRunning) {
          isMessageLoopRunning = true;
          // messageChannel通知，将在下次循环的宏任务中执行scheduledHostCallback
          port.postMessage(null);
        }
    };
    // 取消任务
    cancelHostCallback = function() {
        scheduledHostCallback = null;
    };


```
可以看到代码其实非常简单，主要做了以下几件事情

- 设置全局参数`scheduledHostCallback`为`flushWork`，
- 设置标记位`isMessageLoopRunning`为true,开始新一轮的消息队列循环
- 触发`port.postMessage`，待执行事件将会在下个事件循环开始

`requestHostCallback`可以看成调度阶段的结束，因为这时候已经选出了优先级最高的任务，开启消息循环，待执行的任务将会在下个事件循环开始，开始执行阶段的过程，从这里我们也可以总结出以下

- 所有任务执行的起点是`requestHostCallback`
<a name="nOkxw"></a>
### 执行阶段
上文讲到通过`requestHostCallback`开启了一个任务的执行流程，而任务真正的执行则是在下一轮事件循环中，而在事件循环中被触发的方法就是`performWorkUntilDeadline`
<a name="cvnVo"></a>
#### performWorkUntilDeadline
```typescript
    
    const performWorkUntilDeadline = () => {
      // 在requestHostCallback里面注册的flushWork
      // scheduledHostCallback === flushWork
      if (scheduledHostCallback !== null) {
        // 设置一个当前时间
        const currentTime = getCurrentTime();
        // 确定任务在一帧的执行时间
        // dealline现在还看不到作用，但是在后面会作为判断的依据
        deadline = currentTime + yieldInterval;
        // 设置是否有剩余为true
        const hasTimeRemaining = true;
        try {
          // 执行注册的任务
          const hasMoreWork = scheduledHostCallback(
            hasTimeRemaining,
            currentTime,
          );
          // 判断 1. 如果还有多的任务 设置任务周期继续执行
          // 判断 2. 没有多的任务 结束任务周期 重设标志位 
          if (!hasMoreWork) {
            isMessageLoopRunning = false;
            scheduledHostCallback = null;
          } else {
            // If there's more work, schedule the next message event at the end
            // of the preceding one.
            port.postMessage(null);
          }
        } catch (error) {
          // If a scheduler task throws, exit the current browser task so the
          // error can be observed.
          port.postMessage(null);
          throw error;
        }
      } else {
        isMessageLoopRunning = false;
      }
      needsPaint = false;
    };


```
从方法的名字我们不难看住，`performWorkUntilDeadline`的执行一直到`deadline`截止，为此在`performWorkUntilDeadline`中做了下面的几件事

- 判断`scheduledHostCallback`是否为空，为空结束，不为空继续
- 确定`deadline`可以把这里的`deadline`当成在一帧中最大执行的时间，对应我们上面提到的时间切片
- 执行`scheduledHostCallback`函数，返回一个`hasMoreWork`标记位，这里你可以把这个标记位当成任务是否完成的意思，如果返回**true,**代表着任务并没有解析完成，于是利用`port.postMessage(null)`触发新一轮的消息循环，而我们通过上文知道
> 而在事件循环中被触发的方法就是`performWorkUntilDeadline`

所以相当于形成了一个循环，在没有结束的时候，在下一个事件循环继续没有完成的任务。<br />`performWorkUntilDeadline`的代码并不多，它更多的是流程控制，根据任务返回的结果决定做什么事情，真正执行的函数在`scheduledHostCallback`而我们在分析`requestHostCallback`讲到
> 设置全局参数`scheduledHostCallback`为`flushWork`

所以下一个流程我们需要进入`flushWork`
<a name="LovX0"></a>
#### flushWork
```typescript

function flushWork(hasTimeRemaining, initialTime) {
  // 正式进入执行过程 调度过程标志位关闭
  isHostCallbackScheduled = false;
  // 进入正式
  if (isHostTimeoutScheduled) {
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }
  // 设置执行的标志位
  isPerformingWork = true;
  const previousPriorityLevel = currentPriorityLevel;
  try {
    // 交由workLoop处理
    return workLoop(hasTimeRemaining, initialTime);
  } finally {
    // 复原之前的操作
    currentTask = null;
    currentPriorityLevel = previousPriorityLevel;
    isPerformingWork = false;
  }
}


```
通过代码我们可以看到`flushWork`做的事情很少

- 设置标记位标记执行要正式开始了，其实这里才代表着任务将会被真正的开始，设置标记位可以理解成上车前绑好的安全带，其余的事情都与我无关了
<a name="xuscg"></a>
#### shouldYieldToHost
在介绍`workLoop`之前，我们先来熟悉一个非常重要的函数`shouldYieldToHost`
```typescript

    shouldYieldToHost = function() {
      const currentTime = getCurrentTime();
      // deadline熟悉吗，我们在performWorkUntilDeadline任务开始执行前中设置了它
      // 如果超时
      if (currentTime >= deadline) {
        // 这里代表用户有更高优先级的任务进来，需要被打断
        if (needsPaint || scheduling.isInputPending()) {
          return true;
        }
        // 虽然超时了 但是没有更高优先级的任务 让子弹再飞一会儿
        return currentTime >= maxYieldInterval;
      } else {
        // 如果没有超时 告诉调用者该任务还可以继续进行
        return false;
      }
    };

```
虽然代码很简单，但这里却是`scheduler`让出控制权的关键，通过我们`performWorkUntilDeadline`设定的`deadline`进行判断

- 已经超时，判断当前是否有更优先级的任务，如果有立即让出控制权；如果没有，耗时任务可以再执行一会儿知道超出最大的时间（这里可以理解成就算当前浏览器很空闲，但是超出了最大的时间也需要交还控制权）
- 没有超时，无需交还控制权，继续执行

这里有个`scheduling.isInputPending`api,简单解释一下就是
> isInputPending()是FackBook与Google合作在Chrome浏览器上加入的一个Scheduling API，也是第一个将中断这个操作系统概念用于网页开发的API，开发者可以使用这个API来平衡JS执行、页面渲染及用户输入之间的优先级，就像系统使用中断调度CPU处理IO输入一样。

你可以[点击](https://engineering.fb.com/2019/04/22/developer-tools/isinputpending-api/)这里看到其详意
<a name="p6LB7"></a>
#### workLoop
```typescript


function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  advanceTimers(currentTime);
  currentTask = peek(taskQueue);
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {

    // 如果当前任务尚未过期 表明是一个延时任务 此刻并不需要执行
    // 通过shouldYieldToHost表明有更高优先级的任务或者时间切片已经 
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // This currentTask hasn't expired, and we've reached the deadline.
      break;
    }

    const callback = currentTask.callback;
    // 如果返回的是function说明解析并没有完成
    // task必须是个function 如果不是function的话 直接移出当前操作
    // 这里相当于一个暂停的操作
    
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      markTaskRun(currentTask, currentTime);
      // 开始执行执行操作
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      
      // 函数 任务并没有执行完
      // 不是函数 任务进行完

      // 如果返回的是个方法 将当前任务的回调依然标记为返回的函数
      if (typeof continuationCallback === 'function') {
        currentTask.callback = continuationCallback;
        markTaskYield(currentTask, currentTime);
      } else {
        // 如果不是的话 说明任务已经执行完成
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      
      advanceTimers(currentTime);
    } else {
      pop(taskQueue);
    }
    currentTask = peek(taskQueue);
  }
  
  if (currentTask !== null) {
    return true;
  } else {
    // 如果已经没有当前的任务了 当前任务已经被执行 然后重新选择了一个更高优先级的任务替代
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}


```
`workLoop`是一个非常重要的函数，我们可以把它理解成`scheduler`和`react`交互的集中地，在`react`中执行的函数是`performConcurrentWorkOnRoot`，对于这个函数我们现在并不多做了解，我们只需要知道这个函数的返回值

- 如果返回的类型是一个`Function`是表示`react`中的任务并没有执行完
- 如果返回的并不是`Function`表示任务已经完成

你可以[点击](https://github.com/facebook/react/blob/17.0.2/packages/react-reconciler/src/ReactFiberWorkLoop.new.js)这里看到`performConcurrentWorkOnRoot`的代码<br />我们接下来分析以下`workLoop`做了哪些事情

- 判断任务是否是即刻任务 & 通过`shouldYieldToHost`函数来判断是否应该移交控制权
- 如果需要移交控制权，结束判断，并返回**true**，这里的**true**其实代表着`hasMoreWork`，最终将返回到

`performWorkUntilDeadline`函数体内，进入`performWorkUntilDeadline`流程控制
<a name="w2hwR"></a>
## 流程图
![Scheduler.png](https://cdn.nlark.com/yuque/0/2022/png/29455494/1666849087281-fccd311a-0df8-4c2f-a6c5-7203205bc954.png#clientId=u84d0b4a7-d871-4&crop=0&crop=0&crop=1&crop=1&from=drop&id=ue9d48039&margin=%5Bobject%20Object%5D&name=Scheduler.png&originHeight=1374&originWidth=1938&originalType=binary&ratio=1&rotation=0&showTitle=false&size=266755&status=done&style=none&taskId=ue550b2b3-92ef-43f4-b9db-586d313b0d0&title=)
