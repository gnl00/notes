# Vue.js



<br>

## 开始

**Start with webpack**

1、安装`Vue CLI`

```shell

yarn global add @vue/cli
# 或
npm install -g @vue/cli
```

2、创建项目

```shell
# Vue CLI 2
vue init webpack 项目名
# 回车确认
babel yes
eslint no
router optional
runtimeonly
test no

# Vue CLI 3
vue create 项目名
# 空格选择，回车确认
# 新建项目完成之后从 Vue CLI 2 项目中复制 .browserslistrc
# 新建 vue.config.js 设置项目路径别名

# 启动vue图形界面
vue ui
```



**Start with Vite**

```shell
# npm 6.x
npm init vite@latest YourProjectName --template vue

# npm 7+，需要加上额外的双短横线
npm init vite@latest YourProjectName -- --template vue

# yarn
yarn create @vitejs/app YourProjectName
```



<br>

## 基础

> 稍后补充…

**Vue 3 装饰器**



```shell
npm install -S vue-property-decorator
```

`vue-property-decorator`的装饰器：

- `@Prop`
- `@PropSync`
- `@Provide`
- `@Model`
- `@Watch`
- `@Inject`
- `@Provide`
- `@Emit`
- `@Component`
- `Mixins` 

<br>

`vuex-class`的装饰器：

- `@State`
- `@Getter`
- `@Action`
- `@Mutation`



<br>

## Vue Router

> 稍后补充…



### 路由模式

1、**Hash**，地址栏 URL 中的`#`符号（此 hash 不是密码学里的散列运算）。比如 URL：`http://www.abc.com/#/hello `，hash 的值为 `#/hello`。

**hash 虽然出现在 URL 中，但不会被包括在 HTTP 请求中，对后端完全没有影响，因此改变 hash 不会重新加载页面**。

> URL的 hash URL 的 hash 也就是锚点（#）， 本质上是改变 `window.location` 的`href`属性。 我们可以通过直接赋值 `location.hash` 来改变`href`，但是页面不发生刷新



2、**History**，利用了 **HTML5 History Interface** 中新增的 **pushState() 和 replaceState() 方法**。（需要特定浏览器支持）这两个方法应用于浏览器的**历史记录栈**，在当前已有的 **back()、forward()、go()** 的基础之上，**它们提供了对历史记录进行修改的功能**。**当它们执行修改时，虽然改变了当前的 URL，但浏览器不会立即向后端发送请求。**
因此可以说，hash 模式和 history 模式都属于浏览器自身的特性，Vue-Router 只是利用了这两个特性（通过调用浏览器提供的接口）来实现前端路由



> 根据 Mozilla Develop Network 的介绍，调用 **history.pushState() 相比于直接修改 hash，存在以下优势**：

1、**pushState()** **设置的新 URL 可以是**与**当前 URL 同源的任意 URL**；而 **hash 只可修改 # 后面的部分，因此只能设置与当前 URL 同文档的 URL；**

2、**pushState() 设置的新 URL 可以与当前 URL 一模一样，这样也会把记录添加到栈中；而 hash 设置的新值必须与原来不一样才会触发动作将记录添加到栈中**；
3、**pushState()** **通过 stateObject 参数**可以**添加任意类型的数据到记录中；而 hash 只可添加短字符串；**
4、**pushState() 可额外设置 title 属性供后续使用**。



**总结**

1、**hash 模式下，仅 hash 符号之前的内容会被包含在请求中**，如 `http://www.abc.com/#/aaa`，对于后端来说，即使没有做到对路由的全覆盖，也不会返回 404 错误。

2、**history 模式下，前端的 URL 必须和实际向后端发起请求的 URL 一致，如 `http://www.abc.com/book/id` 如果后端缺少对 `/book/id` 的路由处理，将返回 404 错误。**

> Vue-Router 官网里如此描述：“不过这种模式要玩好，还需要后台配置支持……所以呢，你要在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 index.html 页面，这个页面就是你 app 依赖的页面。”

3、对于一般的 `Vue + Vue-Router + Webpack + XXX `形式的 Web 开发场景，用 history 模式即可，只需在后端或 Nginx 进行简单的路由配置，同时搭配前端路由的 404 页面支持。



<br>

## Vuex

> 稍后补充…



<br>

## eslint

**关闭 eslint**

```js
// vue.config.js
module.exports = defineConfig({
  lintOnSave: false
)}
```



<br>

## axios

**安装**

```shell
npm install axios -S
```



**使用**

```js
// request/axios.js
import axios from "axios";

const config = {
    baseURL: '/api',
    timeout: 10000
}

// 创建实例
const instance = axios.create(config)

// 请求拦截器
instance.interceptors.request.use(
  request => {
    // 进行 token 检查或者必须参数检查

    // 做完处理之后需要 return 请求
    return request
  }
)

// 响应拦截器
instance.interceptors.response.use(
  response => {
    // 检查响应数据
  if (response.data != null) {

    // 仅关心 data 数据
      return response.data
  }
}, error => {
    return Promise.reject(error.response.data)
  })

export default instance
```

```js
// 例子 
// request/xxx.js
import instance from "./axios";

const getString = () => {
    return instance({
        method: 'GET',
        // 请求地址为 协议://IP:port/api/getString
        url: '/getString'
    })
}

export default getString
```

```js
// xxx组件.js
const fetchData = () => {
    getString().then(res => {
        console.log(res)
    }).catch(err => {
        console.log(err)
    })
}
```

<br>

**跨域配置**

```js
// axios.js
import axios from "axios";

const config = {
    baseURL: '/api',
    timeout: 10000
}

// vue.config.js
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8866/',
        changeOrigin: true,
        rewrite: path => path.replace(/^\/api/, '')
      }
    }
  }
})
```



<br>

## TypeScript

**`setup`语法糖**

> 在 setup 语法中不能像以前一样使用 this 关键字获取属性，因为会找不到组件的实例；setup 的调用发生在 data、computed、或 methods 被解析之前

**用法一**

```js
<script lang="ts">
export default {
  name: "MyComponent",
  setup(props, context) {
    // props 父子组件通信
    // Attribute （非响应式对象）
    console.log(context.attrs)
    // Slot 插槽（非响应式对象）
    console.log(context.slots)
    // 触发事件（方法）
    console.log(context.emit)
  }
}
<script>
```

<br>

**用法二（推荐）**

```js
<script setup lang="ts">
<script>
```



<br>

## TailWind CSS

**开始**

1、安装

```shell
npm install tailwindcss@latest postcss@latest autoprefixer@latest -D


# 如果将 Tailwind 与依赖于旧版本 PostCSS 的工具集成在一起，可能会看到如下错误（一般是使用 webpack 的时候）
# Error: PostCSS plugin tailwindcss requires PostCSS 8
# 在这种情况下，安装 PostCSS 7 兼容性版本
# 卸载 Tailwind 并使用兼容性版本重新安装
npm uninstall tailwindcss postcss autoprefixer
npm install tailwindcss@npm:@tailwindcss/postcss7-compat @tailwindcss/postcss7-compat postcss@^7 autoprefixer@^9
```

2、创建配置文件

```shell
# 在工程根目录创建一个最小的 tailwind.config.js 文件
npx tailwindcss init -p
```

```js
// tailwind.config.js
module.exports = {
  content: [
    './public/**/*.html',
    './src/**/*.{js,jsx,ts,tsx,vue}',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}

```

3、将 Tailwind 引入到 CSS 中

```css
/* 
创建一个 CSS 文件，请使用 @tailwind 指令注入 Tailwind 的基础 (base)，组件 (components) 和功能 (utilities) 样式
*/
/* tailwindcss.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

4、在 main 文件中全局引入

```js
import "./assets/tailwindcss.css";
```

5、CSS 文件中引入模板指令

```css
@tailwind components;

@layer components {
    @apply h-full w-full
}
```

6、在标签中使用

```html
<div>
    <p class="text-lg text-purple-600 font-bold">Tailwindcss quick looks</p>
</div>
```

7、常用居中布局

```html
grid 布局内容居中
<div class="grid place-items-center h-screen">
  Centered using Tailwind Grid
</div>

flex 布局内容居中
<div class="flex justify-center items-center h-screen">
  Centered using Tailwind Grid
</div>
```











































## 参考

[Use history mode for router? Vue-router 中hash模式和history模式的区别](https://blog.csdn.net/cplvfx/article/details/107294535)

[vue-router的两种模式（hash和history）及区别](https://blog.csdn.net/xiaokanfuchen86/article/details/105879101/)

[axios 跨域](https://www.jianshu.com/p/ca7d5918cebe)



[Tailwind CSS 中文文档](https://www.tailwindcss.cn/docs)

[TailWind CSS 文档 提取组件](https://www.jianshu.com/p/ea06102e5bb9)



https://www.iconfinder.com/
