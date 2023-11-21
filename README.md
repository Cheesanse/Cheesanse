# Vue 项目添加测试用例

## 一 什么是自动化测试

自动化测试就是以软件代替人工，控制测试流程，比较实际结果与预期结果之间的差异

## 二 为什么需要测试

自动化测试能够预防无意引入的 bug，帮助更快速、自信地构建复杂的 Vue 应用。
另外，应用在开发过程中是不断变化的，自动化测试可以帮助你免去在每次改变后进行重复的测试步骤。

## 三 测试的类型

项目中涉及两种测试

- 单元测试： 检查给定函数的输入是否产生预期的输出
- 组件测试： 检查组件是否正常挂载与渲染，是否可以与之互动，以及表现是否符合预期

## 四 如何在项目中添加测试用例

### ① 安装插件及配置

#### Vitest 和 happy-dom

[Vitest 文档](https://cn.vitest.dev/)

- 安装

```
npm install -D vitest happy-dom @testing-library/vue
```

- 配置

Vitest 的主要优势之一是它与 Vite 的配置统一，如果想在测试期间使用不同的配置可以创建`vitest.congig.ts`
文件。为了不让 vitest 配置文件覆盖 vite 配置，可以用`mergeConfig`方法将 vite 配置和 vitest 配置合并。

项目中完整配置如下：

```ts
import { defineConfig, mergeConfig, configDefaults } from 'vitest/config';
import viteConfig from './vite.config';
import vue from '@vitejs/plugin-vue';
import { fileURLToPath, URL } from 'node:url';

export default mergeConfig(
  viteConfig,
  defineConfig({
    plugins: [vue()],
    resolve: {
      alias: {
        '@': fileURLToPath(new URL('./src', import.meta.url)),
      },
    },
    test: {
      include: ['./test/delivery.test.ts'], //运行的测试文件所在目录
      environment: 'happy-dom', //配置environment，使用happy-dom模拟DOM
      reporters: 'verbose',
      globals: true,
    },
  })
);
```

- 在`package.json`之中添加测试命令

将`--config`选项传递给 CLI`--config ./path/to/vitest.config.ts`

`package.json`中完整的测试命令

```json
"test": "vitest run  --config ./vitest.config.ts",
```

此时，可以在项目中创建文件名以`.test.ts`结尾的文件，并编写单元测试代码进行测试了。
若需要进行组件测试，还需要安装一下工具

#### Vue Test Utils

[Vue Test Utils 文档](https://v1.test-utils.vuejs.org/zh/)

- 安装

```
npm install --save-dev @vue/test-utils
```

- 配置

不需要额外配置

由于项目中使用了`pinia`进行状态管理，所以还需要安装`@pinia/testing`

#### pinia testing

[pinia testing 文档](https://pinia.vuejs.org/zh/cookbook/testing.html)

- 安装

```
npm i -D @pinia/testing@0.1.0
```

- 配置

不需要额外配置

现在，所有必要的工具都已经安装完成了。可以添加测试用例了。

### ② 添加.test.ts 文件 编写测试用例

自动化测试的意义在于代替人工进行测试流程，所以，编写测试用例的原则就是：人工在页面如何操作，测试用例中用代码实现相同的操作。
项目中以模块别建立测试文件进行测试。

以报价模块测试用例为例，解释如何编写测试用例。

#### § 创建.test.ts 文件

在项目中创建.test.ts 文件
并在`vitest.config.ts`文件中的`test.include`字段中添加创建的测试文件路径。

#### § 测试文件中添加`describe`块

`describe(name, fn)`函数创建一个包含相关测试簇的块。第一个参数`string`类型，是测试描述。
第二个参数是回调函数。函数体中可以添加`test`块。

例如：

```ts
describe('报价页面测试', () => {
  test('测试数量显示是否正确', async () => {});
  test('点击筛选按钮，输入筛选条件，点击重置按钮', async () => {});
});
```

`test`块可以直接在顶层，不用`describe`包裹。但是为了使相关联的测试块分组，使用`describe`包裹是很方便的。

#### § `beforeEach`函数

`beforeEach(fn, timeout)`函数在每个测试运行之前都会运行，第一个参数是回调函数，返回一个`promise`在运行测试之前会等待这个`promise`resolve。
第二个参数是可选的，是延迟时间。

可以在这个函数中添加测试前必要且重复的工作。例如，创建 pinia 实例和模拟路由等。

#### § 创建 pinia 实例

固定写法

```ts
setActivePinia(createPinia());
```

要对`store`进行单元测试，最重要的就是创建一个 pinia 实例，这样它就会被任何`useStore()`调用自动接收，而不需要去手动传递。

#### § 清空 DOM 文档

测试中可能会重复挂载组件，所以每次测试运行之前清空 DOM 文档(写在`beforeEach`回调函数中)

```ts
document.body.innerHTML = `
      <div>
        <div id="app"></div>
      </div>
    `;
```

#### § 模拟路由

测试时直接挂载组件，没有路由。所以对于涉及路由传参的页面可以在测试前模拟路由，指定一个路由返回值。

[vi.mock()文档](https://cn.vitest.dev/api/vi#vi-mock)

`vi.mock(path, factory)`用一个新的模块替换以`import`方式导入的 path 路径下的模块。如果定义了 factory，所有的 import 将返回 factory 的返回值。
在 factory 的第一个参数中接收一个`importOriginal`，可以在 factory 中获取原始模块。

> 注意：如果模拟的模块中有默认导出，factory 函数返回值，需要指定一个`default`关键字

```ts
vi.mock('vue-router', async importOriginal => {
  const mod = (await importOriginal()) as object; //路由原始模块
  return {
    ...mod,
    default: {},
    useRoute: vi.fn(() => ({
      params: {
        rfqType: 'N',
      },
    })),
    useRouter: vi.fn(() => ({
      push: () => {},
    })),
  };
});
```

#### § 清空模拟历史并重新实现模拟

测试运行之后有必要重新实现模拟，防止报错。

`afterAll()` 在所有测试运行完成后执行。

```ts
afterAll(() => {
  vi.restoreAllMocks();
});
```

#### § 添加测试函数

以上测试前的准备工作都已就绪，可以开始编写测试用例了。

在`describe`块中添加`test`块。所有的测试代码都在`test`块中。
`test(name, fn, timeout)`测试方法，运行测试就在运行测试文件中的每一个 test 方法。第一个参数，string 类型，对于测试的描述。
第二个参数，回调函数，测试函数体，可以是异步函数。第三个参数，可选参数，指定一个延时。

#### § 挂载组件

[mount()文档](https://v1.test-utils.vuejs.org/zh/api/#mount)
[mount 可以传递的挂载选项文档](https://v1.test-utils.vuejs.org/zh/api/options.html#context)
进行组件测试第一步就是挂载组件，使用 vue test utils 的`mount`方法挂载组件。
`mount(component, options)`。第一个参数，需要挂载的组件。第二个参数，挂载选项。可以指定组件中的`props`，用到的插件(例如，吐司和 loading)等。

例如：

```ts
const wrapper = mount(SpViewRfqMain, {
  props: {},
  global: {
    plugins: [toast, loading, createTestingPinia()],
  },
});
```

> 上面的 plugins 中的`createTestingPinia()`，返回一个仅用于组件单元测试的 pinia 实例

#### § 查找页面元素或组件

挂载组件后就可以用代码实现页面交互。
交互首先需要找到要操作的元素或组件。

查找组件主要有以下四个函数:

1. `find(selector)` 参数是个字符串，传入一个选择器，返回找到的第一个元素
2. `findAll(selector)`,返回一个找到的元素的列表可以通过`.at(index)`选取指定的元素，也可以使用`[index]`方试选取指定元素
3. `findComponent(Component)` 参数是组件，返回找到的第一个组件
4. `findAllComponents(Component)`,返回所有找到的组件列表。
   例如:

```ts
const filterBtn = wrapper.find('.btn-filter');
const resetBtn = wrapper.findAll('.filter-inner-btn').at(0);
const itemDetails = wrapper.findComponent(ItemDetail);
const itemDetail = wrapper.findAllComponents(ItemDetail).at(0);
```

#### § 设置值

[设置值的方法](https://v1.test-utils.vuejs.org/zh/api/wrapper/#setchecked)

找到元素后可以进行交互：点击元素或者给元素设置值。
以 input 设置值为例：

```ts
const itemNameInput = wrapper.find('#inputPassword4');
itemNameInput.setValue('钢笔');
```

#### § 派发事件

[trigger()文档](https://v1.test-utils.vuejs.org/zh/api/wrapper/#trigger)

在 DOM 节点上异步触发一个事件使用`trigger()`

例如：

```ts
const filterBtn = wrapper.find('.btn-filter');
await filterBtn.trigger('click');
```

#### § 模拟后台调用

页面交互后一般都会涉及后台调用，测试时可以模拟返回值。
模拟后台主要分两步

1.监听函数调用

[vi.spyOn()文档](https://cn.vitest.dev/api/vi#vi-spyon)
[mockImplementation()文档](https://cn.vitest.dev/api/mock#mockimplementation)
需要用到`vi.spyOn(object: T, method: K, accessType?: 'get' | 'set')`,第一个参数，监听方法的所属对象。
第二个参数，监听的函数名。第三个参数可选。
可以调用`vi.spyOn().mockImplementation(fn:Function)`,传递一个函数作为 spyOn 创建的 mock 的实现

2.手动改变 state
[变更 state 文档](https://pinia.vuejs.org/zh/core-concepts/state.html#mutating-the-state)
[测试中变更 state 例子](https://pinia.vuejs.org/zh/cookbook/testing.html#unit-testing-components)

项目中通过`store.$patch`方式改变 state

> 注意：只有页面中使用`$subscribe`方式监听 state，才能响应`store.$patch`对 state 的改变。页面中若使用`$onAction`方式监听 action 的调用，以这种方式改变 state 后并不会执行`onAction`的`after()`回调。

模拟后台调用例子：

```ts
const store = useRfqStore();

const spy = vi.spyOn((wrapper.vm as any).SpViewRfqMainStore, 'searchSupplierRFQInfoList').mockImplementation(() => {
  store.$patch({ mark: 'searchSupplierRFQInfoList', getRFQListResult: searchSupplierRFQInfoListResult } as any);
});

//必要时可以手动调用
await(wrapper.vm as any).SpViewRfqMainStore.searchSupplierRFQInfoList(param);
```

#### § 判断结果是否符合预期

无论哪种测试，都需要判断测试结果是否符合预期。此时就需要用到断言函数和匹配器函数

[expect()文档](https://cn.vitest.dev/api/expect#expect)
[匹配器函数文档](https://cn.vitest.dev/api/expect#not)

断言函数:`expect()`

匹配器函数:形如`toBe*()`或`not.toBe*()`

另外，vue Test Utils 中提供一些获取 class 名、判断组件是否存在等方法

[vue test utils 提供方法](https://v1.test-utils.vuejs.org/zh/api/wrapper/#%E6%96%B9%E6%B3%95)

断言匹配函数例子：

```ts
//判断筛选按钮是否变色
expect(filterBtn.classes()).toContain('btn-filter-changed');
```

## 五 结语

以上就是 Vue 项目添加测试用例涉及到的内容，根据上面列举的步骤结合官方文档及例子，就可以完成 Vue 手机项目的单元测试和组件测试。
