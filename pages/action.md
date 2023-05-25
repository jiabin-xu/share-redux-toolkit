---
layout: center
---
# Action
---

# Toolkit

```ts
import { createAction } from '@reduxjs/toolkit'

const increment = createAction<number | undefined>('counter/increment')

let action = increment()
// { type: 'counter/increment' }

action = increment(3)
// returns { type: 'counter/increment', payload: 3 }

console.log(increment.toString())
// 'counter/increment'

console.log(`The action type is: ${increment}`)
// 'The action type is: counter/increment'
```
---

# Toolkit

```ts
import { createAction, nanoid } from '@reduxjs/toolkit'
const addTodo = createAction('todos/add', function prepare(text: string) {
  return {
    payload: {
      text,
      id: nanoid(),
      createdAt: new Date().toISOString(),
    },
  }
})
console.log(addTodo('Write more docs'))
/**
 * {
 *   type: 'todos/add',
 *   payload: {
 *     text: 'Write more docs',
 *     id: '4AJvwMSWEHCchcWYga3dj',
 *     createdAt: '2019-10-03T07:53:36.581Z'
 *   }
 * }
 **/
```
---

# Toolkit

```ts{all|2,15,21|4,5,10|all}
export function createAction(type: string, prepareAction?: Function): any {
  function actionCreator(...args: any[]) {
    if (prepareAction) {
      let prepared = prepareAction(...args)
      if (!prepared) {
        throw new Error('prepareAction did not return an object')
      }
      return {
        type,
        payload: prepared.payload,
        ...('meta' in prepared && { meta: prepared.meta }),
        ...('error' in prepared && { error: prepared.error }),
      }
    }
    return { type, payload: args[0] }
  }
  actionCreator.toString = () => `${type}`
  actionCreator.type = type
  actionCreator.match = (action: Action<unknown>): action is PayloadAction =>
    action.type === type
  return actionCreator
}
```
# Dva
没有 Action Creator
