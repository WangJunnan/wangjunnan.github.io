---
title: React + ES6 + webpack 快速搭建开发环境
tags:
  - javascript
  - react
categories:
  - 前端技术
  - javascript
date: 2017-02-27 17:09:32
---

近期，由用到了react.js来写前端代码，搭建开发环境对于从来没有写过react的同学来说有点复杂，这里简单记录一下搭建过程（本人也是前端渣渣，写错的地方还望大家指出）

## 安装webpack
>需要Node 环境

### 全局安装

>cnpm install -g webpack
 

### 本项目安装

>cnpm install webpack --save-dev

### 安装 react + es6 loaders

>cnpm install --save-dev babel-core babel-loader babel-preset-es2015 babel-preset-react

还有css loader

>cnpm install css-loader style-loader --save-dev

## 配置webpack.config.js 文件

```
var path = require('path');

module.exports = {
  entry: './app/src/main.jsx',
  output: {
        path: path.resolve(__dirname, './app/web'), // 输出文件的保存路径
        filename: 'bundle.js' // 输出文件的名称
    },

    module: {
    loaders: [{
      test: /\\.jsx?$/,
      loaders: 'babel-loader',
      query: {
          presets: ['react', 'es2015']
        }
      },
    {
      test: /\\.css$/,
      loader: 'style-loader!css-loader'
    }
    ]
}
};
```

到这里基本开发环境搭建好了

## 安装react 和react-dom

>cnpm i react-dom --save
>cnpm i react --save

安装完之后会发现package.json下多了这两个依赖项

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-57ba87cb70990aeb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在app/src 下新建一个main.jsx入口文件，简单写下内容

```
import React from 'react';
import ReactDOM from 'react-dom';

ReactDOM.render(
  <h1>Hello, world!</h1>,
  document.getElementById('example')
);
```

然后在app/web下新建index.html ，里面需要引用bundle.js文件

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>webpack 教程</title>
</head>
<body>
	<div id="example"></div>
</body>
<script src="bundle.js"></script>
</html>
```

完成搭建
执行命令 
> webpack

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-e8fc05482a48d53f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行成功，发现在app/web 目录下多了个bundle.js文件

在浏览器打开index.html 文件。正确显示hello world!

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-067e6f0e936120a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***
>cnpm install 【name】--save-dev 与 cnpm install 【name】--save的区别，都是安装一些模块，唯一的区别是加有dev的是在开发时所依赖的模块，而不加的是运行时需要的基本模块。

### 那么怎么实时调试呢？

> cnpm install --save-dev webpack-dev-server react-hot-loader
webpack.config.js 的内容更新如下

```
var path = require('path');
var webpack = require('webpack');

module.exports = {
  entry: {
    index: [
    './app/src/main.jsx'
  ]
  },
  output: {
        path: path.resolve(__dirname, './app/web'), // 输出文件的保存路径
        filename: 'bundle.js' // 输出文件的名称
    },

  module: {
    loaders: [{
      test: /\\.jsx?$/,
      loaders: 'babel-loader',
      query: {
          presets: ['react', 'es2015']
        }
      },
    {
      test: /\\.css$/,
      loader: 'style-loader!css-loader'
    },
    {
      test: /\\.js?$/,
      loaders: ['react-hot'],
      include: [path.join(__dirname, 'app/src')]
    }
    ]
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NoErrorsPlugin(),
    new webpack.LoaderOptionsPlugin({
    options: {
      postcss: function () {
        return [precss, autoprefixer];
      },
      devServer: {
          historyApiFallback:true,
          hot:true,
          inline:true,
          progress:true
      }
    }
  })
]

};
```

package.json 文件需要添加以下


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-9923bd6a5b7a5fd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行
> npm run dev 

访问 localhost:8080 即可

写的很粗浅 惭愧，以后有时间会重新研究一下前端的构建及打包
