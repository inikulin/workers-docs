---
title: 'Routes'
weight: 4
---

The mechanics for if and what script should run on a request based on the URL.

## Workers.dev

For workers.dev zones, scripts are managed by the name of the script. A request to your `script-name.subdomain.workers.dev` subdomain will run through the active script with corresponding name, `script-name`.

All the logic for routing must be done in the script itself. For example, you can use the [router template](https://github.com/cloudflare/worker-template-router).

Each script that runs must have a script name, so there is currently no way to run a script on `subdomain.workers.dev`.

## Cloudflare Sites

For zones proxied on Cloudflare\*, route patterns decide what (if any) script is matched based on the URL of that request. Requests are routed through a Workers script when their URL matches a route pattern assigned to that script. Route patterns are created either from inside the Cloudflare Workers editor, or by using the Cloudflare API.

\* _A zone that you have registered with some registrar (not workers.dev) and setup Cloudflare to serve as [a reverse proxy](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/)._

### Matching Behavior

Route patterns look like this:

```
https://*.example.com/images/*
```

This pattern would match all HTTPS requests destined for a subhost of
example.com and whose paths are prefixed by `/images/`.

While they look similar to a [regex](https://en.wikipedia.org/wiki/Regular_expression) pattern, route patterns follow specific rules:

- The only supported operator is wildcard `*` which matches zero or more of any character.
- Route patterns may not contain infix wildcards or query parameters, e.g.
  neither `example.com/*.jpg` nor `example.com/?foo=*` are valid route patterns.
- When more than one route pattern could match a request URL, the most specific
  route pattern wins. For example, the pattern `www.example.com/*` would take
  precedence over `*.example.com/*` when matching a request for
  `https://www.example.com/`.
- Route pattern matching considers the entire request URL, including the query
  parameter string. Since route patterns may not contain query parameters, the
  only way to have a route pattern match URLs with query parameters is to
  terminate it with a wildcard, `*`.

A route can be specified without being associated with a worker; this will act to negate any less specific patterns. For example, consider this pair of route patterns, one with a Workers script and one without:

```
*example.com/images/cat.png -> <no script>
*example.com/images/*       -> worker-script
```

In this example, all requests destined for example.com and whose paths are prefixed by `/images/` would be routed to `worker-script`, _except_ for `/images/cat.png`, which would bypass Workers completely. Requests with a path of `/images/cat.png?foo=bar` would be routed to `worker-script`, due to the presence of the query string.

### Validity

Here is the full set of rules governing route pattern validity:

- **Route patterns must include your zone**

  If your zone is `example.com`, then the simplest possible route pattern you
  can have is `example.com`, which would match `http://example.com/` and
  `https://example.com/`, and _nothing else_.
  As with a URL, there is an implied path of `/` if you do not specify one.

- **Route patterns may not contain any query parameters**

  For example, `https://example.com/?anything` is not a valid route pattern.

- **Route patterns may optionally begin with http:// or https://**

  If you omit a scheme in your route pattern, it will match both http:// and
  https:// URLs. If you include http:// or https://, it will only match HTTP
  or HTTPS requests, respectively.

  - `https://*.example.com/` matches `https://www.example.com/` but _not_
    `http://www.example.com/`

  - `*.example.com/` matches both `https://www.example.com/` _and_
    `http://www.example.com/`.

- **Hostnames may optionally begin with `*`**

      If a route pattern hostname begins with `*`, then it matches the host *and* all

  subhosts.

      If a route pattern hostname begins with `*.`, then it matches *only* all

  subhosts.

      - `*example.com/` matches `https://example.com/` *and* `https://www.example.com/`

      - `*.example.com/` matches `https://www.example.com/` but *not*
      `https://example.com/`

- **Paths may optionally end with `*`**

  If a route pattern path ends with `*`, then it matches all suffixes of that
  path.

  - `https://example.com/path*` matches `https://example.com/path` _and_
    `https://example.com/path2` _and_ `https://example.com/path/readme.txt`

  - `https://example.com/path/*` matches `https://example.com/path/readme.txt`
    but _not_ `https://example.com/path2`.
