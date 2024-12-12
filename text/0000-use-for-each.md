- Start Date: 2024-12-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

The `useForEach()` hook provides a sane mechanism for calling React Hooks inside loops.

# Basic example

The `useForEach(keys, callback)` hook calls `callback` once for each

```ts
import { useForEach } from "react";

const results = useForEach(keys, (key) => {
  const [conn, setConn] = useState(null);
});
```

# Motivation

React hooks deal with a single piece of data.

- `useEffect` synchronizes a single external resource with React.
- `useMemo` and `useCallback` memoize a single value.

---

Two ubiquitous concepts we encounter in almost every React application are: [lifting state up](https://react.dev/learn/sharing-state-between-components), and [synchronizing with effects](https://react.dev/learn/synchronizing-with-effects).
But using both concepts _at the same time_ can be surprisingly difficult.

## ChatRoom example

This code snippet is taken from the [Lifecycle of React Effects](https://react.dev/learn/lifecycle-of-reactive-effects) chapter, slightly altered with a `useState()` hook to give the component access to the connection object.

```tsx
function ChatRoom({ roomId }) {
  const connection = useSingleConnection(roomId);
  // ...
}

function useSingleConnection(roomId) {
  const [conn, setConn] = useState(null);
  useEffect(() => {
    const connection = createConnection(roomId);
    setConn(connection);
    return () => {
      connection.disconnect();
      setConn(null);
    };
  }, [roomId]);
  return conn;
}
```

Now, imagine we want to extend our chat app, and support connecting to multiple chat rooms at the same time.
We want to render a [tabs](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/) component where each chat room is displayed with a tab.

```tsx
function ChatApp({ rooms }) {
  const [activeTab, setActiveTab] = useState(0);
  return (
    <Tabs
      labels={rooms.map((room) => room.name)}
      activeTab={activeTab}
      onTabChange={(i) => setActiveTab(i)}
    >
      <ChatRoom roomId={rooms[activeTab].id} />
    </Tabs>
  );
}
```

Finally, if there are unread messages in a chatroom, we want to show a badge on that tab.
Assuming we can read this information from the connection object, then we need to lift the state up – the connections must be moved from the `ChatRoom` to the `ChatApp`.

Ideally, we could just use a loop in `ChatApp` to iterate over `rooms`, like this:

```tsx
function ChatApp({ rooms }) {
  const connections = rooms.map((room) => useSingleConnection(room.id));
  // ...
}
```

However, this is forbidden by the [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks).
Instead, we have to move the loop inside the effect.

```tsx
function useMultipleConnections(roomIds) {
  const [conns, setConns] = useState([]);
  useEffect(() => {
    const connections = roomIds.map((roomId) => createConnection(roomId));
    setConns(connections);
    return () => {
      connections.forEach((connection) => connection.disconnect());
      setConns([]);
    };
  }, [roomIds]);
  return conns;
}
```

But this is still not perfect.
Whenever the user connects to a new chatroom, or even just reorders their rooms list, this code closes and re-opens all connections.
This can cause flashing UI elements, and child components possibly losing state or re-triggering their own effects.

To work around this issue, we either need to use some non-idiomatic React Ref trickery; or move the connection management out of React and into our own state management solution.
In any case, we lose the clean React semantics and lifecycle guarantees that `useEffect` provides.

## With `useForEach()`

Idiomatic React is all about _composition_.
Ideally, we want to compose the `useMultipleConnections()` hook from the existing `useSingleConnection()` hook.
The `useForEach()` hook is one possible API design to achieve this goal.

```tsx
function ChatApp({ rooms }) {
  const roomIds = rooms.map((room) => room.id);
  const connections = useForEach(roomIds, (roomId) =>
    useSingleConnection(room.id)
  );
  // ...
}
```

The hook can effectively be used to convert any hook that manages a single state, effect or resource, into a hook that manages an array of said state, effects or resources.

# Detailed design

## Reference

```tsx
import { type Key } from "react";

declare function useForEach<K extends Key, T>(
  keys: readonly K[],
  callback: (key: K) => T
): readonly T[];
```

Call `useForEach` at the top level of your component (or inside another `useForEach` callback) to loop through an array, and call Hooks inside the loop body.

### Parameters

- `keys`: The array controlling the
- `callback`:

### Returns

A frozen array containing the results from calling `callback` with all `keys`.

---

The `useForEach()` hook is called with two parameters, both mandatory:

1. The `keys` parameter is an array of unique strings and/or numbers.
2. The `callback` parameter is a function that accepts a single string or number, and produces an arbitrary value.

The `useForEach()` hook synchronously calls `callback` with each value in `keys`, and returns an array of all the callback return values.
The `keys` array may change over time, including the length, order, and values.
The `callback` function may call hooks, following the normal rules of hooks.
This

https://react.dev/learn/rendering-lists#rules-of-keys

## The `keys` array

The first parameter of the `useForEach()` hook is an array of keys.
These keys serve the same purpose as they do in JSX arrays:
React uses the keys to track an "instance" of `callback` over the lifetime of the containing component.

The `keys` array may change over time, including the length, order, and values.

## The `callback` function

The second parameter of the `useForEach()` hook is a callback function.

The `keys` array must not contain duplicates, as determined by `Object.is()`.
If any duplicates are found, `useForEach()` issues a console warning in development mode, in the same fashion as React warning about duplicate keys in JSX elements.

## Corner cases

- The `useForEach` hook does not catch errors.
  When `callback` throws an error, it will bubble up and terminate the current render.
  This follows the example established by the `useMemo` callback and `useState` initializer callback.

- Passing duplicate values inside the `keys` array triggers a [duplicate keys](https://github.com/facebook/react/blob/a4964987dc140526702e996223fe7ee293def8ac/packages/react-reconciler/src/ReactChildFiber.js#L1070-L1077) error.
  There are two possibilities how this error could be reported:

  1. The error is logged to `console.error` in development, and silently discarded in production.
     React tries to match loop iterations to hook state via the array index.
     If the `keys` array changes in a subsequent render and an array element cannot be matched to its previous, the loop "instances" of the duplicate keys can become "orphaned".
     This follows the example established by JSX keys.
     Appendix "React handling of duplicate keys in JSX" demonstrates this behaviour for JSX elements.
  2. The error is thrown, terminating the current render.
     There is no precedence in React for throwing an error on duplicate keys.

  While it is generally

- To determine key equality, the `useForEach` hook internally converts all elements in the `keys` array to strings.
  All of the following arrays will trigger a duplicate keys error:
  `["1", 1]`, `[{}, {}]`, `["null", null]`
  This follows the example established by JSX keys.

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## Sharing state between iterations

```tsx
function useMultipleConnections(roomIds) {
  const [connections, setConnections] = useState({});
  useForEach(roomIds, (roomId) => {
    useEffect(() => {
      const connection = createConnection(roomId);
      setConnections((prev) => ({ ...prev, [roomId]: connection }));
      return () => {
        connection.disconnect();
        setConn((prev) => {
          const { [roomId]: _, ...rest } = { ...prev };
          return rest;
        });
      };
    }, [roomId]);
  });
  return connections;
}
```

# Drawbacks

Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

Handle collections of state and/or effects outside of React, then synchronize them with React via `useEffect` or `useSyncExternalStore`.

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

The `uesForEach()` hook allows to

# How we teach this

The `useForEach()` hook is a continuation of two established concepts in React: keys and hook composition.

It is an advanced concept, and fits well as its own sub-page in the [Escape Hatches](https://react.dev/learn/escape-hatches) chapter.
, lifting state up, and synchronizing with effects.

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?

Alternative names:

- `useLoop()`, `useRepeat()`
- `useMap()`
- `useNestedHooks()`

# Appendices

All code listings were tested with React v19.0.

## Appendix A: React handling of duplicate keys in JSX

This appendix demonstrates how React handles duplicate keys in JSX arrays.
Before we look at the duplicate keys case, let's first examine a well-behaving program.

https://github.com/user-attachments/assets/eef30d1f-f54b-4eee-96c1-c239f258aaea

<details>
<summary>Listing 10-1: A well-behaving React app with unique keys.</summary>

```tsx
import { useEffect, useState } from "react";

const effectsRun = { Z: 0, A: 0, B: 0, C: 0 };
const cleanupsRun = { Z: 0, A: 0, B: 0, C: 0 };

export function App() {
  const [hidden, setHidden] = useState(false);
  const [n, setN] = useState(0);
  useEffect(() => {
    console.table({ effectsRun, cleanupsRun });
  });

  return (
    <>
      <button onClick={() => setHidden(!hidden)}>
        {hidden ? "show" : "hide"}
      </button>{" "}
      <button onClick={() => setN((n + 1) % 4)}>{"Z >> 1"}</button>
      {!hidden &&
        [
          <Child key="A" value="A" />,
          <Child key="B" value="B" />,
          <Child key="C" value="C" />,
        ].toSpliced(n, 0, <Child key="Z" value="Z" />)}
    </>
  );
}

function Child({ value }) {
  useEffect(() => {
    effectsRun[value]++;
    return () => {
      cleanupsRun[value]++;
    };
  });
  return <div>{value}</div>;
}
```

</details>

Listing 10-1 demonstrates how React operates under normal circumstances.

```diff
export function App() {
  const [hidden, setHidden] = useState(false);
  const [n, setN] = useState(0);
  useEffect(() => {
    console.table({ effectsRun, cleanupsRun });
  });

  return (
    <>
      <button onClick={() => setHidden(!hidden)}>
        {hidden ? "show" : "hide"}
      </button>{" "}
      <button onClick={() => setN((n + 1) % 4)}>{"Z >> 1"}</button>
      {!hidden &&
        [
          <Child key="A" value="A" />,
-         <Child key="B" value="B" />,
-         <Child key="C" value="C" />,
+         <Child key="A" value="B" />,
+         <Child key="A" value="C" />,
        ].toSpliced(n, 0, <Child key="Z" value="Z" />)}
    </>
  );
}
```

_Listing 10-2:_

This code renders four characters `z`, `a`, `b` and `c`, together with a number indicating when each character last got rendered.
Pressing the button moves the `z` character to the next position, wrapping around at the end.
When clicking the button five times, the expected order of characters is:

- `z1, a1, b1, c1`
- `a2, z2, b2, c2`
- `a3, b3, z3, c3`
- `a4, b4, c4, z4`
- `z5, a5, b5, c5`

However, because the characters `a`, `b`, `c` all have the same key, React is unable to track them across multiple renders.
This results in the following output after five clicks (and renders):

- `z1`, `a1`, `b1`, `c1`
- `a1`, `b1`, `a2`, `z2`, `b2`, `c2`
- `a1`, `b1`, `a3`, `b2`, `b3`, `z3`, `c3`
- `a1`, `b1`, `a4`, `b2`, `b4`, `c4`, `z4`
- `a1`, `b1`, `a4`, `b2`, `b4`, `z5`, `a5`, `b5`, `c5`

We can observe how React loses track of elements with duplicate keys.
Instead of updating the existing elements, some elements become orphaned, never receiving updates again.
To keep the total number of rendered elements at 4, React creates totally new elements instead.
Over time, orphaned elements pile up in the DOM.
