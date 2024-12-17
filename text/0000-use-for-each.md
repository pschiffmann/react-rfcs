- Start Date: 2024-12-10
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

The `useForEach` Hook provides a sane mechanism for calling React Hooks inside loops.

# Basic example

The `useForEach(keys, callback)` Hook calls `callback` once for each element in the `keys` iterable.
The `callback` function is allowed to call Hooks, as if it was at the top level of the component.

```ts
import { useEffect, useForEach, useMemo, useState } from "react";

function MyComponent({ keys }) {
  const results = useForEach(keys, (key) => {
    const [state, setState] = useState(/* ... */);
    useEffect(/* ... */);
    return useMemo(/* ... */);
  });
  // ...
}
```

# Motivation

Once you have learned to think in React, synchronizing a _single_ external system with React is straight-forward:
Connect to the system inside an effect, disconnect from the system inside the cleanup of that same effect.
React guarantees that effects and cleanups are executed in a well-defined, predictable order, which makes it relatively easy to reason about race conditions and memory leaks.

Unfortunately, we can't carry over this mental model if we need to synchronize _a dynamic number_ of external systems.
The natural way to process multiple values is to iterate over them, but loops and Hooks don't compose:

1. Placing the `useEffect` call inside a `for ... of` loop is forbidden by the [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks).
2. Placing the `for ... of` loop inside the `useEffect` call will execute the cleanup function for _all_ elements whenever _any_ element changes.

Today, applications that need to connect to a dynamic number of external systems have no other choice than to use non-idiomatic workarounds.
This increases the risk of race conditions and memory leaks, makes the code harder to read, and causes code duplication if both single-connection and multi-connection Hooks are needed for the same external system.

## ChatRooms example

To give a specific example, we will look at a simple chat app.
The app allows users to connect to multiple chat rooms at the same time.
The UI renders one chat room at a time, and users can switch between all connected chat rooms via a [tabs](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/) component.
A badge over each tab tells the user if that chat room has unread messages.

<img width="600" alt="Screenshot of a chat app with 3 connected chat rooms, organized in tabs" src="https://github.com/user-attachments/assets/2ab0f007-9b1c-4ae6-952f-ae30882068a9" />

[live demo](https://pschiffmann.github.io/use-for-each-playground/chat-app-non-idiomatic.html) | [source code](https://github.com/pschiffmann/use-for-each-playground/blob/main/src/chat-app/main-non-idiomatic.tsx)

Connecting to a single chat room is pretty straight-forward, and is covered in great detail in the [Lifecycle of React Effects](https://react.dev/learn/lifecycle-of-reactive-effects) docs.
We can use the `ChatRoom` component from the docs as a starting point, and render one `ChatRoom` per tab.

```tsx
function ChatApp() {
  const [roomIds, setRoomIds] = useState(/* ... */);

  return (
    <Tabs
      tabs={roomIds.map((roomId) => ({
        key: roomId,
        label: roomId,
        badge: 0, // ???
        content: <ChatRoom roomId={roomId} />,
      }))}
    />
  );
}

function ChatRoom({ roomId }) {
  const connection = useSingleConnection(roomId);
  // ...
}

function useSingleConnection(roomId) {
  const [conn, setConn] = useState(null);
  useEffect(() => {
    const connection = new ChatRoomConnection(roomId);
    setConn(connection);
    return () => {
      connection.disconnect();
      setConn(null);
    };
  }, [roomId]);
  return conn;
}
```

While this implementation is a decent start, we need to fix two issues.
First, connections are closed and re-opened whenever we switch tabs (because the `Tabs` component mounts only the active tab).
Second, we can't render the "unread messages" badge count because the `ChatApp` component doesn't have access to the connection objects.

To address both issues, we need to lift the connection state up into `ChatApp`.
Ideally, we could reuse the `useSingleConnection` Hook.

```tsx
function ChatApp() {
  const [roomIds, setRoomIds] = useState(/* ... */);
  const connections = useMultipleConnections(roomIds);
  const unreadCounts = useUnreadCounts(connections);

  return (
    <Tabs
      tabs={roomIds.map((roomId, i) => ({
        key: roomId,
        label: roomId,
        badge: unreadCounts[i],
        content: <ChatRoom connection={connections[i]} />,
      }))}
    />
  );
}

function useMultipleConnections(roomIds) {
  return roomIds.map((roomId) => useSingleConnection(roomId));
}
```

Alas, we can't.
When we connect to another chat room by adding an element to the `roomIds` array, React throws this error:

> React has detected a change in the order of Hooks called by ChatApp.
> This will lead to bugs and errors if not fixed.
> For more information, read the Rules of Hooks: https://react.dev/link/rules-of-hooks

Instead, we have to move the `roomIds.map()` call inside the `useEffect`.

```tsx
function useMultipleConnections(roomIds) {
  const [connections, setConnections] = useState(() => []);
  useEffect(() => {
    const connections = roomIds.map((roomId) => new ChatRoomConnection(roomId));
    setConnections(connections);
    return () => {
      connections.forEach((connection) => connection.close());
    };
  }, [roomIds]);
  return connections;
}
```

With the new Hook implementation, we can make changes to the `roomIds` array without crashing the app.
But with every change to the array, we now close and re-open _all_ connections.
This results in flickering badges and chat content whenever the user connects to a new room, disconnects from a room, or merely moves tabs around.

https://github.com/user-attachments/assets/556a9036-6dcf-4424-883f-04b053ab6f82

[live demo](https://pschiffmann.github.io/use-for-each-playground/chat-app-naive.html) | [source code](https://github.com/pschiffmann/use-for-each-playground/blob/main/src/chat-app/main-naive.tsx)

To avoid closing all connections whenever `roomIds` changes, we need to put the "connect" and "disconnect" code into different effects, with different dependencies.
We also need to use a ref object to share the connections with the "disconnect" effect.

```tsx
function useMultipleConnectionsNonIdiomatic(roomIds) {
  const connectionsRef = useRef(new Map());
  const [connections, setConnections] = useState(() => new Map());

  useEffect(() => {
    let hasChanges = false;

    const connections = connectionsRef.current;
    // Open new connections.
    for (const roomId of roomIds) {
      if (connections.has(roomId)) continue;
      hasChanges = true;
      connections.set(roomId, new ChatRoomConnection(roomId));
    }

    // Close no longer used connections on updates.
    for (const [roomId, connection] of connections) {
      if (roomIds.includes(roomId)) continue;
      hasChanges = true;
      connections.delete(roomId);
      connection.close();
    }

    if (hasChanges) setConnections(new Map(connections));
  }, [roomIds]);

  // Close all connections on unmount with a effect cleanup.
  useEffect(
    () => () => {
      connectionsRef.current.forEach((connection) => connection.close());
      connectionsRef.current.clear();
    },
    []
  );

  return roomIds.map((roomId) => connections.get(roomId) ?? null);
}
```

https://github.com/user-attachments/assets/a472bc84-233c-4832-9706-b980056c552c

[live demo](https://pschiffmann.github.io/use-for-each-playground/chat-app-non-idiomatic.html) | [source code](https://github.com/pschiffmann/use-for-each-playground/blob/main/src/chat-app/main-non-idiomatic.tsx)

This works, but it's messy.
And it only gets worse if the effect grows and depends on more dependencies.

### `useEffect` with `useForEach()`

Idiomatic React is all about _composition_.
Ideally, we want to compose the `useMultipleConnections()` Hook from the existing `useSingleConnection()` Hook.
The `useForEach()` Hook lets us do just that.

```tsx
function useMultipleConnections(roomIds) {
  return useForEach(roomIds, (roomId) => {
    return useSingleConnection(roomId);
  });
}
```

The Hook can effectively be used to convert any Hook (native or userland) that manages a single state, effect or resource, into a Hook that manages an array of said state, effects or resources.

### `useId` with `useForEach`

Our chat app renders a dynamic number of tabs.
[WAI-ARIA](https://w3c.github.io/aria/#tabpanel) requires that both tabs and tab panels have an HTML `id` attribute.

> Authors SHOULD associate a tabpanel element with its tab, by using the aria-controls attribute on the tab to reference the tab panel, and/or by using the aria-labelledby attribute on the tab panel to reference the tab.

Therefore, we need 2×`roomIds.length` unique HTML ids.
Today, we can generate a single id prefix with `useId`, use the room ids as suffixes, and hope that a simple string concatenation results in a valid HTML id.

With `useForEach`, we could instead generate an arbitrary number of ids that are guaranteed to be valid.

```tsx
const ids = useForEach(roomIds, () => useId());
```

### `useSyncExternalStore` with `useForEach`

One implementation detail of `ChatApp` that I skipped over earlier is `useUnreadCounts(connections)`.
The code listing above uses this hook to read the `ChatRoomConnection#unreadCount` properties from all open connections.
Without the `useForEach` hook, this userland hook is surprisingly difficult to implement – at least, if we want to keep the results array stable until one of its elements changes.

The `useForEach` would make this easy to implement.

```tsx
function useUnreadCounts(connections) {
  return useForEach(connections, (connection) => {
    const subscribe = useCallback(
      (onStoreChange) => {
        const unsubscribe = connection?.subscribe("unreadCount", onStoreChange);
        return () => unsubscribe?.();
      },
      [connection]
    );
    const getSnapshot = useCallback(
      () => connection?.readyState ?? 0,
      [connection]
    );
    return useSyncExternalStore(subscribe, getSnapshot);
  });
}
```

# Detailed design

## Reference

```tsx
import { type Key } from "react";

declare function useForEach<K extends Key, T>(
  keys: Iterable<K>,
  callback: (key: K) => T
): readonly T[];
```

### Parameters

- `keys`: The iterable on which the loop operates.
  It should contain only strings and/or numbers, and should not contain duplicates.

  The iterable should be a dynamic value and come from e.g. props or another Hook call.
  This is not a dependency array like for the `useMemo` or `useEffect` Hooks, and should not be an array literal.

- `callback`: The function that is executed for each element in `keys`.
  It should be pure, should take a single `key` argument, and may return a value of any type.
  It may call other React Hooks.

  Hooks that are called inside `callback` use the passed-in `key` to track their state across multiple renders.
  For example, a `useState` Hook will always return the state for the same key, even if that key moves to different indexes in the `keys` iterable over multiple renders.
  Likewise, a `useEffect` Hook will compare the current dependencies with the previous dependencies of the same key to determine whether to execute again.

  If `keys` contains a new key that wasn't present in the previous render, then the Hooks for that key will be newly initialized, like it normally happens during the first render of a component.
  For example, `useMemo` will call its `calculateValue` callback, because there are no previous dependencies to compare yet.

  If `keys` doesn't contain a key that was present in the previous render, then the Hooks associated with that key are "unmounted".
  Effect Hooks like `useEffect` and `useSyncExternalStore` execute their cleanup; stateful Hooks like `useState`, `useMemo` and `useRef` drop all references to their values.
  When that same key appears again in a subsequent render, then it gets newly initialized again.

### Returns

A frozen array containing the results from calling `callback` with all `keys`.

The order of values inside the results array matches the order of `keys`.
For example, if `keys` is `[1, 2, 3]` during one render and `[2, 1, 3]` during the next, then the first results array will be `[callback(1), callback(2), callback(3)]`, and the second will be `[callback(2), callback(1), callback(3)]`.

React will return the same array during consecutive renders if the number of keys hasn't changed, and each index in the results array contains the same value as during the previous render (as determined by `Object.is`).
To prevent inadvertent mutations that would leak into consecutive renders, the results array is [frozen](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze).

## Usage

### Executing an effect for each element in an array

To execute an effect for each element in an array, wrap the `useEffect` call with `useForEach`.

```tsx
function ChatApp({ roomIds }) {
  useForEach(roomIds, (roomId) => {
    useEffect(() => {
      const connection = createConnection(roomId);
      return () => {
        connection.close();
      };
    }, [roomId]);
  });
}
```

You can think of this code as being equivalent to this:

```tsx
function ChatApp({ roomIds }) {
  for (const roomId of roomIds) {
    useEffect(() => {
      const connection = createConnection(roomId);
      return () => {
        connection.close();
      };
    }, [roomId]);
  }
}
```

The second code listing (with the `for ... of` loop) is not valid React code, because it violates the [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks).
The first code listing (with the `useForEach` Hook) is valid React code, follows the Rules of Hooks, and achieves the same goal.

### Associating state with keys

If you need to store some state for each of your keys, you have two options.

1. You can store the state for each key inside a separate state Hook, like this:

   ```tsx
   function ChatApp({ roomIds }) {
     const connections = useForEach(roomIds, (roomId) => {
       const connections = useEffect(() => {
         const [conn, setConn] = useState(null); // <- State variable that stores a single connection object.
         const connection = createConnection(roomId);
         setConn(connection); // <- Write to the local state variable.
         return () => {
           connection.close();
           setConn(null);
         };
       }, [roomId]);
       return conn; // <- Pass the local variable to the parent scope.
     });
   }
   ```

   Call `useState` inside the `useForEach` callback to create a local state variable.
   Write to that state within the effect, and return the state from the callback function.
   `useForEach` returns an array of all the callback results.

   To find the connection corresponding to a specific key, check the results array at that same index:

   ```tsx
   for (let i = 0; i < roomIds.length; i++) {
     const roomId = roomIds[i];
     const connection = connections[i];
   }
   ```

2. Alternatively, you can store all values in a single state variable, inside a [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) or object.

   ```tsx
   function ChatApp({ roomIds }) {
     const [connections, setConnections] = useState({}); // <- State variable that stores all connections.
     useForEach(roomIds, (roomId) => {
       const connections = useEffect(() => {
         const connection = createConnection(roomId);
         setConnections((prev) => ({ ...prev, [roomId]: connection })); // <- Write to the component-level state.
         return () => {
           connection.close();
           setConnections(([roomId]: _, ...rest) => rest);
         };
       }, [roomId]);
       // No `return` needed here.
     });
   }
   ```

   To find the connection corresponding to a specific key, index the map by that key:

   ```tsx
   for (const roomId of roomIds) {
     const connection = connections[roomId];
   }
   ```

Option 1 gives you an array of values, where the element order is guaranteed to match the `keys` order of the current render.
Option 2 gives you a map from key to value, but the iteration order of the map may get out of sync with the `keys` iteration order over time.
Which option is better depends on your use case.

## Corner cases

### Exception handling

The `useForEach` Hook does not catch errors.
When `callback` throws an error, it will bubble up and terminate the current render.
This follows the example established by the `useMemo` callback and `useState` initializer callback.

### Duplicate keys

Passing a `keys` iterable that contains duplicate values triggers a [duplicate keys](https://github.com/facebook/react/blob/a4964987dc140526702e996223fe7ee293def8ac/packages/react-reconciler/src/ReactChildFiber.js#L1070-L1077) error.
The error is logged to `console.error` in development, and silently discarded in production.

For duplicate keys, React tries to match loop `callback` calls to Hook state via the iteration index.
If matching based on the index fails, the loop "instances" of the duplicate keys and the associated Hooks become "orphaned".
Orphaned state Hooks can be garbage collected because they can never be read again, and orphaned effect Hooks will never execute their cleanup function.
See [Appendix A: React handling of duplicate keys in JSX](#appendix-a-react-handling-of-duplicate-keys-in-jsx) for a demonstration of this behaviour for JSX elements.

This follows the example established by JSX keys.

### Key type coercion

To determine key equality, the `useForEach` Hook internally converts all elements in the `keys` array to strings.
All of the following arrays will trigger a duplicate keys error:
`["1", 1]`, `[{}, {}]`, `["null", null]`
This follows the example established by JSX keys.

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

Foot gun - this Hook is powerful, and as such requires a certain level of care and understanding to use properly.
Improper use can cause problems like:

- performance issues due to excessive Hook calls
- memory leaks

The issue is even greater here than with JSX arrays because this Hook will probably be used almost exclusively for effects.
Maybe this complexity should not be made more accessible, and should be left to experienced engineers who build solutions outside of React.

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

The `uesForEach()` Hook allows to

# How we teach this

The `useForEach()` Hook is a continuation of two established concepts in React: keys and Hook composition.

It is an advanced concept, and fits well as its own sub-page in the [Escape Hatches](https://react.dev/learn/escape-hatches) chapter.
, lifting state up, and synchronizing with effects.

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

# Unresolved questions

- Implementation cost, both in term of code size and complexity.
- Is this Hook compatible/composable with all other Hooks?
  I have never used `useActionState`, `useDeferredValue`, `useOptimistic`, and `useTransition`.
- What is a good name for this Hook?
  List of ideas:
  - `useForEach` (from [`Array.forEach`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach))
  - `useMap` (from [`Array.map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map))
  - `useLoop`, `useRepeat`
  - `useNestedHooks`

# Appendices

All code listings were tested with React v19.0 in production mode, inside `<StrictMode>`.
You can run the code locally, [here](https://github.com/pschiffmann/use-for-each-playground) is the repository.

## Appendix A: React handling of duplicate keys in JSX

This appendix demonstrates how React handles duplicate keys in JSX arrays.
Before we look at the duplicate keys case, we first examine a well-behaving program.

https://github.com/user-attachments/assets/eef30d1f-f54b-4eee-96c1-c239f258aaea

[live demo](https://pschiffmann.github.io/use-for-each-playground/appendix-a-unique-keys.html)

<details>
<summary>Listing 10-1: A well-behaving React app with unique keys.</summary>

```tsx
import { useEffect, useState } from "react";

const effectsRun = { Z: 0, A: 0, B: 0, C: 0 };
const cleanupsRun = { Z: 0, A: 0, B: 0, C: 0 };

export function App() {
  const [hidden, setHidden] = useState(true);
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

The "show/hide" button mounts/unmounts a list of four children `A`, `B`, `C`, `Z`.
The "Z >> 1" button moves the `Z` child to the next position in the list, wrapping around after the last position.

The children run an effect after every render, and a cleanup before their next render (or unmount).
The devtools table displays how often each effect and cleanup function is run.
We can see that effects run once per render, and all effects are cleaned up again.

To observe how React handles duplicate keys, we give the children `A`, `B` and `C` the same key.

```diff
      {!hidden &&
        [
          <Child key="A" value="A" />,
-         <Child key="B" value="B" />,
-         <Child key="C" value="C" />,
+         <Child key="A" value="B" />,
+         <Child key="A" value="C" />,
        ].toSpliced(n, 0, <Child key="Z" value="Z" />)}
```

https://github.com/user-attachments/assets/7636308c-ae68-4d87-b16f-326904b79a65

[live demo](https://pschiffmann.github.io/use-for-each-playground/appendix-a-duplicate-keys.html)

<details>
<summary>Listing 10-2: A React app with glitches caused by duplicate keys.</summary>

```tsx
import { useEffect, useState } from "react";

const effectsRun = { Z: 0, A: 0, B: 0, C: 0 };
const cleanupsRun = { Z: 0, A: 0, B: 0, C: 0 };

export function App() {
  const [hidden, setHidden] = useState(true);
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
          <Child key="A" value="B" />,
          <Child key="A" value="C" />,
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

We can see that React doesn't properly unmount all elements, and also doesn't run all effect cleanup callbacks.
In a real application, this can lead to UI glitches with duplicated UI elements, and possibly memory leaks due to external resources that are allocated but never cleaned up.

## Appendix B: Managing an array of external resources with `useEffect`

## Appendix C: `useSyncExternalStore` to access an array of external states
