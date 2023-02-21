# Vue's Instance Identity Model

- [Vue's Instance Identity Model](#vues-instance-identity-model)
  - [Introduction](#introduction)
  - [Source Code](#source-code)
  - [Why Instance Identity Matters](#why-instance-identity-matters)
    - [Inexact Declarations](#inexact-declarations)
    - [Fungibility](#fungibility)
  - [Vue's Instance Identity Model](#vues-instance-identity-model-1)
    - [Static Structure](#static-structure)
    - [Dynamic Collection](#dynamic-collection)
      - [`useDynamicCollection`](#usedynamiccollection)
      - [`BaseRemovable`](#baseremovable)
      - [`DynamicCollection`](#dynamiccollection)
      - [`KeyDynamicCollection`](#keydynamiccollection)
    - [Dynamic Wrapper](#dynamic-wrapper)
      - [Workaround](#workaround)
  - [Instance Identity Model and Patching](#instance-identity-model-and-patching)
  - [Conclusion](#conclusion)
  - [Footnotes](#footnotes)

## Introduction

Vue is a declarative, reactive framework for building user interfaces. The central abstraction is a **component**, which maps state to a user interface declaration. A user interface declaration is a tree composed of components and intrinsic elements (components built into the rendering environment). A component is a blueprint that Vue instantiates creating an **instance**. As state changes, an instance's user interface is updated. Vue must transform the current user interface to the updated user interface. In order to realize this transformation, Vue uses a representation of the current and updated user interfaces called the **virtual DOM** [^1]. Vue compares the current and updated virtual DOMs to determine a course of action. This patch process raises an important question of identity. When should an instance in the current user interface correspond to an instance in the updated user interface? Vue answers this question by following a set of rules that collectively comprise its **instance identity model**. The instance identity model determines when instances are created and destroyed. This creation and destruction influences nearly every part of a reactive user interface. In this document, we will explore Vue's instance identity model in depth.

## Source Code

This document has an associated MIT licensed playground available [here](https://stackblitz.com/edit/vue-instance-identity-model-playground?file=README.md).

All components are SFCs (Single File Components). The SFC complier creates inline handlers. In places where such handlers are used, we will sometimes substitute a descriptive variable name for the function. This makes the examples easier to understand when handler definitions distract from the discussion.

This document generally excludes styles for clarity.

## Why Instance Identity Matters

### Inexact Declarations

In Vue, the declarations in our components elide many details. Let's take a look at a simple `BaseInput` component:

```vue
<script setup lang="ts">
import { ref } from "vue";

const value = ref("");
</script>

<template>
  <input v-model="value" />
</template>
```

The virtual node created by our `BaseInput` on initial render will be[^2]:

```ts
const inputVNode = {
  type: "input",
  key: null,
  props: {
    // valueUpdateHandler is really defined inline by the SFC compiler
    // we use a phantom variable (`valueUpdateHandler`) as a shorthand
    "onUpdate:modelValue": valueUpdateHandler,
  },
  // dirs is an array of `DirectiveBindings`
  // A directive binding is an object that contains the directive itself along
  // with some additional properties needed to apply the directive such as its
  // modifiers
  dirs: [
    {
      // a reference to our `BaseInput` instance (not the input element instance)
      // `instance` is included here for completeness, but is not central to our discussion
      instance,
      // vModelText is the built-in directive for two-way binding text input
      dir: vModelText,
      value: "",
      oldValue: undefined,
    },
  ],
};
```

The declaration and corresponding virtual node leave out many crucial details:

- Is our input focused ?
- If focused, where's the cursor ?
- Is any of the text in our input selected ?

The primary reason these details are left out is **encapsulation**. The built-in `input` is a complex element that provides many features. But this complexity is encapsulated. We get focus, cursor position, and selection support without having to do anything, and more importantly without having to know anything. If our declaration needed to provide these details, we would also have to manage these details. In other words, we would completely break encapsulation.

### Fungibility

**Fungibility** is the property that something is exactly replaceable. The common example of something fungible is a dollar. If the dollar in your pocket is magically swapped with the dollar in my pocket, then we should be indifferent. Fungibility is rare. Most things in the world are not fungible. Even things that are fungible in theory lose fungibility in practice (e.g. each dollar has a serial number).

As discussed, our declarations are necessarily vague. And consequently, our instances are non-fungible because Vue doesn't have the data to create an exact replacement. This non-fungibility is the reason instance identity matters.

## Vue's Instance Identity Model

Vue's instance identity model is comprised of three rules:

1. An instance in the same position in the user interface as in the previous declaration is the same instance.
2. Identity within a single level can be explicated by providing a `key`.[^3]
3. Many instances can simultaneously exist in the same place in the user interface, but only one can be `activated` at any point in time. This behavior is opted into through the use of [`KeepAlive`](https://vuejs.org/guide/built-ins/keep-alive.html).

We will now explore this model in depth in a series of examples.

### Static Structure

Let's walk through the rendering of our `BaseInput` component:

```vue
<script setup lang="ts">
import { ref } from "vue";

const value = ref("");
</script>

<template>
  <input v-model="value" />
</template>
```

The initial render will produce the following virtual node:

```ts
const inputVNode = {
  type: "input",
  key: null,
  props: {
    "onUpdate:modelValue": valueUpdateHandler,
  },
  dirs: [
    {
      instance,
      dir: vModelText,
      value: "",
      oldValue: undefined,
    },
  ],
};
```

As we type, our `BaseInput` is rerendered [^4]. Let's look at our virtual node after we type in `"rerender"`:

```ts
const inputVNode = {
  type: "input",
  key: null,
  props: {
    "onUpdate:modelValue": valueUpdateHandler,
  },
  dirs: [
    {
      instance,
      dir: vModelText,
      value: "rerender",
      oldValue: "rerende",
    },
  ],
};
```

The value of our `input` is captured by the virtual node in the directive binding, but its focus and cursor state are not. Nevertheless, the updated user interface shows the correct value, focus, and cursor position. The `input` instance is the only thing rendered by our `BaseInput`. Trivially, its position in the user interface is constant, and the first identity rule applies. Because the identity of our `input` persists across rerenders, its internal state is maintained.

The importance of identity preservation becomes conspicuous when it's disturbed. Let's make use of the second rule to dynamically change the identity of `input`.

```tsx
<script setup lang="ts">
import { ref } from "vue";

const value = ref("");
</script>

<template>
  <input
    :key="value === 'rerender' ? 1 : 0"
    v-model="value"
  />
</template>

```

`BaseKeyInput` will provide an explicit `key` when the value is `"rerender"`. When we type in `"rerender"`, the virtual node rendered will be:

```ts
const keyInputVNode = {
  type: "input",
  key: 1,
  // The top-level `key` property is set based on the `key` provided in props
  // However, the `key` is left in props.
  props: {
    key: 1,
    "onUpdate:modelValue": valueUpdateHandler,
  },
  dirs: [
    {
      instance,
      dir: vModelText,
      value: "rerender",
      oldValue: "rerende",
    },
  ],
};
```

Vue looks for an `input` with `key` set to `1` in the previous render. However, because it does not exist, Vue creates a new `input` for us. Consequently, input focus and cursor position are lost.

### Dynamic Collection

Instance identity becomes prominent once our user interface is structurally dynamic. Let's consider a user interface that let's us dynamically add and remove `BaseInput`s.

#### `useDynamicCollection`

The `useDynamicCollection` composable lets us manage a dynamic set of keys:

```ts
import { ref, type Ref } from "vue";
// createKey will produce an alphabetically sequential series of keys
import { createKeyFactory } from "../utils/createKeyFactory";

export type DynamicCollection = {
  keys: Ref<string[]>;
  add: () => void;
  remove: (removedKey: string) => void;
};

export const useDynamicCollection = (): DynamicCollection => {
  const keys = ref<string[]>([]);
  const createKey = createKeyFactory();

  const add = () => {
    keys.value = [...keys.value, createKey()];
  };

  const remove = (removedKey: string) => {
    keys.value = keys.value.filter((key) => key !== removedKey);
  };

  return {
    keys,
    add,
    remove,
  };
};
```

#### `BaseRemovable`

The `BaseRemovable` component allows us to wrap other components in a removal user interface.

```vue
<script setup lang="ts">
import BaseButton from "./BaseButton.vue";

const emit = defineEmits<{
  (e: "remove"): void;
}>();
</script>

<template>
  <div>
    <slot />
    <BaseButton @click="emit('remove')">Remove</BaseButton>
  </div>
</template>
```

#### `DynamicCollection`

When tree structure is dynamic, tree position is insufficient to identify instances. The instance in a given position may be completely different on each render. By using `key`s we can help Vue identify instances as long as their movement is contained to a single level.

To more clearly demonstrate the importance of providing `key`s to dynamic children, we will first leave them out.

```vue
<script setup lang="ts">
import { useDynamicCollection } from "../composables/useDynamicCollection";
import BaseButton from "./BaseButton.vue";
import BaseInput from "./BaseInput.vue";
import BaseRemovable from "./BaseRemovable.vue";

const { keys, add, remove } = useDynamicCollection();
</script>

<template>
  <div>
    <BaseButton @click="add">Add</BaseButton>
    <BaseRemovable
      v-for="key in keys"
      @remove="remove(key)"
    >
      <BaseInput />
    </BaseRemovable>
  </div>
</template>
```

Initially, `DynamicCollection` renders:

```ts
const dynamicCollectionVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseButton,
      key: null,
      props: {
        onClick: add,
      },
      children: {
        // Slots are described as a map of slot name to slot function.
        //
        // `withCtx` is an internal compiler function that wraps a slot
        // function. It's included for completeness, but is not relevant to our
        // discussion.
        //
        // `createTextNode` creates a virtual node corresponding to a text node.
        default: withCtx(() => [createTextNode("Add")]),
      },
    },
    {
      type: Fragment,
      key: null,
      // empty removable `BaseInput` array
      children: [],
    },
  ],
};
```

After we click the "Add" button for the first time, our `DynamicCollection` instance renders:

```ts
const dynamicCollectionVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseButton,
      key: null,
      props: {
        onClick: add,
      },
      children: {
        default: withCtx(() => [createTextNode("Add")]),
      },
    },
    {
      type: Fragment,
      key: null,
      children: [
        {
          type: BaseRemovable,
          key: null,
          props: {
            onRemove,
          },
          children: {
            default: withCtx(() => [createVNode(BaseInput)]),
          },
        },
      ],
    },
  ],
};
```

Let's say we type `"first"` into this input and then click the "Add" button again. Our `DynamicCollection` instance will render two removable `BaseInput`s, and the virtual node will look like this:

```ts
const dynamicCollectionVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseButton,
      key: null,
      props: {
        onClick: add,
      },
      children: {
        default: withCtx(() => [createTextNode("Add")]),
      },
    },
    {
      type: Fragment,
      key: null,
      children: [
        {
          type: BaseRemovable,
          key: null,
          props: {
            onRemove,
          },
          children: {
            default: withCtx(() => [createVNode(BaseInput)]),
          },
        },
        {
          type: BaseRemovable,
          key: null,
          props: {
            onRemove,
          },
          children: {
            default: withCtx(() => [createVNode(BaseInput)]),
          },
        },
      ],
    },
  ],
};
```

At this point, our user interface seems to be working correctly. The state from our first `BaseInput` is preserved (it reads `"first"`), and we also have a second removable `BaseInput`. As long as we're just adding `BaseInput`s, tree position is sufficient to identify our instances.

Let's now try to remove the first `BaseInput` by hitting its associated "Remove" button. Unfortunately, when the user interface updates, the second `Input` is removed instead of the first. There was an identity crisis.

Taking a look at the virtual node after removal makes the problem apparent:

```ts
const dynamicCollectionVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseButton,
      key: null,
      props: {
        onClick: add,
      },
      children: {
        default: withCtx(() => [createTextNode("Add")]),
      },
    },
    {
      type: Fragment,
      key: null,
      children: [
        {
          type: BaseRemovable,
          key: null,
          props: {
            onRemove,
          },
          children: {
            default: withCtx(() => [createVNode(BaseInput)]),
          },
        },
      ],
    },
  ],
};
```

That's the same structure rendered after hitting the "Add" button for the first time. Thus, according to tree position, we wanted to preserve our first `BaseInput`.

#### `KeyDynamicCollection`

The `DynamicCollection` bug is easy to fix, we just need to provide a key to each `BaseRemovable`:

```vue
<script setup lang="ts">
import { useDynamicCollection } from "../composables/useDynamicCollection";
import BaseButton from "./BaseButton.vue";
import BaseInput from "./BaseInput.vue";
import BaseRemovable from "./BaseRemovable.vue";

const { keys, add, remove } = useDynamicCollection();
</script>

<template>
  <div>
    <BaseButton @click="add">Add</BaseButton>

    <BaseRemovable
      v-for="key in keys"
      :key="key"
      @remove="remove(key)"
    >
      <BaseInput />
    </BaseRemovable>
  </div>
</template>
```

It's illustrative to go through the virtual nodes when we provide a `key`.

The initial render is exactly the same as with no key.

After hitting "Add" for this first time:

```ts
const keyDynamicCollectionVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseButton,
      key: null,
      props: {
        onClick: add,
      },
      children: {
        default: withCtx(() => [createTextNode("Add")]),
      },
    },
    {
      type: Fragment,
      key: null,
      children: [
        {
          type: BaseRemovable,
          key: "a",
          props: {
            onRemove,
            key: "a",
          },
          children: {
            default: withCtx(() => [createVNode(BaseInput)]),
          },
        },
      ],
    },
  ],
};
```

After hitting "Add" the second time:

```ts
const keyDynamicCollectionVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseButton,
      key: null,
      props: {
        onClick: add,
      },
      children: {
        default: withCtx(() => [createTextNode("Add")]),
      },
    },
    {
      type: Fragment,
      key: null,
      children: [
        {
          type: BaseRemovable,
          key: "a",
          props: {
            onRemove,
            key: "a",
          },
          children: {
            default: withCtx(() => [createVNode(BaseInput)]),
          },
        },
        {
          type: BaseRemovable,
          key: "b",
          props: {
            onRemove,
            key: "b",
          },
          children: {
            default: withCtx(() => [createVNode(BaseInput)]),
          },
        },
      ],
    },
  ],
};
```

After we hit the "Remove" button of our first `BaseRemovable`:

```ts
const keyDynamicCollectionVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseButton,
      key: null,
      props: {
        onClick: add,
      },
      children: {
        default: withCtx(() => [createTextNode("Add")]),
      },
    },
    {
      type: Fragment,
      key: null,
      children: [
        {
          type: BaseRemovable,
          // this tells Vue we want the second input to persist
          key: "b",
          props: {
            onRemove,
            key: "b",
          },
          children: {
            default: withCtx(() => [createVNode(BaseInput)]),
          },
        },
      ],
    },
  ],
};
```

### Dynamic Wrapper

The primary limitation of Vue's instance identity model is that `key`s can only be used to distinguish instances within a single level of the user interface. In other words, if the parent of your instance changes, then Vue will always consider it to have a different identity regardless of `key`.

Let's say we have a `BaseFrame` component, and want to provide a user interface to toggle framing a `BaseInput`.

`DynamicWrapper` is a component that creates a dynamically framable `BaseInput`:

```vue
<script setup lang="ts">
import { ref } from "vue";
import BaseButton from "./BaseButton.vue";
import BaseFrame from "./BaseFrame.vue";
import BaseInput from "./BaseInput.vue";

const shouldFrame = ref(false);

const toggleShouldFrame = () => {
  shouldFrame.value = !shouldFrame.value;
};
</script>

<template>
  <div>
    <BaseFrame v-if="shouldFrame">
      <BaseInput />
    </BaseFrame>
    <BaseInput v-else />

    <BaseButton @click="toggleShouldFrame">Frame</BaseButton>
  </div>
</template>
```

When we render `DynamicWrapper`, we will see that our `BaseInput` is cleared whenever we hit the toggle. On each toggle, Vue destroys our current `BaseInput` instance, and creates a new instance. Vue thinks the identity of our `BaseInput` has changed because it's position in the user interface has changed.

Unfortunately, because this move spans more than a single level, `key` will not help us. The following code is equally broken:

```vue
<script setup lang="ts">
import { ref } from "vue";
import BaseButton from "./BaseButton.vue";
import BaseFrame from "./BaseFrame.vue";
import BaseInput from "./BaseInput.vue";

const shouldFrame = ref(false);

const toggleShouldFrame = () => {
  shouldFrame.value = !shouldFrame.value;
};
</script>

<template>
  <div>
    <BaseFrame v-if="shouldFrame">
      <BaseInput key="a" />
    </BaseFrame>
    <BaseInput
      v-else
      key="a"
    />

    <BaseButton @click="toggleShouldFrame">Frame</BaseButton>
  </div>
</template>
```

You might try to maintain tree stability by rewriting the component as follows:

First we define `BaseFragment.vue`. The is needed because the `Fragment` exported by Vue is just a `symbol`:

```vue
<template>
  <slot />
</template>
```

```vue
<script setup lang="ts">
import { ref } from "vue";
import BaseButton from "./BaseButton.vue";
import BaseFragment from "./BaseFragment.vue";
import BaseFrame from "./BaseFrame.vue";
import BaseInput from "./BaseInput.vue";

const shouldFrame = ref(false);

const toggleShouldFrame = () => {
  shouldFrame.value = !shouldFrame.value;
};
</script>

<template>
  <div>
    <component :is="shouldFrame ? BaseFrame : BaseFragment">
      <BaseInput />
    </component>

    <BaseButton @click="toggleShouldFrame">Frame</BaseButton>
  </div>
</template>
```

This example is syntactically deceptive. It's seems unambiguous that we intend to render a single `BaseInput` throughout the life of a `DynamicWrapper` instance. But Vue does not consider the syntax of our declaration, but rather considers the virtual nodes produced by `DynamicWrapper` on render.

When framing is off, `DynamicWrapper` renders:

```ts
const dynamicWrapperVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseFragment,
      key: null,
      children: {
        default: withCtx(() => [createVNode(BaseInput)]),
      },
    },
    {
      type: BaseButton,
      key: null,
      props: {
        toggleShouldFrame,
      },
      children: {
        default: withCtx(() => [createTextNode("Frame")]),
      },
    },
  ],
};
```

When framing is on, `DynamicWrapper` renders:

```ts
const dynamicWrapperVNode = {
  type: "div",
  key: null,
  children: [
    {
      type: BaseFrame,
      key: null,
      children: {
        default: withCtx(() => [createVNode(BaseInput)]),
      },
    },
    {
      type: BaseButton,
      key: null,
      props: {
        toggleShouldFrame,
      },
      children: {
        default: withCtx(() => [createTextNode("Frame")]),
      },
    },
  ],
};
```

When Vue sees that the `type` switches from `BaseFragment` to `BaseFrame`, it destroys everything else in the tree without further examination.

You might think to try to use [`KeepAlive`](https://vuejs.org/guide/built-ins/keep-alive.html) to preserve the `BaseInput` instance. Unfortunately, this will also not work because [`KeepAlive`](https://vuejs.org/guide/built-ins/keep-alive.html) lets us preserve many instances in the same spot, but does not preserve an instance that moves throughout the user interface.

Here's an attempt using [`KeepAlive`](https://vuejs.org/guide/built-ins/keep-alive.html):

```vue
<script setup lang="ts">
import { ref } from "vue";
import BaseButton from "./BaseButton.vue";
import BaseFragment from "./BaseFragment.vue";
import BaseFrame from "./BaseFrame.vue";
import BaseInput from "./BaseInput.vue";

const shouldFrame = ref(false);

const toggleShouldFrame = () => {
  shouldFrame.value = !shouldFrame.value;
};
</script>

<template>
  <div>
    <KeepAlive>
      <component :is="shouldFrame ? BaseFrame : BaseFragment">
        <BaseInput />
      </component>
    </KeepAlive>

    <BaseButton @click="toggleShouldFrame">Frame</BaseButton>
  </div>
</template>
```

This example does preserve our input, but not in the way we want. There will be two `BaseInput` instances created: one wrapped by `BaseFrame` and the other by `BaseFragment`. The two instances will coexist and toggle between `activated` and `deactivated`. Whatever we type into each input will still be there when we toggle back, but at all times we're dealing with two distinct inputs.

#### Workaround

There is no direct way to preserve the identity of our `BaseInput`.

Since `BaseFrame` is an internal component, we can try to modify its implementation to produce the same effect while maintaining tree structure. The original `BaseFrame` component was implemented like this:

```vue
<template>
  <div class="flex p-3 border-4 rounded-md border-yellow-500">
    <slot />
  </div>
</template>
```

We could change `BaseFrame`'s implementation to the following:

```vue
<script setup lang="ts">
defineProps<{
  shouldFrame: boolean;
}>();
</script>

<template>
  <div :class="shouldFrame && 'flex p-3 border-4 rounded-md border-yellow-500'">
    <slot />
  </div>
</template>
```

And then we could rewrite `DynamicWrapper` as follows:

```vue
<script setup lang="ts">
import { ref } from "vue";
import BaseButton from "./BaseButton.vue";
import BaseFrame from "./BaseFrame.vue";
import BaseInput from "./BaseInput.vue";

const shouldFrame = ref(false);

const toggleShouldFrame = () => {
  shouldFrame.value = !shouldFrame.value;
};
</script>

<template>
  <div>
    <BaseFrame :should-frame="shouldFrame">
      <BaseInput />
    </BaseFrame>

    <BaseButton @click="toggleShouldFrame">Frame</BaseButton>
  </div>
</template>
```

This would work correctly because the tree structure remains identical regardless of frame state. However, this little hack will only work sometimes. In the common case, we cannot easily achieve the desired effect while maintaining the same tree structure. Further, when we're working with third-party components we're out of luck.

The commonly employed workaround for such identity issues is **state-lifting**. With state-lifting, you move your state higher in your user interface. You move your state up until you find somewhere in your tree that has a stable identity for at least the intended lifetime of your state.

Let's look at how we can make use of state-lifting in the frame example.

First we create a `BaseFungibleInput` component:

```vue
<script setup lang="ts">
defineProps<{
  input: string;
}>();

const emit = defineEmits<{
  (e: "input", value: string): void;
}>();
</script>

<template>
  <input
    :value="input"
    @input="emit('input', ($event.target as HTMLInputElement).value)"
  />
</template>
```

Then we make use of it in `LiftDynamicWrapper`:

```vue
<script setup lang="ts">
import { ref } from "vue";
import BaseButton from "./BaseButton.vue";
import BaseFrame from "./BaseFrame.vue";
import BaseFungibleInput from "./BaseFungibleInput.vue";

const input = ref("");
const shouldFrame = ref(false);

const toggleShouldFrame = () => {
  shouldFrame.value = !shouldFrame.value;
};

const setInput = (value: string) => {
  input.value = value;
};
</script>

<template>
  <div>
    <BaseFrame v-if="shouldFrame">
      <BaseFungibleInput
        :input="input"
        @input="setInput"
      />
    </BaseFrame>
    <BaseFungibleInput
      v-else
      :input="input"
      @input="setInput"
    />

    <BaseButton @click="toggleShouldFrame">Frame</BaseButton>
  </div>
</template>
```

Our new `BaseFungibleInput` component is stateless; it takes its state as a prop and emits an event to enable its parent to manage state. The `LiftDynamicWrapper` component now manages `input` state and passes the latest value to `FungibleInput` as a prop.

This version maintains input state as desired. However, like our `BaseInput` instance in the previous example, our `BaseFungibleInput` instance is replaced on each toggle. The difference here is that `BaseFungibleInput`'s identity is meaningless. The replacement instance is identical in every perceptible way.

In this example, state was the only source of non-fungibility. But generally, any aspect of an instance that creates a perceptible difference between itself and its replacement must be managed for this technique to work. The consequences of lost state are often dramatic. However, other fungibility failures can be subtle. For example, the failure to make lifecycle hooks idempotent.

There's a way to understand state-lifting in terms of fungibility and encapsulation. In general, there's a tension between fungibility and encapsulation. If you know every possible thing about something, then you can create an exact copy. On the other hand, encapsulation hides information.

State-lifting is a way to tradeoff encapsulation for fungibility. By moving your state higher in the tree, you break encapsulation by exposing state management details. At the same time, you make the instance stateless. A stateless instance is more easily fungible.

In addition to breaking encapsulation, state-lifting has many other negative consequences: hinders composition, increases complexity, complicates debugging, and hurts performance.

## Instance Identity Model and Patching

The instance identity model is often improperly distinguished from patching. The instance identity model specifies when an instance's identity is preserved, and is part of Vue's public API. On the other hand, the algorithm Vue uses to patch the current user interface to the updated user interface is an implementation detail. Crucially, the patch algorithm must respect the instance identity model.

## Conclusion

Vue's instance identity model impacts nearly every aspect of its programming model. The three rules that Vue uses to determine instance identity are easy to understand and have proved workable in practice. Nevertheless, they are inadequate for expressing common identity requirements. The workarounds for this inadequacy such as state-lifting have serious problems. There is a clear need for a more general instance identity model.

## Footnotes

[^1]: Virtual DOM is the popular name. A more general name is virtual node. Browsers represent a UI as a [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction) tree. Virtual nodes are commonly used to virtualize the DOM. Hence, virtual DOM.
[^2]: In practice, a virtual node contains various internal properties. These are not relevant to the discussion, and elided for clarity.
[^3]: This is almost the same as saying that siblings can be distinguished by providing a key. However, instances at the same level need not coexist.
[^4]: Vue does not guarantee that it will render a new user interface on each state update. Vue only guarantees that it will render a new user interface after state updates asynchronously. Thus, many state updates may be batched into a single paint, and the intermediate states may never materialize. But the core conceptual model is still state/view correspondence.
