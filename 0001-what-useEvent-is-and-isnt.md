---
title: 'React useEvent 是什么？'
date: '2022-05-16'
tags: 'React,hooks,useEvent'
---

[原文在此](https://typeofnan.dev/what-the-useevent-react-hook-is-and-isnt/)
[useEvent RFC 在此](https://github.com/reactjs/rfcs/blob/useevent/text/0000-useevent.md)

上周，`React` 核心团队在一则 `[RFC](https://github.com/reactjs/rfcs/blob/useevent/text/0000-useevent.md)` 中引出了 `useEvent`。此文将阐述 `useEvent`是什么以及我对它的第一印象。

注意：目前 `useEvent` 还处在 `RFC` 状态，并未正式发布，且具体的功能可能有变。

## 试着解决一个实际问题

`useEvent`试图解决一个实际问题。在深入了解之前，我们先看看这个问题是什么。

`React`的执行模型主要由比较前后值的不同来驱动。在组件中如此，`hooks` 中亦如此。

考虑下面的组件：

```js
function MyApp() {
  const [count, setCount] = useState(0);

  return <Counter count={count} />;
}
```

如果 `count`值发生变化，`Counter`组件将重新渲染。如果期望 `count` 变化时，某种副作用跟着执行的话，我们可以用 `useEffect`：

```js
function MyApp() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log(count);
  }, [count]);

  return <Counter count={count} />;
}
```

`count`已经在 `useEffect` 的依赖数组中了，每当 `count`变化时，此副作用都会再次运行。

## 这样会有什么问题？

这样写的话，许多开发者会遇到同一个问题：太多的组件或 `hook` 重渲染，有时甚至死循环。

`RFC`举了很多不错的例子，此处将引用。设想一个聊天应用，组件内有一个 `text` 状态，另外一个组件内有一个 `SendMessage` 按钮：

```js
function Chat() {
  const [text, setText] = useState('');

  const onClick = () => {
    sendMessage(text);
  };

  return <SendButton onClick={onClick} />;
}
```

问题是，每当 `text` 发生变化，`onClick` 函数将被重新创建。也就是说每次敲击键盘都会导致 `SendButton` 组件重新渲染。

我们现在来思考一个 `useEffect`太过频繁的例子。此例取自 [`Dan Abravov` 的推文](https://twitter.com/dan_abramov/status/1522221179813179392)。例子中，当 `route.url`变化的时候，副作用将打印一条页面访问的日志。

```js
function Page({ route, currentUser }) {
  useEffect(() => {
    logAnalytics('visit_page', route.url, currentUser.name);
  }, [route.url, currentUser.name]);
}
```

然而，当用户的姓名发生变化的时候，也会打印一条页面访问日志。这不是我们想要的。虽然我可以将 `currentUser.name` 从依赖数组中移除，但这样做不妥。因为如果不将副作用函数体中用到的所有依赖明列在依赖数组中的话，将会导致过期闭包问题。`React` 核心团队强烈推荐 `react-hooks`插件的 `exhaustive deps`规则，就是为了提示这个问题。

## 提案：`useEvent`

`useEvent`可以确保函数的稳定索引，而不是根据依赖项重新创建一个新的函数。

我们回到聊天应用的例子。用 `useEvent`将点击回调函数包裹，这样即使闭包中的 `text` 发生变化，函数的引用也不会变化。

```js
function Chat() {
  const [text, setText] = useState('');

  const onClick = useEvent(() => {
    sendMessage(text);
  });

  return <SendButton onClick={onClick} />;
}
```

`onClick`在组件多次渲染时，也将指向同一个函数，紧接着 `SendButton` 也就不会重新渲染。

再回到页面访问的例子：

```js
function Page({ route, currentUser }) {
  const logVisit = useEvent((pageUrl) => {
    logAnalytics('visit_page', pageUrl, currentUser.name);
  });

  useEffect(() => {
    logVisit(route.url);
  }, [route.url]);
}
```

这下我们创建了稳定的 `logVisit` 函数，就可以将 `currentUser.name` 从 `useEffect` 的函数体中去除，从而让副作用仅依赖 `route.url`。

## 最初印象

我的最初印象是：啊？这玩意儿是个啥？如果这玩意儿早点出来，可能会节省我很多与副作用依赖搏斗的头痛时刻。总之，这是对 `React` 生态的巨大补充。

## `useEvent` 不是什么？

`useEffect` 不是银弹，这自不必说。它给 `React` 又增加了一条新的概念，也没能改变 `React` 不是真正响应式的现实。我简单提一嘴 `SolidJS`。它是真正响应式的，看看用了 `SolidJS` 后，页面访问组件将变得多简单。

```js
function Page(props) {
  createEffect(() => {
    logAnalytics(
      'visit_page',
      props.route.url,
      untrack(() => props.currentUser.name)
    );
  });
}
```

免责声明：我偏袒 `SolidJS`，因为我在 `SolidJS` 文档组。但我依然觉得这更简单。副作用的两个依赖 (n `props.route.url` 和  `props.currentUser.name`)是自动追踪的。如果要取消追踪的话，得使用 `untrack` 函数。上面的代码将只在 `props.route.url` 变化时触发，但可以获取到包含用户名的当前闭包。
