# ServiceWorker static routing API final form

This is for explaining the final form of the routing API.  Current explainer only covers the initial step.
However, it looks more complex than it should be without showing how the API would be in the future.
This document explains WebIDL and examples if all features are fulfilled.

## WebIDL

```idl
// This follows Jake's proposal.  Router is implemented in the InstallEvent.
interface InstallEvent {
  // `addRoutes` is used to define static routes.
  // Not matching all rules means fallback to the regular process.
  // i.e. fetch handler.
  // Promise is rejected for invalid `rules` e.g. syntax error.
  Promise<undefined> addRoutes((RouterRule or sequence<RouterRule>) rules);
}

// RouterRule defines the condition and source of the static routes.  One entry contains one rule.
dictionary RouterRule {
  (RouterCondition or sequence<RouterCondition>) condition;
  // If no options are needed, developers may write RouterSourceEnum shortly.
  (RouterSource or RouterSourceEnum or sequence<RouterSource or RouterSourceEnum>) source;
}

// Defines when to respond from the source.
dictionary RouterCondition {
};

directory RouterUrlPatternCondition : RouterCondition {
  // A relative path pattern string usable for URLPattern, which implicitly set baseURL
  // from the current origin that the ServiceWorker script belongs.
  (USVString or URLPatternInit) urlPattern;
};

dictionary RouterRequestCondition : RouterCondition {
  RequestMethod requestMethod;
  RequestMode requestMode;
  RequestDestination requestDestination;
};

dictionary RouterTimeCondition : RouterCondition {
  unsigned long long timeFrom = 0;
  unsigned long long timeTo = Infinity;
};

dictionary RouterRunningStatusCondition : RouterCondition {
  RunningStatusEnum runningStatus;
};

enum RunningStatusEnum { "running", "not-running" }

dictionary NetworkInformationRoundTripTime {
  unsigned long lt;
  unsigned long le;
  unsigned long gt;
  unsigned long ge;
};

dictionary RouterNetworkQualityCondition : RouterCondition {
  // https://wicg.github.io/netinfo/#dom-networkinformation-rtt
  NetworkInformationRoundTripTime rtt;
};

dictionary RouterAndCondition : RouterCondition {
  sequence<RouterCondition> and;
};

dictionary RouterOrCondition : RouterCondition {
  sequence<RouterCondition> or;
};

dictionary RouterNotCondition : RouterCondition {
  RouterCondition not;
};

// Defines where the response comes from.
dictionary RouterSource {
  RouterSourceEnum type;
  RouterSourceBehaviorEnum behaviorEnum = "finish-with-success";
};

enum RouterSourceEnum { "network", "cache", "fetch-event", "race-network-and-fetch-handler" };

// Behavior on successful response.
// finish-with-success: finish getting a response if the source is successful. (default)
// continue-discarding-latter-results: response with the successful result, while discarding the latter
// source results.
enum RouterSourceBehaviorEnum { "finish-with-success", "continue-discarding-latter-results" }

dictionary RouterNetworkSource : RouterSource {
  // If response has found and want to update cache with it,
  // set cache name here.
  DOMString? updatedCacheName;
  boolean? cacheErrorResponse;
};

dictionary RouterCacheSource : RouterSource {
  // cache name.
  DOMstring cacheName;

  // A specific request to be used for looking up the cache.
  // Otherwise, the current request will be used.
  Request? request;
};

dictionary RouterFetchEventSource : RouterSource {
  // ID to be used as a routerCallbackId in the fetch event.
  DOMString? id;
};
```

## Examples

### Offline-first

```js
// offline first for all same-origin URLs whose suffix is .png and .css.
addEventListener('install', (event) => {
  event.addRoutes({
    condition: {
      or: [
        {
          urlPattern: "/**/*.png"
        },
        {
          urlPattern: "/**/*.css"
        },
      ]
    },
    source: [
      {
        cacheName: "static resources"
      },
      "network"
    ]
  });
});
```

### Online-first

```js
// online first for all same-origin URLs that start with "/articles".
addEventListener('install', (event) => {
  event.addRoutes({
    condition: {
      urlPattern: "/articles/*"
    },
    source: [
      "network",
      {
        cacheName: "articles"
      },
      {
        cacheName: "articles",
        request: "/articles/offline"
      }
    ]
  });
});
```

### Ignore ServiceWorker for post requests

```js
// not use ServiceWorker for posting to 'form'.
addEventListener('install', (event) => {
  event.addRoutes({
    condition: {
      and: [
        {
          urlPattern: "/form/*"
        },
        {
          requestMethod: "post"
        }
      ],
    },
    source: "network"
  });
});
```

### Use service worker iif running

```js
// Use ServiceWorker for URLs that start with "/articles", if the service worker is currently running.
addEventListener('install', (event) => {
  event.addRoutes({
    condition: {
      and: [
        {
          urlPattern: "/articles/*"
        },
        {
          runningStatus: "running",
        }
      ],
    },
    source: [ "fetch-event", "network" ]
  });
});
```

### stale-while-revalidate

```js
// stale-while-revalidate same-origin URLs that start with "/articles".
addEventListener('install', (event) => {
  event.addRoutes({
    condition: {
      urlPattern: "/articles/*"
    },
    source: [
      {
        cacheName: "articles",
        behavior: "continue-discarding-latter-results",
      },
      {
        updatedCacheName: "articles",
      }
    ]
  });
});
```

### race fetch listener and network

```js
// race for all same-origin URLs that start with "/articles".
addEventListener('install', (event) => {
  event.addRoutes({
    condition: {
      urlPattern: "/articles/*"
    },
    source: "race-network-and-fetch-handler"
  });
});
```

### Avoid using ServiceWorker for non app-shell resources 
```js
// load non app shell resources from network.
addEventListener('install', (event) => {
  event.addRoutes({
    condition: {
      not: {urlPattern: "/app-shell/*"}
    },
    source: "network"
  });
});
```

### Avoid using ServiceWorker for good network connection
```js
// If network round-trip time is good enough, use network directly.
addEventListener('install', (event) => {
  event.addRoutes({
    condition: {
      // RTT <= 150 ms.
      rtt: {le: 150}
    },
    source: "network"
  });
  // Otherwise, fallback to fetch-handler (default).
});
```
