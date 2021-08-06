之前的文章中提到，htm 使用了 JavaScript 的 tagged template literals 特性。Template literal 和 tagged template literals 是 ES2015 引入的两种新 literals。它们看上去非常相似，但是实际上却非常不同。我们使用的更多的是 template literals，是支持插值的、允许多行的 string literals。而相比之下，比较低调的 tagged template literals 却是函数调用。

我相信大多数都已经对 template literals 很熟悉了，因此接下来我们主要来聊聊 tagged template。tagged template 虽然出场不多，但是熟悉 CSS-in-JS 的同学应该对它不陌生。styled-components 作为影响力比较大的 CSS-in-JS 库使用的就是 tagged template 语法。以下是一个 tagged template 的例子：

```js
tagFunction`Hello ${firstName} ${lastName}!`;
```

它看上去就是在 template string 前面加了一个函数。这实际上就触发了一次函数调用。上面的代码可以看作为这样的一次调用：

```js
tagFunction(["Hello", " ", "!"], firstName, lastName);
```

置于 template string 之前的被调用的函数被称为 tag function。tagged function 接受的参数可以分成两部分：template strings 和 substitutions。我们来一个具体的例子：

```js
function tagFunction(tempObj, ...subs) {
  console.log(tempObj, subs);
}

const name = "World";

tagFunction`Hello ${name}!`;
// 输出： [ 'Hello ', '!' ] [ 'World' ]
```

虽然第一个参数看上去是一个普通的数组，但是它还有一个保存 raw 版本 template string 的 `raw` 属性。那么 raw 版本有什么不同之处呢？其实，它们在大多数情况下都是一样的，只在对于 escape 字符 `\` 的处理上有区别：

```js
function tagFunction(tempObj, ...subs) {
  console.log(tempObj);
  console.log(tempObj.raw);
}

tagFunction`Hello\${`;
tagFunction`Hello\``;
tagFunction`Hello\n`;

// 输出依次为
// [ 'Hello${' ]
// [ 'Hello\\${' ]
// [ 'Hello`' ]
// [ 'Hello\\`' ]
// [ 'Hello\n' ]
// [ 'Hello\\n' ]
```

在 “cooked” 版本中，`\` 的作用有：

1. 防止了`${`被解析成 substitution 的开始
2. 将 <code>`</code> 转义成为普通的字符
3. 和 `\` 在 string literals 中一样，将 `n` 、 `x` 等字符转义成特殊字符

而在 raw 版本中，`\` 同样具有前两个功能。但是很特别的是即使在这种发挥出转义作用的 `\` 在 raw 版本还是得到了保留。而其他情况下的 `\` 都被替换成了 `\\`。也就是说，所有的 `\` 都被转义成了普通的反斜杠。

tagged templates 非常适合用来写 DSL。之前提到的 styled-components 和 htm（lit-html）都可以看作是在 JavaScript 中实现的一种 DSL。那接下来我们也来尝试实现一个 DSL。

markdown 是一种轻量的标记语言，为人们提供使用纯文本格式编写文档的能力。因为其语法的简单易用，markdown 几乎成为了文字编辑软件的标配。比如，这篇文字就是使用 markdown 写成的。那我们就来试试在 JavaScript 中嵌入 markdown。

```js
import markdownIt from "markdown-it";

const md = new markdownIt();

function mark(template, ...substitutions) {
  const raw = template.raw;

  let result = "";

  substitutions.forEach((substitution, idx) => {
    const t = raw[idx];

    result += t;
    result += String(substitution);
  });
  result += template[template.length - 1];
  const html = md.render(result);

  return String(html);
}
```

在上述代码，实现了一个简单的 tag function `mark`。 它的作用很简单就是将传入的 substitutions 转为字符串然后和 template strings 拼接称为完整的 markdown 字符串。然后，使用 markdown-it 将该字符串转成 html 字符串。来看它是如何使用的：

```js
const name = "Gary";
const html = mark`Hello *${name}*`; // "<p>Hello <em>Gary</em></p>\n"
```

可能有人会产生质疑，这和下面的代码有什么区别吗：

```js
import markdownIt from "markdown-it";

const md = new markdownIt();

const name = "Gary";
md.render(`Hello *${name}*`); // "<p>Hello <em>Gary</em></p>\n"
```

对于这个例子来说，两个版本的代码输出确实是一致的。但是它们在本质上还是有区别的。之前提到了，template literals 就是一种产生字符串的语法结构，作用和 string literals 类似。而 tagged templates 是触发了一次函数调用。也就是说，`md.render`接收的是一个字符串，而 `mark` 接收的是 template strings 和 substitutions。这也就意味着，`mark`有能力对传入的 substitutions 做各种转换和处理，而 `md.render` 是做不到这一点的。比如说，`mark`可以针对数组类型的 substitutions 做特殊的处理：

```js
if (Array.isArray(substitution)) {
  substitution = substitution.join(" ");
}
```

于是

```js
const names = ["Gary", "Tomas"];

mark`Hello *${names}*`; // "<p>Hello <em>Gary Tomas</em></p>\n"
```

而 `md.render` 版本则返回 `<p>Hello <em>Gary,Tomas</em></p>\n`。这是由于 `names` 在被拼接之前先被转成了字符串。tagged template 版本的 `mark` 还有另外一个优点。注意到我们在取 template strings 时取的是 raw 版本。这会导致以下的差异：

```js
mark`\# Hello *World*`; // "<p># Hello <em>World</em></p>\n"

md.render(`\# Hello *World*`); // "<h1>Hello <em>World</em></h1>\n"
```

注：`\` 在 markdown 中也有转义的作用。

在 `mark` 版本中，转义字符正确的起作用，防止了 `#` 被当成 Heading 的起始字符。而在 `md.render` 中，`\` 却没有起作用。这是因为 `\` 在 template literals 中也担任转义字符的作用。而对于 template literals， `\#` 就是 `#`。所以 `md.render` 拿到的字符串实际上就是`# Hello *World*`。要让 `\` 正确的在 markdown 语法中起效，我们需要对 `\` 进行转义，也就是传入 `\\# Hello *World*`。这样，`md.render` 的返回值就和 `mark` 版本一样了。这就像必须使用 `new RegExp("\\\\")` 才能匹配包含 `\` 的字符串。
