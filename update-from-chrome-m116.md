# What has been changed between Chrome M116 and later?

Chrome’s origin trial on the static routing API starts from [M116](https://groups.google.com/a/chromium.org/g/blink-dev/c/_H8rqHW9ERQ/m/KLnRZaz3AAAJ).
It has been implemented as a minimum viable product, and it only has a limited feature.  From M117, a full set of 
[URLPattern](https://developer.mozilla.org/en-US/docs/Web/API/URLPattern/URLPattern) has been supported, which brings a behavior difference from M116.

## M116 `urlPattern` behavior

In M116, the `urlPattern` condition is recognized as a pathname matching rule.  It matches with a request URL regardless of protocol, username, password,
hostname, port, query, and hash.  The pathname matching will be applied to cross-origin requests automatically. i.e. if you write a pattern like “\*.txt”,
it matches all accesses to a file that ends with “.txt” regardless of where it is under the ServiceWorker’s control.
For example, if you write a following rule:

```
addEventListener('install', (event) => {
  event.registerRouter({
    condition: {
      urlPattern: "*.jpg"
    },
    source: "network"
  });
})
```

Network fallback will happen for all of the following URLs:

*   `http://same.origin.example.org/images/icon.jpg`,
*   `http://some.example.org/images/icon.jpg?token=12345`,
*   `https://the.other.example.org/images/icon.jpg#hash`,
*   and `https://cross.origin.example.com/top.jpg`.

## M117 `urlPattern` behavior

From M117 and later, the `urlPattern` condition takes [URLPatternInput](
https://chromium.googlesource.com/chromium/src/+/02c1d179486e02c6c11fcc14031e494879a2ab5f/third_party/blink/renderer/modules/url_pattern/url_pattern.idl#5)
instead of `USVString`, and is used for creating [URLPattern](https://developer.mozilla.org/en-US/docs/Web/API/URLPattern/URLPattern) for matching.
If it is `USVString`, the ServiceWorker script’s base URL is implicitly used as a base URL argument passed to URLPattern. i.e. 
`new URLPattern(<USVString input>, <ServiceWorker script baseURL>)` is the pattern used for matching.  If you do not want the implicit base URL,
please use `URLPatternInit`.  If it is URLPatternInit, it is passed to URLPattern as-is. i.e. `new URLPattern(<URLPatternInit>)` is the pattern used
for matching.  For example, if you write a following rule and the ServiceWorker script is located in `https://www.example.com/util/sw.js`:

```
addEventListener('install', (event) => {
  event.registerRouter({
    condition: {
      urlPattern: "*.jpg"
    },
    source: "network"
  });
})
```
(Yes, the same pattern as above)

Network fallback will happen for the following URLs:

*   `https://www.example.com/util/top.jpg`,
*   and `https://www.example.com/util/images/icon.jpg`.

However, it does not cause network fallback for followings:

*   `http://www.example.com/images/icon.jpg`,
*   `http://www.example.com/util/icon.jpg?token=12345`,
*   `https://www.example.com/util/icon.jpg#hash`,
*   and `https://cross.origin.example.com/util/icon.jpg`.

That is because the pattern matches with a `.jpg` file under `https://www.example.com/util/` without a query string or a hashtag.

### A workaround to make it work for both M116 and M117

If you need the equivalent behavior with M116, the `urlPattern` field need to be `urlPattern: new URLPattern({pathname: ‘*.jpg’})` instead.
If you only need to match the URL within the same hostname without username, password, query string, or hashtag, you can also write like 
`urlPattern: “/*.jpg”`.  Even for this case, you need to write “/” prefix explicitly to prevent it being recognized as a path under a ServiceWorker
script’s directory.

To make the rule work for both M116 and M117+, we recommend setting the rule in M117+ way first then fallback to M116. i.e.

```
addEventListener('install', (event) => {
  event.registerRouter({  // Try M117+ way first.
    condition: {
      urlPattern: {pathname: "*.jpg"}
    },
    source: "network"
  }).catch(() => {  // Fallback to M116 way.
    event.registerRouter({
      condition: {
        urlPattern: "*.jpg"
      },
      source: "network"
    })
  });
})
```

You can also write this like:

```
addEventListener('install', async event => {
  try {
    await event.registerRouter({  // Try the M117+ way first.
      condition: {
        urlPattern: {pathname: "*.jpg"}
      },
      source: "network"
    })
  } catch(() => {  // Fallback to the M116 way.
    event.registerRouter({
      condition: {
        urlPattern: "*.jpg"
      },
      source: "network"
    });
  }
});
```

## FAQ

### How can I convert the M116’s urlPattern condition to M117’s?

In M116, Chrome only checks the pathname matching.  Thus, it can be simply rewritten like this.

* M116: `urlPattern: “<path pattern>”`
* M117: `urlPattern: new URLPattern({pathname: “<path pattern>”})`

Note that URLPattern automatically sets wildcards for unset fields, and matches anything.

**Example:**

* M116: `urlPattern: “*.html”`
* M117: `urlPattern: new URLPattern({pathname: “*.html”})`, or you can also write like `urlPattern: {pathname: “*.html”}`

### How can I test the urlPattern condition?

To test URLPattern behavior, you can actually construct a [URLPattern](https://developer.mozilla.org/en-US/docs/Web/API/URLPattern/URLPattern)
instance and execute the `test` method on Chrome.  For example, if you want to check URLPattern pattern `{pathname: “*.png”}`,
it will be tested like:

```
p = new URLPattern({pathname: “*.png”});
p.test(“https://www.example.com/icon.png”) // true
p.test(“https://www.example.com/top.html”) // false
```

If you want to check URLPattern `*.png` in M117+ style with the ServiceWorker script base URL is `https://www.example.org/code/sw.js`,
it will be tested like:

```
p = new URLPattern(“*.png”, “https://www.example.org/code/sw.js”);
p.test(“https://www.example.org/code/icon.png”) //  true
p.test(“https://www.example.org/icon.png”) // false
```

If you want to test M116’s condition, please see “How can I convert the M116’s urlPattern condition to M117’s?”.
