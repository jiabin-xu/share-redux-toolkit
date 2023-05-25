---
layout: cover
---
# 回顾 Redux
---

# Redux 是什么

## A Predictable State Container for JS Apps

![Redux](/redux.png)

---

# Redux 核心概念

## Actions

```ts

const addTodoType = "todos/todoAdded"

const addTodo = (todo) => {
  return {
    type: addTodoType,
    payload: todo,
  };
};
```

---

## Reducers

```ts
const initialState = { value: 0 };

function counterReducer(state = initialState, action) {
  // Check to see if the reducer cares about this action
  if (action.type === "counter/incremented") {
    // If so, make a copy of `state`
    return {
      ...state,
      // and update the copy with the new value
      value: state.value + 1,
    };
  }
  // otherwise return the existing state unchanged
  return state;
}
```

---

## Store

```ts
import { applyMiddleware, createStore } from "redux";
import { composeWithDevTools } from "redux-devtools-extension";
import thunkMiddleware from "redux-thunk";

import monitorReducersEnhancer from "./enhancers/monitorReducers";
import loggerMiddleware from "./middleware/logger";
import rootReducer from "./reducers";

export default function configureStore(preloadedState) {
  const middlewares = [loggerMiddleware, thunkMiddleware];
  const middlewareEnhancer = applyMiddleware(...middlewares);

  const enhancers = [middlewareEnhancer, monitorReducersEnhancer];
  const composedEnhancers = composeWithDevTools(...enhancers);

  const store = createStore(rootReducer, preloadedState, composedEnhancers);

  if (process.env.NODE_ENV_ !== "production" && module.hot) {
    module.hot.accept("./reducers", () => store.replaceReducer(rootReducer));
  }
  return store;
}
```

---

## Dispatch

```ts{2|15}
import React, { useState } from "react";
import { useDispatch } from "react-redux";

const Header = () => {
  const [text, setText] = useState("");
  const dispatch = useDispatch();

  const handleChange = (e) => setText(e.target.value);

  const handleKeyDown = (e) => {
    const trimmedText = e.target.value.trim();
    // If the user pressed the Enter key:
    if (e.key === "Enter" && trimmedText) {
      // Dispatch the "todo added" action with this text
      dispatch({ type: "todos/todoAdded", payload: trimmedText });
      // And clear out the text input
      setText("");
    }
  };
  return (
    <input
      type="text"
      placeholder="What needs to be done?"
      autoFocus={true}
      value={text}
      onChange={handleChange}
      onKeyDown={handleKeyDown}
    />
  );
};
export default Header;
```

---

## Selectors

```ts{2|10-13}
import React from 'react'
import { useSelector } from 'react-redux'

import { availableColors, capitalize } from '../filters/colors'
import { StatusFilters } from '../filters/filtersSlice'

// Omit other footer components

const Footer = () => {
  const todosRemaining = useSelector(state => {
    const uncompletedTodos = state.todos.filter(todo => !todo.completed)
    return uncompletedTodos.length
  })

  const { status, colors } = useSelector(state => state.filters)
```