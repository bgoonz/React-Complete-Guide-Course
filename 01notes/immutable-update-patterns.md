# Immutable Update Patterns

The articles listed in [Prerequisite Concepts#Immutable Data Management](https://redux.js.org/usage/structuring-reducers/prerequisite-concepts#immutable-data-management) give a number of good examples for how to perform basic update operations immutably, such as updating a field in an object or adding an item to the end of an array. However, reducers will often need to use those basic operations in combination to perform more complicated tasks. Here are some examples for some of the more common tasks you might have to implement.

## Updating Nested Objects[​](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#updating-nested-objects "Direct link to heading")

The key to updating nested data is that *every* level of nesting must be copied and updated appropriately. This is often a difficult concept for those learning Redux, and there are some specific problems that frequently occur when trying to update nested objects. These lead to accidental direct mutation, and should be avoided.

##### Correct Approach: Copying All Levels of Nested Data[​](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#correct-approach-copying-all-levels-of-nested-data "Direct link to heading")

Unfortunately, the process of correctly applying immutable updates to deeply nested state can easily become verbose and hard to read. Here's what an example of updating `state.first.second[someId].fourth` might look like:

```
function updateVeryNestedField(state, action) {  return {    ...state,    first: {      ...state.first,      second: {        ...state.first.second,        [action.someId]: {          ...state.first.second[action.someId],          fourth: action.someValue        }      }    }  }}
```

Obviously, each layer of nesting makes this harder to read, and gives more chances to make mistakes. This is one of several reasons why you are encouraged to keep your state flattened, and compose reducers as much as possible.

##### Common Mistake #1: New variables that point to the same objects[​](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#common-mistake-1-new-variables-that-point-to-the-same-objects "Direct link to heading")

Defining a new variable does *not* create a new actual object - it only creates another reference to the same object. An example of this error would be:

```
function updateNestedState(state, action) {  let nestedState = state.nestedState  // ERROR: this directly modifies the existing object reference - don't do this!  nestedState.nestedField = action.data  return {    ...state,    nestedState  }}
```

This function does correctly return a shallow copy of the top-level state object, but because the `nestedState` variable was still pointing at the existing object, the state was directly mutated.

##### Common Mistake #2: Only making a shallow copy of one level[​](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#common-mistake-2-only-making-a-shallow-copy-of-one-level "Direct link to heading")

Another common version of this error looks like this:

```
function updateNestedState(state, action) {  // Problem: this only does a shallow copy!  let newState = { ...state }  // ERROR: nestedState is still the same object!  newState.nestedState.nestedField = action.data  return newState}
```

Doing a shallow copy of the top level is *not* sufficient - the `nestedState` object should be copied as well.

## Inserting and Removing Items in Arrays[​](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#inserting-and-removing-items-in-arrays "Direct link to heading")

Normally, a Javascript array's contents are modified using mutative functions like `push`, `unshift`, and `splice`. Since we don't want to mutate state directly in reducers, those should normally be avoided. Because of that, you might see "insert" or "remove" behavior written like this:

```
function insertItem(array, action) {  return [    ...array.slice(0, action.index),    action.item,    ...array.slice(action.index)  ]}function removeItem(array, action) {  return [...array.slice(0, action.index), ...array.slice(action.index + 1)]}
```

However, remember that the key is that the *original in-memory reference* is not modified. As long as we make a copy first, we can safely mutate the copy. Note that this is true for both arrays and objects, but nested values still must be updated using the same rules.

This means that we could also write the insert and remove functions like this:

```
function insertItem(array, action) {  let newArray = array.slice()  newArray.splice(action.index, 0, action.item)  return newArray}function removeItem(array, action) {  let newArray = array.slice()  newArray.splice(action.index, 1)  return newArray}
```

The remove function could also be implemented as:

```
function removeItem(array, action) {  return array.filter((item, index) => index !== action.index)}
```

## Updating an Item in an Array[​](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#updating-an-item-in-an-array "Direct link to heading")

Updating one item in an array can be accomplished by using `Array.map`, returning a new value for the item we want to update, and returning the existing values for all other items:

```
function updateObjectInArray(array, action) {  return array.map((item, index) => {    if (index !== action.index) {      // This isn't the item we care about - keep it as-is      return item    }    // Otherwise, this is the one we want - return an updated value    return {      ...item,      ...action.item    }  })}
```

## Immutable Update Utility Libraries[​](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#immutable-update-utility-libraries "Direct link to heading")

Because writing immutable update code can become tedious, there are a number of utility libraries that try to abstract out the process. These libraries vary in APIs and usage, but all try to provide a shorter and more succinct way of writing these updates. For example, [Immer](https://github.com/mweststrate/immer) makes immutable updates a simple function and plain JavaScript objects:

```
var usersState = [{ name: 'John Doe', address: { city: 'London' } }]var newState = immer.produce(usersState, draftState => {  draftState[0].name = 'Jon Doe'  draftState[0].address.city = 'Paris'  //nested update similar to mutable way})
```

Some, like [dot-prop-immutable](https://github.com/debitoor/dot-prop-immutable), take string paths for commands:

```
state = dotProp.set(state, `todos.${index}.complete`, true)
```

Others, like [immutability-helper](https://github.com/kolodny/immutability-helper) (a fork of the now-deprecated React Immutability Helpers addon), use nested values and helper functions:

```
var collection = [1, 2, { a: [12, 17, 15] }]var newCollection = update(collection, {  2: { a: { $splice: [[1, 1, 13, 14]] } }})
```

They can provide a useful alternative to writing manual immutable update logic.

A list of many immutable update utilities can be found in the [Immutable Data#Immutable Update Utilities](https://github.com/markerikson/redux-ecosystem-links/blob/master/immutable-data.md#immutable-update-utilities) section of the [Redux Addons Catalog](https://github.com/markerikson/redux-ecosystem-links).

## Simplifying Immutable Updates with Redux Toolkit[​](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#simplifying-immutable-updates-with-redux-toolkit "Direct link to heading")

Our [Redux Toolkit](https://redux-toolkit.js.org/) package includes a [`createReducer` utility](https://redux-toolkit.js.org/api/createReducer) that uses Immer internally. Because of this, you can write reducers that appear to "mutate" state, but the updates are actually applied immutably.

This allows immutable update logic to be written in a much simpler way. Here's what the [nested data example](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns#correct-approach-copying-all-levels-of-nested-data) might look like using `createReducer`:

```
import { createReducer } from '@reduxjs/toolkit'const initialState = {  first: {    second: {      id1: { fourth: 'a' },      id2: { fourth: 'b' }    }  }}const reducer = createReducer(initialState, {  UPDATE_ITEM: (state, action) => {    state.first.second[action.someId].fourth = action.someValue  }})
```

This is clearly *much* shorter and easier to read. However, this *only* works correctly if you are using the "magic" `createReducer` function from Redux Toolkit that wraps this reducer in Immer's [`produce` function](https://immerjs.github.io/immer/produce). If this reducer is used without Immer, it will actually mutate the state!. It's also not obvious just by looking at the code that this function is actually safe and updates the state immutably. Please make sure you understand the concepts of immutable updates fully. If you do use this, it may help to add some comments to your code that explain your reducers are using Redux Toolkit and Immer.

In addition, Redux Toolkit's [`createSlice` utility](https://redux-toolkit.js.org/api/createSlice) will auto-generate action creators and action types based on the reducer functions you provide, with the same Immer-powered update capabilities inside.
