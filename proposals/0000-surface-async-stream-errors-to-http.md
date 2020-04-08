# Surfacing Async Stream Errors to HTTP

Michael Rawlings
Marko doesn't do anything specific at the http level since right now the api is rendering to a stream (usually an http response, but could be something like an fs write stream). For errors that aren't handled by `<await>`'s `<@catch>`, an error event is emitted on the stream. But for caught errors, I'm not sure what the best course of action is here. Should marko handle closing the stream or should it emit a caught-error event so that the application can do the right thing based on the protocol?

It's a little strange too because in the case of a caught error we want to finish rendering the page and send it, but we may still want to signal to the browser that something went wrong.

Michael Rawlings


Taylor Hunt
and I guess Marko's whole reason for being is efficiently rendering HTML
which is why you probably shouldn't use it for, say, emitting `text/event-stream` or WebSockets

## Intro

> A _short_ explanation of the proposal.

## Motivation

Marko already has a way to tell _humans_ that the promise for an `<await>` rejected: the `<@catch>` block. Unfortunately, for machines, the original HTTP status code is how they determine the error state of a response — and the HTTP headers are already sent by the time an `<await>` errors.

Say we have Marko render the page’s `<head>` and the site logo/navigation/searchbar while the Node server calls a backend API to populate the page content:

```marko
<!doctype html>
<html lang="en">
<head>
  <PageAssets />
  <PageMetadata />
</head>

<body>
  <SiteHeader />

  <PageContent>
    <await(input.contentFetchRequest)>
      <@then|response|>
        ${response.body}
      </@then>
      <@catch|err|>
        Oh no, the content API is down again
      </@catch>
    </await>
  </PageContent>

  <SiteFooter />
</body>
</html>
```

This is great for the user because their browser gets a head start downloading assets linked in the `<head>` and displaying the site header while the network request to the content API is underway.

However, if the `contentFetchRequest` fails for any of the myriad reasons computers are terrible, the page will show an error message instead of its real content — but as far as the HTTP layer knows, the response completed successfully.

No machine-readable indicator that the streamed response contains erroneous content causes some thorny problems:

- An HTTP cache may store the erroneous content and reuse it, causing users to see the error for longer than they otherwise would

- A search engine will index the erroneous content, since they received no sign they should try again or discard the response as invalid

- HTTP-level tools (debuggers, curl, spiders, etc.) will report the response as successful, even when it wasn’t

Research into HTTP/1.1’s chunked `Transfer-Encoding` and HTTP/2’s stream error handling found that they both have a standardized way of indicating a dynamically-streamed response failed to complete successfully:

<dl>
  <dt>HTTP/1.1 <code>Transfer-Encoding: chunked</code></dt>
  <dd><a href="https://httpwg.org/http-core/draft-ietf-httpbis-messaging-latest.html#incomplete.messages">IETF Draft: HTTP/1.1 Messaging §8 Handling Incomplete Messages</a></dd>
  <dd><p>If a chunked response doesn’t terminate with the zero-length end chunk, the client must assume that the response was <i>incomplete</i> — which at the very least, <a href="https://httpwg.org/http-core/draft-ietf-httpbis-cache-latest.html#rfc.section.3.1">means a cache should double-check with the server before reusing the stored incomplete response</a>.</p></dd>

  <dt>HTTP/2 Streams</dt>
  <dd><a href="https://tools.ietf.org/html/rfc7540#section-5.4.2">RFC 7540 §5.4.2 Stream Error Handling</a></dd>
  <dd><p>An HTTP/2 stream can signal an application error by sending a <code>RST_STREAM</code> frame with an error code of <code>0x2 INTERNAL_ERROR</code>.</p></dd>
</dl>

This proposal explores how Marko should surface errors from server components to the HTTP layer, so the response can properly signal an error with one of the above methods.

## Guide-level explanation

Developers should be able to tell Marko how important an async error is:

1. Our example from the Motivation section is as serious as it gets — you might even want to redirect to a completely different error page 
2. Smaller partials in a larger page may still want to tell proxies and search engines that the response is incomplete, but the user might find value in the rest of the page around the `<@error>`
3. You might not care at all for trivial parts of the page, like pulling weather data to change the background image or something

> Explain the proposal as if it was already implemented and you are now teaching it to another Marko developer. That generally means:
> 
> - Explaining the feature largely in terms of examples.
> - What names and terminology work best for these concepts and why? Is it a continuation of existing Marko patterns, or a wholly new one?
> - Explaining how Marko developers should _think_ about the feature, and how it should impact the way they use Marko. It should explain the impact as concretely as possible.
> - If applicable, describe the differences between teaching this to existing Marko developers and new Marko developers.

> For implementation-oriented proposals, this section should focus on how contributors should think about the change, and give examples of its concrete impact. For policy proposals, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

> Would the acceptance of this proposal mean the Marko docs must be re-organized or altered?

## Reference-level explanation

> This is the technical portion of the proposal. Explain the design in sufficient detail that:
> 
> - Its interaction with other features/tools is clear.
> - It is reasonably clear how the feature would be implemented.
> - Corner cases are dissected by example.
> - The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

It is out of scope for this mechanism to surface anything other than runtime errors — mostly because that’s the only semantic we can surface through either version of HTTP. It would be wrong to use this for a 404 page, for example.

### Trailers

We should send error information in HTTP trailers if any clients would benefit. [Some browsers have traditionally discarded almost all trailers other than `Server-Timing`](https://www.fastly.com/blog/supercharging-server-timing-http-trailers), but other browsers, tools like `curl`, search engines, proxy caches, and debuggers may benefit. For example, an updated `Cache-Control`, `Retry-After`, or even `Refresh`ing to an error page at a new URL if the problem was serious enough.

Even for stricter browsers, error information inside `Server-Timing` could help developers, as many monitoring tools track, expose, and alert information from this header.

```
Server-Timing: markoAsyncErr; dur=87.1; desc="${errorInfo}"
```

What information should be exposed this way?

- The `dur` field is standardized as indicating how far along the event happened in the request
- The `desc` field is any string the developer wishes 
- Multiple Server-Timing events are allowed with the same name, so Marko could emit it repeatedly in the case of multiple async failures, with `desc` disambiguating what happened and where

However, note that over HTTP/1.1, trailers can only be appended _after_ the zero-length terminator chunk. Hmm.

### HTTP/1.1 considerations

To avoid performance problems with `keepalive` connections, it is probably not wise to explicitly call `.close()` — instead, the streamed HTML response should finish without ever sending the zero-length chunk terminator.

But does that mean that kept-alive connection never gets reused by the client, therefore blocking other resources from getting sent over it?

Maybe we should `.close()` the connection only once we know we’re done rendering the page.

The spec also indicates a `Transfer-Encoding` decode error also marks the message as incomplete; a chunk with a _negative_ integer length might trigger that behavior? Or any other data that isn’t a positive hex integer.

A chunk that’s displaying the contents of a `<@catch>` error message may also encode error information as [chunk extensions](https://tools.ietf.org/html/rfc7230#section-4.1.1). I wonder if there’s any standards or _de facto_ implementations that leverage these?

What a headache. At least HTTP/2 has a much more explicit signal for errors during a stream.

### HTTP/2 considerations

Developers could signal that even though an error occurred, the request is safe to retry. [The HTTP/2 spec provides two methods to indicate which streamed requests are safe to retry](https://tools.ietf.org/html/rfc7540#section-8.1.4): a `RST_STREAM` frame with an error code of `REFUSED_STREAM`, or a `GOAWAY` frame indicating which stream IDs are safe to retry.

However, those signals indicate a retry is safe even if the request semantics were not idempotent (like a `POST` request), so they may not be useful by the time the request handling makes it to the Marko layer:

> A server MUST NOT indicate that a stream has not been processed unless it can guarantee that fact. If frames that are on a stream are passed to the application layer for any stream, then `REFUSED_STREAM` MUST NOT be used for that stream, and a `GOAWAY` frame MUST include a stream identifier that is greater than or equal to the given stream identifier.

Still, we might provide a way for Marko authors to signal this safe-to-retry information — the application may be allowed to reason about errors within its own layer.

### Signaling within Node

Node’s `http` and `http2` modules are probably capable of emitting error state as the standards require, but it’s not 100% clear how to transparently work with them in ways that the greater ecosystem would expect.

For example, how does Express react to a `http` stream closing without the zero-length terminator?

Hopefully, Node already does The Right Thing when these modules’ streams produce the protocol-specific error signal, and the ecosystem has had time to discover and handle that occasion. (If not, Node might appreciate a pull request.) More research needed.

Presumably-helpful parts of the API:

- https://nodejs.org/api/net.html#net_socket_end_data_encoding_callback
- https://nodejs.org/api/net.html#net_socket_setkeepalive_enable_initialdelay
- https://nodejs.org/api/http.html#http_request_abort
- https://nodejs.org/api/http.html#http_event_clienterror
- `http2session.destroy([error][, code])`
- `http2session.goaway([code[, lastStreamID[, opaqueData]]])`
- https://nodejs.org/api/http2.html#http2_destruction
- `http2stream.close(code[, callback])`
- `response.addTrailers(headers)`

### HTTP ecosystem considerations

It should be possible for developers to selectively turn this error-signaling behavior off, in the case of misbehaving proxies or known-bad clients.

Research needed into how common reverse proxies or load balancers, such as nginx, handle error-signaled HTTP streams. Real-world examples of how that can happen: https://rhodesmill.org/brandon/2013/chunked-wsgi/ (including the comment at the very bottom)

https://rijulaggarwal.wordpress.com/2018/01/10/atmosphere-long-polling-on-nginx-chunked-encoding-error/

In the common case of the backend speaking HTTP/1.1 to a reverse proxy/front-end terminator/CDN/etc. that translates to HTTP/2 for browsers, what can be done?

## Migration strategy

> Is this proposal backwards incompatible? Does this proposal replace an existing feature/pattern? If so, can we safely and automatically migrate existing apps/components to this proposal? How?

> If applicable, provide sample error messages, deprecation warnings, or migration guidance.

## Drawbacks

So far, Marko is totally agnostic as to what kind of stream it’s outputting to. You could use it to stream to a file, or something really unexpected like an FTP stream transmission or a chat protocol.

It’s possible the server runtime might expose a different method to avoid breaking such cases, like `template.streamToHttp()`.

However, there’s already a lot about Marko’s server rendering that only makes sense delivered over HTTP. For example, if you were rendering to a file to be served later, you’d want to buffer the component init scripts and you’d never want `client-reorder` enabled.

## Alternatives

The impact of not doing this would be no change; we would continue exposing Marko authors to the risks mentioned in the Motivation section.

In the past, browsers other than Internet Explorer accepted `multipart/x-mixed-replace` responses to be loaded in `target="_top"`, and that could potentially be used to swap to a a new copy of the response-in-progress but with error-indicating headers, or potentially even a replacement error page. However, modern browsers have quietly dropped support for `multipart/x-mixed-replace` in the top browsing context.

It _might_ be possible to write `<meta http-equiv="refresh" content="0;url=${errorPageLocation}">`. It’s not _supposed_ to be outside the `<head>`, but I still like it better than how Rails creates a mid-stream error redirect with an inline `<script>`. ([some context on how search engines could better understand this is a temporary redirect, not a permanent one](https://twitter.com/JohnMu/status/969486943351394304))


## Prior art

[You used to have to delay a bit before closing a connection without the zero-length terminator because of this Chrome bug](https://bugs.chromium.org/p/chromium/issues/detail?id=610126)

https://stackoverflow.com/questions/17203379/response-sent-in-chunked-transfer-encoding-and-indicating-errors-happening-after

The issue of signaling errors that occur during a dynamic HTTP stream is not specific to Marko; the problem is very old. Other server runtimes and frameworks may have paved this cowpath already. Some candidates widely-used enough to probably have built-in behavior or guidance:

- Express and other Node HTTP frameworks
    - https://expressjs.com/en/guide/error-handling.html#the-default-error-handler
    - https://github.com/expressjs/express/issues/2700 (!!!)
    - https://zellwk.com/blog/express-errors/#when-streaming
    - https://stackoverflow.com/questions/21509233/error-handling-in-express-while-piping-stream-to-response
- PHP’s output buffers and its `flush()` family of functions
    - https://stackoverflow.com/questions/29894154/chrome-neterr-incomplete-chunked-encoding-error (good lord the answers all reveal a lot of ways this can happen)
- Java JSPs, or its various frameworks
    - https://techblog.bozho.net/error-pages-and-chunked-encoding-its-harder-than-you-think/
- Ruby on Rails
    - https://api.rubyonrails.org/classes/ActionController/Streaming.html#module-ActionController::Streaming-label-Errors
    - https://weblog.rubyonrails.org/2011/4/18/why-http-streaming/
- Perl may be the most interesting of all, as it was widely-used during the migration to HTTP/1.1 — HTTP/1.0 notoriously couldn’t tell the difference between the server finishing a dynamic response or the connection being closed for other reasons.
- [Django basically discourages streaming altogether](https://docs.djangoproject.com/en/3.0/ref/request-response/#django.http.StreamingHttpResponse)
- https://rhodesmill.org/brandon/2013/chunked-wsgi/
- https://www.oreilly.com/library/view/http-the-definitive/1565925092/ch04s07.html
- https://web.archive.org/web/20150114012656/http://blogs.msdn.com/b/aspnetue/archive/2010/05/25/response-end-response-close-and-how-customer-feedback-helps-us-improve-msdn-documentation.aspx

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD?
