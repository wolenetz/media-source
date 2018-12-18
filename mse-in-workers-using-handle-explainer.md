# Media Source Extensions: MediaSourceHandle Explainer
## (also known as MSE-in-Workers)

###### Author: Matthew Wolenetz, [Google Inc.](www.google.com) - December 17, 2018.  Last update December 17, 2018.

## tl;dr

We propose adding a `MediaSourceHandle` object, creatable on either the main
document `Window` context, a `DedicatedWorker` context, or a `SharedWorker`
context. This object's design intends to allow both:

* improved access to, and
* improved performance of the [Media Source Extensions API](https://www.w3.org/TR/media-source/) (MSE).

We plan to incubate this idea via the WICG, with goal of eventually working with
an appropriate WG to get the result of WICG incubation as part of the next
version of MSE.

### Implementation status as of last update

* Chrome - spec'ing and prototyping underway.
* Firefox - no signals yet.
* Safari Technical Preview - no signals yet.
* Current experimental web-platform-test results:
  * Pending further prototyping, etc.

## Background

Web authors have consistently requested that MSE be available from Worker
contexts: [MSE spec issue](https://github.com/w3c/media-source/issues/175).

Separately, we have strong suspicions that user experience
metrics, such as those measuring how long media playback takes to start, can be
impaired by MSE REC spec requiring implementations to delay operations like
`addSourceBuffer()` until after `MediaSource` is attached:
[Chromium's Tracking bug for MSE sourceopen latency improvement work](https://crbug.com/778082).

In the absence of such mechanisms, web authors are forced to await `MediaSource`
attachment completion on the main `Window` context, and perform all operations
on the MSE API also on the main `Window` context. Especially when the main
context is busy servicing other application logic, these restrictions can
significantly impair the user experience. For example, even if an app uses
Worker contexts to fetch media to feed to MSE, such feeding must still occur on
the main context, resulting in delayed media playback start and recurring
playback stalls in extreme cases when the playback reaches points in the
presentation timeline for which the app has been unable to buffer enough media.

Design consideration was given to alternative ideas that are not in scope of
this proposal:

* Upgrading the existing `MediaSource` object to be
  [Transferable](https://developer.mozilla.org/en-US/docs/Web/API/Transferable).
  However, `EventTargets` like `MediaSource` cannot be transferred. A viable,
  ergonomic way of overcoming this appears unlikely.
* Upgrading `HTMLMediaElement` (and its many related objects and APIs) to be
  usable on a Worker context. Though this might be a solution, the complexity of
  this approach led us to propose a simpler change scoped to just MSE.

Note that this proposal does not preclude separate or subsequent effort to
pursue these other ideas.

## Proposed Plan

This proposal is focused on enabling earlier ability to create SourceBuffers and
begin appending media to them, especially when the main `Window` context is
frequently busy, with further improvements allowed by using the MSE API on
`DedicatedWorker` and `SharedWorker` contexts.

The REC version of [Media Source Extensions
API](https://www.w3.org/TR/media-source/) (MSE) allows access to the API only on
the main document `Window` context. It further requires `MediaSource` objects
created by apps to initially be in the `closed` `readyState`, and prevents
applications from successfully calling `addSourceBuffer()` and obtaining
`SourceBuffer`s to begin buffering media until the `MediaSource` object
transitions to `open` `readyState`. That transition is signaled by the
`sourceopen` event, as a result of the potentially lengthy process of attaching
to an `HTMLMediaElement` which requires at least one yielding of the main
execution context before completion.

To maintain backwards compatibility with that behavior, while affording improved
MSE API access and performance, this proposal adds a feature-detectable (by its
existence and visibility) `MediaSourceHandle` object that is creatable on either
the main document `Window` context, a `DedicatedWorker` context, or a
`SharedWorker` context. A `MediaSourceHandle` object is
[Transferable](https://developer.mozilla.org/en-US/docs/Web/API/Transferable),
and can be used to attach the underlying `MediaSource` to an `HTMLMediaElement`
by setting the `HTMLMediaElement`'s `srcObject` attribute to the
`MediaSourceHandle`.

Each `MediaSourceHandle`, when constructed, also constructs a `MediaSource` object
that is available for retrieval by the app in the context that created the
handle, thereby allowing `MediaSource`, `SourceBuffer`, `SourceBufferList`, etc.
MSE API objects to be accessed by web apps on the context where the
`MediaSourceHandle` was constructed.

Unlike `MediaSource` objects directly constructed by apps, a `MediaSource`
constructed for this handle is initially in the `open` `readyState`, even when
the `MediaSourceHandle` has not yet been attached to an `HTMLMediaElement`.
This allows earlier `addSourceBuffer()` and initial `appendBuffer()` calls to
occur without first awaiting a `sourceopen` event on the `MediaSource`.

Note that implementations may still delay action on asynchronous initial
`appendBuffer()` calls until the `MediaSourceHandle` completes the attachment.

Note also that once a `MediaSourceHandle` is detached from an `HTMLMediaElement`,
the underlying `MediaSource` object's `readyState` becomes `closed` just like a
normally detached `MediaSource`. While this prevents immediate ability to create
new `SourceBuffers` for that `MediaSource`, the application can work around this
by creating another `MediaSourceHandle`, retrieving the new underlying
`MediaSource`, transferring the `MediaSourceHandle` to the main document
`Window` context (if it wasn't created there), and attaching it to the
`HTMLMediaElement`. The application could even prepare such in advance of a
detachment.


## Open Questions

##### Why `srcObject` and not (also, or instead) an objectUrl src for attachment of a `MediaSourceHandle` to an `HTMLMediaElement`?

###### Initial Thoughts:

Using `URL.createObjectUrl(MediaSource)` is already problematic in modern web
platform IDL due to MSE overloading a partial interface:
[MSE spec issue](https://github.com/w3c/media-source/issues/211).
Worse, MSE objectUrl usage already can lead to memory leaks if the
application does not correctly `revokeObjectUrl()`.

Since `srcObject` is the modern way in `HTMLMediaElement` to attach objects, we
already have a way forward, so the proposal is to not unnecessarily add yet more
problematic definition and usage of MediaSourceHandle objectUrls.

Note: this is listed explicitly as an open question to help highlight that
feedback on this aspect would especially be appreciated. Also, if making
`MediaSourceHandle` `Transferable` is found to not be viable, we may need to
reconsider routes like using an objectUrl.


##### Can implementations of this proposed API significantly reduce the performance problems and eliminate the API access problems versus REC MSE?

###### Initial Thoughts:

This is a big part of why we're incubating. Of course, standardizing a more
ergonomic API with improved interoperability are also strong reasons for
incubation.


##### Why must we create the `MediaSourceHandle` on a worker context if we wish to use the underlying `MediaSource` on that worker context? Why not instead allow (or require) the `MediaSourceHandle` to be constructed on the main `Window` context, and extracting something transferable from it to transfer to other contexts if needed there?

###### Initial Thoughts:

While this might allow some parallelization of attachment on the main document
`Window` context while the `SharedWorker` or `DedicatedWorker` context is still
starting, we're not convinced yet that the additional complexity is worth it.
For example, using subworkers or other multi-worker arrangements communicating
directly via the Channel Messaging API to reduce the granularity of priority worker
startup might be a reasonable workaround. This is another good reason
to incubate.


##### Can detachment of `MediaSourceHandle` not automatically shutdown all SourceBuffers (and remove all their buffered media?)

###### Initial Thoughts:

Perhaps, but an orthogonal feature proposal to let an app explicitly request a
MediaSource object to behave differently on detach of it (or in this case, its
handle) would reduce the complexity of each proposal and let them be developed
in parallel or separately.
