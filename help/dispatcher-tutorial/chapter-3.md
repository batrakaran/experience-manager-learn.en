---
title: Chapter 3 - Advanced Caching Topics
seo-title: AEM Dispatcher Cache Demystified - Chapter 3 - Advanced Caching Topics
description: Chapter 3 of the AEM Dispatcher Cache Demystified tutorial covers how to overcome the limitations discussed in Chapter 2.
seo-description: Chapter 3 of the AEM Dispatcher Cache Demystified tutorial covers how to overcome the limitations discussed in Chapter 2.
---

# Chapter 3 - Advanced Caching Topics

*"There are only two hard things in Computer Science: cache invalidation and naming things."*

— PHIL KARLTON

## Overview

This is Part 3 of a three part – series to caching in AEM. Where the first two parts focused on plain http caching in the Dispatcher and what limitations there are. This part discusses some ideas on how to overcome these limitations.

## Caching in General

[Chapter 1](chapter-1.md) and [Chapter 2](chapter-2.md) of this series focused mainly on the Dispatcher. We have explained the basics, the limitations and where you need to make certain trade-offs.

The caching complexity and intricacies are not issues unique to the Dispatcher. Caching is difficult in general.

Having the Dispatcher as your only tool in your toolbox would actually be a real limitation.

In this chapter we want to broaden our view on caching further and develop some ideas how you can overcome some of the Dispatchers shortcomings. There is no silver bullet. You will have to make tradeoffs in your project. Remember, that with caching and invalidation accuracy always comes complexity, and with complexity there comes the possibility of errors.

You will need to make trade-offs in these areas,

* Performance and latency
* Resource consumption / CPU Load / Disk Usage
* Accuracy / Currency / Staleness / Security
* Simplicity / Complexity / Cost / Maintainability / Error-proneness

These dimensions are interlinked in a rather complex system. There is no simple if-this-then-that. Making a system simpler can make it faster or slower. It can lower your development costs, but increase costs at the helpdesk, e.g., if customers see stale content or complain about a slow website. All I am saying here is that these factors need to be considered and balanced against each other. But I think you already have the idea, that there is no silver bullet or a single "best-practice" - only a lot of bad practices and a few good ones.

## Chained Caching

### Overview

#### Data Flow

Delivering a page from a server to a client's browser crosses a multitude of systems and subsystems. If you look carefully, there is a number of hops data needs to take from the source to the drain, each of which is a potential candidate for caching.

![Data flow of a typical CMS application](assets/chapter-3/data-flow-typical-cms-app.png)
*Data flow of a typical CMS application*

Let's start our journey with a piece of data that sits on a hard disk and that needs to be displayed in a browser.

#### Hardware and Operating System

First, the hard disk drive (HDD) itself has some built in cache in the hardware. Second, the operating system that mounts the hard disk, uses free memory to cache frequently accessed blocks to speed up access.

#### Content Repository

Next level is the CRX or Oak – the document database used by AEM. CRX and Oak divide the data into segments that can be cached in memory as well to avoid slower access to the HDD.

#### Third Party Data

Most larger Web installations have 3rd party data as well; data coming from a product information system, a customer relation management system, a legacy database or any other arbitrary web service. This data does not need to be pulled from the source any time it is needed – especially not, when it is known to change not too frequently. So, it can be cached, if it is not synchronized in the CRX database.

#### Business Layer – App / Model

Usually your template scripts do not render the raw content coming from CRX via the JCR API. Most likely you have a business layer in between that merges, calculates and/or transforms data in a business domain object. Guess what – if these operations are expensive, you should consider caching them.

#### Markup Fragments

The model now is the base for the rendering of the markup for a component. Why not cache the rendered model as well?

#### Dispatcher, CDN and other Proxies

Off goes the rendered HTML-Page to the Dispatcher. We already discussed that the main purpose of the Dispatcher is to cache HTML pages and other web resources (despite its name). Before the resources reach the browser, it may pass a reverse proxy – that can cache and a CDN – that also is used for caching. The client may sit in an office, that grants web access only via a proxy – and that proxy might decide to cache as well to save traffic.

#### Browser Cache

Last but not least – the browser caches too. This is an easy overlooked asset. But it is the closest and fastest cache you have in the caching chain. Unfortunately – it is not shared between users – but still between different requests of one user.

### Where to Cache and Why

Now – that is a long chain of potential caches. And we all have faced issues where we have seen outdated content – and taking into account how many stages there are, it's a miracle that most of the time it's working at all.

But where in that chain does it make sense to cache at all? At the beginning? At the end? Everywhere? It depends… and it depends on huge number of factors and even two resources in the same website might have a different answer to that question.

To give you a rough idea of what factors you might take into consideration,

**Time to live** – If objects have a short inherent live time (traffic data might have a shorter live than weather data) it might not be worth caching.

**Production Cost –** How expensive (in terms of CPU cycles and I/O) is the re-production and delivery of an object. If it's cheap caching might not be necessary.

**Size** – Large objects require more resources to be cached. That could be a limiting factor and must be balanced against the benefit.

**Access frequency** – If objects are accessed rarely, caching might not be effective. They would just go stale or be invalidated before they are access the second time from cache. Such items would just block memory resources.

**Shared access** – Data that is used by more than one entity should be cached further up the chain. Actually, the caching chain is not a chain, but a tree. One piece of data in the repository might be used by more than one model. These models in turn can be used by more than one render script to generate HTML fragments. These fragments are included in multiple pages which are distributed to multiple users with their private caches in the browser. So "sharing" does not mean sharing between people only, rather between pieces of software. If you want to find a potential "shared" cache, just track back the tree to the root and find a common ancestor – that's where you should cache.

**Geospatial distribution** – If your users are distributed over the world, using a caching distributed network of caches might help reduce latency.

**Network bandwidth and latency** – Speaking of latency, who are your customers and what kind of network are they using? Maybe your customers are mobile customers in under developed market using 3G connection of and older generation smartphones? Consider creating smaller object and cache them in the browser caches.

That list by far is not comprehensive, but we think you get the idea by now.

### Basic Rules for Chained Caching

Again – caching is hard. Let us share some ground rules, that we have extracted from previous projects that can help you avoid issues in your project.

#### Avoid Double Caching

Each of the layers introduced in the last chapter provide some value in the caching chain. Either by saving computing cycles or by bringing data closer to the consumer. It is not wrong to cache a piece of data in multiple stages of the chain – but you should always consider what the uplift and the costs of the next stage is.

#### Mixing Invalidation Strategies

There are three basic invalidation strategies:

**TTL, Time to Live:** An object expires after a fixed amount of time (e.g., 2 hours from now)

**Expiration Date:** The object expires at defined time in the future (e.g., 5:00 PM on June 10, 2019)

**Event based:** The object is invalidated explicitly by an event that happened in the platform (e.g., when a page is changed and activated)

Now you can use different strategies on different cache layers, but there are a few "toxic" ones.

#### Event Based Invalidation

![Pure Event based invalidation](assets/chapter-3/event-based-invalidation.png)
*Pure Event based invalidation: Invalidate from the inner cache to the outer layer*

Pure event-based invalidation is the easiest one to comprehend, easiest to get theoretically right and practically wrong but the most accurate one. The caches are invalidated one by one after the object has changed.

You just need to keep one rule in mind - always invalidate from the inside to the outside cache. If you invalidated an outer cache first, it might re-cache stale content from an inner one. Don't make any assumptions at what time a cache is fresh again – make it sure. Best, by triggering the invalidation of the outer cache after invalidating the inner one.

Now, that's the theory. But in practice there are a number of gotchas. The events must be distributed – potentially over a network.

#### Auto - Healing

With event-based invalidation, you should have a contingency plan. What if an invalidation event is missed? A simple strategy could be to invalidate or purge after a certain amount of time. So - you might have missed that event and now serve stale content. But your objects also have an implicit TTL of several hours (days) only. So eventually the issue auto-heals itself.

#### Pure TTL-based invalidation

![Unsynchronized TTL based invalidation](assets/chapter-3/ttl-based-invalidation.png)
*Unsynchronized TTL based invalidation*

That one also is a quite common scheme. You stack several layers of caches, each one entitled to serve an object for a certain amount of time.

It's easy to implement. Unfortunately, it's hard to predict the effective life span of a piece of data.

![Outer chace prolonging the life pan of an inner object](assets/chapter-3/outer-cache.png)
*Outer cache prolonging the life span of an inner object*

Consider the illustration above. Each caching layer introduce a TTL of 2 min. Now – the overall TTL must 2 min too, right? Not quite. If the outer layer fetches the object just before it would get stale, the outer layer actually prolongs the effective live time of the object. The effective live time can be between 2 and 4 minutes in that case. Consider you agreed with your business department, one day is tolerable – and you have four layers of caches. The actual TTL on each layer must not be longer than six hours… increasing the cache miss-rate…

We are not saying it is a bad scheme. You just should know its limits. And it's a nice and easy strategy to start with. Only if your site's traffic increases you might consider a more accurate strategy.

![Synchronizing Invalidation time by setting a specific date](assets/chapter-3/syncronize-invalidation-by-date.png)
*Synchronizing Invalidation time by setting a specific date*

#### Expiration Date Based Invalidation

You get a more predictable effective life time, if you are setting a specific date on the inner object and propagating that to the outside caches.

 ![Synchronizing expiration dates](assets/chapter-3/synchronize-expiration-dates.png)
*Synchronizing expiration dates*

However, not all caches are able to propagate the dates. And it can become nasty, when the outer cache aggregates two inner objects with different expiration dates.

#### Mixing Event-based and TTL-based invalidation

![Mixing event-based and TTL-based strategies](assets/chapter-3/mixing-event-ttl-strategies.png)
*Mixing event-based and TTL-based strategies*

Also a common scheme in the AEM world is to use event based invalidation at the inner caches (e.g., in-memory caches where events can be process in near real time) and TTL-based caches on the outside – where maybe you don't have access to explicit invalidation.

In the AEM world you would have an in-memory cache for business objects and HTML fragments in the Publish systems, that is invalidated, when the underlying resources change and you propagate this change event to the dispatcher which also works event-based. In front of that you would have for example a TTL-based CDN.

Having a layer of (short) TTL-based caching in front of a Dispatcher could effectively soften a spike that usually would occur after an auto-invalidation.

#### Mixing TTL – and Event-Based Invalidation

 ![Mixing TTL – and event-based Invalidation](assets/chapter-3/mixing-event-ttl-strategies.png)
*Toxic: Mixing TTL – and event-based Invalidation*

This combination is toxic. Never place and event-based cache after a TTL or Expiry-based cached. Remember that spill-over effect that we had in the "pure-TTL" strategy? The same effect can be observed here. Only that the invalidation event of the outer cache already has happened might not happen again. Ever – expanding the life span of you cached object until infinity.

![TTL-based and event-based combined: Spill-over to infinity](assets/chapter-3/mixing-event-ttl-strategies.png)
*TTL-based and event-based combined: Spill-over to infinity*

## Partial Caching and In-Memory Caching

### Words of Warning

#### Respect Access Control

The techniques described here are quite powerful and a must have in each AEM developer's toolbox. But don't get too excited, use them wisely. By storing an object in a cache and sharing it to other users in follow-up requests actually means circumventing access control. That usually is not an issue on public-facing websites but can be when a user needs to login before getting access.

Consider you store a sites main menu's HTML in an in-memory cache to share it between various pages. You are sharing that same structure also with all users – but maybe there are some items in it that are reserved for a certain group of users only. In that case caching can get a bit more complex.

#### Only Cache Custom Business Objects

If any – that's the most important piece of advice, we can give you:

Only cache objects that are yours, that are immutable, that you built yourself, and that have no outgoing reference.

What does that mean?

1. You don't know about the intended live cycle of other people's objects. Consider you get behold of a reference to a request object and decide to cache it. Now, the request is over, and the servlet container wants to recycle that object for the next incoming request. In that case someone else is changing the content that you thought you had exclusive control over. Don't dismiss that – We have seen something like that happening in a project. Customer were seeing other customers data instead of their own.

2. As long as an object is referenced by a chain of other references it cannot be removed from the heap. If you retain a supposedly small object in your cache that references, let's say a 4MB representation of an image you will have a good chance to get trouble with lacking memory. Caches are supposed to be based on weak references. But – weak references do not work as you might expect. That is the absolute best way to produce a memory leak and end in an out-of-memory-error. And – you don't know what the size of retained memory of the foreign objects are, right?

3. Especially in Sling, you can adapt (almost) each object to each other. Consider you put a resource into the cache. The next request (with different access rights), fetches that resource and adapts it into a resourceResolver or a session to access other resources that he would not have access to.

4. Just to make sure, you also must not create a resourceWrapper that wraps the resource into your "own" business object. That's basically the same as the last item.

5. If you want to cache, create your own objects by copying primitive data into your objects. You might want to link your own objects by references. That is fine – but only objects that you just created in the very same request – and no objects that were requested from somewhere else (even if it's your object's space). Copying objects is the key. And make sure to purge the whole structure of linked objects at once and avoid incoming and outgoing references.

6. Yes – and keep your objects immutable. Private properties, only and no setters.

That's a lot of rules, but it's worth following them. Even if you are experienced and super smart and have everything under control. The young colleague in your project just graduated from university. He doesn't know of all these pitfalls. Help him keep it simple and comprehensible.

### Tools and libraries

This series is about understanding concepts and empowering you to build an architecture that best fits your use case.

We are not promoting any tool in particular. But give you hints how to evaluate them. For example, AEM has a simple built in cache with a fixed TTL since version 6.0. Shall you use it? Probably not on publish where an event-based cache follows in the chain (hint: The Dispatcher). But it might by a decent choice for an Author. There is also an HTTP cache by Adobe ACS commons that might be worth considering.

Or you build your own, based on a mature caching framework like Ehcache (see references). With that you could also be used to cache Java objects.

In some simple cases you might also get along with using concurrent hash maps –you will quickly see limits here – either in the tool or in your skills. Concurrency is as hard to master as naming and caching.

#### References

* ACS Commons http Cache - [https://adobe-consulting-services.github.io/acs-aem-commons/features/http-cache/index.html](https://adobe-consulting-services.github.io/acs-aem-commons/features/http-cache/index.html)
* Ehcache caching framework - [https://www.ehcache.org](https://www.ehcache.org)

### Basic terms

We will not go into caching theory too deep here, but we feel obliged to provide a few buzz-words, so that you have a good jump start.

#### Cache Eviction

We talked about invalidation and purging a lot. Cache eviction is related to these terms: After an entry it is evicted it is not available anymore. But eviction happens not when an entry is outdated, but when the cache is full. Newer or "more important" items push older or less important ones out of the cache. Which entries you will have to sacrifice is a case to case decision. You might want to evict the oldest ones or those which have been used very rarely or last accessed a long time.

#### Preemptive Caching

Preemptive Caching means re-creating the entry with fresh content in the moment it is invalidated or considered outdated. Of course - you would do that only with a few resources, that you are sure are accessed frequently and immediately. Otherwise you would waste resources on creating cache-entries that might never be requested. By creating cache-entries preemptively you could reduce the latency of the first request to a resource after cache invalidation.

#### Cache Warming

Cache warming is closely related to preemptive caching. Though you wouldn't use that term for a live system. And it is less time constrained than the former. You don't re-cache immediately after invalidation, but you gradually fill the cache when time permits. For example, you take out a Publish / Dispatcher leg from the load balancer to update it. Before re-integrating it, you automatically crawl the most frequently accessed pages to get them into the cache again. Or maybe you want to also cache some less frequently accessed pages in times where your system is idle to decrease latency when they actually are accessed by real requests. When the cache is "warm" – filled adequately you re-integrate the leg into the load balancer. Or maybe you re-integrate the leg at once, but you throttle the traffic to that leg so that it has a chance to warm it's caches by regular usage.

#### Cache Object Identity, Payload, Invalidation Dependency and TTL

Generally speaking, a cached object or "entry" has five major properties,

#### Key

This is the identity is the property by which you identify and object. Either to retrieve its payload or to purge it from the cache. The dispatcher for example uses the URL of a page as the key. Note, that the dispatcher does not use the pages paths. This is not sufficient to tell different renderings apart. Other caches might use different keys. We will see some examples later.

#### Value / Payload

That is the object's treasure chest, the data that you want to retrieve. In case of the dispatcher it's the files contents. But it can also be a Java object tree.

#### TTL

We covered the TTL already. The time after which an entry is considered stale and should not be delivered any longer.

#### Dependency

This relates to event-based invalidation. What original data is that object depending on? In Part I, we already said, that a true and accurate dependency tracking is too complex. So, with our knowledge of the system we approximate the dependencies with a model. We invalidate enough objects to purge stale content… and maybe inadvertently more than would be required. But yet we try to keep below "purge everything".

Which objects are depending on what others is genuine in each single application. We will give you some examples on how to implement a dependency strategy later.

### HTML Fragment Caching

![Re-using a rendered fragment on different pages](assets/chapter-3/re-using-rendered-fragment.png)
*Re-using a rendered fragment on different pages*

HTML Fragment Caching is a mighty tool. The idea is to cache the HTML markup that was generated by a component in an in-memory cache. You may ask, why should I do that? I am caching the whole page's markup in the dispatcher anyway – including that component's markup. We agree. You do - but once per page. You are not sharing that markup between the pages.

Imagine you are rendering a navigation on top of each page. The markup looks the same on each page. But you are rendering it over and over again for each page, that is not in the Dispatcher. And remember: After auto-invalidation all pages need to be re-rendered. So basically, you are running the same code with the same results hundreds of times.

From our experience, rendering a nested top navigation is a very expensive task. Usually you traverse a good portion of the document tree to generate the navigation items. Even if you only need the navigation title and the URL – the pages have to be loaded into memory. And here they are clogging precious resources. Over and over again.

But the component is shared between many pages. And sharing something is an indicates using a cache. So – what you would want to do is check if the navigation component already has been rendered and cached and instead of re-rendering just emit the caches value.

There are two wonderful niceties of that scheme easily missed:

1. You are caching a Java String. A String does not have any outgoing references and it is immutable. So, considering the warnings above that is super safe.

2. Invalidation also is super easy. Whenever anything changes your website, you want to invalidate this cache entry. Re-building is relatively cheap, as it needs to be performed only once and then is reused by all the hundreds of pages.

That is a big relief to your Publish servers.

### Implementation of Fragment Caches

#### Custom Tags

In the old days, where you used JSP as a templating engine it was quite common to use a custom JSP tag wrapping around the components rendering code.

```
<!-- Pseudo Code -->

<myapp:cache
  key=' ${info.homePagePath} + ${component.path}'
  cache='main-navigation'
  dependency='${info.homePagePath}'>

… original components code ..

</myapp:cache>

```

The custom tag than would capture its body and write it into cache or prevent execution of its body and output the cache-entry's payload instead.

The "Key" is the components path that it would have on the homepage. We don't use the component's path on the current page, as this would create one cache entry per page – that would contradict our intention to share that component. We are also not using just the components relative path (`jcr:conten/mainnavigation`) as this would prevent us from using different navigation components in different sites.

"Cache" is an indicator where to store the entry. You usually have more than one cache where you store items into. Each one of which might behave a bit different. So, it's good to differentiate what is stored – even if in the end it's just strings.

"Dependency" this is what the cache entry depends on. The "main-navigation" cache might have a rule, that if there is any change below the node "dependency", the according entry must be purged. So – your cache implementation would need to register itself as an event listener in the repository to be aware of changes and then apply the cache-specific rules to find out what needs to be invalidated.

The above was just an example. You can of course choose to have a tree of caches. Where the first level is used to separate sites (or tenants) and the second level then branches out into types of contents (for instance "main-navigation") – that could spare you adding the home pages path as in the example above.

By the way – you can also use this approach with more modern HTL based components. You would then have a JSP wrapper around your HTL script.

#### Component Filters

In a pure HTL approach, you could try to build the fragment cache with a Sling component filter. We haven't experienced this in the wild yet, but that is the approach we would take on that issue.

#### Sling Dynamic Include

The fragment cache is used if you have something constant (the navigation) in the context of a changing environment (different pages).

But you may have the opposite as well, a relatively constant context (a page that rarely changes) and some ever-changing fragments on that page (e.g., a live ticker).

In this case, you might give Sling Dynamic Includes a chance. In essence, this is a component filter, which wraps around the dynamic component and instead of rendering the component into the page it creates a reference. This reference can be an Ajax call – so that the component is included by the browser and thus the surrounding page can statically be cached. Alternatively Sling Dynamic Include can generate an SSI directive (Server Side Include) – this directive would be executed in the Apache server. You can even use ESI – Edge Side Include directives if you leverage Varnish or a CDN that supports ESI scripts.

 ![Sequence Diagram of a Request using Sling Dynamic Include](assets/chapter-3/sequence-diagram-sling-dynamic-include.png)
*Sequence Diagram of a Request using Sling Dynamic Include*

The SDI documentation says you should disable caching for URLs ending in "*.nocache.html", which makes sense – as you are dealing with dynamic components.

You might be tempted to _not_ disable the cache to get a fragment-cache similar to the one we described in the last chapter: Pages and component fragments equally and independently are cached in the dispatcher and stitched together by the SSI script in the Apache server when the page is requested. Doing so, you could implement shared components like the main navigation (given you always use the same component URL).

That should work – in theory. But we advise not to do that: You would lose the ability to bypass the cache for the real dynamic components. SDI is configured globally and the changes you would make for your "poor-mans-fragment-cache" would also apply to the dynamic components.

We advise you to carefully study the SDI documentation. There are a few other limitations, but SDI is a valuable tool in some cases.

#### References

* How to write custom JSP tags - [https://docs.oracle.com/cd/E11035\_01/wls100/taglib/quickstart.html](https://docs.oracle.com/cd/E11035_01/wls100/taglib/quickstart.html)
* Creating and using component filters - [https://www.slideshare.net/connectwebex/prsentation-dominik-suess](https://www.slideshare.net/connectwebex/prsentation-dominik-suess)
* Description of Sling Dynamic Includes - [https://sling.apache.org/documentation/bundles/dynamic-includes.html](https://sling.apache.org/documentation/bundles/dynamic-includes.html)
* Setting up Sling Dynamic Includes in AEM - [https://helpx.adobe.com/experience-manager/kt/platform-repository/using/sling-dynamic-include-technical-video-setup.html](https://helpx.adobe.com/experience-manager/kt/platform-repository/using/sling-dynamic-include-technical-video-setup.html)


#### Model Caching

![Model based caching: One business object with two different renderings](assets/chapter-3/model-based-caching.png)
*Model based caching: One business object with two different renderings*

Now you almost know everything that needs to be known about in-memory caching.

Let us revisit the case with the navigation again. We were assuming, that each page would require that same markup of the navigation.

But maybe, that is not the case. You might want to render different markup for the item in the navigation that represents the current page. Or maybe the navigation includes some an image or a claim to introduce to the area the page belongs to (Products, About Us, … could be represented by a different icon in the navigation).

First of all – you should try to solve that via CSS in the browser – if possible. If not – try to use JavaScript. We can safely assume, that all browsers today support JavaScript. If that for some reason is not possible, and the markup cannot be harmonized by no means, you still have one option.

The business object still might be the same for two different renderings. In case of the navigation, the business object actually would be an object graph representing the nodes in the tree. That graph than could easily be stored in an in-memory cache. Remember though, that this graph must not contain any object or reference any object that you did not create yourself.

#### Caching in the Browser

We touched the importance of caching in the browser already, and there are many good tutorials out there. In the end - for the browser - the Dispatcher is just a webserver that follows the HTTP protocol.

However – despite the theory - we have gathered some tid bits of knowledge which we want to share.

In essence, browser caching can be leveraged in two different ways,

1. The browser has a resource cached of which it knows the exact expiry date. In that case it does not request the resource again.

2. The browser has a resource, but it is not sure if it is still valid. In that case it would ask the webserver (the Dispatcher in our case). Please give me the resource if it was modified since you last delivered it. If it hasn't changed, the server answers with "304 – not changed" and only the meta data was transmitted.

#### Debugging

If you are optimizing your Dispatcher settings for browser caching it is extremely useful to use a desktop proxy server between your browser and the webserver. We prefer "Charles Web Debugging Proxy" by Karl von Randow.

Using Charles, you can read the requests and responses, which are transmitted to and from the server. And – you can learn a lot about the HTTP protocol. Modern browsers also offer some debugging capabilities, but the features of a desktop proxy are unprecedented. You can manipulate the data transferred, throttle the transmission, replay single requests and much more. And the user interface is clearly arranged and quite comprehensive.

What we usually do normally is use the website and check in the proxy if the number of the static requests (to /etc/…) is getting smaller over time – as these should be in the cache and not be requested any longer. Some built-in debuggers still show these requests with 0 ms. Which is ok and accurate but not quite clear.

You can then drill-down and check the headers of the transferred files to see, for example, if the "Expires" http headers are correct. You could replay requests with if-modified-since headers set to see if the server correctly responds with a 304 or 200 response code. You can observe the timing of asynchronous calls and you could also test your security assumptions to a certain degree. Remember we told you to not accept all selectors that are not are not explicitly expected? Here you can play around with the URL and the parameters and see if your application behaves well.

There is only one thing we ask you not to do, when you are debugging your cache:

Do not reload pages in the browser!

A browser reload, a "Simple Reload" and a "Shift Reload" is not the same as a normal page request. A simple reload request sets a header

```
Cache-Control: max-age=0
```

And a Shift-Reload (holding down the "Shift" key while clicking the reload button) set a request header

```
Cache-Control: no-cache
```

Both headers have similar yet slightly different effects – but most importantly, they differ completely from a normal request when you open a URL from the URL slot or by using links on the site. Normal browsing does not set Cache-Control headers but probably an if-modified-since header.

So, if you want to debug the normal browsing behavior, you should do exactly that: Browse normally. Using the reload button of your browser is the best way to not see errors in your configuration.

Use your Charles Proxy to see what we are talking about. Yes – and while you have it open – you can replay the requests right there. No need to reload from the browser.

## Performance Testing

By using a proxy, you get a sense of the timing behavior of your pages. Of course, that is by far not a performance test.  A performance test would require a number of clients requesting your pages in parallel.

A common mistake, we have seen too often, is that the performance test only includes a super small number of pages and these pages are delivered from the Dispatcher cache only.

If you are promoting your application to the live system, the load is completely different, from what you have tested.

On the live system, not that small number of equally distributed pages is requested but much larger number and very unevenly distributed. And - of course – they cannot be served 100% from cache – there are invalidation requests coming from the Publish system that are auto-invalidating a huge portion of your precious resources.

Ah yes – and when you are rebuilding your Dispatcher cache, you will find out, that the Publish system also behaves quite differently, depending on whether you request only a handful of pages – or a bigger number. Even if all pages are similarly complex – their number plays a role. Remember what we said about chained caching? If you always request the same small number of pages, chances are good, that the according blocks with the raw data is in the hard drives cache or the blocks are cached by the operating system. Also, there is a good chance, that the Repository has cached the according segment in its main memory. So re-rendering is significantly faster than when you had other pages evicting each other now and then from various caches.

Caching is hard, and so is testing of a system that relies on caching. So, what can you do to have a more accurate real-life scenario?

We think you would have to conduct more than one test, and you would have to provide more than one performance index as a measure of the quality of your solution.

If you already have an existing website, measure the number of requests and how they are distributed. Try to model a test that uses a similar distribution. Adding some randomness couldn't hurt. You don't have to simulate a browser which would load static resources like JS and CSS – those don't really matter. They are cached in the browser or in the Dispatcher eventually and they don't add up to the load significantly. But embedded images do matter. Find their distribution in old log files as well and model a similar request pattern.

Now conduct a test with your Dispatcher not caching at all. That is your worst-case scenario. Find out at what peak load your system is getting unstable under this worst of conditions. You might also make it worse by taking out a few Dispatcher/Publish legs if you want.

Next conduct the same test with all required cache settings to "on". Ramp up your parallel requests slowly to warm the cache and see how much your system can take under the best conditions.

An average case scenario would be to run the test with the Dispatcher enabled but also with some invalidations happening. You can simulate that by touching the statfiles by a cronjob or sending the invalidation requests in irregular intervals to the Dispatcher. Don't forget to also purge some of the non-auto invalidated resources every now and then.

You can vary the last scenario by increasing the invalidation requests and by increasing the load.

That is a bit more complex than just a linear load test – but gives a lot more confidence into your solution.

You might shy away from the effort. But at lease conduct a worst–case test on the Publish system with a larger number of pages (equally distributed) to see the limits of the system. Make sure, you interpret the number of the best-case scenario correctly and provision your systems with enough headroom.