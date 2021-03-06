# Common pitfalls & best practices

Stuck with MobX? This section contains a list of common issues people new to MobX might run into.

#### `Array.isArray(observable([1,2,3])) === false`

In ES5 there is no way to reliably inherit from arrays, and hence observable arrays inherit from objects.
This means that regularly libraries are not able to recognize observable arrays as normal arrays (like lodash, or built-in operations like `Array.concat`).
This can simply be fixed by passing calling `observable.toJS()` or `observable.slice()` before passing the array to another library.
As long as the external library has no intent to modify the array, this will further work completely as expected.
You can use `isObservableArray(observable)` to check whether something is an observable array.

#### `object.someNewProp = value` is not picked up 

MobX observable _objects_ do not detect or react to property assignments that weren't declared observable before.
So MobX observable objects act as records with predefined keys.
You can use `extendObservable(target, props)` to introduce new observable properties to an object.
However object iterators like `for .. in` or `Object.keys()` won't react to this automatically.
If you need a dynamically keyed object, for example to store users by id, create observable _map_s using `asMap`. 

### Use `@observer` on all components that render `@observable`'s.

`@observer` only enhances the component you are decorating, not the components used inside it.
So usually all your components should be decorated. Don't worry, this is not inefficient, in contrast, more `observer` components make rendering more efficient.

### Dereference values as lately as possible

MobX can do a lot, but it cannot make primitive values observable (although it can wrap them in an object see [boxed observables](boxed.md)).
So it are not the _values_ that are observable, but the _properties_ of an object. This means that `@observer` actually reacts to the fact that you dereference a value.
So in our above example, the `Timer` component would **not** react if it was initialized as follows:

```javascript
React.render(<Timer timerData={timerData.secondsPassed} />, document.body)
```

In this snippet just the current value of `secondsPassed` is passed to the `Timer`, which is the immutable value `0` (all primitives are immutable in JS).
That number won't change anymore in the future, so `Timer` will never update. It is the property `secondsPassed` that will change in the future,
so we need to access it *in* the component. Or in other words: always try to pass the owning object of an observable property.

#### Computed values run more often then expected

If a computed property is *not* in use by some reaction (`autorun`, `observer` etc), computed expressions will be evaluated lazily; each time their value is requested (so they just act as normal property).
Computed values will only track their dependencies if they are observed.
This allows MobX to automatically suspend computations that are not actively in use.
See this [blog](https://medium.com/@mweststrate/becoming-fully-reactive-an-in-depth-explanation-of-mobservable-55995262a254) or [issue #356](https://github.com/mobxjs/mobx/issues/356) for an explanation.
So if you fiddle arounds, computed properties might not seem efficient. But when applied in a project that uses `observer`, `autorun` etc, they become very efficient.

#### I have a weird exception when using `@observable` in a React component.

The following exception: `Uncaught TypeError: Cannot assign to read only property '__mobxLazyInitializers' of object` occurs when using a `react-hot-loader` that does not support decorators.
Either use `extendObservable` in `componentWillMount` instead of `@observable`, or upgrade to `react-hot-loader` `"^3.0.0-beta.2"` or higher.

#### The display name of react components is not set

If you use `export const MyComponent = observer((props => <div>hi</div>))`, no display name will be visible in the devtools.
The following approaches can be used to fix this:

```javascript
// 1 (set displayName explicitly)
export const MyComponent = observer((props => <div>hi</div>))
myComponent.displayName = "MyComponent"

// 2 (MobX infers component name from function name)
export const MyComponent = observer(function MyComponent(props) { return <div>hi</div> })

// 3 (transpiler will infer component name from variable name)
const _MyComponent = observer((props => <div>hi</div>)) // 
export const MyComponent = observer(_MyComponent)

// 4 (with default export)
const MyComponent = observer((props => <div>hi</div>))
export default observer(MyComponent)
```

See also: http://mobxjs.github.io/mobx/best/stateless-HMR.html or [#141](https://github.com/mobxjs/mobx/issues/141#issuecomment-228457886).

#### The propType of an observable array is object

Observable arrays are actually objects, so they comply to `propTypes.object` instead of `array`.
`mobx-react` will [soon](https://github.com/mobxjs/mobx-react/pull/59) provide explicit propTypes for observable data structures.

#### Declaring propTypes might cause unnecessary renders in dev mode

See: https://github.com/mobxjs/mobx-react/issues/56 

#### `@observable` properties initialize lazily when using Babel

This issue only occurs when transpiling with Babel and not with Typescript (in which decorator support is more mature).
Observable properties will not be instantiated upon an instance until the first read / write to a property (at that point they all will be initialized).
This results in the following subtle bug:

```javascript
class Todo {
    @observable done = true
    @observable title = "test"
}
const todo = new Todo()

"done" in todo // true
todo.hasOwnProperty("done") // false
Object.keys(todo) // []

console.log(todo.title)
"done" in todo // true
todo.hasOwnProperty("done") // true
Object.keys(todo) // ["done", "title"]
``` 

In practice this is rarely an issue, only when using generic methods like `Object.assign(target, todo)` or `assert.deepEquals` *before* reading or writing any property of the object.
If you want to make sure that this issue doesn't occur, just initialize the fields in the constructor instead of at the field declaration or use `extendObservable` to create the observable properties. 
