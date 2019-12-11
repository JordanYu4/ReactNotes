## useEffect notes ##

Replicating `componentDidMount`: while you can use `useEffect(fin, [])`, it's not an exact equivalent because it will capture initial props and state. It's possible to access latest props and state by writing to a ref or restructuring the code. 

"The mental model for effects is differnet from `componentDidMount` and other lifecycles, and trying to find their exact equivalents may confuse you more than help. To get productive, you need to 'think in effects,' and their mental model is closer to implementing synchronization than to responding to lifecycle events." (Abramov)

Using `[]` with `useEffect` indicates that the effect is independent of any value participating in the React data flow, and will only be applied once. This can cause bugs when an ommitted value actually does need to signal a change. Using `useReducer` and `useCallback` to remove the need for dependencies is often preferable to ommitting them altogether. 

### Understand Rendering 

State values are constants within render functions. Using bracketed values in JSX doesn't embed a "watcher," it merely embeds the constant into the output value. Each time a relevant state value changes, the component is called and a new render output is returned. Thus it can be said that each render sees its own state values.

Similarly, each render also has its own event handlers. Even async functions inside event handlers are scoped to the same "snapshot" of props and state. 

### useEffect

`useEffect` is the same as any other function – it's a different function on each render. 

"React remembers the effect function you provided, and runs it after flushing changes to the DOM  and letting the browser paint the screen [...] Conceptually, you can imagine effects are part of the render result" (Abramov).

Even with a `setTimeOut` delay in the function, each referenced state value (e.g. `count`) will refer to the constant declared in its component call. In contrast, `this.state.count` in a class implementation will refer to the most recent value of `count`. React mutates `this.state` in class impolementations to point to the latest state.  

Note that hooks rely heavily upon JS closures

### Sources 
- [A Complete Guide to useEffect - Dan Abramov](https://overreacted.io/a-complete-guide-to-useeffect/)
