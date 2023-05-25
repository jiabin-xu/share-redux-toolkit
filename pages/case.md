---
layout: cover
---

# 几个提效的场景

---

# dva-loading
  
```ts
loading: {
  global: false,
  models: {
    users: false,
    todos: false,
  },
  effects: {
    'users/fetch': false,
    'users/create': false,
    'todos/fetch': false,
  },
}
```

---

# 在Toolkit 里实现一个 toolkit-loading

```ts
const loadingSlice = createSlice({
  name: 'loading',
  initialState: {
    global: false,
    models: {},
    effects: {},
  } as initialStateType,
  reducers: {},
  extraReducers: (builder) => {
    builder.addMatcher(isPendingAction, (state, action) => {
      console.log(action)
      const [namespace, actionType] = action.type.split('/')
      state.global = true
      state.models[namespace] = true
      state.effects[namespace + '/' + actionType] = true
    })
    builder.addMatcher(isRejectedAction, (state, action) => {
      message.error((action.error as any).message)
      const [namespace, actionType] = action.type.split('/')
      state.effects[namespace + '/' + actionType] = false
      state.models[namespace] = Object.keys(state.effects).some((key) => key.startsWith(namespace) && state.effects[key])
      state.global = Object.keys(state.effects).some((key) => state.effects[key])
    })
    builder.addMatcher(isFulfilledAction, (state, action) => {
      const [namespace, actionType] = action.type.split('/')
      state.effects[namespace + '/' + actionType] = false
      state.models[namespace] = Object.keys(state.effects).some((key) => key.startsWith(namespace) && state.effects[key])
      state.global = Object.keys(state.effects).some((key) => state.effects[key])
    })
  },
})
```
---

# Entity Adapter


```ts {all} {maxHeight: '400px'}
// 定义todoAdapter
const todosAdapter = createEntityAdapter<Todo>({
  selectId: (todo) => todo.id,
  sortComparer: (a, b) => b.id - a.id,
})

// selectors
export const {
  selectIds: selectTodoIds,
  selectEntities: selectTodoEntities,
  selectAll: selectAllTodos,
  selectTotal: selectTotalTodos,
} = todosAdapter.getSelectors()

const initialState  = {
  ids: [],
  entities: {},
}

// reducer
export const todosSlice = createSlice({
  name: 'todos',
  initialState,
  reducers: {
    addOne: todosAdapter.addOne,
    addMany: todosAdapter.addMany,
    upsertOne: todosAdapter.upsertOne,
    upsertMany: todosAdapter.upsertMany,
    setOne: todosAdapter.setOne,
    setMany: todosAdapter.setMany,
    removeOne: todosAdapter.removeOne,
    removeMany: todosAdapter.removeMany,
    updateOne: todosAdapter.updateOne,
    updateMany: todosAdapter.updateMany,
  },
  extraReducers: (builder) => {
    builder.addCase(fetchTodos.fulfilled, (state, action) => {
      state.status = 'idle'
      todosAdapter.setMany(state, action.payload)
    })
  },
})
```
---

# 全局 Effect 错误处理
 

