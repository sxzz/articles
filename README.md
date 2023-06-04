# Preface

Hi, here's 三咲智子, a member of the Vue core team. This release of Vue 3.3 is mainly to improve the DX (developer experience), adding some syntactic sugar and macros, and improvements in TypeScript.

- Generic Component
- Importing External TypeScript Types in SFC (Single File Components)
- Using defineSlots to Define Slot Types
- Convenient Syntax with defineEmits
- Defining Component Options with defineOptions
- (Experimental) Reactive Props Destructuring
- (Experimental) Syntactic Sugar with defineModel
- Deprecation of Reactivity Transform

During the period when I first joined Vue last year, I was actively contributing PR for Vue 3. it was only recently that my contributions were finally incorporated into Vue 3.3. Vue 3.3 has absorbed a total of five or six features from [Vue Macros](https://vue-macros.sxzz.moe/). Today, I would like to briefly discuss the part I contributed myself.

# defineOptions Macro

- PR: https://github.com/vuejs/core/pull/5738
- RFC: https://github.com/vuejs/rfcs/discussions/430
- Development record: https://www.bilibili.com/video/BV1uu411y7WE

## Previously

Think about how before we had ``<script setup>``, it was easy to define props and emits by simply adding a property at the same level as setup. However, with the introduction of ``<script setup>``, we can no longer do that. The setup property is no longer available, so we cannot add properties at the same level. To address this issue, we introduced two macros: defineProps and defineEmits.

But this only solves the two properties props and emits. If we want to define the component's name, inheritances, or other custom attributes, we still have to go back to the original usage - add a normal ``<script>`` tag. This will result in two ``<script>`` tags. For me personally, this is unacceptable.

- Having two script tags can potentially cause unexpected issues with ESLint plugins or Volar.
- If both script tags contain import statements, Vue will perform some peculiar special handling.
- It becomes not only cumbersome but also difficult to understand in terms of DX (Developer Experience).

## Current Situation

So we newly introduced the defineOptions macro in Vue 3.3. As the name suggests, it is mainly used to define the options of the Options API. We can define arbitrary options with defineOptions, except props, emits, expose, slots (since these can be done with defineXXX). We can even use the h function or JSX to write the rendering function render directly in defineOptions without the ``<template>`` tag (of course this is not recommended).

## 🌰 Example
```javascript
<script setup>
defineOptions({
  name: 'Foo',
  inheritAttrs: false,
  // ... More custom attributes
})
</script>
```
[Vue SFC Playground](https://play.vuejs.org/#eNolzDEKgDAMQNGrhC4qiO5FBBdXL9BFNGJA09BEF/HuKo7/D+9ynUh1Hui8a3RKJAaKdkgbeMaFGAcxiqz5FRiAxx09ZH2MWfk18YqJrDNL6mEZN8X330Xgpv611t0P8xsjNQ==)

## The Story Behind

The origin of this feature can be traced back to the refactoring of Element Plus components into ``<script setup>``. For component libraries, we wanted to customize the name of the components (thename property) rather than relying on the default file name. However, we didn't want to revert to the original way of writing components, so I developed a plugin (where the dream began 🤣) called **unplugin-vue-define-options**. After several modifications, it finally evolved into the current defineOptions macro available in Vue 3.3.

# Promote static constants

- PR:  https://github.com/vuejs/core/pull/5752
- Development record: https://www.bilibili.com/video/BV1st4y1G7sG

This feature is an optimization of the Single File Component (SFC) compiler. It introduces the hoistStatic option in the script section.

## hoistStatic in the template

The hoistStatic feature under the template is similar in nature. The Vue compiler has an optimization that allows static element nodes to be hoisted to the top-level scope. This means they are executed only once during code loading, rather than being repeatedly executed every time the render function is invoked (although there may be some drawbacks in extreme cases).

Let's look at an example 🌰.

```javascript
<template>
  <div id="title">Hello World</div>
</template>
```

[Vue SFC Playground](https://play.vuejs.org/#eNqrVnIsKNArK01VslKyKUnNLchJLEm1i8lTULBJySxTyEyxjVEqySzJSY1RsvNIzcnJVwjPL8pJsdEHygKV2ejD9SjVAgDb8hoW)

The above code will be compiled into the following JavaScript (non-essential code omitted):

```javascript
const _hoisted_1 = { id: 'title' }
function render(_ctx, _cache) {
  return _openBlock(), _createElementBlock('div', _hoisted_1, 'Hello World')
}
```
We can see that the **​_hoisted_1** variable is intentionally hoisted to the top level by the compiler. If this feature is disabled, it would be within the render function instead.

## hoistStatic in the Script tag

Before Vue 3.3, only the mentioned optimization existed. In Vue 3.3, we have also made a similar optimization. If a constant's value is a primitive value (string, number, boolean, bigint, symbol, null, undefined), the declaration of that constant will be hoisted to the top level. (Note: Symbol is currently not implemented.)

Since constants of these types cannot be changed, their declaration has the same effect regardless of where they are declared.

## effect

In addition to performance optimization, this feature has a more useful place. Before Vue 3.3, when we used macros, there was no way to pass variables defined in the ``<script setup>`` block. Take a look at the example below.

```javascript
<script setup>
const name = 'Foo'
defineOptions({
  name,
})
</script>
```

[Vue SFC Playground](https://play.vuejs.org/#eNp9jt1KA0EMhV9lmJsqdGdABWGpUl9AvDde1N1UR5kkJLP1ouy7O6st9Ad6me8k+c7WP4mEzYC+9QvrNElxhmWQR6COyYoTZXleZXQPbrZmngH1uE6EL5Xb1es+frsGWsT/D/XWz33KwlqavJLwZUxVsAVyDnaBgW/dH5lYbTDN4D9LEWtjHEi+P0LHOS5rFnWgkjI2PeflbbgJd/exT1YOeUDLzbvyj6FWI/j5wfNY4Qa1Uaz1FfWi7GT3SHiSnUkn5wg0+vEX+UN45g==)

We will get an error.

```javascript
[@vue/compiler-sfc] `defineOptions()` in <script setup> cannot reference locally declared variables because it will be hoisted outside of the setup() function. If your component options require initialization in the module scope, use a separate normal <script> to export the options instead.
```

This is because of the reason mentioned before, defineProps will add a props attribute at the same level as setup, and the name constant is declared in the setup function. We cannot refer to a variable inside a function outside of a function because the variable has not been initialized at all. The following code is definitely wrong.

```javascript
const __sfc__ = {
  props: [propName],
  setup(__props) {
    const propName = 'foo'
  },
}
```

After Vue 3.3, the propName in line 4 will be hoisted to the first line, making the code fully understandable. This feature is enabled by default, and in general, developers do not need to be aware of its existence.

## The Story Behind

The reason for developing this feature is also similar to the previous one. It is because after Element Plus sets the name using defineOptions, it needs to throw an exception under certain conditions. The exception needs to include the component's name to facilitate user debugging.

```javascript
<script setup>
const name = 'ElButton'
defineOptions({
  name,
})
// ...
if (condition) {
  throw new Error(`${name}: something went wrong.`)
}
</script>
```

So I wanted to avoid repetitive code by extracting the component name into a constant and referencing it separately in both defineOptions and when throwing the exception.

# defineModel Macro

- PR: https://github.com/vuejs/core/pull/8018
- RFC: https://github.com/vuejs/rfcs/discussions/503
- Development record: [Phase 1](https://www.bilibili.com/video/BV1MS4y1t7Pi),[Phase 2](https://www.bilibili.com/video/BV1sY4y1P7gD)
- Twitter: https://twitter.com/sanxiaozhizi/status/1644564064931307522

## Motivation

This macro is purely syntactic sugar. Prior to Vue 3.3, defining a two-way binding prop was quite cumbersome.

```javascript
<script setup lang="ts">
const props = defineProps<{
  modelValue: number
}>()

const emit = defineEmits<{
  (evt: 'update:modelValue', value: number): void
}>()

// update value
emit('update:modelValue', props.modelValue + 1)
</script>
```

We had to first define **props** and then define **emits**, which involved a lot of repetitive code. If we needed to modify the value, we also had to manually call the **emit** function."

I was thinking: why don't we wrap a function (macro) to simplify this step? Hence the **defineModel** macro came into existence.

## 🌰 Example

```javascript
<script setup>
const modelValue = defineModel()
modelValue.value++
</script>
```

The cumbersome 7 lines of code mentioned above can be reduced to just two lines in Vue 3.3!

## The difference with useVModel

In VueUse, there is also the useVModel function that can achieve similar effects. So why do we still need to introduce this macro?

That's because VueUse only combines **props** and **emit** functions into a **Ref**, but it cannot define **props** and **emits** for the component. This means that developers still need to manually call **defineProps** and **defineEmits** separately.

## The Story Behind

😛 There is no story behind this, it is purely troublesome to write. At first, I made the **defineModel** macro in [Vue Macros](https://vue-macros.sxzz.moe/) (now renamed to defineModels to distinguish it from the official one), and the effect is not bad.

# Importing external types in SFC

## The Background

Since the release of Vue 3.2, there has been a highly popular issue in the Vue community——[How to import interface for defineProps](https://github.com/vuejs/core/issues/4294).

To address this issue, we have two options ahead of us.

- The Vue SFC compiler to invoke the TypeScript compiler, calculate the final type, and determine which types it includes (such as String, Number, Boolean, Function, etc).
- Achieve our own simplified TypeScript analyzer and address most of the scenarios ourselves.

As we all know, TypeScript's type acrobatics can be quite daunting. If we truly want to solve this issue perfectly, the Vue SFC compiler would need to parse and calculate all types, just like the TypeScript compiler. While the first approach sounds good, it has a significant drawback: it unavoidably relies on the massive and bloated TypeScript compiler, which would greatly slow down the build process.

Ultimately, in [Vue Macros](https://vue-macros.sxzz.moe/) I chose the second approach, despite the current limitation of not being able to pass complex types in macros. However, this limitation is expected to be gradually improved in the future.

## Vue 3.3 and Vue Macors

After Vue Macros implemented this simple analyzer, Vue core also made a similar implementation.

However, as of now, there are still differences. The iteration speed of Vue Macros is much faster than that of Vue core itself. Currently, Vue Macros is supporting more peculiar syntax, while Vue 3.3 still lacks support for certain syntax.

Therefore, in the future, if you come across a type that Vue cannot parse, it's worth trying Vue Macros. If Vue Macros still doesn't work, you can try providing a minimal reproducible code and open an issue in the Vue Macros repository. Alternatively, you can attempt using a simpler syntax that avoids secondary inference.

# defineSlots Macros

- PR: https://github.com/vuejs/core/pull/7982
- Twitter:  https://twitter.com/sanxiaozhizi/status/1641378248448937984

## The Background

Vue 3.3 introduced the defineSlots macro. We can use **defineSlots** to define the types of our own slots. This macro is not often needed in simple components but proves to be very useful for complex components, especially when used in conjunction with generic components. Additionally, we can manually specify the types when Volar cannot accurately infer them.

## 🌰 Example

```javascript
<script setup lang="ts">
const slots = defineSlots<{
  default(props: { foo: string; bar: number }): any
}>()
</script>
```

We manually defined the slot scope type for the "default" component.

## 🌰 Real Example

For example, if we have a pagination component, we can use slots to control how each item should be rendered.

```javascript
<script setup lang="ts" generic="T">
// Children components Paginator
defineProps<{
  data: T[]
}>()

defineSlots<{
  default(props: { item: T }): any
}>()
</script>
```

```javascript
<template>
  <!-- Parent Components -->
  <Paginator :data="[1, 2, 3]">
    <template #default="{ item }">{{ item }}</template>
  </Paginator>
</template>
```

When we pass the “data” parameter, which is of type number[], the “item” is also inferred as a number. The type of the “item” will change based on the type passed to the “data” parameter.

# defineEmits more convenient syntax

- PR: https://github.com/vuejs/core/pull/7992
- Twitter: https://twitter.com/youyuxi/status/1641403989026820098

This feature is also purely syntactic sugar.

## Example

```javascript
<script setup lang="ts">
const emits = defineEmits<{
  (evt: 'update:modelValue', value: string): void
  (evt: 'change'): void
}>()

// ⬇️ Vue 3.3 之后
const emits = defineEmits<{
  'update:modelValue': [value: string],
  'change': []
}>()
</script>
```

Before Vue 3.3, we had to write a few extra characters, but now we can omit them whenever possible.

# Obsolete Reactivity Transform syntactic sugar

As early as the beginning of the year, the Vue team announced the deprecation of the Reactivity Transform syntax. Specifically, it will be deprecated in Vue 3.3, resulting in a warning when used. In Vue 3.4, it will be completely removed.

> Personally, I believe that Reactivity Transform still has its usefulness and relevance.

Although it has been deprecated by the official Vue team, its functionality has been moved to [Vue Macros](https://vue-macros.sxzz.moe/). This means that you don't have to rush to migrate to the old syntax, and you can still use [plugins](https://vue-macros.sxzz.moe/features/reactivity-transform.html) and receive regular bug fixes without any issues.

For specific reasons behind its removal, you can read this [article](https://github.com/vuejs/rfcs/discussions/369#discussioncomment-5059028).

# Postscript

In general, I am thrilled to see that Vue is open to accepting suggestions and proposals from the community. It makes me proud to have contributed to Vue 3.3. If you're interested in learning about more features, I recommend reading the [relevant articles on the Vue blog](https://blog.vuejs.org/posts/vue-3-3), where you'll find more detailed information.

# About Vue Macros

[Vue Macros](https://vue-macros.sxzz.moe/) is currently an independent project separate from Vue official. Unlike Vue official, its purpose is to explore different possibilities.

I am more inclined to see more radical ideas, even if they are not fully matured. We can experiment further in Vue Macros and once they are matured, we can attempt to merge them into the Vue official repository.

Currently, Vue Macros is being maintained by me alone, and I hope to attract more community members to participate in its development! 💕
