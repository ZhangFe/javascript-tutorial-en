# Cross-window communication

The "Same Origin" (same site) policy limits access of windows and frame to each other.

The idea is that if we have two windows open: one from `john-smith.com`, and another one is `gmail.com`, then we wouldn't want a script from `john-smith.com` to read our mail.

## Same Origin [#same-origin]

如果两个 URL 具有相同的协议，域名和端口，则称他们是"同源"的。

以下的几个 URL 都是同源的:

- `http://site.com`
- `http://site.com/`
- `http://site.com/my/page.html`

但是下面几个不是:

- <code>http://<b>www.</b>site.com</code> (`www.` 域名与其他不同)
- <code>http://<b>site.org</b></code> (`.org` 域名与其他不同)
- <code><b>https://</b>site.com</code> (协议与其他不同: `https`)
- <code>http://site.com:<b>8080</b></code> (端口与其他不同: `8080`)

如果我们有另外一个窗口（一个弹出窗口或者 iframe）的引用，并且这个窗口是同源的，那么我们可以使用它做任何事情

If it comes from another origin, then we can only change its location. Please note: not *read* the location, but *modify* it, redirect it to another place. That's safe, because the URL may contain sensitive parameters, so reading it from another origin is prohibited, but changing is not.
如果它不是同源的，那么我们只能改变它的。这样是安全的，因为 URL 可能包含一些敏感的参数，

Also such windows may exchange messages. Soon about that later.

````warn header="Exclusion: subdomains may be same-origin"

There's an important exclusion in the same-origin policy.

If windows share the same second-level domain, for instance `john.site.com`, `peter.site.com` and `site.com`, we can use JavaScript to assign to `document.domain` their common second-level domain `site.com`. Then these windows are treated as having the same origin.

In other words, all such documents (including the one from `site.com`) should have the code:

```js
document.domain = 'site.com';
```

Then they can interact without limitations.

That's only possible for pages with the same second-level domain.
````

## Accessing an iframe contents

An `<iframe>` is a two-faced beast. From one side it's a tag, just like `<script>` or `<img>`. From the other side it's a window-in-window.

The embedded window has a separate `document` and `window` objects.

We can access them like using the properties:

- `iframe.contentWindow` is a reference to the window inside the `<iframe>`.
- `iframe.contentDocument` is a reference to the document inside the `<iframe>`.

When we access an embedded window, the browser checks if the iframe has the same origin. If that's not so then the access is denied (with exclusions noted above).

For instance, here's an `<iframe>` from another origin:

```html run
<iframe src="https://example.com" id="iframe"></iframe>

<script>
  iframe.onload = function() {
    // we can get the reference to the inner window
    let iframeWindow = iframe.contentWindow;

    try {
      // ...but not to the document inside it
      let doc = iframe.contentDocument;
    } catch(e) {
      alert(e); // Security Error (another origin)
    }

    // also we can't read the URL of the page in it
    try {
      alert(iframe.contentWindow.location);
    } catch(e) {
      alert(e); // Security Error
    }

    // ...but we can change it (and thus load something else into the iframe)!
    iframe.contentWindow.location = '/'; // works

    iframe.onload = null; // clear the handler, to run this code only once
  };
</script>
```

The code above shows errors for any operations except:

- Getting the reference to the inner window `iframe.contentWindow`
- Changing its `location`.

```smart header="`iframe.onload` vs `iframe.contentWindow.onload`"
The `iframe.onload` event is actually the same as `iframe.contentWindow.onload`. It triggers when the embedded window fully loads with all resources.

...But `iframe.onload` is always available, while `iframe.contentWindow.onload` needs the same origin.
```

And now an example with the same origin. We can do anything with the embedded window:

```html run
<iframe src="/" id="iframe"></iframe>

<script>
  iframe.onload = function() {
    // just do anything
    iframe.contentDocument.body.prepend("Hello, world!");
  };
</script>
```

### Please wait until the iframe loads

When an iframe is created, it immediately has a document. But that document is different from the one that finally loads into it!

Here, look:


```html run
<iframe src="/" id="iframe"></iframe>

<script>
  let oldDoc = iframe.contentDocument;
  iframe.onload = function() {
    let newDoc = iframe.contentDocument;
*!*
    // the loaded document is not the same as initial!
    alert(oldDoc == newDoc); // false
*/!*
  };
</script>
```

That's actually a well-known pitfall for novice developers. We shouldn't work with the document immediately, because that's the *wrong document*. If we set any event handlers on it, they will be ignored.

...But the `onload` event triggers when the whole iframe with all resources is loaded. What if we want to act sooner, on `DOMContentLoaded` of the embedded document?

That's not possible if the iframe comes from another origin. But for the same origin we can try to catch the moment when a new document appears, and then setup necessary handlers, like this:

```html run
<iframe src="/" id="iframe"></iframe>

<script>
  let oldDoc = iframe.contentDocument;

  // every 100 ms check if the document is the new one
  let timer = setInterval(() => {
    if (iframe.contentDocument == oldDoc) return;

    // new document, let's set handlers
    iframe.contentDocument.addEventListener('DOMContentLoaded', () => {
      iframe.contentDocument.body.prepend('Hello, world!');
    });

    clearInterval(timer); // cancel setInterval, don't need it any more
  }, 100);
</script>
```

Let me know in comments if you know a better solution here.

## window.frames

An alternative way to get a window object for `<iframe>` -- is to get it from the named collection  `window.frames`:

- By number: `window.frames[0]` -- the window object for the first frame in the document.
- By name: `window.frames.iframeName` -- the window object for the frame with  `name="iframeName"`.

For instance:

```html run
<iframe src="/" style="height:80px" name="win" id="iframe"></iframe>

<script>
  alert(iframe.contentWindow == frames[0]); // true
  alert(iframe.contentWindow == frames.win); // true
</script>
```

An iframe may have other iframes inside. The corresponding `window` objects form a hierarchy.

Navigation links are:

- `window.frames` -- the collection of "children" windows (for nested frames).
- `window.parent` -- the reference to the "parent" (outer) window.
- `window.top` -- the reference to the topmost parent window.

For instance:

```js run
window.frames[0].parent === window; // true
```

We can use the `top` property to check if the current document is open inside a frame or not:

```js run
if (window == top) { // current window == window.top?
  alert('The script is in the topmost window, not in a frame');
} else {
  alert('The script runs in a frame!');
}
```

## The sandbox attribute

The `sandbox` attribute allows to forbid certain actions inside an `<iframe>`, to run an untrusted code. It "sandboxes" the iframe by treating it as coming from another origin and/or applying other limitations.

By default, for `<iframe sandbox src="...">` the "default set" of restrictions is applied to the iframe. But we can provide a space-separated list of "excluded" limitations as a value of the attribute, like this: `<iframe sandbox="allow-forms allow-popups">`. The listed limitations are not applied.

In other words, an empty `"sandbox"` attribute puts the strictest limitations possible, but we can put a space-delimited list of those that we want to lift.

Here's a list of limitations:

`allow-same-origin`
: By default `"sandbox"` forces the "different origin" policy for the iframe. In other words, it makes the browser to treat the `iframe` as coming from another origin, even if its `src` points to the same site. With all implied restrictions for scripts. This option removes that feature.

`allow-top-navigation`
: Allows the `iframe` to change `parent.location`.

`allow-forms`
: Allows to submit forms from `iframe`.

`allow-scripts`
: Allows to run scripts from the `iframe`.

`allow-popups`
: Allows to `window.open` popups from the `iframe`

See [the manual](mdn:/HTML/Element/iframe) for more.

The example below demonstrates a sandboxed iframe with the default set of restrictions: `<iframe sandbox src="...">`. It has some JavaScript and a form.

Please note that nothing works. So the default set is really harsh:

[codetabs src="sandbox" height=140]


```smart
The purpose of the `"sandbox"` attribute is only to *add more* restrictions. It cannot remove them. In particular, it can't relax same-origin restrictions if the iframe comes from another origin.
```

## 跨窗口传递消息

通过 `postMessage` 这个接口，我们可以在不同源的窗口内进行通信。

它有两个部分。

### postMessage

想要发送消息的窗口需要调用接收窗口的 [postMessage](mdn:api/Window.postMessage) 方法来传递消息。换句话说，如果我们想把消息发送到 `win`，我们应该调用 `win.postMessage(data, targetOrigin)`。

这个接口有以下参数：

`data`：要发送的数据。可以是任何对象，接口内部会使用"结构化克隆算法"将数据克隆一份。IE 只支持字符串，因此我们需要对复杂对象调用 `JSON.stringify` 以支持该浏览器

`targetOrigin`：指定目标窗口的源，以确保只有来自指定源的窗口才能获得该消息。

`targetOrigin` 是一种安全措施。请记住，如果目标窗口是非同源的，我们无法读取它的 `location`，因此我们就无法确认当前在预期的窗口中打开的是哪个站点：因为用户随时可以跳转走。

指定 `targetOrigin` 可以确保窗口内指定的网站还存在时才会接收数据。在有敏感数据时非常重要。

举个例子：这里只有当 `win` 内的站点是 `http://example.com` 这个源时才会接收消息：

```html no-beautify
<iframe src="http://example.com" name="example">

<script>
  let win = window.frames.example;

  win.postMessage("message", "http://example.com");
</script>
```

如果我们不希望做这个检测，可以将 `targetOrigin` 设置为 `*`。

```html no-beautify
<iframe src="http://example.com" name="example">

<script>
  let win = window.frames.example;

*!*
  win.postMessage("message", "*");
*/!*
</script>
```


### onmessage

为了接收消息，目标窗口应该在 `message` 事件上增加一个处理函数。当 `postMessage` 被调用时这个事件会被触发（并且 `targetOrigin` 检查成功）。

这个事件的 event 对象有一些特殊属性：

`data`：从 `postMessage` 传递来的数据。

`origin`： 发送方的源，举个例子： `http://javascript.info`。

`source`： 对发送方窗口的引用。如果我们需要的话可以立即回复 `postMessage`。

为了处理这个事件，我们需要使用 `addEventListener`，简单使用 `window.onmessage` 不起作用。

这里有一个例子：

```js
window.addEventListener("message", function(event) {
  if (event.origin != 'http://javascript.info') {
    // 从未知源获取的消息，忽略它
    return;
  }

  alert( "received: " + event.data );
});
```

这里有完整的示例：

[codetabs src="postmessage" height=120]

```smart header="There's no delay"
`postMessage` 和 `message` 事件之间完全没有延迟。他们是同步的，甚至比 `setTimeout(...,0)` 还要快。
```

## Summary

To call methods and access the content of another window, we should first have a reference to it.

For popups we have two properties:
- `window.open` -- opens a new window and returns a reference to it,
- `window.opener` -- a reference to the opener window from a popup

For iframes, we can access parent/children windows using:
- `window.frames` -- a collection of nested window objects,
- `window.parent`, `window.top` are the references to parent and top windows,
- `iframe.contentWindow` is the window inside an `<iframe>` tag.

If windows share the same origin (host, port, protocol), then windows can do whatever they want with each other.

Otherwise, only possible actions are:
- Change the location of another window (write-only access).
- Post a message to it.

Exclusions are:
- Windows that share the same second-level domain: `a.site.com` and `b.site.com`. Then setting `document.domain='site.com'` in both of them puts them into the "same origin" state.
- If an iframe has a `sandbox` attribute, it is forcefully put into the "different origin" state, unless the `allow-same-origin` is specified in the attribute value. That can be used to run untrusted code in iframes from the same site.

The `postMessage` interface allows two windows to talk with security checks:

1. The sender calls `targetWin.postMessage(data, targetOrigin)`.
2. If `targetOrigin` is not `'*'`, then the browser checks if window `targetWin` has the URL from  `targetWin` site.
3. If it is so, then `targetWin` triggers the `message` event with special properties:
    - `origin` -- the origin of the sender window (like `http://my.site.com`)
    - `source` -- the reference to the sender window.
    - `data` -- the data, any object in everywhere except IE that supports only strings.

    We should use `addEventListener` to set the handler for this event inside the target window.
