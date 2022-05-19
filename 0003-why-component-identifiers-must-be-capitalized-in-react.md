---
title: '为什么 React 组件 ID 需要大写?'
date: '2022-05-19'
tags: 'React,component'
---

[原文在此](https://indepth.dev/posts/1499/why-component-identifiers-must-be-capitalized-in-react)

此文中，我们讨论下为什么组件需要大写成 `<Foo />`，而不是 `<foo />`。如果我们 `import foo from './Foo'` ，以 `<foo />`  的方式使用，`React` 会告警：

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0003/react-warning.png)

具体效果，猛击此[链接](https://codesandbox.io/s/busy-proskuriakova-npp9gj?file=/src/App.js)

也就是说，`<foo />` 被当做 `DOM` 元素了，但 `foo` 元素并不存在，所以报错。虽然修正此问题很容易，但为什么 `<Foo />` 被认做组件而 `<foo />` 被认做 `DOM` 元素呢？

接下来我们简要讨论 `Babel` 以及这个问题的答案。

# `Babel` 工作简介

`Babel` 通过一系列插件将 `JSX` 转换成浏览器能执行的代码。我们先看 [@babel/plugin-transform-react-jsx](https://github.com/babel/babel/tree/main/packages/babel-plugin-transform-react-jsx) 。

编译的部分过程如下：

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0003/compile-process.png)

`Babel` 将传入的源代码以字符串的形式构建成抽象语法树，然后应用各种各样的插件。如上文提及的 `transform-react-jsx`。

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0003/transformFile.png)

如上图所示，`transformFile` 函数指代编译图中的第二阶段，此时所有选中的插件都将作用于原始抽象语法树。

当访问到某些特定的抽象语法树节点时，`transform-react-jsx` 的某些函数将调用。比如：下图中访问到 `JSXElement` 时，

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0003/exit.png)

> `exit` 函数可被看做 `访问钩子`。辅以 `enter` 函数，此函数于节点将要访问时调用。`enter` 调用完后，将继续访问当前节点的孩子节点，然后再执行 `exit` 函数。也就是 `Node.enter()` → visited children → `Node.exit()`。

上图中，我们在 `<Foo />` 中。我们可以看到，`JSXElement` 中有一个包含名称的 `JSXOpeningElement`。就是在这一步中来决定 `Foo` 到底指向一个组件还是一个 `DOM` 元素。这个决定必不可少，因为在 `JSX` 中既可以使用 `<div></div>`，也可以使用 `<CustomComponent></CustomComponent>`。在某个时间节点，`Babel` 必须选择 `JSXElement` 指代什么，因此 `React` 才能做出对应的响应。

调用完 `JSXElement.exit()` 将来到 `getTag` 函数，此处便能找到开题问题的答案。

![](https://cisolarix-com-1259454290.cos.ap-singapore.myqcloud.com/0003/getTag.png)

如果 `tagName` 以小写字母开头，`getTag` 函数将返回一个字符串字面量。如果以 `<Foo />` 的方式调用，则返回如下结果：

```js
{
  type: "Identifier",
  start: 165,
  end: 168,
  name: "Foo",
  /* ... */
}
```

如果以 `<foo />` 的方式调用，则返回：

```js
{
  type: "StringLiteral",
  value: 'foo',
}
```

`Identifier` 和 `StringLiteral` 的本质区别是前者会调用 `createElement(componentName)` 后者调用 `createElement('componentName')`【注意单引号】。

`React` 根据不同的元素类型选择不同的路径。



# 结论

`isCompatTag` 函数就是问题的核心。



译注：本文仅意译部分内容。
