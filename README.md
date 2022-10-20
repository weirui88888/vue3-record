# 简介
这个库用来学习vue3+vite3项目配置

# 记录
记录一些学习vue3过程中遇到的问题，方便后期学习和回顾

## 1.关于自定义组件ref类型定义的问题

当我们的组件使用了`<script setup>`语法之后，该组件会处于一个封闭的状态，通过ref是无法获取到该组件实例的，不会暴露任何在`<script setup>`中声明的绑定。所以需要我们使用`defineExpose`定义组件需要暴露出去的属性。


```javascript
// Demo.vue

<template>
  <div>{{ msg }}</div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
const exposeMsg = 'this is expose msg'
const noExposeMsg = 'this is no expose msg'
defineExpose({
  exposeMsg
})
</script>
```
上面的代码中，`msg`显示的通过了`defineExpose`对外暴露，而`noExpose`却没有暴露出去，所以

```javascript
// 这是父组件，引用刚才的Demo组件，通过ref获取组件实例

<template>
  <demo ref="demoRef"></demo>
</template>

<script setup lang="ts">
import { Demo } from 'Demo.vue'
import type { Ref } from 'vue'
import { ref, onMounted } from 'vue'

type demoCtx = InstanceType<typeof Demo> // 这里是特别强调的地方，如果这里不定义子组件的类型，是无法直接获取组件完整的实例类型的，会各种ts报错

const demoRef = ref<null | demoCtx> = ref(null)

onMounted(() => {
  console.log(demoRef.value?.exposeMsg) // 能获取到
  console.log(demoRef.value?.noExposeMsg) // 获取不到，因为子组件没有暴露该属性
})
</script>
```
