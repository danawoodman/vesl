# Vesl

> A state container you'll love using.

**This is very much a conceptual tool right now and is just a place to brainstorm, check back later**

## Vision

* **Simple**: The primary goal of Vesl is to be the simplest possible state container for dynamic JavaScript applications. Minimal boilerplate, convention over configuration.
* **Single state tree**: All application data lives in one JavaScript object which represents state.
* **Immutable-ish**: State is mutated with special helpers and subscribers listen to changes to the location on the tree and not the values so you get immutable-like features without all the fuss.
* **Library friendly**: Designed to be library agnostic so it can work with React, Angular, Vue, vanilla JS or any library of the future.
* **Computed data**: A simple, functional way to create computed values that can be listened to like any other value.
* **Univeral**: Works on any browser (from IE 6 and up), mobile, desktop, IoT or anywhere else.
* **Performant**: Because of it's functional nature and immutable-like behavior, you get performance for free. Subscribed components only update when the part of the state they care about changes.
* **Testable**: Actions are simple functions, state is a plain object which makes testing Vesl code simple.
* **Modern**: Written to take advantage of ES6/7/Next features to make Vesl code clear and concise.
* **Helpful debugging tools**: A modern and capable browser extension with history rollback, detailed event logging and much more. Helpful error messages in development guide you to what is wrong and how to fix it. All the tools the modern developer expects.
* **Flexible**: Designed to be easy to pick and choose only the features you need.

## High-Level Overview

At it's core, Vesl is just two things:

1. **State**: Represented as a simple JavaScript object which things like components can be subscribed to changes.
2. **Actions**: Functions that change (mutate) state or call other actions.

We will go into these two core concepts but for the impatient, here is what a very simple Vesl application could look like:

```js
import { createState } from 'vesl'

// Create your state object
const state = createState({ title: 'Vesl is awesome' })

// Listen for changes to the "title" key in your state
state.subscribe('title', title => console.log('Title is now:', title))

// Change the title, which triggers the subscriber above
state.set('title', 'Vesl rocks!')
```

If you run this, you will see "Title is now: Vesl rocks!" logged to the console. This shows the core concepts including creating a state object, subscribing to changes to a particular part of the state and then updating the state (which triggers the subscriber function to get called).

This is about as simple as you can get with Vesl but of course this isn't too useful yet, so keep reading!

### State

Vesl provides a simple abstraction over changing state that allows for the features above, without forcing you to learn any advanced concepts (reducers, dispatchers, observables, immutablility, etc).

With Vesl, you create an initial state that is a simple object and then you create actions that change that state. For example, let's say this is your application's initial state:

```js
const state = {
  todos: [
    { title: 'Get milk', done: false },
    { title: 'Take out trash', done: true },
  ],
}
```

This state could be loaded from a database or fetched from an API, but we will go over all that later.

Now, to hook this state up to Vesl, you just wrap your state object with the `createState` function:

```js
import { createState } from 'vesl'

const state = createState({
  todos: [
    { title: 'Get milk', done: false },
    { title: 'Take out trash', done: true },
  ],
})
```

The `createState` method is where all the magic happens. Under the covers, it wraps your state object in a collection of methods that you can use to get and update state, subscribe to changes and a lot more.

#### Updating and retrieving state

Ok, so now that we have our state object, let's do something useful with it.

First off, you can make simple changes to the state like adding and updating values:

```js
state.push('todos', { title: 'Try out Vesl', done: false })
state.set('todos[2].done', true)
state.get('todos[2]') // { title: 'Try out Vesl', done: true }
```

As you can see here, we are first adding a new item to the `todos` list with the `push` method. This method behaves exactly how JavaScript's `Array.push()` method behaves so you don't need to learn any new concepts when working with data in Vesl.

Next, you'll notice an interesting syntax for getting one of the `todos` by it's array index: `todos[2].done`. This is basically a string version of the syntax you're already used to when working with JavScript objects. So, these are conceptually equivalent:

```js
state.set('todos[2].done', true)
// Conceptually equivalent to:
state.todos[2].done = true
```

_Note: This method of accessing data within an object is referred to as "dot path" retrieval._

Now, you may be wondering why we do this instead of just directly modifying the object. The main reason for this approach is that it makes watching for changes to your application state easy for Vesl to track and, thus, update your application only where necessary. If we just mutated the state object, we wouldn't know what happened without doing tricky things like using `Object.observe`, or other more complex approaches.

An additional benefit to this is that now we can get immutable-like features in our app without having to use tools like Immutable.js which have a steep learning curve and require lots of changes to your application to implement. With Vesl, we get the real benefits of immutability like shallow comparisons, while technically not being purely immutable. If you're interested in learning more about this, please read further down where we discuss how state works in Vesl.

#### Subscribing to changes

Now, this isn't all that useful yet if you can't react to changes. This is where subscribing to changes comes in. In it's simplest form, you can subscribe to particular piece of the state object:

```js
state.subscribe('title', title => console.log(title))
```

Now, if you call something like:

```js
state.set('title', 'Hello world!')
```

You will see Vesl log "Hello world!" to the console. This is showing a shorthand version of subscribing to simple parts of the state tree. If, however, you want to subscribe to more than one part of the tree, you have that option as well:

```js
state.subscribe(
  ['name', 'posts[0].title', 'friends(2)'], // subscriptions
  (name, recentPostTitle, friends) => {
    console.log(
      `${name} has ${
        friends.length
      } friends and his most recent post was called "${recentPostTitle}`
    )
  }
)
```

This is a more involved example showing a few new concepts. First off, subscribe can take an array of positions on the state tree that you want to listen to. The first one is `name` which just simply finds the key "name" at the root of your state tree. The second value uses "dot notation" to access the post with the array index `0` just like you would normally do in JavaScript. The third item is what is called a "Computed Property" which is a way to create a computed (or derived) value form your state. We won't go over this too much just yet; the important takeaway is that it returns a value just like the other paths we saw before. You can [read further down][computed] to learn more about computed properties.

Now, before we go further, using `subscribe` is probably something you won't use if you're using a tool like React as we have custom bindings for React (and soon other frameworks/libraries) that simplify this process drastically and make subscribing to changes more intuitive, so don't worry too much if this is a bit confusing right now, it will get simpler very soon.

### Actions

Now that we've seen how to do basic mutations and subscribing to changes, the next major concept to understand is actions. Actions are just simple JavaScript functions that mutate state, for example:

```js
function addTodo(todo) {
  state.push('todos', todo)
}
```

## Event System

Actions are just functions. Pass actions into components and components fire actions. Components know nothing about what the action is or where it came from.

Actions are typically just simple functions that trigger state change and can optionally be passed props from the component. They can also trigger other actions simply by calling them. Async actions just return a promise. Actions can be chained together to create an declarative flow of actions.

An action can mutate state (without a reducer) which triggers a re-render of any components dependent on that particular state.

Actions create an internal payload that can be inspected based only on the state that changes. This payload includes the path that was mutated, the type of mutation and any values passed in. These actions scan be 'played back' like you can with Redux actions but without having to implement a dispatcher. The dispatcher is in fact the state updating mechanism itself and invisible to you, unless you don’t want it to be (e.g. devtools etc).

## State Management

State is stored in a simple JS object. This object can be serialized and stored in local storage or rehydrated via isomorphic applications.

At first the state can just be a simple object definition but as the app grows, leaves of the tree can be added in a declarative way.

State is updated via actions which trigger a mutation event. This event directly modified the object in place so changes are efficient. Since they mutate state, components subscribe to state changes of the part of the tree they care about.

### `state.subscribe`

```js
state.subscribe(`users.${props.id}.name`, () => console.log('name changed!'))
```

With React, this is used in a higher order component to trigger an intelligent re-render:

## Components

Leverage the huge and powerful ecosystem of React. Strive for pure functions when/where possible. Wrap components in higher order components which provide actions and state.

These higher order components allow for two things:

1. Passing in actions
2. Subscribing to state

### `<StateProvider>`

The `StateProvider` component wraps your React application and provides access to your state when using the `Connect` component. It uses `context` which allows passing down data all the way through the React component tree. This context is only created so that `Connect` has access to the application state.

```js
class StateProvider extends Component {
  getChildContext() {
    return { state: this.props.state }
  }
}

StateProvider.childContextTypes = {
  state: PropTypes.object,
}
```

Usage:

```js
ReactDOM.render(
  <StateProvider state={state}>
    <Application />
  </StateProvider>,
  document.body
)
```

### `<Connect>`

The `Connect` component wraps a React component and provides state and actions to the component. The `Connect` component is never used directly but instead is used by the `connect` higher-order component.

```js
class Connect extends Component {
  componentWillMount() {
    this.context.state.subscribe(
      this.props
    )
  }

  componentWillUnmount() {
    this.context.state.unsubscribe(

    )
  }
￼

  render() {
    return <Component {...this.props} />
  }
}

Connect.contextTypes = {
  state: PropTypes.object
}
```

### `connect()`

The `connect` higher-order component wraps a React component with the `Connect` component that provides state and actions to the component.

```js
function connect(propGetter, Component) {
  const props = propGetter()
  return <Connect Component={Component} {...props} />
}
```

Usage:

```js
connect(
  props => ({
    tags: 'tags',
    user: `users.${props.id}`,
    friends: `friends(${props.id})`,
  }),
  TagList
)
```

### `state.computed`

Create derived state which components can subscribe to. If any dependent values for the computed property change, the property re-evaluates and thus re-renders any listening components.

```js
state.computed({
  friends: {
    listen(props) {
      return {
        people: 'people',
        id: props.id,
      }
    },
    compute({ id, people }) {
      return people.filter(p => id === p.id)
    },
  },

  // Computed state can depend on
  // other computed state:
  besties: {
    listen(props) {
      return {
        friends: `friends(${props.id})`,
      }
    },
    compute({ friends }) {
      return friends.filter(f => f.lovesCats)
    },
  },
})
```

If you call a computed part of state without the function parameters, you will see a helpful error message explaining the issue. In production we automatically call it with no parameters to prevent a runtime exception. Eg;

```js
state.get('besties')
```

Will throw an error in development saying you need to add parens `()` to the call.

## Middleware

## Notes

Situations to address:

* Managing async actions
* Triggering series of actions
* Pure re-rendering of components
* Middleware-like behavior wrapping actions or state changes
* Easy to work with any library but designed for React
* State change navigation history
* Undo behavior should be easy
* Needs a devtools extension to explore and change state
* Creating (and depending on) computed state should be simple

---

# API

## `state.set`

```js
state.set(`users.${props.id}.name`, 'John')
```

Fires event:

```js
{
  path: 'users.1.name',
  method: 'set',
  data: 'John'
}
```

## `state.concat`

```js
state.concat(`users.${props.id}.interests`, ['travel', 'coding'])
```

Fires event:

```js
{
  path: 'users.1.interests',
  method: 'concat',
  data: [ 'travel', 'coding' ]
}
```

## `state.merge`

Given the state:

```js
{
  colors: {
    blue: '#00f'
  }
}
```

Calling:

```js
state.merge('colors', { red: '#f00' })
```

Fires event:

```js
{
  path: 'colors',
  method: 'merge',
  data: { red: '#f00' }
}
```

Which changes state to:

```js
{ colors: { blue: '#00f', red: '#f00' } }
```

## `state.splice`

Given the state:

```js
{
  tags: ['a', 'b']
}
```

Calling:

```js
state.splice(
  'tags',
  1, // index
  0, // delete count
  'z'
)
```

Fires event:

```js
{
  path: 'tags',
  method: 'merge',
  data: { red: '#f00' },
  args: [ 1, 0 ]
}
```

Which changes state to:

```js
{
  tags: ['a', 'z', 'b']
}
```

Methods: set, push, unshift, merge, concat, splice, pop (all reasonable array and object methods)

## `state.trigger`

You can replay events with the internal 'trigger' method:

```js
const event = {
  path: 'users.1.name',
  method: 'set',
  data: 'John',
}
state.trigger(event)
```

This is just the underlying implementation of the event helper methods which are just syntax sugar. This is only really useful for things like the Vesl devtools extension.

---

## The Name

Pronounced: _Vessel (/ˈvesəl/)_

Meaning: _a hollow container, especially one used to hold liquid, such as a bowl or cask._

The first letter in Vesl is capitalized and never all lower or uppercase.

[computed]: /#computed-properties

## Credits

Copyright Dana Woodman &copy; 2018.

Many ideas were borrowed (stolen?) from other great tools, including:

* Redux
* MobX
* cerebral.

## License

MIT
