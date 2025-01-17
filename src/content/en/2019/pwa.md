---
part_number: II
chapter_number: 11
title: Progressive Web Apps
authors: [tomayac, jeffposnick]
reviewers: [hyperpress, ahmadawais]
---

## Introduction
Progressive Web Apps (PWA) are a new class of web applications, building on top of platform primitives 
like the [Service Worker APIs](https://developer.mozilla.org/en/docs/Web/API/Service_Worker_API). 
Service workers allow apps to support network-independent loading by acting as a network proxy, 
intercepting your web app's outgoing requests, and replying with programmatic or cached responses. 
Service workers can receive push notifications, and synchronize data in the background even when 
the corresponding app is not running. Additional, service workers — together with 
[Web App Manifests](https://developer.mozilla.org/en-US/docs/Web/Manifest) — 
allow users to install PWAs to their devices’ home screens.

Service workers were [first implemented in Chrome 40](https://blog.chromium.org/2014/12/chrome-40-beta-powerful-offline-and.html), 
back in December 2014, and the term Progressive Web Apps was 
[coined by Frances Berriman and Alex Russell](https://infrequently.org/2015/06/progressive-apps-escaping-tabs-without-losing-our-soul/) 
in 2015. As service workers are now finally [implemented in all major browsers](https://jakearchibald.github.io/isserviceworkerready/), 
we were wondering how many PWAs are actually out there, and how do they make use of these new technologies? 
Certain advanced APIs like [Background Sync](https://developers.google.com/web/updates/2015/12/background-sync) 
are currently still [only available on Chromium-based browsers](https://caniuse.com/#feat=background-sync), 
so as an additional question, we looked into which features these PWAs actually use.


All data in the HTTP Archive can be [queried through BigQuery](https://github.com/HTTPArchive/legacy.httparchive.org/blob/master/docs/bigquery-gettingstarted.md),
where multiple tables are available in the `httparchive` project. As these tables tend to get fairly big, 
they are partitioned, but multiple associated tables can be queried using the [wildcard symbol '*'](https://cloud.google.com/bigquery/docs/querying-wildcard-tables).
For our research, three families of tables are relevant:

- `httparchive.lighthouse.*`, which contains data about [Lighthouse](https://developers.google.com/web/tools/lighthouse/) runs. Note that Lighthouse data is only available for mobile pages.
- `httparchive.pages.*`, which contain the JSON-encoded parent documents’ HAR data.
- `httparchive.response_bodies.*`, which contains the raw response bodies of all resources and sub-resources of all sites in the archive.

## Service Workers
### Service Worker Registrations and Installability
The first metric we explore are service worker installations. Looking at the data exposed through 
feature counters in the HTTP Archive, we find that 0.44% of all desktop and 0.37% of all mobile pages 
register a service worker, and both curves over time are steeply growing. Now this might not look overly 
impressive, but taking traffic data from Chrome Platform Status into account, we can see that about 
[a service worker controlled 15% of all page loads](https://www.chromestatus.com/metrics/feature/timeline/popularity/990),
which can be interpreted as popular, high-traffic sites increasingly having started to embrace service workers. 

``` <timeseries chart of 11_01b> ```

**Figure 1:** Service Worker installation over time for desktop and mobile

Looking at Lighthouse data in the HTTP Archive, 1.56% of mobile pages are [installable](https://developers.google.com/web/tools/lighthouse/audits/install-prompt),
that is, they pass Lighthouse’s *user can be prompted to install the web app* test. 
Lighthouse tests currently are only available for mobile pages. To control the install experience, 
0.82% of all desktop and 0.94% of all mobile pages use the [`OnBeforeInstallPrompt` interface](https://w3c.github.io/manifest/#beforeinstallpromptevent-interface).

### Service Worker Events

In a service worker one can [listen for a number of events](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle):

- `install`, which occurs upon service worker installation. 
- `activate`, which occurs upon service worker activation. 
- `fetch`, which occurs whenever a resource is fetched.
- `push`, which occurs when a push notification arrives.
- `notificationclick`, which occurs when a notification is being clicked.
- `notificationclose`, which occurs when a notification is being closed.
`- message`, which occurs when a message sent via `postMessage()`  arrives.
- `sync`, which occurs when a Background Sync event occurs.

We have examined which of these events are being listened to by service workers we could find in the HTTP Archive. 
The results for mobile and desktop are very similar with `fetch`, `install`, and `activate` being the three 
most popular events, followed by `notificationclick` and `push`. If we interpret these results, offline use 
cases that service workers enable are the most attractive feature for app developers, far ahead of 
push notifications. Due to its limited availability, and less common use case, background sync doesn’t 
play a significant role at the moment. 

``` <bar chart of 11_03 mobile> ```

**Figure 2a:** Service worker events on mobile, ordered by decreasing frequency.

``` <bar chart of 11_03 desktop> ```

**Figure 2b:** Service worker events on desktop, ordered by decreasing frequency.

### Service Worker File Sizes
File size or lines of code are in general a bad proxy for the complexity of the task at hand. 
In this case, however, it is definitely interesting to compare (compressed) file sizes of service workers 
for mobile and desktop. The median service worker file on desktop is 895 bytes, whereas on mobile it’s 694 bytes. 
Throughout all percentiles desktop service workers are larger than mobile service workers. 
We note that these stats don’t account for dynamically imported scripts through the 
[`importScripts()`](https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/importScripts) method, 
which likely skews the results higher.

``` <distribution of 11_03b mobile> ```

**Figure 3a:** Percentiles of service worker file sizes on mobile.

``` <distribution of 11_03b desktop> ```

**Figure 3b:** Percentiles of service worker file sizes on desktop.

## Web App Manifests
### Web App Manifest Properties
The web app manifest is a simple JSON file that tells the browser about a web application 
and how it should behave when installed on the user's mobile device or desktop. A typical 
manifest file includes information about the app name, icons it should use, the start URL 
it should open at when launched, and more. Only 1.54% of all encountered manifests were 
invalid JSON, and the rest parsed correctly.

We looked at the different properties defined by the 
[specification](https://w3c.github.io/manifest/#webappmanifest-dictionary), 
and also considered non-standard proprietary properties. According to the Web App Manifest spec, 
the following properties are allowed: `dir`, `lang`, `name`, `short_name`, `description`, 
`icons`, `screenshots`, `categories`, `iarc_rating_id`, `start_url`, `display`, `orientation`, 
`theme_color`, `background_color`, `scope`, `serviceworker`, `related_applications`, and 
`prefer_related_applications`. The only property that we didn’t observe in the wild was 
`iarc_rating_id`, which is a string that represents the International Age Rating Coalition (IARC) 
certification code of the web application. It is intended to be used to determine which ages 
the web application is appropriate for. The proprietary properties we encountered still 
frequently were `gcm_sender_id` and `gcm_user_visible_only` from the legacy 
Google Cloud Messaging (GCM) service. Interestingly there’re almost no differences between mobile 
and desktop. On both platforms, however, there’s a long tail of properties that are not interpreted 
by browsers but that contain potentially useful metadata like `author` or `version`. We also found 
a non-trivial amount of mistyped properties, our favorite being `shot_name`. An interesting outlier 
is the `serviceworker` property which is standard, but not implemented by any browser vendor — 
nevertheless, it was found on 0.09% of all web app manifests used by mobile and desktop pages. 

```<bar chart of 11_04 mobile>```

**Figure 4a:** Web App Manifest properties ordered by decreasing popularity on mobile.

```<bar chart of 11_04 mobile>```

**Figure 4b:** Web App Manifest properties ordered by decreasing popularity on desktop.

### Display Values
Looking at the values developers set for the `display` property, it becomes immediately clear 
that they want PWAs to be perceived as “proper” apps that don’t  reveal their web technology origins. 
By choosing `"standalone"`, they make sure no browser UI is shown to the end-user. This is reflected 
by the majority of apps that make use of the `prefers_related_applications` property: more that 97% 
of both mobile and desktop applications do *not* prefer native applications.

```<11_04c mobile>```

**Figure 5a:** Values for the `display` property on mobile.

```<11_04c desktop>```

**Figure 5b:** Values for the `display` property on desktop.

### Category Values
The `categories` member describes the expected application categories to which the web application belongs. 
It is only meant as a hint to catalogs or stores listing web applications, and it is expected that 
these will make a best effort to find appropriate categories (or category) under which to list the 
web application. There were not too many manifests that made use of the property, but it is 
interesting to see the shift from *shopping* being the most popular category on mobile to *business*, 
*technology*, and *web* (whatever may be meant with that) on desktop that share the first place evenly.

```<11_04d mobile>```

**Figure 6a:** Values for the `categories` property on mobile.

```<11_04d desktop>```

**Figure 6b:** Values for the `categories` property on desktop.

### Icon Sizes
Lighthouse [requires](https://developers.google.com/web/tools/lighthouse/audits/manifest-contains-192px-icon) 
at least an icon sized 192×192, but common favicon generation tools create a plethora of other sizes, too. 
Lighthouse’s rule is probably the culprit for 192×192 being the most popular choice of icon size on both 
desktop and mobile, despite [Google’s documentation](https://developers.google.com/web/fundamentals/web-app-manifest#icons) 
additionally explicitly recommending 512×512, which doesn’t show as a particularly prominent option.

```<11_04f mobile>```

**Figure 7a:** Popular icon sizes on mobile.

```<11_04f desktop>```

**Figure 7b:** Popular icon sizes on desktop.

### Orientation Values
The valid values for the `orientation` property are [defined](https://www.w3.org/TR/screen-orientation/#dom-orientationlocktype)
in the Screen Orientation API specification. Namely there are `"any"`, `"natural"`, `"landscape"`,  
`"portrait"`, `"portrait-primary"`, `"portrait-secondary"`, `"landscape-primary"`, and `"landscape-secondary"`. 
Portrait orientation is the clear winner on both platforms, followed by any orientation.

```<11_04g mobile>```

**Figure 8a:** Popular orientation values on mobile.

```<11_04g desktop>```

**Figure 8b:** Popular orientation values on desktop.

## Workbox
Workbox is a set of libraries that help with common service worker use cases. For instance, Workbox has tools 
that can plug in to your build process and generate a manifest of files, which are then precached by your 
service worker. Workbox includes libraries to handle runtime caching, request routing, cache expiration, 
background sync, and more.

Given the low-level nature of the service worker APIs, many developers have turned to Workbox as a way of 
structuring their service worker logic into higher-level, reusable chunks of code. Workbox adoption is also 
driven by its inclusion as a feature in a number of popular JavaScript framework starter kits, like 
[`create-react-app`](https://create-react-app.dev/) and [Vue's PWA plugin](https://www.npmjs.com/package/@vue/cli-plugin-pwa).

The HTTP Archive shows that, out of the total population of sites that register a service worker, 
12.71% of them are using at least one of the Workbox libraries. This percentage is roughly consistent 
across desktop and mobile, with a slightly lower percentage (11.46%) on mobile compared to desktop (14.36%).
