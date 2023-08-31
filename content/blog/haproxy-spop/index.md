---
id: haproxy-spop
date: 2023-08-31T00:00:00
title: A HAProxy SPOP library tour
author: Fionera
tags: [projects]
---

I learned about HAProxy SPOP and its extreme versatility while getting introduced to the browser validation at one of my old jobs. 

There are multiple ways to filter out bad clients, one of the simplest ones are small websites that are solving a proof-of-work challenge. Cloudflare is a perfect example of such a client validation that nearly everyone have seen at least once. 
To allow implementation of such a system, the loadbalancer needs to conditionally redirect clients based on a secret the browser provides, mostly based on a cookie. Either the browser sends one that is valid, where the webserver also has to actually validate it, or redirect to the validation page. 

Most people would tackle that by injecting lua into their config, e.g. with openresty or haproxy too, but as the complexity of such payloads increases it gets increasingly difficult to use lua. 

HAProxy solved and enabled many other things this by implementing SPOP (Stream Processing Offload Protocol). 

To state the [SPOE.txt](https://www.haproxy.org/download/2.0/doc/SPOE.txt) introduction:
> SPOE is a feature introduced in HAProxy 1.7. It makes possible the
communication with external components to retrieve some info. The idea started
with the problems caused by most ldap libs not working fine in event-driven
systems (often at least the connect() is blocking). So, it is hard to properly
implement Single Sign On solution (SSO) in HAProxy. The SPOE will ease this
kind of processing, or we hope so.

---

There are already multiple implementations of this protocol in go and other languages, but they do have different issues. The first one I learned about and is also used by [Coraza](https://github.com/corazawaf/coraza) is https://github.com/criteo/haproxy-spoe-go. It is a pretty good implementation, but not actively maintained. It also does not have any way of constructing your own messages for e.g. unit tests.

Another implementation was made by negasus at https://github.com/negasus/haproxy-spoe-go which appears to be more actively maintained, but it does spit out lots of `interface{}` types. You could use it and just panic if a wrong type appears (foreshadowing: I do that in my lib), but it is definitely possible to make this completely statically typed and even construct a validator for messages to prevent any wrong messages in the first place. 

So we reached the point where, as always, a XKCD comic describes the world perfectly:
![](https://imgs.xkcd.com/comics/standards.png)

---

Let me introduce you to one of my favorite hobbies: Zero alloc Go!
I always love to increase performance and since memory allocation isn't fast, it's a good way of doing that. 

Yes most of the time, the allocation isn't the reason why code is slow, but in this case I already knew that the wire protocol is fairly simple and from my experience with the [criteo](https://github.com/criteo/haproxy-spoe-go) library I knew that allocations are one of the biggest performance hurdles these libraries have.

Because I wanted to write an open source client validation tooling for HAProxy anyway and I need to start somewhere, I wrote my own SPOP library. You can find it [here](https://github.com/fionera/haproxy-go/tree/master/spop).

When implementing a protocol like this without doing any allocations inside the hot path is fairly simple. It basically results into three things: reuse everything, pools are your friends, pprof is a blessing.

HAProxy sends everything as frame where the first one contains the maximum frame size for all subsequent ones on that connection. Based on this we can allocate the first ever message to be received as 16k byte buffer (This is described inside the SPOE.txt) and just read it into there. 

While decoding, reslice the existing buffer for the decoder, decode it and write all values into a pre-allocated space inside the message struct.
When a user wants to use the variable we allow this by exposing the raw value and functions that convert it to the correct type. It is a bit weird to use as you are not allowed to reuse any data after leaving the scope, but that's the tradeoff you have when doing zeroalloc.

Funfact: If you convert `[]byte` to string with `string(foo)` you actually do a copy. __But__ if you use it for comparing two `[]byte` values, it doesn't.

Code from [bytes.Equal](https://github.com/golang/go/blob/master/src/bytes/bytes.go#L18):
```go
func Equal(a, b []byte) bool {
	// Neither cmd/compile nor gccgo allocates for these string conversions.
	return string(a) == string(b)
}
```

--- 

I may update this post in the future but since I am still working on finding out how I actually want to design the API, also I already had a [POC](https://github.com/fionera/haproxy-go/tree/master/peers) for the HAProxy Sticktable sync protocol, but please don't look at it, it's just a POC and nothing pretty.