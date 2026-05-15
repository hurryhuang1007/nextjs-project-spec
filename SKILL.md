---
name: nextjs-project-spec
description: >-
  Enforces Next.js project structure, directory layout, MobX stores, fetchApi,
  Tailwind+CSS modules, modal patterns, and naming conventions for business
  apps. Use when scaffolding, reviewing, or implementing features in Next.js
  projects that follow this specification, or when the user mentions src layout,
  @/com, @/modal, fetchApi, or nextjs-project-spec conventions.
---

# Next.js 项目规范

编写或审查 Next.js 业务项目代码时，遵循本规范。完整 `fetchApi` 模板见 [reference.md](reference.md)。

## 项目目录、结构、配置相关

1. 源码（除去项目配置文件）全部编写在src文件夹下
2. css使用tailwindcss+css modules
3. 前端所使用的资源尽量全部放置在项目仓库中，经webpack打包使用。常见的如图片、小尺寸视频等
4. 媒体资源中 较为通用的放置在asset文件夹中，较为某组件专用的放置在使用的组件旁
5. public目录仅存放需要直接访问到的资源，一般的有协议文件、百度或第三方网站验证文件。（资源文件虽然也可以放在此目录，然后将此目录上传至oss进行访问，但是对于服务器来说 这是多余的资源，而且不经过webpack，网路路径在文件变化时不会改变，浏览器缓存无法失效）
6. webpack路径别名尽量只定义@到src这一个就好
7. next.js通常使用在业务场景，所以模块目录就不要编写index汇总导出文件，按文件路径查找更加方便
8. 模块目录中较为通用且不太好归类的简短功能代码可以存放在命名为common的文件中
9. 较为通用的组件存放在 @/com 文件夹下，通用性不高的直接编写在使用他的页面组件旁边即可
10. 项目文件夹、文件命名全部使用小驼峰命名。（除seo要求的特殊路由命名规则外）
11. 所有弹窗都需要被封装成命令式函数组件，较为通用的 放置在 @/modal 文件夹下，通用性不高的 放置在相关组件文件旁边即可
12. 项目ts定义不需要完全归纳在一个文件夹中，遵循高内聚原则，写在离高内聚代码相邻或相邻文件中即可
13. 是否为测试环境使用 NEXT_PUBLIC_IS_TEST 环境变量来标识
14. 相同功能应尽量使用一个库完成

### 项目目录规定

- app：页面路由文件夹
- asset：资源文件夹
- com：通用组件文件夹
- config：项目配置，通常一些定值或通过简单判断返回定值的函数
- module：拥有生命周期或特定时机需要被使用的方法以及模块，常见的如：项目载入被单次执行的代码逻辑、项目用于管理某种资源或业务逻辑的管理器。（一般很少用到此目录，因为这种代码较多时，通常被认为是不健康的）
- service：api定义文件夹（不使用api名称是为了区别于next的api模块）
- store：项目通用共享状态管理store文件夹
- util：工具函数文件夹，通常要求是一些无生命周期，不是在特定时机被使用的函数
- modal：通用弹窗组件文件夹
- type（在项目根目录）：全局无需显性导入的通用ts定义以及类型工具函数

## 项目通用共享状态管理

1. store模块使用oop编程思想，内聚程度较高的状态集中在一个store中。相关属性更改、更新以及监听逻辑代码也写在同一个store文件中
2. 使用mobx，不使用装饰器
3. 尽量减少全局store数量，允许页面/组件拥有自己的私有store
4. 全局store通常只需要一个：userInfo

## fetch api封装

接口请求包括sse请求使用自行封装好的fetchApi，不允许使用任何第三方封装。实现与选项说明见 [reference.md](reference.md) 中的模板。

api异常情况捕捉逻辑需要根据项目自身情况来调整，没有相关信息则提示用户自己按需优化。

## 杂项

1. 功能较为复杂，后续迭代可能性较高的函数以及类构造函数入参应为对象。这样可以避免写出较为丑陋传参方式
2. 组件应尽可能是服务端组件，因为服务端组件有着较为便捷的数据获取方式以及更好的用户体验
3. 客户端组件使用异步函数，应使用ahooks中的useRequest调用
4. 函数缓存以及异步函数调用请求合并，可使用fn-merge-cache库
5. 流式接口数据/文字输出平滑处理，可使用smooth-output库

## Agent 执行清单

新增或修改代码时自检：

- [ ] 源码在 `src/` 下，路径别名仅用 `@` → `src`
- [ ] 目录/文件小驼峰；无模块 `index`  barrel 导出
- [ ] 资源走 webpack（`asset/` 或组件旁）；`public/` 仅直链必需文件
- [ ] 通用组件 `@/com`，通用弹窗 `@/modal`；弹窗为命令式封装
- [ ] API 经 `fetchApi`，无 axios 等第三方 HTTP 封装
- [ ] 全局 store 以 MobX OOP、无装饰器；全局通常仅 `userInfo`
- [ ] 优先 Server Component；客户端异步用 `useRequest`
- [ ] 测试环境用 `NEXT_PUBLIC_IS_TEST`
