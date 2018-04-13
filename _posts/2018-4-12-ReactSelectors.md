---
layout: post
title: Be Selective With Your State
---

# Be Selective With Your State
## A Dive into Selectors featuring `reselect` and `re-reselect`.

This article focuses on selectors in the context of a `React` application backed by a `redux` store. If you or your friends are writing a `React` application with `redux` and like to do things right.. this article is for you!

What is a selector anyways? If a `redux` store is kind of like a database then selectors are like queries. And just like you would normalize a database you should be storing minimal state in your `redux` store. One problem with minimal state is that the derived data your components depend on can be computation intensive or complicated to get from the minimal representation. Selectors can solve all these problems and more. Selectors are such a central part of application data flow that at Riipen we decided to audit our selector usage. We found that we were not getting any benefit from the way we used `reselect` and that needed to change.

Lets explore the motivation for using selectors. One common usage of selectors is to cache expensive computations. This way the value won’t get recomputed until the underlying relevant state changes. Another usage is to return the exact same (===) array or object on a computed set of data. This is useful to keep `React` from needlessly re-rendering a component. An example of this is below.
This component re-renders sooo often.

In the example above the component will re-render every time it receives new props. Changing state.someArray.filter(Math.isEven) to someArrayEvenSelector(state) with a proper implementation of the selector means the component doesn’t re-render until the underlying state changes. With thousands of components and functions more intensive than isEven this performance benefit can add up quickly.

Note that `reselect` and `redux` are built on the idea of immutable state. Mutating objects in `redux` state will break `reselect` and cause your selectors not to recompute when they should!

Lets look at some code!

What is `reselect`? `reselect` is a simple selector library for `redux`. The classic example is using a selector to calculate total price based on some items in a cart and a tax rate. Lets deviate a bit and break down a simplified real-world example from Riipen.

First you need some context. Here is the shape of our `redux` store and an example.

```javascript
/* 
 * const stateShape = {
 *   entities: {
 *     [pluralEntityName]: {
 *       [entityId]: entityObject,
 *     },
 *   },
 * };
 */
const exampleState = {
  entities: {
    users: {
      '123': { name: 'John', id: 123, hobby: 'sports' },
    },
  },
};
```
The relevant piece of our `redux` store.

And here is an example of selecting entities from the `redux` store above.

```javascript
import { createSelector } from 'reselect';

const entitiesSelector = createSelector(
  // Return entities based on type
  // Will be recomputed if state, type, or name change.
  (state, type, /* name */) => state.entities[type],
  
  // Return the name exactly as passed in
  // Will be recomputed if state, type, or name change.
  (state, type, name) => name,
  
  // Expensive calculation here!!!
  // Return something computed from the other 2 return values.
  // Outputs of the other functions are used as inputs to the last function.
  // Will be recomputed if entities or name change. 
  (entities, name) => 
    Object.values(entities)
      .filter((entity) => entity.name === name),
);

// Example usage:
const mapStateToProps = (state) => ({
  allUsersNamedBob: entitiesSelector(state, 'users', name: 'Bob'),
});

```
An example for selecting entities.

If you are experienced with `reselect` you will see the problems right away. You may be thinking, “I hate contrived examples in tech articles”. Unfortunately this is not very contrived and has been far too real for far too long 😭.

If you are not experienced with `reselect` don’t worry, read on!

In `reselect`’s createSelector the final function takes the results of all the previous functions as arguments. Each of the “previous functions” takes the arguments the selector is called with. There is an alternative way to write the functions to draw a clear distinction between the two types of function. Putting the first set of functions in array identifies all the intermediate selectors.

```javascript
const entitiesSelector = createSelector(
  [
    (state, type, /* name */) => state.entities[type],
    (state, type, name) => name,
  ],
  (entities, name) =>
    Object.values(entities)
      .filter((entity) => entity.name === name),
);
```
The previous example with clearer syntax.

Great! We can select and filter arrays of entities by name everywhere in our application! Wow isn’t this convenient, and we wrote very little code! But what is the “magic” that `reselect` is doing? The benefit of `reselect` is memoization. Memoization means storing the results of function calls to use them in place of recomputing the value.

    A functional programming aside: A pure function gives the same output for any set of inputs and doesn’t have any side effects. This means if you call the function twice with the same arguments you can use the result of the first call and forget about the second call all together. A perfect candidate for memoization! And… selectors are pure functions (a side-effect of this is they are super testable too)!

`reselect` selectors keep track of the arguments and return values and only recalculate the return value if the arguments have changed. I said “magic” above but `reselect` is around 75 lines of code and there is not much magic going on. It is a library that is definitely worth reading if you use it!

Unfortunately there is a classic gotcha in the example above. entitiesSelector has a cache size of 1 so using the selector in more than one location (or more than one instance of a component) results in recalculating often.

```javascript
const mapStateToProps = (state) => ({
  bobUsers: entitiesSelector(state, 'users', 'Bob'),
  karenUsers: entitiesSelector(state, 'users', 'Karen'),
});
```
Improper selector usage, no memoization.

The example above will break memoization because of the change in the name argument. In fact, even cross component usage in two separate mapStateToProps functions will break memoization!

One solution is to build a factory for each option.

```javascript
const allUsersNamedFactory = (name) => createSelector(
  (state) => entitiesSelector(state, {
    type: 'users',
    name,
  }),
  (users) => users,
});

// We pull this out of the mapStateToProps body because otherwise a new 
// selector is created on every call of mapStateToProps, this also renders the
// selector's memoization useless.
const karenSelector = allUsersNamedFactory('Karen');
const bobSelector = allUsersNamedFactory('Bob');

// Example usage:
const mapStateToProps = (state) => ({
  bobUsers: bobSelector(state),
  karenUsers: karenSelector(state),
});
```
An example with a selector factory.

This is perfect! We are back in business. These 2 selectors can be used all over the application with expected memoization. The drawbacks of this are:

    These selectors are no longer configurable; we can’t dynamically select based on props in mapStateToProps
    There needs to be a selector created for every combination of argument values that is used

Another solution (that maintains dynamic selecting ability based on props) is to make an entitySelector factory and then use makeMapStateToProps instead of mapStateToProps. This will call the factory per component instance giving you a fresh instance of entitySelector:

```javascript
// First we need to change entitySelector into a factory
const makeEntitySelector = () => createSelector(
  (state, type, /* name */) => state.entities[type],
  (state, type, name) => name,
  (entities, name) => 
    Object.values(entities)
      .filter((entity) => entity.name === name),
);

// Example usage:
const makeMapStateToProps = () => {
  // A new entitySelector is created for each component instance.
  const entitySelector = makeEntitySelector();

  const mapStateToProps = (state, props) => ({
    users: entitySelector(state, 'users', props.name),
  });

  return mapStateToProps;
}
```
An example of a selector factory with dynamic selecting.

Hopefully this illustrates there is a lot of boilerplate and thought that goes into correct selector usage. But luckily you are already using `redux` so you love boilerplate. One minor drawback to boilerplate is that it is easy to forget, or mess up. In the case of selectors this most likely results in your application silently using selectors incorrectly. Hmm, silent failure. That sounds pretty bad.

This is where `re-reselect` comes in. `re-reselect` builds a cache of multiple `reselect` selectors cached based on input arguments. So our entitiesSelector would now be:

```javascript
import createCachedSelector from 're-reselect';

const entitiesSelector = createCachedSelector(
  (state, type, /* name */) => state.entities[type],
  (state, type, name) => name,
  (entities, name) => 
    Object.values(entities)
      .filter((entity) => entity.name === name),
)(
  (state, type, name) => `${type}:${name}`
);

// Example usage:
const mapStateToProps = (state) => ({
  bobUsers: entitiesSelector(state, 'users', 'Bob'),
  karenUsers: entitiesSelector(state, 'users', 'Karen'),
});
```
An example of a cached selector.

The only difference is now we include a function to calculate the cache key based on selector input arguments. The usage is the same as `reselect` but with proper memoization! This means `re-reselect` can be a drop in replacement even if you are using `reselect` incorrectly. `re-reselect` uses the cache key to create/get a different `reselect` selector based on different arguments.

At Riipen, we were using `reselect` incorrectly for a long time. The selectors were a part of the code base that nobody wanted to touch. It seemed like they had a high degree of complexity and a huge impact. Almost every single component uses selectors and a refactor would have a huge impact with no current perceivable benefit. We eventually bit the bullet on this piece of technical debt and invested approximately a week of developer time into adding unit tests for our selectors and refactoring. The benefit has been a huge confidence boost in the teams dealings with selectors and no mysterious performance degradation in the future. Hopefully our mistakes can help inform your decisions on how to properly integrate `reselect` and `re-reselect` into your `React`/`redux` project.
