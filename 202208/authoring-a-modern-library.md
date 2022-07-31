# 编写现代 JavaScript 库

现代开发的过程，开发者常常会使用到各种各样的库。到目前为止，npm 上的包的数量已经超过了 200 万。而且这个数字还在不断的增加。那么，库对于我们开发者来说有什么意义呢？这个问题的答案是显而易见的。当开发者发现自己在不同的项目中需要编写类似的代码时，他们可以直接将一个项目的代码复制粘贴到另一个项目中去。但是这种做法的缺点是很明显的。一旦这段代码中有任何改动，开发者需要手动将这个改动同步到所有项目中。这无疑是非常耗费精力的行为，而且很难保证不会有项目被遗漏。于是，为了避免这个繁琐的工作，开发者会将这些公共的功能代码段提取出来，组成一个“工具箱”。之后，开发者需要使用某种工具时，只需要从这个“工具箱”中取。而这个“工具箱”也就是我们所说的库。在完成对库的编写之后，下一步要做的就是将库发布出去。这一步骤并不简单，因为有很多方面需要我们考虑。不过这次，我们主要讨论的是这个库支持哪些模块系统。

## JavaScript 的模块系统

前端总是在迅速的发展和变化之中。似乎在前端领域，唯一且正确的做法往往是不存在的。即使是当下最流行的库或者最推崇的解决方式，很可能已经走在成为明日黄花的道路上了。当初，Grunt 方兴未艾，却在不久之后被 gulp 抢走风头。而最后，这两者都已经鲜有人问津了。而在 JavaScript 的模块系统方面，也是如此。

### iife

最近才接触前端开发的同学可能会认为 JavaScript 有模块系统是理所当然的。的确，模块系统已经成为我们开发时离不开的一部分。现在的前端开发者在一个新建的 JavaScript 文档中输入的第一个词很可能就是`import`。但是，在我刚刚接触前端开发的时候，情况大不一样。那是一个大家还会争论应该学习原生 JavaScript 还是 jQuery 的年代。而在那个 ES 模块标准还没有诞生的“原始”的时代，库作者们会以什么样的形式发布他们刚完成的库呢？

首先，我们来看看当时的前端开发者通常是如何使用库的。假如开发者想要在一个项目中使用 jQuery，那么他会打开浏览器，进入 jQuery 的官网，点击下载最新版本的 jQuery，然后将下载到的文件移动到项目的某个目录下。接下来，他在`index.html`中插入了一个新`script`标签，将其`src`设置为刚刚移入的 jQuery 文件的路径。于是，他就可以在自己的代码中直接通过`$`使用 jQuery 提供的丰富的功能了。Hmm🤔️，这确实不如执行一次`npm install`来的方便啊。Anyway，为了能够让用户通过这种方式使用我们最新最潮的库，我们应该怎么做？

```js
var $ = (function (exports) {
  "use strict";

  var add = function (a, b) {
    return a + b;
  };

  exports.add = add;

  Object.defineProperty(exports, "__esModule", { value: true });

  return exports;
})({});
```

以上代码是`format`被设置为`iife`时，`rollup`生成的目标代码。

所谓 iife，就是 Immediately Invoked Function Expression，即“立即执行函数表达式”的缩写。iife 通常是`(function(){})()`形式。早期的库很多都是通过这种方式来模拟模块，因此这种模式也被称作是 module pattern。这样一来，函数中的变量就不会对全局作用域中的变量产生冲突，也就是避免了对全局命名空间的污染。不过在现在，这种技术被使用的越来越少了。因为我们现在通常是在模块作用域中创建变量，而即使是模块作用域中的`var`变量也不会污染全局命名空间。可惜的是，这种方式算不上是完美。

- **不存在依赖管理（dependency management）**：如果这个库依赖其他库，那么用户需要同时引入其依赖库，并且要确保`script`标签的顺序是正确的。比如，在引入 bootstrap 时，就必须先引入 jQuery。当项目有大量依赖时，对这些依赖的管理也会成为一个问题。
- **命名冲突**：虽然 iife 已经尽量在避免，但是仍然还是创建了一个全局变量，存在与用户代码产生冲突的可能性。

### CommonJS

在 2009 年，社区内出现了将 JavaScript 引入服务器端的讨论。于是，Kevin Dangoor 创建了 ServerJS 项目。

> What I’m describing here is not a technical problem. It’s a matter of people getting together and making a decision to step forward and start building up something bigger and cooler together.
>
> — Kevin Dangoor

后在 09 年的八月，这个项目被重命名为 CommonJS，以表示其 API 被应用于更多领域的可能性。CommonJS 是一个标准化组织。正如 ECMA 组织制定了 ECMAScript 的语言规范、W3C 组织规范了包含 DOM 在内的 JavaScript web API，CommonJS 的目标是为服务器端、桌面端等非浏览器环境中的 JavaScript 创建一个通行的准则。

服务器端应用没有 HTML 页面也没有 script 标签，因此一个真正的模块系统是不可或缺的。CommonJS 也确实为 JavaScript 增加了 modules 相关的 API。这个模块形式得到了广泛的应用，尤其是应用在使用 NodeJS 进行服务器端开发中。相信大家对其已经非常熟悉了，因此对其语法以及如何使用就不再进行过多赘述。

```js
"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

var add = function (a, b) {
  return a + b;
};

exports.add = add;
```

以上代码是`format`被设置为`cjs`时，`rollup`生成的目标代码。

于是，在使用我们的库时，用户只需要安装它，使用`require("my-awesome-library")`将模块引入代码中。每个模块的依赖清晰可见，令人厌恶的全局变量却消失无踪。Aha！iife 存在的两个问题都已经被解决。

### Asynchronous Module Definition（AMD）

CommonJS 模块也存在一个不是问题的问题 ———— 模块都是同步的。这意味着当执行到`var foo = require('some-module')`时，host 会暂停当前代码的执行，直到运行完模块之中的代码，再将模块 export 的值赋给`foo`变量。在服务器端，这样的处理是能够被接受的。但在浏览器端，这会为用户带来一个不好的体验。于是，Asynchronous Module Definition(AMD)应运而生。

> RequireJS 文档中关于为什么要使用 AMD 的阐述：[Why AMD?](https://requirejs.org/docs/whyamd.html#amd)

```js
define(["exports"], function (exports) {
  "use strict";

  var add = function (a, b) {
    return a + b;
  };

  exports.add = add;

  Object.defineProperty(exports, "__esModule", { value: true });
});
```

以上代码是`format`被设置为`amd`时，`rollup`生成的目标代码。

等等，这个`define`函数是从哪里冒出来的，它又有什么作用呢？别急，我们先来看看用户会如何使用这个包。

#### RequireJS

RequireJS 是一个帮我们加载 amd 模块的工具。使用 RequireJS 加载我们的库的代码将类似于：

```html
<!-- 引入require.js -->
<script src="/js/require.js"></script>

<script>
  /**
   * 对RequireJS进行配置
   */
  requirejs.config({
    baseUrl: "js/lib",
    paths: {
      app: "../app",
    },
  });

  /**
   *加载 js/lib/my-module.js 中定义的模块
   */
  requirejs(["my-module"], function ($) {
    console.log($.add(1, 2)); // 使用模块
  });
</script>
```

查阅了 RequireJS 的[官方文档](https://requirejs.org/docs/api.html)，我们发现 RequireJS 为我们提供了名为`requirejs`/`require`的用于加载模块的函数。它接收的第一个参数是由依赖组成的数组。RequireJS 会根据我们的配置以及模块的 id，生成 JS 文件的路径，然后加载该文件。在所有依赖加载完成后，RequireJS 将执行作为第二个参数传入的回调函数，并把各个模块 export 的内容作为参数依次传入。

继续阅读文档，我们知道 RequireJS 提供了`define`以供我们定义模块。`define`接收的第一个参数同样也是依赖数组，第二个参数是包含我们模块的代码的函数。RequireJS 首先要加载依赖的模块。依赖自身可能又依赖于其他模块，所以 RequireJS 会重复这个过程，直到所有的依赖都加载完成。接着，RequireJS 会调用我们提供的函数，并且将依赖的模块 export 的内容作为参数传入。

> RequireJS 是使用`head.appendChild()`加载模块所在的 JS 文件的。由于模块之间可能会形成复杂的依赖关系，所以 RequireJS 要确保这些模块以正确的顺序执行。

再回头看 rollup 生成的代码，可以发现生成的模块依赖了`exports`这个模块。事实上，RequireJS 为我们提供了三个“Magic Module”，分别是`require`、`exports`和`module`。他们并不是通常在 JS 文件中定义的模块，而是三个特殊的模块。在`exports`上新增的属性将成为模块公共接口的一部分。下面的代码会不会让你联想到 CommonJS 模块呢？

```js
define(["require", "exports", "my-module"], function (require, exports) {
  "use strict";

  const $ = require("my-module");

  var add = function (a) {
    return function (b) {
      return $.add(a, b);
    };
  };

  exports.add = add;
});
```

甚至，RequireJS 还提供了被称为 simplified CommonJS Wrapper 的语法糖：["Sugar" section in the Why AMD page](https://requirejs.org/docs/whyamd.html#sugar)。不过需要注意的是，尽管该语法糖有这样一个名字，这不意味着它和 CommonJS 模块是 100%[兼容](https://requirejs.org/docs/whyamd.html#commonjscompat)。

AMD 也不是没有缺点的。它常被人诟病的是语法比较繁琐，至少和 CommonJS 相比如此。模块的代码被包裹在`define`调用之中，增加了一层缩进。对于大型的模块来说，这可能不太理想。更重要的是，依赖数组中依赖的顺序需要和回调函数中参数的顺序一一对应。当依赖过多时，这会给我们增加麻烦。尽管如此，AMD 还是一个非常不错的 JavaScript 模块解决方案。

### Universal Module Definition（UMD）

现在，我们知道在不同的平台，偏好的模块形式是不同的。那我们编写库的时候，该选择哪种以哪种模块形式发布呢？或者，我们希望所有的平台开发者都可以使用我们的库，我们是要发布多个不同的版本吗？别急，我们的救星 UMD 来了。

```js
(function (global, factory) {
  typeof exports === "object" && typeof module !== "undefined"
    ? factory(exports)
    : typeof define === "function" && define.amd
    ? define(["exports"], factory)
    : ((global =
        typeof globalThis !== "undefined" ? globalThis : global || self),
      factory((global.$ = {})));
})(this, function (exports) {
  "use strict";

  var add = function (a, b) {
    return a + b;
  };

  exports.add = add;

  Object.defineProperty(exports, "__esModule", { value: true });
});
```

以上代码是`format`被设置为`umd`时，`rollup`生成的目标代码。

原来，UMD 就是一系列用于判断当前环境支持什么样的模块形式的`if`/`else`语句。打开`1.11.3`版本的 jQuery 的源码，我们可以发现 jQuery 正是使用了这种形式的模块。

### ECMAScript 模块

ECMAScript 2015 是近些年最为重要的一个 ECMAScript 版本，它为 JS 增加了非常多的 features，比如箭头函数、`let`/`const`变量等。而 ECMAScript 模块是其中的重量级 feature 之一。不过前端开发者都明白，理想和现实往往是有差距的。比如在 ES5 时代，如果所有的浏览器都能完全按照 ECMAScript specs 实现，那么前端开发者就不需要了解那么多与浏览器兼容性相关的 hacks，工作就能轻松很多。ECMAScript 2015 当然也面临着同样的困境。从 caniuse 网站上可以看到，原生支持 ES 模块的浏览器直到 2017 年才慢慢开始出现。即使到了今天，我们可能还是无法直接发布使用了 ES 模块 的 web app。不过，我们并不需要苦苦等待浏览器厂商更新他们的产品。现代化前端开发时代造就已经来临。而这一切都要感谢 babel、webpack 以及 TypeScript 等工具的出现。

babel 是一个 JavaScript 的编译器，能够将包含新 ECMAScript 语言特性的 JS 代码转化成为兼容性更好的目标代码。而且 babel 能做的远远不止这些。通过编写插件，我们可以为 babel 增加对 ECMAScript 标准之外的特性的支持。这意味着开发者可以做一些很有意思很酷的事，比如设计一个像 JSX 语法那样嵌入于 JS 中的 DSL。

有了这些工具，前端开发者们逐渐习惯于使用 ES 模块。而关于其语法，我相信大家都已经熟稔于心了，在此处不再作展开。

## 现代 JavaScript 库的开发

现在，让我们回到最开始的话题 ———— 我们应该如何发布我们的库。而且我们又增加了一个讨论的前提，那就是这个库的源代码使用的是 ES 模块，因为我们开发时已经离不开它了。为了让尽可能多的用户能够使用我们的库，我们会根据用户的使用方式提供不同版本的库。

> 由于有了 rollup 等工具的帮助，生成多个版本代码成为了发布库时的主流做法。

### 通过 `<script>` 引入

如果我们允许用户能够直接通过`<script>`标签引入库，那么我们应该输出一份使用 UMD 的版本以供用户下载使用。

### 在 NodeJS 中引入

NodeJS 同时支持 CommonJS 模块和 ECMAScript 模块。CommonJS 模块文件后缀名为`.cjs`，而 ES 模块文件后缀名为`.mjs`。而以`.js`结尾的文件会被 NodeJS 当作哪种模块加载则取决于`package.json`的`type`项。当`type`未设置或者值为`commonjs`时，`.js`文件被当作包含 CommonJS 模块的文件；当`type`的值为`module`时，`.js`文件被当作包含 ES 模块的文件。

这里有一个小问题需要留意。NodeJS 有两种 modules loader。ES module loader 是异步的，负责处理`import`语句和`import()`表达式。它不仅能够加载 ES 模块，也可以加载 CommonJS 模块。而 CommonJS module loader 是同步的，负责处理`require`调用。但因为同步这一性质，它仅能加载 CommonJS 模块。这就导致了当我们将`package.json`的`main`项指向一个 ES 模块文件，用户就无法直接使用`require`引入这个包。（如果想在 CommonJS 模块中引入 ES 模块，可以使用`import()`表达式。）

> 补充一些关于`import`、`import()`和`require()`的信息
>
> 1. `import React from "react";`是语句（Statement），正如`if (cond) {} else {}`。`import`语句中`from`之后只能跟字符串字面量。
> 2. `import('react')`是表达式（Expression），正如`cond ? foo : bar`。这里的`import`是关键字而不是函数。`import()`表达式中的可以是 AssignmentExpression。所以以下代码是符合语法的：
>
>    ```js
>    const module = "react";
>    const React = await import(module);
>    ```
>
> 3. `require(id)`是函数调用。

还好，NodeJS 注意到了`main`的局限性，并新增了一种在`package.json`中定义入口文件的方法 ———— 使用`exports`项。对于支持`exports`的 NodeJS 版本而言，其优先级高于`main`项。`exports` 补足了 `main` 只可以指定一个主入口文件的缺点，并增加了根据环境的不同解析到不同入口文件的能力（conditional exports），并且阻止了用户访问未定义的入口文件。利用 `exports`， 我们可以提供多种版本的文件，供 NodeJS 根据情况进行选择。一个使用`exports`的例子：

```json
// package.json
{
  "exports": {
    "import": "./index-module.js",
    "require": "./index-require.cjs"
  },
  "type": "module"
}
```

> The "exports" provides a modern alternative to "main" allowing multiple entry points to be defined, conditional entry resolution support between environments, and preventing any other entry points besides those defined in "exports".
>
> —— NodeJS 官方文档

`exports` 让包作者有了一定控制 NodeJS 如何解析包中模块路径的能力。NodeJS 还增加了`imports`项，进一步方便了包作者。不过这与本文的主题关系不大，因此不作过多介绍。只是，要提醒的一点是，bundler 未必支持这些 NodeJS 的新特性。因此，`imports`项的使用可能会导致 bundler 找不到模块的问题。

### bundler

大多数情况下，引入库的工作其实是交给了 webpack 之类的 bundler 处理。由于 bundler 有直接处理 ES 模块的能力，我们可以输出一份保留 ES 模块的版本。并且，将`package.json`的`module`设置为 ESM 版本文件的路径。

> ES modules 优于 CJS modules 的是前者对于 tree shaking 或者 dead code removal 更友好。

## 实例

目前比较适合用来库开发时使用的 bundler 有 rollup 和 esbuild。esbuild 的优势在于其 build 的速度非常快，rollup 的优势在于便捷的配置、优秀的插件机制以及丰富的社区生态等。一个使用 rollup 打包的例子： [modern-library-template](https://github.com/panchao5/modern-library-template)。

## 拓展：为什么不应该使用`export default`

库作者们应该尽量避免`export default`的使用。首先，`export default`只是语法糖，并不是开发所必须的。

```js
export default function () {}
```

与

```js
function _default_() {}

export { _default_ as default };
```

是等价的。

> 这里的`_default_`只起示意的作用，并不意味着真的存在这个变量。

其次，`export default`会导致项目中变量命名的混乱。最后也是最为关键的一点，`export deafult` 导致的与 CommonJS module 之间的兼容性问题。

假如有如下源文件：

```js
// hello.js

export default "foo";
```

大多数 bundler 输出的结果会类似于：

```js
"use strict";

Object.defineProperty(exports, "__esModule", { value: true });

var hello = "hello";

exports["default"] = hello;
```

这会导致：

1. 在 CJS 模块中，用户需要使用 `require('my-module').default` 来访问我们 export 的内容。这会给用户造成使用上的不便。
2. 在 ESM 模块中，用户使用`import foo from 'my-module'`来访问我们的库。但问题在于，`foo`的值应该是？对于这个问题，esbuild 作者在[文档中](https://esbuild.github.io/content-types/#default-interop)有过比较详细的描述。由于 bundler 和 Nodejs 的处理可能不同，所以尽量还是不要使用`export default`。


