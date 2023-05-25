---
layout: center
---
# Slice & Namespace
---

# Toolkit
  
```ts
import { createSlice } from '@reduxjs/toolkit'
import type { PayloadAction } from '@reduxjs/toolkit'

interface CounterState {
  value: number
}

const initialState = { value: 0 } as CounterState

const counterSlice = createSlice({
  name: 'counter',
  initialState,
  reducers: {
    increment(state) {
      state.value++
    },
    decrement(state) {
      state.value--
    },
    incrementByAmount(state, action: PayloadAction<number>) {
      state.value += action.payload
    },
  },
})

export const { increment, decrement, incrementByAmount } = counterSlice.actions
export default counterSlice.reducer

```
---

# CreateSlice 实现
  
```ts {all|2,15,21|4,5,10|all}{maxHeight:'400px'}
export function createSlice (
  options: CreateSliceOptions<State, CaseReducers, Name>
): Slice<State, CaseReducers, Name> {
  const { name } = options

  const initialState =
    typeof options.initialState == 'function'
      ? options.initialState
      : freezeDraftable(options.initialState)

  const reducers = options.reducers || {}

  const reducerNames = Object.keys(reducers)

  const sliceCaseReducersByName: Record<string, CaseReducer> = {}
  const sliceCaseReducersByType: Record<string, CaseReducer> = {}
  const actionCreators: Record<string, Function> = {}

  reducerNames.forEach((reducerName) => {
    const maybeReducerWithPrepare = reducers[reducerName]
    const type = getType(name, reducerName)

    let caseReducer: CaseReducer<State, any>
    let prepareCallback: PrepareAction<any> | undefined

    if ('reducer' in maybeReducerWithPrepare) {
      caseReducer = maybeReducerWithPrepare.reducer
      prepareCallback = maybeReducerWithPrepare.prepare
    } else {
      caseReducer = maybeReducerWithPrepare
    }

    sliceCaseReducersByName[reducerName] = caseReducer
    sliceCaseReducersByType[type] = caseReducer
    actionCreators[reducerName] = prepareCallback
      ? createAction(type, prepareCallback)
      : createAction(type)
  })

  function buildReducer() {

    const [
      extraReducers = {},
      actionMatchers = [],
      defaultCaseReducer = undefined,
    ] =
      typeof options.extraReducers === 'function'
        ? executeReducerBuilderCallback(options.extraReducers)
        : [options.extraReducers]

    const finalCaseReducers = { ...extraReducers, ...sliceCaseReducersByType }

    return createReducer(initialState, (builder) => {
      for (let key in finalCaseReducers) {
        builder.addCase(key, finalCaseReducers[key] as CaseReducer<any>)
      }
      for (let m of actionMatchers) {
        builder.addMatcher(m.matcher, m.reducer)
      }
      if (defaultCaseReducer) {
        builder.addDefaultCase(defaultCaseReducer)
      }
    })
  }

  let _reducer: ReducerWithInitialState<State>

  return {
    name,
    reducer(state, action) {
      if (!_reducer) _reducer = buildReducer()

      return _reducer(state, action)
    },
    actions: actionCreators as any,
    caseReducers: sliceCaseReducersByName as any,
    getInitialState() {
      if (!_reducer) _reducer = buildReducer()

      return _reducer.getInitialState()
    },
  }
}

```
---

# Dva

```ts {maxHeight:'400px'}
app.model({
  namespace: 'todo',
  state: [],
  reducers: {
    add(state, { payload: todo }) {
      // 保存数据到 state
      return [...state, todo];
    },
  },
  effects: {
    *save({ payload: todo }, { put, call }) {
      // 调用 saveTodoToServer，成功后触发 `add` action 保存到 state
      yield call(saveTodoToServer, todo);
      yield put({ type: 'add', payload: todo });
    },
  },
  subscriptions: {
    setup({ history, dispatch }) {
      // 监听 history 变化，当进入 `/` 时触发 `load` action
      return history.listen(({ pathname }) => {
        if (pathname === '/') {
          dispatch({ type: 'load' });
        }
      });
    },
  },
});
```


