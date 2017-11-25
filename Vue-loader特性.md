## Vue-loader特性

### 一、ES2015

当项目中配置了 `babel-loader` 或者 `buble-loader`，`vue-loader` 会使用他们处理所有 `.vue` 文件中的 `<script>`部分，允许我们在 Vue 组件中使用 ES2015。

下面是导入其它 Vue 组件的典型写法：

```js
<script>
import ComponentA from './ComponentA.vue'
import ComponentB from './ComponentB.vue'

export default {
  components: {
    ComponentA,
    ComponentB
  }
}
</script>
```

我们使用 ES2015 的属性的简洁表示法去定义子组件，`{ ComponentA }` 是 `{ ComponentA: ComponentA }` 的简写，Vue 会自动的将 key 转换为`component-a`，所以你可以在 template 中使用 `<component-a>`。



#### 转换普通 `.js` 文件

由于 `vue-loader` 只处理 `.vue` 文件，你需要告诉 Webpack 如何使用 `babel-loader` 或者 `buble-loader` 处理普通 `.js` 文件，在 Webpack 中配置 `babel-loader` 或者 `buble-loader`。脚手架工具 `vue-cli` 已经为你做了这些。



#### 使用 `.babelrc` 配置 Babel

`babel-loader` 遵守 [`.babelrc`](https://babeljs.io/docs/usage/babelrc/)，因此这是配置 Babel presets 和插件推荐的方法。



### 二、CSS 作用域

#### 有作用域的 CSS

当 `<style>` 标签有 `scoped` 属性时，它的 CSS 只作用于当前组件中的元素。这类似于 Shadow DOM 中的样式封装。它有一些注意事项，但不需要任何 polyfill。它通过使用 PostCSS 来实现以下转换：

```html
<style scoped>
.example {
  color: red;
}
</style>

<template>
  <div class="example">hi</div>
</template>
```

转换结果：

```html
<style>
.example[data-v-f3f3eg9] {
  color: red;
}
</style>

<template>
  <div class="example" data-v-f3f3eg9>hi</div>
</template>
```



#### 混用本地和全局样式

你可以在一个组件中同时使用有作用域和无作用域的样式：

```css
<style>
/* 全局样式 */
</style>

<style scoped>
/* 本地样式 */
</style>
```



#### 子组件的根元素

使用 `scoped` 后，父组件的样式将不会渗透到子组件中。不过一个子组件的根节点会同时受其父组件有作用域的 CSS 和子组件有作用域的 CSS 的影响。这样设计是为了让父组件可以从布局的角度出发，调整其子组件根元素的样式。



#### 深度作用选择器

如果你希望 `scoped` 样式中的一个选择器能够作用得“更深”，例如影响子组件，你可以使用 `>>>` 操作符：

```css
<style scoped>
.a >>> .b { /* ... */ }
</style>
```

上述代码将会编译成：

```css
.a[data-v-f3f3eg9] .b { /* ... */ }
```

有些像 SASS 之类的预处理器无法正确解析 `>>>`。这种情况下你可以使用 `/deep/` 操作符取而代之——这是一个 `>>>`的别名，同样可以正常工作。



#### 动态生成的内容

通过 `v-html` 创建的 DOM 内容不受作用域内的样式影响，但是你仍然可以通过深度作用选择器来为他们设置样式。



#### 还有一些要留意

- **CSS 作用域不能代替 class**。考虑到浏览器渲染各种 CSS 选择器的方式，当 `p { color: red }` 设置了作用域时 (即与特性选择器组合使用时) 会慢很多倍。如果你使用 class 或者 id 取而代之，比如 `.example { color: red }`，性能影响就会消除。你可以在[这块试验田](https://stevesouders.com/efws/css-selectors/csscreate.php)中测试它们的不同。
- **在递归组件中小心使用后代选择器!** 对选择器 `.a .b` 中的 CSS 规则来说，如果匹配 `.a` 的元素包含一个递归子组件，则所有的子组件中的 `.b` 都将被这个规则匹配。



### 三、CSS Modules

[CSS Modules](https://github.com/css-modules/css-modules)是一个用于模块化和组合 CSS 的流行系统。`vue-loader` 提供了与 CSS 模块的一流集成，可以作为模拟 CSS 作用域的替代方案。



#### 使用

在你的 `<style>` 上添加 `module` 属性：

```css
<style module>
.red {
  color: red;
}
.bold {
  font-weight: bold;
}
</style>
```

这将为 `css-loader` 打开 CSS Modules 模式，生成的 CSS 对象将为组件注入一个名叫 `$style` 的计算属性，你可以在你的模块中使用动态 class 绑定：

```html
<template>
  <p :class="$style.red">
    This should be red
  </p>
</template>
```

由于它是一个计算属性，它也适用于 `:class` 的 object/array 语法：

```html
<template>
  <div>
    <p :class="{ [$style.red]: isRed }">
      Am I red?
    </p>
    <p :class="[$style.red, $style.bold]">
      Red and bold
    </p>
  </div>
</template>
```

你也可以在 JavaScript 访问它：

```js
<script>
export default {
  created () {
    console.log(this.$style.red)
    // -> "_1VyoJ-uZOjlOxP7jWUy19_0"
    // an identifier generated based on filename and className.
  }
}
</script>
```



#### 自定义注入名称

在 `.vue` 中你可以定义不止一个 `<style>`，为了避免被覆盖，你可以通过设置 `module` 属性来为它们定义注入后计算属性的名称。

```css
<style module="a">
  /* identifiers injected as a */
</style>

<style module="b">
  /* identifiers injected as b */
</style>
```



#### 配置 `css-loader` Query

CSS Modules 处理是通过 [css-loader](https://github.com/webpack/css-loader)。默认 query 如下：

```js
{
  modules: true,
  importLoaders: true,
  localIdentName: '[hash:base64]'
}
```

你可以使用 `vue-loader` 的 `cssModules` 选项去为 `css-loader` 添加 query 配置：

```js
// webpack 1
vue: {
  cssModules: {
    // overwrite local ident name
    localIdentName: '[path][name]---[local]---[hash:base64:5]',
    // enable camelCase
    camelCase: true
  }
}

// webpack 2
module: {
  rules: [
    {
      test: '\.vue$',
      loader: 'vue-loader',
      options: {
        cssModules: {
          localIdentName: '[path][name]---[local]---[hash:base64:5]',
          camelCase: true
        }
      }
    }
  ]
}
```



### 四、PostCSS

由`vue-loader` 处理的 CSS 输出，都是通过 [PostCSS](https://github.com/postcss/postcss) 进行作用域重写，你还可以为 PostCSS 添加自定义插件，例如 [autoprefixer](https://github.com/postcss/autoprefixer) 或者 [CSSNext](http://cssnext.io/)。

#### 使用配置文件

`vue-loader` 从 11.0 版本开始支持通过 [`postcss-loader`](https://github.com/postcss/postcss-loader#usage) 自动加载同一个配置文件：

- `postcss.config.js`
- `.postcssrc`
- `package.json` 中的 `postcss`

使用配置文件允许你在由 `postcss-loader` 处理的普通CSS文件和 `*.vue` 文件中的 CSS 之间共享相同的配置，这是推荐的做法。



#### 内联选项

或者，你可以使用 `vue-loader` 的 `postcss` 选项来为 `.vue` 文件指定配置。

Webpack 1.x 例子：

```js
// webpack.config.js
module.exports = {
  // other configs...
  vue: {
    // use custom postcss plugins
    postcss: [require('postcss-cssnext')()]
  }
}
```

Webpack 2.x 例子：

```js
// webpack.config.js
module.exports = {
  // other options...
  module: {
    // module.rules is the same as module.loaders in 1.x
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        // vue-loader options goes here
        options: {
          // ...
          postcss: [require('postcss-cssnext')()]
        }
      }
    ]
  }
}
```

除了插件数组之外，`postcss` 配置选项也接受：

- 返回插件数组的函数；
- 要传递给 PostCSS 处理器的包含 options 的对象。当你使用的 PostCSS 项目依赖自定义 parser/stringifiers 时，这很有用：

```js
postcss: {
  plugins: [...], // list of plugins
  options: {
    parser: sugarss // use sugarss parser
  }
}
```



### 五、热重载

"热重载"不是当你修改文件的时候简单重新加载页面。启用热重载后，当你修改 `.vue` 文件时，所有该组件的实例会被替换，**并且不需要刷新页面**。它甚至保持应用程序和被替换组件的当前状态！当你调整模版或者修改样式时，这极大的提高了开发体验。



#### 状态保留规则

- 当编辑一个组件的 `<template>` 时，这个组件实例将就地重新渲染，并保留当前所有的私有状态。能够做到这一点是因为模板被编译成了新的无副作用的渲染函数。
- 当编辑一个组件的 `<script>` 时，这个组件实例将就地销毁并重新创建。(应用中其它组件的状态将会被保留) 是因为 `<script>` 可能包含带有副作用的生命周期钩子，所以将“重新加载”替换为重新渲染是必须的，这样做可以确保组件行为的一致性。这也意味着，如果你的组件带有全局副作用，则整个页面将会被重新加载。
- `<style>` 会通过 `vue-style-loader` 自行热重载，所以它不会影响应用的状态。





#### 用法

当使用脚手架工具 `vue-cli` 时，热重载是开箱即用的。

当手动设置你的工程时，热重载会在你启动 `webpack-dev-server --hot` 服务时自动开启。



### 六、函数式组件

#### 函数式组件的模板

> 新增于 13.1.0，需要 Vue 版本 >= 2.5.0

从 `vue-loader >= 13.3.0` 开始，在一个 `*.vue` 文件中以单文件形式定义的函数式组件，现在在模板编译、有作用域的 CSS 和热重载也有了良好的支持。

要声明一个应该编译为函数式组件的模板，请将 `functional` 特性添加到模板块中。这样做以后就可以省略 `<script>` 块中的 `functional` 选项。

模板中的表达式会在[函数式渲染上下文](https://cn.vuejs.org/v2/guide/render-function.html#函数式组件)中求值。这意味着在模板中，prop 需要以 `props.xxx` 的形式访问：

```html
<template functional>
  <div>{{ props.foo }}</div>
</template>
```









