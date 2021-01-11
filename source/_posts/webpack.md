---
layout:     post
title:      "基于Webpack项目工程化"
subtitle:   "又是一波整合的配置"
date:       2021-01-11 18:47:00
author:     "Long"
header-img: "post-bg.jpg"
tags:
    - 项目工程化
---

## Webpack

> webpack是一个打包工具，可以打包一切前端相关资源

学习webpack，掌握3个要素

- 理解打包过程
- 熟悉打包配置
- 了解webpack生态

比较其他的打包工具👇

[rollup](https://rollupjs.org/guide/en/)：一般用打包JS库，打包单纯的JS用的比较多，打包其他资源需要借助插件

[parcel](https://parceljs.org/)：0配置打包工具

[gulp](https://gulpjs.com/)：基于流的打包工具，逐渐被替代

### 概念

核心配置：

- entry: 打包入口
- output: 打包输出目录

```js
output: {
  filename: '[name].bundle.[chunkhash].[ext]',
  // 绝对路径  
  path: path.resolve(__dirname, 'dist'),
  // 配置动态加载
  chunkFilename: '[name].bundle.js'
}
```

- module(Loader): 处理各种资源

```js
module: {
  rules: [
    {
      test: /\.jsx$/,
      use: [
        {
          loader: 'babel-loader',
          options: {
            // 支持解析jsx以及tsx
            presets: ["@babel/preset-react", "@babel/preset-typescript"],
            // 缓存编译目录
            cacheDirectory: true
          }
        }
      ]
    },
    {
      // 处理样式
      test: /\.css$/,
      // 执行的顺序 less-loader => css-loader => style-loader
      // ExractTextPlugin 提取css到单独文件,同时配置参数
      // plugins: [
  		//   new ExtractTextPlugin({
      //     filename:  (getPath) => {
      //       return getPath('css/[name].css').replace('css/js', 'css');
      //   	 },
      //     allChunks: true
      //   })
      // ]
      use: ExractTextPlugin.extract({
        fallback: 'style-loader',
        use: [
          {
            // 解析css
            loader: 'css-loader',
            options: {
              // 开启css模块化
              modules: true
            }
          },
          'postcss-loader',
          {
            // 将less转换为css
            loader: 'less-loader',
            options: {
              xxx: xxx
            }
          }
        ]
      })
    },
    {
      // 处理图片
      test: /\.(png|svg|jpg|gif)$/,
      use: [
        'file-loader'
      ]
    },
    {
      // 处理字体文件
      test: /\.(woff|woff2|eot|ttf|otf)$/,
      use: [
        'file-loader'
      ]
    }
  ]
}
```

- plugins: 扩展webpack打包能力

```js
// 常用的插件
plugins: [
  // 打包自动生成 html 入口文件
  new HtmlWebpackPlugin({ title: 'output page' }),
  // 清理打包之后的目录
  new CleanWebpackPlugin(['dist']),
  // 定义环境变量
  new webpack.DefinePlugin({
    'process.env.NODE_ENV': JSON.stringify('development')
  }),
  // 压缩代码
  new UglifyJSPlugin()
]
```

### 开发相关

- mode: 打包模式，用于webpack内部，外界只需要传`development | production`
- devtool: 配置`source map`

> source map的作用是将编译后的代码映射回原始源代码

- devserver: 配置`webpack-dev-server`，官方提供的开发服务器
  - hot: `true | false` 是否启动热加载
  - proxy: 解决跨域

> 热加载的作用是局部更新代码，如果我们没有使用框架，则需要自己实现热加载逻辑

- resolver: 解决文件识别问题
  - extensions: `['js', 'jsx']` 自动识别文件后缀，引入时可不加后缀
  - alias: `{ 'src': path.resolve(__dirname, "src/") }` 路径别名

### 优化相关

- 分析工具
  - [webpack-chart](https://alexkuz.github.io/webpack-chart/): webpack 数据交互饼图
  - [webpack-visualizer](https://chrisbateman.github.io/webpack-visualizer/): 可视化并分析你的 bundle，检查哪些模块占用空间，哪些可能是重复使用的
  - [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer): 一款分析 bundle 内容的插件及 CLI 工具，以便捷的、交互式、可缩放的树状图形式展现给用户

- `mode: production`，默认开始`tree shaking`

> tree shaking的作用打包中剔除没有使用过的代码

- 动态加载/懒加载：采用ES6`import`语法，`import('path/to/module') -> Promise`，不会影响总打包的大小，提升代码执行速度
- 浏览器缓存

```js
entry: {
  main: './src/index.js',
  vendor: ['lodash', 'react']
}
plugins: [
  // 确保hash值代表一个bundle
  new webpack.HashedModuleIdsPlugin(),
  // 抽取第三方模块
  new webpack.optimize.CommonsChunkPlugin({
     name: 'vendor'
  }),
  // 抽取webpack执行上下文
  new webpack.optimize.CommonsChunkPlugin({
     name: 'manifest'
  })
]
```

- webpack深度优化

```js
optimization: {
    minimizer: [
      new TerserWebpackPlugin({
        terserOptions: {
          warnings: false,
          compress: {
            warnings: false,
            // 是否注释掉console
            drop_console: false,
            dead_code: true,
            drop_debugger: true,
          },
          output: {
            comments: false,
            beautify: false,
          },
          // 混淆
          mangle: true,
        },
        // 多线程打包
        parallel: true,
        // 是否开启source map
        sourceMap: false,
      })
    ],
    splitChunks: {
      cacheGroups: {
        commons: {
          name: 'commons',
          chunks: 'initial',
          minChunks: 3,
          enforce: true
        }
      }
    }
  }
```

### 环境隔离

一般使用`webpack-merge`第三库，合并配置并输出

```js
const webpackMerge = require('webpack-merge')

const baseWebpackConfig = require('./webpack.config.base')

const webpackConfig = webpackMerge(baseWebpackConfig, {
  mode: 'development',
  devtool: 'eval-source-map',
  stats: { children: false }
})

module.exports = webpackConfig
```

### 流行脚手架特性

- 具备开发服务器 [webpack-dev-server](https://github.com/webpack/webpack-dev-server)
- 打包HTML，CSS，JS，图片，字体文件
- 整合Eslint（检查代码风格），Typescript（静态编译检查），Babel（语法降级，优化编译）
- 生产构建优化
- 开发环境打包速度优化





