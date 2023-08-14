# Explainer: ServiceWorker Static Routing API

## Authors

*   Yoshisato Yanagisawa ([@yoshisatoyanagisawa](https://github.com/yoshisatoyanagisawa))
*   Shunya Shishido ([@sisidovski](https://github.com/sisidovski))

## Participate

* https://github.com/w3c/ServiceWorker/issues/1373

## Background

[ServiceWorker](https://www.w3.org/TR/service-workers/) is a web platform feature that brings application-like experience to users.
It works in the background of a web browser to provide a cache feature, offline support, and other functionality.

## A problem and a solution

Starting ServiceWorkers is known to be a slow process, and web users need to wait for its startup if the ServiceWorker intercepts
loading the page resources.  At the same time, the ServiceWorker brings flexibility to the transport layer, and it behaves as
a client-side proxy.  Developers can implement offline support or provide client-side content modification with it.
Since using the ServiceWorker has a certain overhead, we need to use it wisely.

Currently, ServiceWorkers intercept all navigation for pages within their scope, which can cause a performance penalty.
To avoid this penalty, we need a way to specify when ServiceWorkers and JavaScript should not be run.

### NavigationPreload

[NavigationPreload](https://developer.mozilla.org/en-US/docs/Web/API/NavigationPreloadManager) was introduced in Chrome 59 
(June, 2017).  To avoid the navigation being blocked by the ServiceWorker startup, the browser sends a request to a network while
starting a ServiceWorker to hide a performance penalty to start a ServiceWorker.

However, there is still a delay while the ServiceWorker startup.  This may happen on slow devices, which can be unacceptable for
partners who care about web performance.  Additionally, this feature changes the execution order, which can make it difficult
to analyze and debug issues.  As a result, some partners have stopped using navigation preload.

Unlike navigation preload, the static routing API proposed below allows developers to exclude the ServiceWorker from the navigation
critical path where it is configured.  We can avoid the delay for the slow ServiceWorker start-up in that case.  The execution
order is easy to understand and predict.

## A proposal

We propose a new version of declarative routing API originally proposed by Jake Archibald in 2018-2019
([github](https://github.com/w3c/ServiceWorker/issues/1373),
[site](https://jakearchibald.com/2019/service-worker-declarative-router/)).
It defines set of static routing rules for the following purpose:

*   be able to bypass a ServiceWorker for particular requests.
*   speed up simple offline-first, online-first routes by avoiding ServiceWorker startup time.
*   be polyfillable.
*   be extensible.

Our proposal is heavily based on the proposal, with minor changes to use the modern primitives (URLPattern),
and simplify the API surface.  This is [a full picture of the proposal](final-form.md).  However, since it is too large,
we will start from something minimal (Refer to the FAQ section for the detailed list of changes applied in this proposal).

From the user's perspective, this proposal brings performance improvement by skipping waiting for ServiceWorker where it is
not needed.  Even with slow devices, users do not need to wait for the page load blocked by slow ServiceWorker start-up.
Not directly covered by this explainer, but we have offline first support in our big picture, and more pages can be used
offline when it is realized.

### WebIDL

Below is the WebIDL of the API.  You may find it more complicated than what it can do now.  That is because it is designed to
allow evolution to [the full picture](final-form.md) in the future.

```webidl
// This follows Jake's proposal.  Router is implemented in the InstallEvent.
interface InstallEvent : ExtendableEvent {
  // `registerRouter` is used to define static routes.
  // Not matching all rules means fallback to the regular process.
  // i.e. fetch handler.
  // Promise is rejected for invalid `rules` e.g. syntax error.
  Promise<undefined> registerRouter((RouterRule or sequence<RouterRule>) rules);
}

// RouterRule defines the condition and source of the static routes.  One entry contains one rule.
dictionary RouterRule {
  // In the first version, only the single condition and source should be enough because we only
  // support urlPattern, and "network".
  RouterCondition condition;
  RouterSourceEnum source;
}

// Defines when to respond from the source.
dictionary RouterCondition {
};

dictionary RouterUrlPatternCondition : RouterCondition {
  // a relative path pattern string usable for URLPattern.
  USVString urlPattern;
};

enum RouterSourceEnum { "network" };
```

### Examples

#### Bypassing ServiceWorker for particular resources

```js
// Go straight to the network and bypass invoking "fetch" handlers for all same-origin URLs that start
// with '/form/'.
addEventListener('install', (event) => {
  event.registerRouter({
    condition: {
      urlPattern: "/form/*"
    },
    source: "network"
  });
})

// Go straight to the network and bypass invoking "fetch" handlers for all same-origin URLs that start
// with '/videos/' and '/images/'.
addEventListener('install', (event) => {
  event.registerRouter([{
    condition: {
      urlPattern: "/images/*"
    },
    source: "network"
  },
  {
    condition: {
      urlPattern: "/videos/*"
    },
    source: "network"
  }]);
});
```

## Origin Trial

We (Google Chrome team) will start the origin trial, which allows developers to opt-in the feature on their sites. Regarding the introduction and basic registration process, please refer to [this page](https://developer.chrome.com/docs/web-platform/origin-trials/).

Once a token string is generated from [the dashboard](https://developer.chrome.com/origintrials), developers need to set the HTTP response header to their ServiceWorker files.

```
Origin-Trial: TOKEN_GOES_HERE
```

One important thing to mention here is that the header has to be set to **the ServiceWorker script, not to the page**. Also, HTML meta tags are not supported. The feature will take effect after the ServiceWorker registration.

For local testing, you can enable the feature by flipping the `Service Worker Static Router` flag from chrome://flags.

## FAQ

### How is the proposal different from Jake’s original proposal?

We propose `registerRouter()` to set routes with specified routes instead of `add()` and `get()`.  Unlike `add()` or `get()`,
`registerRouter()` sets all the routes at once, and it can only be called once.  This is for ease of understanding the latest routes.
`registerRouter` will be a part of the `install` event [^1].
Since `registerRouter()` is only the method to set the router rules, we put it as a part of the `install` event.
When the `install` listener is executed, no routes are set.  Web developers can call `registerRouter()` to set
routes at that time.

Our proposal uses [`URLPattern`](https://developer.mozilla.org/en-US/docs/Web/API/URLPattern), which was not available when Jake made
the original proposal.  It is natural evolution to use `URLPattern` instead of URL related conditions in the proposal.

[^1]: We followed [Jake's proposal](https://github.com/w3c/ServiceWorker/issues/1373#issuecomment-451436409) to use the install event.

#### Summary

*   Introduce `registerRouter()` method, and won’t provide `add()` or `get()` methods.
    *   `registerRouter()` method sets ServiceWorker routes with specified routes.
        This can only be called once, and calling this twice will cause an error.
*   URL related conditions are merged into `URLPattern`.

### How does it work with [empty fetch listeners](https://github.com/yoshisatoyanagisawa/service-worker-skip-no-op-fetch-handler)?

If [all the fetch listeners are empty functions](https://w3c.github.io/ServiceWorker/#all-fetch-listeners-are-empty-algorithm),
routes may not be evaluated because routes expect some is handled by the fetch listeners while others are not.
If empty fetch listeners are set, routes should not be needed.

### How does it work with Navigation Preload?

Routes are evaluated before Navigation Preload.  If no routes are matched, navigation falls back to the regular ServiceWorker
fetch handler path, and Navigation Preload would be used if it is configured.

### How would the full picture be rolled out?

[The full picture](final-form.md) is large, and it is not easy to implement it at once.  We plan to gradually roll it out with
the following order:

*   `RouterURLPatternCondition`, `RouterNetworkSource` (this proposal)
*   `RouterRequestCondition`, `RouterAndCondition`, `RouterNotCondition`
*   `RouterRunningStatusCondition`, `RouterOrCondition`
*   `RouterTimeCondition`
*   `RouterCacheSource`, `RouterFetchSource`, `RouterSourceBehaviorEnum`, allowing sequence of sources. (offline/online-first support)
*   Stale-While-Revalidate support.
*   race-network-and-fetch-event support

### How Chrome implements this?

The Google Chrome team starts the Oritin Trial from M116, and the implementation has slightly been chagned from M117.
It has been explained in [a separate document](update-from-chrome-m116.md).
