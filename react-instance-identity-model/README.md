# React's Instance Identity Model

- [React's Instance Identity Model](#reacts-instance-identity-model)
  - [Introduction](#introduction)
  - [Source Code](#source-code)
  - [Why Instance Identity Matters](#why-instance-identity-matters)
    - [Inexact Declarations](#inexact-declarations)
    - [Fungibility](#fungibility)
  - [React's Instance Identity Model](#reacts-instance-identity-model-1)
    - [Static Structure](#static-structure)
    - [Dynamic Collection](#dynamic-collection)
      - [`useDynamicCollection`](#usedynamiccollection)
      - [`Removable`](#removable)
      - [`DynamicCollection`](#dynamiccollection)
      - [`KeyDynamicCollection`](#keydynamiccollection)
    - [Dynamic Wrapper](#dynamic-wrapper)
      - [Workaround](#workaround)
  - [Instance Identity Model and Reconciliation](#instance-identity-model-and-reconciliation)
  - [Conclusion](#conclusion)
  - [Footnotes](#footnotes)

## Introduction

React is a declarative, reactive framework for building user interfaces. The central abstraction is a **component**, which maps state to a user interface declaration. A user interface declaration is a tree composed of components and intrinsic elements (components built into the rendering environment). A component is a blueprint that React instantiates creating an **instance**. As state changes, an instance's user interface is updated. React must transform the current user interface to the updated user interface. In order to realize this transformation, React uses a representation of the current and updated user interfaces called the **virtual DOM** [^1]. React compares the current and updated virtual DOMs to determine a course of action. This reconciliation process raises an important question of identity. When should an instance in the current user interface correspond to an instance in the updated user interface? React answers this question by following a set of rules that collectively comprise its **instance identity model**. The instance identity model determines when instances are created and destroyed. This creation and destruction influences nearly every part of a reactive user interface. In this document, we will explore React's instance identity model in depth.

## Source Code

This document has an associated MIT licensed playground available [here](https://stackblitz.com/edit/react-instance-identity-model-playground?file=README.md).

This document generally excludes styles for clarity.

## Why Instance Identity Matters

### Inexact Declarations

In React, the declarations in our components elide many details. Let's take a look at a simple `Input` component:

```tsx
import { useState, type FunctionComponent } from "react";

export const Input: FunctionComponent = () => {
  const [value, setValue] = useState("");

  const onChange: React.ChangeEventHandler<HTMLInputElement> = (e) => {
    setValue(e.target.value);
  };

  return (
    <input
      value={value}
      onChange={onChange}
    />
  );
};
```

The virtual node created by our `Input` on initial render will be[^2]:

```ts
const inputVNode = {
  type: "input",
  key: null,
  props: {
    onChange,
    value: "",
  },
};
```

The declaration and corresponding virtual node leave out many crucial details:

- Is our input focused ?
- If focused, where's the cursor ?
- Is any of the text in our input selected ?

The primary reason these details are left out is **encapsulation**. The built-in `input` is a complex element that provides many features. But this complexity is encapsulated. We get focus, cursor position, and selection support without having to do anything, and more importantly without having to know anything. If our declaration needed to provide these details, we would also have to manage these details. In other words, we would completely break encapsulation.

### Fungibility

**Fungibility** is the property that something is exactly replaceable. The common example of something fungible is a dollar. If the dollar in your pocket is magically swapped with the dollar in my pocket, then we should be indifferent. Fungibility is rare. Most things in the world are not fungible. Even things that are fungible in theory lose fungibility in practice (e.g. each dollar has a serial number).

As discussed, our declarations are necessarily vague. And consequently, our instances are non-fungible because React doesn't have the data to create an exact replacement. This non-fungibility is the reason instance identity matters.

## React's Instance Identity Model

React's instance identity model is comprised of two rules:

1. An instance in the same position in the user interface as in the previous declaration is the same instance.
2. Identity within a single level can be explicated by providing a `key`.[^3]

We will now explore this model in depth in a series of examples.

### Static Structure

Let's walk through the rendering of our `Input` component:

```tsx
import { useState, type FunctionComponent } from "react";

export const Input: FunctionComponent = () => {
  const [value, setValue] = useState("");

  const onChange: React.ChangeEventHandler<HTMLInputElement> = (e) => {
    setValue(e.target.value);
  };

  return (
    <input
      value={value}
      onChange={onChange}
    />
  );
};
```

The initial render will produce the following virtual node:

```ts
const inputVNode = {
  type: "input",
  key: null,
  props: {
    onChange,
    value: "",
  },
};
```

As we type, our `Input` is rerendered [^4]. Let's look at our virtual node after we type in `"rerender"`:

```ts
const inputVNode = {
  type: "input",
  key: null,
  props: {
    onChange,
    value: "rerender",
  },
};
```

The value of our `input` is captured by the virtual node, but its focus and cursor state are not. Nevertheless, the updated user interface shows the correct value, focus, and cursor position. The `input` instance is the only thing rendered by our `Input`. Trivially, its position in the user interface is constant, and the first identity rule applies. Because the identity of our `input` persists across rerenders, its internal state is maintained.

The importance of identity preservation becomes conspicuous when it's disturbed. Let's make use of the second rule to dynamically change the identity of `input`.

```tsx
import { useState, type FunctionComponent } from "react";

export const KeyInput: FunctionComponent = () => {
  const [value, setValue] = useState("");

  const onChange: React.ChangeEventHandler<HTMLInputElement> = (e) => {
    setValue(e.target.value);
  };

  return (
    <input
      key={value === "rerender" ? 1 : 0}
      value={value}
      onChange={onChange}
    />
  );
};
```

`KeyInput` will provide an explicit `key` when the value is `"rerender"`. When we type in `"rerender"`, the virtual node rendered will be:

```ts
const keyInputVNode = {
  type: "input",
  key: 1,
  props: {
    onChange,
    value: "rerender",
  },
};
```

React looks for an `input` with `key` set to `1` in the previous render. However, because it does not exist, React creates a new `input` for us. Consequently, input focus and cursor position are lost.

### Dynamic Collection

Instance identity becomes prominent once our user interface is structurally dynamic. Let's consider a user interface that let's us dynamically add and remove `Input`s.

#### `useDynamicCollection`

The `useDynamicCollection` hook lets us manage a dynamic set of keys:

```ts
import { useRef, useState } from "react";
// createKey will produce an alphabetically sequential series of keys
import { createKeyFactory } from "../utils/createKeyFactory";

export type DynamicCollection = {
  keys: string[];
  add: () => void;
  remove: (removedKey: string) => void;
};

export const useDynamicCollection = (): DynamicCollection => {
  const [keys, setKeys] = useState<string[]>([]);
  const createKey = useRef(createKeyFactory()).current;

  const add = () => {
    setKeys((keys) => [...keys, createKey()]);
  };

  const remove = (removedKey: string) => {
    setKeys((keys) => keys.filter((key) => key !== removedKey));
  };

  return {
    keys,
    add,
    remove,
  };
};
```

#### `Removable`

The `Removable` component allows us to wrap other components in a removal user interface.

```tsx
import type { FunctionComponent } from "react";
import { Button } from "./Button";

export type RemovableProps = {
  onRemove: () => void;
  children: React.ReactNode;
};

export const Removable: FunctionComponent<RemovableProps> = ({
  onRemove,
  children,
}) => {
  return (
    <div>
      {children}
      <Button onClick={onRemove}>Remove</Button>
    </div>
  );
};
```

#### `DynamicCollection`

When tree structure is dynamic, tree position is insufficient to identify instances. The instance in a given position may be completely different on each render. By using `key`s we can help React identify instances as long as their movement is contained to a single level.

To more clearly demonstrate the importance of providing `key`s to dynamic children, we will first leave them out.

```tsx
import { type FunctionComponent } from "react";
import { useDynamicCollection } from "../hooks/useDynamicCollection";
import { Button } from "./Button";
import { Input } from "./Input";
import { Removable } from "./Removable";

export const DynamicCollection: FunctionComponent = () => {
  const { keys, add, remove } = useDynamicCollection();

  return (
    <div>
      <Button onClick={add}>Add</Button>

      {keys.map((key) => (
        <Removable
          onRemove={() => {
            remove(key);
          }}
        >
          <Input />
        </Removable>
      ))}
    </div>
  );
};
```

Initially, `DynamicCollection` renders:

```ts
const dynamicCollectionVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Button,
        key: null,
        props: {
          onClick: add,
          children: "Add",
        },
      },
      // empty removable `Input` array
      [],
    ],
  },
};
```

After we click the "Add" button for the first time, our `DynamicCollection` instance renders:

```ts
const dynamicCollectionVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Button,
        key: null,
        props: {
          onClick: add,
          children: "Add",
        },
      },
      [
        {
          type: Removable,
          key: null,
          props: {
            onRemove,
            children: {
              type: Input,
              key: null,
              props: {},
            },
          },
        },
      ],
    ],
  },
};
```

Let's say we type `"first"` into this input and then click the "Add" button again. Our `DynamicCollection` instance will render two removable `Input`s, and the virtual node will look like this:

```ts
const dynamicCollectionVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Button,
        key: null,
        props: {
          onClick: add,
          children: "Add",
        },
      },
      [
        {
          type: Removable,
          key: null,
          props: {
            onRemove,
            children: {
              type: Input,
              key: null,
              props: {},
            },
          },
        },
        {
          type: Removable,
          key: null,
          props: {
            onRemove,
            children: {
              type: Input,
              key: null,
              props: {},
            },
          },
        },
      ],
    ],
  },
};
```

At this point, our user interface seems to be working correctly. The state from our first `Input` is preserved (it reads `"first"`), and we also have a second removable `Input`. As long as we're just adding `Input`s, tree position is sufficient to identify our instances.

Let's now try to remove the first `Input` by hitting its associated "Remove" button. Unfortunately, when the user interface updates, the second `Input` is removed instead of the first. There was an identity crisis.

Taking a look at the virtual node after removal makes the problem apparent:

```ts
const dynamicCollectionVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Button,
        key: null,
        props: {
          onClick: add,
          children: "Add",
        },
      },
      [
        {
          type: Removable,
          key: null,
          props: {
            onRemove,
            children: {
              type: Input,
              key: null,
              props: {},
            },
          },
        },
      ],
    ],
  },
};
```

That's the same structure rendered after hitting the "Add" button for the first time. Thus, according to tree position, we wanted to preserve our first `Input`.

#### `KeyDynamicCollection`

The `DynamicCollection` bug is easy to fix, we just need to provide a key to each `Removable`:

```tsx
import { type FunctionComponent } from "react";
import { useDynamicCollection } from "../hooks/useDynamicCollection";
import { Button } from "./Button";
import { Input } from "./Input";
import { Removable } from "./Removable";

export const KeyDynamicCollection: FunctionComponent = () => {
  const { keys, add, remove } = useDynamicCollection();

  return (
    <div>
      <Button onClick={add}>Add</Button>

      {keys.map((key) => (
        <Removable
          key={key}
          onRemove={() => {
            remove(key);
          }}
        >
          <Input />
        </Removable>
      ))}
    </div>
  );
};
```

It's illustrative to go through the virtual nodes when we provide a `key`.

The initial render is exactly the same as with no key.

After hitting "Add" for this first time:

```ts
const keyDynamicCollectionVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Button,
        key: null,
        props: {
          onClick: add,
          children: "Add",
        },
      },
      [
        {
          type: Removable,
          key: "a",
          props: {
            onRemove,
            children: {
              type: Input,
              key: null,
              props: {},
            },
          },
        },
      ],
    ],
  },
};
```

After hitting "Add" the second time:

```ts
const keyDynamicCollectionVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Button,
        key: null,
        props: {
          onClick: add,
          children: "Add",
        },
      },
      [
        {
          type: Removable,
          key: "a",
          props: {
            onRemove,
            children: {
              type: Input,
              key: null,
              props: {},
            },
          },
        },
        {
          type: Removable,
          key: "b",
          props: {
            onRemove,
            children: {
              type: Input,
              key: null,
              props: {},
            },
          },
        },
      ],
    ],
  },
};
```

After we hit the "Remove" button of our first `Removable`:

```ts
const keyDynamicCollectionVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Button,
        key: null,
        props: {
          onClick: add,
          children: "Add",
        },
      },
      [
        {
          type: Removable,
          // this tells React we want the second input to persist
          key: "b",
          props: {
            onRemove,
            children: {
              type: Input,
              key: null,
              props: {},
            },
          },
        },
      ],
    ],
  },
};
```

### Dynamic Wrapper

The primary limitation of React's instance identity model is that `key`s can only be used to distinguish instances within a single level of the user interface. In other words, if the parent of your instance changes, then React will always consider it to have a different identity regardless of `key`.

Let's say we have a `Frame` component, and want to provide a user interface to toggle framing an `Input`.

`DynamicWrapper` is a component that creates a dynamically framable `Input`:

```tsx
import { useState, type FunctionComponent } from "react";
import { Button } from "./Button";
import { Frame } from "./Frame";
import { Input } from "./Input";

export const DynamicWrapper: FunctionComponent = () => {
  const [shouldFrame, setShouldFrame] = useState(false);

  const toggleShouldFrame = () => {
    setShouldFrame((v) => !v);
  };

  return (
    <div>
      {shouldFrame ? (
        <Frame>
          <Input />
        </Frame>
      ) : (
        <Input />
      )}
      <Button onClick={toggleShouldFrame}>Frame</Button>
    </div>
  );
};
```

When we render `DynamicWrapper`, we will see that our `Input` is cleared whenever we hit the toggle. On each toggle, React destroys our current `Input` instance, and creates a new instance. React thinks the identity of our `Input` has changed because it's position in the user interface has changed.

Unfortunately, because this move spans more than a single level, `key` will not help us. The following code is equally broken:

```tsx
import { useState, type FunctionComponent } from "react";
import { Button } from "./Button";
import { Frame } from "./Frame";
import { Input } from "./Input";

export const DynamicWrapper: FunctionComponent = () => {
  const [shouldFrame, setShouldFrame] = useState(false);

  const toggleShouldFrame = () => {
    setShouldFrame((v) => !v);
  };

  return (
    <div>
      {shouldFrame ? (
        <Frame>
          <Input key="a" />
        </Frame>
      ) : (
        <Input key="a" />
      )}
      <Button onClick={toggleShouldFrame}>Frame</Button>
    </div>
  );
};
```

You might try to maintain tree stability by rewriting the component as follows:

```tsx
import { Fragment, useState, type FunctionComponent } from "react";
import { Button } from "./Button";
import { Frame } from "./Frame";
import { Input } from "./Input";

export const DynamicWrapper: FunctionComponent = () => {
  const [shouldFrame, setShouldFrame] = useState(false);

  const toggleShouldFrame = () => {
    setShouldFrame((v) => !v);
  };

  const Wrapper = shouldFrame ? Frame : Fragment;

  return (
    <div>
      <Wrapper>
        <Input />
      </Wrapper>

      <Button onClick={toggleShouldFrame}>Frame</Button>
    </div>
  );
};
```

This example is syntactically deceptive. It's seems unambiguous that we intend to render a single `Input` throughout the life of a `DynamicWrapper` instance. But React does not consider the syntax of our declaration, but rather considers the virtual nodes produced by our `DynamicWrapper` on render.

When framing is off, `DynamicWrapper` renders:

```ts
const dynamicWrapperVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Fragment,
        key: null,
        props: {
          children: {
            type: Input,
            key: null,
            props: {},
          },
        },
      },
      {
        type: Button,
        key: null,
        props: {
          onClick,
          children: "Frame",
        },
      },
    ],
  },
};
```

When framing is on, `DynamicWrapper` renders:

```ts
const dynamicWrapperVNode = {
  type: "div",
  key: null,
  props: {
    children: [
      {
        type: Frame,
        key: null,
        props: {
          children: {
            type: Input,
            key: null,
            props: {},
          },
        },
      },
      {
        type: Button,
        key: null,
        props: {
          onClick,
          children: "Frame",
        },
      },
    ],
  },
};
```

When React sees that the `type` switches from `Fragment` to `Frame`, it destroys everything else in the tree without further examination.

#### Workaround

There is no direct way to preserve the identity of our `Input`.

Since `Frame` is an internal component, we could try to modify its implementation to produce the same effect while maintaining tree structure. The original `Frame` component was implemented like this:

```tsx
import type { FunctionComponent } from "react";

export type FrameProps = {
  children: React.ReactNode;
};

export const Frame: FunctionComponent<FrameProps> = ({ children }) => {
  return (
    <div className="flex p-3 border-4 rounded-md border-yellow-500">
      {children}
    </div>
  );
};
```

We could change `Frame`'s implementation to the following:

```tsx
import type { FunctionComponent } from "react";

export type FrameProps = {
  shouldFrame: boolean;
  children: React.ReactNode;
};

export const Frame: FunctionComponent<FrameProps> = ({
  shouldFrame,
  children,
}) => {
  return (
    <div
      className={
        shouldFrame ? "flex p-3 border-4 rounded-md border-yellow-500" : ""
      }
    >
      {children}
    </div>
  );
};
```

And then we could rewrite `DynamicWrapper` as follows:

```tsx
import { useState, type FunctionComponent } from "react";
import { Button } from "./Button";
import { Frame } from "./Frame";
import { Input } from "./Input";

export const DynamicWrapper: FunctionComponent = () => {
  const [shouldFrame, setShouldFrame] = useState(false);

  const toggleShouldFrame = () => {
    setShouldFrame((v) => !v);
  };

  return (
    <div>
      <Frame shouldFrame={shouldFrame}>
        <Input />
      </Frame>

      <Button onClick={toggleShouldFrame}>Frame</Button>
    </div>
  );
};
```

This would work correctly because the tree structure remains identical regardless of frame state. However, this little hack will only work sometimes. In the common case, we cannot easily achieve the desired effect while maintaining the same tree structure. Further, when we're working with third-party components we're out of luck.

The commonly employed workaround for such identity issues is **state-lifting**. With state-lifting, you move your state higher in your user interface. You move your state up until you find somewhere in your tree that has a stable identity for at least the intended lifetime of your state.

Let's look at how we can make use of state-lifting in the frame example.

First we create a `FungibleInput` component:

```tsx
import type { FunctionComponent } from "react";

export type FungibleInputProps = {
  value: string;
  onChange: (value: string) => void;
};

export const FungibleInput: FunctionComponent<FungibleInputProps> = ({
  value,
  onChange,
}) => {
  const handleChange: React.ChangeEventHandler<HTMLInputElement> = (e) => {
    onChange(e.target.value);
  };

  return (
    <input
      value={value}
      onChange={handleChange}
    />
  );
};
```

Then we make use of it in `LiftDynamicWrapper`:

```tsx
import { useState, type FunctionComponent } from "react";
import { Button } from "./Button";
import { Frame } from "./Frame";
import { FungibleInput } from "./FungibleInput";

export const LiftDynamicWrapper: FunctionComponent = () => {
  const [value, setValue] = useState("");
  const [shouldFrame, setShouldFrame] = useState(false);

  const toggleShouldFrame = () => {
    setShouldFrame((v) => !v);
  };

  return (
    <div>
      {shouldFrame ? (
        <Frame>
          <FungibleInput
            value={value}
            onChange={setValue}
          />
        </Frame>
      ) : (
        <FungibleInput
          value={value}
          onChange={setValue}
        />
      )}

      <Button onClick={toggleShouldFrame}>Frame</Button>
    </div>
  );
};
```

Our new `FungibleInput` component is stateless; it takes its `value` and `onChange` handler as props. The `LiftDynamicWrapper` component now manages `input` state, and passes the required props to `FungibleInput`.

This version maintains `value` as desired. However, like our `Input` instance in the previous example, our `FungibleInput` instance is replaced on each toggle. The difference here is that `FungibleInput`'s identity is meaningless. The replacement instance is identical in every perceptible way.

In this example, state was the only source of non-fungibility. But generally, any aspect of an instance that creates a perceptible difference between itself and its replacement must be managed for this technique to work. The consequences of lost state are often dramatic. However, other fungibility failures can be subtle. For example, failure to make effects idempotent[^5].

There's a way to understand state-lifting in terms of fungibility and encapsulation. In general, there's a tension between fungibility and encapsulation. If you know every possible thing about something, then you can create an exact copy. On the other hand, encapsulation hides information.

State-lifting is a way to tradeoff encapsulation for fungibility. By moving your state higher in the tree, you break encapsulation by exposing state management details. At the same time, you make the instance stateless. A stateless instance is more easily fungible.

In addition to breaking encapsulation, state-lifting has many other negative consequences: hinders composition, increases complexity, complicates debugging, and hurts performance.

## Instance Identity Model and Reconciliation

The instance identity model is often improperly distinguished from reconciliation. The instance identity model specifies when an instance's identity is preserved, and is part of React's public API. On the other hand, the algorithm React uses to reconcile the current user interface to the updated user interface is an implementation detail. Crucially, the reconciliation algorithm must respect the instance identity model.

## Conclusion

React's instance identity model impacts nearly every aspect of its programming model. The two rules that React uses to determine instance identity are easy to understand and have proved workable in practice. Nevertheless, they are inadequate for expressing common identity requirements. The workarounds for this inadequacy such as state-lifting have serious problems. There is a clear need for a more general instance identity model.

## Footnotes

[^1]: Virtual DOM is the popular name. A more general name is virtual node. Browsers represent a UI as a [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction) tree. Virtual nodes are commonly used to virtualize the DOM. Hence, virtual DOM.
[^2]: In practice, a virtual node contains various internal properties. These are not relevant to the discussion, and elided for clarity.
[^3]: This is almost the same as saying that siblings can be distinguished by providing a key. However, instances at the same level need not coexist.
[^4]: React does not guarantee that it will render a new user interface on each state update. React only guarantees that it will render a new user interface after state updates asynchronously. Thus, many state updates may be batched into a single paint, and the intermediate states may never materialize. But the core conceptual model is still state/view correspondence.
[^5]: In React, your effects should always be idempotent. A dependency array even with perfect referential integrity is incapable of ensuring accurate effect triggering.
