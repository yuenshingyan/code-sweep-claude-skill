# Frontend Logic & State

Check for logical fallacies and faulty reasoning in state/derivations — bugs that render or build fine but produce the wrong value or wrong screen:

- **Incorrect boolean/conditional logic**: De Morgan's-law mistakes, negation errors, operator precedence bugs in boolean chains
- **Stale closures**: Callbacks or reactive effects capturing outdated state/props from an earlier render/update cycle
- **Wrong reactive dependencies**: Missing dependencies in a reactive computation (an effect, memo, computed value, or watcher) causing stale reads, or extra dependencies causing thrashing
- **Duplicated derived state**: A value copied into local state instead of computed on render/update, able to drift out of sync with its source
- **Off-by-one errors**: List slicing, pagination offsets, or index math that's one short/long
- **Reference-equality bugs**: Comparing objects/arrays by reference instead of value, causing missed updates; loose vs. strict equality/type-coercion bugs
- **Async state races**: Out-of-order fetch/promise responses applied to state with no request-id or cancellation guard, so a stale response can overwrite a fresher one
- **Wrong truthy/falsy assumptions**: Treating `0`, `''`, or `NaN`/`None` as "no value" incorrectly, or vice versa
- **Direct state mutation**: Mutating an array/object in place instead of via the framework's setter/store API, silently missing re-renders
- **Lossy merges**: Spread/merge operations that drop nested fields the caller expected to survive
