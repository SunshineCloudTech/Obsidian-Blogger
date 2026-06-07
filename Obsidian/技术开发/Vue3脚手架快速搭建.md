---
title: Vue3企业级脚手架搭建完整指南
date: 2024-06-06
tags:
  - Vue3
  - 前端架构
  - 脚手架搭建
  - Composition API
  - 项目实战
description: 详细介绍Vue3企业级项目脚手架的搭建过程，包括项目初始化、组合式API应用、TypeScript集成、Pinia状态管理、路由配置、组件设计和构建部署等完整流程。
icon: code
category:
  - 编程相关
  - 前端开发
index: true
cover: https://picsum.photos/1200/600?random=26
---
# Vue3企业级脚手架搭建完整指南

## 概述

Vue3作为Vue.js框架的最新主要版本，带来了诸多革命性的改进，包括更好的性能、更小的包体积、更好的TypeScript支持以及Composition API等。本文详细介绍如何从零开始搭建一个完整的Vue3企业级项目脚手架，涵盖项目初始化、组合式API应用、状态管理、路由配置、组件库集成与构建优化等关键环节。

## 目录

- [环境准备与项目初始化](#环境准备与项目初始化)
- [项目结构设计](#项目结构设计)
- [Composition API实践](#composition-api实践)
- [路由配置与管理](#路由配置与管理)
- [Pinia状态管理](#pinia状态管理)
- [组件库与UI框架](#组件库与ui框架)
- [开发工具配置](#开发工具配置)
- [构建与部署优化](#构建与部署优化)
- [最佳实践](#最佳实践)

## 环境准备与项目初始化

### 前置条件检查

```bash
# 检查Node.js版本（推荐v16+）
node --version

# 检查npm版本
npm --version

# 检查Vue CLI版本（如果使用Vue CLI）
vue --version
```

### 使用Vite创建项目

```bash
# 使用npm
npm create vite@latest enterprise-vue3-app -- --template vue-ts

# 使用yarn
yarn create vite enterprise-vue3-app --template vue-ts

# 使用pnpm
pnpm create vite enterprise-vue3-app -- --template vue-ts

# 进入项目目录
cd enterprise-vue3-app
```

### 安装核心依赖

```bash
# 安装依赖
npm install
# 或
yarn
# 或
pnpm install

# 添加Vue生态系统依赖
npm install vue-router@4 pinia @vueuse/core
npm install axios unplugin-auto-import unplugin-vue-components

# 添加Element Plus
npm install element-plus @element-plus/icons-vue

# 添加开发依赖
npm install -D sass typescript @types/node unplugin-icons
npm install -D eslint eslint-plugin-vue @typescript-eslint/eslint-plugin @typescript-eslint/parser
npm install -D vitest @vue/test-utils jsdom @testing-library/vue
```

## 项目结构设计

### 推荐的目录结构

```
enterprise-vue3-app/
├── public/                  # 静态资源
├── src/                   
│   ├── api/                # API接口封装
│   │   ├── index.ts
│   │   ├── user.ts
│   │   └── modules/
│   ├── assets/             # 静态资源
│   │   ├── images/
│   │   └── styles/
│   │       ├── variables.scss
│   │       ├── mixins.scss
│   │       └── global.scss
│   ├── components/         # 通用组件
│   │   ├── common/         # 基础组件
│   │   └── business/       # 业务组件
│   ├── composables/        # 组合式函数
│   ├── directives/         # 自定义指令
│   ├── hooks/              # 自定义Hooks
│   ├── layouts/            # 布局组件
│   ├── plugins/            # 插件
│   ├── router/             # 路由配置
│   ├── stores/             # Pinia状态管理
│   ├── types/              # TypeScript类型定义
│   ├── utils/              # 工具函数
│   ├── views/              # 页面组件
│   ├── App.vue
│   ├── main.ts
│   ├── env.d.ts
│   └── auto-imports.d.ts   # 自动导入的类型定义
├── tests/                  # 测试文件
├── .env                    # 环境变量
├── .env.development
├── .env.production
├── vite.config.ts          # Vite配置
├── tsconfig.json           # TypeScript配置
├── tsconfig.node.json      # Node.js TypeScript配置
└── package.json
```

### 创建基础目录和文件

```bash
# 创建目录结构
mkdir -p src/{api,assets/styles,components/{common,business},composables,directives,hooks,layouts,plugins,types,utils}

# 创建基础配置文件
touch src/assets/styles/{variables,mixins,global}.scss
touch src/types/index.ts
touch .env.development .env.production
```

## Composition API实践

### 基础组合式API应用

```vue
<!-- src/components/common/UserProfile.vue -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useUserStore } from '@/stores/user'

interface Props {
  userId: string
}

const props = defineProps<Props>()
const emit = defineEmits<{
  (e: 'update', id: string): void
}>()

// 状态管理
const userStore = useUserStore()

// 响应式状态
const loading = ref(false)
const error = ref<string | null>(null)

// 计算属性
const fullName = computed(() => {
  return `${userStore.user?.firstName || ''} ${userStore.user?.lastName || ''}`
})

// 生命周期钩子
onMounted(async () => {
  loading.value = true
  try {
    await userStore.fetchUserById(props.userId)
  } catch (err) {
    error.value = 'Failed to load user data'
  } finally {
    loading.value = false
  }
})

// 方法
const updateUser = async () => {
  loading.value = true
  try {
    await userStore.updateUser()
    emit('update', props.userId)
  } catch (err) {
    error.value = 'Failed to update user'
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="user-profile">
    <el-card v-loading="loading">
      <template #header>
        <div class="card-header">
          <h3>{{ fullName }}</h3>
        </div>
      </template>
      
      <div v-if="error" class="error-message">
        {{ error }}
      </div>
      
      <div v-else-if="userStore.user" class="user-details">
        <p><strong>Email:</strong> {{ userStore.user.email }}</p>
        <p><strong>Role:</strong> {{ userStore.user.role }}</p>
      </div>
      
      <div class="actions">
        <el-button type="primary" @click="updateUser">Update</el-button>
      </div>
    </el-card>
  </div>
</template>

<style scoped lang="scss">
.user-profile {
  max-width: 600px;
  
  .error-message {
    color: red;
    margin: 10px 0;
  }
  
  .actions {
    margin-top: 20px;
    display: flex;
    justify-content: flex-end;
  }
}
</style>
```

### 自定义组合式函数

```typescript
// src/composables/usePagination.ts
import { ref, computed, watch } from 'vue'
import type { Ref } from 'vue'

interface PaginationOptions {
  pageSize?: number
  currentPage?: number
  total?: number
}

export function usePagination<T>(
  listData: Ref<T[]>,
  options: PaginationOptions = {}
) {
  const pageSize = ref(options.pageSize || 10)
  const currentPage = ref(options.currentPage || 1)
  const total = ref(options.total || 0)
  
  // 计算当前页数据
  const paginatedData = computed(() => {
    const start = (currentPage.value - 1) * pageSize.value
    const end = start + pageSize.value
    return listData.value.slice(start, end)
  })
  
  // 更新页码
  const updatePage = (page: number) => {
    currentPage.value = page
  }
  
  // 更新每页数量
  const updatePageSize = (size: number) => {
    pageSize.value = size
    // 重置到第一页
    if (currentPage.value > 1) {
      currentPage.value = 1
    }
  }
  
  // 监听数据变化，更新总数
  watch(listData, (newData) => {
    total.value = newData.length
  }, { immediate: true })
  
  return {
    pageSize,
    currentPage,
    total,
    paginatedData,
    updatePage,
    updatePageSize
  }
}
```

### 使用自定义Hook

```typescript
// src/hooks/useRequest.ts
import { ref } from 'vue'
import type { Ref } from 'vue'
import axios, { AxiosRequestConfig } from 'axios'

interface RequestOptions<T> extends AxiosRequestConfig {
  onSuccess?: (data: T) => void
  onError?: (error: Error) => void
}

export function useRequest<T = any>() {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)
  
  const run = async (url: string, options: RequestOptions<T> = {}) => {
    loading.value = true
    error.value = null
    
    try {
      const response = await axios(url, options)
      data.value = response.data
      options.onSuccess?.(response.data)
      return response.data
    } catch (err) {
      const errorObj = err as Error
      error.value = errorObj
      options.onError?.(errorObj)
      throw errorObj
    } finally {
      loading.value = false
    }
  }
  
  return {
    data,
    loading,
    error,
    run
  }
}
```

## 路由配置与管理

### 基础路由配置

```typescript
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import type { RouteRecordRaw } from 'vue-router'
import Layout from '@/layouts/Layout.vue'

const routes: Array<RouteRecordRaw> = [
  {
    path: '/login',
    name: 'Login',
    component: () => import('@/views/auth/Login.vue'),
    meta: { 
      title: '登录',
      requiresAuth: false 
    }
  },
  {
    path: '/',
    component: Layout,
    redirect: '/dashboard',
    children: [
      {
        path: 'dashboard',
        name: 'Dashboard',
        component: () => import('@/views/dashboard/Index.vue'),
        meta: { 
          title: '仪表盘',
          icon: 'dashboard',
          requiresAuth: true 
        }
      }
    ]
  },
  {
    path: '/system',
    component: Layout,
    meta: { 
      title: '系统管理',
      icon: 'system',
      requiresAuth: true 
    },
    children: [
      {
        path: 'users',
        name: 'Users',
        component: () => import('@/views/system/Users.vue'),
        meta: { 
          title: '用户管理',
          icon: 'user',
          permissions: ['system:user:view']
        }
      },
      {
        path: 'roles',
        name: 'Roles',
        component: () => import('@/views/system/Roles.vue'),
        meta: { 
          title: '角色管理',
          icon: 'role',
          permissions: ['system:role:view']
        }
      }
    ]
  },
  {
    path: '/:pathMatch(.*)*',
    name: 'NotFound',
    component: () => import('@/views/error/404.vue'),
    meta: { title: '页面不存在' }
  }
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior: (to, from, savedPosition) => {
    return savedPosition || { top: 0 }
  }
})

export default router
```

### 路由守卫配置

```typescript
// src/router/guards.ts
import router from './index'
import { useUserStore } from '@/stores/user'
import NProgress from 'nprogress'
import 'nprogress/nprogress.css'
import { ElMessage } from 'element-plus'
import { getToken } from '@/utils/auth'

NProgress.configure({ showSpinner: false })

const whiteList = ['/login', '/404'] // 白名单路由

router.beforeEach(async (to, from, next) => {
  NProgress.start()

  // 设置页面标题
  document.title = to.meta?.title ? `${to.meta.title} - 企业管理系统` : '企业管理系统'

  const hasToken = getToken()
  
  if (hasToken) {
    if (to.path === '/login') {
      next({ path: '/' })
      NProgress.done()
    } else {
      const userStore = useUserStore()
      const hasUserInfo = userStore.permissions.length > 0
      
      if (hasUserInfo) {
        // 检查页面权限
        if (to.meta?.permissions) {
          const hasPagePermission = userStore.permissions.some((permission: string) =>
            (to.meta.permissions as string[]).includes(permission)
          )
          
          if (hasPagePermission) {
            next()
          } else {
            ElMessage.error('您没有访问此页面的权限')
            next('/404')
          }
        } else {
          next()
        }
      } else {
        try {
          // 获取用户信息和权限
          await userStore.getUserInfo()
          next({ ...to, replace: true })
        } catch (error) {
          // 清除token并重定向到登录页
          await userStore.resetToken()
          ElMessage.error('登录状态异常，请重新登录')
          next(`/login?redirect=${to.path}`)
          NProgress.done()
        }
      }
    }
  } else {
    if (whiteList.includes(to.path)) {
      next()
    } else {
      next(`/login?redirect=${to.path}`)
      NProgress.done()
    }
  }
})

router.afterEach(() => {
  NProgress.done()
})
```

## Pinia状态管理

### Pinia Store配置

```typescript
// src/stores/index.ts
import { createPinia } from 'pinia'
import { markRaw } from 'vue'
import router from '@/router'

// 创建pinia实例
const pinia = createPinia()

// 将路由实例添加到pinia
pinia.use(({ store }) => {
  store.$router = markRaw(router)
})

export default pinia
```

### 用户模块Store

```typescript
// src/stores/user.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { login, logout, getUserInfo } from '@/api/user'
import { getToken, setToken, removeToken } from '@/utils/auth'
import router from '@/router'

interface UserInfo {
  id: string
  name: string
  avatar: string
  introduction?: string
  email: string
  role: string
}

export const useUserStore = defineStore('user', () => {
  // 状态
  const token = ref<string>(getToken() || '')
  const user = ref<UserInfo | null>(null)
  const permissions = ref<string[]>([])
  
  // 动作
  async function login(userInfo: { username: string; password: string }) {
    try {
      const { data } = await login(userInfo)
      setToken(data.accessToken)
      token.value = data.accessToken
      return data
    } catch (error) {
      console.error('Login error:', error)
      throw error
    }
  }
  
  async function getUserInfo() {
    try {
      const { data } = await getUserInfo()
      if (!data) {
        throw new Error('验证失败，请重新登录')
      }
      
      user.value = {
        id: data.id,
        name: data.name,
        avatar: data.avatar,
        introduction: data.introduction,
        email: data.email,
        role: data.role
      }
      
      permissions.value = data.permissions || []
      
      if (!permissions.value || permissions.value.length <= 0) {
        throw new Error('getUserInfo: 权限必须是非空数组!')
      }
      
      return data
    } catch (error) {
      console.error('Get user info error:', error)
      throw error
    }
  }
  
  async function logOut() {
    try {
      await logout()
    } finally {
      resetToken()
      router.push('/login')
    }
  }
  
  function resetToken() {
    removeToken()
    token.value = ''
    user.value = null
    permissions.value = []
  }
  
  return {
    token,
    user,
    permissions,
    login,
    getUserInfo,
    logOut,
    resetToken
  }
})
```

### App状态Store

```typescript
// src/stores/app.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useAppStore = defineStore('app', () => {
  // 状态
  const sidebar = ref({
    opened: true,
    withoutAnimation: false
  })
  
  const device = ref<'desktop' | 'mobile'>('desktop')
  const size = ref<'default' | 'large' | 'small'>('default')
  
  // 计算属性
  const sidebarOpened = computed(() => sidebar.value.opened)
  
  // 动作
  function toggleSideBar() {
    sidebar.value.opened = !sidebar.value.opened
    sidebar.value.withoutAnimation = false
  }
  
  function closeSideBar(withoutAnimation: boolean) {
    sidebar.value.opened = false
    sidebar.value.withoutAnimation = withoutAnimation
  }
  
  function toggleDevice(val: 'desktop' | 'mobile') {
    device.value = val
  }
  
  function setSize(val: 'default' | 'large' | 'small') {
    size.value = val
  }
  
  return {
    sidebar,
    device,
    size,
    sidebarOpened,
    toggleSideBar,
    closeSideBar,
    toggleDevice,
    setSize
  }
})
```

## 组件库与UI框架

### Element Plus集成

```typescript
// src/plugins/element-plus.ts
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'
import zhCn from 'element-plus/es/locale/lang/zh-cn'
import * as ElementPlusIconsVue from '@element-plus/icons-vue'
import type { App } from 'vue'

export default {
  install(app: App) {
    // 注册ElementPlus
    app.use(ElementPlus, {
      locale: zhCn,
      size: 'default'
    })
    
    // 全局注册所有图标
    for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
      app.component(`ElIcon${key}`, component)
    }
  }
}
```

### 封装通用表格组件

```vue
<!-- src/components/common/BaseTable.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'
import type { PropType } from 'vue'

interface Column {
  type?: string
  prop?: string
  label?: string
  width?: string | number
  minWidth?: string | number
  fixed?: boolean | string
  align?: string
  headerAlign?: string
  slot?: string
  sortable?: boolean | string
  formatter?: (row: any, column: any, cellValue: any, index: number) => any
  showOverflowTooltip?: boolean
  [key: string]: any
}

interface Pagination {
  currentPage: number
  pageSize: number
  total: number
  pageSizes?: number[]
  layout?: string
  small?: boolean
  background?: boolean
  pagerCount?: number
  hideOnSinglePage?: boolean
}

const props = defineProps({
  data: {
    type: Array as PropType<any[]>,
    default: () => []
  },
  columns: {
    type: Array as PropType<Column[]>,
    default: () => []
  },
  loading: {
    type: Boolean,
    default: false
  },
  showPagination: {
    type: Boolean,
    default: true
  },
  pagination: {
    type: Object as PropType<Pagination>,
    default: () => ({
      currentPage: 1,
      pageSize: 20,
      total: 0,
      pageSizes: [10, 20, 50, 100],
      layout: 'total, sizes, prev, pager, next, jumper',
      background: true
    })
  },
  // Element Table 原生属性
  height: String,
  maxHeight: String,
  stripe: {
    type: Boolean,
    default: false
  },
  border: {
    type: Boolean,
    default: false
  },
  size: {
    type: String as PropType<'large' | 'default' | 'small'>,
    default: 'default'
  },
  fit: {
    type: Boolean,
    default: true
  },
  showHeader: {
    type: Boolean,
    default: true
  },
  highlightCurrentRow: {
    type: Boolean,
    default: false
  },
  rowClassName: Function as PropType<(row: any, rowIndex: number) => string>,
  rowStyle: Function as PropType<(row: any, rowIndex: number) => object>,
  cellClassName: Function as PropType<(row: any, column: any, rowIndex: number, columnIndex: number) => string>,
  cellStyle: Function as PropType<(row: any, column: any, rowIndex: number, columnIndex: number) => object>,
  headerRowClassName: Function as PropType<(row: any, rowIndex: number) => string>,
  headerRowStyle: Function as PropType<(row: any, rowIndex: number) => object>,
  headerCellClassName: Function as PropType<(row: any, column: any, rowIndex: number, columnIndex: number) => string>,
  headerCellStyle: Function as PropType<(row: any, column: any, rowIndex: number, columnIndex: number) => object>,
  rowKey: [String, Function] as PropType<string | ((row: any) => string)>,
  emptyText: {
    type: String,
    default: '暂无数据'
  },
  defaultExpandAll: {
    type: Boolean,
    default: false
  },
  expandRowKeys: Array,
  defaultSort: Object,
  tooltipEffect: {
    type: String,
    default: 'dark'
  },
  showSummary: {
    type: Boolean,
    default: false
  },
  sumText: {
    type: String,
    default: '合计'
  },
  summaryMethod: Function,
  spanMethod: Function,
  selectOnIndeterminate: {
    type: Boolean,
    default: true
  },
  indent: {
    type: Number,
    default: 16
  },
  lazy: {
    type: Boolean,
    default: false
  },
  load: Function,
  treeProps: Object
})

const emit = defineEmits([
  'selection-change',
  'current-change',
  'size-change',
  'select',
  'select-all',
  'cell-mouse-enter',
  'cell-mouse-leave',
  'cell-click',
  'cell-dblclick',
  'row-click',
  'row-contextmenu',
  'row-dblclick',
  'header-click',
  'header-contextmenu',
  'sort-change',
  'filter-change',
  'header-dragend',
  'expand-change'
])

// 表格实例
const tableRef = ref()

// 分页处理
const handleSizeChange = (val: number) => {
  emit('size-change', val)
}

const handleCurrentPageChange = (val: number) => {
  emit('current-change', val)
}

// 事件转发
const handleSelectionChange = (selection: any[]) => {
  emit('selection-change', selection)
}

const handleSelect = (selection: any[], row: any) => {
  emit('select', selection, row)
}

const handleSelectAll = (selection: any[]) => {
  emit('select-all', selection)
}

const handleCellMouseEnter = (row: any, column: any, cell: HTMLElement, event: Event) => {
  emit('cell-mouse-enter', row, column, cell, event)
}

const handleCellMouseLeave = (row: any, column: any, cell: HTMLElement, event: Event) => {
  emit('cell-mouse-leave', row, column, cell, event)
}

const handleCellClick = (row: any, column: any, cell: HTMLElement, event: Event) => {
  emit('cell-click', row, column, cell, event)
}

const handleCellDblclick = (row: any, column: any, cell: HTMLElement, event: Event) => {
  emit('cell-dblclick', row, column, cell, event)
}

const handleRowClick = (row: any, column: any, event: Event) => {
  emit('row-click', row, column, event)
}

const handleRowContextmenu = (row: any, column: any, event: Event) => {
  emit('row-contextmenu', row, column, event)
}

const handleRowDblclick = (row: any, column: any, event: Event) => {
  emit('row-dblclick', row, column, event)
}

const handleHeaderClick = (column: any, event: Event) => {
  emit('header-click', column, event)
}

const handleHeaderContextmenu = (column: any, event: Event) => {
  emit('header-contextmenu', column, event)
}

const handleSortChange = (param: { column: any; prop: string; order: string }) => {
  emit('sort-change', param)
}

const handleFilterChange = (filters: any) => {
  emit('filter-change', filters)
}

const handleHeaderDragend = (newWidth: number, oldWidth: number, column: any, event: Event) => {
  emit('header-dragend', newWidth, oldWidth, column, event)
}

const handleExpandChange = (row: any, expandedRows: any[]) => {
  emit('expand-change', row, expandedRows)
}

// 公开方法
defineExpose({
  tableRef,
  // 转发El-Table方法
  clearSelection: () => tableRef.value?.clearSelection(),
  toggleRowSelection: (row: any, selected?: boolean) => tableRef.value?.toggleRowSelection(row, selected),
  toggleAllSelection: () => tableRef.value?.toggleAllSelection(),
  toggleRowExpansion: (row: any, expanded?: boolean) => tableRef.value?.toggleRowExpansion(row, expanded),
  setCurrentRow: (row: any) => tableRef.value?.setCurrentRow(row),
  clearSort: () => tableRef.value?.clearSort(),
  clearFilter: (columnKeys?: string[]) => tableRef.value?.clearFilter(columnKeys),
  doLayout: () => tableRef.value?.doLayout(),
  sort: (prop: string, order: string) => tableRef.value?.sort(prop, order)
})
</script>

<template>
  <div class="base-table">
    <el-table
      ref="tableRef"
      v-loading="loading"
      :data="data"
      :height="height"
      :max-height="maxHeight"
      :stripe="stripe"
      :border="border"
      :size="size"
      :fit="fit"
      :show-header="showHeader"
      :highlight-current-row="highlightCurrentRow"
      :row-class-name="rowClassName"
      :row-style="rowStyle"
      :cell-class-name="cellClassName"
      :cell-style="cellStyle"
      :header-row-class-name="headerRowClassName"
      :header-row-style="headerRowStyle"
      :header-cell-class-name="headerCellClassName"
      :header-cell-style="headerCellStyle"
      :row-key="rowKey"
      :empty-text="emptyText"
      :default-expand-all="defaultExpandAll"
      :expand-row-keys="expandRowKeys"
      :default-sort="defaultSort"
      :tooltip-effect="tooltipEffect"
      :show-summary="showSummary"
      :sum-text="sumText"
      :summary-method="summaryMethod"
      :span-method="spanMethod"
      :select-on-indeterminate="selectOnIndeterminate"
      :indent="indent"
      :lazy="lazy"
      :load="load"
      :tree-props="treeProps"
      @select="handleSelect"
      @select-all="handleSelectAll"
      @selection-change="handleSelectionChange"
      @cell-mouse-enter="handleCellMouseEnter"
      @cell-mouse-leave="handleCellMouseLeave"
      @cell-click="handleCellClick"
      @cell-dblclick="handleCellDblclick"
      @row-click="handleRowClick"
      @row-contextmenu="handleRowContextmenu"
      @row-dblclick="handleRowDblclick"
      @header-click="handleHeaderClick"
      @header-contextmenu="handleHeaderContextmenu"
      @sort-change="handleSortChange"
      @filter-change="handleFilterChange"
      @current-change="handleCurrentChange"
      @header-dragend="handleHeaderDragend"
      @expand-change="handleExpandChange"
    >
      <template v-for="(column, index) in columns" :key="index">
        <el-table-column
          v-if="column.type === 'selection'"
          type="selection"
          :width="column.width || 55"
          :fixed="column.fixed"
          :align="column.align || 'center'"
        />
        <el-table-column
          v-else-if="column.type === 'index'"
          type="index"
          :label="column.label || '序号'"
          :width="column.width || 80"
          :fixed="column.fixed"
          :align="column.align || 'center'"
        />
        <el-table-column
          v-else
          :prop="column.prop"
          :label="column.label"
          :width="column.width"
          :min-width="column.minWidth"
          :fixed="column.fixed"
          :render-header="column.renderHeader"
          :sortable="column.sortable"
          :sort-method="column.sortMethod"
          :sort-by="column.sortBy"
          :sort-orders="column.sortOrders"
          :resizable="column.resizable !== false"
          :formatter="column.formatter"
          :show-overflow-tooltip="column.showOverflowTooltip"
          :align="column.align"
          :header-align="column.headerAlign"
          :class-name="column.className"
          :label-class-name="column.labelClassName"
          :selectable="column.selectable"
          :reserve-selection="column.reserveSelection"
          :filters="column.filters"
          :filter-placement="column.filterPlacement"
          :filter-multiple="column.filterMultiple"
          :filter-method="column.filterMethod"
          :filtered-value="column.filteredValue"
        >
          <template #default="scope">
            <slot
              v-if="column.slot"
              :name="column.slot"
              :row="scope.row"
              :column="scope.column"
              :$index="scope.$index"
            />
            <span v-else>{{ scope.row[column.prop as string] }}</span>
          </template>
        </el-table-column>
      </template>
    </el-table>
    
    <!-- 分页 -->
    <el-pagination
      v-if="showPagination"
      v-model:current-page="pagination.currentPage"
      v-model:page-size="pagination.pageSize"
      :page-sizes="pagination.pageSizes"
      :layout="pagination.layout"
      :total="pagination.total"
      :small="pagination.small"
      :background="pagination.background"
      :pager-count="pagination.pagerCount"
      :hide-on-single-page="pagination.hideOnSinglePage"
      class="pagination"
      @size-change="handleSizeChange"
      @current-change="handleCurrentPageChange"
    />
  </div>
</template>

<style lang="scss" scoped>
.base-table {
  .pagination {
    margin-top: 20px;
    text-align: right;
  }
}
</style>
```

## 开发工具配置

### TypeScript配置

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ESNext",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "moduleResolution": "Node",
    "strict": true,
    "jsx": "preserve",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "lib": ["ESNext", "DOM"],
    "skipLibCheck": true,
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "types": ["vite/client", "element-plus/global"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.tsx", "src/**/*.vue"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### Vite配置

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers'
import Icons from 'unplugin-icons/vite'
import IconsResolver from 'unplugin-icons/resolver'

// https://vitejs.dev/config/
export default defineConfig({
  base: './',
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src')
    }
  },
  plugins: [
    vue(),
    AutoImport({
      imports: ['vue', 'vue-router', 'pinia'],
      dts: 'src/auto-imports.d.ts',
      resolvers: [
        ElementPlusResolver(),
        IconsResolver({
          prefix: 'Icon',
        }),
      ],
    }),
    Components({
      resolvers: [
        ElementPlusResolver(),
        IconsResolver({
          enabledCollections: ['ep'],
        }),
      ],
      dts: true
    }),
    Icons({
      autoInstall: true,
    }),
  ],
  server: {
    port: 5173,
    open: true,
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
        rewrite: path => path.replace(/^\/api/, '')
      }
    }
  },
  build: {
    outDir: 'dist',
    assetsDir: 'assets',
    cssCodeSplit: true,
    chunkSizeWarningLimit: 1500,
    rollupOptions: {
      output: {
        manualChunks: {
          'vue': ['vue', 'vue-router', 'pinia'],
          'element-plus': ['element-plus'],
        }
      }
    }
  }
})
```

### ESLint配置

```javascript
// .eslintrc.cjs
module.exports = {
  root: true,
  env: {
    browser: true,
    es2021: true,
    node: true,
  },
  extends: [
    'eslint:recommended',
    'plugin:vue/vue3-recommended',
    'plugin:@typescript-eslint/recommended',
  ],
  parser: 'vue-eslint-parser',
  parserOptions: {
    ecmaVersion: 'latest',
    parser: '@typescript-eslint/parser',
    sourceType: 'module',
  },
  plugins: ['vue', '@typescript-eslint'],
  rules: {
    'vue/multi-word-component-names': 'off',
    '@typescript-eslint/no-explicit-any': 'off',
    '@typescript-eslint/no-unused-vars': ['warn', { 
      argsIgnorePattern: '^_',
      varsIgnorePattern: '^_'
    }],
    'vue/no-v-html': 'off',
    'vue/require-default-prop': 'off',
    'vue/no-unused-components': 'warn',
    'vue/html-self-closing': ['warn', {
      html: {
        void: 'always',
        normal: 'always',
        component: 'always'
      },
      svg: 'always',
      math: 'always'
    }],
    'vue/max-attributes-per-line': ['error', {
      singleline: {
        max: 3
      },      
      multiline: {
        max: 1
      }
    }]
  },
}
```

## 构建与部署优化

### 环境变量配置

```bash
# .env.development
VITE_NODE_ENV=development
VITE_APP_TITLE=企业管理系统（开发）
VITE_APP_API_BASE_URL=http://localhost:3000/api
VITE_APP_ENABLE_MOCK=true

# .env.production
VITE_NODE_ENV=production
VITE_APP_TITLE=企业管理系统
VITE_APP_API_BASE_URL=https://api.example.com
VITE_APP_ENABLE_MOCK=false
```

### Docker部署配置

```dockerfile
# Dockerfile
FROM node:16-alpine as builder

# 设置工作目录
WORKDIR /app

# 复制依赖文件
COPY package.json pnpm-lock.yaml ./

# 安装依赖
RUN npm install -g pnpm && pnpm install --frozen-lockfile

# 复制源代码
COPY . .

# 构建应用
RUN pnpm build

# 生产阶段
FROM nginx:stable-alpine as production

# 复制构建文件
COPY --from=builder /app/dist /usr/share/nginx/html

# 复制nginx配置
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 暴露端口
EXPOSE 80

# 启动nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Nginx配置

```nginx
# nginx.conf
server {
    listen 80;
    server_name localhost;
    
    # gzip配置
    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 6;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png font/woff font/ttf image/svg+xml;
    gzip_vary on;
    
    # 前端资源
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
    
    # API代理
    location /api/ {
        proxy_pass http://backend-service:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        root /usr/share/nginx/html;
        expires 7d;
        add_header Cache-Control "public, max-age=604800, immutable";
    }
    
    # 错误处理
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

## 最佳实践

### 1. 代码规范

```json
// package.json scripts
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "build:analyze": "vite build --mode analyze",
    "preview": "vite preview",
    "lint": "eslint . --ext .vue,.js,.jsx,.cjs,.mjs,.ts,.tsx,.cts,.mts --fix --ignore-path .gitignore",
    "format": "prettier --write src/",
    "test": "vitest",
    "test:coverage": "vitest run --coverage",
    "prepare": "husky install"
  },
  "lint-staged": {
    "*.{vue,js,jsx,cjs,mjs,ts,tsx,cts,mts}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{css,scss,less}": [
      "prettier --write"
    ]
  }
}
```

### 2. Composition API最佳实践

```typescript
// src/utils/composables.ts

// 1. 始终使用setup脚本宏
// <script setup lang="ts"> ... </script>

// 2. 为复杂组件提取逻辑到独立的组合式函数
export function useUserPermissions(role: Ref<string>) {
  // 权限状态
  const canEdit = computed(() => ['admin', 'editor'].includes(role.value))
  const canDelete = computed(() => role.value === 'admin')
  const canExport = computed(() => ['admin', 'editor', 'viewer'].includes(role.value))
  
  // 权限检查函数
  function hasPermission(permission: string): boolean {
    switch (permission) {
      case 'edit': return canEdit.value
      case 'delete': return canDelete.value
      case 'export': return canExport.value
      default: return false
    }
  }
  
  return {
    canEdit,
    canDelete,
    canExport,
    hasPermission
  }
}

// 3. 使用toRefs解构props以保持响应性
// 在组件中:
// const { title, description } = toRefs(props)

// 4. 使用provide/inject进行深层组件通信
export function useAppTheme() {
  // 状态
  const theme = ref<'light' | 'dark'>('light')
  
  // 操作
  function toggleTheme() {
    theme.value = theme.value === 'light' ? 'dark' : 'light'
    document.documentElement.setAttribute('data-theme', theme.value)
  }
  
  // Provider
  provide('app-theme', {
    theme,
    toggleTheme
  })
  
  return {
    theme,
    toggleTheme
  }
}

// 消费者组件中使用inject接收
export function useThemeConsumer() {
  const theme = inject<{
    theme: Ref<'light' | 'dark'>,
    toggleTheme: () => void
  }>('app-theme')
  
  if (!theme) {
    throw new Error('useThemeConsumer must be used within a component that is a descendant of a ThemeProvider')
  }
  
  return theme
}
```

### 3. 性能优化技巧

```typescript
// src/utils/performance.ts

// 1. 使用shallowRef/shallowReactive优化大型对象
export function useDataTable(fetchDataFn: () => Promise<any[]>) {
  // 使用shallowRef优化大型数据
  const tableData = shallowRef<any[]>([])
  
  async function fetchData() {
    tableData.value = await fetchDataFn()
  }
  
  return {
    tableData,
    fetchData
  }
}

// 2. 合理使用v-memo优化重复渲染
// 在模板中: <div v-memo="[item.id, item.selected]">{{ item.name }}</div>

// 3. 虚拟列表优化大数据渲染
export function useVirtualList<T>(list: Ref<T[]>, options: {
  itemHeight: number,
  windowHeight: number
}) {
  const { itemHeight, windowHeight } = options
  
  const visibleCount = computed(() => Math.ceil(windowHeight / itemHeight) + 2)
  const scrollTop = ref(0)
  
  const startIndex = computed(() => {
    return Math.floor(scrollTop.value / itemHeight)
  })
  
  const endIndex = computed(() => {
    return Math.min(startIndex.value + visibleCount.value, list.value.length)
  })
  
  const visibleList = computed(() => {
    return list.value.slice(startIndex.value, endIndex.value)
  })
  
  const offsetY = computed(() => {
    return startIndex.value * itemHeight
  })
  
  const totalHeight = computed(() => {
    return list.value.length * itemHeight
  })
  
  function handleScroll(e: Event) {
    scrollTop.value = (e.target as HTMLElement).scrollTop
  }
  
  return {
    visibleList,
    totalHeight,
    offsetY,
    handleScroll
  }
}

// 4. 使用批处理更新优化
export function useBatchUpdate<T>(list: Ref<T[]>, batchSize = 100) {
  const pending = ref<T[]>([])
  
  function addItems(items: T[]) {
    pending.value.push(...items)
    nextTick(processQueue)
  }
  
  function processQueue() {
    if (pending.value.length === 0) return
    
    const batch = pending.value.splice(0, batchSize)
    list.value.push(...batch)
    
    if (pending.value.length > 0) {
      requestAnimationFrame(processQueue)
    }
  }
  
  return {
    addItems
  }
}
```

### 4. 安全性配置

```typescript
// src/utils/security.ts

// XSS防护
export function escapeHtml(str: string): string {
  const div = document.createElement('div')
  div.appendChild(document.createTextNode(str))
  return div.innerHTML
}

// CSRF Token处理
export function getCSRFToken(): string {
  const meta = document.querySelector('meta[name="csrf-token"]')
  return meta ? meta.getAttribute('content') || '' : ''
}

// 基于角色的权限控制指令
export const permissionDirective = {
  mounted(el: HTMLElement, binding: DirectiveBinding) {
    const { value } = binding
    const userStore = useUserStore()
    
    // 检查权限
    if (value && value.length > 0) {
      const hasPermission = userStore.permissions.some(
        permission => value.includes(permission)
      )
      
      if (!hasPermission) {
        // 从DOM中移除元素
        el.parentNode?.removeChild(el)
      }
    }
  }
}

// 自定义异常处理
export class AppError extends Error {
  code: string
  details?: any
  
  constructor(message: string, code: string, details?: any) {
    super(message)
    this.name = 'AppError'
    this.code = code
    this.details = details
  }
}

// 表单数据验证
export function useFormValidation<T extends object>(rules: Record<keyof T, any[]>) {
  const errors = reactive<Record<keyof T, string>>({} as Record<keyof T, string>)
  const isValid = ref(true)
  
  function validate(data: T): boolean {
    let valid = true
    
    for (const field in rules) {
      const value = data[field]
      const fieldRules = rules[field]
      
      for (const rule of fieldRules) {
        if (rule.required && !value) {
          errors[field] = rule.message || '此字段为必填项'
          valid = false
          break
        }
        
        if (rule.pattern && !rule.pattern.test(value as string)) {
          errors[field] = rule.message || '格式不正确'
          valid = false
          break
        }
        
        if (rule.validator && !rule.validator(value)) {
          errors[field] = rule.message || '验证失败'
          valid = false
          break
        }
      }
    }
    
    isValid.value = valid
    return valid
  }
  
  function resetErrors() {
    for (const field in errors) {
      errors[field] = ''
    }
    isValid.value = true
  }
  
  return {
    errors,
    isValid,
    validate,
    resetErrors
  }
}
```

---
通过本指南，您可以搭建一个功能完整、性能优化、可维护性强的Vue3企业级项目脚手架，充分利用Vue3的新特性和生态系统，满足现代企业级应用开发的各种需求。Vue3的组合式API、TypeScript集成、更好的性能和更小的包体积，为企业级应用开发提供了更为强大的工具和更加优雅的开发体验。

