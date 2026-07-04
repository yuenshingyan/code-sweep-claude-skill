# Frontend Bugs & Rendering

- **Missing/incorrect list key**: List rendering without a stable unique key, or keyed by array index where items reorder — causes state to bleed across the wrong items
- **Unbounded re-renders**: State set directly in the render body, or an effect whose own state update re-triggers itself
- **Memory leaks**: Event listeners, timers, subscriptions, or observers registered without a corresponding cleanup/teardown
- **Update-after-unmount**: State/store updates from an async callback that resolves after the component has unmounted, with no cleanup or abort
- **Defeated memoization**: Inline function/object/array literals passed as props to a memoized child, forcing it to re-render every time anyway
- **Reactivity-rule violations**: Reactive primitives invoked conditionally, in loops, or after an early return, when the framework requires a stable call/registration order
- **Controlled/uncontrolled input flip**: An input's bound value switches between `undefined`/unset and a defined value across renders
- **Duplicate fetch/subscription**: An effect re-running (e.g. double-invoke in strict/dev mode, or a changed dependency) fires the same fetch or subscription again without guarding against duplicates
