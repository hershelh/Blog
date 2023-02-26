---
title: 前端自动化测试实战
description: 实战前端项目的单元测试、组件测试
date: 2023-01-02T01:08:00.000+00:00
lang: zh
duration: 1h
---

在这篇文章中，我们来对一个地址列表小应用进行单元测试和组件测试。技术栈主要是 Vue3、Pinia 和 TypeScript。
源代码仓库在[这里](https://github.com/hershelh/addressList)，我还提供了使用 Vuex 的分支，你可以拉下来边学习边对照。

### 准备工作

#### 使用 Vue Test Utils

[Vue Test Utils](https://test-utils.vuejs.org/) 是官方提供的组件挂载库，它提供了许多实用的 API 来支持对 Vue 组件的测试，我们来尝试一下。

首先安装包：

```
npm install -D @vue/test-utils
```

然后新建一个测试文件，输入以下代码：

```js
import { expect, test } from 'vitest'
import { mount } from '@vue/test-utils'
import { defineComponent } from 'vue'

const Component = defineComponent({
  template: '<p>Hello World!</p>',
})

test('Component', () => {
  const wrapper = mount(Component)

  expect(wrapper.find('p').text()).toBe('Hello World!')
})
```

运行测试后终端会显示测试通过。

我们使用了 mount 方法来挂载组件，mount 方法内部会先创建一个包含该组件的父组件，然后调用 `createApp()` 方法创建 Vue 应用，将父组件作为根组件传进去，最后挂载到一个 div DOM 节点上。

mount 方法还支持传入一个配置对象来支持对组件的渲染或初始化进行更多配置，我挑几个较常用的配置项列在下面：

- data：用于覆盖组件默认的 data 数据，比如：

    ```js
    const Component = defineComponent({
      data() {
        return {
          msg: 'Hello World!',
        }
      },
      template: '<p>{{ msg }}</p>',
    })

    test('Component', () => {
      const wrapper = mount(Component, {
        data() {
          return {
            msg: '111',
          }
        },
      })

      expect(wrapper.find('p').text()).toBe('111')
    })
    ```
- props：设置渲染组件的 props：

    ```js
    const Component = defineComponent({
      props: {
        msg: {
          type: String,
          required: true,
        },
      },
      template: '<p>{{ msg }}</p>',
    })

    test('Component', () => {
      const wrapper = mount(Component, {
        props: {
          msg: 'Hello World!',
        },
      })

      expect(wrapper.find('p').text()).toBe('Hello World!')
    })
    ```
- globals：
    - plugins：设置要应用到所创建的 app 的插件；
    - stubs：设置对待测组件的子组件的存根，当你不想渲染某些子组件或者模拟子组件时这个选项会很有用。

- shallow：当不想渲染所有子组件时可以将这个选项置为 true。

mount 方法的配置选项的几个字段就介绍到这里，为了避免篇幅过多，还是建议大家去看对应的 [API 文档](https://test-utils.vuejs.org/api/#mount)，讲得很详细。

调用 mount 方法会返回一个 VueWrapper 类型的对象，它提供了许多工具方法来方便对组件进行断言或更新组件的状态。比如上面几个示例的 text 方法就可以返回一个元素的文本内容，这里列举几个其他几个常用的方法，更多详情可以[看这](https://test-utils.vuejs.org/api/#wrapper-methods)：

- emitted：返回组件发出的所有事件，使用示例如下：

    ```js
    const Component = defineComponent({
      emits: ['fetch'],
      setup(props, { emit }) {
        emit('fetch', '123')
      },
    })

    test('Component', () => {
      const wrapper = mount(Component)

      expect(wrapper.emitted()).toHaveProperty('fetch')
      expect(wrapper.emitted('fetch')?.[0]).toEqual(['123'])
    })
    ```
- find：查询组件中的 DOM 节点，返回一个 DOMWrapper 类型的对象。DOMWrapper 在使用上与 VueWrapper 差不多，都可以使用很多工具方法；

- trigger：触发组件 DOM 事件：

    ```js
    const Component = defineComponent({
      data() {
        return {
          count: 0,
        }
      },
      template: '<button @click="count++">{{ count }}</button>.',
    })

    test('Component', async () => {
      const wrapper = mount(Component)
      const button = wrapper.find('button')
      expect(button.text()).toBe('0')

      await button.trigger('click')

      expect(button.text()).toBe('1')
    })
    ```
    
    注意，为了保证触发事件后进行断言时 DOM 已更新，trigger 方法返回了一个 Promise，它只有在 DOM 更新后才会 resolve，所以我们需要进行 await；
    
 - unmount：卸载组件。

Vue Test Utiles 还暴露了一个 [flushPromises](https://test-utils.vuejs.org/api/#flushpromises) 方法，调用并 await 它可以确保所有微任务（包括 DOM 更新）都会执行完毕。它内部同时使用了宏任务和微任务来达到这个目的。

Vue Test Utiles 的基本使用就介绍到这，之所以介绍得比较简短，除了节省篇幅外，主要原因是我们并不使用它来作为 Vue 组件的挂载库，我们使用的是 [Vue Testing Library](https://testing-library.com/docs/vue-testing-library/intro)。

#### 使用 Vue Testing Library

Vue Testing Library 是一个用于 Vue 的测试库，它内部依赖了 [DOM Testing Library](https://testing-library.com/docs/dom-testing-library/intro) 和 Vue Test Utils。相比于 Vue Test Utils，Vue Testing Library 可以使用更简洁的 API 来与组件进行交互，它摒弃了操作、查询 Vue 组件时需要使用的过度依赖其内部实现的 API，而将这些操作简化为最原始的，更加抽象的原生 DOM 操作。

Testing Library 是一个专注于模拟用户的行为进行测试的库，它只暴露可以让使用者以一种接近用户使用方式进行测试的 API，它的指导原则是：

> The more your tests resemble the way your software is used, the more confidence they can give you.
 
这同时也是我们对组件进行测试的测试原则，即我们的测试不应过度依赖待测试对象的内部实现，而是从一个用户的角度思考其输入和输出，大多数情况下，对于一个组件来说，其输入可以是：用户的交互、Props、其它从外部输入的数据（例如 store、API 调用）；其输出可以是：视图、事件、其它 API 调用（例如调用 router、store 的方法）。

只注重组件的输入输出可以让我们写出易维护的测试代码，让我们有信心对代码进行重构，当我们进行迭代时测试也会在合适的时候失败，而不是改个类名就直接报错。

Vue Testing Library 使用 Queries API 来查询组件内部的 DOM 结点，Queries API 是从 DOM Testing Library 引入的方法，我们来简单介绍一下。

（虽然我们使用 Vue Testing Library 进行测试，但是我还是推荐你阅读一下 Vue Test Utiles 的文档，因为前者也是基于 Vue Test Utiles 开发出来的，渲染组件的配置字段和更新组件的方法有部分重合；此外，它的文档还较系统地介绍了如何测试一个 Vue 组件，包括自定义事件、路由、状态管理等等，非常值得一读。）

##### Queries

如果只查询一个 DOM 结点的话，按照查询 DOM 的结果来分类，Queries API 可以分为 3 种：

- getBy**：当没有查询到或查询到多个结果时报错；
- queryBy**：当没有查询到时返回 null，查询到多个结果时报错；
- findBy**：异步查询 DOM，当没有查询到或查询到多个结果时报错，返回一个 Promise。这在查询只有在视图更新后才会变化的 DOM 时会很有用。

如果要查询多个 DOM 结点的话：

- getAllBy**：查询结果返回一个数组，其它与 getBy** 相同；
- queryAllBy**：没有查询到时返回空数组，查询到时返回一个数组；
- findAllBy**：查询结果返回一个数组，其它与 findBy** 相同。

按照查询 DOM 的方式来分类，可以分为 8 种，具体可以[看这里](https://testing-library.com/docs/vue-testing-library/cheatsheet#search-types)，就不列举了。同时文档还为这些 API 的使用优先级[排了序](https://testing-library.com/docs/queries/about#priority)。

DOM Testing Library 本质上是对给定的 DOM 元素进行各种 DOM API（如 `querySelector()`） 的调用最后返回查询结果，使用方式大致如下：

```js
const input = getByLabelText(container, 'Username')
```

可以看到，使用时需要先传入一个根节点，DOM Testing Library 会对其子元素进行查询。

由于 Vue 组件的根节点一般是固定的，Vue Testing Library 修改了 Queries API 的实现，省略了根节点的传入：

```js
const { getByText } = render(Component)

getByText('Hello World!')
```

##### render

render 方法用于挂载 Vue 组件，相当于 Vue Test Utils 的 mount 方法，但是略有不同，接口如下：

```js
function render(Component, options, callbackFunction) {
  return {
    ...DOMTestingLibraryQueries,
    container,
    baseElement,
    debug(element),
    unmount,
    html,
    emitted,
    rerender(props),
  }
}
```

使用方式与 mount 方法差不多，但是返回了 Queries API 和几个变量和方法，具体可以看[这里](https://testing-library.com/docs/vue-testing-library/api#rendercomponent-options)。

render 方法的内部实现也很简单，大致就是修改了组件挂载的节点然后调用 mount 方法而已。

##### fireEvent

fireEvent 方法顾名思义，用来给 DOM 结点触发事件，使用方式如下：

```js
await fireEvent.click(getByText('Click me'))
```

跟 Vue Test Utils 的 trigger 方法一样，为了保证 DOM 的更新，调用它会返回一个 Promise，我们需要对它进行 await。

fireEvent 的原理是对所传入的元素调用 dispatchEvent 方法触发事件，然后调用 Vue Test Utils 的 `flushPromises()` 等待 DOM 更新。

##### cleanup

cleanup 方法用于卸载所有已挂载的组件。Vue Testing Library 内部维护了一个存放已挂载组件的列表，当调用 render 函数时就会将所渲染的组件添加到该列表中。调用 cleanup 时就会对列表中的每个组件调用 Vue Test Utils 的 unmount 方法进行卸载。

在默认情况下 Vue Testing Library 会在 afterEach 钩子中调用 cleanup 函数，所以我们可以不用手动调用它。但是还有一个问题需要注意，我们放在后面讲。

------------------------------------------------------------------

Vue Testing Library 的基本使用就介绍到这里，API 不多，上手非常容易，另外它的源码量也不多，只有不到 200 行，感兴趣的同学可以阅读一下。

#### 内联组件库

如果我们所测试的组件依赖了组件库提供的组件的话，在 Vitest 下可能会出现报错：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09ce297550754b1aa099f3c345bd1e5c~tplv-k3u1fbpfcp-watermark.image?)

从报错信息可以看出，Vitest 无法识别 vant 某个组件的 CSS 文件。出现这个问题是因为 Vitest 在默认情况下不会对 `node_modules` 的模块进行转换，而是直接交给 Node 执行，所以当然就不认识 CSS 文件了。之所以这么做是因为 `node_modules` 里的包一般都是 Node 能识别的 ESM 或 CJS 格式，出于性能考虑，当然不必对它们进行处理，Vitest 也不会将它们纳入模块图。

所以这个报错的解决方法已经很明了了，就是让 Vitest 对 vant 进行转换，可以使用 `deps.inline` 选项来达到这个目的：

```ts
// vite.config.ts
test: {
  deps: {
    inline: ['vant'],
  },
},
```

#### 其它配置

测试的目录结构直接照搬 src 目录的就行，方便维护和后期迭代。

如果要使用 `vi.useFakeTimers()` 时记得这样做：

```js
vi.useFakeTimers({
  toFake: ['setTimeout', 'clearTimeout'],
})
```

具体原因可以看这条 [issue](https://github.com/vitest-dev/vitest/issues/649)。在示例项目中我已将以上代码放到 setup file 中。

如果你使用 Vite，还需要在配置文件加上一条配置：

```ts
resolve: {
    conditions: process.env.VITEST ? ['node'] : [],
},
```

具体看这个 [issue](https://github.com/vitest-dev/vitest/issues/1918)。

最后，将配置文件的 `test.globals` 置为 true，为什么呢？为了兼容 Jest 生态。现在大部分库都是兼容 Jest 的，这意味着它们会假定 expect、afterEach 等 API 是可以从全局获取的。

比如 Vue Testing Library 会在 afterEach 钩子中调用 cleanup 函数来卸载所有 Vue 组件：

```js
if (typeof afterEach === 'function' && !process.env.VTL_SKIP_AUTO_CLEANUP) {
  afterEach(() => {
    cleanup()
  })
}
```

如果不开启 globals 的话，我们就需要手动调用 cleanup。

再比如前文的 jest-dom 也还是需要引入所有 matcher 然后手动进行扩展，以及后文要介绍的 Pinia 提供的用于测试的 createTestingPinia 方法也是这样。所以，为了避免测试时出现无法预料的问题，还是建议开启 globals。

### 测试 LoginForm

测试实战的第一个示例，我们来测试 LoginForm，即登录表单组件。功能很简单，提交时进行调用 API 进行登录，登录成功后存储 token 并调用 router 跳转到新页面。此外还包含表单验证、按钮禁用的小功能。

所以我们要测试的功能和对应的用例如下：

- 填写表单登录成功后将 token 存储到本地存储。输入为用户填写表单，输出为 localStorage 的 token 字段。由于 jsdom 提供了本地存储的支持，所以我们可直接调用 localStorage。如果不支持的话，就需要 mock 了；

- 填写表单登录成功 1 秒后调用 router 跳转页面。输入为用户填写表单，输出为调用 `router.replace()` 和传入的参数。所以我们需要 mock `vue-router` 模块，才能断言对 `useRouter()` 等方法的调用。你可能会问，为什么不等跳转到新页面后直接断言页面的 URL 呢？
    
    由于我们仅仅挂载了待测的组件，如果要跳到新页面的话就需要使用 RouterView 组件，然后还需要挂载一个 APP 组件来放置 RouterView，接着配置路由表创建一个 router 实例，将其应用到 APP 组件中。可见工作量还是非常大的，如果不嫌麻烦的话就可以这样做。但是个人认为这么做也没有什么意义，本质上也还是模拟一个 router，因此直接 mock `vue-router` 模块就足够了。
    
- 提交表单时提交按钮被禁用、提交失败时按钮启用；输入为提交表单，输出为按钮的状态；

- 表单验证：输入框失焦或提交表单时如果有未填写的则显示提示信息。输入分别为输入框失焦和提交表单，输出为显示提示信息。

按照这样的思路，即组件功能的输入输出设计测试用例，是一个推荐的做法。

登录时需要发起请求，所以还需要模拟调用的后端 API。通过前面的学习你应该知道怎么模拟函数了，像这样：

```js
import * as loginAPI from '~/api/userManagement'

vi.spyOn(loginAPI, 'login').mockImplementation(vi.fn().mockResolvedValue({
  token: 'acbdefgfedbca123',
}))
```

如果要模拟返回成功结果，可以像上面这样使用 mockResolvedValue 方法，它可以模拟返回一个 resolve 的 Promise。如果要模拟失败结果，则可以使用 mockRejectedValue 方法：

```js
vi.mocked(loginAPI.login).mockImplementation(vi.fn().mockRejectedValue('rejected'))
```

现在我们就可以写出第一个测试用例：

```js
describe('填写表单进行登录', () => {
  test('输入用户名和密码进行登录可以登录成功, 将 token 存储到本地存储中', async () => {
    // 模拟后端 API
    vi.spyOn(loginAPI, 'login').mockImplementation(vi.fn().mockResolvedValue({
      token: 'acbdefgfedbca123',
    }))
    const { getByPlaceholderText, getByTestId } = render(LoginForm)
    expect(localStorage.getItem('token')).toBeNull()

    await fireEvent.update(getByPlaceholderText('用户名'), 'jeanmay')
    await fireEvent.update(getByPlaceholderText('密码'), 'password123456')
    await fireEvent.submit(getByTestId('form'))

    expect(localStorage.getItem('token')).toBe('acbdefgfedbca123')

    // 清除本地存储
    localStorage.removeItem('token')
    vi.clearAllMocks()
  })
})
```

注意，我将测试里的代码分为了四个步骤：

- 第一个步骤是进行测试前的初始化，完成模拟 API、渲染组件和“控制变量“这些准备工作；
- 第二个步骤是进行测试，即触发原先规定好的输入和输出，这里我们填写表单内容并提交。一定要记得调用 fireEvent 后还要 await 它确保视图更新；
- 第三个步骤是进行断言，断言输出结果是否符合我们的预期，这里断言了本地存储中是否有我们模拟的 token；
- 最后是进行测试的收尾，一些状态或副作用的清除在这一步完成，这里我们完成了本地存储的 token 和模拟的 API 调用记录的删除，此外还有 Vue Testing Library 自动帮我们卸载组件。

这四个步骤非常重要，按照这个方式来组织测试代码可以很清晰地表达测试的意图，确保测试的独立性和可维护性。

一些重复的初始化和收尾工作可以提取出来放到钩子函数中或提到更上层的作用域，抽离出来后最终代码是这样的：

```ts
describe('LoginForm', () => {
  afterEach(() => {
    vi.clearAllMocks()
  })

  describe('填写表单进行登录', () => {
    vi.spyOn(loginAPI, 'login').mockImplementation(vi.fn().mockResolvedValue({
      token: 'acbdefgfedbca123',
    }))

    afterEach(() => {
      localStorage.removeItem('token')
    })

    test('输入用户名和密码进行登录可以登录成功, 将 token 存储到本地存储中', async () => {
      const { getByPlaceholderText, getByTestId } = render(LoginForm)
      expect(localStorage.getItem('token')).toBeNull()

      await fireEvent.update(getByPlaceholderText('用户名'), 'jeanmay')
      await fireEvent.update(getByPlaceholderText('密码'), 'password123456')
      await fireEvent.submit(getByTestId('form'))

      // await waitFor(() => expect(localStorage.getItem('token')).toBe('acbdefgfedbca123'))
      expect(localStorage.getItem('token')).toBe('acbdefgfedbca123')
    })
  }）
})
```

接下来写第二个用例的代码，由于使用了 router，我们需要模拟 vue-router 模块，模拟代码如下：

```ts
import type * as VueRouter from 'vue-router'

const replace = vi.fn()
vi.mock('vue-router', async () => ({
  ...await vi.importActual<typeof VueRouter>('vue-router'),
  useRouter: () => ({
    replace,
  }),
}))
```

由于源代码使用的是 `router.replace()`，这里我们只需要模拟 useRouter 和 replace 就足够了。

测试代码我直接贴出来：

```js
test('输入用户名和密码进行登录可以登录成功, 1 秒后调用 router.replace()', async () => {
  const { getByPlaceholderText, getByTestId } = render(LoginForm)
  expect(replace).not.toHaveBeenCalled()

  await fireEvent.update(getByPlaceholderText('用户名'), 'jeanmay')
  await fireEvent.update(getByPlaceholderText('密码'), 'password123456')
  await fireEvent.submit(getByTestId('form'))
  vi.advanceTimersByTime(1000)

  expect(replace).toHaveBeenCalledTimes(1)
  expect(replace).toHaveBeenCalledWith('/address/shipAddress')
})
```

由于源代码中用到了定时器，我们还需要使用 `vi.useFakeTimers()`，这个工作已经在 setup file 中完成了就不必再做了。

其它几个测试比较简单，所以就不必多讲了。测试 LoginForm 的介绍就到这里了，在这一小节中，我讲了如何根据待测组件的功能从输入输出的角度设计测试用例、组织测试代码的四个步骤和常见的模拟模块的方式。

此外还有几个常用技巧，比如使用 `toBeInTheDocument()` 匹配器判断 DOM 是否存在、使用 `toBeEnabled()`、`toBeDisabled()` 判断按钮是否禁用或启用等等。

### 测试 AddressListItem

AddressListItem 组件通过 Props 接收地址信息，然后将其渲染到视图上，点击时跳转到新页面，长按一秒时抛出 longTouch 事件。

根据上一小节提供的方法应该可以很容易地想出如何设计测试用例，所以这里就不再介绍了。这里我们来细说一下点击跳转新页面这个功能，因为这个过程涉及到调用 store。

这个项目使用的状态管理是 Pinia，Pinia 提供了 createTestingPinia 方法来简化测试的复杂度，用法如下：

```js
render(Component, {
  global: {
    plugins: [createTestingPinia()],
  },
})
`````

调用 createTestingPinia 会返回一个专门用于测试的 pinia 实例，将其作为插件传入 `global.plugins` 之后，所有对 store 的获取都会返回一个模拟的 store 而不是原先定义的 store，所以我们不必担心调用 store 上的 action 或修改其中的状态会对其它测试或源代码中的 store 造成影响。这个模拟的 store 与原来的没有什么区别，唯一的一点不同是 pinia 会用一个模拟函数（比如 `vi.fn()`）来替换掉所有 action，所以我们可以直接对这些 action 进行监听而不必担心它会发起网络请求或修改状态。

（注：createTestingPinia 假定 `vi.fn()` 或 `jest.fn()` 是可以从全局获取的，所以需要开启 globals）

对 action 进行修改的源码是这样的：

```js
const createSpy = _createSpy || typeof jest !== 'undefined' && jest.fn || typeof vi !== 'undefined' && vi.fn
if (!createSpy)
  throw new Error('[@pinia/testing]: You must configure the `createSpy` option.')

pinia$1._p.push(({ store, options }) => {
  Object.keys(options.actions).forEach((action) => {
    store[action] = stubActions ? createSpy() : createSpy(store[action])
  })
  store.$patch = stubPatch ? createSpy() : createSpy(store.$patch)
})
```

stubActions 是传入 createTestingPinia 的一个选项。可以看到，如果 stubActions 为 false，则会使用原先的实现并启动监听。

除了传入 stubActions 选项外，我们还可以设置 store 的状态的初始值：

```js
render(Component, {
  global: {
    plugins: [
      createTestingPinia({
        initialState: {
          counter: { n: 20 },
        },
      }),
    ],
  },
})
```

如果需要改变 getter 的值，我们也可以强制对其进行写入：

```js
const counter = useCounter()

// @ts-expect-error: usually it's a number
counter.double = 2
```

但是需要使用 `@ts-expect-error` 注释绕过 TS 编译器的检查。

接下来我们来测试"点击后设置 store 的 currentAddressId" 这个用例，代码如下：

```js
const renderAddressListItem = () => {
  return render(AddressListItem, {
    props: {
      addressInfo,
    },
    global: {
      plugins: [createTestingPinia()],
    },
  })
}

describe('AddressListItem', () => {
  afterEach(() => {
    vi.clearAllMocks()
  })

  test('点击后设置 store 的 currentAddressId', async () => {
    const { getByTestId } = renderAddressListItem()
    const address = useAddressStore()
    expect(address.currentAddressId).toBe('')

    await fireEvent.click(getByTestId('item'))

    expect(address.currentAddressId).toBe(addressInfo.addressId)
  })
})
```

当调用 render 的配置项较多且重复时可以将这个操作抽离成一个函数，这里是 renderAddressListItem 函数，它初始化了用于展示的地址信息，并调用了  createTestingPinia 方法。

测试代码比较简单，没有什么可以讲的地方，使用和断言 store 的方式也跟测试 router 差不多。主要是学会 createTestingPinia 方法的使用。

### 测试 AddressList

AddressList 组件调用 store 的 action 获取地址列表数据并传入 AddressListItem，获取地址列表后及地址列表的数量变化时都会抛出 fetch 事件，此外监听 AddressListItem 的 longTouch 事件，事件回调中调用 action 删除地址列表项。

我们来看"获取并展示地址列表信息"这个测试的代码：

```js
test('获取并展示地址列表信息', async () => {
  const { findAllByTestId } = renderAddressList()

  expect(await findAllByTestId('item')).toHaveLength(3)
})
```

由于源代码中会调用 action 发起请求获取地址列表，这是一个异步的过程，所以需要使用 findAllByTestId()。

我们封装的用于渲染组件的函数如下：

```js
const renderAddressList = (stubs = false) => {
  const spy = () => {
    return vi.fn(async () => {
      const address = useAddressStore()
      address.addressInfoList.push(...mockedAddressInfoList)
    })
  }

  if (stubs) {
    const AddressListItem = defineComponent({
      emits: ['longTouch'],
      setup(props, { emit }) {
        const emitLongTouch = async () => {
          emit('longTouch')
        }
        emitLongTouch()
      },
      template: '<div />',
    })

    return render(AddressList, {
      global: {
        stubs: {
          AddressListItem,
        },
        plugins: [createTestingPinia({
          createSpy: spy,
        })],
      },
    })
  }
  else {
    return render(AddressList, {
      global: {
        plugins: [createTestingPinia({
          createSpy: spy,
        })],
      },

    })
  }
}
```

由于后面几个测试用例会测试接收 AddressListItem 的 longTouch 事件并删除列表项的功能逻辑，需要模拟 AddressListItem 组件，所以渲染组件时需要分为模拟和不模拟两种情况，通过 stubs 参数来控制，默认是 false。

另外，我们还自己定义了一个传入 createSpy 选项的 spy 函数，因为 AddressList 创建前就会立即调用 `store.getAddressInfoList()` 获取地址列表，这意味我们必须在开始渲染该组件前模拟这个 action，创建一个新的 createSpy 函数就可以达到这个目的。在 spy 函数中我们重写了所有 action，让它们都更新 `address.addressInfoList`，因为测试场景比较简单，所以这样做不会出现什么大问题，当我们需要在组件创建前实现不同的 action 时可以将 spy 函数作为参数传入。

如果组件在创建前后不会立即调用 action，我们不需要重写 createSpy，直接在挂载后修改就行，比如这个测试用例：

```js
test('监听到 Item 组件的 longTouch 事件后弹出弹窗，点击确定即可删除该 Item', async () => {
  mockedAddressInfoList.splice(0, 2)
  const { findAllByTestId, queryAllByTestId } = renderAddressList(true)
  const address = useAddressStore()
  vi.mocked(address.deleteAddress).mockImplementation(vi.fn(async () => {
    address.addressInfoList = []
  }))
  expect(await findAllByTestId('item')).toHaveLength(1)

  await fireEvent.click(screen.getByText('确认'))

  expect(address.deleteAddress).toHaveBeenCalledWith('3')
  expect(queryAllByTestId('item')).toHaveLength(0)
})
```

这里在组件挂载后调用了 mockImplementation 更改了 `address.deleteAddress` 的实现。

### 测试 AddressForm

测试 AddressForm 这里有两个地方需要注意。

一个是设置初始的 getter，虽然 createTestingPinia 只支持初始化 state，但是初始化 getter 也不难，因为 getter 本身就是从 state 计算得到的，所以直接设置初始 state 就可以了。

第二个是在测试用例内重写模块的模拟函数的实现，比如这个测试：

```js
test('正确填写表单并提交成功后，1 秒后调用 router.back()', async () => {
  const back = vi.fn()
  vi.mocked(useRouter, {
    partial: true,
  }).mockImplementation(() => ({
    back,
  }))
  const { getByPlaceholderText, getByText, getByRole, getByTestId } = renderAddressForm()
  expect(back).not.toHaveBeenCalled()

  await fireEvent.update(getByPlaceholderText('请填写收货人姓名'), addressInfo.name)
  await fireEvent.update(getByPlaceholderText('手机号码'), addressInfo.mobilePhone)
  await fireEvent.click(getByPlaceholderText('点击选择省市区'))
  await fireEvent.click(screen.getByText('确认'))
  await fireEvent.update(getByPlaceholderText('详细地址'), addressInfo.detailAddress)
  await fireEvent.click(getByText('家'))
  await fireEvent.click(getByRole('switch'))
  await fireEvent.submit(getByTestId('form'))
  vi.advanceTimersByTime(1000)

  expect(back).toHaveBeenCalledTimes(1)
})
```

需要注意的地方是，调用 `vi.mocked()` 是需要额外传入一个值为 true 的 partial 字段，表明只模拟模块的部分 API。

### 测试 Pinia stores

除了测试组件外，我们还需要测试 store，因为 store 通常管理一个或多个业务模块的状态，负责模块级别的数据层的调度和维护，是一个 Web 应用重要的组成部分，所以对它们进行测试是自动化测试中非常重要的一环。

在 Pinia 中测试 store 非常简单，因为本质上就是对一个个 getter 和 action 做单元测试，粒度比组件要小很多。唯一要注意的地方是要记得加上这一段代码：

```js
beforeEach(() => {
  setActivePinia(createPinia())
})
```

因为想要使用 store，需要有一个已注册的 pinia 实例，否则就需要手动将其传入 `useAddressStore()` 方法中，以上代码可以自动帮我们完成这件事情。

完成以上这件事后，剩下的事情就简单多了，也没啥好介绍的了，大伙们直接看仓库代码就够了。

-------------------------------------------------------------------

前端自动化测试的组件测试实战就到这里了，我重点介绍了进行组件测试时的测试原则、测试技巧和注意事项，如果你理解并熟练了之后就会发现写测试其实真的不难，本质上还是围绕组件功能的输入输出做文章，并按照四个步骤组织测试代码，剩下的就是对各种 API 的熟练程度了。

## 总结

从入门到实战，以上就是前端测试的介绍的全部内容了，希望对你有所帮助。另外，在示例项目中我还使用了 Cypress 进行端到端测试，感兴趣的同学可以看一下。