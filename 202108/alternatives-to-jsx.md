上一篇文章提到了 JSX 并不是什么神秘的东西，它的存在只是为了将开发者从写冗长的`createElement` 或者`h` 的函数调用中解脱出来。而且由于 JSX 长得和 HTML 非常类似，对于前端开发者来说基本上没有学习的负担。但是 JSX 的缺点是它是一门浏览器并不认识的“方言”。只有通过 babel 等工具编译成普通的函数调用才能被浏览器执行。

避免写`createElment`调用并不是只有 JSX 这一种解决方案。比如 Vue 就支持使用 template 来描述 UI：

```js
const App = defineComponent({
  template: `
    <div>
      <button @click="inc">{{ count }}</button>
    </div>
  `,
  setup() {
    const count = ref(0);

    const inc = () => {
      count.value = count.value + 1;
    };

    return {
      count,
      inc,
    };
  },
});
```

注意，要使用 template option，必须使用带 compiler 版本的 vue。因为 compile 这一步骤被从 build time 移到了 runtime。这虽然帮我们省去了 build 这一过程，但是却增加了运行时所需要的步骤。而且引入的 vue 文件体积也会增加。不过，我个人不是非常喜欢 template，因为相比起 JSX，template 更有黑魔法的感觉。比如，上面的代码中使用到了 `count` 变量。然而这个变量来自哪里以及为什么可以访问到对于刚接触 Vue 的人来说可能是比较神秘的。

类似的，Preact 也在推行一种名为 HTM 的 JSX 替代方案。HTM 基于 JavaScript 的 Tagged Templates 特性，同样可以做到无需 build。而且它的语法和 JSX 非常类似。官网上的例子：

```html
<script type="module">
  import { h, Component, render } from "https://unpkg.com/preact?module";
  import htm from "https://unpkg.com/htm?module";

  // Initialize htm with Preact
  const html = htm.bind(h);

  function App(props) {
    return html`<h1>Hello ${props.name}!</h1>`;
  }

  render(html`<${App} name="World" />`, document.body);
</script>
```

可以看到，由于现代浏览器对于 ECMAScript 标准的逐步完善，上述代码可以直接在浏览器中执行。这给人一种返璞归真感，仿佛时间就回到了当年讨论 `<script>` 标签是应该放在`<head>`还是 `<body>` 最底部的 old days。而且，这不意味着我们必须在性能方面做出牺牲，HTM 也提供能将 `html` 表达式转换成为 `h` 函数调用的 babel 插件。

当然，这些语法之间并没有孰优孰略，只有适用的场景有所区别。JSX 在某些方面肯定也有上面介绍的两种方案所没有的优势。比如说，JSX 具有很好的 TypeScript 支持。在代码智能提示方面，JSX 的体验目前应该还是最优的。

更为重要的是，很多技术其实比较 generic，并不局限于某个框架或者某个库。比如之前提到的我们完全可以使用 JSX 来开发 Vue 项目。HTM 其实也是如此。由于它可以和任何形式为 `h(type, props, ...children)` 的函数绑定，那么理论上它也完全可以配合 Vue 来使用。接下来，我们就来试验一下是否确实如此。

## htm 绑定 Vue

根据文档，我们需要将 htm 绑定到一个签名为 `h(type, props, ...children)` 的函数。最简单的，我们可以写出这样一个函数：

```js
import * as vue from "vue";

function h(type, props, ...children) {
  if (children.length === 0) {
    return vue.h(type, props);
  }

  return vue.h(type, props, children);
}
```

再将 htm 绑定到这个函数，得到 `html` 函数：

```js
import htm from "htm";

const html = htm.bind(h);
```

我们来试试效果：

```js
test("html should return vnode", () => {
  const vnode = html`<div>Hello World</div>`;

  expect(isVNode(vnode)).toBe(true);
  expect(vnode.type).toBe("div");
  expect(vnode.children).toEqual(["Hello World"]);
});
```

这些 matchers 都顺利通过了，也就是用 htm 来写 Vue 代码是完全可行的。但是目前的 `h` 还过于简陋了，没有考虑用 `html` 写 slots 的情况。slots 用 Vue 的 `h` 写是这样的：

```js
render() {
  // `<div><child v-slot="props"><span>{{ props.text }}</span></child></div>`
  return h('div', [
    h(
      resolveComponent('child'),
      null,
      // pass `slots` as the children object
      // in the form of { name: props => VNode | Array<VNode> }
      {
        default: (props) => h('span', props.text)
      }
    )
  ])
}
```

用 `html` 写出来是这样：

```js
html`<${resolveComponent("child")}
  >${{ default: (props) => h("span", props.text) }}<//
>`;
```

但实际上这样的写法是错误的。因为最终，我们传给 `Vue.h` 的是 `[{ default: (props) => h('span', props.text) }]`，而这是一个数组。显然，这和官方给出的使用 `Vue.h` 的版本代码是不一样的。因此，我们需要对 htm 给我们的 children 进行一些处理，而且我们也和 vue-jsx 一样增加对 `v-slots` 的支持。最终，我们实现的效果应该是这样的：

```js
html`<${resolveComponent("child")}
  v-slots=${{ default: (props) => h("span", props.text) }}
><//>`;

// 或者

html`<${resolveComponent("child")}>${(props) => h("span", props.text)}<//>`;
```

为此我们对 `h` 进行修改：

```js
function h(type, props, ...children) {
  let slots = props?.["v-slots"];

  if (children.length === 1) {
    if (typeof children[0] === "function") {
      slots = slots ?? {};

      if (!slots.hasOwnProperty("default")) {
        slots.default = children[0];
      }
    }
  } else if (children.length === 0) {
    return vue.h(type, props);
  }

  return vue.h(type, props, slots ?? children);
}
```

我们简单的认为如果 children 只包含一个函数，那么那个函数应该被作为 default slot。并且我们认为 `v-slots` 中的 `default` 应该比我们推断的 default 优先级更高。当然，上面的代码没有考虑一些 edge cases。不过可以看到的是，对 htm 传给我们的参数做处理是非常容易。我们可以根据自己的需求对 `h` 进行修改，从而支持更高级的语法。比如，我们想增加对于 Custom Directives 的支持。目前，我们如果想使用 Custom Directives 可以这样写：

```js
import { vShow } from "vue";

withDirectives(html`<div>Hello World</div>`, [[vShow, false]]);
```

显然，这种方式使用起来非常不方便。我们希望看到的是：

```js
html`<div v-show=${false}>Hello World</div>`;
```

于是我们对传入的 props 进行一些修改：

```js
function h(type, props, ...children) {
  const newProps = {};

  const directives = [];

  for (const key in props) {
    if (props.hasOwnProperty(key)) {
      const result = /^v-(?<directiveName>[a-zA-Z_][0-9a-zA-Z_]*)$/.exec(key);
      if (!result) {
        newProps[key] = props[key];
      } else {
        const { directiveName } = result.groups;
        const directive =
          directiveName === "show"
            ? vue.vShow
            : vue.resolveDirective(directiveName);
        if (directive) {
          directives.push([directive, props[key]]);
        }
      }
    }
  }

  let slots = props?.["v-slots"];

  if (children.length === 1) {
    if (typeof children[0] === "function") {
      slots = slots ?? {};

      if (!slots.hasOwnProperty("default")) {
        slots.default = children[0];
      }
    }
  } else if (children.length === 0) {
    return vue.withDirectives(vue.h(type, newProps), directives);
  }

  return vue.withDirectives(
    vue.h(type, newProps, slots ? slots : children),
    directives
  );
}
export default h;
```

运行效果：

```js
test("html supports v-show", () => {
  const wrapper = mount(() => html`<div v-show=${false}>Hello World</div>`);
  expect(wrapper.html()).toMatchSnapshot();
});
// snapshot: exports[`html supports v-show 1`] = `"<div style=\\"display: none;\\">Hello World</div>"`;

test("suports custom directives", () => {
  /**
   * @type Directive<any, string>
   */
  const directive = {
    mounted(_, { value }) {
      console.log(`Hello ${value}`);
    },
  };

  const spy = jest.spyOn(global.console, "log");

  const wrapper = mount(() => html`<div v-greeting="World">Hello World</div>`, {
    global: {
      directives: { greeting: directive },
    },
  });
  expect(spy).toHaveBeenCalledWith("Hello World");
  expect(wrapper.html()).toMatchSnapshot();

  spy.mockRestore();
});

// snapshot: exports[`html suports custom directives 1`] = `"<div>Hello World</div>"`;
```

完整代码在[GitHub](https://github.com/panchao5/htm-vue-demo)。

可以看到，我们能够很方便的对语法进行增强以方便使用，而且我们能够做的还有很多很多，比如增加对 global 注册的组件的支持，对 `v-model` 和 `v-if` 的支持等。和之前我们对 JSX 进行增强时一样，限制我们的只有想象力。
