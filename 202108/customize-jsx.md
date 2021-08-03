# 增强 JSX

JSX 是 React 推荐使用的一种用于描述 UI 的语法。在经过 babel 等工具的处理之后，JSX 会被转换成为普通的 JavaScript 代码。正如 React 官方文档所说：

> Fundamentally, JSX just provides syntactic sugar for the `React.createElement(component, props, ...children)` function.

也就是说 JSX 并不是编写 React 代码所必须的，任何的 JSX 代码都可以改写成普通的 JavaScript 代码。但是使用`React.createElement`的代码会非常冗长，非常影响可阅读性。而 JSX 可以大大简化代码，提供开发效率。需要注意的是，由于 JSX 的出生，很多人会以为 JSX 是和 React 绑定的。而事实上，JSX 是通用的 Template DSL。我们完全可以在 Vue 项目中使用 JSX，只不过此时转换出的代码不再是`React.createElement`的函数调用，而是`h`。甚至，[不使用 Virtual DOM 技术的库](https://www.solidjs.com)也可以使用 JSX 来描述 UI。

在有了以上基础之后，我们回到文章的标题。对于使用了 Virtual DOM 技术的库来说，增强 JSX 其实就是增强创建 VDOM 节点的函数。由于创建 Virtual DOM 节点的函数一般被命名为`h`，下文将统一使用`h`而不是`React.createElement`。下面将通过一个简单的例子来介绍一下增强 JSX 大致是如何做的。

上面提到 Vue 也可以使用 JSX，但是由于 JSX 的灵活性，Vue 中的 JSX 所支持的有些特性在 React 中是不支持的。比如说，Vue 的 JSX 支持 class prop。相比起 React 中只能接受字符串的`className`，`class`支持更多的形式，比如数组、对象等等。当然 React 社区也有库能够实现类似的功能，比如`classnames`和`clsx`等，从而将开发者从拼接类名字符串的工作中解脱出来。但是，这也意味着写出来的标签会类似`<div className={clsx(["a", "b"])}></div>`。显然，相比起 Vue 中的实现，这样的代码稍微看上去稍微有一点乱，而且这也会导致我们需要在很多文件中都引入`clsx`库。之前也提到，JSX 支持什么样的特性是由`h`决定的，所以我们想要为 JSX 增加功能的话只需要增强`h`就可以了。不过，增强`h`比较 low level，是否值得这么做还是需要仔细衡量的。

那接下来，我们就尝试来实现对`h`的增强。在开始之前，我们先来写几个测试。

```jsx
import React from "react";
import { render, screen } from "@testing-library/react";
import "@testing-library/jest-dom";

test("class accepts an array", () => {
  render(<div class={["a", "b"]}>Hello World</div>);

  expect(screen.queryByText("Hello World")?.className).toBe("a b");
});

test("class accepts an object", () => {
  render(<div class={{ a: false, b: true }}>Hello World</div>);

  expect(screen.queryByText("Hello World")?.className).toBe("b");
});

test("class appends to className", () => {
  render(
    <div class={["b", "c"]} className="a">
      Hello World
    </div>
  );

  expect(screen.queryByText("Hello World")?.className).toBe("a b c");
});
```

如果这个测试是使用 TypeScript 编写的，你就会发现 TypeScript 编译器会对 class prop 报错。报错信息大致为 Property 'class' does not exist on type '`DetailedHTMLProps<HTMLAttributes<HTMLDivElement>, HTMLDivElement>`'。因此我们最后还需要对一些 TS 类型定义进行修改，从而解决这个问题。

在增强`createElement`之前，我们需要知道`createElement`是一个什么样的函数。它的第一个参数`type`接受各种类型的值。创建 intrinsic elements 时，`type`是字符串，比如`div`、`span`。而为组件创建 element 时，`type`是组件的定义，可能是 class 或者函数。第二个参数 `props` 是传给组件的 props，也就是我们的重点关注。在这个简单的例子中，我们只需要对 props 中包含的 className 值进行一些处理，就能实现想要的效果了。剩余的参数则构成组件的 children。

代码的大致框架如下：

```js
import * as React from "react";
import clsx from "clsx";

const createElement = (type, props, ...children) => {
  if (props == null || !props.hasOwnProperty("class")) {
    return React.createElement(type, props, ...children);
  }

  // ...
};
```

如果传入的 props 中不含有`className`，那我们不需要做任何处理，直接调用`React.createElement`创建 element 就可以了。如果含有`className`，则将该值交由 clsx 处理。然后我们使用返回的 className 创建新 props。

```js
const className = clsx(props["className"], props["class"]);

const newProps = { className };

for (let key in props) {
  if (
    hasOwnProperty.call(props, key) &&
    key !== "class" &&
    key !== "className"
  ) {
    newProps[key] = props[key];
  }
}
```

最后，使用新 props 创建 element：

```js
React.createElement(type, newProps, ...children);
```

之后，我们来为这个函数增加 TS 类型定义。创建内容如下的`.d.ts`文件：

```ts
import * as React from "react";

declare const createElement: typeof React.createElement;

export default createElement;
```

于是，我们就可以使用这个函数来创建 element 了。以下的测试用例已经可以通过。

```js
test("class accepts an array", () => {
  render(createElement("div", { class: ["a", "b"] }, "Hello World"));

  expect(screen.queryByText(/Hello World/)?.className).toBe("a b");
});
```

只是此时，我们仍不能在 JSX 中使用 class prop。为此，我们需要更加具体的了解 babel 是如何将 jsx 转换成为普通的 JS 语句的。[`@babel/plugin-transform-react-jsx`](https://babeljs.io/docs/en/babel-plugin-transform-react-jsx) 是一个包含于`@babel/preset-react`中的负责 jsx 转换的插件。这个插件有两种 React Runtime 可以选择。其中一种是 React Classis Runtime。在这种情况下，

```jsx
<div className="red">Hello World</div>
```

会被转换成为

```js
"use strict";

/*#__PURE__*/
React.createElement(
  "div",
  {
    className: "red",
  },
  "Hello World"
);
```

这也就是我们在使用 JSX 的文件中需要能够访问到 `React` 这个变量的原因。 通过设置 `pragma` Option，我们能够：

> Replace the function used when compiling JSX expressions。

而在 React Automatic Runtime 下，这个 import 的过程将自动完成。观察在该 runtime 下生成的代码有什么不同：

```js
"use strict";

var _jsxRuntime = require("react/jsx-runtime");

/*#__PURE__*/
(0, _jsxRuntime.jsx)("div", {
  className: "red",
  children: "Hello World",
});
```

可以看到，在生成的代码中 element 的创建工作是交给了 `jsx` 函数来完成。而且，代码中已经包含有引入 `jsx` 的代码。这也就以为着，`React` 这个变量不再需要在 scope 中。我们还可以发现 `jsx` 的函数签名和`React.createElement`的也有一些区别。通过设置 `importSource` Option，我们能够：

> Replaces the import source when importing functions.

一个配置的例子：

```json
{
  "presets": [
    [
      "@babel/preset-react",
      {
        "runtime": "automatic",
        "importSource": "preact"
      }
    ]
  ]
}
```

关于这两种 JSX transform，[React 官方文档](https://reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)有更为详细的叙述。

为了兼容这两种 transform，我们需要增加两个名为 `jsx-runtime.js` 以及 `jsx-dev-runtime.js`的文件。

```js
// jsx-runtime.js
import { mergeClassProp, hasOwnProperty } from "./utils";
import * as ReactJSXRuntime from "react/jsx-runtime";

export function jsx(type, props, key) {
  if (!hasOwnProperty.call(props, "class")) {
    return ReactJSXRuntime.jsx(type, props, key);
  }

  const newProps = mergeClassProp(props);

  return ReactJSXRuntime.jsx(type, newProps, key);
}

export function jsxs(type, props, key) {
  if (!hasOwnProperty.call(props, "class")) {
    return ReactJSXRuntime.jsxs(type, props, key);
  }

  const newProps = mergeClassProp(props);

  return ReactJSXRuntime.jsxs(type, newProps, key);
}
```

```js
// jsx-dev-runtime.js
import { mergeClassProp, hasOwnProperty } from "./utils";
import * as ReactJSXENVRuntime from "react/jsx-dev-runtime";

export const Fragment = ReactJSXENVRuntime.Fragment;

export function jsxDEV(type, props, key, isStaicChildren, source, self) {
  if (!hasOwnProperty.call(props, "class")) {
    return ReactJSXENVRuntime.jsxDEV(
      type,
      props,
      key,
      isStaicChildren,
      source,
      self
    );
  }

  const newProps = mergeClassProp(props);

  return ReactJSXENVRuntime.jsxDEV(type, newProps, key);
}
```

代码和之前的`createElement`其实没有很大的不同，都只是将传入的`props`做了一点简单的处理，然后用新 props 创建 element。除此之外，我们还需要提供自己的 JSX 类型定义：

```ts
// 内容参考了 @emotion/react
import "react";

import { ClassValue } from "clsx";

type WithConditionalClassProp<P> = "className" extends keyof P
  ? string extends P["className" & keyof P]
    ? { class?: ClassValue }
    : {}
  : {};

export namespace CustomJSX {
  interface Element extends JSX.Element {}
  interface ElementClass extends JSX.ElementClass {}
  interface ElementAttributesProperty extends JSX.ElementAttributesProperty {}
  interface ElementChildrenAttribute extends JSX.ElementChildrenAttribute {}

  type LibraryManagedAttributes<C, P> = WithConditionalClassProp<P> &
    JSX.LibraryManagedAttributes<C, P>;

  interface IntrinsicAttributes extends JSX.IntrinsicAttributes {}

  type IntrinsicElements = {
    [K in keyof JSX.IntrinsicElements]: JSX.IntrinsicElements[K] & {
      class?: ClassValue;
    };
  };
}
```

相比起 `@types/react` 提供的 JSX 定义，`CustomJSX` 对 `LibraryManagedAttributes` 和 `IntrinsicElements` 进行了增强，使组件能够接受 class prop。

在对`babel`进行配置之后，我们就可以使用我们提供的 `jsx` 来创建 element 了。完整的代码已放到[GitHub](https://github.com/panchao5/react-class-prop.git)上。
