# 用Sanitizer 预防XSS

`Sanitizer API` 是一个新的提案，目标是构建一个强大的处理器，以便将任意字符串更安全地插入到 `HTML` 页面。

![图片](https://mmbiz.qpic.cn/mmbiz_png/aDoYvepE5x2IGDTqAZeVmEbib8xCldibK8OJjhQGyAFNODLCMKGiaQeFwgh8zyHFmEa8vDs5OCQXO0Auzn6e3hq6Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

多年来，`DOM XSS` 一直是最普遍且最危险的 Web 安全漏洞之一。根据之前发布的 `Imperva` 报告，XSS 漏洞是 2014 年、2015 年、2016 年和 2017 年最普遍的基于 Web 的攻击形式。`2018` 年的漏洞冠军被 `SQL` 注入拿到了，`XSS` 漏洞仍然排在第二位。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/aDoYvepE5x2IGDTqAZeVmEbib8xCldibK8RnmLJecvwTkFrgGicFHad8FPGbdSOSaicL4lUhqB3mJql83GOzyiaGhGw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

它的主要原因就是，对于一些动态渲染到页面上的字符串，我们无法确保它的安全，里面可能掺杂恶意的攻击脚本。

新的 `Sanitizer API` 提案可以让我们将任意字符串安全地插入到页面中：

```
// Expanded Safely !!
$div.setHTML(`<em>hello world</em><img src="" onerror=alert(0)>`, new Sanitizer())
```

传统防止 XSS 的方法一般就是两种，对用户输入进行转义或消除。

## 转义

将用户输入、查询字符串、`cookie` 内容等插入 `DOM` 时，必须要正确转义这些字符串。通过 `.innerHTML` 插入未转义的字符串是 XSS 的典型来源。

```
const user_input = `<em>hello world</em><img src="" onerror=alert(0)>`
$div.innerHTML = user_input
```

如果在上面的输入字符串中转义 `HTML` 特殊字符或使用 `.textContent` 对它进行展开，`alert(0)` 就不会被执行。但是这样 `<em>` 标签也会被转义成字符串，这样原本预期中的 `HTML` 修饰也无法生效。

这种场景下，我们最好的方式不是进行转义，而是直接消除恶意脚本。

## 消除

转义是指把 `HTML 实体` 替换未特殊的 `HTML` 字符。而消除则指的是从 `HTML` 字符串中删除可能产生危害的脚本。

比如下面的例子：

```
// XSS 🧨
$div.innerHTML = `<em>hello world</em><img src="" onerror=alert(0)>`
// Sanitized ⛑
$div.innerHTML = `<em>hello world</em><img src="">`
```

为了实现正确的消除效果，可能需要将输入字符串解析为 HTML，省略被认为有风险的标签和属性，并保留安全的部分。

`Sanitizer API` 的目标就是提供这样的处理作为用于浏览器的标准API。

## Sanitizer API

```
const $div = document.querySelector('div')
const user_input = `<em>hello world</em><img src="" onerror=alert(0)>`
const sanitizer = new Sanitizer()
$div.setHTML(user_input, sanitizer) // <div><em>hello world</em><img src=""></div>
```

上面就是一个 `Sanitizer API` 的标准使用方法，值得注意的是，`setHTML()` 是定义在 `HTML Element` 下的一个方法，解析在内部完成一次，结果会直接扩展到 `DOM` 中。

如果不想直接扩展到 DOM，也可以直接把结果创建成一个 `HTMLElement`：

```
const user_input = `<em>hello world</em><img src="" onerror=alert(0)>`
const sanitizer = new Sanitizer()
sanitizer.sanitizeFor("div", user_input) // HTMLDivElement <div>
```

## 自定义配置

`Sanitizer API` 默认的配置会删除可能触发脚本执行的字符串，你也可以添加一些自定义配置：

```
const config = {
  allowElements: [],
  blockElements: [],
  dropElements: [],
  allowAttributes: {},
  dropAttributes: {},
  allowCustomElements: true,
  allowComments: true
};
// sanitized result is customized by configuration
new Sanitizer(config)
```

下面的选项可以指定清理结果应如何处理指定的元素。

- `allowElements`：Sanitizer 应保留的元素名称。
- `blockElements`：Sanitizer 应删除的元素名称，同时保留其子元素。
- `dropElements`：Sanitizer 应移除的元素名称及其子元素。

```
const str = `hello <b><i>world</i></b>`

new Sanitizer().sanitizeFor("div", str)
// <div>hello <b><i>world</i></b></div>

new Sanitizer({allowElements: [ "b" ]}).sanitizeFor("div", str)
// <div>hello <b>world</b></div>

new Sanitizer({blockElements: [ "b" ]}).sanitizeFor("div", str)
// <div>hello <i>world</i></div>

new Sanitizer({allowElements: []}).sanitizeFor("div", str)
// <div>hello world</div>
```

你还可以使用 `allowAttributes、dropAttributes` 来允许还是删指定的属性：

```
const str = `<span id=foo class=bar style="color: red">hello</span>`

new Sanitizer().sanitizeFor("div", str)
// <div><span id="foo" class="bar" style="color: red">hello</span></div>

new Sanitizer({allowAttributes: {"style": ["span"]}}).sanitizeFor("div", str)
// <div><span style="color: red">hello</span></div>

new Sanitizer({allowAttributes: {"style": ["p"]}}).sanitizeFor("div", str)
// <div><span>hello</span></div>

new Sanitizer({allowAttributes: {"style": ["*"]}}).sanitizeFor("div", str)
// <div><span style="color: red">hello</span></div>

new Sanitizer({dropAttributes: {"id": ["span"]}}).sanitizeFor("div", str)
// <div><span class="bar" style="color: red">hello</span></div>

new Sanitizer({allowAttributes: {}}).sanitizeFor("div", str)
// <div>hello</div>
```

## 与 DomPurify 的对比

```
DOMPurify` 是一个著名的库，也是提供类似的清理功能，`Sanitizier API` 和 `DOMPurify` 之间的主要区别在于 `DOMPurify` 可能会以字符串形式返回结果，你需要自己再调用 `.innerHTML.
const user_input = `<em>hello world</em><img src="" onerror=alert(0)>`
const sanitized = DOMPurify.sanitize(user_input)
$div.innerHTML = sanitized
// `<em>hello world</em><img src="">`
```

当浏览器中未实现 `Sanitizer API` 时，`DOMPurify` 可以作为 `backup`。

`DOMPurify` 实现还有几个缺点。如果返回一个字符串，则输入字符串会被 `DOMPurify` 和 `.innerHTML` 解析两次。这种双重解析会浪费处理时间，但也可能导致由于第二次解析的结果与第一次不同的情况引起其他的漏洞。比如下面这个漏洞：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

`Sanitizer API` 改进了 `DOMPurify` 的缺点，并且它未来会作为浏览器原生的 API 支持，目前各大浏览器正在实现中。

