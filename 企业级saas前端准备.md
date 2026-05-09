可以把这个项目包装成一句话：

**我负责的是企业级 SaaS 后台平台的前端工程化和中后台通用能力建设，核心不是单纯写页面，而是把权限、动态表单、通用表格、请求层、性能优化、工程规范沉淀成可复用的平台能力，支撑多角色、多租户、多业务线快速配置和迭代。**

下面按模块详细展开。

---

# 一、权限系统

权限系统是后台管理项目里最容易体现“架构能力”的模块。面试时不要只说“我做了路由权限”，要分成：

**路由权限、菜单权限、按钮权限、接口权限、角色配置、权限刷新、异常兜底。**

## 1. 整体权限流程

典型流程是：

用户登录
→ 获取 token
→ 拉取用户信息
→ 获取角色、权限码、菜单树
→ 根据后端返回的菜单或权限码生成动态路由
→ 渲染左侧菜单
→ 页面内通过权限码控制按钮显示
→ 请求层根据接口错误码处理 401 / 403
→ 退出登录清空权限和路由

你可以这样讲：

> 我们系统的权限不是写死在前端，而是由后端维护角色和权限点。用户登录后，前端拿到 token，再请求用户信息和权限菜单。前端会根据后端返回的菜单树动态生成路由，同时把按钮权限码存入 Pinia，全局通过指令或者工具函数控制按钮显示。接口权限由后端兜底，前端主要做体验层控制，避免用户看到无权限入口。

## 2. 动态路由

动态路由有两种常见方案。

### 方案一：后端返回完整路由配置

后端返回：

```js
[
  {
    path: "/system",
    name: "System",
    component: "Layout",
    meta: {
      title: "系统管理",
      icon: "setting"
    },
    children: [
      {
        path: "user",
        name: "SystemUser",
        component: "system/user/index",
        meta: {
          title: "用户管理",
          permission: "system:user:list"
        }
      }
    ]
  }
]
```

前端根据 `component` 字符串映射组件：

```js
const modules = import.meta.glob("@/views/**/*.vue")

function loadView(component) {
  return modules[`/src/views/${component}.vue`]
}
```

然后递归生成路由：

```js
function transformRoutes(menuList) {
  return menuList.map(item => {
    const route = {
      path: item.path,
      name: item.name,
      meta: item.meta || {},
      component: item.component === "Layout"
        ? Layout
        : loadView(item.component)
    }

    if (item.children?.length) {
      route.children = transformRoutes(item.children)
    }

    return route
  })
}
```

最后：

```js
dynamicRoutes.forEach(route => {
  router.addRoute(route)
})
```

### 方案二：前端维护静态路由表，后端只返回权限码

前端维护所有异步路由：

```js
const asyncRoutes = [
  {
    path: "/system",
    name: "System",
    meta: {
      title: "系统管理",
      permission: "system"
    },
    children: [
      {
        path: "user",
        name: "SystemUser",
        component: () => import("@/views/system/user/index.vue"),
        meta: {
          title: "用户管理",
          permission: "system:user:list"
        }
      }
    ]
  }
]
```

然后根据权限码过滤：

```js
function filterRoutes(routes, permissions) {
  return routes.filter(route => {
    const permission = route.meta?.permission
    const hasPermission = !permission || permissions.includes(permission)

    if (route.children?.length) {
      route.children = filterRoutes(route.children, permissions)
    }

    return hasPermission
  })
}
```

面试时可以说：

> 如果业务权限变化频繁，我更倾向于后端返回菜单树，前端做组件映射；如果项目稳定、前端希望类型更安全，我会用前端完整路由表加权限码过滤。我们项目采用的是后端菜单配置 + 前端组件白名单映射，兼顾灵活性和安全性。

## 3. 菜单权限

菜单通常来自动态路由或后端菜单树。

关键点：

```js
function generateMenus(routes) {
  return routes
    .filter(route => !route.meta?.hidden)
    .map(route => ({
      title: route.meta?.title,
      icon: route.meta?.icon,
      path: route.path,
      children: route.children ? generateMenus(route.children) : []
    }))
}
```

注意点：

1. 菜单不等于路由。
2. 有些路由需要存在，但不展示在菜单里，比如详情页、编辑页。
3. 面包屑、标签页、菜单高亮都依赖 `meta` 配置。
4. 动态路由刷新页面时，要先恢复权限再放行路由。

面试高频追问：

**刷新页面后动态路由丢失怎么办？**

你可以这样回答：

> 动态路由是运行时通过 `addRoute` 注册的，刷新后内存会丢失。所以我们在路由守卫里判断权限路由是否已经生成。如果没有生成，就先用 token 请求用户权限，重新生成动态路由，再通过 `next({ ...to, replace: true })` 重新进入目标页面，避免 404。

## 4. 按钮权限

按钮权限一般用权限码控制。

例如：

```js
permissions = [
  "system:user:add",
  "system:user:edit",
  "system:user:delete",
  "order:export"
]
```

### 工具函数方式

```js
function hasPermission(code) {
  const userStore = useUserStore()
  return userStore.permissions.includes(code)
}
```

页面使用：

```vue
<el-button v-if="hasPermission('system:user:add')">
  新增
</el-button>
```

### 自定义指令方式

```js
app.directive("permission", {
  mounted(el, binding) {
    const userStore = useUserStore()
    const code = binding.value

    if (!userStore.permissions.includes(code)) {
      el.parentNode?.removeChild(el)
    }
  }
})
```

页面使用：

```vue
<el-button v-permission="'system:user:add'">
  新增
</el-button>
```

更灵活的方式支持数组：

```vue
<el-button v-permission="['system:user:add', 'system:user:edit']">
  保存
</el-button>
```

可以支持 `some` 或 `every` 策略。

面试时可以强调：

> 按钮权限只是前端体验控制，不能替代后端接口鉴权。比如用户手动调接口，后端仍然要判断角色和权限，否则前端隐藏按钮没有任何安全意义。

## 5. 接口权限

接口权限必须由后端控制。前端主要做：

1. 401：未登录或 token 过期。
2. 403：无权限访问。
3. 404：接口不存在。
4. 业务错误码：统一提示。
5. 特殊错误码：跳转无权限页。

例如：

```js
if (code === 401) {
  userStore.logout()
  router.replace("/login")
}

if (code === 403) {
  router.replace("/403")
}
```

面试可以说：

> 前端不会把接口权限当成安全边界，真正的接口权限由后端网关或服务端鉴权完成。前端只负责控制入口和优化交互体验。

## 6. 角色配置

企业级后台常见关系是：

用户
→ 角色
→ 权限
→ 菜单 / 按钮 / 接口

更复杂的 SaaS 会有：

租户
→ 组织
→ 用户
→ 角色
→ 数据权限
→ 功能权限

可以这样包装：

> 我们的权限模型支持 RBAC。一个用户可以绑定多个角色，一个角色可以绑定多个菜单和按钮权限。前端并不直接维护角色逻辑，而是消费后端计算后的权限结果。这样可以避免前端处理复杂角色合并，也能保证权限统一由后端控制。

## 7. 数据权限

这是加分点。

功能权限决定“能不能看这个页面、能不能点这个按钮”。

数据权限决定“能看到哪些数据”。

比如：

1. 只能看自己创建的数据。
2. 可以看本部门数据。
3. 可以看本部门及下级部门数据。
4. 可以看全部数据。
5. SaaS 场景下只能看当前租户数据。

前端通常不会计算数据权限，但会在请求中携带：

```js
tenantId
orgId
departmentId
```

或者由 token 中的上下文决定。

面试说法：

> 数据权限没有放在前端判断，因为前端不可信。前端最多负责切换租户、组织、部门筛选条件，真正的数据隔离由后端根据 token 和租户上下文处理。

---

# 二、动态表单

动态表单的核心是：

**通过 schema 描述表单，而不是每个页面手写一堆重复组件。**

面试时不要只说“封装了 Form 组件”，要讲：

**schema 配置、组件映射、字段联动、异步数据源、校验规则、回显转换、提交转换、插槽扩展。**

## 1. schema 配置

一个典型 schema：

```js
const formSchema = [
  {
    field: "username",
    label: "用户名",
    component: "Input",
    required: true,
    props: {
      placeholder: "请输入用户名"
    }
  },
  {
    field: "roleId",
    label: "角色",
    component: "Select",
    required: true,
    asyncOptions: async () => {
      return await getRoleList()
    },
    props: {
      placeholder: "请选择角色"
    }
  },
  {
    field: "status",
    label: "状态",
    component: "Switch",
    defaultValue: true
  }
]
```

动态表单组件只关心：

1. 渲染什么组件。
2. 字段名是什么。
3. label 是什么。
4. 校验规则是什么。
5. 组件 props 是什么。
6. 是否显示。
7. 是否禁用。
8. options 从哪里来。

## 2. 组件映射

```js
const componentMap = {
  Input: ElInput,
  Select: ElSelect,
  DatePicker: ElDatePicker,
  Switch: ElSwitch,
  Radio: ElRadioGroup,
  Checkbox: ElCheckboxGroup
}
```

渲染时：

```vue
<component
  :is="componentMap[item.component]"
  v-model="formModel[item.field]"
  v-bind="item.props"
/>
```

如果是 Select：

```vue
<el-option
  v-for="option in item.options"
  :key="option.value"
  :label="option.label"
  :value="option.value"
/>
```

## 3. 默认值处理

初始化时根据 schema 生成 model：

```js
function createFormModel(schema) {
  const model = {}

  schema.forEach(item => {
    model[item.field] = item.defaultValue ?? undefined
  })

  return model
}
```

面试说法：

> 动态表单初始化时，我不会在页面里手写 model，而是根据 schema 自动生成初始值，这样新增字段只需要改 schema，不需要同时改 template、model、rules 多个地方。

## 4. 校验规则

schema 中配置：

```js
{
  field: "phone",
  label: "手机号",
  component: "Input",
  rules: [
    { required: true, message: "请输入手机号", trigger: "blur" },
    { pattern: /^1\d{10}$/, message: "手机号格式不正确", trigger: "blur" }
  ]
}
```

也可以自动生成 required：

```js
function createRules(schema) {
  const rules = {}

  schema.forEach(item => {
    const fieldRules = []

    if (item.required) {
      fieldRules.push({
        required: true,
        message: `${item.label}不能为空`,
        trigger: item.trigger || "blur"
      })
    }

    if (item.rules) {
      fieldRules.push(...item.rules)
    }

    rules[item.field] = fieldRules
  })

  return rules
}
```

## 5. 字段联动

字段联动是动态表单里最容易体现复杂度的点。

例如：

选择省份后，城市列表变化。

```js
{
  field: "provinceId",
  label: "省份",
  component: "Select",
  asyncOptions: getProvinceList,
  onChange: async (value, model, schemaApi) => {
    model.cityId = undefined
    const cityOptions = await getCityList(value)
    schemaApi.updateField("cityId", {
      options: cityOptions
    })
  }
},
{
  field: "cityId",
  label: "城市",
  component: "Select",
  options: []
}
```

更通用的设计可以是：

```js
{
  field: "cityId",
  label: "城市",
  component: "Select",
  dependencies: ["provinceId"],
  asyncOptions: async (model) => {
    if (!model.provinceId) return []
    return await getCityList(model.provinceId)
  }
}
```

然后监听依赖字段：

```js
watch(
  () => schema.map(item => item.dependencies?.map(dep => formModel[dep])),
  () => {
    reloadDependentOptions()
  },
  { deep: true }
)
```

面试可以说：

> 我们没有把字段联动写死在页面里，而是在 schema 里声明依赖关系。比如城市字段依赖省份字段，当前端监听到省份变化，会自动清空城市值并重新加载城市数据。这样多个业务表单都可以复用同一套联动机制。

## 6. 显示隐藏联动

例如：

当用户类型为企业时，显示企业名称。

```js
{
  field: "companyName",
  label: "企业名称",
  component: "Input",
  visible: model => model.userType === "company"
}
```

渲染时：

```js
function isVisible(item, model) {
  if (typeof item.visible === "function") {
    return item.visible(model)
  }

  return item.visible !== false
}
```

注意：

隐藏字段是否提交，要提前设计。

两种策略：

1. 隐藏但保留值。
2. 隐藏后清空值。

面试可以说：

> 我们对隐藏字段做了策略控制，有些场景只是 UI 隐藏但提交时仍保留值，有些场景隐藏后必须清空，避免提交脏数据。所以 schema 里支持 `clearWhenHidden` 配置。

## 7. 异步数据源

比如角色列表、部门树、字典项。

```js
{
  field: "roleId",
  label: "角色",
  component: "Select",
  dataSource: {
    api: getRoleList,
    labelField: "roleName",
    valueField: "id"
  }
}
```

统一转换：

```js
async function loadOptions(item) {
  const res = await item.dataSource.api()

  item.options = res.map(row => ({
    label: row[item.dataSource.labelField],
    value: row[item.dataSource.valueField]
  }))
}
```

可以加缓存：

```js
const optionCache = new Map()

async function loadOptionsWithCache(key, fetcher) {
  if (optionCache.has(key)) {
    return optionCache.get(key)
  }

  const data = await fetcher()
  optionCache.set(key, data)

  return data
}
```

适合面试表达：

> 字典类数据我会做缓存，比如状态、类型、角色这种低频变化的数据，避免每次打开弹窗都重复请求。对于强实时数据，比如用户列表、商品列表，则不做永久缓存，或者设置短期缓存时间。

## 8. 回显转换

编辑页面最常见问题：

后端返回：

```js
{
  startTime: "2026-05-01 00:00:00",
  endTime: "2026-05-09 23:59:59"
}
```

前端表单需要：

```js
dateRange: ["2026-05-01 00:00:00", "2026-05-09 23:59:59"]
```

schema 可以配置：

```js
{
  field: "dateRange",
  label: "时间范围",
  component: "DateRange",
  transformIn: data => [data.startTime, data.endTime],
  transformOut: value => ({
    startTime: value?.[0],
    endTime: value?.[1]
  })
}
```

面试说法：

> 动态表单最容易出问题的是后端数据结构和前端组件结构不一致。比如后端是 startTime、endTime，前端 DateRange 需要数组。所以我们在 schema 里设计了 transformIn 和 transformOut，分别处理回显转换和提交转换，避免每个页面重复写转换逻辑。

## 9. 提交转换

提交前统一处理：

```js
function transformSubmitData(schema, model) {
  const result = {}

  schema.forEach(item => {
    const value = model[item.field]

    if (item.transformOut) {
      Object.assign(result, item.transformOut(value, model))
    } else {
      result[item.field] = value
    }
  })

  return result
}
```

## 10. 动态表单面试总结话术

你可以这样说：

> 我们的动态表单是 schema-driven 的设计。页面只维护字段配置，不直接写大量模板。表单组件内部根据 schema 自动生成 model、rules，并通过 componentMap 渲染不同控件。复杂场景下支持字段联动、异步数据源、显示隐藏、回显转换和提交转换。这样新增一个业务表单时，大部分情况下只需要写 schema 和接口，不需要重复写表单布局、校验、提交逻辑。

---

# 三、通用表格

通用表格是后台系统最高频模块。面试时建议包装成：

**ProTable / BasicTable / CRUD Table 组件。**

核心能力：

查询、分页、排序、操作列、权限按钮、导出、列配置、插槽扩展、批量操作、数据缓存。

## 1. 通用表格整体结构

页面通常由三部分组成：

1. 查询表单。
2. 表格主体。
3. 分页和操作按钮。

可以封装为：

```vue
<ProTable
  :columns="columns"
  :request="getUserList"
  :search-schema="searchSchema"
  row-key="id"
/>
```

页面只配置：

```js
const columns = [
  {
    prop: "username",
    label: "用户名",
    minWidth: 120
  },
  {
    prop: "phone",
    label: "手机号",
    minWidth: 140
  },
  {
    prop: "status",
    label: "状态",
    dict: "user_status"
  },
  {
    prop: "action",
    label: "操作",
    fixed: "right",
    width: 180,
    actions: [
      {
        label: "编辑",
        permission: "system:user:edit",
        onClick: row => openEdit(row)
      },
      {
        label: "删除",
        permission: "system:user:delete",
        danger: true,
        onClick: row => handleDelete(row)
      }
    ]
  }
]
```

## 2. 查询条件

查询表单也可以用动态表单 schema。

```js
const searchSchema = [
  {
    field: "username",
    label: "用户名",
    component: "Input"
  },
  {
    field: "status",
    label: "状态",
    component: "Select",
    options: [
      { label: "启用", value: 1 },
      { label: "禁用", value: 0 }
    ]
  }
]
```

查询时合并分页参数：

```js
const queryParams = reactive({
  pageNum: 1,
  pageSize: 10
})

async function loadData() {
  const res = await props.request({
    ...searchModel,
    ...queryParams,
    ...sortParams
  })

  tableData.value = res.list
  total.value = res.total
}
```

点击查询：

```js
function handleSearch() {
  queryParams.pageNum = 1
  loadData()
}
```

点击重置：

```js
function handleReset() {
  resetSearchModel()
  queryParams.pageNum = 1
  loadData()
}
```

面试重点：

> 查询时需要把页码重置为第一页，否则用户在第 8 页筛选数据，可能直接显示空数据，这是后台项目里很常见的体验问题。

## 3. 分页

分页封装重点：

```vue
<el-pagination
  v-model:current-page="queryParams.pageNum"
  v-model:page-size="queryParams.pageSize"
  :total="total"
  layout="total, sizes, prev, pager, next, jumper"
  @change="loadData"
/>
```

要处理：

1. 当前页变化。
2. 每页条数变化。
3. 删除当前页最后一条数据后，页码回退。
4. 查询条件变化后回到第一页。

删除后：

```js
if (tableData.value.length === 1 && queryParams.pageNum > 1) {
  queryParams.pageNum -= 1
}

loadData()
```

## 4. 排序

Element Plus 表格排序：

```vue
<el-table
  @sort-change="handleSortChange"
>
```

```js
function handleSortChange({ prop, order }) {
  sortParams.sortField = prop
  sortParams.sortOrder = order === "ascending" ? "asc" : "desc"
  loadData()
}
```

后端排序要注意字段映射：

```js
const sortFieldMap = {
  createTime: "create_time",
  username: "username"
}
```

避免直接把前端字段传给后端造成不一致。

## 5. 操作列

操作列要支持：

1. 权限判断。
2. 条件显示。
3. 二次确认。
4. loading。
5. 自定义渲染。

```js
{
  label: "删除",
  permission: "system:user:delete",
  visible: row => row.status !== "admin",
  confirm: "确认删除该用户吗？",
  onClick: async row => {
    await deleteUser(row.id)
    loadData()
  }
}
```

渲染时：

```vue
<template v-for="action in getVisibleActions(row)">
  <el-button
    v-if="hasPermission(action.permission)"
    :type="action.type || 'primary'"
    link
    @click="handleAction(action, row)"
  >
    {{ action.label }}
  </el-button>
</template>
```

面试说法：

> 操作列不是简单写死几个按钮，而是配置化生成。每个 action 支持权限码、显示条件、二次确认和回调函数。这样不同业务表格只需要声明 actions，不需要重复写操作列模板。

## 6. 权限按钮

结合权限系统：

```js
function getVisibleActions(row) {
  return actions.filter(action => {
    const permissionOk = !action.permission || hasPermission(action.permission)
    const visibleOk = !action.visible || action.visible(row)

    return permissionOk && visibleOk
  })
}
```

## 7. 导出

导出有两种：

### 前端导出当前页

适合数据量小：

```js
exportCurrentPage(tableData.value)
```

### 后端导出全部符合条件的数据

更常见：

```js
async function handleExport() {
  await exportUserList({
    ...searchModel,
    ...sortParams
  })
}
```

注意：

导出应该使用当前查询条件，但通常不带分页参数。

面试可以说：

> 我们的导出不是只导出当前页，而是把当前查询条件传给后端，由后端生成 Excel。这样用户筛选后的全量数据都能导出，同时避免前端一次性加载大量数据导致卡顿。

## 8. 列配置

用户可以控制列显示隐藏、顺序、宽度。

```js
const columnSettings = [
  {
    prop: "username",
    label: "用户名",
    visible: true
  },
  {
    prop: "phone",
    label: "手机号",
    visible: false
  }
]
```

保存到 localStorage：

```js
localStorage.setItem(
  `table-columns:${route.name}`,
  JSON.stringify(columnSettings)
)
```

也可以保存到后端，实现跨设备同步。

面试说法：

> 列配置我做了本地持久化，根据路由 name 作为 key 保存到 localStorage。对于管理端用户来说，不同页面的列显示习惯不同，所以不能全局共用一个配置。

## 9. 字典转换

后端返回：

```js
status: 1
```

前端显示：

```js
启用
```

列配置：

```js
{
  prop: "status",
  label: "状态",
  dict: "user_status"
}
```

统一渲染：

```js
function formatDict(dictType, value) {
  const options = dictStore.getDict(dictType)
  return options.find(item => item.value === value)?.label || value
}
```

## 10. 通用表格面试总结话术

> 通用表格主要解决后台页面重复 CRUD 的问题。我把查询表单、分页、排序、loading、导出、列配置、操作列权限都封装到 ProTable 中。业务页面只需要提供 columns、searchSchema 和 request 方法。这样可以减少重复代码，也能保证不同页面的交互体验统一，比如查询自动回到第一页、删除最后一条自动回退页码、导出复用当前查询条件等。

---

# 四、请求封装

请求封装是体现工程质量的核心模块。

重点讲：

**axios 实例、token 注入、token 刷新、错误码处理、取消重复请求、接口重试、loading 合并、文件下载、请求日志。**

## 1. axios 实例

```js
const service = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 15000
})
```

请求拦截器：

```js
service.interceptors.request.use(config => {
  const token = getToken()

  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }

  return config
})
```

响应拦截器：

```js
service.interceptors.response.use(
  response => {
    const res = response.data

    if (res.code !== 0) {
      handleBusinessError(res)
      return Promise.reject(res)
    }

    return res.data
  },
  error => {
    handleHttpError(error)
    return Promise.reject(error)
  }
)
```

## 2. Token 刷新

这是面试高频重点。

问题：

如果 token 过期，多个接口同时返回 401，不能每个接口都刷新一次 token。

正确做法：

1. 第一个 401 发起刷新 token。
2. 其他请求进入队列等待。
3. 刷新成功后，重放队列请求。
4. 刷新失败，统一退出登录。

伪代码：

```js
let isRefreshing = false
let requestQueue = []

function addQueue(callback) {
  requestQueue.push(callback)
}

function runQueue(newToken) {
  requestQueue.forEach(callback => callback(newToken))
  requestQueue = []
}
```

响应拦截：

```js
if (error.response?.status === 401) {
  const config = error.config

  if (!isRefreshing) {
    isRefreshing = true

    try {
      const newToken = await refreshToken()
      setToken(newToken)
      runQueue(newToken)

      config.headers.Authorization = `Bearer ${newToken}`
      return service(config)
    } catch (e) {
      logout()
      return Promise.reject(e)
    } finally {
      isRefreshing = false
    }
  } else {
    return new Promise(resolve => {
      addQueue(token => {
        config.headers.Authorization = `Bearer ${token}`
        resolve(service(config))
      })
    })
  }
}
```

面试说法：

> token 刷新最关键的是并发控制。如果十几个接口同时 401，不能同时刷新十几次 token。我用了一个 `isRefreshing` 标记和请求队列，第一个请求负责刷新，其他请求挂起等待。刷新成功后统一重放请求，刷新失败则统一退出登录。

## 3. 错误码处理

错误分层：

### HTTP 状态码

1. 401：未登录。
2. 403：无权限。
3. 404：资源不存在。
4. 500：服务器错误。
5. 504：网关超时。

### 业务状态码

```js
{
  code: 10001,
  message: "用户不存在",
  data: null
}
```

统一处理：

```js
function handleBusinessError(res) {
  switch (res.code) {
    case 401:
      logout()
      break
    case 403:
      router.push("/403")
      break
    default:
      ElMessage.error(res.message || "请求失败")
  }
}
```

注意有些接口不需要全局提示：

```js
request({
  url: "/api/check",
  method: "get",
  showError: false
})
```

面试说法：

> 不是所有接口错误都适合全局弹窗，所以我在请求配置里加了 `showError` 开关。比如表单校验类接口，有些错误需要页面局部处理，而不是统一 Message 弹出。

## 4. 取消重复请求

场景：

1. 用户连续点击查询。
2. 快速切换筛选条件。
3. 重复提交表单。
4. 路由切换后旧请求返回，污染新页面数据。

生成请求 key：

```js
function getRequestKey(config) {
  return [
    config.method,
    config.url,
    JSON.stringify(config.params),
    JSON.stringify(config.data)
  ].join("&")
}
```

使用 AbortController：

```js
const pendingMap = new Map()

service.interceptors.request.use(config => {
  const key = getRequestKey(config)

  if (pendingMap.has(key)) {
    pendingMap.get(key).abort()
    pendingMap.delete(key)
  }

  const controller = new AbortController()
  config.signal = controller.signal
  pendingMap.set(key, controller)

  return config
})

service.interceptors.response.use(
  response => {
    const key = getRequestKey(response.config)
    pendingMap.delete(key)
    return response
  },
  error => {
    if (error.config) {
      const key = getRequestKey(error.config)
      pendingMap.delete(key)
    }

    return Promise.reject(error)
  }
)
```

面试可以说：

> 我们对重复请求做了取消处理。比如同一个接口、同一组参数在短时间内重复触发，会取消上一次请求，避免旧响应覆盖新响应，也减少服务端压力。

## 5. 接口重试

适合：

1. 网络抖动。
2. 502 / 503 / 504。
3. 幂等请求，比如 GET。

不适合：

1. 创建订单。
2. 支付。
3. 新增数据。
4. 非幂等 POST。

配置：

```js
request({
  url: "/api/list",
  method: "get",
  retry: 2,
  retryDelay: 1000
})
```

实现：

```js
async function retryRequest(error) {
  const config = error.config

  if (!config || !config.retry) {
    return Promise.reject(error)
  }

  config.__retryCount = config.__retryCount || 0

  if (config.__retryCount >= config.retry) {
    return Promise.reject(error)
  }

  config.__retryCount += 1

  await new Promise(resolve => {
    setTimeout(resolve, config.retryDelay || 1000)
  })

  return service(config)
}
```

面试说法：

> 接口重试不能无脑做，尤其是新增、支付、审批这类非幂等接口不能自动重试。我们主要对 GET 查询类接口和部分幂等接口开放 retry 配置，并且限制最大重试次数。

## 6. loading 合并

普通写法会导致多个接口同时请求时 loading 闪烁。

解决：

```js
let loadingCount = 0
let loadingInstance = null

function showLoading() {
  if (loadingCount === 0) {
    loadingInstance = ElLoading.service()
  }

  loadingCount++
}

function hideLoading() {
  loadingCount--

  if (loadingCount <= 0) {
    loadingCount = 0
    loadingInstance?.close()
    loadingInstance = null
  }
}
```

请求配置：

```js
request({
  url: "/api/list",
  loading: true
})
```

面试说法：

> 多个请求同时发生时，如果每个请求都独立开关 loading，会出现闪烁。我用了计数器方式合并 loading，第一个请求开始时打开，最后一个请求结束后关闭。

## 7. 文件下载

文件下载容易踩坑。

```js
async function downloadFile(config, filename) {
  const res = await service({
    ...config,
    responseType: "blob"
  })

  const blob = new Blob([res])
  const url = window.URL.createObjectURL(blob)

  const link = document.createElement("a")
  link.href = url
  link.download = filename
  link.click()

  window.URL.revokeObjectURL(url)
}
```

注意：

如果后端返回 blob 错误信息，要解析：

```js
if (blob.type.includes("application/json")) {
  const text = await blob.text()
  const json = JSON.parse(text)
  ElMessage.error(json.message)
}
```

---

# 五、性能优化

后台系统性能优化不是只说“懒加载”，要结合真实场景说。

重点：

**首屏加载、路由切分、组件懒加载、缓存、虚拟列表、表格优化、打包分析、请求优化。**

## 1. 路由懒加载

```js
{
  path: "/user",
  component: () => import("@/views/system/user/index.vue")
}
```

面试说法：

> 后台系统页面多，如果所有页面都打进首屏 bundle，会导致首屏加载很慢。所以路由统一采用懒加载，访问哪个页面才加载对应 chunk。

## 2. 组件懒加载

对于弹窗、图表、富文本编辑器、大型组件：

```js
const UserDialog = defineAsyncComponent(() => import("./UserDialog.vue"))
```

适合懒加载的组件：

1. 富文本编辑器。
2. 图表组件。
3. 复杂弹窗。
4. Excel 导入导出组件。
5. 地图组件。
6. 低频使用的详情组件。

面试说法：

> 我不会所有组件都懒加载。高频小组件没必要懒加载，否则反而增加请求碎片。主要针对富文本、图表、地图、复杂弹窗这类体积大、低频使用的组件做异步加载。

## 3. KeepAlive 缓存

典型场景：

用户从列表页进入详情页，再返回列表页，希望保留查询条件、页码、滚动位置。

```vue
<router-view v-slot="{ Component, route }">
  <keep-alive :include="cachedViews">
    <component :is="Component" :key="route.fullPath" />
  </keep-alive>
</router-view>
```

缓存页面名称：

```js
const cachedViews = ["UserList", "OrderList"]
```

注意点：

1. 不是所有页面都缓存。
2. 编辑页通常不缓存。
3. 缓存页面需要处理刷新时机。
4. 配合 tabs-view 会更常见。

面试说法：

> 列表页适合缓存，因为用户经常从列表进入详情再返回。如果不缓存，查询条件和页码会丢失，体验很差。但表单编辑页不适合盲目缓存，否则可能出现旧数据残留。所以我们通过路由 meta 控制哪些页面进入 keep-alive。

## 4. 虚拟列表

后台常见问题：

1. 下拉框几千个选项。
2. 表格几万行数据。
3. 树节点很多。
4. 日志列表很长。

虚拟列表核心思想：

只渲染可视区域内的数据，而不是一次性渲染全部 DOM。

面试说法：

> 如果表格或下拉框数据量很大，瓶颈不一定是接口，而是 DOM 渲染。比如一次渲染几千个 el-option，会明显卡顿。我们会用虚拟列表，只渲染可视区域的数据，滚动时动态替换 DOM。

## 5. 表格性能优化

具体措施：

1. 后端分页，不一次性加载全量。
2. 大表格避免过多复杂 slot。
3. 列配置按需渲染。
4. 操作列按钮过多时折叠成更多菜单。
5. 固定列不要滥用。
6. 避免每个单元格里写复杂计算。
7. 字典数据提前 map 化。

字典优化：

```js
const dictMap = computed(() => {
  return dictList.value.reduce((map, item) => {
    map[item.value] = item.label
    return map
  }, {})
})
```

不要每个 cell 都：

```js
dictList.find(item => item.value === row.status)
```

面试说法：

> 表格里最容易被忽略的是单元格重复计算。如果每一行都通过 find 查字典，几百行几千行就会有性能损耗。我一般会提前把字典数组转成 Map，单元格里 O(1) 取值。

## 6. 请求优化

1. 字典缓存。
2. 重复请求取消。
3. 防抖搜索。
4. 切换页面取消未完成请求。
5. 多接口合并。
6. 低频数据预加载。

搜索防抖：

```js
const handleSearch = debounce(() => {
  queryParams.pageNum = 1
  loadData()
}, 300)
```

## 7. 打包分析

Vite 项目可以使用可视化分析插件：

```js
import { visualizer } from "rollup-plugin-visualizer"

plugins: [
  visualizer({
    open: true,
    gzipSize: true,
    brotliSize: true
  })
]
```

重点分析：

1. 哪些依赖体积大。
2. 是否重复打包。
3. moment、lodash、echarts、富文本是否过大。
4. 是否需要按需引入。
5. chunk 拆分是否合理。

## 8. 第三方库优化

例如 Element Plus 按需引入：

```js
import AutoImport from "unplugin-auto-import/vite"
import Components from "unplugin-vue-components/vite"
import { ElementPlusResolver } from "unplugin-vue-components/resolvers"
```

ECharts 按需引入：

```js
import * as echarts from "echarts/core"
import { BarChart, LineChart } from "echarts/charts"
import { GridComponent, TooltipComponent } from "echarts/components"
import { CanvasRenderer } from "echarts/renderers"

echarts.use([
  BarChart,
  LineChart,
  GridComponent,
  TooltipComponent,
  CanvasRenderer
])
```

## 9. 性能优化面试总结话术

> 性能优化我主要从首屏体积、运行时渲染和请求数量三个方向做。首屏方面通过路由懒加载、组件懒加载、依赖按需引入和打包分析降低 bundle 体积；运行时方面对大表格、大下拉、大树使用虚拟列表和缓存；请求方面做字典缓存、重复请求取消、防抖搜索和接口合并。优化不是盲目做，而是通过打包分析和实际页面卡顿点定位瓶颈。

---

# 六、工程化

工程化是五年前端必须重点讲的内容。

核心包括：

**Vite、TypeScript、ESLint、Prettier、环境变量、目录规范、Git 规范、自动部署、构建优化。**

## 1. Vite

可以讲：

> 项目使用 Vite 作为构建工具，开发阶段利用原生 ESM 提升启动速度，生产环境通过 Rollup 打包。相比传统 webpack 项目，Vite 在冷启动和热更新上明显更快，尤其适合后台系统这种页面和组件比较多的项目。

常见配置：

```js
export default defineConfig({
  plugins: [
    vue()
  ],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "src")
    }
  },
  server: {
    proxy: {
      "/api": {
        target: "https://api.example.com",
        changeOrigin: true,
        rewrite: path => path.replace(/^\/api/, "")
      }
    }
  }
})
```

## 2. TypeScript

TS 重点不是“我会写类型”，而是：

1. 接口返回值类型。
2. 表单 schema 类型。
3. 表格 columns 类型。
4. 路由 meta 类型。
5. store 类型。
6. 组件 props 类型。

例如接口类型：

```ts
interface PageResult<T> {
  list: T[]
  total: number
}

interface UserItem {
  id: string
  username: string
  phone: string
  status: number
}

function getUserList(params: UserQuery): Promise<PageResult<UserItem>> {
  return request.get("/user/list", { params })
}
```

表格列类型：

```ts
interface TableColumn<T = any> {
  prop: keyof T | string
  label: string
  width?: number
  minWidth?: number
  fixed?: "left" | "right"
  dict?: string
  render?: (row: T) => VNode | string
}
```

面试说法：

> TS 在后台项目里最大的价值不是炫技，而是把接口数据、表单 schema、表格 columns 这些高频配置约束起来。否则配置化越多，越容易出现字段名写错、返回结构变更后页面运行时报错的问题。

## 3. ESLint + Prettier

ESLint 管代码质量，Prettier 管格式。

ESLint 处理：

1. 未使用变量。
2. 禁止隐式 any。
3. 禁止 console。
4. 组件命名规范。
5. import 顺序。
6. hooks 使用规则。

Prettier 处理：

1. 缩进。
2. 分号。
3. 引号。
4. 换行。
5. 最大行宽。

面试说法：

> 我们没有把代码风格依赖人工 review，而是通过 ESLint 和 Prettier 自动约束。提交前会跑 lint，避免低级格式问题进入代码仓库，让 code review 更关注业务逻辑和架构设计。

## 4. 环境变量

常见环境：

1. development
2. test
3. staging
4. production

文件：

```bash
.env.development
.env.test
.env.production
```

内容：

```bash
VITE_API_BASE_URL=https://dev-api.example.com
VITE_APP_ENV=development
```

使用：

```js
const baseURL = import.meta.env.VITE_API_BASE_URL
```

注意：

Vite 中暴露给客户端的变量必须以 `VITE_` 开头。

面试说法：

> 不同环境的接口地址、上传地址、静态资源地址都通过环境变量管理，避免在代码中硬编码。构建时根据 mode 加载不同的 env 文件。

## 5. 目录结构

一个比较清晰的后台项目结构：

```bash
src
├── api
│   ├── system
│   └── order
├── assets
├── components
│   ├── ProTable
│   ├── ProForm
│   └── PermissionButton
├── composables
│   ├── useTable.ts
│   ├── useForm.ts
│   └── usePermission.ts
├── directives
│   └── permission.ts
├── layout
├── router
├── stores
│   ├── user.ts
│   ├── permission.ts
│   └── dict.ts
├── styles
├── utils
│   ├── request.ts
│   ├── storage.ts
│   └── auth.ts
└── views
    ├── system
    └── dashboard
```

面试说法：

> 我们会把业务页面、通用组件、请求模块、状态管理、工具函数分层。像 ProTable、ProForm 这种组件放在 components，和业务无关；具体接口放在 api；权限状态放在 store；业务页面只负责组合配置和处理少量业务逻辑。

## 6. 自动部署

常见流程：

提交代码
→ GitLab CI / GitHub Actions 触发
→ 安装依赖
→ ESLint 检查
→ TypeScript 检查
→ 构建
→ 上传 dist
→ 部署到 Nginx / OSS / Docker

示例流程：

```yaml
stages:
  - install
  - lint
  - build
  - deploy
```

部署到 Nginx 后要注意前端路由刷新 404。

Nginx 配置：

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

面试说法：

> 因为我们使用的是 history 路由模式，所以 Nginx 需要配置 `try_files`，否则用户刷新二级路由会出现 404。

## 7. 构建产物优化

Vite 配置 chunk：

```js
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        vue: ["vue", "vue-router", "pinia"],
        elementPlus: ["element-plus"],
        echarts: ["echarts"]
      }
    }
  }
}
```

注意不要过度拆分。

面试说法：

> chunk 拆分不是越细越好，太碎会增加请求数量。一般会把 Vue 生态、UI 库、图表库这类稳定依赖拆出来，业务模块通过路由懒加载自然分包。

## 8. Git 规范

常见提交规范：

```bash
feat: 新增用户管理页面
fix: 修复权限按钮不显示问题
refactor: 重构请求封装
style: 调整样式
docs: 更新文档
chore: 修改构建配置
```

可以配合：

1. husky
2. lint-staged
3. commitlint

提交流程：

```bash
git commit
→ lint-staged 检查暂存文件
→ commitlint 检查提交信息
→ 通过后允许提交
```

---

# 七、SaaS 平台加分点

如果项目包装成 SaaS，建议额外准备下面几个点。

## 1. 多租户

SaaS 常见核心：

不同企业客户之间数据隔离。

前端关注：

1. 当前租户选择。
2. 请求头携带 tenantId。
3. 菜单和权限按租户变化。
4. 切换租户后清空缓存和重新拉取权限。
5. 某些功能按租户套餐控制。

```js
service.interceptors.request.use(config => {
  const tenantStore = useTenantStore()

  if (tenantStore.currentTenantId) {
    config.headers["X-Tenant-Id"] = tenantStore.currentTenantId
  }

  return config
})
```

面试说法：

> SaaS 场景下，用户可能属于多个租户。切换租户后，前端需要重新拉取当前租户下的角色、菜单、权限和字典数据，并清空和租户相关的缓存，避免出现 A 租户的数据残留到 B 租户的问题。

## 2. 套餐权限

不同套餐开放不同功能。

例如：

```js
{
  featureCode: "advanced_report",
  enabled: false
}
```

前端控制：

```vue
<FeatureGuard code="advanced_report">
  <AdvancedReport />
</FeatureGuard>
```

没权限时展示升级提示。

面试说法：

> SaaS 平台除了角色权限，还有套餐权限。角色权限解决某个用户能不能用，套餐权限解决某个租户有没有购买这个能力。这两个维度要分开设计。

## 3. 白标配置

不同客户可能有不同 logo、主题色、系统名称。

```js
{
  appName: "客户管理平台",
  logo: "https://xxx/logo.png",
  themeColor: "#1677ff"
}
```

前端启动时加载租户配置，动态设置主题。

---

# 八、面试时可以这样介绍整个项目

你可以准备一段 1 分钟项目介绍：

> 我之前做的是一个企业级后台管理系统，也可以理解成 SaaS 管理平台。这个系统主要服务多角色、多租户、多业务线的管理场景。我负责的重点不是简单写 CRUD 页面，而是沉淀了一套中后台通用能力，包括动态权限、动态路由、按钮权限、schema 动态表单、通用表格、axios 请求封装、token 刷新、重复请求取消、路由和组件懒加载、以及 Vite + TypeScript 的工程化规范。
>
> 比如权限系统这块，用户登录后会获取菜单树和权限码，前端根据菜单动态生成路由，同时渲染左侧菜单，页面按钮通过权限指令控制，接口权限由后端兜底。表单和表格这块，我们做了配置化封装，业务页面主要写 schema 和 columns，减少重复代码。请求层统一处理 token、错误码、loading、重复请求和文件下载。整体目标是让业务开发更快，同时保证系统在权限、安全、性能和代码规范上的一致性。

---

# 九、每个模块的面试追问准备

## 权限系统可能被问

**Q：前端权限安全吗？**

答：

> 前端权限不作为安全边界，只做用户体验控制。真正的权限校验必须在后端。前端隐藏菜单和按钮只是避免用户看到无权限入口，接口层仍然要通过 token、角色、权限码做校验。

## 动态路由可能被问

**Q：刷新页面后动态路由丢失怎么办？**

答：

> 刷新后内存中的动态路由会丢失，所以路由守卫里会判断权限路由是否已经生成。如果没有生成，就先根据 token 拉取用户信息和菜单权限，重新 addRoute，然后用 replace 方式重新进入目标路由，避免进入 404。

## 动态表单可能被问

**Q：动态表单复杂联动怎么处理？**

答：

> 我会在 schema 里声明字段依赖关系，比如城市字段依赖省份字段。当依赖字段变化时，自动清空下游字段并重新加载 options。同时支持 visible、disabled、asyncOptions、transformIn、transformOut，让联动、异步数据和数据转换都配置化。

## 通用表格可能被问

**Q：通用表格怎么避免过度封装？**

答：

> 我会把稳定重复的逻辑封装进去，比如分页、查询、排序、loading、导出、列配置、权限按钮。但复杂业务渲染通过 slot 或 render 函数暴露出去，避免组件为了兼容所有场景变得过于臃肿。通用组件只解决 80% 高频场景，特殊页面允许业务自定义。

## 请求封装可能被问

**Q：多个请求同时 401 怎么办？**

答：

> 不能让每个请求都刷新 token。我会用 `isRefreshing` 标记和队列机制。第一个 401 请求触发刷新 token，其他请求进入队列等待。刷新成功后统一重放队列请求，刷新失败则统一退出登录。

## 性能优化可能被问

**Q：后台系统主要优化哪些地方？**

答：

> 我主要从三个方向优化：首屏加载、运行时渲染、请求数量。首屏用路由懒加载、组件懒加载、依赖按需引入和 chunk 拆分；运行时针对大表格、大下拉、大树使用虚拟列表和缓存；请求层做字典缓存、重复请求取消、防抖搜索和接口合并。

---

# 十、最后给你一个高级表达

面试时可以重点强调这句话：

**我做后台管理系统时，不会把它理解成一堆 CRUD 页面，而是把它拆成权限、表单、表格、请求、缓存、路由、工程规范这些稳定能力。只要这些基础设施做好，后续业务页面大部分都可以通过配置快速生成，开发效率、代码一致性和后期维护成本都会明显改善。**
