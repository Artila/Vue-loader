## Vue-loader配置

### 一、预处理器

#### 使用预处理器

在 Webpack 中，所有的预处理器需要匹配对应的 loader。`vue-loader` 允许你使用其它 Webpack loader 处理 Vue 组件的某一部分。它会根据 `lang` 属性自动推断出要使用的 loader。

#### CSS

例如，使用 SASS 编译我们的 `<style>` 语言块：

```js
npm install sass-loader node-sass --save-dev
```

```css
<style lang="sass">
  /* write sass here */
</style>
```

在内部，`<style>` 标签中的内容将会先由 `sass-loader` 进行处理，然后再传递进行下一步处理。

#### sass-loader 警告

与名称相反，[*sass*-loader](https://github.com/jtangelder/sass-loader) 默认解析 *SCSS* 语法。如果你想要使用 *SASS* 语法，你需要配置 `vue-loader` 的选项：

```js
{
  test: /\.vue$/,
  loader: 'vue-loader',
  options: {
    loaders: {
      scss: 'vue-style-loader!css-loader!sass-loader', // <style lang="scss">
      sass: 'vue-style-loader!css-loader!sass-loader?indentedSyntax' // <style lang="sass">
    }
  }
}

```

如要获得更多关于 `vue-loader` 的配置信息，请查看 [loader 进阶配置](https://vue-loader.vuejs.org/zh-cn/configurations/advanced.html)章节。

#### 加载一个全局设置文件

在每个组件里加载一个设置文件，而无需每次都将其显式导入，是一个常见的需求。比如为所有组件全局使用 scss 变量。为了达成此目的：

```js
npm install sass-resources-loader --save-dev
```

然后增加下面的 webpack 规则：

```js
{
  loader: 'sass-resources-loader',
  options: {
    resources: path.resolve(__dirname, '../src/style/_variables.scss')
  }
}

```

举个例子，如果你使用了 [vuejs-templates/webpack](https://github.com/vuejs-templates/webpack)，请如下修改 `build/utils.js`：

```js
scss: generateLoaders('sass').concat(
  {
    loader: 'sass-resources-loader',
    options: {
      resources: path.resolve(__dirname, '../src/style/_variables.scss')
    }
  }
),

```

在这个文件里，为了避免在最终编译后的文件中出现重复的 CSS，建议只包含变量、mixins 等。

#### JavaScript

Vue 组件中的所有 JavaScript 默认使用 `babel-loader` 处理。你也可以改变处理方式：

```
npm install coffee-loader --save-dev
```

```js
<script lang="coffee">
  # Write coffeescript!
</script>
```

#### 模版

模版的处理方式略有不同，因为大多数 Webpack 模版处理器 (比如 `pug-loader`) 会返回模版处理函数，而不是编译的 HTML 字符串，我们使用原始的 `pug` 替代 `pug-loader`：

```
npm install pug --save-dev
```

```
<template lang="pug">
div
  h1 Hello world!
</template>
```

> **重要:** 如果你使用 `vue-loader@<8.2.0`，你还需要安装 `template-html-loader`。

#### 行内 Loader Requests

你可以在 `lang` 属性中使用 [Webpack loader requests](https://webpack.github.io/docs/loaders.html#introduction)：

```css
<style lang="sass?outputStyle=expanded">
  /* use sass here with expanded output */
</style>
```

但是，这使你的 Vue 组件只适用于 Webpack，不能与 Browserify 和 [vueify](https://github.com/vuejs/vueify) 一同使用。**如果你打算将你的 Vue 组件作为可重复使用的第三方组件，请避免使用这个语法。**





### 二、资源路径处理

默认情况下，`vue-loader` 使用 [css-loader](https://github.com/webpack/css-loader) 和 Vue 模版编译器自动处理样式和模版文件。在编译过程中，所有的资源路径例如 `<img src="...">`、`background: url(...)` 和 `@import` **会作为模块依赖**。

例如，`url(./image.png)` 会被转换为 `require('./image.png')`，而

```html
<img src="../image.png">
```

将会编译为：

```js
createElement('img', { attrs: { src: require('../image.png') }})
```

因为 `.png` 不是一个 JavaScript 文件，你需要配置 Webpack 使用 [file-loader](https://github.com/webpack/file-loader) 或者 [url-loader](https://github.com/webpack/url-loader) 去处理它们。`vue-cli` 脚手器工具已经为你配置好了。

使用它们的好处：

1. `file-loader` 可以指定要复制和放置资源文件的位置，以及如何使用版本哈希命名以获得更好的缓存。此外，这意味着 **你可以就近管理图片文件，可以使用相对路径而不用担心布署时URL问题**。使用正确的配置，Webpack 将会在打包输出中自动重写文件路径为正确的URL。
2. `url-loader` 允许你有条件将文件转换为内联的 base-64 URL (当文件小于给定的阈值)，这会减少小文件的 HTTP 请求。如果文件大于该阈值，会自动的交给 `file-loader` 处理。





### 三、进阶配置

你有时可能想实现：

1. 对语言应用自定义 loader string，而不是让 `vue-loader` 去推断；
2. 覆盖默认语言的内置 loader 配置。
3. 使用自定义 loader 预处理或后处理特定语言块。

为此，请指定 `vue-loader` 的 `loaders` 选项：

> 注意 `preLoaders` 和 `postLoaders` 只在 10.3.0+ 版本支持

#### Webpack 2.x

```js
module.exports = {
  // other options...
  module: {
    // module.rules 与 1.x 中的 module.loaders 相同
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          // `loaders` 覆盖默认 loaders。
          // 以下配置会导致所有无 "lang" 特性的 <script> 标签加载 coffee-loader
          loaders: {
            js: 'coffee-loader'
          },

          // `preLoaders` 会在默认 loaders 之前加载。
          // 你可以用来预处理语言块——一个例子是用来处理构建时的 i18n
          preLoaders: {
            js: '/path/to/custom/loader'
          },

          // `postLoaders` 会在默认 loaders 之后加载。
          //
          // - 对于 `html`, 默认 loader 返回会编译为 JavaScript 渲染函数
          //
          // - 对于 `css`, 由`vue-style-loader` 返回的结果通常不太有用。使用 postcss 插件将会是更好的选择。
          postLoaders: {
            html: 'babel-loader'
          }

          // `excludedPreLoaders` 应是正则表达式
          excludedPreLoaders: /(eslint-loader)/
        }
      }
    ]
  }
}

```

#### Webpack 1.x

```js
// webpack.config.js
module.exports = {
  // other options...
  module: {
    loaders: [
      {
        test: /\.vue$/,
        loader: 'vue'
      }
    ]
  },
  // vue-loader configurations
  vue: {
    loaders: {
      // same configuration rules as above
    }
  }
}

```

进阶配置更实际的用法是[提取组件中的 CSS 到单个文件](https://vue-loader.vuejs.org/zh-cn/configurations/extract-css.html)。



### 四、提取 CSS 文件

#### 提取 CSS 到单个文件

```
npm install extract-text-webpack-plugin --save-dev
```

#### 简单的方法

> requires vue-loader@^12.0.0 and webpack@^2.0.0

```js
// webpack.config.js
var ExtractTextPlugin = require("extract-text-webpack-plugin")

module.exports = {
  // other options...
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          extractCSS: true
        }
      }
    ]
  },
  plugins: [
    new ExtractTextPlugin("style.css")
  ]
}

```

上述内容将自动处理 `*.vue` 文件内的 `<style>` 提取，并与大多数预处理器一样开箱即用。

注意这只是提取 `*.vue` 文件 - 但在 JavaScript 中导入的 CSS 仍然需要单独配置。

#### 手动配置

将所有 Vue 组件中的所有已处理的 CSS 提取为单个 CSS 文件配置示例：

#### Webpack 2.x

```js
// webpack.config.js
var ExtractTextPlugin = require("extract-text-webpack-plugin")

module.exports = {
  // other options...
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          loaders: {
            css: ExtractTextPlugin.extract({
              use: 'css-loader',
              fallback: 'vue-style-loader' // <- 这是vue-loader的依赖，所以如果使用npm3，则不需要显式安装
            })
          }
        }
      }
    ]
  },
  plugins: [
    new ExtractTextPlugin("style.css")
  ]
}

```

#### Webpack 1.x

```js
npm install extract-text-webpack-plugin --save-dev

// webpack.config.js
var ExtractTextPlugin = require("extract-text-webpack-plugin")

module.exports = {
  // other options...
  module: {
    loaders: [
      {
        test: /\.vue$/,
        loader: 'vue'
      },
    ]
  },
  vue: {
    loaders: {
      css: ExtractTextPlugin.extract("css"),
      // 你还可以引入 <style lang="less"> 或其它语言
      less: ExtractTextPlugin.extract("css!less")
    }
  },
  plugins: [
    new ExtractTextPlugin("style.css")
  ]
}
```



### 五、自定义块

> 在大于 10.2.0 中支持

在 `.vue` 文件中，你可以自定义语言块。自定义块的内容将由 `vue-loader` 的 options 中的 `loader` 对象中指定的 loader 处理，然后被组件模块依赖。类似 [Loader 进阶配置](https://vue-loader.vuejs.org/zh-cn/configurations/advanced.html)中的配置，但使用的是标签名匹配，而不是 `lang` 属性。

如果找到一个与自定义块匹配的 loader，该自定义块将被处理；否则自定义块将被忽略。 另外，如果找到的 loader 返回一个函数，该函数将以 `* .vue` 文件的组件作为参数来调用。

#### 单个文档文件的例子

这是提取自定义块 `<docs>` 的内容到单个 docs 文件中的例子：

##### component.vue

```vue
<docs>
## This is an Example component.
</docs>

<template>
  <h2 class="red">{{msg}}</h2>
</template>

<script>
export default {
  data () {
    return {
      msg: 'Hello from Component A!'
    }
  }
}
</script>

<style>
comp-a h2 {
  color: #f00;
}
</style>

```

##### webpack.config.js

```js
// Webpack 2.x
var ExtractTextPlugin = require("extract-text-webpack-plugin")

module.exports = {
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          loaders: {
            // 提取 <docs> 中的内容为原始文本
            'docs': ExtractTextPlugin.extract('raw-loader'),
          }
        }
      }
    ]
  },
  plugins: [
    // 输出 docs 到当个文件中
    new ExtractTextPlugin('docs.md')
  ]
}

```

#### 运行时可用的文档

这里有一个向组件注入 `<docs>` 自定义块使其在运行时可用的例子。

##### docs-loader.js

为了使得自定义块内容被注入，我们需要一个自定义的 loader：

```vue
module.exports = function (source, map) {
  this.callback(null, 'module.exports = function(Component) {Component.options.__docs = ' +
    JSON.stringify(source) +
    '}', map)
}
```

##### webpack.config.js

现在我们将为 `<docs>` 自定义块配置我们的 webpack 自定义 loader。

```js
const docsLoader = require.resolve('./custom-loaders/docs-loader.js')

module.exports = {
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          loaders: {
            'docs': docsLoader
          }
        }
      }
    ]
  }
}

```

##### component.vue

现在我们可以在运行时访问已导入组件的 `<docs>` 块内容了。

```vue
<template>
  <div>
    <component-b />
    <p>{{ docs }}</p>
  </div>
</template>

<script>
import componentB from 'componentB';

export default = {
  data () {
    return {
      docs: componentB.__docs
    }
  },
  components: {componentB}
}
</script>
```




