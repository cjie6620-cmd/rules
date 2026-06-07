# Vue3 前端项目规范

> 参考：Vue.js 3 Style Guide（Priority A/B/C）、Pinia 官方文档、Vue Router 4、TailwindCSS 官方规范

## 技术栈（必须遵守）
- 框架：Vue 3.4+，Composition API（禁止 Options API）
- 语言：TypeScript 5+，`strict: true` + `isolatedModules: true`
- 构建工具：Vite 5+
- 状态管理：Pinia（禁止 Vuex）
- 路由：Vue Router 4+
- UI 组件库：Ant Design Vue（优先用现成组件，不重复造轮子）
- 样式：TailwindCSS（优先原子类，减少自定义 CSS）
- 请求：axios（封装成 `src/utils/request.ts`，禁止直接用 axios）
- 图标：@iconify/vue
- 测试：Vitest + @vue/test-utils

## 组件规范

### 命名（Vue Priority A/B）
- 文件名 PascalCase：`UserProfile.vue`（不用 kebab-case）
- 组件名必须多单词，避免与 HTML 元素冲突（Priority A）
- 基础/通用组件加前缀：`BaseButton.vue`、`AppHeader.vue`（Priority B）
- 紧耦合子组件带父名前缀：`TodoListItem.vue`、`TodoListItemButton.vue`（Priority B）
- 用完整单词，禁止缩写：`UserProfileOptions.vue`，不是 `UProfOpts.vue`

### 文件结构（Priority C，顺序固定）
```
<script setup lang="ts">  ← 第一
<template>                 ← 第二
<style scoped>             ← 第三（Tailwind 下基本为空）
```

### script setup 内部顺序
```
1. imports（vue 核心 → 第三方库 → 类型 → 本地模块）
2. defineProps / defineEmits / defineExpose
3. inject / provide
4. store 实例（useXxxStore()）
5. 响应式状态（ref, reactive）
6. computed
7. watchers（watch, watchEffect）
8. 生命周期（onMounted, onUnmounted）
9. 方法 / 事件处理函数
```

### Props 与 Emits（Priority A）
- 必须用类型声明：`defineProps<{}>()`
- 默认值用 `withDefaults()`（Vue 3.3-3.4）或响应式解构（Vue 3.5+）
- Props 声明必须尽可能详细，至少指定类型（Priority A）
- Prop 命名：camelCase
- Emits 类型声明：`defineEmits<{ change: [id: number] }>()`
- 事件处理参数必须显式类型：`event: Event`

### 模板规范（Priority A/B）
- `v-if` 和 `v-for` 禁止在同一元素上（Priority A）→ 用 computed 过滤替代
- `v-for` 必须带 `:key`（Priority A）
- 多属性元素必须换行，每行一个属性（Priority B）
- 无内容组件必须自闭合：`<MyComponent />`（Priority B）
- 模板表达式必须简单，复杂逻辑提取到 computed 或方法（Priority B）
- 指令缩写（`:`、`@`、`#`）全项目统一用或统一不用（Priority B）

### 大小限制
- 单个组件不超过 200 行，超过拆子组件或提取 composable
- 单一职责，一个组件只做一件事

## 标准组件模板

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useUserStore } from '@/stores/useUserStore'
import type { User } from '@/types/user'

interface Props {
  userId: number
  showAvatar?: boolean
}

const props = defineProps<Props>()
const emit = defineEmits<{
  update: [user: User]
  delete: [id: number]
}>()

const userStore = useUserStore()
const loading = ref(false)
const userName = computed(() => userStore.users.find(u => u.id === props.userId)?.name ?? '')

onMounted(() => {
  // 初始化逻辑
})
</script>

<template>
  <div class="flex items-center gap-4">
    <span>{{ userName }}</span>
  </div>
</template>

<style scoped>
/* Tailwind 优先，此处基本为空 */
</style>
```

## Pinia Store 规范

### 文件与命名
- 文件放 `src/stores/`，命名：`useXxxStore.ts`
- Store id 与文件名对应：`defineStore('user', ...)`
- 每个 Store 只负责一个业务域

### 必须用 setup 函数写法（不用 options 写法）
```ts
export const useUserStore = defineStore('user', () => {
  // ref() = state
  const users = ref<User[]>([])
  const loading = ref(false)

  // computed() = getters
  const activeUsers = computed(() => users.value.filter(u => u.active))

  // functions = actions
  async function fetchUsers() {
    loading.value = true
    try {
      users.value = await getUserList()
    } finally {
      loading.value = false
    }
  }

  return { users, loading, activeUsers, fetchUsers }
})
```

### Store 使用规范
- 状态/getter 解构必须用 `storeToRefs()`，否则丢失响应性
- action 可直接解构，不需要 `storeToRefs`
- action 命名：动词开头（`fetchUsers`、`updateUser`、`deleteUser`）
- action 内部用 try/catch 处理错误
- 禁止在 `<script setup>` / `setup()` 外部调用 store

```ts
const store = useUserStore()
const { users, loading } = storeToRefs(store)   // 响应性保留
const { fetchUsers, deleteUser } = store         // action 直接解构
```

## TypeScript 规范

### 基础规则
- `tsconfig.json` 必须 `strict: true` + `isolatedModules: true`
- 禁止 `any`，用 `unknown` + 类型守卫替代
- 路径别名在 Vite 和 tsconfig 里都要配：`@/` → `src/`

### 类型组织
- 全局/共享类型放 `src/types/`，按领域分文件：`user.ts`、`api.ts`、`common.ts`
- 组件 Props 接口定义在 SFC 文件内部，`defineProps` 之前
- API 返回数据必须定义类型，禁止裸用响应

### 响应式数据类型
```ts
// ref — 基本类型自动推断
const count = ref(0)                        // Ref<number>

// ref — 复杂/联合类型显式声明
const year = ref<string | number>('2020')

// ref — 初始值可能为空
const user = ref<User>()                    // Ref<User | undefined>

// reactive — 通过接口类型声明，不用泛型参数
const book: Book = reactive({ title: 'Vue 3 Guide' })

// computed — 自动推断返回值
const double = computed(() => count.value * 2)  // ComputedRef<number>
```

## API 请求规范

### 目录结构
```
src/
  api/
    user.ts          # 用户相关接口
    activity.ts      # 活动相关接口
  utils/
    request.ts       # axios 实例 + 拦截器
  types/
    api.ts           # Result<T> 等通用响应类型
```

### request.ts 封装要求
- 创建唯一 axios 实例，`baseURL` 从环境变量读取
- 请求拦截器：自动附加 Token
- 响应拦截器：解包 `Result<T>`，统一处理 HTTP 错误
- 错误统一用 toast 提示，禁止 `console.error`

### API 函数规范
```ts
import request from '@/utils/request'
import type { User, UserQuery } from '@/types/user'
import type { Result, PageResult } from '@/types/api'

export function getUserList(params: UserQuery): Promise<PageResult<User>> {
  return request.get('/api/users', { params })
}

export function createUser(data: Partial<User>): Promise<Result<void>> {
  return request.post('/api/users', data)
}
```
- 命名：动词+名词（`getUserList`、`createUser`、`updateProfile`）
- 禁止在组件里直接调用 `axios.get()`，必须用封装后的 request

## 路由规范

### 目录结构
```
src/
  router/
    index.ts         # createRouter 实例
    routes.ts        # 路由定义
    guards.ts        # 全局导航守卫
    modules/         # 按功能拆分的路由模块（可选）
```

### 路由定义
- 路径 kebab-case：`/user-profile`、`/activity-list`
- 路由 name PascalCase：`UserProfile`、`ActivityList`
- **所有页面组件必须懒加载**（Vue Router 官方推荐）
- 必须有兜底 404 路由：`{ path: '/:pathMatch(.*)*', component: NotFound }`

```ts
{
  path: '/user-profile',
  name: 'UserProfile',
  component: () => import('@/views/user/UserProfile.vue'),
  meta: { requiresAuth: true, title: '用户资料', icon: 'user' }
}
```

### Meta 字段约定
```ts
interface RouteMeta {
  requiresAuth?: boolean    // 需要登录
  roles?: string[]          // 角色权限
  title?: string            // 页面标题
  icon?: string             // 菜单图标
  hidden?: boolean          // 侧边栏隐藏
  keepAlive?: boolean       // 组件缓存
}
```

### 导航守卫（Vue Router 4 推荐写法）
- 用返回值，不用 `next()` 回调（Vue Router 4 官方推荐，`next()` 已不推荐）
- `beforeEach` 处理登录校验
- `afterEach` 更新页面标题

```ts
router.beforeEach(async (to) => {
  if (to.meta.requiresAuth && !isAuthenticated) {
    return { name: 'Login' }
  }
})
```

## TailwindCSS 规范

### 核心原则
- 优先用 Tailwind 原子类，这是项目的样式规范
- 响应式：移动优先，用 `sm:` `md:` `lg:` `xl:` 断点
- 状态变体：`hover:` `focus:` `active:` `disabled:`
- 暗色模式：`dark:` 前缀（如有需要）

### 类名顺序（Tailwind 推荐）
```
1. 布局：    display, position, overflow, z-index
2. 盒模型：  width, height, padding, margin, border, rounded
3. 排版：    font-size, font-weight, text-align, text-color
4. 背景：    bg-color
5. 效果：    shadow, opacity, transition
6. 响应式：  sm:, md:, lg:, xl:（同上顺序）
7. 状态：    hover:, focus:, active:
8. 暗色：    dark:（永远放最后）
```

### 何时提取组件（不是提取 CSS 类）
- 同一组 5+ 个 Tailwind 类出现 3 次以上 → 提取为 Vue 组件
- `@apply` 只用于真正的设计 token，不要滥用
- 颜色、间距等在 `tailwind.config.js` 定义，禁止魔法值 `text-[#1a2b3c]`

### `<style>` 基本为空
- Tailwind 项目 `<style scoped>` 几乎不需要写内容
- 只有真正无法用工具类表达的自定义 CSS 才写在 `<style>` 里

## 测试规范（Vitest）

### 文件组织
- 测试文件放 `src/__tests__/` 或与源文件同目录：`UserProfile.spec.ts`
- Vitest 读 `vite.config.ts` 的 `test` 配置块

### 命名规范
```ts
describe('UserProfile', () => {
  it('should display user name when provided', () => { ... })
  it('should emit update event on form submit', () => { ... })
  it('should show loading state while fetching data', () => { ... })
})
```

### 组件测试模板
```ts
import { mount } from '@vue/test-utils'
import UserProfile from './UserProfile.vue'

describe('UserProfile', () => {
  it('should render user name', () => {
    const wrapper = mount(UserProfile, { props: { userId: 1 } })
    expect(wrapper.text()).toContain('John')
  })
})
```

### Store 测试模板
```ts
import { setActivePinia, createPinia } from 'pinia'
import { useUserStore } from './useUserStore'

describe('useUserStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('should fetch users', async () => {
    const store = useUserStore()
    await store.fetchUsers()
    expect(store.users.length).toBeGreaterThan(0)
  })
})
```

### Mock 规范
- 用 `vi.mock()` mock 模块，`vi.fn()` spy 函数
- 测试行为和输出，不测实现细节

## 性能规范

### 加载性能
- 所有路由组件必须懒加载（Vite 代码分割）
- 重型依赖按需加载（动态 import）
- 用 `lodash-es` 替代 `lodash`（tree-shaking 友好）
- 新增依赖前检查包体积（bundlephobia.com）

### 更新性能
- `v-for` 必须带 `:key`（Priority A）
- 禁止 `v-if` + `v-for` 同元素（Priority A）→ 用 computed 过滤
- 派生数据用 `computed`，不在模板里调方法（每次渲染都重新计算）
- 大对象不需要深度响应时用 `shallowRef()`
- 列表超 100 条考虑虚拟滚动（`@tanstack/vue-virtual`）
- 静态内容用 `v-once`
- 复杂列表用 `v-memo` 优化

## 前后端联调与 UI 布局代码审查规范

### 组件-接口绑定审查（每个业务页面逐项核查）

> 前端每个按钮/下拉框/搜索框，必须能在代码中追溯到对应的 API 调用，且数据展示链路完整。

**审查方法：从组件 `@click` 事件 → `api/xxx.ts` 函数 → 后端 Controller，逐层追踪。**

| 组件类型 | 代码审查点 | 典型问题 |
|----------|-----------|---------|
| **搜索/查询按钮** | `@click` → `fetchList({ keyword, status, ... })` → API 函数 → 后端参数名一致 | 前端 `pageNum` vs 后端 `page`（命名错配） |
| **新增按钮** | `@click` → `openForm()` → 弹窗字段 ↔ 后端 DTO 字段一一对应 | 弹窗字段多余或缺失 |
| **编辑按钮** | `@click` → `getDetail(id)` → 回显到表单 `v-model` → `submitForm()` 调用 PUT | 未调详情 API，表单为空 |
| **删除按钮** | `@click` → `Modal.confirm` → `deleteById(row.id)` → 列表刷新 | 无二次确认，或传错 ID |
| **下拉框** | `options` 数据来源 → 哪个 API；`@change` → 是否触发查询刷新 | options 硬编码而非 API 获取 |
| **分页组件** | `@change` 事件 → `fetchList({ page, pageSize })` ↔ 后端分页参数 | 切页后列表未刷新 |
| **导出按钮** | `@click` → `exportXxx({ ...queryForm })` ↔ 后端导出接口参数 | 未传当前筛选条件 |
| **批量操作** | `selectedRowKeys` → `batchXxx(ids)` ↔ 后端 `@RequestBody List<Long> ids` | 传了 undefined 或空数组 |

### CRUD 数据流闭环审查

**每个业务实体必须在代码中验证以下 5 步全部存在：**

```
① 列表展示（搜索+分页）→ ② 新增 → ③ 编辑（回显+保存）→ ④ 删除 → ⑤ 列表刷新
```

| 步骤 | 代码审查点 | 典型缺陷 |
|------|-----------|---------|
| ① 列表 | `onMounted` 中调用 `fetchList()`，搜索 `@click` 调用 `fetchList()` | 页面加载无数据 |
| ② 新增 | 提交成功后 `emit('success')` 或调用 `fetchList()` | 弹窗关闭但列表不更新 |
| ③ 编辑回显 | 打开弹窗调用详情 API，`v-model` 字段名与 VO 一致 | 编辑表单空白 |
| ③ 编辑保存 | 提交调用 PUT/PATCH 接口，参数名与后端一致 | 调了新增接口而非编辑 |
| ④ 删除 | `Modal.confirm` 二次确认，传 `row.id` | 误删，或删错记录 |
| ⑤ 刷新 | 增/删/改 成功后调用 `fetchList()` | 操作成功但列表不更新 |

### UI 布局代码审查（美观 + 无遮挡 + 无重叠）

> 组件美观展示给用户，布局合理，不出现遮挡、重叠、溢出。从代码层面审查 Tailwind 类和 CSS 规则。

| 审查项 | 代码层面要求 | 检查方式 |
|--------|-------------|---------|
| **无遮挡** | 弹窗 `z-index` 高于 header/sidebar；fixed 元素不覆盖内容区 | 检查 Tailwind `z-` 类或自定义 `z-index` 值 |
| **无重叠** | 同级元素用 flex/grid 布局，不靠 absolute 定位堆叠 | 检查模板中是否滥用 `absolute`/`fixed` |
| **无溢出** | 长文本容器有 `truncate` 或 `overflow-hidden` | 检查 `<a-table>` 列是否设置 `ellipsis: true` |
| **间距合理** | 组件间用 Tailwind `gap-`/`p-`/`m-` 系统间距，不用魔法数字 | 检查是否出现 `style="margin-left: 13px"` 等内联 |
| **响应式** | 外层容器用 `flex-1`/`w-full`/`min-w-0`，不写死 `width: 1200px` | 检查是否有硬编码像素宽度 |
| **对齐一致** | 表单用 `a-form-item` 统一对齐；按钮组用 `a-space` | 检查手动 `position` 定位 |
| **表格列宽** | `<a-table>` 每列 `width` 明确设置，总宽度不超容器 | 检查列宽之和是否合理 |
| **弹窗尺寸** | 弹窗 `width` 适配内容，长表单用 `drawer` 替代 `modal` | 检查弹窗宽度是否合理 |

**Tailwind 间距参考（代码审查对照）：**

| 场景 | 类名 | 像素值 |
|------|------|--------|
| 同级元素间距 | `gap-2` ~ `gap-4` | 8px ~ 16px |
| 卡片内边距 | `p-4` ~ `p-6` | 16px ~ 24px |
| 页面边距 | `p-6` ~ `p-8` | 24px ~ 32px |
| 标题与内容 | `mb-2` ~ `mb-4` | 8px ~ 16px |
| 表单字段间距 | `mb-4` | 16px |

**典型布局缺陷（代码审查重点）：**

| 缺陷 | 代码特征 | 修复方式 |
|------|---------|---------|
| 弹窗遮挡 header | 弹窗未设 `z-index` 或低于 header | 加 `z-50` 或检查 Ant Design 默认值 |
| 表格横向溢出 | 列宽之和 > 容器宽度，缺 `scroll` | 设置 `:scroll="{ x: 'max-content' }"` |
| 长文本撑破单元格 | 列未设 `ellipsis: true` | 加 `ellipsis: true` |
| 按钮与内容重叠 | 用 `absolute` 定位按钮 | 改为 `flex justify-between` |
| 表单字段拥挤 | 字段间无 `margin-bottom` | 加 `mb-4` 或用 `a-form` 默认间距 |
| 弹窗内容超长无滚动 | 弹窗 body 未设 `max-height` | 加 `max-h-[60vh] overflow-y-auto` |

### 异常处理与交互审查

| 场景 | 代码审查点 | 要求 |
|------|-----------|------|
| **接口失败** | API 调用是否有 `catch` + `message.error()` | 禁止无错误处理 |
| **表单校验** | `rules` 与后端 DTO 注解对齐（`required` ↔ `@NotBlank`） | 前端通过但后端报错 |
| **空数据** | 列表为空时有 `a-empty` 占位 | 禁止空白页面 |
| **loading** | 请求中按钮 `loading=true`，防止重复提交 | 防连点 |
| **权限** | `v-if="hasPermission('xxx:edit')"` ↔ 后端 `@SaCheckPermission` | 前后端权限一致 |


## 禁止事项（完整清单）

### Vue 组件
- 禁止 Options API，必须 Composition API + `<script setup>`
- 禁止 `v-if` 和 `v-for` 在同一元素
- 禁止直接修改 props（单向数据流）
- 禁止 template 里写复杂逻辑，提取到 computed
- 禁止全局注册所有组件，用局部 import

### TypeScript
- 禁止 `any`，用 `unknown` + 类型守卫
- 禁止不带类型的 `ref()`（复杂对象必须显式泛型）
- 禁止不带类型的事件参数（`function handleClick(event: Event)`）

### Pinia
- 禁止 Vuex
- 禁止 options 写法，必须 setup 函数写法
- 禁止不带 `storeToRefs()` 解构 state/getter
- 禁止在 `<script setup>` 外部调用 store
- 禁止混合职责，一个 store 只管一个业务域

### API 请求
- 禁止在组件里直接 `axios.get()` / `axios.post()`
- 禁止无错误处理的 API 调用
- 禁止裸用未类型的 API 响应
- 禁止硬编码 API URL，用环境变量

### 样式
- 禁止内联样式 `style="..."`，用 Tailwind 类
- 禁止自定义 CSS 覆盖已有 Tailwind 功能
- 禁止魔法颜色值，用 `tailwind.config.js` 设计 token
- 禁止混用 Tailwind 和其他 CSS 框架

### 路由
- 禁止非懒加载的路由组件
- 禁止用旧版 `next()` 回调（用返回值替代）
- 禁止硬编码路由路径，用命名路由：`router.push({ name: 'UserProfile' })`
- 禁止缺少兜底 404 路由
- 禁止受保护路由缺少鉴权守卫

---

## 开发规则整合

### 架构设计
- 优先采用当前主流且经过生产验证的企业级方案
- 以中型公司实际落地标准设计
- 满足业务需求即可，不允许过度设计

### 编码原则
- 使用最少代码完成需求
- 优先可读性，其次是代码量
- 避免重复代码（DRY）

### 代码要求
- 所有代码必须包含中文注释
- 必须进行必要的判空处理
- 必须进行必要的异常处理

### 性能原则
- 先保证正确性
- 再保证可维护性
- 最后再考虑性能优化
