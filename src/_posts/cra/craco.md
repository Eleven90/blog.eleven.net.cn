---
title: 无 eject 重写 CRA 配置 — Craco 详解
date: 2020-09-11 16:43:40
category: React
tags: create-react-app
cover: /img/basketball.jpg
---

> 内置 craco 扩展配置的 CRA Template，可直接使用。  
> TS 版本：[@eleven.fe/cra-template-typescript](https://www.npmjs.com/package/@eleven.fe/cra-template-typescript)
> ES 版本：[@eleven.fe/cra-template](https://www.npmjs.com/package/@eleven.fe/cra-template)  

使用 CRA 脚手架创建的项目，如果想要修改编译配置，通常可能会选择 `npm run eject` 弹出配置后魔改。但是，eject 是不可逆操作，弹出配置后，你将无法跟随官方的脚步去升级项目的 react-scripts 版本。

如果想要无 eject 重写 CRA 配置，一般可以有以下几种方式

1. 通过 CRA 官方支持的 `--scripts-version` 参数，创建项目时使用自己重写过的 react-scripts 包
2. 使用 [react-app-rewired](https://github.com/timarney/react-app-rewired) + [customize-cra](https://github.com/arackaf/customize-cra) 组合覆盖配置
3. 使用 [craco](https://github.com/gsoft-inc/craco) 覆盖配置

第一种方式很棒，但这里暂时不做讨论，感兴趣的可以从下方链接了解更多：

1. CRA 官方的介绍：[Alternatives to Ejecting](https://create-react-app.dev/docs/alternatives-to-ejecting/)
2. [Customizing create-react-app: How to Make Your Own Template](https://auth0.com/blog/how-to-configure-create-react-app/)

第二种方式相对第三种略复杂一些，并且我注意到 AntDesign 官方也开始推荐 [craco](https://github.com/gsoft-inc/craco) 了，这里详细讨论一下第三种 [craco](https://github.com/gsoft-inc/craco) 的使用，经过项目的实测，用起来还算顺手。

## 基础配置

1. 安装包

    ```sh
    yarn add @craco/craco
    ```

2. 项目根目录创建 `craco.config.js` 文件

    ```js
    /* craco.config.js */

    module.exports = {
      ...
    }
    ```

3. 修改 package.json 的 scripts 命令

    ```json
    /* package.json */

    "scripts": {
    -   "start": "react-scripts start",
    +   "start": "craco start",
    -   "build": "react-scripts build",
    +   "build": "craco build"
    -   "test": "react-scripts test",
    +   "test": "craco test"
    }
    ```
    
基础的配置到此完成了，接下来是处理各种配置的覆盖，完整的 craco.config.js 配置文件结构，可以在 craco 官方的文档中详细查询：[Configuration Overview](https://github.com/gsoft-inc/craco/blob/master/packages/craco/README.md#configuration-overview) 。
    
## 区分环境

@craco/craco 提供了 `whenDev、whenProd、whenTest、when` 函数，用于在书写 craco.config.js 配置文件时根据不同的编译环境确定配置，先看一下 craco 相关的源码：

```js
function when(condition, fct, unmetValue) {
    if (condition) {
        return fct();
    }

    return unmetValue;
}

function whenDev(fct, unmetValue) {
    return when(process.env.NODE_ENV === "development", fct, unmetValue);
}

function whenProd(fct, unmetValue) {
    return when(process.env.NODE_ENV === "production", fct, unmetValue);
}

function whenTest(fct, unmetValue) {
    return when(process.env.NODE_ENV === "test", fct, unmetValue);
}

module.exports = {
    when,
    whenDev,
    whenProd,
    whenTest
};
```

- `whenDev、whenProd、whenTest` 分别对应的是 `process.env.NODE_ENV` 的 3 种值（`development`、`production`、`test`），接收两个参数：

    - 第一个参数是 function，对应该编译环境下，执行函数，函数一般应当将需要的配置作为返回值。
    - 第二个参数是默认值，非指定编译环境时，返回该默认值。
    
- `when` 是区分环境的万金油函数，接收 3 个参数：

    - 第一个参数是判断的变量。
    - 第二个参数是 function，参数一的变量为 true 时，执行该函数，函数一般应当将需要的配置作为返回值。
    - 第三个参数是默认值，参数一的变量为 false 时，返回该默认值。
    
引用方式：

```js
const { whenDev, whenProd, when } = require('@craco/craco')
```

下方的示例中有多处使用，可以参照了解其用途。
    
## 常用配置

craco 有提供一些 [plugins](https://github.com/gsoft-inc/craco#community-maintained-plugins)，集成了诸多功能，让覆盖配置变得更加容易。

除此之外，则需要我们对 webpack 的配置有一定的了解，根据 craco.config.js 的文件结构去增加配置。

### 通过 configure 函数扩展 webpack 配置

几乎所有的 webpack 配置均可以在 configure 函数中读取、覆盖，webpack 详细的配置参数结构可以查阅 webpack 官方的文档：https://webpack.js.org/configuration/ 。

需要注意一点，configure 既可以定义为对象，也可以定义为函数，使用过程中，我发现二者是互斥的关系。这里推荐使用函数形式，因为当我想要覆盖 `mini-css-extract-plugin` 的配置时，发现对象形式的 configure 定义，不能达到目的。

configure 可以扩展所有的 webpack 配置，你可以将所有的配置都在 configure 中写完，但是，craco 提供了若干快捷的方式去定义指定的配置，例如：babel、alias、webpack 的 plugins 等。因此，推荐有快捷的方式尽量用快捷方式，搞不定的才放到 configure 中去定义。

```js
/* craco.config.js */
const { whenDev, whenProd, when } = require('@craco/craco')

module.exports = {
  babel: {},
  webpack: {
    /**
     * 重写 webpack 任意配置
     *  - 与直接定义 configure 对象方式互斥
     *  - 几乎所有的 webpack 配置均可以在 configure 函数中读取，然后覆盖
     */
    configure: (webpackConfig, { env, paths }) => {
      // 修改 output
      webpackConfig.output = {
        ...webpackConfig.output,
        ...{
          filename: whenDev(() => 'static/js/bundle.js', 'static/js/[name].js'),
          chunkFilename: 'static/js/[name].js',
        },
      }

      // 关闭 devtool
      webpackConfig.devtool = false

      // 配置扩展扩展名
      webpackConfig.resolve.extensions = [
        ...webpackConfig.resolve.extensions,
        ...['.scss', '.less'],
      ]

      // 配置 splitChunks
      webpackConfig.optimization.splitChunks = {
        ...webpackConfig.optimization.splitChunks,
        ...{
          chunks: 'all',
          name: true,
        },
      }

      // 覆盖已经内置的 plugin 配置
      webpackConfig.plugins.map((plugin) => {
        whenProd(() => {
          if (plugin instanceof MiniCssExtractPlugin) {
            Object.assign(plugin.options, {
              filename: 'static/css/[name].css',
              chunkFilename: 'static/css/[name].css',
            })
          }
        })
        return plugin
      })

      return webpackConfig
    },
  }
}
```

### 扩展 babel 配置

虽然可以在 configure 中定义 babel 配置，但 craco 也提供了快捷的方式单独去书写，添加 `@babel/preset-env` 配置示例如下：

```js
/* craco.config.js */

module.exports = {
  babel: {
    presets: [
      [
        '@babel/preset-env',
        {
          modules: false, // 对ES6的模块文件不做转化，以便使用tree shaking、sideEffects等
          useBuiltIns: 'entry', // browserslist环境不支持的所有垫片都导入
          // https://babeljs.io/docs/en/babel-preset-env#usebuiltins
          // https://github.com/zloirock/core-js/blob/master/docs/2019-03-19-core-js-3-babel-and-a-look-into-the-future.md
          corejs: {
            version: 3, // 使用core-js@3
            proposals: true,
          },
        },
      ],
    ],
    plugins: [
      
    ],
  },
}
```

更详细的 babel 相关配置，推荐我之前整理的：https://webpack.eleven.net.cn/content/babel/ 。

### 扩展 webpack alias（别名）

虽然可以在 configure 中定义 alias，但 craco 也提供了快捷的方式单独去书写。

```js
/* craco.config.js */

module.exports = {
  babel: {},
  webpack: {
    configure: (webpackConfig, { env, paths }) => {
    
    },
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  }
}
```

这里有一个坑点，如果你使用的是 cra 创建的 typescript 项目，添加完 alias 后，你还需要在 tsconfig.json 文件中新增配置，否则会提示出错。并且，新版本 cra 创建的项目，在编译时会自动格式化 tsconfig.json，你向里面添加的若干配置，可能会自动被清除掉。

怎么办？这里寻找到一个机智的办法，利用 tsconfig 的 extends 参数去扩展需要的配置。

tsconfig.json

```json
{
  "extends": "./tsconfig.edit.json",
  "compilerOptions": {
    "target": "es5",
    "lib": [
      "dom",
      "dom.iterable",
      "esnext"
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
    "jsx": "react"
  },
  "include": [
    "src"
  ]
}
```

tsconfig.edit.json

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}

```

通过继承的方式，添加如上配置，alias 修改即可完成。

另外，craco 也提供了一个 plugin 专门用于修改 alias，以及处理 tsconfig 的问题，[传送门](https://github.com/risenforces/craco-alias)，该页面下方的 Examples 中，详细说明了 tsconfig 的方案，推荐使用。

### 新增 webpack plugins

如果想要新增一些 webpack plugins，可以直接在与 configure 平级的位置添加。当然，你在 configure 里添加也可以，但不推荐。

需要注意的是，如果你想修改内置的某些 webpack plugin 参数，必须到 configure 里去修改，示例见上方的 configure 介绍。

示例如下：

```js
/* craco.config.js */
const { whenDev, whenProd, when } = require('@craco/craco')
const CircularDependencyPlugin = require('circular-dependency-plugin')
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer')

module.exports = {
  babel: {},
  webpack: {
    configure: (webpackConfig, { env, paths }) => {
    
    },
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
    plugins: [ // 新增 webpack plugin
      // 新增模块循环依赖检测插件
      ...whenDev(
        () => [
          new CircularDependencyPlugin({
            exclude: /node_modules/,
            include: /src/,
            failOnError: true,
            allowAsyncCycles: false,
            cwd: process.cwd(),
          }),
        ],
        [],
      ),
      // 新增打包产物分析插件
      ...whenProd(
        () => [
          // https://www.npmjs.com/package/webpack-bundle-analyzer
          new BundleAnalyzerPlugin({
            analyzerMode: 'static', // html 文件方式输出编译分析
            openAnalyzer: false,
            reportFilename: path.resolve(__dirname, `analyzer/index.html`),
          }),
        ],
        [],
      ),
    ],
  }
}
```

### 扩展 react-hot-loader

> CRA v4.0 开始已经内置了 fast refresh —— https://github.com/facebook/create-react-app/releases/tag/v4.0.0 ，不必自己扩展 hot loader。

常用的热更新方案 react-hot-loader，craco 提供了专门的 craco plugin（[传送门](https://github.com/HasanAyan/craco-plugin-react-hot-reload)），配置如下：

```sh
yarn add craco-plugin-react-hot-reload -D
```

```js
/* craco.config.js */
const { whenDev, whenProd, when } = require('@craco/craco')
const reactHotReloadPlugin = require('craco-plugin-react-hot-reload')

module.exports = {
  babel: {
  
  },
  webpack: {
  
  },
  // craco 提供的插件
  plugins: [
    // 新增 craco-plugin-react-hot-reload
  	...whenDev(
      () => [
        {
          plugin: reactHotReloadPlugin,
        },
      ],
      [],
    ),
  ]
}
```

添加完配置，还需在入口文件中修改一下 render。

```sh
yarn add react-hot-loader -D
```

```js
/* 入口 index.ts */

import React from 'react'
import ReactDOM from 'react-dom'
+ import { hot } from 'react-hot-loader'
+ import { isDev } from '@/utils' // isDev 你可以自己编写定义，例如：const isDev = process.env.NODE_ENV === 'development'
import App from './App'

+ const Root = isDev ? hot(module)(App) : App
- ReactDOM.render(<App />, document.getElementById('root'))
+ ReactDOM.render(<Root />, document.getElementById('root'))
```

### 比 react-hot-loader 更好的方案 craco-fast-refresh

> CRA v4.0 开始已经内置了 fast refresh —— https://github.com/facebook/create-react-app/releases/tag/v4.0.0 ，不必自己扩展 hot loader。

这是最近发现的新 craco plugin，相对于 react-hot-loader 好用得多，零配置，不需要修改项目代码，据说性能也更好。

```sh
yarn add craco-fast-refresh -D
```

```js
/* craco.config.js */
const { whenDev } = require('@craco/craco')
const fastRefreshCracoPlugin = require('craco-fast-refresh')

module.exports = {
  babel: {
  
  },
  webpack: {
  
  },
  // craco 提供的插件
  plugins: [
  	...whenDev(
      () => [
        {
          plugin: fastRefreshCracoPlugin,
        },
      ],
      [],
    ),
  ]
}
```

### AntDesign 自定义主题 & 按需加载

- 在 bebel 配置中新增 `babel-plugin-import`，搞定 AntDesign 按需加载
- 新增 craco 提供的 plugin `craco-less`，搞定自定义主题

```sh
yarn add babel-plugin-import craco-less -D
```

```js
/* craco.config.js */
const CracoLessPlugin = require('craco-less')

module.exports = {
  babel: {
    plugins: [
      // 配置 babel-plugin-import
      [
        'import',
        {
          libraryName: 'antd',
          libraryDirectory: 'es',
          style: 'css',
        },
        'antd',
      ],
    ],
  },
  webpack: {
  
  },
  // craco 提供的插件
  plugins: [
    // 配置 less
  	{
      plugin: CracoLessPlugin,
      options: {
        lessLoaderOptions: {
          lessOptions: {
            modifyVars: {
              // 自定义主题（如果有需要，单独文件定义更好一些）
              '@primary-color': '#1DA57A',
            },
            javascriptEnabled: true,
          },
        },
      },
    },
  ]
}
```

craco 也提供了专门的 plugin 来处理 antd 的集成（[传送门](https://github.com/DocSpring/craco-antd)），与上方的配置方式有区别。从 github 的介绍中能够看到如下介绍：

craco-antd includes:

- Less (provided by craco-less)
- babel-plugin-import to only import the required CSS, instead of everything
- An easy way to customize the theme. Set your custom variables in ./antd.customize.less

相对来讲使用会更简洁一些，推荐使用。

## 总结

经过项目中的试用，确实能够在不 eject 弹出配置的情况下，自定义所有的 cra 构建配置，因此，在后续的编码中，我应该还会继续使用。

上方是最近在项目中，使用到的一些配置，更多的使用方式建议通过阅读官方的文档（[传送门](https://github.com/gsoft-inc/craco/blob/master/packages/craco/README.md)）去了解，尤其需要详细了解 craco.config.js 文件的配置列表（[Configuration Overview](https://github.com/gsoft-inc/craco/blob/master/packages/craco/README.md#configuration-overview)）。

有几点建议如下：

1. craco 有提供一些好用的 plugin（[传送门](https://github.com/gsoft-inc/craco#community-maintained-plugins)），推荐优先考虑使用现成的插件去解决问题。

    注意 craco 的 plugin 与 webpack plugin 不是同一概念，二者配置的位置也不同。
    
2. webpack 相关的配置覆盖，优先使用 craco 提供的快捷方式去配置。

    解决不了的问题，在 configure 函数中配置，并且，推荐 configure 使用函数形式，而非对象形式。虽然，函数形式更复杂了一点，但是二者是互斥的，只好选择其中一种。
    
最后，放一份完整的 craco.config.js 配置文件，方便参照。

> 版本是 webpack v4、create-react-app v4，其它不同的版本，可能需要略加调整。

```js
/**
 * Craco 重写 CRA 配置
 *  - GitHub：https://github.com/gsoft-inc/craco
 *  - 配置参数：https://github.com/gsoft-inc/craco/blob/master/packages/craco/README.md#configuration-overview
 *  - 快速指南：https://blog.eleven.net.cn/2020/09/11/cra/craco/
 *
 * Tips：
 *  1、区分 node 运行环境 —— NODE_ENV
 *    - whenDev ☞ process.env.NODE_ENV === 'development'
 *    - whenTest ☞ process.env.NODE_ENV === 'test'
 *    - whenProd ☞ process.env.NODE_ENV === 'production'
 *  2、NODE_ENV 可以区分 node 运行环境，仅支持 development、test、production，
 *    自定义的 REACT_APP_BUILD_ENV 用于区分编译环境，支持 development、test、uat、production。
 *  3、craco 有提供一些好用的 plugin（https://github.com/gsoft-inc/craco#community-maintained-plugins），推荐优先考虑使用现成的 craco plugin 去解决问题。
 *  4、CRA 脚手架相关的配置覆盖，优先使用 craco 提供的快捷方式去配置。解决不了的问题，在 configure 函数中配置。
 *    推荐 configure 配置使用函数形式，而非对象形式。虽然，函数形式更复杂了一点，但是二者是互斥的，只能选择其中一种。
 */

const path = require('path');
const TerserPlugin = require('terser-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const { whenDev, whenProd, when } = require('@craco/craco');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
const CracoLessPlugin = require('craco-less');
const CracoScopedCssPlugin = require('craco-plugin-scoped-css');
const CircularDependencyPlugin = require('circular-dependency-plugin');
const TsconfigPathsPlugin = require('tsconfig-paths-webpack-plugin');
const vConsolePlugin = require('vconsole-webpack-plugin');
const genericNames = require('generic-names');
const antdTheme = require('./antd.theme');

// 判断编译环境是否为生产
const isBuildProd = process.env.REACT_APP_BUILD_ENV === 'production';
// 判断 node 运行环境是否为 production
const isProd = process.env.NODE_ENV === 'production';
const isBuildAnalyzer = process.env.BUILD_ANALYZER === 'true';
const removeFilenameHash = process.env.REMOVE_FILENAME_HASH === 'true';
const shouldDropDebugger = process.env.SHOULD_DROP_DEBUGGER === 'true';
const shouldDropConsole = process.env.SHOULD_DROP_CONSOLE === 'true';
const enableVConsole = process.env.ENABLE_VCONSOLE === 'true';
const localIdentName = '[local]-[hash:base64:5]';

module.exports = {
  /**
   * 扩展 babel 配置
   */
  babel: {
    loaderOptions: {
      /**
       * Babel 编译时，会处理 core-js（未来可能会被修复），
       * 导致 polyfill 内部代码发生了变化，产生一些微小的影响，如 Symbol 问题。
       * 暂时我们手动声明略过。
       * https://github.com/zloirock/core-js/issues/514
       * https://github.com/rails/webpacker/pull/2031
       */
      exclude: [
        /node_modules[\\/]core-js/,
        /node_modules[\\/]react-app-polyfill/,
      ],
    },
    presets: [
      [
        '@babel/preset-env',
        {
          modules: false, // 对 ES6 的模块文件不做转化，以便使用 webpack 支持的 tree shaking、sideEffects
          useBuiltIns: 'entry', // entry ☞ 指定的 browserslist 环境，不支持的特性垫片都导入
          // https://babeljs.io/docs/en/babel-preset-env#usebuiltins
          // https://github.com/zloirock/core-js/blob/master/docs/2019-03-19-core-js-3-babel-and-a-look-into-the-future.md
          corejs: {
            version: 3, // 使用 core-js@3
            proposals: true,
          },
        },
      ],
    ],
    plugins: [
      /**
       * AntDesign 按需加载
       */
      [
        'babel-plugin-import',
        {
          libraryName: 'antd',
          libraryDirectory: 'es',
          style: true,
        },
        'antd',
      ],
      [
        // @babel/plugin-proposal-decorators 需要在 @babel/plugin-proposal-class-properties 之前，保证装饰器先处理
        '@babel/plugin-proposal-decorators',
        {
          legacy: true, // 推荐
        },
      ],
      [
        '@babel/plugin-proposal-class-properties',
        {
          loose: true, // babel 编译时，对 class 的属性采用赋值表达式，而不是 Object.defineProperty（更简洁）
        },
      ],
      /**
       * babel-plugin-react-css-modules
       *  - GitHub: https://github.com/gajus/babel-plugin-react-css-modules
       *  - http://ekoneko.github.io/blog/engineering/stop-using-css-in-js/
       *  - https://zhuanlan.zhihu.com/p/26878157
       */
      [
        'react-css-modules',
        {
          exclude: 'node_modules',
          filetypes: {
            '.scss': {
              syntax: 'postcss-scss',
            },
            '.less': {
              syntax: 'postcss-less',
            },
          },
          // 必须保持和 css-modules 的 localIdentName 一致的命名
          // https://github.com/gajus/babel-plugin-react-css-modules/issues/291
          // generic-names 用于解决 css-loader v4 hash 的兼容
          generateScopedName: genericNames(localIdentName),
          webpackHotModuleReloading: true,
          autoResolveMultipleImports: true,
          handleMissingStyleName: 'warn',
        },
      ],
      /**
       * https://github.com/tleunen/babel-plugin-module-resolver
       *
       * 解决 babel-plugin-react-css-modules 不兼容 webpack alias 问题
       *
       * Support for Webpack resolve aliases
       *  https://github.com/gajus/babel-plugin-react-css-modules/issues/46
       */
      [
        'module-resolver',
        {
          alias: {
            '@': path.resolve(__dirname, 'src'),
          },
        },
      ],
    ],
  },
  style: {
    /**
     * css modules 配置
     */
    modules: {
      // 必须保持和 babel-plugin-react-css-modules 一致的命名
      localIdentName,
    },
  },
  /**
   * 扩展 webpack 相关配置
   */
  webpack: {
    /**
     * 新增 webpack plugin
     */
    plugins: [
      /**
       * 模块间循环依赖检测插件
       *  - https://juejin.im/post/6844904017848434702
       */
      ...whenDev(
        () => [
          new CircularDependencyPlugin({
            exclude: /node_modules/,
            include: /src/,
            failOnError: true,
            allowAsyncCycles: false,
            cwd: process.cwd(),
          }),
        ],
        []
      ),
      /**
       * 编译产物分析
       *  - https://www.npmjs.com/package/webpack-bundle-analyzer
       */
      ...when(isBuildAnalyzer, () => [new BundleAnalyzerPlugin()], []),
      /**
       * vconsole-webpack-plugin
       *  - 生产环境，强制不会生效
       *
       * 必须 entry 为数组插件才能生效，如果不是需自己改写
       * https://github.com/diamont1001/vconsole-webpack-plugin/issues/44
       */
      ...when(
        !isBuildProd,
        () => [
          new vConsolePlugin({
            enable: !isBuildProd && enableVConsole,
          }),
        ],
        []
      ),
    ],
    /**
     * 重写 webpack 任意配置
     *  - configure 能够重写 webpack 相关的所有配置，但是，仍然推荐你优先阅读 craco 提供的快捷配置，把解决不了的配置放到 configure 里解决；
     *  - 这里选择配置为函数，与直接定义 configure 对象方式互斥；
     */
    configure: (webpackConfig, { env, paths }) => {
      /**
       * 改写 entry 为数组，确保 vconsole-webpack-plugin 可以生效
       * https://github.com/diamont1001/vconsole-webpack-plugin/issues/44
       */
      if (isProd && typeof webpackConfig.entry === 'string') {
        webpackConfig.entry = [webpackConfig.entry];
      }

      /**
       * 修改 output
       */
      webpackConfig.output = {
        ...webpackConfig.output,
        ...when(
          isProd,
          () => ({
            // 命名上移除 chunk 字样
            chunkFilename: 'static/js/[name].[contenthash:8].js',
          }),
          {}
        ),
        // 支持移除 js 文件名的 hash 值
        ...when(
          isProd && removeFilenameHash,
          () => ({
            filename: 'static/js/[name].js',
            chunkFilename: 'static/js/[name].js',
          }),
          {}
        ),
      };

      /**
       * 配合私有 source map 文件，修改 map 文件链接时，需设置为 false
       * （仅编译生产代码时修改 map 文件链接）
       */
      webpackConfig.devtool = isBuildProd ? false : webpackConfig.devtool;

      /**
       * 扩展 extensions
       */
      webpackConfig.resolve.extensions = [
        ...webpackConfig.resolve.extensions,
        ...['.scss', '.less'],
      ];

      webpackConfig.resolve.plugins = [
        ...webpackConfig.resolve.plugins,
        ...[
          /**
           * 自动将 tsconfig 里的 paths 注入到 webpack alias
           * （意味着你不再需要额外增加 webpack alias）
           *  - https://github.com/dividab/tsconfig-paths-webpack-plugin
           */
          new TsconfigPathsPlugin({
            extensions: ['.ts', '.tsx', '.js', '.jsx'],
          }),
        ],
      ];

      webpackConfig.optimization.minimizer.map(plugin => {
        /**
         * TerserPlugin
         */
        if (plugin instanceof TerserPlugin) {
          Object.assign(plugin.options.terserOptions.compress, {
            drop_debugger: shouldDropDebugger, // 删除 debugger
            drop_console: shouldDropConsole, // 删除 console
          });
        }

        return plugin;
      });

      /**
       * webpack split chunks
       */
      webpackConfig.optimization.splitChunks = {
        ...webpackConfig.optimization.splitChunks,
        ...{
          chunks: 'all',
          name: true,
        },
      };

      /**
       * 覆盖已经内置的 webpack plugins
       */
      webpackConfig.plugins.map(plugin => {
        /**
         * 支持移除 css 文件名的 hash 值
         */
        whenProd(() => {
          if (plugin instanceof MiniCssExtractPlugin) {
            Object.assign(
              plugin.options,
              {
                // 命名上移除 chunk 字样
                chunkFilename: 'static/css/[name].[contenthash:8].css',
              },
              when(
                removeFilenameHash,
                () => ({
                  filename: 'static/css/[name].css',
                  chunkFilename: 'static/css/[name].css',
                }),
                {}
              )
            );
          }
        });

        return plugin;
      });

      // 返回重写后的新配置
      return webpackConfig;
    },
  },
  /**
   * 新增 craco 提供的 plugin
   */
  plugins: [
    /**
     * 支持 less
     *  - https://github.com/DocSpring/craco-less
     *  - options 参数：https://github.com/DocSpring/craco-less#configuration
     */
    {
      plugin: CracoLessPlugin,
      options: {
        modifyLessRule(lessRule, context) {
          return {
            ...lessRule,
            ...{
              test: /\.less$/,
              exclude: /\.module\.less$/,
            },
          };
        },
        lessLoaderOptions: {
          lessOptions: {
            // 自定义 antd 主题
            modifyVars: antdTheme,
            javascriptEnabled: true,
          },
        },
      },
    },
    /**
     * 支持 less module
     *  - https://github.com/DocSpring/craco-less#configuration
     */
    {
      plugin: CracoLessPlugin,
      options: {
        modifyLessRule(lessRule, context) {
          return {
            ...lessRule,
            ...{
              test: /\.module\.less$/,
              exclude: undefined,
            },
          };
        },
        cssLoaderOptions: {
          modules: {
            // 必须保持和 babel-plugin-react-css-modules 一致的命名
            localIdentName,
          },
        },
      },
    },
    /**
     * react scoped css (only scss/css)
     *  - https://github.com/gaoxiaoliangz/react-scoped-css
     */
    {
      plugin: CracoScopedCssPlugin,
    },
  ],
};
```