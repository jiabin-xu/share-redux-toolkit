---
layout: center
---
# Async Middleware
---

# 传统 Redux

```ts
import { call, put, takeEvery, takeLatest } from 'redux-saga/effects'
import Api from '...'

// worker Saga: will be fired on USER_FETCH_REQUESTED actions
function* fetchUser(action) {
  try {
    const user = yield call(Api.fetchUser, action.payload.userId)
    yield put({ type: 'USER_FETCH_SUCCEEDED', user: user })
  } catch (e) {
    yield put({ type: 'USER_FETCH_FAILED', message: e.message })
  }
}

 
function* mySaga() {
  yield takeEvery('USER_FETCH_REQUESTED', fetchUser)
}
```

---

# 传统 Redux

- 每个 saga 都要注册
- 每个 error 都要 catch
- generator 造成类型检查无法闭环

```ts
import { call, put, takeEvery, takeLatest } from 'redux-saga/effects'
import Api from '...'

// worker Saga: will be fired on USER_FETCH_REQUESTED actions
function* fetchUser(action) {
  try {
    const user = yield call(Api.fetchUser, action.payload.userId)
    yield put({ type: 'USER_FETCH_SUCCEEDED', user: user })
  } catch (e) {
    yield put({ type: 'USER_FETCH_FAILED', message: e.message })
  }
}

 
function* mySaga() {
  yield takeEvery('USER_FETCH_REQUESTED', fetchUser)
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
--- 

# Dva Effect 
```ts
function getWatcher(key, _effect, model, onError, onEffect, opts) {
  let effect = _effect;
  let type = 'takeEvery';
  let ms;
  let delayMs;
  function noop() {}

  function* sagaWithCatch(...args) {
    const { __dva_resolve: resolve = noop, __dva_reject: reject = noop } =
      args.length > 0 ? args[0] : {};
    try {
      yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@start` });
      const ret = yield effect(...args.concat(createEffects(model, opts)));
      yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@end` });
      resolve(ret);
    } catch (e) {
      onError(e, {
        key,
        effectArgs: args,
      });
      if (!e._dontReject) {
        reject(e);
      }
    }
  }
  switch (type) {
    case 'watcher':
      return sagaWithCatch;
    case 'takeLatest':
      return function*() {
        yield sagaEffects.takeLatest(key, sagaWithCatch);
      };
    case 'throttle':
      return function*() {
        yield sagaEffects.throttle(ms, key, sagaWithCatch);
      };
    default:
      return function*() {
        yield sagaEffects.takeEvery(key, sagaWithCatch);
      };
  }
}
```

---

# Dva Effect 
```ts
export default function getSaga(resolve, reject, effects, model, onError, onEffect) {
  return function *() {
    for (const key in effects) {
      if (Object.prototype.hasOwnProperty.call(effects, key)) {
        const watcher = getWatcher(resolve, reject, key, effects[key], model, onError, onEffect);
		// 将 watcher 分离到另一个线程去执行
        const task = yield sagaEffects.fork(watcher);
		// 同时 fork 了一个线程，用于在 model 卸载后取消正在进行中的 task
		// `${model.namespace}/@@CANCEL_EFFECTS` 的发出动作在 index.js 的 start 方法中，unmodel 方法里。
        yield sagaEffects.fork(function *() {
          yield sagaEffects.take(`${model.namespace}/@@CANCEL_EFFECTS`);
          yield sagaEffects.cancel(task);
        });
      }
    }
  };
}
```

---

# Toolkit
```ts
const updateUser = createAsyncThunk(
  'users/update',
  async (userData, { rejectWithValue }) => {
    const { id, ...fields } = userData
    try {
      const response = await userAPI.updateById(id, fields)
      return response.data.user
    } catch (err) {
      // Use `err.response.data` as `action.payload` for a `rejected` action,
      // by explicitly returning it using the `rejectWithValue()` utility
      return rejectWithValue(err.response.data)
    }
  }
)
```
---

# Toolkit
Handing Thunk Results
```ts
const fetchUserById = createAsyncThunk(
  'users/fetchByIdStatus',
  async (userId: number, thunkAPI) => {
    const response = await userAPI.fetchById(userId)
    return response.data
  }
)
const usersSlice = createSlice({
  name: 'users',
  initialState,
  reducers: {
    // standard reducer logic, with auto-generated action types per reducer
  },
  extraReducers: (builder) => {
    // Add reducers for additional action types here, and handle loading state as needed
    builder.addCase(fetchUserById.fulfilled, (state, action) => {
      // Add user to the state array
      state.entities.push(action.payload)
    })
  },
})
```

---

# CreateAsyncThunk 实现
```ts {all|1-6|173-182|7-26|70-75|162-170|88,104-150,161}{maxHeight:'400px'}
export const createAsyncThunk = (() => {
  function createAsyncThunk (
    typePrefix: string,
    payloadCreator,
    options?: AsyncThunkOptions<ThunkArg, ThunkApiConfig>
  ){
    const fulfilled: AsyncThunkFulfilledActionCreator<
      Returned,
      ThunkArg,
      ThunkApiConfig
    > = createAction(
      typePrefix + '/fulfilled',
      (
        payload: Returned,
        requestId: string,
        arg: ThunkArg,
        meta?: FulfilledMeta
      ) => ({
        payload,
        meta: {
          ...((meta as any) || {}),
          arg,
          requestId,
          requestStatus: 'fulfilled' as const,
        },
      })
    )

    const pending: AsyncThunkPendingActionCreator<ThunkArg, ThunkApiConfig> =
      createAction(
        typePrefix + '/pending',
        (requestId: string, arg: ThunkArg, meta?: PendingMeta) => ({
          payload: undefined,
          meta: {
            ...((meta as any) || {}),
            arg,
            requestId,
            requestStatus: 'pending' as const,
          },
        })
      )

    const rejected: AsyncThunkRejectedActionCreator<ThunkArg, ThunkApiConfig> =
      createAction(
        typePrefix + '/rejected',
        (
          error: Error | null,
          requestId: string,
          arg: ThunkArg,
          payload?: RejectedValue,
          meta?: RejectedMeta
        ) => ({
          payload,
          error: ((options && options.serializeError) || miniSerializeError)(
            error || 'Rejected'
          ) as GetSerializedErrorType<ThunkApiConfig>,
          meta: {
            ...((meta as any) || {}),
            arg,
            requestId,
            rejectedWithValue: !!payload,
            requestStatus: 'rejected' as const,
            aborted: error?.name === 'AbortError',
            condition: error?.name === 'ConditionError',
          },
        })
      )

    let displayedWarning = false

    function actionCreator(
      arg: ThunkArg
    ): AsyncThunkAction<Returned, ThunkArg, ThunkApiConfig> {
      return (dispatch, getState, extra) => {
        const requestId = options?.idGenerator
          ? options.idGenerator(arg)
          : nanoid()

        const abortController = new AC()
        let abortReason: string | undefined

        let started = false
        function abort(reason?: string) {
          abortReason = reason
          abortController.abort()
        }

        const promise = (async function () {
          let finalAction: ReturnType<typeof fulfilled | typeof rejected>
          try {
            let conditionResult = options?.condition?.(arg, { getState, extra })
            if (isThenable(conditionResult)) {
              conditionResult = await conditionResult
            }
            started = true

            const abortedPromise = new Promise<never>((_, reject) =>
              abortController.signal.addEventListener('abort', () =>
                reject({
                  name: 'AbortError',
                  message: abortReason || 'Aborted',
                })
              )
            )
            dispatch(
              pending(
                requestId,
                arg,
                options?.getPendingMeta?.(
                  { requestId, arg },
                  { getState, extra }
                )
              )
            )
            finalAction = await Promise.race([
              abortedPromise,
              Promise.resolve(
                payloadCreator(arg, {
                  dispatch,
                  getState,
                  extra,
                  requestId,
                  signal: abortController.signal,
                  abort,
                  rejectWithValue: ((
                    value: RejectedValue,
                    meta?: RejectedMeta
                  ) => {
                    return new RejectWithValue(value, meta)
                  }) as any,
                  fulfillWithValue: ((value: unknown, meta?: FulfilledMeta) => {
                    return new FulfillWithMeta(value, meta)
                  }) as any,
                })
              ).then((result) => {
                if (result instanceof RejectWithValue) {
                  throw result
                }
                if (result instanceof FulfillWithMeta) {
                  return fulfilled(result.payload, requestId, arg, result.meta)
                }
                return fulfilled(result as any, requestId, arg)
              }),
            ])
          } catch (err) {
            finalAction =
              err instanceof RejectWithValue
                ? rejected(null, requestId, arg, err.payload, err.meta)
                : rejected(err as any, requestId, arg)
          }

          const skipDispatch =
            options &&
            !options.dispatchConditionRejection &&
            rejected.match(finalAction) &&
            (finalAction as any).meta.condition

          if (!skipDispatch) {
            dispatch(finalAction)
          }
          return finalAction
        })()
        return Object.assign(promise as Promise<any>, {
          abort,
          requestId,
          arg,
          unwrap() {
            return promise.then<any>(unwrapResult)
          },
        })
      }
    }

    return Object.assign(
      actionCreator ,
      {
        pending,
        rejected,
        fulfilled,
        typePrefix,
      }
    )
  }
  createAsyncThunk.withTypes = () => createAsyncThunk

  return createAsyncThunk as CreateAsyncThunk<AsyncThunkConfig>
})()
```



