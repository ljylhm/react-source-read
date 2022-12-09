我们知道`useState`和`useReducer`同样可以触发一次更新,在`更新流程`中，那它们之间到底有什么异同了<br />你可以[点击](https://www.yuque.com/junyuan-ruxka/foof2g/ctg6gv)这里查看`useReducer`的解析
<a name="YBMFn"></a>
## 挂载阶段
我们来到`mountState`，这里是`useState`更新的起点
```typescript
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  });
  const dispatch: Dispatch<
    BasicStateAction<S>,
    > = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
}
```
从代码可以看出，`mountState`和`mountReducer`非常相似<br />你可以点击这里查看`mountReducer`的源码<br />与`useReducer`在挂载阶段唯一不同的是

- **basicStateReducer**

将`lastRenderedReducer`赋值为一个固定的`reducer`
```typescript
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}
```
可以看到`baseRenderedReducer`只是判断了传进来的`action`是什么<br />事实上你可以这么写一个状态的更新
```typescript

  const UserStatus = {
    admin: 1,
    normal: 2,
    vip: 3,
  }

	const [status, setStatus] = useState(UserStatus.admin)

	// 某个点击事件 举例用
  changeUserStatus = () => {
    // 可以这么写
    setStatus(UserStatus.normal)
    // 也可以这么写
    setStatus((status) => {
      return status === UserStatus.admin ? UserStatus.normal : UserStatus.vip
    })
  }


```
<a name="ApCmf"></a>
## 更新阶段
```typescript
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```
通过代码可以看到，`updateState`直接返回的就是`updateReducer`，而传进去的`basicStateReducer`依然是固定的`reducer`
<a name="yoxsL"></a>
## 总结
`useState`和`useReducer`之间没有区别，`useState`只是固定了传进去的`reducer`<br />你可以[点击](https://www.yuque.com/junyuan-ruxka/foof2g/ctg6gv)这里查看`useReducer`的解析
