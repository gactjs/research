# Svelte's Instance Identity Model

- [Svelte's Instance Identity Model](#sveltes-instance-identity-model)
  - [Introduction](#introduction)
  - [Source Code](#source-code)
  - [Why Instance Identity Matters](#why-instance-identity-matters)
    - [Inexact Declarations](#inexact-declarations)
    - [Fungibility](#fungibility)
  - [Svelte's Instance Identity Model](#sveltes-instance-identity-model-1)
    - [Static Structure](#static-structure)
    - [Dynamic Collection](#dynamic-collection)
      - [`useDynamicCollection`](#usedynamiccollection)
      - [`Removable`](#removable)
      - [`DynamicCollection`](#dynamiccollection)
    - [Dynamic Wrapper](#dynamic-wrapper)
      - [Workaround](#workaround)
  - [Instance Identity Model and Patching](#instance-identity-model-and-patching)
  - [Conclusion](#conclusion)

## Introduction

Svelte is a declarative, reactive framework for building user interfaces. The central abstraction is a **component**, which maps state to a user interface declaration. A user interface declaration is a tree composed of components and intrinsic elements (components built into the rendering environment). A component is a blueprint that Svelte instantiates creating an **instance**. As state changes, an instance's user interface is updated. Svelte must transform the current user interface to the updated user interface. Svelte compiles components into code that can handle this transformation. This patch process raises an important question of identity. When should an instance in the current user interface correspond to an instance in the updated user interface? Svelte answers this question by following a set of rules that collectively comprise its **instance identity model**. The instance identity model determines when instances are created and destroyed. This creation and destruction influences nearly every part of a reactive user interface. In this document, we will explore Svelte's instance identity model in depth.

## Source Code

This document has an associated MIT licensed playground available [here](https://stackblitz.com/edit/svelte-instance-identity-model-playground?file=README.md).

This document generally excludes styles for clarity.

## Why Instance Identity Matters

### Inexact Declarations

In Svelte, the declarations in our components elide many details. Let's take a look at a simple `Input` component:

```svelte
<script lang="ts">
  let value = "";
</script>

<input bind:value />
```

The declaration leaves out many crucial details:

- Is our input focused ?
- If focused, where's the cursor ?
- Is any of the text in our input selected ?

The primary reason these details are left out is **encapsulation**. The built-in `input` is a complex element that provides many features. But this complexity is encapsulated. We get focus, cursor position, and selection support without having to do anything, and more importantly without having to know anything. If our declaration needed to provide these details, we would also have to manage these details. In other words, we would completely break encapsulation.

### Fungibility

**Fungibility** is the property that something is exactly replaceable. The common example of something fungible is a dollar. If the dollar in your pocket is magically swapped with the dollar in my pocket, then we should be indifferent. Fungibility is rare. Most things in the world are not fungible. Even things that are fungible in theory lose fungibility in practice (e.g. each dollar has a serial number).

As discussed, our declarations are necessarily vague. And consequently, our instances are non-fungible because Svelte doesn't have the data to create an exact replacement. This non-fungibility is the reason instance identity matters.

## Svelte's Instance Identity Model

Svelte's instance identity model is comprised of three rules:

1. An instance declared outside of control flow has persistent identity.
2. Each change in the active branch of a conditional declaration creates new instances.
3. Instances within an `each` block have identity tied to their corresponding index by default and to a key when specified.

We will now explore this model in depth in a series of examples.

### Static Structure

Let's walk through the rendering of our `Input` component:

```svelte
<script lang="ts">
  let value = "";
</script>

<input bind:value />
```

As we type, Svelte updates the `value` of our `input`. But Svelte does nothing to manage its focus or cursor state. Nevertheless, the updated user interface shows the correct value, focus, and cursor position. The `input` instance is declared outside of control flow, and the first identity rule applies. Because the identity of our `input` persists across rerenders, its internal state is maintained.

The importance of identity preservation becomes conspicuous when it's disturbed. We can use either the second or third identity rule to dynamically change the identity of `input`.

The following `BranchInput` component alters `input` identity through control flow:

```svelte
<script lang="ts">
  let value = "";
</script>

{#if value === "rerender"}
  <input bind:value />
{:else}
  <input bind:value />
{/if}
```

`BranchInput` will switch between branches when the `value` becomes and changes from `"rerender"`. By the second identity rule, our `input` identity changes. As a result, focus and cursor position are lost.

`KeyInput` provides an alternative expression of the same identity semantics:

```svelte
<script lang="ts">
  let value = "";
</script>

{#each [value] as currentValue (currentValue === "rerender" ? 1 : 0)}
  <input bind:value />
{/each}
```

When the value becomes and changes from `"rerender"`, the specified key toggles. Svelte looks for an `input` in the previous render with the key provided in the current render. When it cannot find one, it creates a new `input` for us. Consequently, input focus and cursor position are lost.

### Dynamic Collection

Instance identity becomes prominent once our user interface is structurally dynamic. Let's consider a user interface that let's us dynamically add and remove `Input`s.

#### `useDynamicCollection`

The `useDynamicCollection` hook lets us manage a dynamic set of keys:

```ts
import { writable, type Readable } from "svelte/store";
// createKey will produce an alphabetically sequential series of keys
import { createKeyFactory } from "../utils/createKeyFactory";

export type DynamicCollection = {
  keys: Readable<string[]>;
  add: () => void;
  remove: (removedKey: string) => void;
};

export const useDynamicCollection = (): DynamicCollection => {
  const { subscribe, update } = writable<string[]>([]);

  const createKey = createKeyFactory();

  const add = () => {
    update((keys) => [...keys, createKey()]);
  };

  const remove = (removedKey: string) => {
    update((keys) => keys.filter((key) => key !== removedKey));
  };

  return {
    keys: { subscribe },
    add,
    remove,
  };
};
```

#### `Removable`

The `Removable` component allows us to wrap other components in a removal user interface.

```svelte
<script setup lang="ts">
  import { createEventDispatcher } from "svelte";
  import Button from "./Button.svelte";

  const dispatch = createEventDispatcher();
</script>

<div>
  <slot />
  <Button on:click={() => dispatch("remove")}>Remove</Button>
</div>
```

#### `DynamicCollection`

When declaring instances based on a dynamic collection, index position is an unreliable indicator of identity. The corresponding index of an instance may be completely different on each update. By specifying a key we can help Svelte correctly correlate identity between renders.

To more clearly demonstrate the importance of declaring a key with a dynamic collection, we will first leave them out.

```svelte
<script lang="ts">
  import { useDynamicCollection } from "../hooks/useDynamicCollection";
  import Button from "./Button.svelte";
  import Input from "./Input.svelte";
  import Removable from "./Removable.svelte";

  const { keys, add, remove } = useDynamicCollection();
</script>

<div>
  <Button on:click={add}>Add</Button>
  {#each $keys as key}
    <Removable on:remove={() => remove(key)}>
      <Input />
    </Removable>
  {/each}
</div>
```

After we click the "Add" button for the first time, the value of `$keys` is `["a"]`, and our `DynamicCollection` instance renders a removable `Input`. Let's say we type `"first"` into this input and then click the "Add" button again. The value of `$keys` is now `["a", "b"]` and `DynamicCollection` renders two removable `Input`s.

At this point, our user interface seems to be working correctly. The state from our first `Input` is preserved (it reads `"first"`), and we also have a second removable `Input`. As long as we're just adding `Input`s, index is sufficient to identify our instances.

Let's now try to remove the first `Input` by hitting its associated "Remove" button. Unfortunately, when the user interface updates, the second `Input` is removed instead of the first. There was an identity crisis.

The value of `$keys` after removal is `["b"]`, which is correct. But because the index of `"b"` is `0`, Svelte preserved the first `Input`.

This bug is easy to fix, we just need to specify a key:

```svelte
<script lang="ts">
  import { useDynamicCollection } from "../hooks/useDynamicCollection";
  import Button from "./Button.svelte";
  import Input from "./Input.svelte";
  import Removable from "./Removable.svelte";

  const { keys, add, remove } = useDynamicCollection();
</script>

<div>
  <Button on:click={add}>Add</Button>
  {#each $keys as key (key)}
    <Removable on:remove={() => remove(key)}>
      <Input />
    </Removable>
  {/each}
</div>
```

Svelte now associates identity by the specified key. When we go through the same actions as before, the value of `$keys` is again `["b"]`, but this time Svelte keeps the second `Input` because it's associated with `"b"`.

### Dynamic Wrapper

The primary limitation of Svelte's instance identity model is that identity can only be distinguished within a single level of the user interface. Whenever a structural change span levels, we have no direct means to preserve identity.

Let's say we have a `Frame` component, and want to provide a user interface to toggle framing an `Input`.

`DynamicWrapper` is a component that creates a dynamically framable `Input`:

```svelte
<script lang="ts">
  import Button from "./Button.svelte";
  import Frame from "./Frame.svelte";
  import Input from "./Input.svelte";

  let shouldFrame = false;

  const toggleShouldFrame = () => {
    shouldFrame = !shouldFrame;
  };
</script>

<div>
  {#if shouldFrame}
    <Frame>
      <Input />
    </Frame>
  {:else}
    <Input />
  {/if}

  <Button on:click={toggleShouldFrame}>Frame</Button>
</div>
```

When we render `DynamicWrapper`, we will see that our `Input` is cleared whenever we hit the toggle. On each toggle, Svelte destroys our current `Input` instance, and creates a new instance. Svelte thinks the identity of our `Input` has changed because the active branch changed.

Unfortunately, because this move spans more than a single level, `key` will not help us. The following code is equally broken:

```svelte
<script lang="ts">
  import Button from "./Button.svelte";
  import Frame from "./Frame.svelte";
  import Input from "./Input.svelte";

  let shouldFrame = false;

  const toggleShouldFrame = () => {
    shouldFrame = !shouldFrame;
  };
</script>

<div>
  {#if shouldFrame}
    <Frame>
      {#each ["a"] as key (key)}
        <Input />
      {/each}
    </Frame>
  {:else}
    {#each ["a"] as key (key)}
      <Input />
    {/each}
  {/if}

  <Button on:click={toggleShouldFrame}>Frame</Button>
</div>
```

#### Workaround

There is no direct way to preserve the identity of our `Input`.

Since `Frame` is an internal component, we could try to modify its implementation to produce the same effect while maintaining tree structure. The original `Frame` component was implemented like this:

```svelte
<div class="flex p-3 border-4 rounded-md border-yellow-500">
  <slot />
</div>
```

We could change `Frame`'s implementation to the following:

```svelte
<script lang="ts">
  export let shouldFrame: boolean;
</script>

<div class={shouldFrame ? "flex p-3 border-4 rounded-md border-yellow-500" : ""}>
  <slot />
</div>
```

And then we could rewrite `DynamicWrapper` as follows:

```svelte
<script lang="ts">
  import Button from "./Button.svelte";
  import Frame from "./Frame.svelte";
  import Input from "./Input.svelte";

  let shouldFrame = false;

  const toggleShouldFrame = () => {
    shouldFrame = !shouldFrame;
  };
</script>

<div>
  <Frame {shouldFrame}>
    <Input />
  </Frame>

  <Button on:click={toggleShouldFrame}>Frame</Button>
</div>
```

This would work correctly because declarations outside control flow have persistent identity. However, this little hack will only work sometimes. In the common case, we cannot easily achieve the desired effect without using conditional declarations. Further, when we're working with third-party components we're out of luck.

The commonly employed workaround for such identity issues is **state-lifting**. With state-lifting, you move your state higher in your user interface. You move your state up until you find somewhere in your tree that has a stable identity for at least the intended lifetime of your state.

Let's look at how we can make use of state-lifting in the frame example.

First we create a `FungibleInput` component:

```svelte
<script lang="ts">
  export let value: string;
</script>

<input {value} on:input />
```

Then we make use of it in `LiftDynamicWrapper`:

```svelte
<script setup lang="ts">
  import Button from "./Button.svelte";
  import Frame from "./Frame.svelte";
  import FungibleInput from "./FungibleInput.svelte";

  let value = "";
  let shouldFrame = false;

  const toggleShouldFrame = () => {
    shouldFrame = !shouldFrame;
  };

  const handleInput = (event: InputEvent) => {
    value = (event.target as HTMLInputElement).value;
  };
</script>

<div>
  {#if shouldFrame}
    <Frame>
      <FungibleInput on:input={handleInput} {value} />
    </Frame>
  {:else}
    <FungibleInput on:input={handleInput} {value} />
  {/if}

  <Button on:click={toggleShouldFrame}>Frame</Button>
</div>
```

Our new `FungibleInput` component is stateless; it takes `value` as a prop and forwards input events to enable its parent to mange state. The `LiftDynamicWrapper` component now manages `input` state, and passes the latest `value` to `FungibleInput`.

This version maintains `value` as desired. However, like our `Input` instance in the previous example, our `FungibleInput` instance is replaced on each toggle. The difference here is that `FungibleInput`'s identity is meaningless. The replacement instance is identical in every perceptible way.

In this example, state was the only source of non-fungibility. But generally, any aspect of an instance that creates a perceptible difference between itself and its replacement must be managed for this technique to work. The consequences of lost state are often dramatic. However, other fungibility failures can be subtle. For example, failure to make lifecycle functions idempotent.

There's a way to understand state-lifting in terms of fungibility and encapsulation. In general, there's a tension between fungibility and encapsulation. If you know every possible thing about something, then you can create an exact copy. On the other hand, encapsulation hides information.

State-lifting is a way to tradeoff encapsulation for fungibility. By moving your state higher in the tree, you break encapsulation by exposing state management details. At the same time, you make the instance stateless. A stateless instance is more easily fungible.

In addition to breaking encapsulation, state-lifting has many other negative consequences: hinders composition, increases complexity, complicates debugging, and hurts performance.

## Instance Identity Model and Patching

The instance identity model is often improperly distinguished from patching. The instance identity model specifies when an instance's identity is preserved, and is part of Svelte's public API. On the other hand, the code Svelte uses to patch the current user interface to the updated user interface is an implementation detail. Crucially, the patch code must respect the instance identity model.

## Conclusion

Svelte's instance identity model impacts nearly every aspect of its programming model. The three rules that Svelte uses to determine instance identity are easy to understand and have proved workable in practice. Nevertheless, they are inadequate for expressing common identity requirements. The workarounds for this inadequacy such as state-lifting have serious problems. There is a clear need for a more general instance identity model.
