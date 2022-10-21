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

## 2.关于定义组件prop的两种方式

### 2.1 通过vue2那种熟悉的方式

```javascript
<script>
  // defineProps和defineEmits都不需要手动引入
  const props = defineProps({
    name: {
      type: String,
      default: '化氏一味'
    },
    show: {
      type: Boolean,
      default: false
    }
})
</script>
```

### 2.2 通过ts泛型参数定义类型

```javascript
<script>
  const props = defineProps<{
    name: string
    show: boolean
    labels?: Record<string,any>[]
}>()
</script>
```

但是通过这种方式定义prop会有一个问题，那就是没法给prop属性添加默认值，所以还需要配合`withDefaults`使用

```javascript
<script>
  interface Props {
    name: string
    show: boolean
    labels?: Record<string,any>[]
  }
  const props = withDefaults(defineProps<Props>(), {
    name: '化氏一味',
    show: false
  })
</script>
```

上述两种方式，目前还不清楚孰优孰劣，具体的还是要看个人喜好。当然为了使代码风格趋于统一，还是选择固定的一种模式比较稳妥

## 3.关于封装插件方式与vue2方式的区别

**注**：所谓的插件，就是将一个常用组件以一种全局化的方式暴露给我们的程序。举个简单的例子，一个toast弹窗组件，如果不以插件的形式暴露给使用者的话，那么我们使用的方式往往是先引入该组件，然后通过v-if/v-show来控制该组件的显示与否，形如：

```javascript
<template>
  <div>
    <Toast v-if="showToast" />
    <button @click="showToast = true">展示弹窗</button>
  </div>
</template>

<script>
import Toast from './components/toast.vue'
export default {
  components: {
    Toast
  },
  data() {
    return {
      showToast: false
    }
  }
}
</script>
```

可以明显的看出来，组件的常规化使用步骤是：

- 1.开发组件
- 2.在需要的地方**重复**引入该组件（不考虑全局注册组件的方式）
- 3.通过变量控制组件的展示与否

因为需要重复的引入，表现形式往往比较单一，我们能不能通过一些其他的方式，可以注册一次组件，在多个地方调用呢？（不是指Vue.component全局注册组件），这就是插件的意义和区别

### 3.1 Vue2注册插件方式

```javascript
// popup.js
import Popup from "./popup.vue"

const buildPopupPropData = (
  userOption,defaultOption
) => {
  // 根据用户每次使用this.$popup方法传入的userOption，和初始化该插件时传入的默认配置defaultOption，合并得出最终的选项，最终传入组件中
  return {
    ...
  }
}

const popupPlugin = {
  install(Vue, defaultOption = {}) {
    const PopupConstructor = Vue.extend(Popup)
    Vue.prototype.$popup = popup

    function popup(userOption) {
      if (!userOption) return
      const propsData = buildPopupPropData(userOption, defaultOption)
      const popupInstance = new PopupConstructor({
        propsData
      })
      document.body.appendChild(popupInstance.$mount().$el)
      popupInstance.show()
    }
  }
}

export default popupPlugin

```

然后在主入口文件中引入该popup.js，全局注册即可

```javascript
// main.js
import Vue from "vue"
import Popup from './component/popup.js'
...

Vue.use(Popup)

...
```

### 3.2 Vue3注册插件方式

定义插件方式

```javascript
// toast.ts
import { App, createApp } from 'vue'
import Toast from './toast.vue'

const toastPlugin = {
  install(app: App, defaultOption: Record<string, any> = {}) {
    app.config.globalProperties.$toast = ( // 这里区别于vue2，其他逻辑差不多
      message: string,
      customOption: Record<string, any> = {}
    ) => {
      const option = Object.assign({}, defaultOption, customOption)
      const toast = createApp(Toast, {
        message,
        ...option
      })
      const root = document.createElement('div')
      root.classList.add('app-toast')
      document.body.appendChild(root)
      toast.mount(root)
    }
  }
}

export default toastPlugin
```

注册插件

```javascript
// 入口文件
import Toast from './components/toast.ts'
...
app.use(Toast)
...
```

使用插件

```javascript
// 任意组件中，想要使用注册的插件toast
import { getCurrentInstance } from 'vue'

const appInstance = getCurrentInstance()
const { $toast } = appInstance?.appContext.config.globalProperties

$toast('😊已经到底了')
```
