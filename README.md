# Origin Isolation explainer

**Origin isolation** refers to segregating cross-origin documents into separate [agent clusters](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism). Translated into developer-observable effects, this means:

* preventing the [`document.domain`](https://html.spec.whatwg.org/multipage/origin.html#relaxing-the-same-origin-restriction) setter from relaxing the same-origin policy; and
* preventing `SharedArrayBuffer`s from being shared with cross-origin (but same-site) documents.

If a developer chooses to give up these capabilities, then the browser has more flexibility in how it allocates resources like processes, threads, and event loops.

This proposal allows web applications to give up these capabilities for their origin, and also hint at why they are doing so, to better guide the browser in its resource allocation decisions.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of contents

- [Example](#example)
- [Objectives](#objectives)
- [Proposed hints](#proposed-hints)
- [Implementation strategies](#implementation-strategies)
- [How it works](#how-it-works)
  - [Background: agent clusters](#background-agent-clusters)
  - [Specification plan](#specification-plan)
  - [More complicated example](#more-complicated-example)
- [Design choices and alternatives considered](#design-choices-and-alternatives-considered)
  - [Origin policy](#origin-policy)
  - [The browsing context group scope of isolation](#the-browsing-context-group-scope-of-isolation)
  - [Not using feature/document policy](#not-using-featuredocument-policy)
  - [Usage hints](#usage-hints)
  - [Opt-in, instead of implicit, origin isolation](#opt-in-instead-of-implicit-origin-isolation)
- [Adjacent work](#adjacent-work)
  - [`COOP` + `COEP`](#coop--coep)
  - [Automatic origin isolation via `COOP` + `COEP`](#automatic-origin-isolation-via-coop--coep)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Example

The delivery mechanism for a web application to achieve origin isolation is [origin policy](https://github.com/WICG/origin-policy). The application would have a JSON file at `/.well-known/origin-policy` looking something like:

```json
{
  "id": "my-policy",
  "isolation": {
    "prefer_isolated_event_loop": true,
    "prefer_isolated_memory": true
  }
}
```

and would add a HTTP header

```
Origin-Policy: allowed=("my-policy" null)
```

to their responses.

The presence of the `"isolation"` field indicates that any realms (documents or workers) derived from that origin should be separated from other cross-origin realms—even if those realms are [same site](https://html.spec.whatwg.org/multipage/origin.html#same-site). In terms of observable, specified consequences for web developers, this means:

* Attempts to set `document.domain` will fail, so cross-origin realms will not be able to synchronously script each other, even if they are same site.
* Attempts to share `SharedArrayBuffer`s to cross-origin realms via `postMessage()` will fail, even if those realms are same site.

The two hints provided, `"prefer_isolated_event_loop"` and `"prefer_isolated_memory"`, indicate that the primary reasons the origin is giving up these capabilities is in the hopes that the browser can provide better isolation of the origin's event loop and memory. See below for more details on [the possible hints](#proposed-hints).

The browser can then use these hints to choose a behind-the-scenes implementation strategy such as isolating the origin into its own process, or its own thread, or memory heap, or any other such implementation construct. These are just hints; ultimately the browser remains in charge of such implementation decisions.

## Objectives

Goals:

* Allow web applications to isolate themselves from cross-origin memory sharing and synchronous scripting (in a guaranteed way).
* Allow user agents to use various implementation strategies to isolate origins when circumstances allow, and the developer has opted in to origin isolation, thus potentially achieving benefits for performance, security, and memory metrics.
* Allow web applications to provide information to the browser about why they want isolation, in order to better guide the browser's implementation strategies.
* Ensure consistent web-developer-observable behavior regardless of implementation choices (such as process allocation) or the sub-preferences expressed by an origin. (Modulo side channels.)
* Allow web applications to easily configure origin isolation for all URLs on their origin.

Non-goals:

* Mandate process isolation or other implementation choices for user agents.
* Give web developers visibility into process allocation.

## Proposed hints

The following hints are proposed, all booleans:

* `"prefer_isolated_event_loop"`: indicates that one reason the origin is opting in to isolation is in the hope that its event loop will be isolated from those of other origins. Origins might want this in order to get more consistent and predictable performance, isolated from whatever is happening in cross-origin iframes or popups.

* `"prefer_isolated_memory"`: indicates that one reason the origin is opting in to isolation is in the hope of having an as-large-as-possible amount of memory available, such that large allocations from cross-origin pages should not cause the origin's allocations to start failing. Origins that plan on using lots of memory might want this.

* `"for_side_channel_protection"`: indicates that one reason the origin is opting in to isolation is in the hopes of preventing side-channel attacks such as [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)). Origins might want this if they deal with sensitive data, which would be problematic if exfiltrated.

* `"for_memory_measurement"`: indicates that one reason the origin is opting in to isolation is in the hopes of enabling accurate memory measurement via the [proposed `performance.measureMemory()` API extensions](https://github.com/ulan/javascript-agent-memory/blob/master/explainer.md#future-api-extensions). This API is proposed to reject if the browser cannot restrict its measurements to only same-origin resources (or cross-origin ones which have opted in via `Cross-Origin-Resource-Policy`). Thus, by indicating that the origin plans to use this API, the browser can take steps to ensure the measurements are restricted.

If none of these hints are present, i.e. the manifest contains `"isolation": {}` or `"isolation": true`, then it is assumed that the origin wants isolation solely for increased encapsulation, i.e. for the direct effects of preventing `document.domain` usage and `SharedArrayBuffer` memory sharing.

## Implementation strategies

Currently, the state of the art in isolation is process-based. That is, current browser implementation strategies would likely place each origin-isolated origin in its own process.

The nuances of this proposal come into play under the following scenarios:

* When allocating a process-per-origin is infeasible, e.g. because you are operating on a low-memory device or have already allocated a ton of processes. In such scenarios the browser can use the different hints to prioritize how it allocates processes, e.g. prioritizing protection from side-channel attacks above allowing memory measurement. 

* When alternate isolation techniques are available, which give some but not all of the benefits as proccess isolation. For example, Blink is exploring a ["multiple Blink isolates/threads"](https://docs.google.com/document/d/12wEWJsZmxVnNwVGuxuEJF4922OWUr4fCs1xKHi9mTiI/edit#heading=h.g6fq85as9ptv) project, which would provide isolated event loops and heaps, but not full side-channel protection. A site which expressed a preference only for isolated event loops could thus be given a new thread, instead of a new process, saving resources.

## How it works

### Background: agent clusters

An important concept in the browser processing model is the [agent cluster](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism). Agent clusters are how the specification expresses which realms (windows/workers/etc.) are guaranteed to be able to share memory with each other using `SharedArrayBuffer`. They also constrain which pages can potentially synchronously script each other, e.g. via `someIframe.contentWindow.someFunction()`.

The HTML Standard currently [keys](https://html.spec.whatwg.org/multipage/webappapis.html#agent-cluster-key) agent clusters on scheme-and-registrable-domain ("site"), which roughly means that any documents within the same [browsing context group](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context-group) that are same-site can share memory with each other or synchronously script each other. Examples:

* `https://sub1.example.org/` can share memory with a `https://sub2.example.org/` iframe. It cannot share with a `http://sub1.example.org/` iframe (mismatched scheme).
* `https://example.com/` can share memory with a `https://example.com/` popup that it opens. It cannot share with a completely separate `https://example.com/` tab, nor with a popup opened with `noopener` (mismatched browsing context group).
* `https://whatwg.github.io/` cannot share memory with a `https://jsdom.github.io/`, because `github.io` is on the Public Suffix List.

Moving from the spec to the implementation level, the agent cluster boundary generally constrains how an implementation performs _process_ separation. That is, since the specification mandates certain documents be able to share memory with each other, implementations are constrained to put them in the same process, if they want to follow the specifications.

This means that the current state of the art in terms of segregating web content into different processes is [_site isolation_](https://www.chromium.org/Home/chromium-security/site-isolation), which segregates documents into separate processes according to the agent cluster rules. This can be done with no observable effects for JavaScript code.

Moving to origin isolation requires some observable changes, and thus the opt-in we propose here.

### Specification plan

The specification for this feature in itself is fairly small. Most of the work is done by the foundational mechanisms of [origin policy](https://github.com/WICG/origin-policy) and the HTML Standard's [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map).

In particular, when creating a document or worker, we use the following in place of the current algorithm for [obtain an agent cluster key](https://html.spec.whatwg.org/#obtain-agent-cluster-key). It takes as input an _origin_ of the realm to be created, _requestsIsolation_ (a boolean, derived from the `"isolation"` key in the origin policy manifest), and a browsing context group _group_. It outputs the [agent cluster key](https://html.spec.whatwg.org/#agent-cluster-key) to be used, which will be either a site or an origin.

1. If _origin_ is an [opaque origin](https://html.spec.whatwg.org/#concept-origin-opaque), then return _origin_.
1. If _origin_'s [host](https://html.spec.whatwg.org/#concept-origin-host)'s [registrable domain](https://url.spec.whatwg.org/#host-registrable-domain) is null, then return _origin_.
1. If _group_'s [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map)\[_origin_] exists, then return _origin_.
1. Let _site_ be (_origin_'s [scheme](https://html.spec.whatwg.org/#concept-origin-scheme), _origin_'s [host](https://html.spec.whatwg.org/#concept-origin-host)'s [registrable domain](https://url.spec.whatwg.org/#host-registrable-domain)).
1. If _requestsIsolation_ is true:
    1. If _group_'s [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map)\[_site_] exists, and _group_'s [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map)\[_site_] contains any agents which contain any realms whose [settings object](https://html.spec.whatwg.org/#concept-realm-settings-object)'s [origin](https://html.spec.whatwg.org/#concept-settings-object-origin) are [same-origin](https://html.spec.whatwg.org/#same-origin) with _origin_, then return _site_.
    1. Return _origin_.
1. Return _site_.

This algorithm has two interesting features:

* Even if origin isolation is not requested for a particular document/worker creation, if origin isolation has previously created an origin-keyed agent cluster, then we put the new document/worker in that origin-keyed agent cluster.
* Even when origin isolation is requested, if there is a site-keyed agent cluster with same-origin realms, then we put the new document/worker in that site-keyed agent cluster, ignoring the isolation request.

Both of these are consequences of a desire to ensure that same-origin sites do not end up isolated from each other.

You can see a more full analysis of what results this algorithm produces in our [scenarios document](./scenarios.md).

As far as the consequences of using origins for agent cluster keys, this will automatically cause `SharedArrayBuffer`s to no longer be shareable cross-origin, because of the agent cluster check in [StructuredDeserialize](https://html.spec.whatwg.org/multipage/structured-data.html#structureddeserialize). We'd also need to add a check to the [`document.domain` setter](https://html.spec.whatwg.org/multipage/origin.html#dom-document-domain) to make it throw. (Ideally we'd find some way of restructuring the existing checks in `document.domain` so that they are also tied to agent cluster boundaries.)

### More complicated example

TODO: move this to scenarios.md?

Consider the following scenario:

* `https://example.org/` embeds an iframe for `https://a.example.com/` which embeds an iframe for `https://b.example.com/1`
* We open tab #1 to `https://example.org/`. Both `https://a.example.com/` and `https://b.example.com/1` responses point to origin policies with `"isolation": true` set.

(To simplify this example, assume at all times that we use "blocking" origin policies, i.e. no [async updates](https://github.com/WICG/origin-policy/issues/10) are involved.)

Now, things get fun:

* While we have tab #1 open, the server operator updates both `https://a.example.com/.well-known/origin-policy` and `https://b.example.com/.well-known/origin-policy` to set `"isolation": false`.
* Then, in tab #1, `https://a.example.com/` inserts a new iframe, pointing to `https://b.example.com/2`. Since the `https://b.example.com/` policy on the server has been updated, the response used for creating this second child iframe is no longer requesting origin isolation.
* This `https://b.example.com/2` iframe inserts an iframe for `https://c.example.com/`, which has no origin policy. Then `https://b.example.com/2` tries to `postMessage()` a `SharedArrayBuffer` to `https://c.example.com/`.

What happens?

The answer that the above specification plan gives is that the `postMessage()` fails. Within the browsing context group (i.e. tab #1), `https://b.example.com/` is in the list of isolated origins, so even though the `https://b.example.com/2` iframe was loaded with an origin policy saying `"isolation": false`, it still gets origin-isolated.

OK, let's go further.

* Now we open up a new tab to `https://example.org/`, tab #2. Because of the server update, the origin policies corresponding to the nested iframes for `https://a.example.com/` and `https://b.example.com/1` have `"isolation": false` set.
* The `https://a.example.com/` iframe tries to `postMessage()` a `SharedArrayBuffer` to the `https://b.example.com/1` iframe.

What happens this time?

This time, we are in a new browsing context group, with a new agent cluster map. So this time, the iframes are not origin-isolated, and the sharing succeeds.

This means that, if you want to "un-isolate" your origin, you'll need to do so via a new, disconnected browsing context group, separate from the isolated one.

## Design choices and alternatives considered

### Origin policy

Origin policy is a natural fit for this feature, as isolation causes origin-wide effects. Compared to a per-document mechanism, such as a HTTP header, this better ensures that all documents on the origin are aligned behind the decision to be isolated.

### The browsing context group scope of isolation

The [more complicated example](#more-complicated-example) above explains how the current specification plan works in tricky scenarios involving origin policies changing between loads of documents from that origin. In short, origins get isolated per browsing context group.

We considered two other possible models:

* Per-document: in this model, both `postMessage()`s would succeed, as even in the tab #1 browsing context group, we would put `https://b.example.com/2` and `https://c.example.com/` in the same agent cluster.
* Per-user agent: in this model, both `postMessage()`s would fail, because the user agent keeps a global list of isolated origins, that does not get cleared until it restarts.

The downside of the per-document model is that it leads to strange scenarios like `https://b.example.com/1` and `https://b.example.com/2` being in different agent clusters. That is, in the example above, the per-document model would put `https://b.example.com/1` in an agent clustered keyed by the `(https, b.example.com, 443)` origin, while it puts `https://b.example.com/2` in an agent cluster keyed by the `(https, example.com)` site. This is not desirable, as same-origin pages should not be isolated from each other by the origin isolation feature.

The downside of the per-user agent model is that it makes it very hard for web application developers to roll back origin isolation. They essentially have to communicate to the user to restart their browser.

### Not using feature/document policy

Given the choice of origin policy to deliver this, why did we choose a new top-level key (`"isolation"`) instead of using [feature policy](https://github.com/w3c/webappsec-feature-policy) or [document policy](https://github.com/w3c/webappsec-feature-policy/blob/master/document-policy-explainer.md)? Both of those are envisioned to be configurable on an origin-wide basis using origin policy, after all.

It turns out, both of these mechanisms are designed for dealing with embedded content, and as such have inheritance models that don't make much sense for origin isolation. If you want to isolate your own origin, that doesn't necessarily mean anything about how your subframes should behave—especially considering that the effects of feature/document policy are not limited to same-origin, or even same-site subframes.

### Usage hints

We've considered several previous designs for origin isolation, such as just `"isolated": true` to indicate isolation with no additional information, or something like `"isolation_implementation_preference": "process"` / `"thread"` / `"separate-heap"` to directly indicate the preferred implementation strategy. See some discussion in [#1](https://github.com/domenic/origin-isolation/issues/1).

A simple boolean gives the user agent too little information to make the right choices. Faced with multiple possible implementation strategies and constrained resources, how can it best decide what strategy to use? Currently Chrome uses heuristics, like "a page containing a password input probably needs side-channel protection". Making this more explicit can only benefit pages. In the end, user agents which don't find the extra information useful can ignore it.

On the other end of the spectrum, directly indicating implementation choices to the browser is overstepping a bit. It's unlikely a site author has enough information to confidently declare that they need a process; instead, they know things about their performance, memory, or security needs, which the browser can take into account.

### Opt-in, instead of implicit, origin isolation

Another potential road we explored was to try to find scenarios where a process-per-origin could be done transparently, without an explicit opt-in. This would be possible if:

* We [restricted `SharedArrayBuffer` to same-origin usage](https://github.com/whatwg/html/issues/4920) at all times. (It's currently unknown if doing this would be web-compatible.)
* Pages used the [`"document-domain"` feature policy](https://html.spec.whatwg.org/multipage/infrastructure.html#document-domain-feature) to turn off their own ability to use `document.domain`.

Under these conditions, origin-based process isolation could be done by user agents unobservably, similar to how site-based process isolation is done today.

In addition to the same concerns as stated above about how this doesn't give useful hints to the user agent, we avoided this approach because of the complexities involved in the use of the `"document-domain"` feature policy. In particular, it can only allow isolation if all documents in a browsing context group impose the policy upon themselves. Since feature policy is a per-document, not per-origin or per-browsing context group configuration, this is hard to guarantee. What if a new page joins the browsing context group, which does not impose the feature policy? There are workarounds, e.g. changing the semantics of `"document-domain"` so that its default allowlist depends on other pages in the browsing context group, but these end up making the `"document-domain"` feature policy no longer mean "disable `document.domain`", but instead have a bunch of other side effects and action at a distance.

## Adjacent work

### `COOP` + `COEP`

The [`Cross-Origin-Opener-Policy`](https://gist.github.com/annevk/6f2dd8c79c77123f39797f6bdac43f3e) and [`Cross-Origin-Embedder-Policy`](https://mikewest.github.io/corpp/) headers, when combined correctly, can be used to create a "cross-origin isolated" agent cluster: that is, an agent cluster which contains only same-origin content, or content which has (via [`Cross-Origin-Resource-Policy`](https://fetch.spec.whatwg.org/#cross-origin-resource-policy-header)) opted in to being treated as same-origin with the embedding agent cluster.

Origin isolation is complementary to the type of "cross-origin isolation" provided by `COOP` + `COEP`. In particular, `COOP` cuts off the opener relations for same-site cross-origin popups, isolating them from their opener. But `COEP` does not cut off the relationship with nested browsing contexts (e.g. iframes); it just ensures that they are OK being embedded. Thus, under a `COOP` + `COEP` world, `document.domain` still allows synchronous scripting to cross-origin same-site iframes, and `postMessage()` can still send `SharedArrayBuffer`s to such destinations. Whereas origin isolation cuts goes further and cuts off that possibility.

Stated in spec terms, `COOP` + `COEP` cross-origin isolated agent clusters are still site-keyed (although any cross-origin popups will not be placed in them, so with regard to popups, they are effectively origin-keyed). Whereas origin isolation changes the agent cluster to be origin-keyed.

However, see the next section...

### Automatic origin isolation via `COOP` + `COEP`

A recent idea, raised in [whatwg/html#5122](https://github.com/whatwg/html/issues/5122), is to have `COOP` + `COEP` automatically turn on origin isolation. That is, documents which include both headers will automatically change their agent cluster key to use the origin, instead of the site. As discussed previously, this has observable effects on `document.domain` and `SharedArrayBuffer`. Note that, in contrast to the problems explained above with using feature/document policy, this model works, since `COEP` is required to match for all descendant browsing contexts anyway, and `COOP` is consulted via the top-level browsing context.

We are optimistic about this approach for getting the observable effects of origin isolation. In particular, preventing the detrimental effects of `document.domain` is a long-time goal of the web standards and browser community, and tying that to headers like `COOP` + `COEP` and the various carrots they unlock seems like a great strategy.

However, we think that there remains room for this separate origin isolation proposal. In particular:

* Turning on `COOP` + `COEP` does not provide a clear signal that the origin desires origin isolation, or why it would desire origin isolation. As discussed at length above, these signals can be useful for the browser in making decisions on implementation strategies. For example, consider a site with 30 iframes containing ads, which wishes to achieve side-channel protection. With an explicit dedicated isolation signal, the top-level site could explain its desire for side-channel protection, and recieve a dedicated process, while letting the ad frames get coalesced into a single process. Whereas if `COOP` + `COEP` by itself automatically created a process-isolation signal, then the browser could only use heuristics to guess whether to allocate 2 processes, or 31 processes, or something in between.

* Deploying `COOP` + `COEP` could be a significant burden for sites, e.g. in terms of how it requires all of their embedded content to be updated with `CORP` headers. Some sites may not want to use any of the features enabled by `COOP` + `COEP` (e.g. `SharedArrayBuffer` or `process.measureMemory()`), and so be unmotivated to take on this burden. But they still might want origin isolation, e.g. of the event loop or heap isolation variety. In such cases, this origin-policy based approach would be much easier to deploy than `COOP` + `COEP`, since it does not require any embeddee cooperation.
