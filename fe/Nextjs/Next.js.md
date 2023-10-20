# Next.js

## 启动

> [官方文档](https://nextjs.org/docs/getting-started/installation)

```shell
npx create-next-app@latest

What is your project named? my-app
Would you like to use TypeScript? No / Yes
Would you like to use ESLint? No / Yes
Would you like to use Tailwind CSS? No / Yes
Would you like to use `src/` directory? No / Yes
Would you like to use App Router? (recommended) No / Yes
Would you like to customize the default import alias (@/*)? No / Yes
What import alias would you like configured? @/*
```



## 文件结构

> https://nextjs.org/docs/getting-started/project-structure



## 路由映射

> https://nextjs.org/docs/app/building-your-application/routing

1、`app/layout.tsx` 根布局

2、`app/page.tsx` 根页面。根页面的内容会被挂载到根布局进行渲染，访问根页面 `http://localhost:3000`

3、其他页面

* `app/dashboard/page.tsx`，访问 `http://localhost:3000/dashboard`
* `app/goods/page.tsx`，访问 `http://localhost:3000/goods`
* `app/goods/detail/page.tsx`，访问 `http://localhost:3000/goods/detail`
* …



### Layout

> The app directory **must** include a root layout.

> A layout is UI that is **shared** between multiple pages. On navigation, layouts preserve state, remain interactive, and do not re-render. Layouts can also be [nested](https://nextjs.org/docs/app/building-your-application/routing/pages-and-layouts#nesting-layouts).

#### 共享组件

```tsx
import Navbar from '@/components/Navbar'
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div>
      <Navbar /> {/* Include shared UI here e.g. a header or sidebar or navbar */}
      {children}
    <div/>
  )
}
```

#### 布局嵌套

1、层级目录

```
\---src
    +---app
    |   |   layout.tsx
    |   |   page.tsx
    |   |
    |   +---dashboard
    |       |   layout.tsx
    |       |   page.jsx
    |       |
    |       \---setting
    |               page.jsx
```

2、访问方式

* 访问根路径 `http://localhost:3000`
* 访问 dashboard 路径 `http://localhost:3000/dashboard`

…



### Page

> A page is UI that is **unique** to a route.

…

---

## 参考

* https://nextjs.org/docs/app/building-your-application