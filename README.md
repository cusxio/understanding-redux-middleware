# Introduction
Redux's [applyMiddleware](https://github.com/rackt/redux/blob/master/src/utils/applyMiddleware.js "Redux Middleware") function is a very succinct and beautiful piece of code. The beauty comes from functional programming techniques such as *currying* and *compose* and understanding these two techniques is fundamental to understand the redux applyMiddleware function.

# Pre-requisites 
Reading and understanding the pre-requisites concept is important to understand the Redux's applyMiddleware function.

## Functional Programming Concept - Currying
The first concept to understand in currying. Currying is a simple concept to understand. The most pragmatic definition is from the book [Mostly adequate guide to FP](https://github.com/MostlyAdequate/mostly-adequate-guide). It states that currying is when:
> You can call a function with fewer arguments than it expects, and it returns a function that takes the remaining arguments. 

You can see currying in action from this piece of code: 

```javascript
function adder(x) {
    return function(y) {
        return x+y;
    }
}

var addOne = adder(1);
var addTen = adder(10);

addOne(1) // 2
addTen(1) // 11
```
With currying, you are supplying partial information/arguments into the function. The function's `addone` and `addTen` are called **partially applied** functions, because they were partially supplied with `1` and `10` respectively and they both return a **function** that *remembers* the partially supplied information via [closures](http://javascriptissexy.com/understand-javascript-closures-with-ease/). This is the essence of currying, any functions can be curried by using libraries such as [lodash-curry](https://lodash.com/docs#curry) or [ramda-curry](http://ramdajs.com/0.18.0/docs/#curry).

## Functional Programming Concept - Compose
The second concept to understand is compose. Compose is a utility technique that can be used to 'pipe' data through functions.

```javascript
var compose = function(f,g) {
    return function(x) {
        return f(g(x));
    };
};
```
The two functions `f` anad `g`are composed into a new singular function via the `compose` utility function. From the code above, it can be seen that `g(x)` is evaluated first and the data/result is later passed into the function `f(x)`, creating a data flow from right to left. As an example:

```javascript
var firstItem = function(x) {
    return x[0];
};

var toUpper = function(x) {
    return x.toUpperCase();
};

var firstItemToUpper = compose(toUpper, firstItem);

firstItemToUpper(['ironman', 'batman']); // 'IRONMAN'
```
To re-emphasize, data flows from right to left. **This is important**. 

The string `'ironman'` is first extracted from the array, and then being supplied to the `toUpper` function as the argument resulting in `'IRONMAN'`.
At this stage, the compose function is limited to compose functions from two individual functions. Redux has a better implementation of `compose` which we will see later.

## ES6 Features
1. Spread Operator:
[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator)
2. Rest Parameters:
[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters)
3. Arrow Functions: [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)

* TIP: If you are confused by the ES6 Syntax, you can use [BABEL/REPL](https://babeljs.io/repl/) to output the code to ES5 so you can have a better grasp of the function. Or check out https://github.com/addyosmani/es6-equivalents-in-es5.

# Redux's Middleware
To begin understanding how redux's [applyMiddleware](https://github.com/rackt/redux/blob/master/src/utils/applyMiddleware.js "Redux Middleware") function work. It is a good idea to know the function signature of Redux's middleware. According to the [official doc's]([applyMiddleware](https://github.com/rackt/redux/blob/master/src/utils/applyMiddleware.js "Redux Middleware"), The middleware should have a function signature of `middleware:: store => next => action`.

As an example: 
```javascript
// ES6
const logger = store => next => action => {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}
```
As mentioned before, if you have trouble reading the ES6 syntax, just paste the ES6 code into  [BABEL/REPL](https://babeljs.io/repl/) and see the ES5 output.

```javascript
//ES5 Equivalent
var logger = function logger(store) {
  return function (next) {
    return function (action) {
      console.group(action.type);
      console.info('dispatching', action);
      var result = next(action);
      console.log('next state', store.getState());
      console.groupEnd(action.type);
      return result;
    };
  };
};
```
At this stage, all you need to know about Redux's middleware is that it accepts a `store` as the first argument, and returns a function that accepts a `next` argument. The returned function will then return a new function that accepts the `action` argument. Hence the function signature ``store => next => action``  . ***This is all you need to know***, it needs a `store` a `next` and a `action` argument to be fully invoked. With that in mind, lets move on.


# Redux's applyMiddleware
```javascript
// To create a store with middleware: 

import { createStore, applyMiddleware } from 'redux';
import { reducerFunc } from ./reducer;

let createStoreWithMiddleware = applyMiddleware(middleware)(createStore)

// middleware can be any middleware created with the function signature of store => next => action as specified above.

let store = createStoreWithMiddleware(reducerFunc);
```
To understand how this works, lets see the source for [applyMiddleware.js](https://github.com/rackt/redux/blob/master/src/utils/applyMiddleware.js "Redux Middleware")

```javascript
function applyMiddleware(...middlewares) {
  return (next) => (reducer, initialState) => {
    var store = next(reducer, initialState)
    var dispatch = store.dispatch
    var chain = []

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```
___
```javascript
function applyMiddleware(...middlewares)
```
The function is using ES6 spread operator, which allows it to accept an unspecified number of arguments. Each arguments (in this case, the middleware function's) are then placed into an `array` called `middlewares`.

*TIP: You can check the [ES5 output](http://bit.ly/1Wr57Jh) of applyMiddleware to be sure* . 

Once `applyMiddleware(middleware)` is invoke, we get the returned function `(next) => reducer, initialState)`.
This is where Redux's [createStore.js](https://github.com/rackt/redux/blob/master/src/createStore.js) is supplied as the argument. This is necessary to get an *instance* of the store, when `var store = next(reducer, initialState)` is invoked.

`var store = next(reducer,initialState)` is equivalent to `var store = createStore(reducer, initialState)`

This is why the last and final argument to the `applyMiddleware` function is the `reducer` function.
___
```javascript
var dispatch = store.dispatch
```
A variable is created to hold the store's original dispatch function.
___
```javascript
var chain = [];
```
An empty array `chain` is created to hold each middleware function injected with the `middlewareAPI` which will be explained below.

___
```javascript
var middlewareAPI = {
    getState: store.getState,
    dispatch: (action) => dispatch(action)
}
```
This steps creates an object wrapping the `getState` and `dispatch` method from the store instance. Essentially, the store's API is extracted and stored into a variable called `middlewareAPI`.

```javascript
chain = middlewares.map(middleware => middleware(middlewareAPI))
```
The extracted store's API is then injected into each middleware function. As mentioned earlier, the spread operator places each middleware function into an array called `middlewares`, and the `middlewares` array is then iterated so that each middleware function can be injected with the store's API.
This is the pseudocode for the result:
```javascript
chain = [middleware1(middlewareAPI), middleware2(middlewareAPI), middleware3(middlewareAPI)]  //etc..
```
___

```javascript
dispatch = compose(...chain)(store.dispatch)
```
Finally, the hardest line of code to understand. Below is Redux's implementation of compose.
```javascript
//ES6
function compose(...funcs) {
    return arg => funcs.reduceRight((composed, f) => f(composed), arg)
}

// ES5
function compose() {
  for (var _len = arguments.length, funcs = Array(_len), _key = 0; _key < _len; _key++) {
    funcs[_key] = arguments[_key];
  }

  return funcs.reduceRight(function (composed, f) { // composed = previousValue, f = currentValue;
    return f(composed);
  });
}
```
We know that compose flows data from **left to right**. Now, imagine we have 3 middleware that are already injected with the middlewareAPI as mentioned above. Our middleware functions in the `chain` variable should have the following signature.
```javascript
// ES6
const middlewareOne = next => action => {
  let result = next(action)
  return result
}
const middlewareTwo = next => action => {
  let result = next(action)
  return result
}

const middlewareThree = next => action => {
  let result = next(action)
  return result
}
```
We also know that `store.dispatch` is a function awaiting for an `action` argument object as seen in [createStore.js](https://github.com/rackt/redux/blob/master/src/createStore.js). The compose function flows data from right to left, hence each middleware will be injected as the `next` argument into the next middleware.

Redux's compose uses [Array.reduceRight](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Array/reduceRight) The result of compose will look something like this (pesudo-code):

call  | previousValue | currentValue | index | array | returnValue(pseudo-code)
------------- | ------------- | ------------- | :-------------: | ------------- | ------------- 
first call | `store.dispatch`| middlewareThree | 2 | [middlewareOne, middlewareTwo, middlewareThree] | `middlewareThree(store.dispatch)`
second call| `middlewareThree(store.dispatch)` | middlewareTwo | 1 |[middlewareOne, middlewareTwo, middlewareThree] | `middlewareTwo(middlewareThree(store.dispatch))`
third call | `middlewareTwo(middlewareThree(store.dispatch))` | middlewareTwo | 0 |[middlewareOne middlewareTwo, middlewareThree] | `middlewareOne(middlewareTwo(middlewareThree(store.dispatch)))`

Therefore, `store.dispatch(action)` action will always be the last function called. Hence, middleware working as it should, working **in between** a request and a response. 

More pseudo-code to ease-understanding
```javascript
// First Call
// calling middlewareThree(store.dispatch) will return a function as below: 

middlewareThreeReturn(action) {
    let result = store.dispatch(action);
    return result;
}

// Second Call
//calling middlewareTwo(MiddlewareThree(store.dispatch)) will return a function as below: 
middlewareTwoReturn(action) {
    let result = middlewareThreeReturn(action);
    return result;
}
// Third Call
// calling middlewareOne(MiddlewareTwo(MiddlewareThree(store.dispatch)))
middlewareOne(action) {
    let result = middlewareTwo(action);
    return result;
}
```
Following the logic from the pseudo-code, `middlewareThree` will always be invoked last, because invoking `middlewareThree` invokes `store.dispatch` which ultimately dispatches the **final** action.

___

The final line of code
```javascript
  return {
      ...store,
      dispatch
    }
```
basically [extends](http://bit.ly/1Wr57Jh) the original `store` object with the newly composed dispatch function with all the middleware goodness.


## Conclusion

This is how I understand middleware in Redux. Hope it helps :)
