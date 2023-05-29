---
sidebar: auto
---

# 脚手架工具升级

## 背景
* 2年未更新，一些新的特性无法使用
* 依赖的基础工具比较陈旧
* webpack5在性能上有提升，减少一些繁琐配置
* 依赖的组件库需要升级
* 模板仓库需要更新

## 升级实践

### 依赖库调研
* 首先，一定要确保打包机的环境，需要兼容打包机，主要是 Node.js 的版本。否则还需要对打包机进行升级，这个成本较高，需要同时兼容webapck4 和 5。所幸，我们打包机的 Node.js 为 12.0.0，因此评估下来，是可以进行webpack升级的。
* 在调研过程中，webpack5 构建中，用到的 loader, plugin，都需要做相应的版本调研，需要同时满足支持 webpack5 和 Node.js 12 版本。

### npm 包相关的依赖版本升级
* npm 包升级原则：
  - 需要兼容现有的功能
  - 使用官方推荐的版本或者更适合的 npm 包
  - 在现有的开发环境下，升级到可用的最高版本

#### webpack 相关
* "webpack": "^4.46.0" => "5.82.0"
* "webpack-merge": "^4.2.1" => "5.8.0"
* "webpack-dev-server": "^4.7.4" => 无需升级

#### loader 相关
* "babel-loader": "^8.2.2" => 无需升级
* "file-loader": "^4.2.0" => "5.1.0"
* "less-loader": "^5.0.0" => "10.2.0"
* "postcss-loader": "^2.0.6" => "6.2.1"
* "sass-loader": "^6.0.7" => "12.6.0"
* "style-loader": "^0.18.2" => "3.3.2"
* "ts-loader": "^7.0.3" => "9.4.2"
* "typescript": "^3.8.3" => "4.9.5"
* "url-loader": "^2.1.0" => 无需升级 
* "happypack" 替换为 "thread-loader"
  - "happypack" webpack官方已不推荐使用，而且 "happypack" 已经不再维护了，推荐使用 "thread-loader".

#### plugin 相关
* "html-webpack-plugin": "^4.5.2" => "5.5.1"
* "mini-css-extract-plugin": "^0.8.0" => "2.7.5"
* "optimize-css-assets-webpack-plugin": "^5.0.3"
  - 替换为 "css-minimizer-webpack-plugin": "3.4.1"
* "add-asset-html-webpack-plugin": "^3.2.2" => "5.0.2"
* 新增 "terser-webpack-plugin": "5.3.8"，代码压缩。
* 新增 "@soda/friendly-errors-webpack-plugin": "1.8.1"，日志打印

#### 其他
* "react-dev-utils": "^11.0.3" => "12.0.0"
  - 解决报错问题：`message.split is not a function`
* "ora": "^3.0.0" => "6.3.0"
* "chalk": "^2.4.1" => "4.1.2"
* "commander": "^2.17.1" => "2.20.1"
  - 大版本升级存在 breaking changes，暂不升级。

### 构建优化
* 去掉了冗余的配置项，优化部分loader配置
* 增加 `CSS module` 的配置
* webpack5配置 api 修改

#### devServer API 升级
* listen 修改为 startCallback
* 原有代码
```js
devServer.listen(port, HOST, err => {
  if (err) {
    return console.log(err);
  }
  console.log(chalk.cyan('正在启动服务...\n'));
  openBrowser(urls.localUrlForBrowser);
});
```
* 修改后的代码
```js
devServer.startCallback(() => {
  console.log(chalk.cyan('正在启动服务...\n'));
  openBrowser(urls.localUrlForBrowser);
});
```

#### 缓存配置
* 增加配置
```js
{
  cache: {
    // 持久化缓存，改善构建速度，将境编译结果写入硬盘缓存（node_modules/.cache/webpack）
    type: "filesystem",
  },
}
```

#### `polyfill` 优化
* 去掉不再使用的 polyfill 导入。使用 "@babel/preset-env" 和 "core-js" 来配置自动 polyfill。
* 增加 Node.js 的 polyfill 配置，webpack5 不再默认导入 polyfill ，代码中使用到的 Node.js API 需要手动配置。
* 

#### `url-loader`配置优化
* 官方不再推荐使用`url-loader`来处理文件，已经内置。
* 原有配置
```js
{
  test: [/\.bmp$/, /\.gif$/, /\.jpe?g$/, /\.png$/],
  loader: require.resolve("url-loader"),
  options: {
    limit: 10000,
    name: packageJson.name + "/media/[name].[hash:8].[ext]",
  },
},
```
* 修改后的配置
```js
{
  test: [/\.bmp$/, /\.gif$/, /\.jpe?g$/, /\.png$/],
  include: [paths.appSrc],
  type: 'asset/resource'
},
```

#### `sass-loader` 配置修改
* 原有配置
```js
{
  test: /\.(c|sc|sa)ss$/,
  use: [
    require.resolve('style-loader'),
    {
      loader: require.resolve('css-loader'),
      options: {
        importLoaders: 1
      }
    },
    {
      loader: require.resolve('postcss-loader'),
      options: {
        ident: 'postcss',
      }
    },
    require.resolve('sass-loader')
  ]
},
```

* 修改后的配置，原有的API不兼容
```js
{
  test: /(\.module)?\.(c|sc|sa)ss$/,
  use: [
    require.resolve('style-loader'),
    {
      loader: require.resolve('css-loader'),
      options: {
        importLoaders: 1,
        modules: {
          localIdentName: '[name]---[local]---[hash:base64:5]',
          auto: (resourcePath) => (/\.module\.(sc|sa)ss$/).test(resourcePath),
        }
      },
    },
    {
      loader: require.resolve('postcss-loader'),
      options: {
        postcssOptions: {
          plugins: [
            [
              "autoprefixer",
              {
                // Options
              },
            ],
          ],
        },
      },
    },
    require.resolve('sass-loader'),
  ] 
},
```

#### `happypack` 修改为 `thread-loader`
* 去掉 `happypack` 的配置，官方不推荐，也不再维护。推荐使用`thread-loader`
* `thread-loader` 会默认开启 CPU 支持的最多核数，使用默认配置即可。
```js
{
  test: /\.(js|jsx)$/,
  use: [
    {
      loader: require.resolve('thread-loader'),
    },
    {
      loader: require.resolve('babel-loader'),
      options: {
        plugins: [
          [
            require.resolve("@babel/plugin-proposal-decorators"),
            { legacy: true },
          ],
          require.resolve("@babel/plugin-transform-runtime"),
          require.resolve("@babel/plugin-proposal-optional-chaining")
        ],
        presets: [
          [
            require.resolve("babel-preset-react-app"),
            { absoluteRuntime: false },
          ],
        ],
      } 
    }
  ]
},
```

#### 去掉 node 相关配置
* webpack5 已不再支持。
```js
{
  node: {
    dgram: "empty",
    fs: "empty",
    net: "empty",
    tls: "empty",
    child_process: "empty",
  },
}
```

#### 增加性能优化配置
```js
{
  optimization: {
    minimize: true,
    minimizer: [new TerserPlugin(), new CssMinimizerPlugin()],
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // vendor: 打包node_modules中的文件（上面的 lodash）
        vendor: {
          test: /node_modules/,
          chunks: "all",
          priority: 10
        }
      },
    },
  },
}
```

## 测试与发布
* 先在本地进行开发和测试，测试ok后，发布 alpha 版本，然后在 CD 流水线中进行部署测试。
* alpha 版本测试功能 OK 后，发布 beta 版本，让团队同学试用，结合反馈进行优化。
* 最终发布线上版本。

### 本地开发测试
* 脚手架功能开发完成后，比如命名为 `blackey-demo-cli`，
* link 到全局，进入 `blackey-demo-cli` 目录，执行 `npm link`，这样就把本地项目 link 到全局环境。
* 进入使用脚手架的项目目录，比如命名为 `test-project`,
* 进入 `test-project` 目录，安装脚手架项目 `npm i @scope/blackey-demo-cli`，其中`@scope`为公司私服前缀。这样，就把脚手架项目安装完成。可按照之前的方式，在本地进行 `dev` 或 `build`

### 发布 alpha 或 beta 版本
* 修改项目版本，`npm version 1.0.0-alpha.0`
* 发布 `alpha` 版本： `npm publish --tag alpha`
* 要发布 `beta` 版本，跟上述命令类似，`alpha` 替换为 `beta`

### 发布线上
* 打正式的tag，`npm version 1.0.0`
* 发布，`npm publish`

## 注意事项
* 发布版本的时候，尽量使用私服。
* 一定要跟运维同学确认好打包机的环境配置
* 脚手架的使用方式，尽量在项目中进行集成，而非全局安装，这样升级和测试比较灵活。
* 如果有内置的其他二方包依赖，尽量统一版本号，并使用精确的版本，可以减少 npm 包的安装数。
* 二方包和三方包，推荐精确的版本安装，可以减少 `package-lock` 文件的冲突概率。