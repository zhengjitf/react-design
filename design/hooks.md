# 概览

## memoizedState
`fiber.memoizedState`：`FunctionComponent` 对应 `fiber` 保存的 `Hooks` 链表。
`hook.memoizedState`：`Hooks` 链表中保存的单一 `hook` 对应的数据

不同类型 `hook` 的 `memoizedState` 保存不同类型数据，具体如下：

- `useState`：对于 `const [state, updateState] = useState(initialState)` ，`memoizedState` 保存 `state` 的值

- `useReducer`：对于 `const [state, dispatch] = useReducer(reducer, {})`，`memoizedState` 保存 `state` 的值

- `useEffect`：`memoizedState` 保存包含 `useEffect` 回调函数、依赖项等的链表数据结构 `effect`。`effect` 链表同时会保存在`fiber.updateQueue` 中
```js
type Effect = {
  tag: HookEffectTag,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<mixed> | null,
  next: Effect,
}
```

- `useRef`：对于 `useRef(1)` ，`memoizedState` 保存 `{current: 1}`

- `useMemo`：对于 `useMemo(callback, [depA])`，`memoizedState` 保存 `[callback(), depA]`

- `useCallback`：对于 `useCallback(callback, [depA])`，`memoizedState` 保存`[callback, depA]`。与 `useMemo` 的区别是，`useCallback` 保存的是 `callback` 函数本身，而 `useMemo` 保存的是 `callback` 函数的执行结果

# useState

## mount 阶段

```js
// packages/react-reconciler/src/ReactFiberBeginWork.js
updateFunctionComponent =>

// packages/react-reconciler/src/ReactFiberHooks.js
renderWithHooks => ReactCurrentDispatcher.current = HooksDispatcherOnMount => 

// packages/react/src/ReactHooks.js
useState => 

// packages/react-reconciler/src/ReactFiberHooks.js
// mountWorkInProgressHook: 生成一个 workInProgressHook（第一个 hook） 挂载到 workInProgress.memoizedState，第二个 hook 挂载到 workInProgressHook 的 next 属性上，以此类推
// TIPS: useState 第一个参数可以是函数，以返回值作为初始值
HooksDispatcherOnMount.useState => mountState ( return [hook.memoizedState, dispatch] ) => mountWorkInProgressHook
```

`workInProgressHook` 对象结构如下：

```ts
type Update = {
  expirationTime: expirationTime,
  suspenseConfig: suspenseConfig,
  // const [count, setCount] = useState(0)
  // setCount 传入的参数就是 action，这里可以是数字或传入当前 state 返回新 state 的函数
  action: action,
  eagerReducer: null,
  // 需要提交的最新值
  eagerState: null,
  next: null
}

type Queue = {
  pending: Update,
  dispatch: typeof dispatchAction,
  // 默认值 basicStateReducer，一般 useReducer 才会替换该值
  lastRenderedReducer: basicStateReducer,
  // 最新（上一次计算的）状态值
  lastRenderedState: initialState
}

type Hook = {
  // 当前状态值
  memoizedState: any
  // 初始值？
  baseState: any
  baseQueue: null
  queue: Queue
  next: Hook
}
```

## update 阶段

```js
调用 dispatch => dispatchAction => 生成 update 对象，挂载到 queue.pending => scheduleWork (scheduleUpdateOnFiber) => ensureRootIsScheduled => performSyncWorkOnRoot => 

// packages/react-reconciler/src/ReactFiberHooks.js
renderWithHooks => ReactCurrentDispatcher.current = HooksDispatcherOnUpdate => 

// packages/react/src/ReactHooks.js
useState => 

// packages/react-reconciler/src/ReactFiberHooks.js
HooksDispatcherOnUpdate.useState => updateState ( return [hook.memoizedState, dispatch] ) => updateReducer => updateWorkInProgressHook
```

在 `updateWorkInProgressHook` 中存放了一个指针 `currentHook` 指向当前的 `Hook`，取的是 `mount` 阶段挂载到 fiber 对象 memoizedState 属性上的 hook（链表），执行一个 hook 函数后，`currentHook` 指向 `currentHook.next`，所以每次 hook 函数调用的顺序必须一致，或者说每次 render 所有 hook 函数都得调用，不能有条件判断控制 hook 函数调用

# useEffect

## mount 阶段
```js
// packages/react-reconciler/src/ReactFiberHooks.js
// 在 mountEffectImpl 内 
mountEffect => mountEffectImpl => pushEffect


commitBeforeMutationEffects => flushPassiveEffects (scheduleCallback) => flushPassiveEffectsImpl => commitPassiveHookEffects => commitHookEffectListUnmount => commitHookEffectListMount =>

// packages/react-reconciler/src/ReactFiberCommitWork.js
commitHookEffectListMount
```
