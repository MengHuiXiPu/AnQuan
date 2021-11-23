# Web 安全 - 简单案例演示如何使用 CSP 来防止 XSS 攻击

**内容安全策略**（**Content Security Policy，缩写 CSP**），配置好策略后，就算黑客发现安全漏洞也无法注入恶意脚本。

## 什么是 CSP？

**CSP 实施的是白名单策略，服务器告诉客户端哪些资源可加载和执行，不符合条件的会被阻止，是阻止 XSS 攻击的一种有效解决方案**。

CSP 使用分为两种，一是**设置 HTTP Response Headers** `Content-Security-Policy: "script-src 'self';"` 响应头。

![图片](https://mmbiz.qpic.cn/mmbiz_png/wnIMIiaEIIrgZYH7Sk2iaWXJXlSjSUorZ72Nd7SPHTm5rHxOqiaYc8IHkEHwicx3IEDC64aicUEaQhiaSdbiarGPxnnYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

另一种是**网页的  标签** `<meta http-equiv="Content-Security-Policy" content="script-src 'self';" />` 标签设置

Content-Security-Policy 内容由不容的指令组成，详情参考 Content-Security-Policy。

## CSP 案例演示

结合 Demo 做演示，进一步理解 CSP 的应用。结合上一讲 “[Web 安全 - 跨站脚本攻击 XSS 三种类型及防御措施](https://mp.weixin.qq.com/s?__biz=MzU3NTg5MjU1Mw==&mid=2247484403&idx=1&sn=4507ea22cd1a8659db10a6d683f0255c&scene=21#wechat_redirect)” XSS 例子做改造，首先设置 CSP 响应头如下：

```
app.get('/', (req, res) => { // app.js
  res.set('Content-Security-Policy', "script-src 'self' https;")
})
```

修改 `views/index.pug` 模版，加载一个 JS 脚本。

```
html
  head
    title= title
    script(src="https://www.nodejs.red/index.js") 
  body
    h1= message
```

运行之后，在控制台得到如下错误提示，意思是**我们设置的 CSP 策略为只有当前域的 JS 文件且是 HTTPS 协议才可加载**，`www.nodejs.red` 显然此时是非法的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/wnIMIiaEIIrgZYH7Sk2iaWXJXlSjSUorZ7LUYkuqiaE1pWb0PdlNyt0XPhnCSicC7Pbqj7aO6Y0QJWlbCbVLwLKDsg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为了解决这个问题，需要修改 CSP 策略，允许能够加载 `www.nodejs.red` 域下的 JS 文件。

```
app.get('/', (req, res) => { // app.js
  res.set('Content-Security-Policy', "script-src www.nodejs.red 'self' https;")
})
```

## CSP 策略如何允许一段脚本可执行

除此之外，根据我们上面设置的 CSP 策略，如果想在控制台通过 `document.write("<script> console.log(111) </script>")` 写入一段脚本代码，浏览器也会为阻止，无法执行。

![图片](https://mmbiz.qpic.cn/mmbiz_png/wnIMIiaEIIrgZYH7Sk2iaWXJXlSjSUorZ7sAiauPdrZvFdy1O5ZPWHuo3ceofeibmTbw3NicQ7icHbKdTYnbMaUkMAkw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

有没有想过一个问题：“如何在 CSP 策略里允许一段脚本可加载？”，细心的同学可能发现报错日志中有这样一句话 `a hash ('sha256-9JfoB1vAaB60i4F+k48fZHFrPLMI73jH+c94nYGoDEk=')` ，我们也可对可信任的脚本使用 sha256 算法计算出 hash 值（`[<hash-algorithm>-<base64-value>](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src)`），设置到 CSP 的响应头中。

注意，生成 hash 时不要包含 `<script>` 标签，以 Node.js 为例：

```
const crypto = require('crypto')
const getHash = (code, algorithm = 'sha256') => 
  `${algorithm}-${crypto.createHash(algorithm).update(code, 'utf8').digest("base64")}`
const hash = getHash(`console.log('Hello 五月君')`);
app.get('/', (req, res) => {
  res.set('Content-Security-Policy', `script-src '${hash}' www.nodejs.red 'self' https;`)
})
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/wnIMIiaEIIrgZYH7Sk2iaWXJXlSjSUorZ7nQSbr7jEQuoXDPNmtL2qdmjoNE2liawnlKsibYibxopvA9dGkSZbStXag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 上报 XSS 攻击记录

CSP 除了用来防止 XSS 攻击，还可上报攻击记录，`report-uri` 指定上报的服务器地址，浏览器会以 POST 请求 `application/csp-report` 协议将攻击记录放在 body 中发送至服务器。

```
res.set('Content-Security-Policy', `script-src 'self' https; report-uri /csp/report;`)
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/wnIMIiaEIIrgZYH7Sk2iaWXJXlSjSUorZ7S9JrcKia62qqYIuqOOBeI7yHfFic8wTp2zgbyqQfHuJM9icpPsKtic5PKA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 仅记录违规行为

内容安全策略 CSP 还可使用 `Content-Security-Policy-Report-Only` 响应头仅用来上报违规记录，不限制执行。

```
res.set('Content-Security-Policy-Report-Only', `script-src 'self' https; report-uri /csp/report;`)
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## Reference

- Content Security Policy 入门教程
- Content-Security-Policy

