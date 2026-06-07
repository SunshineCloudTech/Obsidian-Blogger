---
title: Vue2企业级脚手架搭建完整指南
date: 2024-06-04
tags:
  - Vue2
  - 前端架构
  - 脚手架搭建
  - vue-router
  - 项目实战
description: 详细介绍Vue2企业级项目脚手架的搭建过程，包括项目初始化、路由配置、组件设计、状态管理、构建部署等完整流程。
icon: code
category:
  - 编程相关
  - 前端开发
index: true
cover: https://picsum.photos/1200/600?random=25
---
# Vue2企业级脚手架搭建完整指南

## 概述

Vue2作为主流的前端框架，在企业级项目中应用广泛。本文详细介绍如何从零开始搭建一个完整的Vue2企业级项目脚手架，包含路由管理、状态管理、组件库集成、构建优化等企业级特性。

## 目录

- [环境准备与项目初始化](#环境准备与项目初始化)
- [项目结构设计](#项目结构设计)
- [路由配置与管理](#路由配置与管理)
- [状态管理集成](#状态管理集成)
- [组件库与UI框架](#组件库与ui框架)
- [开发工具配置](#开发工具配置)
- [构建与部署优化](#构建与部署优化)
- [最佳实践](#最佳实践)

## 环境准备与项目初始化

### 前置条件检查

```bash
# 检查Node.js版本（推荐v14+）
node --version

# 检查npm版本
npm --version

# 检查Vue CLI版本
vue --version
```

### 安装Vue CLI并创建项目

```bash
# 全局安装Vue CLI（如果未安装）
npm install -g @vue/cli

# 创建Vue2项目
vue create enterprise-vue-app

# 选择配置选项
? Please pick a preset: Manually select features
? Check the features needed for your project:
 ◉ Babel
 ◉ TypeScript
 ◉ Progressive Web App (PWA) Support
 ◉ Router
 ◉ Vuex
 ◉ CSS Pre-processors
 ◉ Linter / Formatter
 ◉ Unit Testing
 ◉ E2E Testing

# 进入项目目录
cd enterprise-vue-app
```

### 完整项目初始化配置

```bash
# 添加常用依赖
npm install --save axios element-ui vue-i18n nprogress js-cookie

# 添加开发依赖
npm install --save-dev @types/js-cookie compression-webpack-plugin

# 安装代码质量工具
npm install --save-dev husky lint-staged @commitlint/cli @commitlint/config-conventional
```

## 项目结构设计

### 推荐的目录结构

```
enterprise-vue-app/
├── public/                 # 静态资源
├── src/                   
│   ├── api/               # API接口封装
│   │   ├── index.ts
│   │   ├── user.ts
│   │   └── modules/
│   ├── assets/            # 静态资源
│   │   ├── images/
│   │   ├── styles/
│   │   │   ├── variables.scss
│   │   │   ├── mixins.scss
│   │   │   └── global.scss
│   │   └── fonts/
│   ├── components/        # 通用组件
│   │   ├── common/        # 基础组件
│   │   └── business/      # 业务组件
│   ├── directives/        # 自定义指令
│   ├── filters/           # 过滤器
│   ├── layouts/           # 布局组件
│   ├── mixins/            # 混入
│   ├── plugins/           # 插件
│   ├── router/            # 路由配置
│   ├── store/             # Vuex状态管理
│   ├── types/             # TypeScript类型定义
│   ├── utils/             # 工具函数
│   ├── views/             # 页面组件
│   ├── App.vue
│   └── main.ts
├── tests/                 # 测试文件
├── .env                   # 环境变量
├── .env.development
├── .env.production
├── vue.config.js          # Vue配置
└── package.json
```

### 创建基础目录和文件

```bash
# 创建目录结构
mkdir -p src/{api,assets/styles,components/{common,business},directives,filters,layouts,mixins,plugins,types,utils}

# 创建基础配置文件
touch src/assets/styles/{variables,mixins,global}.scss
touch src/types/index.ts
touch .env.development .env.production
```

## 路由配置与管理

### 基础路由配置

```typescript
// src/router/index.ts
import Vue from 'vue'
import VueRouter, { RouteConfig } from 'vue-router'
import Layout from '@/layouts/Layout.vue'

Vue.use(VueRouter)

const routes: Array<RouteConfig> = [
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
    path: '/404',
    name: '404',
    component: () => import('@/views/error/404.vue'),
    meta: { title: '页面不存在' }
  },
  {
    path: '*',
    redirect: '/404'
  }
]

const router = new VueRouter({
  mode: 'history',
  base: process.env.BASE_URL,
  routes,
  scrollBehavior: () => ({ y: 0 })
})

export default router
```

### 路由守卫配置

```typescript
// src/router/guards.ts
import router from './index'
import store from '@/store'
import NProgress from 'nprogress'
import 'nprogress/nprogress.css'
import { Message } from 'element-ui'

NProgress.configure({ showSpinner: false })

const whiteList = ['/login', '/404'] // 白名单路由

router.beforeEach(async (to, from, next) => {
  NProgress.start()

  // 设置页面标题
  document.title = to.meta?.title ? `${to.meta.title} - 企业管理系统` : '企业管理系统'

  const hasToken = store.getters.token
  
  if (hasToken) {
    if (to.path === '/login') {
      next({ path: '/' })
      NProgress.done()
    } else {
      const hasPermissions = store.getters.permissions.length > 0
      
      if (hasPermissions) {
        // 检查页面权限
        if (to.meta?.permissions) {
          const hasPagePermission = store.getters.permissions.some((permission: string) =>
            to.meta.permissions.includes(permission)
          )
          
          if (hasPagePermission) {
            next()
          } else {
            Message.error('您没有访问此页面的权限')
            next('/404')
          }
        } else {
          next()
        }
      } else {
        try {
          // 获取用户信息和权限
          await store.dispatch('user/getUserInfo')
          next()
        } catch (error) {
          // 清除token并重定向到登录页
          await store.dispatch('user/resetToken')
          Message.error('登录状态异常，请重新登录')
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

### 动态路由生成

```typescript
// src/utils/permission.ts
import { RouteConfig } from 'vue-router'
import Layout from '@/layouts/Layout.vue'

const modules = require.context('@/views', true, /\.vue$/)

// 动态导入组件
const importComponent = (view: string) => {
  return () => modules(`./${view}.vue`)
}

// 根据权限生成路由
export function generateRoutes(permissions: string[]): RouteConfig[] {
  // 这里可以从后端获取路由配置
  const asyncRoutes: RouteConfig[] = [
    {
      path: '/advanced',
      component: Layout,
      meta: { 
        title: '高级功能',
        permissions: ['advanced:view']
      },
      children: [
        {
          path: 'analysis',
          name: 'Analysis',
          component: importComponent('advanced/Analysis'),
          meta: { 
            title: '数据分析',
            permissions: ['advanced:analysis:view']
          }
        }
      ]
    }
  ]

  return filterAsyncRoutes(asyncRoutes, permissions)
}

// 过滤有权限的路由
function filterAsyncRoutes(routes: RouteConfig[], permissions: string[]): RouteConfig[] {
  const accessedRoutes: RouteConfig[] = []

  routes.forEach(route => {
    const temp = { ...route }
    
    if (hasPermission(permissions, temp)) {
      if (temp.children) {
        temp.children = filterAsyncRoutes(temp.children, permissions)
      }
      accessedRoutes.push(temp)
    }
  })

  return accessedRoutes
}

// 检查权限
function hasPermission(permissions: string[], route: RouteConfig): boolean {
  if (route.meta && route.meta.permissions) {
    return permissions.some(permission => 
      route.meta.permissions.includes(permission)
    )
  }
  return true
}
```

## 状态管理集成

### Vuex Store结构

```typescript
// src/store/index.ts
import Vue from 'vue'
import Vuex from 'vuex'
import { IAppState } from './modules/app'
import { IUserState } from './modules/user'

Vue.use(Vuex)

export interface IRootState {
  app: IAppState
  user: IUserState
}

export default new Vuex.Store<IRootState>({})
```

### 用户模块Store

```typescript
// src/store/modules/user.ts
import { VuexModule, Module, Action, Mutation, getModule } from 'vuex-module-decorators'
import { login, logout, getUserInfo } from '@/api/user'
import { getToken, setToken, removeToken } from '@/utils/auth'
import router, { resetRouter } from '@/router'
import store from '@/store'

export interface IUserState {
  token: string
  name: string
  avatar: string
  introduction: string
  permissions: string[]
  email: string
}

@Module({ dynamic: true, store, name: 'user' })
class User extends VuexModule implements IUserState {
  public token = getToken() || ''
  public name = ''
  public avatar = ''
  public introduction = ''
  public permissions: string[] = []
  public email = ''

  @Mutation
  private SET_TOKEN(token: string) {
    this.token = token
  }

  @Mutation
  private SET_NAME(name: string) {
    this.name = name
  }

  @Mutation
  private SET_AVATAR(avatar: string) {
    this.avatar = avatar
  }

  @Mutation
  private SET_INTRODUCTION(introduction: string) {
    this.introduction = introduction
  }

  @Mutation
  private SET_PERMISSIONS(permissions: string[]) {
    this.permissions = permissions
  }

  @Mutation
  private SET_EMAIL(email: string) {
    this.email = email
  }

  @Action
  public async Login(userInfo: { username: string; password: string }) {
    let { username, password } = userInfo
    username = username.trim()
    
    const { data } = await login({ username, password })
    setToken(data.accessToken)
    this.SET_TOKEN(data.accessToken)
  }

  @Action
  public ResetToken() {
    removeToken()
    this.SET_TOKEN('')
    this.SET_PERMISSIONS([])
  }

  @Action
  public async GetUserInfo() {
    if (this.token === '') {
      throw Error('GetUserInfo: token is undefined!')
    }
    
    const { data } = await getUserInfo()
    if (!data) {
      throw Error('Verification failed, please Login again.')
    }
    
    const { permissions, name, avatar, introduction, email } = data
    
    // permissions must be a non-empty array
    if (!permissions || permissions.length <= 0) {
      throw Error('GetUserInfo: permissions must be a non-null array!')
    }
    
    this.SET_PERMISSIONS(permissions)
    this.SET_NAME(name)
    this.SET_AVATAR(avatar)
    this.SET_INTRODUCTION(introduction)
    this.SET_EMAIL(email)
  }

  @Action
  public async LogOut() {
    if (this.token === '') {
      throw Error('LogOut: token is undefined!')
    }
    await logout()
    removeToken()
    resetRouter()
    
    // Reset visited views and cached views
    // to fixed https://github.com/PanJiaChen/vue-element-admin/issues/2485
    TagsViewModule.delAllViews()
    
    this.SET_TOKEN('')
    this.SET_PERMISSIONS([])
  }
}

export const UserModule = getModule(User)
```

## 组件库与UI框架

### Element UI集成

```typescript
// src/plugins/element.ts
import Vue from 'vue'
import {
  Button,
  Form,
  FormItem,
  Input,
  Table,
  TableColumn,
  Pagination,
  Dialog,
  MessageBox,
  Message,
  Loading
} from 'element-ui'

Vue.use(Button)
Vue.use(Form)
Vue.use(FormItem)
Vue.use(Input)
Vue.use(Table)
Vue.use(TableColumn)
Vue.use(Pagination)
Vue.use(Dialog)
Vue.use(Loading.directive)

Vue.prototype.$msgbox = MessageBox
Vue.prototype.$alert = MessageBox.alert
Vue.prototype.$confirm = MessageBox.confirm
Vue.prototype.$prompt = MessageBox.prompt
Vue.prototype.$message = Message
```

### 全局组件注册

```typescript
// src/components/index.ts
import Vue from 'vue'

// 基础组件
import BaseTable from '@/components/common/BaseTable.vue'
import BaseForm from '@/components/common/BaseForm.vue'
import BaseDialog from '@/components/common/BaseDialog.vue'

// 业务组件
import UserSelector from '@/components/business/UserSelector.vue'
import DepartmentTree from '@/components/business/DepartmentTree.vue'

const components = [
  BaseTable,
  BaseForm,
  BaseDialog,
  UserSelector,
  DepartmentTree
]

export default {
  install() {
    components.forEach(component => {
      Vue.component(component.name, component)
    })
  }
}
```

### 封装通用表格组件

```vue
<!-- src/components/common/BaseTable.vue -->
<template>
  <div class="base-table">
    <el-table
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
          :resizable="column.resizable"
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
            <span v-else>{{ scope.row[column.prop] }}</span>
          </template>
        </el-table-column>
      </template>
    </el-table>
    
    <!-- 分页 -->
    <el-pagination
      v-if="showPagination"
      :current-page="pagination.currentPage"
      :page-sizes="pagination.pageSizes"
      :page-size="pagination.pageSize"
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

<script lang="ts">
import { Component, Vue, Prop, Emit } from 'vue-property-decorator'

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
  formatter?: Function
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

@Component({
  name: 'BaseTable'
})
export default class BaseTable extends Vue {
  @Prop({ type: Array, default: () => [] }) private data!: any[]
  @Prop({ type: Array, default: () => [] }) private columns!: Column[]
  @Prop({ type: Boolean, default: false }) private loading!: boolean
  @Prop({ type: Boolean, default: true }) private showPagination!: boolean
  @Prop({ type: Object, default: () => ({
    currentPage: 1,
    pageSize: 20,
    total: 0,
    pageSizes: [10, 20, 50, 100],
    layout: 'total, sizes, prev, pager, next, jumper',
    background: true
  }) }) private pagination!: Pagination

  // Element Table 原生属性
  @Prop({ type: String, default: '' }) private height!: string
  @Prop({ type: String, default: '' }) private maxHeight!: string
  @Prop({ type: Boolean, default: false }) private stripe!: boolean
  @Prop({ type: Boolean, default: false }) private border!: boolean
  @Prop({ type: String, default: 'medium' }) private size!: string
  @Prop({ type: Boolean, default: true }) private fit!: boolean
  @Prop({ type: Boolean, default: true }) private showHeader!: boolean
  @Prop({ type: Boolean, default: false }) private highlightCurrentRow!: boolean
  @Prop({ type: Function }) private rowClassName!: Function
  @Prop({ type: Function }) private rowStyle!: Function
  @Prop({ type: Function }) private cellClassName!: Function
  @Prop({ type: Function }) private cellStyle!: Function
  @Prop({ type: Function }) private headerRowClassName!: Function
  @Prop({ type: Function }) private headerRowStyle!: Function
  @Prop({ type: Function }) private headerCellClassName!: Function
  @Prop({ type: Function }) private headerCellStyle!: Function
  @Prop({ type: [String, Function] }) private rowKey!: string | Function
  @Prop({ type: String, default: '暂无数据' }) private emptyText!: string
  @Prop({ type: Boolean, default: false }) private defaultExpandAll!: boolean
  @Prop({ type: Array }) private expandRowKeys!: any[]
  @Prop({ type: Object }) private defaultSort!: object
  @Prop({ type: String, default: 'dark' }) private tooltipEffect!: string
  @Prop({ type: Boolean, default: false }) private showSummary!: boolean
  @Prop({ type: String, default: '合计' }) private sumText!: string
  @Prop({ type: Function }) private summaryMethod!: Function
  @Prop({ type: Function }) private spanMethod!: Function
  @Prop({ type: Boolean, default: true }) private selectOnIndeterminate!: boolean
  @Prop({ type: Number, default: 16 }) private indent!: number
  @Prop({ type: Boolean, default: false }) private lazy!: boolean
  @Prop({ type: Function }) private load!: Function
  @Prop({ type: Object }) private treeProps!: object

  @Emit('selection-change')
  private handleSelectionChange(selection: any[]) {
    return selection
  }

  @Emit('current-change')
  private handleCurrentPageChange(currentPage: number) {
    return currentPage
  }

  @Emit('size-change')
  private handleSizeChange(pageSize: number) {
    return pageSize
  }

  // Element Table 事件处理
  @Emit('select')
  private handleSelect(selection: any[], row: any) {
    return { selection, row }
  }

  @Emit('select-all')
  private handleSelectAll(selection: any[]) {
    return selection
  }

  @Emit('cell-mouse-enter')
  private handleCellMouseEnter(row: any, column: any, cell: any, event: Event) {
    return { row, column, cell, event }
  }

  @Emit('cell-mouse-leave')
  private handleCellMouseLeave(row: any, column: any, cell: any, event: Event) {
    return { row, column, cell, event }
  }

  @Emit('cell-click')
  private handleCellClick(row: any, column: any, cell: any, event: Event) {
    return { row, column, cell, event }
  }

  @Emit('cell-dblclick')
  private handleCellDblclick(row: any, column: any, cell: any, event: Event) {
    return { row, column, cell, event }
  }

  @Emit('row-click')
  private handleRowClick(row: any, column: any, event: Event) {
    return { row, column, event }
  }

  @Emit('row-contextmenu')
  private handleRowContextmenu(row: any, column: any, event: Event) {
    return { row, column, event }
  }

  @Emit('row-dblclick')
  private handleRowDblclick(row: any, column: any, event: Event) {
    return { row, column, event }
  }

  @Emit('header-click')
  private handleHeaderClick(column: any, event: Event) {
    return { column, event }
  }

  @Emit('header-contextmenu')
  private handleHeaderContextmenu(column: any, event: Event) {
    return { column, event }
  }

  @Emit('sort-change')
  private handleSortChange(param: { column: any; prop: string; order: string }) {
    return param
  }

  @Emit('filter-change')
  private handleFilterChange(filters: any) {
    return filters
  }

  @Emit('current-change')
  private handleCurrentChange(currentRow: any, oldCurrentRow: any) {
    return { currentRow, oldCurrentRow }
  }

  @Emit('header-dragend')
  private handleHeaderDragend(newWidth: number, oldWidth: number, column: any, event: Event) {
    return { newWidth, oldWidth, column, event }
  }

  @Emit('expand-change')
  private handleExpandChange(row: any, expandedRows: any[]) {
    return { row, expandedRows }
  }
}
</script>

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
    "target": "esnext",
    "lib": [
      "esnext",
      "dom",
      "dom.iterable",
      "es6"
    ],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "preserve",
    "baseUrl": ".",
    "paths": {
      "@/*": [
        "src/*"
      ]
    },
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.tsx",
    "src/**/*.vue",
    "tests/**/*.ts",
    "tests/**/*.tsx"
  ],
  "exclude": [
    "node_modules"
  ]
}
```

### ESLint配置

```js
// .eslintrc.js
module.exports = {
  root: true,
  env: {
    node: true
  },
  extends: [
    'plugin:vue/essential',
    '@vue/standard',
    '@vue/typescript/recommended'
  ],
  parserOptions: {
    ecmaVersion: 2020
  },
  rules: {
    'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
    'no-debugger': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
    '@typescript-eslint/no-explicit-any': 'off',
    '@typescript-eslint/explicit-module-boundary-types': 'off',
    'vue/no-unused-components': 'warn',
    'vue/no-unused-vars': 'warn',
    'space-before-function-paren': ['error', 'never'],
    'comma-dangle': ['error', 'never']
  },
  overrides: [
    {
      files: [
        '**/__tests__/*.{j,t}s?(x)',
        '**/tests/unit/**/*.spec.{j,t}s?(x)'
      ],
      env: {
        jest: true
      }
    }
  ]
}
```

### Prettier配置

```js
// .prettierrc.js
module.exports = {
  semi: false,
  singleQuote: true,
  trailingComma: 'none',
  endOfLine: 'auto',
  tabWidth: 2,
  useTabs: false,
  printWidth: 100,
  bracketSpacing: true,
  arrowParens: 'avoid'
}
```

## 构建与部署优化

### Vue配置优化

```javascript
// vue.config.js
const path = require('path')
const CompressionWebpackPlugin = require('compression-webpack-plugin')
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin

function resolve(dir) {
  return path.join(__dirname, dir)
}

// 生产环境下的优化
const isProd = process.env.NODE_ENV === 'production'

module.exports = {
  publicPath: isProd ? '/enterprise-app/' : '/',
  outputDir: 'dist',
  assetsDir: 'static',
  lintOnSave: process.env.NODE_ENV === 'development',
  productionSourceMap: false,
  
  devServer: {
    port: 8080,
    open: true,
    overlay: {
      warnings: false,
      errors: true
    },
    proxy: {
      '/api': {
        target: process.env.VUE_APP_API_BASE_URL,
        changeOrigin: true,
        pathRewrite: {
          '^/api': ''
        }
      }
    }
  },

  configureWebpack: config => {
    // 路径别名
    config.resolve.alias = {
      '@': resolve('src'),
      '@components': resolve('src/components'),
      '@views': resolve('src/views'),
      '@assets': resolve('src/assets'),
      '@utils': resolve('src/utils')
    }

    if (isProd) {
      // 生产环境优化
      config.plugins.push(
        // Gzip压缩
        new CompressionWebpackPlugin({
          filename: '[path][base].gz',
          algorithm: 'gzip',
          test: /\.(js|css|html|svg)$/,
          threshold: 8192,
          minRatio: 0.8
        })
      )

      // 打包分析（可选）
      if (process.env.ANALYZE) {
        config.plugins.push(new BundleAnalyzerPlugin())
      }

      // 代码分割优化
      config.optimization = {
        splitChunks: {
          chunks: 'all',
          cacheGroups: {
            vendor: {
              name: 'chunk-vendors',
              test: /[\\/]node_modules[\\/]/,
              priority: 10,
              chunks: 'initial'
            },
            elementUI: {
              name: 'chunk-elementUI',
              priority: 20,
              test: /[\\/]node_modules[\\/]_?element-ui(.*)/
            },
            commons: {
              name: 'chunk-commons',
              test: resolve('src/components'),
              minChunks: 3,
              priority: 5,
              reuseExistingChunk: true
            }
          }
        }
      }
    }
  },

  chainWebpack: config => {
    // 预加载优化
    config.plugin('preload').tap(() => [
      {
        rel: 'preload',
        include: 'initial',
        fileBlacklist: [/\.map$/, /hot-update\.js$/, /runtime\..*\.js$/]
      }
    ])

    // 预获取优化
    config.plugin('prefetch').tap(options => {
      options[0].fileBlacklist = options[0].fileBlacklist || []
      options[0].fileBlacklist.push(/chunk-vendors\..*\.js$/)
      return options
    })

    // SVG处理
    config.module
      .rule('svg')
      .exclude.add(resolve('src/assets/icons'))
      .end()
    config.module
      .rule('icons')
      .test(/\.svg$/)
      .include.add(resolve('src/assets/icons'))
      .end()
      .use('svg-sprite-loader')
      .loader('svg-sprite-loader')
      .options({
        symbolId: 'icon-[name]'
      })
      .end()
  },

  css: {
    extract: isProd,
    sourceMap: false,
    loaderOptions: {
      scss: {
        prependData: `@import "@/assets/styles/variables.scss";`
      }
    }
  },

  pluginOptions: {
    'style-resources-loader': {
      preProcessor: 'scss',
      patterns: [
        path.resolve(__dirname, 'src/assets/styles/variables.scss')
      ]
    }
  }
}
```

### 环境变量配置

```bash
# .env.development
NODE_ENV=development
VUE_APP_API_BASE_URL=http://localhost:3000/api
VUE_APP_TITLE=企业管理系统（开发）

# .env.production  
NODE_ENV=production
VUE_APP_API_BASE_URL=https://api.example.com
VUE_APP_TITLE=企业管理系统
```

### Docker部署配置

```dockerfile
# Dockerfile
FROM node:16-alpine as build-stage

# 设置工作目录
WORKDIR /app

# 复制依赖文件
COPY package*.json ./

# 安装依赖
RUN npm ci --only=production

# 复制源代码
COPY . .

# 构建应用
RUN npm run build

# 生产阶段
FROM nginx:stable-alpine as production-stage

# 复制构建文件
COPY --from=build-stage /app/dist /usr/share/nginx/html

# 复制nginx配置
COPY nginx.conf /etc/nginx/nginx.conf

# 暴露端口
EXPOSE 80

# 启动nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Nginx配置

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    
    sendfile        on;
    keepalive_timeout  65;
    
    # Gzip压缩
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rss+xml
        application/vnd.geo+json
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        font/opentype
        image/bmp
        image/svg+xml
        image/x-icon
        text/cache-manifest
        text/css
        text/plain
        text/vcard
        text/vnd.rim.location.xloc
        text/vtt
        text/x-component
        text/x-cross-domain-policy;

    server {
        listen       80;
        server_name  localhost;
        
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
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
        
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
}
```

## 最佳实践

### 1. 代码规范

```bash
# package.json scripts
{
  "scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build",
    "build:analyze": "ANALYZE=true npm run build",
    "lint": "vue-cli-service lint",
    "lint:fix": "vue-cli-service lint --fix",
    "test:unit": "vue-cli-service test:unit",
    "test:e2e": "vue-cli-service test:e2e",
    "commit": "git-cz"
  },
  "gitHooks": {
    "pre-commit": "lint-staged",
    "commit-msg": "commitlint -E GIT_PARAMS"
  },
  "lint-staged": {
    "*.{js,jsx,vue,ts,tsx}": [
      "vue-cli-service lint --fix",
      "git add"
    ]
  }
}
```

### 2. 性能优化技巧

```typescript
// src/utils/performance.ts

// 路由懒加载
export const lazyLoad = (view: string) => {
  return () => import(`@/views/${view}.vue`)
}

// 组件懒加载
export const asyncComponent = (importFunc: () => Promise<any>) => {
  return () => ({
    component: importFunc(),
    loading: () => import('@/components/common/Loading.vue'),
    error: () => import('@/components/common/Error.vue'),
    delay: 200,
    timeout: 10000
  })
}

// 防抖函数
export function debounce(func: Function, delay: number) {
  let timeoutId: NodeJS.Timeout
  return function(...args: any[]) {
    clearTimeout(timeoutId)
    timeoutId = setTimeout(() => func.apply(this, args), delay)
  }
}

// 节流函数
export function throttle(func: Function, delay: number) {
  let lastCall = 0
  return function(...args: any[]) {
    const now = Date.now()
    if (now - lastCall >= delay) {
      lastCall = now
      func.apply(this, args)
    }
  }
}
```

### 3. 安全性配置

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

// 权限验证
export function checkPermission(permission: string | string[]): boolean {
  const userPermissions = store.getters.permissions
  
  if (Array.isArray(permission)) {
    return permission.some(p => userPermissions.includes(p))
  }
  
  return userPermissions.includes(permission)
}
```

### 4. 监控与错误处理

```typescript
// src/utils/errorHandler.ts
import { Message } from 'element-ui'

// 全局错误处理
Vue.config.errorHandler = (err, vm, info) => {
  console.error('Vue Error:', err, info)
  
  // 发送错误到监控服务
  if (process.env.NODE_ENV === 'production') {
    sendErrorToMonitoring({
      error: err.message,
      stack: err.stack,
      component: vm?.$options.name || 'Unknown',
      info
    })
  }
  
  Message.error('系统发生错误，请稍后重试')
}

// Promise错误处理
window.addEventListener('unhandledrejection', event => {
  console.error('Unhandled Promise Rejection:', event.reason)
  
  if (process.env.NODE_ENV === 'production') {
    sendErrorToMonitoring({
      error: event.reason,
      type: 'unhandledRejection'
    })
  }
  
  event.preventDefault()
})

function sendErrorToMonitoring(errorInfo: any) {
  // 实现错误监控逻辑
  fetch('/api/errors', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      ...errorInfo,
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent,
      url: location.href
    })
  }).catch(console.error)
}
```

---
通过本指南，您可以搭建一个功能完整、性能优化、可维护性强的Vue2企业级项目脚手架，满足现代企业级应用开发的各种需求。

