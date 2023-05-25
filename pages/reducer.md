---
layout: center
---
# Reducer
---

# 传统 Redux

```ts
const initialState = { value: 0 }

function counterReducer(state = initialState, action) {
  switch (action.type) {
    case 'increment':
      return { ...state, value: state.value + 1 }
    case 'decrement':
      return { ...state, value: state.value - 1 }
    case 'incrementByAmount':
      return { ...state, value: state.value + action.payload }
    default:
      return state
  }
}

```
---

# Toolkit
Map Object

```ts
const increment = createAction('increment')
const decrement = createAction('decrement')

const counterReducer = createReducer(0, {
  [increment]: (state, action) => state + action.payload,
  [decrement.type]: (state, action) => state - action.payload
})

```

---

# Toolkit
Builder Callback
```ts

const initialState: Record<string, string> = {}
const resetAction = createAction('reset-tracked-loading-state')

function isPendingAction(action: AnyAction): action is PendingAction {
  return action.type.endsWith('/pending')
}

const reducer = createReducer(initialState, (builder) => {
  builder
    .addCase(resetAction, () => initialState)
    // matcher can be defined outside as a type predicate function
    .addMatcher(isPendingAction, (state, action) => {
      state[action.meta.requestId] = 'pending'
    })
    .addDefaultCase((state, action) => {
      state.otherActions++
    })
})

```
---

# CreateReducer 实现

```ts {all|3|7-11|21-26|30-35}{maxHeight:'300px'}
export function createReducer<S extends NotFunction<any>>(
  initialState: S | (() => S),
  mapOrBuilderCallback 
  actionMatchers: ReadonlyActionMatcherDescriptionCollection<S> = [],
  defaultCaseReducer?: CaseReducer<S>): ReducerWithInitialState<S> {

  let [actionsMap, finalActionMatchers, finalDefaultCaseReducer] =
    typeof mapOrBuilderCallback === 'function'
      ? executeReducerBuilderCallback(mapOrBuilderCallback)
      : [mapOrBuilderCallback, actionMatchers, defaultCaseReducer]

let getInitialState: () => S
  if (isStateFunction(initialState)) {
    getInitialState = () => freezeDraftable(initialState())
  } else {
    const frozenInitialState = freezeDraftable(initialState)
    getInitialState = () => frozenInitialState
  }

  function reducer(state = getInitialState(), action: any): S {
    let caseReducers = [
      actionsMap[action.type],
      ...finalActionMatchers
        .filter(({ matcher }) => matcher(action))
        .map(({ reducer }) => reducer),
    ]
    if (caseReducers.filter((cr) => !!cr).length === 0) {
      caseReducers = [finalDefaultCaseReducer]
    }

    return caseReducers.reduce((previousState, caseReducer): S => {

      const draft = previousState as Draft<S> // We can assume this is already a draft
      const result = caseReducer(draft, action)

      if (result === undefined) {
        return previousState
      }

      return result as S

    }, state)
  }

  reducer.getInitialState = getInitialState

  return reducer as ReducerWithInitialState<S>
}
```
---

# Dva
  
```ts
app.model({
  namespace: 'count',
  state: 0,
  reducers: {
    add(state) { return state + 1 },
  },
  effects: {
    *addAfter1Second(action, { call, put }) {
      yield call(delay, 1000);
      yield put({ type: 'add' });
    },
  },
});
```