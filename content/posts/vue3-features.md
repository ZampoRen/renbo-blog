---
title: "Vue.js 3.0 新特性详解与实战"
date: 2024-02-15T13:45:00+08:00
draft: false
author: "博主"
description: "深入解析Vue.js 3.0的新特性，包括Composition API、Teleport、Fragments等，并提供实际项目中的应用示例。"
tags: ["Vue.js", "前端开发", "JavaScript", "组合式API"]
categories: ["技术"]
---

Vue.js 3.0带来了许多令人兴奋的新特性，今天我们来深入了解这些变化以及如何在实际项目中应用它们。

## 🚀 Vue 3的主要改进

### 性能提升
- **更快的虚拟DOM**：重写了虚拟DOM，性能提升1.3-2倍
- **更小的包体积**：通过tree-shaking支持，减少了41%的包体积
- **更好的TypeScript支持**：完全用TypeScript重写

### 新的响应式系统
基于ES6的Proxy实现，相比Vue 2的Object.defineProperty：
- 支持数组索引和length的监听
- 支持Map、Set、WeakMap、WeakSet
- 支持13种数组变更方法的监听

## 💡 Composition API详解

### 为什么需要Composition API？

Vue 2的Options API在处理复杂组件时存在一些问题：
- 逻辑复用困难
- 类型推断不够友好  
- 大组件难以维护

### 基础语法

```javascript
import { ref, reactive, computed, watch, onMounted } from 'vue'

export default {
  setup() {
    // 响应式数据
    const count = ref(0)
    const state = reactive({
      name: '张三',
      age: 25
    })
    
    // 计算属性
    const doubleCount = computed(() => count.value * 2)
    
    // 方法
    const increment = () => {
      count.value++
    }
    
    // 生命周期
    onMounted(() => {
      console.log('组件已挂载')
    })
    
    // 监听器
    watch(count, (newVal, oldVal) => {
      console.log(`count从${oldVal}变为${newVal}`)
    })
    
    return {
      count,
      state,
      doubleCount,
      increment
    }
  }
}
```

### 组合函数（Composables）

```javascript
// composables/useCounter.js
import { ref, computed } from 'vue'

export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  
  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => count.value = initialValue
  
  const isEven = computed(() => count.value % 2 === 0)
  
  return {
    count: readonly(count),
    increment,
    decrement,
    reset,
    isEven
  }
}

// 在组件中使用
import { useCounter } from './composables/useCounter'

export default {
  setup() {
    const { count, increment, decrement, isEven } = useCounter(10)
    
    return {
      count,
      increment,
      decrement,
      isEven
    }
  }
}
```

### 逻辑复用示例

```javascript
// composables/useFetch.js
import { ref, reactive } from 'vue'

export function useFetch(url) {
  const data = ref(null)
  const loading = ref(true)
  const error = ref(null)
  
  const fetchData = async () => {
    try {
      loading.value = true
      error.value = null
      const response = await fetch(url)
      if (!response.ok) throw new Error('请求失败')
      data.value = await response.json()
    } catch (err) {
      error.value = err.message
    } finally {
      loading.value = false
    }
  }
  
  fetchData()
  
  return { data, loading, error, refetch: fetchData }
}

// 在组件中使用
export default {
  setup() {
    const { data: users, loading, error } = useFetch('/api/users')
    
    return { users, loading, error }
  }
}
```

## ⚡ 新特性深入

### 1. Teleport（传送门）

```vue
<template>
  <div class="modal-container">
    <button @click="showModal = true">打开模态框</button>
    
    <!-- 将模态框传送到body下 -->
    <Teleport to="body">
      <div v-if="showModal" class="modal-backdrop">
        <div class="modal">
          <h3>模态框标题</h3>
          <p>这是模态框内容</p>
          <button @click="showModal = false">关闭</button>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<script>
import { ref } from 'vue'

export default {
  setup() {
    const showModal = ref(false)
    return { showModal }
  }
}
</script>
```

### 2. Fragments（片段）

Vue 3支持组件有多个根节点：

```vue
<template>
  <!-- Vue 2中这样写会报错 -->
  <header>...</header>
  <main>...</main>
  <footer>...</footer>
</template>
```

### 3. Suspense（实验性）

```vue
<template>
  <Suspense>
    <!-- 异步组件 -->
    <AsyncComponent />
    
    <!-- 加载状态 -->
    <template #fallback>
      <div>加载中...</div>
    </template>
  </Suspense>
</template>

<script>
import { defineAsyncComponent } from 'vue'

export default {
  components: {
    AsyncComponent: defineAsyncComponent(() => import('./AsyncComponent.vue'))
  }
}
</script>
```

## 🔄 响应式系统升级

### ref vs reactive

```javascript
import { ref, reactive, toRefs } from 'vue'

// ref：用于基本类型
const count = ref(0)
const name = ref('张三')

// reactive：用于对象类型
const state = reactive({
  user: {
    name: '李四',
    age: 30
  },
  posts: []
})

// toRefs：将reactive对象转为ref
const { user, posts } = toRefs(state)
```

### 响应式工具函数

```javascript
import { 
  readonly, 
  shallowRef, 
  shallowReactive,
  toRef,
  unref,
  isRef,
  isReactive
} from 'vue'

// 只读
const original = reactive({ count: 0 })
const copy = readonly(original)

// 浅层响应式
const shallowState = shallowReactive({
  foo: 1,
  nested: { bar: 2 }
})

// 判断类型
console.log(isRef(count))        // true
console.log(isReactive(state))   // true
```

## 🎨 模板语法增强

### v-memo（3.2+新增）

```vue
<template>
  <div v-for="item in list" :key="item.id" v-memo="[item.id, item.selected]">
    <!-- 只有当id或selected改变时才重新渲染 -->
    <span>{{ item.name }}</span>
    <span>{{ item.selected ? '已选择' : '未选择' }}</span>
  </div>
</template>
```

### CSS中的v-bind

```vue
<template>
  <div class="dynamic-color">文本颜色动态改变</div>
</template>

<script>
import { ref } from 'vue'

export default {
  setup() {
    const color = ref('red')
    
    setTimeout(() => {
      color.value = 'blue'
    }, 2000)
    
    return { color }
  }
}
</script>

<style>
.dynamic-color {
  color: v-bind(color);
}
</style>
```

## 🛠️ 开发工具和生态

### Vite集成

```bash
# 创建Vue 3项目
npm create vue@latest my-vue-app
cd my-vue-app
npm install
npm run dev
```

### Vue Devtools

Vue 3需要使用新版本的devtools：
- 支持Composition API调试
- 更好的性能分析
- 时间旅行调试

### Pinia状态管理

```javascript
// stores/counter.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const doubleCount = computed(() => count.value * 2)
  
  function increment() {
    count.value++
  }
  
  return { count, doubleCount, increment }
})

// 在组件中使用
import { useCounterStore } from '@/stores/counter'

export default {
  setup() {
    const store = useCounterStore()
    
    return {
      count: store.count,
      increment: store.increment
    }
  }
}
```

## 📚 实战项目示例

### 待办事项应用

```vue
<template>
  <div class="todo-app">
    <h1>待办事项</h1>
    
    <form @submit.prevent="addTodo">
      <input 
        v-model="newTodo" 
        placeholder="添加新任务..." 
        required
      />
      <button type="submit">添加</button>
    </form>
    
    <div class="filters">
      <button 
        v-for="filter in filters" 
        :key="filter"
        :class="{ active: currentFilter === filter }"
        @click="currentFilter = filter"
      >
        {{ filter }}
      </button>
    </div>
    
    <ul class="todo-list">
      <li 
        v-for="todo in filteredTodos" 
        :key="todo.id"
        :class="{ completed: todo.completed }"
      >
        <input 
          type="checkbox" 
          v-model="todo.completed"
        />
        <span>{{ todo.text }}</span>
        <button @click="removeTodo(todo.id)">删除</button>
      </li>
    </ul>
    
    <div class="stats">
      <span>总计: {{ todos.length }}</span>
      <span>已完成: {{ completedCount }}</span>
      <span>未完成: {{ incompleteCount }}</span>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue'

export default {
  name: 'TodoApp',
  setup() {
    // 状态
    const todos = ref([])
    const newTodo = ref('')
    const currentFilter = ref('全部')
    const filters = ['全部', '进行中', '已完成']
    
    // 计算属性
    const filteredTodos = computed(() => {
      switch (currentFilter.value) {
        case '进行中':
          return todos.value.filter(todo => !todo.completed)
        case '已完成':
          return todos.value.filter(todo => todo.completed)
        default:
          return todos.value
      }
    })
    
    const completedCount = computed(() => 
      todos.value.filter(todo => todo.completed).length
    )
    
    const incompleteCount = computed(() => 
      todos.value.filter(todo => !todo.completed).length
    )
    
    // 方法
    const addTodo = () => {
      if (newTodo.value.trim()) {
        todos.value.push({
          id: Date.now(),
          text: newTodo.value.trim(),
          completed: false
        })
        newTodo.value = ''
      }
    }
    
    const removeTodo = (id) => {
      const index = todos.value.findIndex(todo => todo.id === id)
      if (index > -1) {
        todos.value.splice(index, 1)
      }
    }
    
    return {
      todos,
      newTodo,
      currentFilter,
      filters,
      filteredTodos,
      completedCount,
      incompleteCount,
      addTodo,
      removeTodo
    }
  }
}
</script>
```

## 🔧 迁移指南

### 从Vue 2迁移到Vue 3

#### 1. 全局API变化
```javascript
// Vue 2
import Vue from 'vue'
import App from './App.vue'

Vue.config.productionTip = false
Vue.component('MyComponent', MyComponent)

new Vue({
  render: h => h(App)
}).$mount('#app')

// Vue 3
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.component('MyComponent', MyComponent)
app.mount('#app')
```

#### 2. 组件定义变化
```javascript
// Vue 2
export default {
  data() {
    return {
      count: 0
    }
  },
  methods: {
    increment() {
      this.count++
    }
  }
}

// Vue 3 (Options API仍然支持)
export default {
  data() {
    return {
      count: 0
    }
  },
  methods: {
    increment() {
      this.count++
    }
  }
}

// Vue 3 (推荐使用Composition API)
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)
    const increment = () => count.value++
    
    return { count, increment }
  }
}
```

## 🎯 最佳实践

### 1. 何时使用Composition API
- 逻辑复杂的组件
- 需要逻辑复用的场景
- TypeScript项目
- 大型项目

### 2. 性能优化技巧
```javascript
// 使用shallowRef避免深度响应式
const largeObject = shallowRef({ /* 大对象 */ })

// 使用markRaw标记不需要响应式的对象
const nonReactiveObj = markRaw({ /* 普通对象 */ })

// 合理使用v-memo
<template>
  <ExpensiveComponent v-memo="[valueA, valueB]" />
</template>
```

## 📊 总结

Vue 3的主要优势：

| 特性 | Vue 2 | Vue 3 |
|------|-------|-------|
| **性能** | 基准 | 提升1.3-2x |
| **包体积** | 基准 | 减少41% |
| **TypeScript** | 支持 | 原生支持 |
| **组合性** | Mixins | Composition API |
| **响应式** | Object.defineProperty | Proxy |

Vue 3不仅带来了性能提升，更重要的是提供了更好的开发体验和代码组织方式。虽然学习曲线存在，但掌握后能显著提高开发效率。

---

*你在使用Vue 3的过程中遇到了哪些问题？欢迎在评论区交流！*