# Media Source Extensions: Eviction Policies Explainer

###### Author: Matthew Wolenetz, [Google Inc.](www.google.com) - May 17, 2019.  Last update May 17, 2019.

## tl;dr

We propose adding new coded frame eviction policies to `SourceBuffer` objects,
enabling web applications to have greater control over how the implementation
manages buffered media. Over time, further eviction policies may be incubated
beyond those described herein. In addition to new eviction policies, the
previous flexibility over which presentation ranges to select for removal is
retained in a defined eviction policy, with new non-normative guidance intended
to improve interop.

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

The REC version of [Media Source Extensions API](https://www.w3.org/TR/media-source/) (MSE)
gives user agents large amount of control over which buffered coded frames to
evict. The current specification indicates the user agent just needs to find _"a
list of presentation time ranges that can be evicted from the presentation to
make room for the new data"_ as part of the [Coded Frame Eviction
Algorithm](https://www.w3.org/TR/media-source/#sourcebuffer-coded-frame-eviction).

In a non-normative note, it further warns applications not to depend on a
specific behavior, with guidance that applications can observe the
[buffered](https://www.w3.org/TR/media-source/#dom-sourcebuffer-buffered)
attribute to discover what has changed. User agents are also allowed to
determine when they were unable to evict enough to make room, and signal
`QuotaExceededError` instead of continuing and buffering the appended media.

This flexibility given to user agents limits applications and impairs interop:

* If an application wishes to fetch and append media *before* seeking to play
  it, it has no assurance that the media will remain buffered at the eventual
  seek target if the application issues multiple `appendBuffer()` calls to
  buffer that new media prior to issuing the seek. (The coded frame eviction
  algorithm runs only at the beginning of each `appendBuffer()` call, so if only
  one call is made to buffer media at the eventual seek target, that media
  should exist in the timeline; but subsequent appendBuffer() calls could remove
  it.)

* Furthermore, user agent implementations may even decide to evict the
  currently-playing media such that playback could immediately stall for a
  playback which previously had multiple coded frames buffered but not yet
  decoded before the next random access point.

* Though implementations like Chromium take great care to not evict media which
  may play soon, lack of clear specification limits interop and increases
  application complexity.

## New Use Cases

MSE also prevents user agents (and applications which can explicitly evict
presentation time ranges from a `SourceBuffer` using `remove()` calls) from
removing coded frames without also removing all of their potential decode
dependencies. Normally, this is desirable behavior, but in some streaming cases
where random access points are extremely infrequent (for instance, to reduce
coded frame fetching latency for the same network bandwidth), web developers
have requested the ability for MSE to evict everything that is in the past
relative to the current media being fed to the decoder -- including coded frames
since the most recent random access point. This would allow such streams to
buffer and play with MSE without hitting `QuotaExceededError`, albeit with
consequent reduction of ability to resume playback or finalize seek (even to
`currentTime`) without the application re-buffering media.

Web developers have similarly requested to be able to tell the MSE API to
automatically collect everything in the past relative to the current media being
fed to the decoder -- except retain all coded frames since the most recent
random access point. This would generally allow for reduced buffered media
memory footprint for apps which infrequently expect to seek backwards, while
retain reliability of seeking to currentTime and resuming from suspended
playback.

## Non-Goals

Design consideration was given to ideas that are not in scope of this proposal:

* Using an optional argument to `addSourceBuffer()` to specify the eviction
  policy. Feature-detection of a new argument to `addSourceBuffer()` is not
  viable. Instead, the proposal lets the app vary the eviction policy of a
  `SourceBuffer` using a new attribute which also allows feature detection.

* Providing an API whereby applications could query `SourceBuffer` to get the
  presentation time of the most recent random access point.
  [link](https://github.com/w3c/media-source/issues/173)

* Providing eviction policies (and possibly events to notify the app) that would
  allow the user agent to pre-emptively evict (not just during the
  [Prepare Append Algorithm](https://www.w3.org/TR/media-source/#sourcebuffer-prepare-append))
  any coded frames at the discretion of the user agent. These policies remain
  within the larger objectives described in the MSE bug which motivated this
  proposal. [link](https://github.com/w3c/media-source/issues/232)

Note that this proposal does not preclude separate or subsequent effort to
pursue these other ideas.

## Proposed API

This proposal is focused on providing ability for applications to select among
a variety of eviction policies for a SourceBuffer, including policies that allow
the new use cases described above. To assist interop of the pre-existing
eviction flexibility afforded to user agents, it also defines a policy for that
which contains non-normative text to help guide expectations.

`SourceBuffer` will have a new set of eviction policies:

```
  enum EvictionPolicy {
      "normal",
      "before-current-gop",
      "before-next-demuxed"
  };
```

`SourceBuffer` will have a new attribute by which an application can set and
query the current EvictionPolicy:

```
  attribute EvictionPolicy evictionPolicy;
```
On getting this attribute, the user agent should return the initial value or the
last value that was successfully set.
Upon setting the `evictionPolicy` attribute of a `SourceBuffer` object, the
following steps should be taken by the user agent:

1. If this object has been removed from the `sourceBuffers` attribute of the
   parent media source, then throw an `INVALID_STATE_ERR` exception and abort
   these steps.
2. If the `updating` attribute equals true, then throw an `INVALID_STATE_ERR`
   exception and abort these steps.
3. Let `new eviction policy` equal the new value being assigned to this
   attribute.
4. If the `readyState` attribute of the parent media source is in the "ended"
   state then run the following steps:
   1. Set the `readyState` attribute of the parent media source to "open".
   2. Queue a task to fire a simple event named `sourceopen` at the parent
     media source.
5. If the append state equals `PARSING_MEDIA_SEGMENT`, then throw an
   `INVALID_STATE_ERR` exception and abort these steps.
6. Update the attribute to `new eviction policy`.


### EvictionPolicy semantics

#### "normal"

This policy encapsulates the existing MSE-REC behavior. This proposal strongly
suggests that the initial `evictionPolicy` of a `SourceBuffer` object be
"normal" to afford improved backwards compatibility. (Other incubations, e.g.,
of pre-emptive eviction policies, may allow user agents to choose a different
initial `evictionPolicy`.)

While much flexibility is given to user agents in this policy, the following
general non-normative guidance is proposed for this policy:

* User agents should conservatively collect buffered coded frames to reduce the
  need for applications to re-fetch and re-append media which they expected to
  have been retained.
* User agents should try to retain the currently playing GOP (group of
  pictures, here meaning a random access point and all of the coded frames
  having decode dependency upon it) and the most recently appended GOP.
* The following successive steps are recommended when the user agent needs to
  evict coded frames to attempt to make room for newly appended media (here,
  "current playback time" means roughly the `currentTime` of the media element
  when the eviction ranges are selected; it should be approximately earlier than
  the presentation time of media that has not finished decoding yet).

  1. If the last appended coded frame's position in the presentation timeline is
     earlier than the current playback time, consider deleting GOPs between that
     last appended position and the most recent random access point before the
     current playback time.
  2. If there is an unsatisfied pending seek:
     1. Consider removing data earlier than the seek target in the presentation
        timeline. If the seek target is actually buffered (but not yet decoded
        sufficiently to satisfy the seek), do not remove anything between the
        most recent random access point at or before the the seek target and the
        seek target.
     2. Consider removing data from the end of the presentation timeline,
        backwards, until either the seek target or the most recently appended
        GOP, exclusively.
     3. Consider greedily removing GOPs from the front of the presentation
        timeline.
  3. If there is no pending seek:
     1. Consider removing GOPs from the front of the presentation timeline up
        to, but not including the earlier of: the start of the range most
        recently appended to, or the current playback time.
     2. Consider removing GOPs from the end of the presentation timeline
        backwards to, but not including the later of: the most recently appended
        coded frames, or the current playback time.
  4. If there is still not enough room to buffer the newly appended media, issue
     a `QuotaExceededError` per the [Prepare Append
     Algorithm](https://www.w3.org/TR/media-source/#sourcebuffer-prepare-append).

#### "before-current-gop"

This policy more liberally removes everything buffered before the GOP containing
the next coded frame to be decoded, and if more room is needed at that point,
the steps in the "normal" policy may be taken by the user agent. Note that if
there is an unsatisfied pending seek, then this policy's effective behavior
should be equivalent to the "normal" policy.

Note: In discussion at recent FOMS 2018 and 2019 conferences, this policy was
loosely referred to as "CMAF-Low Latency".

#### "before-next-demuxed"

This policy even-more-liberally removes everything buffered before the next
coded frame to be decoded, including coded frames at or after the most recent
random access point prior to that next coded frame to be decoded. If more room
is needed, the steps in the "normal" policy may be taken by the user agent. If
there is an unsatisfied pending seek, this policy's effective behavior should be
equivalent to the "normal" policy.

Note: In discussion at recent FOMS 2018 and 2019 conferences, this policy was
loosely referred to as one which allows MSE to play "infinite GOP" streams such
as a stream which has only one keyframe followed by only 

### Additional changes necessary due to "before-next-demuxed"

#### Collect partial GOPs on seek start

"before-next-demuxed" may leave partial GOPs buffered that no longer have all
their decode prerequisites. This proposal requires that when a seek is issued on
a media element to which the `SourceBuffer` is attached by MSE, all partial GOPs
that are buffered but no longer have their random access point buffered should
be removed before attempting to satisfy the seek. This behavior must occur
regardless of current `evictionPolicy`, because switching eviction policies away
from "before-next-demuxed" doesn't evict a currently-playing partial GOP that no
longer has its random access point buffered, and (for stability) seeking must
start decoder preroll from a random access point.

#### buffered and seekable

Seekable ranges (unless extended explicitly by `setLiveSeekableRange()` usage)
should not include presentation time intervals corresponding to any partial GOPs
that no longer have their random access point buffered.

Buffered ranges logic remains unchanged; partial GOPs still-buffered
presentation intervals should be included in the calculation.

## Open Questions

##### Should a user agent which implements this proposal be allowed to implement only a subset of the defined eviction policies, and if so, how can an app see that a policy is not supported?

###### Initial Thoughts:

A new exception may be needed for this. It would allow for future-compatibility
with other eviction policies, not all of which every implementation may feasibly
support.

##### Should there be a settable eviction policy, such as "auto", that gives UA flexibility to choose whichever policy it prefers? Should the initial eviction policy of every SourceBuffer follow this "auto" logic?

###### Initial Thoughts:

Some implementations that currently do not support MSE may only implement
pre-emptive policies (in future incubation) as a requirement for supporting MSE
at all. The proposed initial "normal" policy is non-preemptive. It's unclear how
such eventual preemptive-only policies may be communicated to applications,
other than perhaps an exception upon attempted appendBuffer specific to that
case.

Alternatively, switching always to "auto" as initial policy could impair
existing interop if existing MSE implementations prefer different policies that
do not correspond to the MSE REC "normal" `EvictionPolicy`.
