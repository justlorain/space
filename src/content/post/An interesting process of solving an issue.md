---
title: "An interesting process of solving an issue"
description: "This article focuses on the process of solving an issue for Hertz."
publishDate: "28 Nov 2022"
tags: ["webdev", "opensource", "beginners", "go"]
---

## Foreword

This article focuses on the process of solving an issue for [Hertz](https://github.com/cloudwego/hertz). The link to the issue is [here](https://github.com/cloudwego/hertz/issues/399).

## Issue Analysis

The originator of this issue was a user who wanted to migrate from FastHTTP to Hertz, but for some reason couldn't get all of his code based on Hertz at once, so now he's trying to get Hertz to replace some of his previous FastHTTP code.

Now he had some problems with the FastHTTP Cookie docking with Hertz, so we can take a look at the core code he sent.

```Go
// FastHTTP code
cookie := FastHTTPCtx.Response.Header.Peek(fasthttp.HeaderCookie)
// Hertz code
cookieNew := protocol.AcquireCookie()
cookieNew.Parse(string(cookie))
HertzCtx.Response.Header.SetCookie(cookieNew)
```

If we take a quick look, we can see that the user is trying to get the value of the cookie field (`fasthttp.`*`HeaderCookie`*) in the response header using FastHTTP's Peek function. Then use Hertz to get an empty cookie from the pool and use the `Parse` function to parse the FastHTTP cookie field and set it into the empty Cookie, and finally set it into the response header using Hertz's `SetCookie` function.

There are two bugs in this code. The first one is very easy to find. You can see the user `Peek` the HTTP response header. However, the `cookie` field is not in the response header. Only the HTTP request header has the `cookie` field. So you can't actually get the value here, the correct code would look like this:

- Get the `cookie` field of the request header

```Go
cookie := FastHTTPCtx.Request.Header.Peek(fasthttp.HeaderCookie)
```

- Get the `set-cookie` field of the response header

```Go
cookie := FastHTTPCtx.Response.Header.Peek(fasthttp.HeaderSetCookie)
```

The second bug, which is a little harder to spot, is that we `Peek` all the values in the cookie field, which contains many cookies, but the `Parse` function can't parse them all at once and set them to the empty allocated cookie.

To verify this bug, I took a snippet of the value of the `cookie` field that I brought when I visited github, and then parsed it with the `Parse` function. Finally, we printed the values set on the cookie, and we can see that the last cookie contains only the value of the first cookie.

```Go
package main

import "github.com/cloudwego/hertz/pkg/protocol"

func main() {
   cookie := protocol.AcquireCookie()
   defer protocol.ReleaseCookie(cookie)
   _ = cookie.Parse("_device_id=12ion3sbdfs6c06dde9f4oroiu5nd32dsfio9; _octo=GH1.1.199263436.16523146515; user_session=21tB__7-tTVwYCm0unrewjbSdX7Bp0UU59d9BWy-MQgFDUEJRN; __Host-user_session_same_site=21wR__7-tTVwYCm0uasdibqwyD7Bp0UU59d9BWy-MQgFDUEJRN; logged_in=yes; dotcom_user=justlorain; preferred_color_mode=light; has_recent_activity=1;")
   println(cookie.String()) //_device_id=12ion3sbdfs6c06dde9f4oroiu5nd32dsfio9
}
```

## Solution

We can `Parse` and `SetCookie` one by one using the `VisitAllCookie function`, as follows:

```Go
FastHttpctx.Response.Header.VisitAllCookie(func(key, value []byte) {
   cookie := protocol.AcquireCookie()
   defer protocol.ReleaseCookie(cookie)
   _ = cookie.ParseBytes(value)
   HertzCtx.Response.Header.SetCookie(cookie)
})
```

## Summary

That's it for analyzing and resolving this issue. I hope you found it helpful and I'm very welcome to try out the [Hertz HTTP framework](https://github.com/cloudwego/hertz). You can find more information about Hertz in my previous posts. And welcome to some very valuable issues for Hertz, the CloudWeGo community members will do their best to answer.

## Reference List

- https://github.com/cloudwego/hertz

- https://github.com/cloudwego/hertz/issues/399

- https://github.com/valyala/fasthttp