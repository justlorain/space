---
title: "Use middleware to implement distributed session solution"
description: "The main content of this post is to introduce a bizdemo hertz_session."
publishDate: "12 Dec 2022"
tags: ["webdev", "opensource", "tutorial", "go"]
---

## Introduction

The main content of this post is to introduce a bizdemo `hertz_session`. The link to the demo is [here](https://github.com/cloudwego/hertz-examples/tree/main/bizdemo/hertz_session).

This demo is designed to help users quickly get started with the [Session middleware](https://github.com/hertz-contrib/sessions) and [CSRF middleware](https://github.com/hertz-contrib/csrf) of the [Hertz](https://github.com/cloudwego/hertz) framework, and to show the distributed Session solution based on Redis.

If you don't know what Hertz is, then you can check out my previous articles which will help you get started with this Golang HTTP framework quickly.

The main features of the `hertz_session` :

- Use `thrift` IDL to define `HTTP` interface
- Use `hz` to generate code
- Use `hertz-contrib/sessions` to store sessions
- Use `hertz-contrib/csrf` to prevent Cross-Site Request Forgery attacks
- Use `Gorm` and `MySQL`
- Use `AdminLTE` as frontend page

## Hertz

[Hertz](https://github.com/cloudwego/hertz) is an ultra-large-scale enterprise-level microservice HTTP framework, featuring high ease of use, easy expansion, and low latency etc.

Hertz uses the self-developed high-performance network library `Netpoll` by default. In some special scenarios, Hertz has certain advantages in QPS and latency compared to go net.

In internal practice, some typical services, such as services with a high proportion of frameworks, gateways and other services, after migrating Hertz, compared to the Gin framework, the resource usage is significantly reduced, **CPU** **usage is reduced by 30%-60% with the size of the traffic**.

> For more details, see [cloudwego/hertz](https://github.com/cloudwego/hertz).

## How to get this demo

Use the following command to get `hertz_session`:

```shell
git clone https://github.com/cloudwego/hertz-examples.git
cd bizdemo/hertz_session
```

## Project structure


![ps](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tt30ux4ue5hfur2x55t8.png)

- `biz` holds the main business logic code, including `handler` for HTTP requests, `dal` for database operations, and `mw` for middleware
- `idl` stores `thrift` IDL
- `pkg` for some utility methods, business constants, `errmsg`, template rendering, etc
- `static` holds the frontend static files, all taken from `AdminLTE`
- Other files in the root directory include the main startup file `main.go`, docker config files, etc

## Use of middleware

### Session middleware

The distributed session solution based on redis is to store the sessions of different servers in redis or redis cluster,
which aims to solve the problem that the sessions of multiple servers are not synchronized in the case of distributed system.

**1. We store the session in Redis by using Hertz's session middleware, which is initialized as follows:**

```go
// biz/mw/session.go
func InitSession(h *server.Hertz) {
	store, err := redis.NewStore(consts.MaxIdleNum, consts.TCP, consts.RedisAddr, consts.RedisPasswd, []byte(consts.SessionSecretKey))
	if err != nil {
		panic(err)
	}
	h.Use(sessions.New(consts.HertzSession, store))
}
```

- First connect to Redis using `redis.NewStore` by passing the address, password, etc
- Use the Session middleware via the `h.Use` method and pass in the storage connection object that we just returned. The first parameter is the name of the Cookie, which we pass in as a defined constant

**2. Once initialized, we can store the user's information (username in this case) in the session after the user is authenticated and logged in. Here's the core code for the `Login` Handler Session:**

```go
// biz/handler/user/user_service.go/Login
session := sessions.Default(c)
session.Set(consts.Username, req.Username)
_ = session.Save()
```

- First get a Session object via `sessions.Default`
- The username is stored in the Session using the `Set` method
- Finally, we call `Save` to save the Session object

After the user is logged in, we will store a copy of the user's information in the session object stored in Redis, so the user can visit the corresponding page without logging in.

**3. In this case, there is only one home page, `index.html`, which will be checked if the user is logged in and redirected to the login page `login.html` if they are not logged in. The core code to check if they are logged in is as follows:**

```go
// pkg/render/render.go
h.GET("/index.html", func(ctx context.Context, c *app.RequestContext) {
    session := sessions.Default(c)
    username := session.Get(consts.Username)
    if username == nil {
        c.HTML(http.StatusOK, "index.html", hutils.H{
            "message": utils.BuildMsg(consts.PageErr),
        })
        c.Redirect(http.StatusMovedPermanently, []byte("/login.html"))
        return
    }
    c.HTML(http.StatusOK, "index.html", hutils.H{
        "message": utils.BuildMsg(username.(string)),
    })
})
```

It's exactly the same as `Login` in step 2, but instead of `Set`, it's `Get` and without  `Save`

**4. Finally, we need to clean up the user's session when the user logs out, the core code is as follows:**

```go
// biz/handler/user/user_service.go/Logout
session := sessions.Default(c)
session.Delete(consts.Username)
_ = session.Save()
```

Again, note that you need to `Save` otherwise the delete operation will be invalid.

Session middleware encapsulates most of the complex logic that needs to be considered, such as the storage of different user sessions in Redis, and we only need to call a simple interface to complete the corresponding business process.

### CSRF middleware

Next up is the use of CSRF middleware. The following explanation of what CSRF attacks are is taken from Wikipedia:

> **Cross-site request forgery**, also known as **one-click attack** or **session riding** and abbreviated as **CSRF** (sometimes pronounced *sea-surf*[[1\]](https://en.wikipedia.org/wiki/Cross-site_request_forgery#cite_note-Shiflett-1)) or **XSRF**, is a type of malicious [exploit](https://en.wikipedia.org/wiki/Exploit_(computer_security)) of a [website](https://en.wikipedia.org/wiki/Website) or [web application](https://en.wikipedia.org/wiki/Web_application) where unauthorized commands are submitted from a [user](https://en.wikipedia.org/wiki/User_(computing)) that the web application trusts.[[2\]](https://en.wikipedia.org/wiki/Cross-site_request_forgery#cite_note-Ristic-2) There are many ways in which a malicious website can transmit such commands; specially-crafted image tags, hidden forms, and [JavaScript](https://en.wikipedia.org/wiki/JavaScript) [fetch](https://en.wikipedia.org/wiki/XMLHttpRequest#Fetch_alternative) or XMLHttpRequests, for example, can all work without the user's interaction or even knowledge. Unlike [cross-site scripting](https://en.wikipedia.org/wiki/Cross-site_scripting) (XSS), which exploits the trust a user has for a particular site, **CSRF exploits the trust that a site has in a user's browser**.[[3\]](https://en.wikipedia.org/wiki/Cross-site_request_forgery#cite_note-Synopsys-3) In a CSRF attack, an innocent end user is tricked by an attacker into submitting a web request that they did not intend. This may cause actions to be performed on the website that can include inadvertent client or server data leakage, change of session state, or manipulation of an end user's account.

After reading the relevant information, I understand that malicious websites use the trust of some websites on the user's browser, such as the use of cookies, to launch some attacks.

Here we use CSRF middleware to protect registration, login POST form submission and POST logout. It is worth noting that the logout is initially using the GET method, but the browser sometimes caches the GET request, so there is a "bug" that fails to delete the Session. This problem was solved by changing the request method to POST and adding CSRF protection, check out this [PR](https://github.com/cloudwego/hertz-examples/pull/64) for details.

In this demo, since the CSRF middleware is added later, it was not considered when defining IDL at the beginning, so there is only one GET request to log out after login. Since GET is considered to be a safe method, here we mainly use the CSRF middleware to protect the registration and login two POST form submissions.

**1. Let's also take a look the initialization and usage of CSRF as follows:**

```go
func InitCSRF(h *server.Hertz) {
	h.Use(csrf.New(
		csrf.WithSecret(consts.CSRFSecretKey),
		csrf.WithKeyLookUp(consts.CSRFKeyLookUp),
		csrf.WithNext(utils.IsLogout),
		csrf.WithErrorFunc(func(ctx context.Context, c *app.RequestContext) {
			c.String(http.StatusBadRequest, errors.New(consts.CSRFErr).Error())
			c.Abort()
		}),
	))
}
```

- Call `h.Use` using CSRF middleware
- Define the token with `csrf.WithSecret`
- Retrieve the CSRF Token from the post form using the `csrf.WithKeyLookup` definition, default to the request header
- Use `csrf.WithNext` to skip the no-login case, i.e. no Cookie exists
- Custom exception handling with `csrf.WithErrorFunc`

**2. After initialization, we only need to submit the generated CSRF Token through the `hidden` field when submitting the form after login, and the middleware will automatically help us verify whether it is valid. If there is an error, the exception handling function just defined will be used, because this Demo uses the template rendering mode. The core code is as follows (in the case of registration, the same goes for login) :**

```go
// pkg/render/render.go
h.GET("/register.html", func(ctx context.Context, c *app.RequestContext) {
    if !utils.IsLogout(ctx, c) {
        token = csrf.GetToken(c)
    }
    c.HTML(http.StatusOK, "register.html", hutils.H{
        "message": utils.BuildMsg("Register a new membership"),
        "token":   utils.BuildMsg(token),
    })
})
```

The core HTML template is as follows:

```html
<div>
	<input type="hidden" name="csrf" value="{[{ .token | BuildMsg }]}">
</div>
```

- After verifying that the user is logged in, we can get the corresponding token and put it in the form to submit, and let the middleware handle the rest.

## Run the demo

Execute the following command to run the demo:

```shell
docker-compose up
go run .
```

then visit `localhost:8888/register.html` to visit the register page, here are the sample pages:


![registerpage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kppabfi4o1d8jd9ksh2e.png)


![loginpage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/we871234jaa3uxgs3ce8.png)


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ld855igi1c4rqa4p2xan.png)


## Summary

That's all for this article, I would be happy if you found this helpful. Feel free to leave a comment if you have any questions.

## Reference List

- https://github.com/cloudwego/hertz-examples
- https://github.com/hertz-contrib/sessions
- https://github.com/hertz-contrib/csrf
- https://github.com/ColorlibHQ/AdminLTE
- https://dev.to/llance_24/golang-csrf-defense-in-practice-10k