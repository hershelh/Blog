---
title: Webpack 5 实践：你不知道的 Tree Shaking
description: 探究 Webpack Tree Shaking 原理与实践
date: 2022-06-03T22:36:00.000+00:00
lang: zh
duration: 17min
---

本篇文章从 什么是 Tree Shaking、如何使用 Tree Shaking、Tree Shaking 的原理：`usedExports` 和 `sideEffects` 以及 如何实践 Tree Shaking 和相关注意事项四个维度剖析 Tree Shaking，希望对你有所帮助。

## 什么是 Tree Shaking

Tree Shaking 是一个术语，通常用于描述移除 JavaScript 上下文中的未引用代码 (dead-code)。它依赖于 ES2015 模块语法的静态结构特性，通过在运行过程中静态分析模块之间的导入导出，确定 ESM 模块中哪些导出值未曾被其它模块使用，并将其删除，以此实现打包产物的优化。

Tree Shaking 较早前由 Rich Harris 在 Rollup 中率先实现，Webpack 自 2.0 版本开始接入，至今已经成为一种应用广泛的性能优化手段。

## 启用 Tree Shaking

在 Webpack5 中，Tree Shaking 在生产环境下默认启动。如果想在开发环境启动 Tree Shaking，需要如下配置：

   -   配置 `optimization.usedExports` 为 true，启动标记功能；

   -   启动代码优化功能，可以通过如下方式实现：

        -   配置 `optimization.minimize = true`；
        -   提供 `optimization.minimizer` 数组。

当然，使用 Tree Shaking 的大前提是使用 ESM 规范语法来编写你的模块。那么为什么使用 CommonJs、AMD 等模块化方案无法支持 Tree Shaking 呢？

因为在 CommonJs、AMD、CMD 等旧版本的 JavaScript 模块化方案中，导入导出行为是高度动态，难以预测的，例如：

```ts
if (process.env.NODE_ENV === 'development') {
  require('./bar')
  exports.foo = 'foo'
}
```

   而 ESM 方案则从规范层面规避这一行为，它要求所有的导入导出语句只能出现在模块顶层，且导入导出的模块名必须为字符串常量，这意味着下述代码在 ESM 方案下是非法的：

```ts
if (process.env.NODE_ENV === 'development') {
  import bar from 'bar'
  export const foo = 'foo'
}
```

   所以，ESM 下模块之间的依赖关系是高度确定的，与运行状态无关，编译工具只需要对 ESM 模块做静态分析，就可以从代码字面量中推断出哪些模块值未曾被其它模块使用，这是实现 Tree Shaking 技术的必要条件。

示例：

```ts
// src/math.js
export function square(x) {
  return x * x
}

export function cube(x) {
  return x * x * x
}
```
```ts
// src/index.js
import { cube } from './math.js'

console.log(cube(5))
```
```ts
// webpack.config.js
const path = require('path')

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'dist'),
  },
  mode: 'development',
  optimization: {
    usedExports: true,
  },
}
```

   （需要将 `mode` 配置设置成 `development` 以确定 bundle 不会被压缩。）

   该示例中，`math.js` 导出了 `square`、`cube` 两个函数，而 `index.js` 仅仅导入并调用了 `cube` 函数，我们并没有从 `math.js` 中 `import` 另外一个 `square` 方法，因此这个函数体就是所谓的“未引用代码(dead code)”，

   查看打包结果：

```ts
/***/ (function (module, __webpack_exports__, __webpack_require__) {
  'use strict'
  /* unused harmony export square */
  /* harmony export (immutable) */ __webpack_exports__.a = cube
  function square(x) {
    return x * x
  }

  function cube(x) {
    return x * x * x
  }
})
```

   可以看到，`square` 函数的导出语句被 shake 掉，接下来只要启用压缩工具就可将 `square` 的定义清除掉以达到完整的 Tree Shaking 效果。

   使用以下三个配置均可启用代码压缩工具：

   -   配置 `mode = production`；
   -   配置 `optimization.minimize = true`；
   -   提供 `optimization.minimizer` 数组。

## Tree Shaking 原理探索

### `optimization.usedExports`

通过上述示例，我们知道要启用 Webpack 的 Tree Shaking 功能，需配置 `optimization.usedExports` 为 true，那么该字段的作用是什么呢？

`usedExports` 用于在 Webpack 编译过程中启动标记功能，它会将每个模块中没有被使用过的导出内容标记为 `unused`，当生成产物时，被标记的变量对应的导出语句会被删除。

当然，仅仅删除未使用变量的导出语句是不够的，若 Webpack 配置启用了代码压缩工具，如 Terser 插件，那么在打包的最后它还会删除所有引用被标记内容的代码语句，这些语句一般称作 Dead Code。可以说，真正执行 Tree Shake 操作的是 Terser 插件。

但是，并不是所有 Dead Code 都会被 Terser 删除。沿用以上示例：

```ts
// src/math.js
export function square(x) {
  return x * x
}

export function cube(x) {
  return x * x * x
}
```
```ts
// src/index.js
import { cube } from './math.js'

console.log(square(10))
console.log(cube(5))
```

   我们添加一条打印语句，它打印了调用 `squre` 函数的返回结果，`index.js` 保留原样。按照我们之前的设想，打包后会删除与 `squre` 函数相关的代码语句，即 `squre` 函数的声明语句、打印语句都会被删除。

   打包结果：

   
![image-20220326211348176.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/148500021ee1472e9dc46215b7f5170a~tplv-k3u1fbpfcp-watermark.image?)

   可以看到，`math.js` 模块中，`square` 函数的痕迹被完全清除，但是打印语句仍然被保留。这是因为，这条语句存在**副作用**。
   
   **副作用（side effect）** 的定义是，在导入时会执行特殊行为的代码，而不是仅仅暴露一个 export 或多个 export。例如 polyfill，它影响全局作用域，因而存在副作用。

显然，以上示例的 `console.log()` 语句存在副作用。Terser 在执行 Tree Shaking 时，会保留存在副作用的代码，而不是将其删除。

Terser 为什么选择不删除存在副作用的语句呢？因为**有副作用不代表有害**，例如 polyfill ，它会影响全局作用域，但是可以让我们使用 ES6+ 来书写代码而不必考虑目标浏览器的兼容性。

事实上，要判断一串存在副作用的代码是否对项目”有害“是非常麻烦的，Terser 尝试去解决这个问题，但在很多情况下，它不太确定。但这并不意味着 terser 由于无法解决这些问题而运作得不好，而是由于在 JavaScript 这种动态语言中实在很难去确定。因此 Terser 采取保守策略，选择将副作用保留。

作为开发者，如果你非常清楚某条语句会被判别有副作用但其实是无害的，应该被删除，可以使用 `/*#__PURE__*/` 注释，来向 terser 传递信息，表明这条语句是纯的，没有副作用，terser 可以放心将它删除：

```ts
// src/math.js
export function square(x) {
  return x * x
}

export function cube(x) {
  return x * x * x
}
/* #__PURE__ */ console.log(square(10))
```

   打包结果：

   
![image-20220326213939041.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f6fd70426384502804308948ea0d97e~tplv-k3u1fbpfcp-watermark.image?)

   可以看到，`console.log` 语句已被删除。
### `"sideEffects"`

#### 探索

与 `/*#__PURE__*/` 注释类似，`"sideEffects"` 也可以标记不存在副作用的内容，与前者不同的是，它作用于模块层面。

`"sideEffects"` 是 `package.json` 的一个字段，默认值为 `true`。如果你非常清楚你的 package 是纯粹的，不包含副作用，那么可以简单地将该属性标记为 `false`，来告知 webpack 可以安全地删除未被使用的代码（Dead Code）；如果你的 package 中有些模块确实有一些副作用，可以改为提供一个数组：

```json
// package.json
{
  "name": "your-project",
  "sideEffects": ["./src/some-side-effectful-file.js"]
}
```

为了更清楚地表达 `"sideEffects"` 字段的意图，我们创建一个 package：

```json
// package.json
{
  "name": "mypackage",
  "main": "index.js"
}
```
```ts
// index.js
export * from './math.js'
export * from './print.js'

// math.js
export function square(x) {
  return x * x
}
export function cube(x) {
  return x.sum(x)
}
console.log(square(10))

// print.js
export function print() {
  console.log('Hello World!')
}
```

   然后使用 `npm link` 在全局创建一个指向该 package 文件位置的符号链接，然后在另一个项目中使用 `npm link mypackage` 引入该 package。

   在项目的 `index.js` 中，我们引入但仅调用该 package 的 `cube` 函数：

```ts
import { cube } from 'mypackage'
cube(5)
```

   准备工作完毕。我们先不使用 `sideEffects` 字段，仅开启 `usedExports`。结合前面对该字段的阐述，我们知道它会标记出未被使用的导出内容，打包时 terser 就会将引用被标记内容的语句删除。
    
   打包结果如下：
    
   ![image-20220326222056336.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4734b0a722e841c59022244c7f8a8946~tplv-k3u1fbpfcp-watermark.image?)
   
   可以看到打包结果符合我们的预期：`square` 和 `print` 函数的痕迹被清除，`console.log` 语句由于具有副作用所以没有被删除。
    
   - 注：`index` 模块的 `a` 函数是压缩混淆前的运行时函数 `__webpack_require__`，用于导入指定的模块以支撑 bundle 的模块化特性。
    
   但是可以看到，`print` 模块仍然被保留，尽管它的内容为空，保留它并不会造成什么影响，但是难免引起项目冗余，而且 `index` 中仍然导入了 `print` 模块，代码执行过程中难免会有性能损耗，另外如果该模块是一个 async chunk 的话还会造成额外的网络开销。为了将这些冗余的模块 shake 干净，我们可以使用 `sideEffects` 字段。
    
   `print` 模块之所以不会被删除掉，是 `sideEffects` 字段默认为 true 的缘故，导致 package 中包括 `print` 在内的所有模块都被标记为有副作用，因此 terser 不会贸然将它们删除。 所以，我们可以这样设置：

```json
// package.json
{
  "name": "mypackage",
  "main": "index.js",
  "sideEffects": ["./index.js"]
}
```

   仅标记 `index` 模块为有副作用，其他模块没有副作用，我们再来打包看看：
    
   ![image-20220326224031988.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1df849b01cb042fdbd48b4b1151279b9~tplv-k3u1fbpfcp-watermark.image?)
    
   可以看到 `print` 模块被删除，并且 `index` 中对 `print` 的导入语句也被清除了！
    
   接下来更进一步，全部设置为无副作用试试：

```json
// package.json
{
  "name": "mypackage",
  "main": "index.js",
  "sideEffects": false
}
```

   打包构建，结果如下：
    
   ![image-20220326225556305.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4184d52f87b6409f824def894a28a0b8~tplv-k3u1fbpfcp-watermark.image?)
    
   可以看到作为入口文件的 `index` 模块也被删除了，仅保留了 `math` 模块。所以设置了 `"sideEffects": false` ，表明整个 package 不存在任何副作用，Webpack 可以安心执行 Tree Shaking 了。
    
   前面都在讲 JS 文件，我们再来看看存在 CSS 文件的情况。
    
   在项目下新建一个 CSS 文件，然后修改 index 的内容：

```css
// style.css
.hello-world {
  color: red;
}
```
```ts
// index.js
import './style.css'
```

   我们在项目根目录的 `package.json` 中设置 `sideEffetcs` 字段的值为 false 来达到完整的 Tree Shaking 效果。打包结果如下：
    
   ![image-20220326232104735.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/170d7e7b446b4fb4b08d3e5b6ef3e63b~tplv-k3u1fbpfcp-watermark.image?)
    
   打包结果竟然为空，我们不是将 CSS 文件 import 进来了吗，怎么会被删除呢？
    
   这是因为，在打包过程中，`css-loader` 会将 CSS 文件转译为导出该文件中所有 CSS 规则集的 JS 模块。而我们在 index 中并没有导入它的导出值，仅仅是简单的将其 import 进来，导致这个 ”CSS 模块“ 的导出值被标记为 unused，由于还被标记为无副作用，所以整个模块就被删除了。
    
   因此，当项目中存在 CSS 文件时，我们就不能简单粗暴的将 `sideEffects` 标记为 false 了。
    
#### 结论

`sideEffetcs` 作用于整个模块，它不会分析整个模块内部的代码是否具有副作用：

   -   当你对模块设置了 `"sideEffects": false`，就表明这个模块没有副作用，相当于告诉 Webpack：喂！我没有副作用啊，如果我的导出值没有被别的模块使用那就请把我清除掉吧！
   -   当你对模块设置了 `"sideEffects": true`，就表明这个模块有副作用，相当于告诉 Webpack：喂！我有副作用啊，就算我没有被别的模块导入（指导出值被使用）也不要把我清除啊！

因此，对于 CSS 文件，需要使用 `sideEffects` 标记所有 CSS 文件，来保留所有 CSS 文件，以及对 CSS 文件的导入语句。

如果你仍想对 CSS 文件使用 `"sideEffects: false"`，并且想保留这个 CSS 文件，可以这样：
        
   ![image-20220325125037647.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42fb554b7e3941959fb6986fffd5f7d6~tplv-k3u1fbpfcp-watermark.image?)
    
   这样的话，CSS 文件的导出值（默认导出值）被消费，Terser 就不会将其 shake 掉。
        
### 总结

Webpack 的 Tree Shaking 机制由 `optimization.usedExports` 和`sideEffects` 共同承担，两者都具备 Tree Shake 掉多余代码的功能：

   -   `usedExports` 作用于代码语句层面，依赖于 terser 去检测语句中的副作用；
   -   `sideEffects` 作用于模块层面，用于标记整个模块的副作用。

`usedExports` 和 terser 在生产环境下默认开启，它会删除项目所有模块中未被引用的导出变量以及对应的导出语句，同时保留具有副作用的语句。

被标记为 `sideEffects: false` 的模块，如果导出值未被引用，在打包后会被删除。

## Tree Shaking 实践

### 应用程序

如果我们所开发的是一个应用程序（application），为了达到最佳的 Tree Shaking 效果，是不是要在项目下的 `package.json` 中设置 `sideEffects: false` 呢？

答案是否定的，在日常开发中，除了手动 import CSS 文件之外，我们还经常会使用 `MiniCssExtractPlugin` 将所有 CSS 从它们所在的 chunk 中抽离出来成为单独的文件，以利用并行加载和按需加载来优化网页加载性能，这意味着，如果设置了 `sideEffects: false` 的话打包时 Webpack 就会将它们删除。因此我们需要改用数组语法来标记它们的副作用：

```json
// package.json
{
  "sideEffects": ["**/*.css"]
}
```

    
   这看起来似乎是个最佳实践，我们保留了 CSS 文件，同时又删除了未被引用的模块。让我们看看管不管用：

![image-20220327133059498.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb10b6a83ae34fc5b8224687fcc52d0b~tplv-k3u1fbpfcp-watermark.image?)

   可以看到除了手动引入的 CSS 文件以外，剩下的 CSS 文件全都被删除。尽管我们已经标记了项目下的 CSS 文件的副作用，但是很明显，被 `MiniCssExtractPlugin` 分离出的 CSS 文件并不在 `"sideEffects"` 标记列表内。

因此，在应用程序中使用 `"sideEffects"` 会导致无法预料的后果，而且使用它的收益也不会很高，因为项目中的模块我们基本都会引用，没有被引用的也不会被 Webpack 纳入模块依赖图。

因此，个人不建议在应用程序中使用 `"sideEffects"`。

Vant 组件库的[文档](https://vant-contrib.gitee.io/vant/v4/#/zh-CN/quickstart#dao-ru-suo-you-zu-jian-bu-tui-jian)中，推荐使用 [babel-plugin-import](https://github.com/ant-design/babel-plugin-import) 插件来引入组件，它会在编译过程中将 import 语句自动转换为按需引入的形式：

```ts    
// 原始代码
import { Button } from 'vant'

// 编译后代码
import Button from 'vant/es/button'
import 'vant/es/button/style'
```

实际上，不使用 [babel-plugin-import](https://github.com/ant-design/babel-plugin-import) ，仅使用上面的原始代码也可以导入组件，并且支持 Tree Shaking。很多人对上例中原始代码的导入方式有误解，认为从 `"vant"` 路径导入组件的方式不支持 Tree Shaking，而从 `'vant/es/button'` 路径导入组件的方式就支持 Tree Shaking。对于不使用 ESM 的库确实如此，比如 `lodash`，但是对于使用 ESM 的库，两种引入方式就都一样了。一个库支不支持 Tree Shaking 取决于这个库打包出的 bundle 是否是 ESM 语法仅此而已。而 Vant 明显满足这个条件。

Vant 推荐使用 [babel-plugin-import](https://github.com/ant-design/babel-plugin-import) 的原因就是它可以自动引入组件，可以省去手动引入的麻烦。

实际上，[babel-plugin-import](https://github.com/ant-design/babel-plugin-import) 的作用不止如此。它的强大之处在于它能让不使用 ESM 的库支持 "Tree Shaking"，比如 `lodash`。原因很简单，因为它能把原始的导入语句转换为更加精确的导入语句：

```ts
// 原始代码
import { random } from 'lodash'
// 转换后
import { random } from 'lodash/random'
```

   转换之后的导入语句仅仅导入 `lodash` 的 `random` 模块而不是整个 `lodash` 库，因此 Webpack 打包时也仅打包 `random` 而不是整个 `lodash`，从而达到类似于 Tree Shaking 的效果。

综上，如果你是一个应用程序的开发者，想要达到最佳的 Tree Shaking 效果，你应该这样做：

   -   （个人见解）使用 `optimization.usedExports` 而不是 `"sideEffects"`。前者在生产环境下默认启动，换句话说，你什么也不用做；

   -   优先使用按需引入的方式导入项目所需要的组件、API；

   -   优先使用支持 ESM 语法并设置了 `"sideEffects"` 的库版本。如果你所使用的库并未设置 `"sideEffects"`，那就给作者提个 issue 吧！

     
        ![image-20220327145501325.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8134b120463b4f5ab96a4b057fbd2e8d~tplv-k3u1fbpfcp-watermark.image?)

### 库

`"sideEffects"` 的强大之处，体现在它能**大大减少项目所引用的包的体积**。如果项目所引用的包支持 ESM 模块语法，且设置了 `"sideEffects: false"`，那么在打包时 Webpack 就能删除包中所有未被引入的代码，减少 bundle 体积。

诸如 `vue`、`vuex`、`vue-router` 的 `package.json` 都添加了 `"sideEffects": false`。

因此，如果你是一个库（library）的开发者，你应该在你的 `package.json`中设置 `"sideEffects"`，并打包出使用 ESM 格式的 bundle，以支持 Tree Shaking。

然而，令人遗憾的是，Webpack 尚不支持打包 ESM 格式的 bundle：


![image-20220327150215351.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67d3b668d9d74fbf84476135d2b6b29f~tplv-k3u1fbpfcp-watermark.image?)

因此对于库开发者，推荐使用 [Rollup](https://rollupjs.org/guide/en/) 作为构建工具，仅需如下配置：

```ts
// rollup.config.js
export default {
  // ...,
  output: {
    file: 'bundle.es.js',
    format: 'es'
  }
}
```

   就能打包出使用 ESM 格式的 bundle。

但是，我们在开发一个库的过程中还要考虑兼容性的问题，很明显打包出 ESM 格式的 bundle 的话旧浏览器是无法支持的，并且出于构建性能的考虑， [Vue CLI 等脚手架所集成的 babel-loader](https://github.com/vuejs/vue-docs-zh-cn/blob/master/vue-cli-plugin-babel/README.md) 默认情况下会排除 `node_modules` 内部的文件。用户如果使用了我们发布的使用 ESM的包就必须配置复杂的规则以把我们的包加入编译的白名单。

因此为了能在支持 Tree Shaking 的同时又能兼容低版本的浏览器，最佳实践是**打包出两个版本的 bundle**，一份使用 ESM 规范语法以支持 Tree Shaking，一份使用其它模块语法如 CommonJS 做回退处理。这需要使用 `package.json` 的 `module` 字段。

使用了 `package.json` 的 `module` 字段之后，当打包工具遇到我们的模块时：

- 如果它支持 `module` 字段，则会优先解析该字段所指定的文件；

- 如果它还无法识别 `module` 字段，则会解析 `main` 字段所指定的文件。

因此我们可以**把 `module` 字段的值指定为使用 ESM 语法的 bundle 路径，把 `main` 字段指定为使用其它模块语法的 bundle 路径**。

查看 vuex 的 `package.json` 会发现它也是这么做的：


![image-20220327153532631.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29ba9923471d4f2baccdf8e54c00b6ca~tplv-k3u1fbpfcp-watermark.image?)
## *参考链接*

-   [Webpack 原理系列九：Tree-Shaking 实现原理](https://mp.weixin.qq.com/s/McigcfZyIuuA-vfOu3F7VQ)；
-   [Tree Shaking](https://webpack.docschina.org/guides/tree-shaking/#mark-the-file-as-side-effect-free)；
-   [聊聊 package.json 文件中的 module 字段](https://loveky.github.io/2018/02/26/tree-shaking-and-pkg.module/)。