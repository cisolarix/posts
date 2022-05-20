---
title: '深入介绍 Webpack SplitChunksPlugin 插件'
date: '2022-05-19'
tags: 'Webpack,plugin,optimization'
---

[原文在此](https://indepth.dev/posts/1490/webpack-an-in-depth-introduction-to-splitchunksplugin)

这个插件解决的是代码重复问题。根据配置，此插件能将重复加载的代码分割到单独的块。这样，大的模块只需要加载一次。

## 初始

以下面的代码结构开始：[链接在此](https://stackblitz.com/edit/node-qaj9rk?file=readme.md)

```js
├── a.js
├── b.js
├── c.js
├── d.js
├── e.js
├── f.js
├── g.js
├── index.js
├── node_modules
│   ├── x.js
│   ├── y.js
│   └── z.js
└── webpack.config.js
```

`webpack.config.js` 配置如下：

```js
{
  mode: 'production',
  entry: {
    main: './src',
  },
  output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name].js',
    clean: true,
  },
  optimization: {
    minimize: false, // 让 webpack 不要混淆最终代码
    splitChunks: false,
  },
  context: __dirname,
}
```

运行 `npm run build` 后，`dist` 的样子将如图所示：

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0004/output-dir-before-first-diagram.png)

供生成了 5 个块，`main.js` 由配置中的 `entry` 指定，剩下的 4 个由导入函数生成。

下图描述了这个关系：

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0004/file-structure.png)

图中黑色箭头表示导入的 `ESM` 模块，紫色箭头表示函数导入，绿框表示块，内嵌绿框表示块的父子关系。

## 引入问题

`main` 块包含许多 `webpack` 运行时代码。而其他四个文件却只有少量的运行时代码。

问：这五个文件中，`x`模块被拷贝了几次？将 `x.js` 内的文件内容复制后进行全局搜索，我们可以发现，`x`模块被复制了 3 次。

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0004/nr-times-x.png)

设想一下，如果 `x.js` 文件很大，那么这样的复制将导致巨大的资源浪费，虽然这样做并不会导致 `x`多次执行。最终应用中将只有一个 `x.js` 的实例。

回想一下，获取一个块意味着一次 `HTTP` 请求。那么 `x.js` 的文件内容将被下载三次，如果 `x.js` 包含的内容很多的话，这将是个糟糕的方案。

注：`f, d, y` 也有同样的问题，不过为了简便起见，我们仅以 `x` 为例。

## `SplitChunksPlugin` 是如何解决这个问题的

如果将 `x.js` 作为块独立出来，由于缓存的存在，除了第一次真正发送 `HTTP` 请求，后续都将从缓存中获取。

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0004/x-in-separate-chunk_-1-.png)

`Webpack` 中启用 `SplitChunksPlugin` 的配置：

```js
{
  mode: 'production',
  entry: {
    main: './src',
  },
  output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name].js',
    clean: true,
  },
  optimization: {
    minimize: false, // 让 webpack 不要混淆最终代码
    splitChunks: {
      minSize: 0,
    },
  },
  context: __dirname,
}
```

块创建需要的条件由缓存组 `cacheGroup` 决定。

## `cacheGroup` 选项

`cacheGroup` 就一系列规则，用来定义新块该如何创建。举例：

```js
cacheGroupFoo: {
  minChunks: 3, // 模块必须出现在块中的次数
  minSize: 10, // 块的总字节数
  modulePathPattern: /node_modules/ // 将哪些模块考虑在内
}
```

以上条件需同时满足。这就是个缓存组。符合这个条件的模块可能不止一个。比如 `x` 和 `w` 模块都同时满足上述条件，那么要分两种情况：

- 如果二者满足的条件雷同，则它俩会被分入同一个块
- 如果二者满足的条件不同，则它俩各自生成各自的块

如果 `SplitChunksPlugin` 使用了默认选项，更确切地说，没有明确指定缓存组，则将使用两个默认的缓存组 `default` 和 `defaultVendors`。

```js
/* ... */
optimization: {
  // splitChunks: false,
  splitChunks: {
    minSize: 0,
  },
},
/* ... */
```

等同于：

```js
/* ... */
optimization: {
  splitChunks: {
  	default: {
      idHint: "",
      reuseExistingChunk: true,
      minChunks: 2,
      priority: -20
    },
    defaultVendors: {
      idHint: "vendors",
      reuseExistingChunk: true,
      test: NODE_MODULES_REGEXP,
      priority: -10
    }
  },
},
/* ... */
```

`default` 组命中应用中的任意模块。`defaultVendors` 命中所有在 `node_modules` 中的模块。如您所见，示例中 `defaultVendors` 的优先级更高。一个模块只会命中优先级最高的缓存组。