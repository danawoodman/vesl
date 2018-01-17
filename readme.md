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

## Event System

Actions are just functions. Pass actions into components and components fire actions. Components know nothing about what the action is or where it came from.

Actions are typically just simple functions that trigger state change and can optionally be passed props from the component. They can also trigger other actions simply by calling them. Async actions just return a promise. Actions can be chained together to create an declarative flow of actions.

An action can mutate state (without a reducer) which triggers a re-render of any components dependent on that particular state.

Actions create an internal payload that can be inspected based only on the state that changes. This payload includes the path that was mutated, the type of mutation and any values passed in. These actions scan be 'played back' like you can with Redux actions but without having to implement a dispatcher. The dispatcher is in fact the state updating mechanism itself and is invest less to you, unless you don’t want it to be (ala devtools etc).

### `state.set`

```js
state.set(`users.${props.id}.name`, 'John')
```

Results in:

```js
{
  path: 'users.1.name',
  method: 'set',
  data: 'John'
}
```

### `state.concat`

```js
state.concat(`users.${props.id}.interests`, ['travel', 'coding'])
```

Results in:

```js
{
  path: 'users.1.interests',
  method: 'concat',
  data: [ 'travel', 'coding' ]
}
```

### `state.merge`

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

Results in:

```js
{
  path: 'colors',
  method: 'merge',
  data: { red: '#f00' }
}
```

New state:

```js
{ colors: { blue: '#00f', red: '#f00' } }
```

### `state.splice`

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

Results in:

```js
{
  path: 'tags',
  method: 'merge',
  data: { red: '#f00' },
  args: [ 1, 0 ]
}
```

New state:

```js
{
  tags: ['a', 'z', 'b']
}
```

Methods: set, push, unshift, merge, concat, splice, pop (all reasonable array and object methods)

### Replaying events

You can replay events with the internal 'trigger' method:

```js
const event = {
  path: 'users.1.name',
  method: 'set',
  data: 'John',
}
state.trigger(event)
```

This is just the underlying implementation of the event helper methods which are just syntax sugar.

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

Existing Solutions

* Redux
* MobX
* cerebral.

## The Name

Pronounced: _Vessel (/ˈvesəl/)_

Meaning: _a hollow container, especially one used to hold liquid, such as a bowl or cask._

The first letter in Vesl is capitalized and never all lower or uppercase.
