下面这部分可以理解成：**5年前端面试里，React / Vue 框架深入到底要深入到什么程度。**

不是要求你把源码每一行都背下来，而是要求你能回答这几个问题：

> 为什么页面会更新？
> 为什么有时候不更新？
> 为什么有时候更新太多？
> 状态应该放在哪里？
> 组件应该怎么拆？
> 性能问题怎么定位和解决？
> 你的项目里有没有真的用这些能力解决过问题？

---

# 一、框架深入的核心：从“会写组件”到“理解运行机制”

很多人写了几年 Vue / React，仍然停留在：

```text
会写页面
会调接口
会用组件库
会写表单
会写列表
会写弹窗
```

这只能算熟练业务开发。

5年前端需要达到：

```text
知道状态变化后框架如何触发更新
知道组件为什么重新渲染
知道如何避免无效渲染
知道复杂状态如何管理
知道组件如何抽象成可复用能力
知道项目规模变大后怎么维护
知道性能瓶颈在哪里
知道框架能力和业务场景怎么结合
```

面试官真正想听的是：**你不只是使用框架，而是理解框架背后的更新模型，并能把这种理解用在真实项目里。**

---

# 二、Vue 深入：核心是“响应式驱动视图”

Vue 的底层思想可以简化成一句话：

> 数据变了，依赖这个数据的视图自动更新。

Vue 3 里常用的 `ref()` 会返回一个带 `.value` 的响应式对象，读取 `.value` 会被追踪，修改 `.value` 会触发相关 effect 更新；`reactive()` 则会返回对象的响应式代理，并且默认是深层响应式。([Vue.js][1])

所以 Vue 面试的核心不是“ref 怎么写”，而是：

```text
谁被追踪？
什么时候追踪？
谁触发更新？
更新范围有多大？
怎么减少不必要更新？
```

---

## 1. Vue 响应式要怎么理解？

Vue 3 响应式大致可以这样理解：

```text
读取数据 → track 收集依赖
修改数据 → trigger 触发依赖
依赖变化 → 组件重新执行渲染函数
新旧 VNode 对比 → patch 到真实 DOM
```

面试里你可以这样说：

> Vue 的响应式本质是依赖收集和派发更新。组件渲染时会读取响应式数据，这个读取过程会建立数据和渲染 effect 的依赖关系。后续当响应式数据变化时，Vue 会触发对应 effect 重新运行，生成新的虚拟 DOM，再和旧的虚拟 DOM 做 patch，最后更新真实 DOM。

这个回答比“Vue3 用 Proxy 实现响应式”更完整。

---

## 2. Vue 里 ref 和 reactive 的区别

### `ref`

适合：

```text
基本类型
单个值
需要整体替换的数据
组件内部简单状态
```

例如：

```ts
const loading = ref(false)
const keyword = ref('')
const count = ref(0)
```

### `reactive`

适合：

```text
对象
表单对象
复杂配置对象
多个字段有关联的状态
```

例如：

```ts
const form = reactive({
  username: '',
  password: '',
  remember: false
})
```

面试回答可以这样说：

> `ref` 更适合管理单值状态，访问时通过 `.value`。如果传入对象，也会被深层响应式转换。`reactive` 直接代理对象，适合表单、配置、查询条件这类结构化状态。项目里我通常对基础状态用 `ref`，对表单和查询参数用 `reactive`，对大体积且不需要深层响应式的数据会考虑 `shallowRef` 或普通变量。

这个回答里有一个高级点：**大体积数据不一定要深层响应式。**

Vue 官方性能建议也提到，Vue 响应式默认是深层的，大型不可变数据结构可能带来额外开销，可以用 `shallowRef()` / `shallowReactive()` 作为“浅层响应式”逃生口，但需要把内部对象按不可变方式处理。([Vue.js][2])

---

## 3. computed 和 watch 的区别

这是 Vue 高频题。

### computed

适合：

```text
由已有状态推导出新状态
有缓存
依赖不变就不重新计算
一般不做副作用
```

例如：

```ts
const totalPrice = computed(() => {
  return cartList.value.reduce((sum, item) => sum + item.price * item.count, 0)
})
```

Vue 官方文档明确说明，computed 会自动追踪响应式依赖，并且基于依赖缓存；依赖不变时，多次访问会直接返回之前的结果，而方法调用会在每次重新渲染时都执行。([Vue.js][3])

### watch

适合：

```text
监听一个状态变化
执行副作用
调接口
写缓存
操作 DOM
触发异步逻辑
```

例如：

```ts
watch(keyword, async (newVal) => {
  await fetchList(newVal)
})
```

面试回答：

> computed 是声明式的派生状态，强调“根据已有状态算出一个值”，并且有缓存。watch 更像命令式监听，强调“当某个状态变化后做一件事”，比如调接口、写缓存、埋点、同步外部系统。项目里我会优先用 computed 表达数据关系，只有需要副作用时才用 watch。

这句话非常重要：**能用 computed 表达的，不要滥用 watch。**

---

## 4. watch 和 watchEffect 的区别

### watch

特点：

```text
明确指定监听源
可以拿到新值和旧值
更适合精确控制
```

### watchEffect

特点：

```text
自动收集依赖
立即执行一次
依赖变化后重新执行
适合依赖不固定或简单副作用
```

面试可以这样讲：

> `watch` 更精确，适合监听指定字段，比如监听路由参数、搜索关键词、表单字段。`watchEffect` 会自动收集内部用到的响应式依赖，适合依赖关系比较自然的场景，但大型项目里如果逻辑复杂，我更倾向于使用 `watch`，因为依赖更明确，可维护性更好。

---

## 5. Vue 渲染机制：模板不是直接变 DOM

Vue 的模板会被编译成渲染函数。组件挂载时，运行渲染函数生成虚拟 DOM，再创建真实 DOM；当依赖变化时，渲染 effect 会重新执行，生成新的虚拟 DOM，然后和旧虚拟 DOM 对比并 patch 到真实 DOM。([Vue.js][4])

你可以按这个流程记：

```text
template
  ↓ 编译
render function
  ↓ 执行
VNode
  ↓ mount
真实 DOM
  ↓ 数据变化
重新生成 VNode
  ↓ diff / patch
局部更新真实 DOM
```

面试回答：

> Vue 的模板最终不是直接操作 DOM，而是编译成 render 函数。render 函数执行后会生成 VNode。响应式数据变化时，组件渲染 effect 重新执行，生成新的 VNode，然后和旧 VNode 做 patch，最后把必要的变化更新到真实 DOM。

---

## 6. Vue 的 diff 和 key

面试官经常问：

> v-for 为什么要加 key？为什么不能用 index？

你可以这样答：

> key 是 Vue 判断新旧节点是否是同一个节点的重要依据。稳定且唯一的 key 可以帮助 Vue 在列表更新时尽量复用正确的 DOM 和组件实例。如果用 index，当列表发生插入、删除、排序时，index 会变化，可能导致错误复用，尤其是表单输入、组件内部状态、动画列表这些场景更容易出问题。

举例：

```vue
<!-- 不推荐 -->
<Item v-for="(item, index) in list" :key="index" />

<!-- 推荐 -->
<Item v-for="item in list" :key="item.id" />
```

能说出“组件实例错误复用”，就比只说“性能不好”更准确。

---

## 7. Vue 状态管理：什么时候用 Pinia？

5年前端不能只说“全局状态就用 Pinia”。

你要能区分状态层级：

```text
组件内部状态：ref / reactive
父子共享状态：props / emits / v-model
跨层级共享状态：provide / inject
全局业务状态：Pinia
服务端缓存状态：请求缓存 / TanStack Query 类工具
URL 状态：route query / params
持久化状态：localStorage / IndexedDB
```

面试回答：

> 我不会所有状态都放到 Pinia。组件内部的 loading、弹窗开关、临时表单状态一般放在组件内。多个兄弟组件共享的业务状态，或者登录用户信息、权限、主题、全局配置、购物车这类跨页面状态，才适合放到 Pinia。对于接口列表数据，如果只是页面级缓存，我会优先考虑请求层缓存，而不是强行放进全局 store。

这个回答体现的是：**你知道状态应该放在离使用它最近的地方。**

---

## 8. Vue 性能优化怎么讲？

Vue 性能优化不要只背“懒加载、防抖、节流”。要分层讲：

### 第一层：加载性能

```text
路由懒加载
组件懒加载
第三方库按需引入
图片懒加载
资源压缩
CDN
减少首屏接口阻塞
```

### 第二层：更新性能

```text
减少无效响应式依赖
稳定 props
合理使用 computed
避免 watch 深度监听大对象
v-once
v-memo
大列表虚拟滚动
大数据用 shallowRef
```

Vue 官方性能建议里明确提到，大列表渲染是常见性能问题，即使框架本身很快，渲染成千上万个 DOM 节点仍然会慢；虚拟列表通过只渲染视口附近的项目来改善性能。([Vue.js][2])

### 第三层：组件设计性能

```text
不要过度抽象
不要在大列表里塞太多组件层级
不要每次 render 生成新对象传给子组件
不要让所有子项都依赖一个频繁变化的父状态
```

举个典型场景。

不太好的写法：

```vue
<ListItem
  v-for="item in list"
  :key="item.id"
  :item="item"
  :active-id="activeId"
/>
```

这里 `activeId` 一变，所有子项都可能更新。

更好的写法：

```vue
<ListItem
  v-for="item in list"
  :key="item.id"
  :item="item"
  :active="item.id === activeId"
/>
```

这样只有 active 状态真正变化的项更需要更新。

面试表达：

> 我做 Vue 性能优化时，会先区分是加载慢、接口慢、渲染慢还是交互慢。如果是大列表卡顿，我不会一上来就加防抖，而是先看 DOM 数量、组件层级、props 是否稳定、是否有深度 watch、大对象是否被深层响应式代理。比如几千行表格，我会优先做分页或虚拟滚动，再考虑 `v-memo`、`shallowRef`、列渲染简化等优化。

---

# 三、React 深入：核心是“状态变化触发重新渲染”

React 的核心思路可以简化成：

> 状态变化后，组件函数重新执行，生成新的 UI 描述，再由 React 决定如何更新真实 DOM。

React 面试更关注：

```text
为什么组件重新渲染？
重新渲染的边界在哪里？
props 变化如何影响子组件？
Hooks 为什么有依赖数组？
useMemo / useCallback / memo 什么时候有用？
状态应该放本地、Context、Redux 还是其他 store？
```

---

## 1. React 渲染机制怎么理解？

React 函数组件本质上就是一个函数：

```tsx
function UserCard({ user }) {
  return <div>{user.name}</div>
}
```

状态变化后，这个函数会重新执行。

你可以这样回答：

> React 的更新可以理解为：状态或 props 变化后，组件函数重新执行，返回新的 React Element 树。React 再通过协调过程比较新旧结果，决定真实 DOM 需要怎么变。React 的渲染不是等于 DOM 更新，render 阶段更多是计算 UI 描述，commit 阶段才会把变化提交到 DOM。

这是 React 面试里很重要的区分：

```text
render 阶段：计算新的 UI
commit 阶段：提交真实 DOM 更新和副作用
```

---

## 2. React 为什么会重新渲染？

常见原因：

```text
自身 state 改变
父组件重新渲染
props 改变
context value 改变
外部 store 订阅变化
```

这里有一个面试重点：

> 父组件重新渲染时，默认子组件也会重新渲染。

React 官方 useMemo 文档也说明，默认情况下，当一个组件重新渲染时，React 会递归重新渲染它的所有子组件；如果确认某个子组件重新渲染很慢，可以用 `memo` 在 props 相同的时候跳过渲染。([React][5])

所以 React 性能优化的核心之一就是：**控制重新渲染范围。**

---

## 3. useState 更新后为什么不能马上拿到新值？

例如：

```tsx
const [count, setCount] = useState(0)

function handleClick() {
  setCount(count + 1)
  console.log(count)
}
```

很多人会疑惑为什么打印还是旧值。

可以这样解释：

> React 的状态更新不是直接修改当前 render 闭包里的变量，而是通知 React 在下一次渲染中使用新状态。当前函数执行期间拿到的 `count` 来自这一次 render 的闭包，所以立刻打印仍然是旧值。

再进一步：

> 如果下一次状态依赖上一次状态，应该用函数式更新。

```tsx
setCount(prev => prev + 1)
```

面试官继续问：

> 连续 setCount 三次会怎样？

你就说：

```tsx
setCount(count + 1)
setCount(count + 1)
setCount(count + 1)
```

如果 `count` 是 0，这三次都基于当前闭包里的 0，结果通常是 1。

更推荐：

```tsx
setCount(prev => prev + 1)
setCount(prev => prev + 1)
setCount(prev => prev + 1)
```

这样结果是 3。

React 也会对状态更新做批处理，React 文档说明 React 可能会把多个 state 更新合并到一次重新渲染里以提升性能；React 18 起默认对更多更新场景启用批处理。([React][6])

---

## 4. useEffect 到底是干什么的？

很多人把 `useEffect` 当成“监听器”或者“生命周期替代品”，这不够准确。

React 官方文档强调：如果你不是要和某个外部系统同步，可能并不需要 Effect；`useEffect` 只能在组件或自定义 Hook 顶层调用，不能放在循环或条件语句里。([React][7])

面试时你可以说：

> `useEffect` 主要用于把 React 组件和外部系统同步，比如接口请求、订阅事件、操作定时器、连接 WebSocket、同步浏览器 API。它不是普通数据计算工具。如果只是根据 props 或 state 计算一个值，应该直接计算或使用 `useMemo`，而不是 `useEffect + setState`。

不推荐：

```tsx
const [fullName, setFullName] = useState('')

useEffect(() => {
  setFullName(firstName + lastName)
}, [firstName, lastName])
```

推荐：

```tsx
const fullName = firstName + lastName
```

或者计算比较重时：

```tsx
const fullName = useMemo(() => {
  return firstName + lastName
}, [firstName, lastName])
```

这个点非常容易加分，因为它说明你知道：**滥用 useEffect 会制造额外渲染链。**

---

## 5. useEffect 依赖数组怎么理解？

依赖数组不是“想不想执行”的开关，而是：

> Effect 内部使用了哪些响应式值，就应该声明哪些依赖。

React 官方文档说明，依赖项包括 props、state，以及组件函数体内声明并被 Effect 使用的变量和函数；React 会用 `Object.is` 比较依赖项变化。([React][7])

面试回答：

> 我不会随便删依赖来阻止 effect 执行。依赖缺失可能导致闭包拿到旧值。正确做法通常是调整代码结构，比如把不需要响应式的逻辑移出 effect，把事件逻辑放进事件处理函数，或者用函数式更新减少依赖。

常见坑：

```tsx
useEffect(() => {
  fetchUser(userId)
}, [])
```

如果 `userId` 可能变化，这就是错误的。

应该：

```tsx
useEffect(() => {
  fetchUser(userId)
}, [userId])
```

---

## 6. useMemo、useCallback、React.memo 到底怎么用？

这三个经常被混淆。

### React.memo

作用：

```text
缓存组件渲染结果
props 没变时跳过子组件重新渲染
```

### useMemo

作用：

```text
缓存计算结果
避免昂贵计算重复执行
保持对象/数组引用稳定
```

React 官方文档说明，`useMemo` 适合计算明显较慢且依赖很少变化的场景，或者把结果传给被 `memo` 包裹的子组件以跳过重新渲染；否则滥用会降低代码可读性，而且一个“总是新的”值就可能破坏整个 memoization。([React][5])

### useCallback

作用：

```text
缓存函数引用
主要用于传给 memo 子组件
或者作为其他 Hook 的依赖
```

React 官方文档说明，`useCallback` 缓存的是函数本身，`useMemo` 缓存的是函数调用结果；`useCallback` 应该作为性能优化使用，如果没有它代码就不能工作，应该先修复底层问题。([React][8])

面试时不要说：

> useCallback 可以防止函数创建。

更准确是：

> 函数每次 render 仍然会创建，但 React 在依赖不变时返回缓存的函数引用。它的价值主要是让传给 memo 子组件的函数 props 保持引用稳定。

示例：

```tsx
const Child = memo(function Child({ onSave }) {
  return <button onClick={onSave}>Save</button>
})

function Parent({ id }) {
  const handleSave = useCallback(() => {
    save(id)
  }, [id])

  return <Child onSave={handleSave} />
}
```

但是要注意：**不是所有函数都需要 useCallback。**

面试表达：

> 我不会全项目无脑加 `useMemo` 和 `useCallback`。我会先用 React DevTools Profiler 看具体是哪一块慢。如果是子组件重渲染导致的，并且 props 里有对象、数组或函数引用不稳定，我才会配合 `memo`、`useMemo`、`useCallback` 做优化。

这个回答非常高级。

---

## 7. React 状态管理怎么选？

React 状态管理要按状态类型拆。

```text
组件内部状态：useState / useReducer
跨层级但低频状态：Context
复杂业务全局状态：Redux / Zustand / MobX / Jotai
服务端数据状态：React Query / SWR
表单状态：React Hook Form / Formik
URL 状态：路由 query / params
不触发渲染的可变值：useRef
```

面试官问：

> Redux、Context、Zustand 怎么选？

可以这样说：

> Context 更适合主题、语言、用户信息这类低频变化的全局配置。如果是高频更新或复杂业务状态，单纯 Context 容易导致较大范围更新。Redux 更适合大型团队、复杂状态流、需要严格规范和可追踪状态变化的项目。Zustand 使用成本更低，适合中小型项目或模块级 store。服务端列表数据我一般不直接放 Redux，而是用请求缓存工具管理，因为它涉及缓存、失效、重试、刷新、分页等服务端状态问题。

这说明你知道一个关键点：

> 客户端状态和服务端状态不是一回事。

---

# 四、React 和 Vue 的核心区别

这是5年前端很容易被问的。

## 1. Vue 更强调响应式依赖追踪

Vue 是：

```text
数据被读取时收集依赖
数据变化时精确触发相关更新
模板编译器还能做静态分析优化
```

Vue 官方渲染机制文档也提到，Vue 控制编译器和运行时，可以通过模板静态分析给运行时留下优化提示，这种方式被称为 Compiler-Informed Virtual DOM。([Vue.js][4])

## 2. React 更强调重新执行组件函数

React 是：

```text
状态变化后组件函数重新执行
通过 reconciliation 判断如何更新 UI
开发者通过 memo、useMemo、useCallback 控制部分重渲染
```

所以二者性能优化思路不完全一样。

Vue 更常见的优化点：

```text
避免过度响应式
稳定 props
computed 缓存
v-memo
shallowRef
虚拟列表
减少深度 watch
```

React 更常见的优化点：

```text
减少状态提升
拆分组件边界
memo
useMemo
useCallback
useRef
避免无效 effect
避免 Context 大范围更新
虚拟列表
```

---

# 五、组件设计：5年前端最该讲出深度的地方

组件设计是面试里区分“业务熟手”和“高级前端”的关键。

很多人组件封装是这样的：

```text
复制一个页面
把重复代码抽成组件
props 传一堆
哪里不够再加字段
最后组件越来越难维护
```

高级一点的组件设计应该考虑：

```text
职责边界
数据输入
事件输出
插槽/children 扩展
默认行为
受控/非受控
错误状态
loading 状态
空状态
权限
可测试性
类型约束
可组合性
```

---

## 1. 组件分层

建议你面试时这样讲：

```text
基础组件：Button、Input、Select、Modal
业务组件：UserSelect、DepartmentTree、PermissionTable
页面组件：具体业务页面
逻辑组件：hooks / composables
布局组件：PageContainer、SearchPanel、Toolbar
```

Vue 里对应：

```text
components
composables
directives
stores
views
```

React 里对应：

```text
components
hooks
stores
pages
services
layouts
```

面试表达：

> 我一般会把组件分成基础组件、业务组件和页面组件。基础组件尽量不绑定业务，业务组件承载业务语义，页面组件负责数据组装和流程编排。复杂逻辑不直接堆在组件里，而是抽到 hooks 或 composables 里，这样组件更专注于渲染。

---

## 2. 组件应该受控还是非受控？

### 受控组件

外部传值，外部控制：

```tsx
<Input value={value} onChange={setValue} />
```

适合：

```text
表单
筛选条件
弹窗开关
需要外部统一控制的状态
```

### 非受控组件

组件内部自己管理状态：

```tsx
<Upload defaultFileList={list} />
```

适合：

```text
内部临时状态
不需要外部实时感知的交互
```

面试表达：

> 通用组件我会优先支持受控模式，因为它更容易被外部表单、状态管理和业务流程接管。但对于一些内部交互，比如展开收起、hover、临时输入，也可以支持非受控，降低使用成本。高级组件最好同时支持受控和非受控。

---

## 3. 表格组件怎么设计？

这是后台系统高频题。

你可以这样讲：

```text
columns 配置列
request 负责数据请求
pagination 负责分页
toolbar 负责顶部操作
rowActions 负责行操作
selection 负责批量操作
permission 控制按钮
slots / render 扩展单元格
loading / empty / error 做状态兜底
```

示例结构：

```ts
type Column = {
  title: string
  dataIndex: string
  width?: number
  render?: (value: any, record: any) => ReactNode
  permission?: string
}
```

面试表达：

> 我封装表格时不会只封一层 UI，而是把查询、分页、loading、空状态、错误状态、权限按钮、操作列、单元格自定义都考虑进去。这样普通 CRUD 页面只需要配置 columns 和 request，复杂页面也能通过 slot 或 render 扩展。

---

## 4. 动态表单怎么设计？

动态表单是最能体现高级前端能力的题。

核心设计：

```text
schema 描述字段
component 指定组件类型
rules 描述校验
visible 控制显示隐藏
disabled 控制禁用
options 支持同步/异步
dependencies 控制联动
transform 处理提交转换
initialValue 处理默认值
```

示例：

```ts
const schema = [
  {
    field: 'username',
    label: '用户名',
    component: 'Input',
    rules: [{ required: true, message: '请输入用户名' }]
  },
  {
    field: 'role',
    label: '角色',
    component: 'Select',
    options: async () => fetchRoles()
  }
]
```

面试表达：

> 动态表单的关键不是把组件循环出来，而是处理字段联动、异步数据源、校验规则、默认值、回显转换、提交转换和自定义扩展。我的设计会让 schema 只描述“是什么”，具体渲染、校验和提交逻辑由表单引擎统一处理。

---

# 六、性能优化：面试要讲“定位 → 原因 → 方案 → 结果”

5年前端不能只说：

```text
我用了懒加载
我用了防抖
我用了缓存
```

要这样讲：

```text
先定位是哪类慢
再判断瓶颈在哪里
然后选择具体方案
最后说优化结果
```

---

## 1. 性能问题分类

```text
加载慢：首屏白屏、资源大、接口慢
渲染慢：DOM 太多、组件太重、diff 成本高
交互慢：输入卡顿、滚动掉帧、点击延迟
内存问题：页面越用越卡、监听器没清理
网络问题：接口重复、请求阻塞、无缓存
```

---

## 2. Vue 项目性能案例怎么讲？

示例回答：

> 我之前遇到过一个后台列表页卡顿问题，页面一次性渲染几千条数据，每一行还有多个下拉、按钮和状态标签。最开始以为是接口慢，后来用 Performance 和 Vue Devtools 看，发现主要瓶颈是 DOM 数量和组件实例过多。
>
> 后来我做了几步优化：第一，把全量渲染改成分页和虚拟滚动；第二，把不需要深层响应式的大列表数据改成 `shallowRef`；第三，减少行组件内部不必要的 watch；第四，把 active 状态提前计算成稳定 props，避免父状态变化导致所有行更新。最后滚动和搜索明显流畅很多。

这个回答有深度，因为它说出了：

```text
不是拍脑袋优化
是先定位瓶颈
再针对 DOM、响应式、组件更新做处理
```

---

## 3. React 项目性能案例怎么讲？

示例回答：

> 我做过一个配置页面，左侧树、右侧表单、下方表格都依赖同一个大对象。最开始每次输入表单字段，整个页面都会重新渲染，表格和树也跟着卡。
>
> 我先用 React DevTools Profiler 定位，发现状态被提升得太高，导致很多无关子组件跟着 render。后面把临时表单状态下沉到表单组件内部，把真正需要共享的状态保留在父层；对重型表格组件使用 `memo`；传给子组件的 columns 和 handlers 用 `useMemo`、`useCallback` 保持引用稳定；另外把一些不触发渲染的临时值放到 `useRef`。优化后，输入框卡顿明显减少。

这个回答也有面试价值，因为它说出了 React 优化核心：

```text
状态下沉
减少无关重渲染
memo 配合稳定 props
不要滥用全局状态
```

---

# 七、面试官常见深挖问题

## 1. Vue 为什么有时候数据变了页面没变？

可能原因：

```text
修改的不是响应式数据
ref 忘了 .value
解构 reactive 后丢失响应式
shallowRef 内部修改不会触发
数组/对象被错误替换或引用不稳定
key 使用错误导致组件复用异常
```

高级回答：

> 我会先确认数据本身是不是响应式，再看修改方式是否能触发响应式更新。如果是 `reactive` 解构导致的响应式丢失，可以用 `toRefs`。如果是 `shallowRef`，内部对象变化不会触发更新，需要整体替换 `.value`。如果数据已经更新但视图异常，再检查 key 和组件复用问题。

---

## 2. React 为什么有时候状态变了 UI 没按预期变？

可能原因：

```text
直接修改 state 对象
闭包拿到旧值
key 错误导致组件复用
memo 比较阻止更新
useEffect 依赖缺失
Context value 引用问题
异步请求竞态
```

高级回答：

> React 状态更新依赖引用变化。如果直接修改对象再 set 回原对象，React 可能认为状态没有变化。对于依赖旧状态的新状态，我会用函数式更新。对于 effect 里拿到旧值的问题，我会检查依赖数组或改用 ref 保存最新值。对于列表异常，我会优先检查 key 是否稳定。

---

## 3. Vue 和 React 组件通信分别有哪些？

Vue：

```text
props / emits
v-model
provide / inject
Pinia
event bus，不太推荐
路由参数
本地缓存
```

React：

```text
props
callback
children
Context
state management library
custom hooks
URL params
external store
```

面试不要只罗列，要补一句：

> 我会优先选择最简单、作用域最小的通信方式。父子组件用 props 和事件，跨层级低频状态用 Context 或 provide/inject，真正跨页面共享的业务状态才放全局 store。

---

## 4. 为什么不建议所有状态都放全局 store？

回答：

> 全局状态会增加隐式依赖，让组件复用和测试变难。很多状态其实只是页面内部临时状态，比如弹窗开关、表单输入、hover 状态、loading 状态。如果全部放全局，状态来源会变复杂，也容易导致无关组件更新。我的原则是状态放在离使用它最近的位置，只有确实跨组件、跨页面共享，或者需要持久化和统一管理时，才上全局 store。

---

# 八、5年前端应该准备的框架知识清单

## Vue 必备清单

```text
Vue2 / Vue3 响应式区别
Proxy 和 Object.defineProperty 区别
ref / reactive / shallowRef / readonly
computed / watch / watchEffect
nextTick
组件渲染机制
模板编译
虚拟 DOM
diff / patch
key 的作用
组件通信
slot / scoped slot
v-model 原理
自定义指令
自定义 composable
Pinia 设计
Vue Router 权限
keep-alive
Teleport
Suspense
性能优化
动态表单设计
组件库封装
```

---

## React 必备清单

```text
函数组件渲染机制
state 更新机制
批处理
闭包问题
Hooks 规则
useState
useEffect
useMemo
useCallback
useRef
useReducer
useContext
custom hooks
React.memo
受控 / 非受控组件
Context 性能问题
状态管理选型
React Router
列表 key
Fiber 基本概念
render / commit
性能优化
错误边界
Suspense
Next.js 基础
组件设计
```

---

# 九、面试时如何把“框架深入”讲得像5年经验？

你可以用这个表达模板：

```text
我理解这个问题分三层：

第一层是用法，比如这个 API 怎么写；
第二层是机制，比如它为什么能触发更新，什么时候会重新渲染；
第三层是项目落地，比如在复杂表单、大列表、权限系统、动态配置里怎么设计，怎么避免性能问题。

我在项目里通常会先根据状态作用域决定状态放在哪里，再根据组件职责拆分结构。如果遇到性能问题，会先用 DevTools / Performance 定位是网络、渲染、脚本还是组件重渲染问题，再决定是做缓存、虚拟列表、状态下沉、memo、shallowRef，还是拆分组件。
```

这段很适合作为面试总起手。

---

# 十、一句话总结

React / Vue 框架深入，不是背 API，而是掌握这条主线：

```text
状态如何变化
依赖如何收集
组件如何渲染
更新如何发生
性能如何失控
架构如何控制复杂度
```

面试官真正想确认的是：**你能不能从框架机制出发，解释项目里的技术决策。**

对于5年前端，最加分的表达不是“我会 Vue3 / React Hooks”，而是：

> 我知道这个状态为什么放这里，知道这个组件为什么这样拆，知道页面为什么会重新渲染，也知道卡顿时该从哪里定位。

