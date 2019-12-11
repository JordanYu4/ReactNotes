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

It might not be the most straightforward path to the solution, but I found the some of the detours instructive in how hooks work, so thought I'd share: 

```JS
useEffect(() => {
  const ellipsisAnimation = setInterval(() => {
    setEllipsis(ellipsis.length > 2 ? '.' : `${ellipsis}.`);
  }, 500);
  return () => clearInterval(ellipsisAnimation);
});
```
Simply running setInterval in useEffect accomplishes the animation, but a new interval is set and cleared every time the useEffect runs. Gets the job done, but less than ideal: 

```JS
useEffect(() => {
  const ellipsisAnimation = setInterval(() => {
    setEllipsis(ellipsis.length > 2 ? '.' : `${ellipsis}.`);
  }, 500);
  return () => clearInterval(ellipsisAnimation);
}, []);
```
Using `[]` signals that the effect is independent of any value participating in the React data flow, and will only be applied once upon the very first render and never again. In this way it can closely simulate `componentDidMount` – with a nuanced difference in that it closes over the initial props and state, whereas `componentDidMount` accesses the most recent mutation of `this.state`. When we tried adding `[]` to the animation effect, the effect ran once the modal was rendered in the background, saw `isHidden = false`, and never set the interval. That's why the animation never ran. 

```JS
useEffect(() => {
  const ellipsisAnimation = setInterval(() => {
    setEllipsis(ellipsis.length > 2 ? '.' : `${ellipsis}.`);
  }, 500);
  return () => clearInterval(ellipsisAnimation);
}, [isHidden]);
```
When we use [isHidden], the effect only fires when the value of isHidden changes (i.e. the modal toggles visibility). Since the modal can't close until payment finishes processing, then the effect only fires once – that's great, since we only want to set a single interval. 

But `useState` declares each state value as a constant every time a functional component is called. This means that both the value of `ellipsis` and the ellipsis iteration callback are contained within a closure when we pass them to `setInterval`. Regardless of when it’s fired or how many times the component re-renders, the interval callback still only sees the constant value of `ellipsis` from when the callback is defined right before the first render. Every time the callback runs, it still thinks that `ellipsis` is "...", and the animation gets stuck in the first iteration.

```JS
const savedCallback = useRef();

const callback = () => {
  setEllipsis(ellipsis.length > 2 ? '.' : `${ellipsis}.`);
};

useEffect(() => {
  savedCallback.current = callback;
});

useEffect(() => {
  const iterateEllipsis = () => savedCallback.current();

  const ellipsisInterval = setInterval(iterateEllipsis, 500);
  return () => clearInterval(ellipsisInterval);
}, []);
```

The solution, as Sean suggested (looks like a simplified version of Abramov's `useInterval` [custom hook](https://overreacted.io/making-setinterval-declarative-with-react-hooks/)), is to redefine the iteration callback with a separate `useEffect` every time the component is called, so that it can close over the most recent `ellipsis` value. Keeping track of this callback with `useRef` allows the interval to call the most recent version. The ellipsis animation works again! However, the interval is still being set in the background, causing unnecessary background work. 

```JS
// in review.jsx

{!isProcessingModalHidden && (
  <ProcessingModal
    isHidden={isProcessingModalHidden}
  />
)}
```
As Sean suggested, rendering the component conditionally in the parent component’s JSX prevents the component from being called until it’s opened and ensures the interval is only set when we need it. 
```JS
useEffect(() => {
  let ellipsisInterval;
  if (!isHidden && isHidden !== undefined) {
    const iterateEllipsis = () => savedCallback.current();
    ellipsisInterval = setInterval(iterateEllipsis, 500);
  }
  return () => clearInterval(ellipsisInterval);
}, [isHidden]);
```
However, I think it makes more sense to keep that optimization in the modal component instead of relying upon the parent, so I went with Annie's suggestion of adding a condition inside of the `useEffect` hook. 