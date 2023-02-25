---
title: 前端自动化测试入门
description: 学习前端自动化测试的基本概念、目的和步骤
date: 2022-12-17T01:08:00.000+00:00
lang: zh
duration: 1h
---

## 前端程序员所要进行的测试

作为前端开发人员，当构建一个 Web 或其他类型的应用时，从被测试对象的角度，可以进行以下三类测试：

- 单元测试。前面提到，单元测试的大部分工作应该由开发人员完成，前端程序员也是如此。我们需要对单个独立的函数、类或一个组合式函数、hook 进行测试，将其与应用的其他部分隔离开来。而且应该进行功能测试，侧重于被测单元在功能上的正确性，而非进行兼容性测试、性能测试等其他测试。而且，由于前端应用的特殊性，为了创建一个与外界隔离的环境，我们往往需要模拟应用环境的很大一部分，如第三方模块、网络请求等；

- 组件测试。如今大多数 Web 应用都会使用 Vue、React 这类提倡组件化开发的框架进行开发，因此对所编写的组件进行测试在前端测试中应当占据比较大的比重。组件测试需要检查组件是否正常挂载或渲染、是否可以正常交互，以及表现是否符合预期；

- 端到端（E2E）测试。当完成单元测试和组件测试之后，我们还需要进行端到端测试，将整个应用部署到实际运行环境或模拟真实运行环境下运行，从用户的视角对整个应用进行从头到尾、端到端的业务测试，确保应用表现符合预定需求。端到端测试除了测试用户界面的真实交互效果和逻辑行为外，还会发起真实的网络请求，向后端的数据库获取数据，并且在真实浏览器上运行，捕捉应用运行出错时的快照、视频等信息。端到端测试可以说是一种系统功能测试。当然，（自动化的）端到端测试也不是非要做，也不是非要前端做，在实际开发过程中还应结合实际情况选择合适的测试方案。

除了进行以上三种功能测试外，前端程序员还可进行性能测试，如借助浏览器的 LightHouse、Performance 功能检测页面的渲染、交互性能。还可进行兼容性测试等其他测试。由于不是本文重点内容，就不进行介绍了。

对一个庞大的应用进行单元测试、组件测试和端到端测试，往往需要设计大量的测试用例，执行多次且重复的测试，要想大幅缩短在测试上所花费的时间，自然就需要用到自动化测试，通过使用测试工具、编写测试脚本来提高测试效率，所幸前端领域经过这么多年的发展，在社区上早已出现了很多优秀的开源测试工具。接下来，我将介绍如何利用测试工具进行自动化测试，编写测试脚本，让你全面地入门自动化测试。

## 开始入门

如果现在要你测试以下这个函数，你要怎么做？

```ts
function sum(a, b) {
  return a + b
}
```

第一步自然是设计测试用例，比如输入 1 和 2，这个函数会输出 3。设计好测试用例之后，当然就要让这个函数跑起来，传入 1 和 2，打印函数返回值看看是否为 3。于是可以写出以下这段测试代码：

```js
console.log(sum(1, 2))
```

然后运行这段代码，检查打印结果是否为 3。这样，一个测试代码就完成了。当然，这个函数过于简单，用静态测试的方法也能进行测试，这里只是方便举例。除此之外，这段代码运行起来还都需要人工观察运行结果来检验测试成果，这其实也不属于自动化测试的范畴。

当类似的测试做多了之后我们就可以发现一个规律，大多数测试用例，都是设计一个或多个输入数据，以及对应的输出数据，通过传入这些输入数据时被测代码是否产生或返回这些输出数据来判断被测代码是否运行正常，这个判断的过程就叫作**断言（assertion）**。

### 断言

Node 的 assert 模块就提供了进行断言的 API，比如使用 equal 方法对上述的 sum 函数进行断言，可以这样：

```js
assert.equal(sum(1, 2), 3)
```

运行这段代码，如果 sum 函数的实现不符合预期，equal 方法就会抛出一个 AssertionError 错误，并打印详细的错误原因。

除了 Node 提供的 assert 模块外，社区还出现了很多断言库，提供了多样的断言风格，最具代表性的当属 [Chai](https://www.chaijs.com/) 和 [Jest](https://jestjs.io/)。

#### Chai

Chai 提供了三种不同的断言风格供用户选择。

##### assert

assert 风格与 Node 的 assert 模块类似，但是提供了更多 API，并且可以在浏览器上运行：

```js
const assert = require('chai').assert
const foo = 'bar'

assert.typeOf(foo, 'string') // without optional message
assert.typeOf(foo, 'string', 'foo is a string') // with optional message
assert.equal(foo, 'bar', 'foo equal `bar`')
```

assert 风格的 API 允许使用者传入一个可选的描述断言行为的字符串到最后一个参数，当断言失败后错误信息中就会显示这个字符串。

##### BDD

BDD 风格提供两类断言：expect 和 should，两者都支持链式调用的语法让使用者可以用一种贴近自然语言的方式进行断言。使用方式如下：

```js
// expect:
const expect = require('chai').expect
const foo = 'bar'

expect(foo).to.be.a('string')
expect(foo).to.equal('bar')

// should:
const should = require('chai').should()
const foo = 'bar'

foo.should.be.a('string')
foo.should.equal('bar')
foo.should.have.lengthOf(3)
```

仔细观察这两类 API　的使用方式就可以看出差别：使用 expect 时只需将待测结果包裹进 `expect()` 函数便可进行链式调用，而使用 should 语法时则只需调用 `should()` 方法就可直接在待测结果上进行链式调用，其原理也很明显：调用`should()` 函数后在对象的原型上添加了 `should()` 方法的定义。

#### Jest

Jest 风格的 API 与 Chai 的 expect 语法类似，但是不提供链式调用，而是直接调用一个方法进行断言：

```js
expect(2 + 2).toBe(4)
expect('How time flies').toContain('time')
expect({ a: 1 }).not.toEqual({ b: 2 })
```

如上例所示，像 `toBe()`、`toEqual` 这类对待测内容的某个方面进行断言的方法，称为匹配器（Matcher）。常用的匹配器有 `toBe`、`toEqul`、`toContain` 等等。可以查阅 [Jest 的匹配器 API 文档](https://jestjs.io/docs/expect) 了解更多内容，匹配器的数量不多，也就不到 40 个，相信你可以轻松搞定，这里就不赘述了。 

### 使用 Jest

通过对单元测试最基本的步骤，即断言的介绍，我们了解了三种断言风格及相应的 API，在具备该编写单元测试的基本能力之后，我们来正式地学习如何使用自动化测试工具来进行单元测试，以 Jest 为例。

Jest 除了是一种断言风格之外，还是一个用于单元测试的测试框架，具备运行测试脚本的能力。它对普通的 JS 项目无需配置，开箱即用，同时支持快照测试、并行测试等优秀能力。

我们来尝试一下使用 Jest 进行单元测试。首先安装 Jest：

```
npm install jest -D
```

安装完毕后，我们新建一个 `__tests__` 目录，然后创建一个 `sum.spec.js` 文件。默认情况下当运行测试时 Jest 会自动搜索并运行 `__tests__` 目录下的所有 `.js`, `.jsx`, `.ts` 文件和根目录下所有带有 `.test` or `.spec` 后缀的文件，所以我们不必手动设置测试文件的位置。

在 `sum.spec.js` 文件下我们可以输入以下测试代码：

```js
function sum(a, b) {
  return a + b
}

describe('sum', () => {
  test('输入 1 和 2，输出 3', () => {
    expect(sum(1, 2)).toBe(3)
  })
})
```

写好测试代码之后，输入以下命令就可以启动 Jest 来运行测试：

```
npx jest
```

测试运行完毕后，Jest 就会在控制台输出以下内容表明测试通过：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e7af2027df54bcdaef1da88d8cead80~tplv-k3u1fbpfcp-watermark.image?)

OK，一个超级简单的单元测试就完成了！

我们来详细介绍一下测试代码中使用到的函数：

### 用于组织测试代码的 `describe()` 和 `test()`

第一个是 `test()` 方法，用于声明一个测试用例（test case，可直接称为一个测试，即 test）。我们在写单元测试时基本上就是以测试用例为单位来组织测试，它的第一个参数接受一个字符串，用于描述这个测试用例的内容，这里我们以“输入xx，输出xx”的格式来描述这个测试用例，这样可以清晰地表明这个测试用例的意图。

`test()` 方法的第二个参数是一个函数，包含了这个测试用例的主体内容，即断言。一个测试用例可以包含多个断言，但是所断言的内容应该符合这个测试用例的意图。

`test()` 方法还接收一个可选的 timeout 参数，用以指定测试的超时时间，默认是 5 秒。

`test()` 方法还有一个别名：`It()`，如果使用 `It()` 来描述测试用例可以采用更符合自然语言的语法，比如：

```ts
It('should return the correct result', () => {
  expect(sum(1, 2)).toBe(3)
  expect(sum(2, 4)).toBe(6)
  expect(sum(10, 100)).toBe(110)
})
```

`describe()` 方法可以组织一个或多个测试用例，将多个相关的测试组合成一个块，这种块叫作测试套件（test suite）。使用 `describe()` 来组织测试用例是一个推荐的写法，可以将测试内容与其他内容隔离，更有利于维护。

`describe()` 方法可以嵌套使用，比如可以像这样（来自官网示例）：

```js
describe('binaryStringToNumber', () => {
  describe('given an invalid binary string', () => {
    test('composed of non-numbers throws CustomError', () => {
      expect(() => binaryStringToNumber('abc')).toThrowError(CustomError)
    })

    test('with extra whitespace throws CustomError', () => {
      expect(() => binaryStringToNumber('  100')).toThrowError(CustomError)
    })
  })

  describe('given a valid binary string', () => {
    test('returns the correct number', () => {
      expect(binaryStringToNumber('100')).toBe(4)
    })
  })
})
```

嵌套的 `describe()` 块允许我们对测试用例进行更细粒度的分配和组织。当然，如果你不喜欢或不习惯用 `describe()` 也是可以的，你可以直接在顶层上下文中使用 `test()` 方法，Jest 会自动为其包裹一个测试套件。

到此为止，用于组织编写测试代码最常用的两个函数：`describe()`、`test() / It()` 就介绍到这里了。此外，使用这两个函数时还支持使用 `skip`、`only` 等扩展方法来跳过或在某些条件下跳过或过滤测试套件和测试用例的运行：

```ts
test.skip('跳过这个测试', () => {
  expect(sum(1, 2)).toBe(3)
})

test.only('只允许这个测试', () => {
  expect(sum(1, 2)).toBe(3)
})
```

更多 API 及详细内容可以查阅[文档](https://jestjs.io/zh-Hans/docs/api#%E6%96%B9%E6%B3%95)，这里不过多介绍了。

看到这里，你可能会问，`describe()`、`test()` 这些函数在使用之前不需要先引入吗？答案是不用，Jest 在执行测试代码之前会自动将这些全局 API 注入到全局上下文中，可以直接使用，不必手动引入。如果你更想要手动引入，可以新建一个 Jest 的配置文件，将 `injectGlobals` 字段的值置为 false 即可关闭全局注入。

### 钩子函数

当编写的测试代码较复杂，包含很多重复的如初始化的操作时，我们可以将这些重复的内容拆解（tear down）出来，放到钩子函数中执行。一个测试文件、测试套件和测试用例的执行也是有生命周期的，钩子函数将这些生命周期拆分为执行前和执行后两个阶段。Jest 提供了四个钩子函数允许使用者在这些生命周期中进行一些自定义行为。

`beforeAll()` 和 `afterAll()` 允许使用者注册一个回调，该回调会在当前上下文中的所有测试运行之前或之后被调用一次。

比如如果将 `beforeAll()` 放在顶层上下文中调用：

```js
beforeAll(() => {
  console.log(1)
})

describe('sum1', () => {
  test('测试1', () => {
    expect(sum(1, 2)).toBe(3)
  })

  test('测试2', () => {
    expect(sum(1, 2)).toBe(3)
  })
})

describe('sum2', () => {
  test('测试3', () => {
    expect(sum(1, 2)).toBe(3)
  })

  test('测试4', () => {
    expect(sum(1, 2)).toBe(3)
  })
})
```

则 `console.log(1)` 语句只会在两个套件内的测试执行前执行一次。`afterAll()` 也是同理。

而如果将 `beforeAll()` 放到测试套件内执行：

```ts
describe('sum1', () => {
  test('测试1', () => {
    expect(sum(1, 2)).toBe(3)
  })

  test('测试2', () => {
    expect(sum(1, 2)).toBe(3)
  })
})

describe('sum2', () => {
  beforeAll(() => {
    console.log(1)
  })

  test('测试3', () => {
    expect(sum(1, 2)).toBe(3)
  })

  test('测试4', () => {
    expect(sum(1, 2)).toBe(3)
  })
})
```

则 `console.log(1)` 语句只会在 sum2 套件内的测试执行前执行一次。

`beforeEach()` 和 `afterEach()` 所注册的回调会在当前上下文中的每个测试运行之前或之后被调用一次。注意与 `beforeAll()` 和 `afterAll()` 的区别，前者是运行每个测试前后执行一次，后者是在运行所有测试前后执行一次。

这四个钩子函数是我们编写测试代码时非常常用的函数了，一些模拟、初始化和清除状态的逻辑我们都会放到钩子函数中进行。你可能会问，如果测试文件之间也包含一些重复的逻辑时要怎么处理呢？

Jest 允许我们编写一个在每个测试文件的测试代码运行之前运行的 setup file 进行跨文件的配置。我们需要先新建一个 setup file，比如在根目录下创建一个 `jest-setup.js` 文件，输入以下内容： 

```js
beforeAll(() => {
  console.log(1)
})
```

接着在根目录下新建一个 Jest 的配置文件 `jest.config.js`，输入以下内容：

```js
/** @type {import('jest').Config} */
module.exports = {
  setupFilesAfterEnv: ['<rootDir>/jest-setup.js'],
}
```

`setupFilesAfterEnv` 字段用于指定一个 setup file 数组的路径，这些文件会在 Jest 的执行环境（包括 `describe()`、钩子函数等全局 API 的初始化）安装之后、测试代码运行之前执行。

--------------------------------------------------------------------

有关使用 Jest 进行单元测试的两个基础知识就介绍到这里了，在开始接下来的重点内容之前，我们来聊聊 Jest 这个测试框架本身。

通过以上几个示例，我们算是小小地入门了 Jest 的使用，充分体会到了 Jest 的”无需配置“的妙处，即安装之后即可开始编写测试代码，并且无需手动引入相关 API，测试代码写完之后启动一行命令即可开始运行测试，不得不说真的很方便。

但是，我们刚才所展示的仅仅是一个非常简单的测试一个 JS 函数的场景，要是应用场景更复杂一点，比如对 Web 应用进行单元测试，Jest 可能就不会像现在这样方便了。为什么这么说呢？

思考一下，Jest 是如何运行测试文件的？自然是用 Node 运行的，详细点说，就是在注入 `describ()`、`beforeAll` 等全局 API 后，就使用 Node 来运行测试代码，处理所导入的依赖的路径解析和加载。这时如果导入的是一个 vue 文件，测试就会立即失败，因为 Jest 不认识这个类型的文件。甚至如果直接使用 TypeScript 来编写测试代码，也会导致测试失败。这就意味着，如果是 `.ts`、`.vue`、`.jsx` 类型的文件 Jest 就无法先天支持了，因为它只认识 JS 语法。要想支持其他语法的运行，就需要使用一些 transformer 进行转换， 将 `.ts`、`jsx` 等语法转换为标准 JS 语法，才能继续执行测试代码。

比如想让 Jest 能够加载、运行 `.ts`、`.vue` 格式的文件，就需要这样配置：

```js
// jest.config.js
module.exports = {
  transform: {
    '^.+\\.(j|t)sx?$': 'babel-jest',
    '^.+\\.vue$': '@vue/vue3-jest'
  }
}
```

我们使用 babel 来处理 Typescript 内容，将其中的类型注解删除掉，需要提前安装 `@babel/core`、`@babel/preset-env`、`@babel/preset-typescript` 这几个包并新建一个 `babel.config.js` 来配置 babel 的行为：

```js
// babel.config.js
module.exports = {
  presets: [
    '@babel/preset-typescript',
    ['@babel/preset-env', { targets: { node: 'current' } }],
  ],
}
```

要想 Jest 支持 Vue 文件的转换需要使用 [`vue-jest`](https://github.com/vuejs/vue-jest)，这里我用的是支持 Vue3 的版本。

此外，Jest 还未支持 ESM 规范，仍处于实验阶段，也需要使用 babel 降级。所以，如果想要使用 Jest 来测试一个 Web 应用，需要进行更多配置。另外，由于 Jest 的测试和构建工具的开发、构建是在两个不同的转换管道中进行的，需要使用两个不同的配置文件，这也无形中加大了项目前期搭建的负担。

因此，相比于使用 Jest，我更推荐你使用 [Vitest](https://vitest.dev/) ！

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f216e6eee8c4f38a7776decf8c1a0ca~tplv-k3u1fbpfcp-watermark.image?)

Vitest 是一个由 Vite 提供支持的极速单元测试框架，它底层依赖了 Vite，借助于 Vite 强大的能力，Vitest 支持了以下这些优秀特性：

- 与 Vite 共享同一套配置！如果你的项目也是选择 Vite 作为构建工具，那么你可以与 Vite 共享同一套配置。这是因为 Vitest 同样使用 Vite 对你的测试代码及其引入的所有模块使用 Vite 的解析器、加载器、转换器进行处理，这意味着当 Vitest 运行时你的 Vite 配置文件中配置的除用于生产环境以外的所有插件都会被调用并发挥作用，你的测试环境会和开发环境一样使用同一个运行管道，而不需要像 Jest 一样进行额外的配置！

- 真正的开箱即用，而且速度超快！借助 Esbuild 对 TypeScript、JSX 和 ESM 语法的天然支持，Vite 原生就具备处理这几类语法的能力，促使 Vitest 做到真正的开箱即用，并且速度超快！

- 测试的 HMR！Vite 的开发服务器在对模块进行加载、转换的过程中，会逐步构建出一个缓存转换结果、记录模块依赖关系等信息的模块图（Module Graph）。借助模块图，Vite 可以清晰地处理模块热更新的边界，确定进行更新的范围。而 Vitest 借助 Vite 的 HMR 能力，同样可以做到在修改源代码后重新运行依赖该源代码的测试文件，做到测试的 HMR。Vitest 在默认情况下会启动监听模式（watch mode），即自动进行 HMR，这对喜欢使用 TDD 的模式进行开发的同学来说无疑是个福音！

除了以上三点通过 Vite 得到支持的优秀能力之外，Vitest 还具备以下几个功能：

- 多线程运行测试文件。通过使用 Worker 线程尽可能多地并发运行测试文件，使多个测试同时运行。同时，Vitest 还隔离了每个测试文件的运行环境，使某一个文件的状态不会对其他文件造成影响。

- 支持多数测试框架的常用功能。例如快照测试、模拟（Mock）、测试覆盖率、并发运行测试和模拟 DOM 等功能。

- 兼容 Chai 和 Jest 的 API。内置 Chai 的断言 API 和 Jest 的大多数 API。

Vue 官方的脚手架 `create-vue` 已将 Vitest 作为默认的单元测试框架。如果你还在犹豫不决，觉得 Vitest 还是一个较新的测试框架，怀疑是否可以在实际项目中使用的话，可以看[这篇文章](https://mp.weixin.qq.com/s/Ara_KWegpSRssIPz3Oddtw)。

### 使用 Vitest

在介绍完 Vitest 的功能之后，我们来尝试一下它的基本使用。同样先安装 Vitest：

```
npm install -D vitest
```

安装完毕后，我们来编写测试代码，同样对 sum 函数进行测试：

```js
import { describe, expect, test } from 'vitest'

function sum(a, b) {
  return a + b
}

describe('sum', () => {
  test('输入 1 和 2，输出 3', () => {
    expect(sum(1, 2)).toBe(3)
  })
})
```

由于 Vitest 在默认情况下不自动注入全局 API，因此我们需要手动引入 `describe()`、`test()` 等方法。当然，可以通过配置 globals 字段来开启自动注入，这里我们先不开启。

测试代码编写好后，运行以下命令启动 Vitest：

```
npx vitest
```

与 Jest 不同，Vitest 会将所有带有 spec 和 test 后缀的 js、ts 等类型文件视为测试文件。具体可以看 [include](https://cn.vitest.dev/config/#include) 字段

当控制台打印以下信息时说明测试通过：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/126333d58e6042d49acc2e56fcff7f6c~tplv-k3u1fbpfcp-watermark.image?)

以上就是 Vitest 的基本使用了，关于 Vitest 的更多内容可以看下文的实战小节。现在我们来继续学习单元测试的常用功能，这部分是重点内容。

### 测试异步代码

在真实的场景中测试异步代码是一件非常常见的事，比如测试后端 API，异步函数等等。由于异步代码的特殊性，在测试它们时需要做更多的工作。

比如我们要测试以下这个异步函数：

```js
async function hello() {
  return 'Hello World!'
}
```

要断言它返回了 "Hello World!" 字符串，如果按照测试同步函数的方式进行测试：

```js
test('hello', () => {
  expect(hello()).toBe('Hello World!')
})
```

运行这个测试会直接报错：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c016a331d6c8451086529268ca8c79aa~tplv-k3u1fbpfcp-watermark.image?)

原因很简单，我们断言的并不是 "Hello World!" 这个字符串，而是这个函数返回的 Promise 对象。

知道了原因之后，我们可以很自然地进行改进：

```js
test('hello', async () => {
  const res = await hello()
  expect(res).toBe('Hello World!')
})
```

我们改为将一个异步函数传入 `test()` 方法，该异步函数使用 await 关键字等待 hello 函数 resolve 并返回结果，接着就可以对其返回结果进行断言。

除了使用 await 等待异步函数调用完成之外，我们还可以使用 `resolves()` 和 `rejects()` 方法。使用方式如下：

```js
test('hello', async () => {
  await expect(hello()).resolves.toBe('Hello World!')
})
```

可以看到，使用 `resolves()` 可以从 hellow 函数返回的 Promise 对象中提取所 resolve 的值，然后直接对该值进行断言。

`rejects()` 方法的使用方式同理：

```js
async function hello() {
  throw new Error('Hello World!')
}

test('hello', async () => {
  await expect(hello()).rejects.toThrow('Hello World!')
})
```

以上便是测试异步代码的两种方法，比较简单，相信你可以很快掌握。

### 处理定时器

虽然定时器回调也算是异步代码的一种，但是它毕竟不返回 Promise，我们还需对其进行其他处理。

比如测试以下代码：

```js
let a = 1

function timer() {
  setTimeout(() => {
    a = 2
  }, 3000)
}
```

要断言调用 timer 函数会在 3 秒后将 a 的值置为 2，我们要怎么做呢？如果直接使用同步的方式，即：

```js
test('timer', () => {
  expect(a).toBe(1)
  timer()
  expect(a).toBe(2)
})
```

运行结果自然是报错，因为第二个断言会在回调调用之前执行：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec358300fc1444bfb7ca90b35915cbbd~tplv-k3u1fbpfcp-watermark.image?)

（注：在这个示例中我们在对函数进行调用之前断言了 a 的初始状态，即第一个断言，这是为了保证在进行正式的断言之前待测对象的状态不会发生意外改变，确保正式的断言的结果是由我们的操作（这里是调用 timer 函数和）产生的，而非外界的干扰。你可以把这个步骤理解为一种**控制变量**的做法。）

要想做到断言定时器的操作结果，我们可以使用 vitest 模块导出的 vi 工具对象中的 `useFakeTimers()` 方法，该方法的作用顾名思义——使用假的定时器。当调用 `useFakeTimers()` 方法使用 fake timers 之后，所有对定时器的调用，包括 `setTimeout`、`setInterval`、`nextTick`、`setImmediate` ，所传入的回调都会被"滞留"在定时器队列中，得不到执行，即使达到了指定的 timeout 时间。需要手动调用 `vi.runAllTimers()` 或 `vi.advanceTimersByTime()` 等方法才可以执行这些回调。比如：

```js
test('timer', () => {
  vi.useFakeTimers()
  expect(a).toBe(1)

  timer()
  vi.runAllTimers()

  expect(a).toBe(2)

  vi.useRealTimers()
})
```

调用了 `vi.useFakeTimers()` 使用 fake timers 之后，我们可以调用 `vi.runAllTimers()` 来运行所有处于队列中的定时器回调。另外，为了避免对其他测试造成影响，在测试的最后我们还需要调用 `vi.useRealTimers()` 恢复使用真实定时器。在实际场景中，我们可以选择在钩子函数中处理这些初始化、清除副作用的操作。

我们也可以使用 `vi.advanceTimersByTime()`，它可以只执行所传入的毫秒数以内对应的超时时间的回调：

```js
test('timer', () => {
  vi.useFakeTimers()
  expect(a).toBe(1)

  timer()

  vi.advanceTimersByTime(2000)
  expect(a).toBe(1)

  vi.advanceTimersByTime(3000)
  expect(a).toBe(2)

  vi.advanceTimersByTime(4000)
  expect(a).toBe(2)

  vi.useRealTimers()
})
```

除了模拟定时器外，`vi.useFakeTimers()` 还可以模拟日期（Date），具体用法可以看 [`vi.setSystemTime()`](https://cn.vitest.dev/api/#vi-setsystemTime) 方法。

### 模拟（Mock）

在真实的测试场景中，我们需要应付许多调用后端 API 的模块，由于调用这些接口需要发起网络请求，导致测试时间变相增长，降低测试效率，并且，我们做的也不是端到端测试，而是将待测对象与外界隔离的单元测试，不必发起真实的网络请求。另外，在单元测试下为了屏蔽其他模块，比如第三方模块，我们还需要避免对它们的调用，甚至伪造一个假的模块。更重要的是，很多时候我们还需要断言待测对象对其他模块或方法的调用，即进行监听。在这种情况下，模拟（Mock）就派上用场了。

模拟的方式，大致可以分为两种：stub（存根） 和 spy（监听）。

stub 会改变被模拟对象的实现，即伪造另一个版本来代替被模拟的对象。与之相反，spy 无需改变被模拟对象的实现，但是会监听对其的使用，如监听函数的调用次数、传入的参数等等。

我这里仅仅按照"实现是否被更改"来对模拟的方式进行分类，也有将模拟分为 mock、stub 和 fake 等等的分类，其实也不必纠结这几种分类和模拟方式之间的差异，大多数场合将它们统称为模拟（Mock）即可。

接下来我按照模拟函数、模拟模块的顺序来介绍模拟的具体使用。

#### 模拟函数

比如现在我们要监听对 obj 对象的 sum 方法的调用，获取对该方法的调用次数、参数、返回值等信息，要怎么做呢：

```js
const obj = {
  sum: (a: number, b: number) => {
    return a + b
  }
}
```

可以使用 `vi.spyOn()` 方法：

```js
test('spy', () => {
  vi.spyOn(obj, 'sum')

  obj.sum(1, 2)

  expect(obj.sum).toBeCalledTimes(1)
  expect(obj.sum).toBeCalledWith(1, 2)
  expect(obj.sum).toHaveReturnedWith(3)

  vi.mocked(obj.sum).mockClear()
})
```

`vi.spyOn()` 可以监听一个对象上的方法。在调用被监听的函数之后，我们就可以通过 `toBeCalledTimes()`、`toBeCalledWith()` 等等这类匹配器来断言调用信息。

`vi.spyOn()` 返回一个 SpyInstance 类型的对象，我们也可以直接在这个对象上进行断言，比如：

```js
test('spy', () => {
  const spy = vi.spyOn(obj, 'sum')

  obj.sum(1, 2)

  expect(spy).toHaveBeenCalledOnce()
  expect(spy).toHaveBeenNthCalledWith(1, 1, 2)
  expect(spy).toHaveReturnedWith(3)

  spy.mockClear()
})
```

你可能已经注意到了我们在测试最后调用了一个 [`mockClear()`](https://cn.vitest.dev/api/#mockclear) 方法，该方法用于清除被模拟的对象的所有调用信息。使用它的目的与前文使用的 `vi.useRealTimers()` 一样，为了不对其他测试造成影响。类似的方法还有 [`mockReset()`](https://cn.vitest.dev/api/#mockreset) 和 [`mockRestore()`](https://cn.vitest.dev/api/#mockrestore)，前者用于清除调用信息和将被模拟对象的实现置为一个空函数，后者用于清除调用信息和还原被模拟对象的原始实现。这个示例中由于我们仅仅是对函数进行监听，没有修改内部实现， 因此调用 `mockClear()` 就足够了。

对每个模拟对象调用 `mockClear()`、`mockReset()` 很快会变成重复的行为，我们可以使用 [`vi.clearAllMocks()`](https://cn.vitest.dev/api/#vi-clearallmocks)、[`vi.resetAllMocks()`](https://cn.vitest.dev/api/#vi-resetallmocks) 和 [`vi.restoreAllMocks()`](https://cn.vitest.dev/api/#vi-restoreallmocks) 一次性对所有的模拟对象进行这些操作，通常把对这三个方法的调用放到钩子函数里。

`mockClear()` 是 SpyInstance 和 MockInstance 类型上的方法，所以我们可以直接在 `vi.spyOn()` 返回的对象上调用它，如果我们想直接在原函数上调用该方法，像下面这样：

```js
obj.sum.mockClear()
```

如果你使用的是 JS，这可以行得通，但是如果你使用的是 TS 的话，就会直接报错了。在这种情况下，可以使用 `vi.mocked()` 来为被模拟的对象提供类型支持：

```js
vi.mocked(obj.sum).mockClear()
```

如果我们要模拟另一个模块导出的函数要怎么做呢？比如：

```js
// math.ts
export function sum(a: number, b: number) {
  return a + b
}
```

这时候我们可以以命名空间的形式来导入 math 模块：

```js
import * as math from './math'

test('spy', () => {
  vi.spyOn(math, 'sum')

  math.sum(1, 2)

  expect(math.sum).toBeCalledTimes(1)
  expect(math.sum).toBeCalledWith(1, 2)
  expect(math.sum).toHaveReturnedWith(3)

  vi.mocked(math.sum).mockClear()
})
```

可以看到十分简单粗暴。看到这里你可能会有疑问：只能监听对象上的方法吗，不能直接监听函数吗？

据我所知，好像不能。如果真想直接监听函数的话，可以这样做：

```js
import { sum } from './math'

test('spy', () => {
  const math = { sum }
  vi.spyOn(math, 'sum')

  math.sum(1, 2)

  expect(math.sum).toBeCalledTimes(1)
  expect(math.sum).toBeCalledWith(1, 2)
  expect(math.sum).toHaveReturnedWith(3)

  vi.mocked(math.sum).mockClear()
})
```

直接将它放到一个对象上就行了。

学完了怎么监听函数，我们来看看怎么模拟一个函数。比如我们想将 sum 函数模拟成以下这样：

```js
function sum(a: number, b: number) {
  return a + b + 100
}
```

可以直接在 SpyInstance 上调用 `mockImplementation()` 方法：

```js
test('mock', () => {
  vi.spyOn(obj, 'sum').mockImplementation((a, b) => a + b + 100)

  obj.sum(1, 2)

  expect(obj.sum).toHaveReturnedWith(103)

  vi.mocked(obj.sum).mockRestore()
})
```

`mockImplementation()` 方法可以直接在 SpyInstance 和 MockInstance（继承了 SpyInstance）上使用，用于模拟被模拟对象的实现。由于我们更改了 sum 的内部实现，因此测试完毕后需要调用 `mockRestore()` 将其还原。

#### 模拟模块

介绍完了如何监听和模拟函数，我们来看看如何模拟模块。

模拟模块需要使用 `vi.mock()` 方法，比如要模拟刚刚的 math 模块，我们可以这样做：

```js
import { sum } from './math'

vi.mock('./math')

test('mock', () => {
  sum(1, 2)

  expect(sum).toHaveBeenCalledOnce()
  expect(sum).toHaveBeenCalledWith(1, 2)

  vi.mocked(sum).mockRestore()
})
```

我们将要模拟的模块的路径传入 `vi.mock()` 方法后，该方法会自动模拟被模拟模块的所有导出内容，所以当我们调用了该模块的某一个导出函数后，我们就可以直接对其进行断言。

我们也可以传入一个工厂函数来定义该模块要导出什么内容，比如：

```
vi.mock('./math', () => ({
  sum: (a: number, b: number) => a + b + 100
}))
```

我们模拟了 math 模块的导出内容，其返回了一个新的 sum 方法。但是运行测试发现测试失败了：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9e97a3e86764e9291221c36f306e196~tplv-k3u1fbpfcp-watermark.image?)

这是因为我们仅仅模拟了 math 模块，而没有模拟它导出的 sum 函数。我们来学习模拟函数的另一种方法：`vi.fn()`。

调用 `vi.fn()` 会返回一个空的 Mock 类型的模拟函数，Mock 也继承了 SpyInstance，我们可以直接对该函数调用 `toHaveBeenCalledOnce()` 等匹配器。直接调用模块函数会返回 undefined。我们可以传入一个函数到 `vi.fn()` 中来模拟其返回的模拟函数的实现。以上代码可以修改为：

```js
vi.mock('./math', () => ({
  sum: vi.fn((a: number, b: number) => a + b + 100)
}))
```

运行测试后显示测试通过，说明我们模拟成功了。

如果我们只想模拟模块导出的某个特定函数，其他导出内容维持原样，可以这样做：

```js
import { sum } from './math'
import * as Math from './math'

vi.mock('./math', async () => ({
  ...await vi.importActual < typeof Math > ('./math'),
  sum: vi.fn((a: number, b: number) => a + b + 100)
}))
```

`vi.importActual()` 可以原封不动地导入模块的所有导出内容。注意当使用 TS 时，记得传入类型。

除了传入一个工厂函数外，我们还可以将要模拟的导出内容放到一个 `__mocks__` 目录里，这样当调用 `vi.mock()` 时如果 `__mocks__` 目录下存在同名文件，所有导入都会返回其导出。比如在 `__tests__` 目录下新建一个 `__mocks__` 目录，然后创建一个 `math.ts` 文件，内容如下：

```js
import { vi } from 'vitest'

export const sum = vi.fn((a: number, b: number) => a + b + 100)
```

然后将测试的模拟代码修改为：

```js
vi.mock('./math')
```

重新运行测试，测试会通过。

注意，对 `vi.mock()` 的调用会被自动提升到顶层上下文，即使在测试套件或测试内调用它也是如此。所以如果你只是想在某个套件或测试内模拟模块，可以使用 `vi.importMock()` 方法：

```js
import * as Math from './math'

test('mock', async () => {
  const { sum } = await vi.importMock < typeof Math > ('./math')
  sum(1, 2)

  expect(sum).toHaveBeenCalledOnce()
  expect(sum).toHaveBeenCalledWith(1, 2)

  sum.mockRestore()
})
```

该方法使用方式与 `vi.mock()` 相同，只是将模拟的行为定义在测试套件或测试内而已。另外，调用该方法后会返回原函数类型和 Mock 类型的交叉类型，所以可以不用使用 `vi.mocked()` 就可以获取类型信息。

#### 模拟全局变量

模拟全局变量的方式比较简单，使用 `vi.stubGlobal()` 就可以。这里直接贴出文档示例：

```js
import { vi } from 'vitest'

const IntersectionObserverMock = vi.fn(() => ({
  disconnect: vi.fn(),
  observe: vi.fn(),
  takeRecords: vi.fn(),
  unobserve: vi.fn(),
}))

vi.stubGlobal('IntersectionObserver', IntersectionObserverMock)
```

### 测试覆盖率

很多人在写完单元测试后会想知道自己写的测试是否已经够多了，这时候他们会看测试的覆盖率是否够高。

测试覆盖率，顾名思义，就是检查测试所覆盖的源代码量占源代码总数的比例。Vitest 支持通过 [c8](https://github.com/bcoe/c8) 和 [istanbul](https://istanbul.js.org/) 获得测试的覆盖率，我们来尝试一下。

Vitest 默认情况下使用 c8，我们需要先安装对应的包：

```
npm i -D @vitest/coverage-c8
```

然后更新测试代码，这次我们还是来测试 sum 函数：

```js
import { expect, test } from 'vitest'
import { sum } from '../src/math'

test('sum', () => {
  expect(sum(1, 2)).toBe(3)
})
```

然后在命令行中输入以下命令：

```
npx vitest run --coverage
```

然后控制台就输出了测试覆盖率的报告：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c3da55f8fbc24318a1a606ee716fc064~tplv-k3u1fbpfcp-watermark.image?)

以上这四个参数的含义分别是：语句覆盖率（statements）、分支覆盖率、函数覆盖率和行覆盖率。同时根目录下还生成了一个 coverage 目录，记录了更详细的统计信息。

使用 istanbul 的方式也是差不多，安装对应的包就行了，不再赘述了。

istanbul 的原理是把源代码进行转译，插入用于记录某个函数、语句被调用次数等记录的代码，然后将记录到的信息存储到某个变量中，测试完毕后就可以通过这个变量获取统计到的信息，生成覆盖率报告。而 c8 是直接使用 V8 引擎内置的覆盖率统计，测试完成后直接生成报告。

在实际项目中为了保证程序员们写单测的数量或质量，会限定测试覆盖率的阈值，然后在代码提交前或者在集成管道中检查测试覆盖率是否达到这个阈值。我们来尝试一下。

如果你使用的是 Vite，那么你可以直接在 `vite.config.ts` 里进行配置：

```js
/// <reference types="vitest" />
import { defineConfig } from 'vite'

export default defineConfig({
  // 其它配置项...

  test: {
    coverage: {
      lines: 80,
      functions: 80,
      branches: 80,
      statements: 80
    }
  },
})
```

我们将 80% 作为阈值。为了方便演示，我们来修改 sum 函数的实现：

```ts
export function sum(a: number, b: number) {
  if (a > 100)
    return 100

  return a + b
}
```

然后同样运行刚才那个命令运行测试，覆盖率报告如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42697a45e4fb4ce4af5ab8fe8ce0dff9~tplv-k3u1fbpfcp-watermark.image?)

由于未达到阈值，控制台报错，然后我们就可以观察哪部分代码的分支或行等没有被覆盖到，为其补充测试用例。这种根据程序的内部实现，如分支、函数等创建测试用例进行测试的方式，其实就属于白盒测试。如果要判断黑盒测试的覆盖率，可以通过判断所使用的测试用例所对应的等价类占总的等价类（包括有效等价类和无效等价类）及边界值的比例来得出。感兴趣的朋友可以自行查阅相关资料。

关于如何输出测试覆盖率并做覆盖率检查的使用就介绍到这里了。测试覆盖率作为检查单元测试是否充分的手段，在一定程度上确实是一个有效的工具。但是，高测试覆盖率不等于高的测试质量，在很多情况下高测试覆盖率其实是一个很容易达到的数字。比如我们可以把测试用例改成这样：

```js
test('sum', () => {
  expect(sum(1, 2)).not.toBe(100)
})
```

以上这个测试用例是：输入 1 和 2，不会输出 100。测试覆盖率达到了 100%，超额完成了任务要求，但是这个测试的质量就真的很高么？答案显然是否定的，因为这个测试用例并没有任何意义，我们应该断言它返回了正确的结果（即 3），而不是断言它返回了其它无关的数字，除非进行穷举，断言它不等于除 3 以外的所有数字，但这显然是不可能的。

很多人写测试时以高覆盖率为目标，会以覆盖率达到 100% 为傲，但这并没有什么用，你应该做的、思考的，是如何设计出高质量的测试用例，而不是盯着一个数字疯狂堆用例。很多情况下，即使达到了 100% 也不能说明程序就没有问题，正如文章开头说的那样，软件测试是检验其是否满足规定的需求，或者找出程序中潜在的错误。

Martin Fowler 在[这篇文章](https://www.martinfowler.com/bliki/TestCoverage.html)中提到：高测试覆盖率并不意味着什么，它反而在帮助检查源代码中还没有被测试的地方这个方面有效果。他认为，如果你做到了以下这两点，就说明你写的测试已经充足了：

- 你很少在生产中碰到 bug；

- 当修改代码时你很少会犹豫、害怕它会导致生产事故。

关于测试覆盖率我要说的就是这些了，希望能提高你对测试覆盖率的认知。

### 配置类浏览器环境

Vitest、Jest 等单元测试框架由于是运行在 Node 环境中的，如果我们要测试一个 Web 应用，进行组件测试，就需要有类浏览器环境，以支持 DOM API、本地存储、Cookie 等浏览器特性。

Vitest 提供了 environment 选项来配置测试环境，除 Node 之外还支持了 [jsdom](https://github.com/jsdom/jsdom)、[Happy DOM](https://github.com/capricorn86/happy-dom) 和 [Edge Runtime](https://edge-runtime.vercel.app/packages/vm)。

jsdom 是一个用于 Node 环境的对许多 Web 标准的 JS 实现，它的使用示例如下：

```js
const jsdom = require('jsdom')
const { JSDOM } = jsdom

const dom = new JSDOM('<!DOCTYPE html><p>Hello world</p>')
console.log(dom.window.document.querySelector('p').textContent)
```

可以看到，只要将 HTML 字符串传入 JSDOM 构造函数中，就可以在返回的实例上使用许多包括 `querySelector()` 等众多 Web API。

尽管 jsdom 实现了许多 Web API，但是它毕竟运行在一个模拟的浏览器环境（即无头浏览器）中，许多特性仍然无法实现，一个是布局（layout），即无法计算某个元素在页面中的布局，如在视口中的位置（`getBoundingClientRects()`）和 offsetTop 等属性；一个是 navigation。所以在某些场景下使用 jsdom、Happy DOM 进行 Web 环境下的测试可能无法很好地满足你的需求，在这种情况下你需要让待测对象在一个真实的浏览器上运行，比如使用 [Cypress](https://www.cypress.io/) 来进行测试。

Happy DOM 与 jsdom 一样实现了 Web 浏览器的诸多特性，相比于后者，它拥有更高的性能，但实现的特性要少一点。

我们来使用 jsdom 来配置类浏览器环境，首先需要安装 jsdom：

```
npm -D install jsdom
```

接着修改配置：

```ts
// vite.config.ts
test: {
    environment: "jsdom",
},
```

就可以在全局使用 Web API 了：

```ts
test('dom', () => {
  const div = document.createElement('div')
  div.className = 'dom'
  document.body.appendChild(div)
  expect(document.querySelector('.dom')).toBe(div)
})
```

从 `0.23.0` 开始，Vitest 支持使用自定义的环境，需要创建一个命名格式为 `vitest-environment-${name}` 的导出环境对象的包，并且还导出了 `populateGlobal` 方法方便填充 global 对象。你可以点击[这里](https://cn.vitest.dev/guide/environment.html)查看指引。

#### 使用 `jest-dom`

当在 Web 环境下对 DOM 进行测试时，我们会发现对 DOM 结点进行断言会比较麻烦，比如断言其是否有某个属性、是否可见，一个按钮是否被禁用，输入框是否聚焦等等，我们通常需要调用多个 DOM API 逐步提取出想要的属性或值等才能达到我们的目的。

[`jest-dom`](https://github.com/testing-library/jest-dom) 提供许多 Jest 匹配器来帮助我们简化这些步骤，由于 Vitest 兼容 Jest 的断言风格，所以 jest-dom 也可以在 Vitest 上使用。我们来尝试一下。

首先进行安装：

```
npm install -D @testing-library/jest-dom
```

安装完毕后我们需要应用这些匹配器，可以选择在 setup file 中进行这个操作：

```ts
// __tests__/vitest-setup.ts
import '@testing-library/jest-dom'
```

注意，引入这个包时它会在内部使用 `expect.extend()` 方法来应用这些自定义匹配器，这意味着 `expect` 必须是一个全局 API。Vitest 默认情况下关闭全局 API 的注入，我们可以手动开启，并配置 setup file 的路径：

```ts
/// <reference types="vitest" />
import path from 'path'
import { defineConfig } from 'vite'

export default defineConfig({
  // 其它配置项...

  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: path.resolve(__dirname, '__tests__/vitest-setup'),
  },
})
```

如果你不喜欢开启全局注入，可以将 setup file 的内容改成这样：

```ts
// __tests__/vitest-setup.ts
import matchers from '@testing-library/jest-dom/matchers'
import { expect } from 'vitest'

expect.extend(matchers)
```

现在就能使用 jest-dom 提供的匹配器了：

```ts
test('dom', () => {
  const div = document.createElement('div')
  div.className = 'dom'
  document.body.appendChild(div)
  expect(div).toBeInTheDocument()
})
```

jest-dom 提供的匹配器数量不多，只有二十几个，建议你到仓库把它们都看一遍熟悉一下。

### 快照测试

快照是一个序列化的字符串，你可以用它来确保待测对象的输出不会发生改变。使用方式如下：

```ts
test('sum', () => {
  const res = sum(1, 2)
  expect(res).toMatchSnapshot()
})
```

`toMatchSnapshot()` 匹配器用于对所传入的期望与之前保存的快照进行比较。当第一次使用它时会在测试文件的目录下新建一个 `__snapshots__` 目录存放每个测试文件中的快照，内容大致如下：

```ts
// Vitest Snapshot v1

exports['sum 1'] = '3'
```

当第二次使用 `toMatchSnapshot()` 匹配器时就会进行比较，如果不匹配就会报错，比如：

```ts
test('sum', () => {
  const res = sum(100, 200)
  expect(res).toMatchSnapshot()
})
```

修改测试代码，重新运行测试后就会报错：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7629f642d54544dabe09f09da22aaf41~tplv-k3u1fbpfcp-watermark.image?)

如果快照不匹配是预期的行为，可以在控制台键入“u”更新失败的快照。

如果你不希望将快照保存在另一个目录中，可以选择内联快照，使用 `toMatchInlineSnapshot` 匹配器：

```ts
test('sum', () => {
  const res = sum(1, 2)
  expect(res).toMatchInlineSnapshot()
})
```

使用 `toMatchInlineSnapshot()` 后运行测试，生成的快照会作为参数直接被写入匹配器的括号内：

```ts
test('sum', () => {
  const res = sum(1, 2)
  expect(res).toMatchInlineSnapshot('3')
})
```

Jest 文档中推荐了快照测试的另一个用途：测试 React 组件，提供的示例如下：

```ts
import renderer from 'react-test-renderer';
import Link from '../Link';

it('renders correctly', () => {
  const tree = renderer
    .create(<Link page="http://www.facebook.com">Facebook</Link>)
    .toJSON();
  expect(tree).toMatchSnapshot();
});
```

以上代码渲染了 Link 组件，然后对序列化后的结果进行快照测试。所保存的快照是这个样子：

```ts
exports['renders correctly 1'] = `
<a
  className="normal"
  href="http://www.facebook.com"
  onMouseEnter={[Function]}
  onMouseLeave={[Function]}
>
  Facebook
</a>
`
```

通过对组件的渲染结果进行快照测试，可以很方便地找出所修改的内容与之前的版本不匹配的地方，然后进行修复或更新。

但是，你不应该过度依赖快照测试，也不应该过分对组件进行快照测试，因为快照测试并不能很好地表达测试用例的意图而仅仅比较序列化后的结果，当快照出现不匹配时我们无法立即断定这是因为代码某处地方出现 bug 还是代码更新后的正常现象，为了找出不匹配的原因我们可能会在这个不匹配的地方上浪费大量的时间，甚至放弃思考武断地选择更新快照。

快照测试是一把双刃剑，它在某些场景下可能会很有用，但是也有可能让测试走向另一个极端。我个人还是建议开发者编写有意图的测试，从程序的输入输出等方面入手，专注设计高质量的测试用例。

Testing Library 的作者 Kent C. Dodds 在他的[这篇博客](https://kentcdodds.com/blog/effective-snapshot-testing)中介绍了几个他觉得非常适合使用快照测试的地方，感兴趣的同学可以看一看。

--------------------------------------------------------------------

到此为止，关于如何使用单元测试框架进行自动化测试的入门内容就介绍到这里了，相信你已经收获良多了。