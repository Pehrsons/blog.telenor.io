---
layout: post
title:  "Clone your MediaStreamTracks in Firefox 48"
date:   2016-04-25 15:33:00
categories: webrtc
author: "Andreas Pehrson"
tags: webrtc Firefox
comments: false
---

For the last year and a half, I have been part of Telenor and Mozilla's WebRTC Competency Center where all participants help Mozilla improve their WebRTC stack.

Most of my work so far has been on features and bug fixes in and around the MediaStream and MediaStreamTrack implementation.

Today I landed a huge rewrite, rerouting most of the main thread communications between streams and their sources and sinks, leading to finally enabling `MediaStream.clone()` and `MediaStreamTrack.clone()` in Firefox 48.

Read more for a background on how MediaStreams in Firefox work, and what changes I did to it.

<!--more-->

---

### Background

Gecko's MediaStream and MediaStreamTrack implementation largely has two parts: a main thread API - much of which is js-exposed, and an internal MediaStreamGraph that processes media data on a background thread. I'll walk you through what major changes have been done to both below.

### Outline

- [MediaStreamGraph is complex](#msg-complex)
- [The old stream-centered way](#stream-centered)
- [MediaStreamGraph has been taught what tracks are](#msg-tracks)
- [We have MediaStreamTrackSources now](#track-sources)
- [Security needs to be track-centered](#security)
- [Fixing tricky intermittent bugs - and some perf issues](#intermittents)
- [Wait what? What does this actually mean?](#tldr)

### <a name="msg-complex"></a> MediaStreamGraph is complex

The [MediaStreamGraph](https://dxr.mozilla.org/mozilla-central/rev/55d557f4d73ee58664bdf2fa85aaab555224722e/dom/media/MediaStreamGraph.h) is the central engine for MediaStreams in Gecko. It is easiest explained as a mapping of how all streams and tracks are connected to each other, and on every iteration (~10ms) it goes through all tracks to ensure they contain data for the current time.

There are three main types of internal streams used by the MediaStreamGraph:

- [SourceMediaStream](https://dxr.mozilla.org/mozilla-central/rev/55d557f4d73ee58664bdf2fa85aaab555224722e/dom/media/MediaStreamGraph.h#722)
    - Data either gets pushed to the source by a producer or pulled in by the MediaStreamGraph.
- [TrackUnionStream](https://dxr.mozilla.org/mozilla-central/rev/55d557f4d73ee58664bdf2fa85aaab555224722e/dom/media/TrackUnionStream.h#17)
    - Consists of tracks coming from other SourceMediaStreams or TrackUnionStreams.
- [AudioNodeStream](https://dxr.mozilla.org/mozilla-central/rev/55d557f4d73ee58664bdf2fa85aaab555224722e/dom/media/webaudio/AudioNodeStream.h#34)
    - A stream for a WebAudio node. Most WebAudio nodes are built on top of this stream.

The main thread APIs communicate with the MediaStreamGraph through message passing to ensure lock free inter-thread communication. The way the underlying streams are set up to accommodate MediaStream and MediaStreamTrack needs, like cloning, track disabling, etc. is a bit complicated and nothing I will go through here (if you're interested anyway, read the [code comments for `DOMMediaStream`](https://dxr.mozilla.org/mozilla-central/rev/55d557f4d73ee58664bdf2fa85aaab555224722e/dom/media/DOMMediaStream.h#68)), but let me for now just mention that every MediaStream and MediaStreamTrack is backed by a TrackUnionStream. A MediaStreamTrack in addition has a `TrackID` - an integer uniquely identifying it within that TrackUnionStream.

Similarly,

- A `MediaStream` coming from `getUserMedia` is backed by a TrackUnionStream with a SourceMediaStream as input. The tracks in the `SourceMediaStream` are fed by our `MediaManager` and `MediaEngine` classes.
- A `new MediaStream()` is backed by a new TrackUnionStream with no inputs.
- A cloned track is backed by a new TrackUnionStream with the clonee's input track as input.
- A cloned stream is backed by a new TrackUnionStream with all the clonee's input tracks as inputs - new future tracks at the clonee won't get forwarded.

### <a name="stream-centered"></a> The old stream-centered way

Until now, everything has been centered around streams, including the assumption that a stream contains at most one video track and one audio track. This works fine for basic getUserMedia streams, but doesn't cater for complicated cases where you want to combine multiple tracks from different sources, like screen capture, camera capture, WebAudio destination nodes, canvas and media element capturing.

Other assumptions were:

- All tracks in a stream come from the same source. Security wise (cross origin access) we only have to care about that source. And it won't change throughout the lifetime of the stream.
- Any consumer of a stream (or track) has to use a MediaStreamListener (internal class) for data access, inadvertently getting notified about the activity of all tracks in the stream it is attached to. While this made sense for APIs that want all the tracks, it is now much more track centered, see for instance MediaStreamAudioSourceNodes and RTCPeerConnections. An example of an API that accepts streams is MediaRecorder, though its spec doesn't mention how to treat added and removed tracks much.

### <a name="msg-tracks"></a> MediaStreamGraph has been taught what tracks are

As mentioned above we had a `MediaStreamListener` class for listening to changes and new data for a stream. We now have a `MediaStreamTrackListener` class that is a first class citizen in the MediaStreamGraph. This could in essence be achieved before by using a `MediaStreamListener` and filtering on `TrackID` but now we can avoid those extra cycles by only raising events for the track in question.

The MediaStreamGraph now also has means of forwarding single tracks from one stream to another, where before you could only hook up streams to each other. A [`MediaInputPort`](https://dxr.mozilla.org/mozilla-central/rev/55d557f4d73ee58664bdf2fa85aaab555224722e/dom/media/MediaStreamGraph.h#973) is used to connect a stream to a `TrackUnionStream`. It can be locked to a single `TrackID` to forward only that track, it can forward all tracks, or it can forward all tracks except those blocked in the port.

APIs that take tracks can listen to the tracks it needs and doesn't have to worry about those tracks being removed from their parent stream, etc.

### <a name="track-sources"></a> We have MediaStreamTrackSources now

I implemented a general interface of a `MediaStreamTrackSource` through which all `MediaStreamTrack` instances can communicate. A track shares its `MediaStreamTrackSource` instance with all of its clones.

Previously there was a special MediaStream sub class for getUserMedia streams that allowed methods like applyConstraints to be called from a getUserMedia track up to the proper source. This would naturally not work if that track is contained in another stream type. Now with MediaStreamTrackSources we have generic access to the source from all tracks, and they don't have to go through a MediaStream on the way. All sources simply have to implement applyConstraints() in some way. Most sources ignore it since it doesn't apply, but for getUserMedia sources it gets applied appropriately.

Similarly, we do the same forwarding to the source of calls like `stop()` (after all clones have been stopped), `takePhoto()` for image capture, and various internal methods.

### <a name="security"></a> Security needs to be track-centered

Being stream-centered like we used to, a stream would be tied to the origin within which it was created. If you added a track from another origin to it we'd combine the principal () from the added track into the stream's current principal, upgrading it to the system principal if needed. If the same track was removed again, we wouldn't touch the principal. Should we have done so, we would probably have leaked real data because the track removal operation happens on main thread, and it would take a bounce of message passing to the MediaStreamGraph (and an iteration) and a bounce back to main thread, to actually get new media data to apply. During these two bounces the current data would be protected by a downgraded principal - hence we never downgraded it.

What I have implemented now is a system where we send the principal (main thread only) from the track source to the MediaStreamGraph, which then notifies the `MediaStreamTrack` when a new principal has been applied. In summary this allows us to do the following:

- When the source of a `MediaStreamTrack` changes its principal, for instance a 2d canvas that we drew something cross-origin to or a media element backed by a MediaSource played a chunk from another origin, we'd first combine the new principal with the old (could be an upgrade or a downgrade), and only when we have confirmation from the MediaStreamGraph that the new principal has been rendered, would we apply the new principal completely.
- On adding a track to a stream, we immediately combine the new track's principal into the stream's. We also keep around a second principal for the stream, which is solely for its video tracks - this for APIs that only want to access video content of the stream, for instance when you draw a media element playing a stream onto a canvas.
- On removing a track from a stream, we keep returning the old principal (we keep the removed track's principal in a set of "tracks pending removal") until the MediaStreamGraph has confirmed that we have now rendered another principal.

### <a name="intermittents"></a> Fixing tricky intermittent bugs - and some perf issues

All landings that cause major refactoring or changes in timing internally in the process tend to cause intermittent failures in automation.

After all the things above had been implemented we noticed how some automated tests started timing out, especially on platforms that were running on virtual machines. This wouldn't be reproduced on a similar setup on a local virtual machine either, a characteristic many of these intermittents share. [This particular performance issue](https://treeherder.mozilla.org/#/jobs?repo=try&revision=1678e9b22fa0) turned out to happen when sending a disabled screensharing-frame over an RTCPeerConnection. The fact that it was disabled was interesting and eventually pointed to two things:

- Frames going to be sent over an RTCPeerConnection did image format conversion if needed on the MediaStreamGraph thread (stalling the MSG in the worst case).
- Disabled frames had a separate buffer allocated and written to each time they came through - also on the MediaStreamGraph thread.

It was actually the latter case above that caused problems on this virtual machine. The allocation (screensharing frames tend to be high resolution!) took longer than a MediaStreamGraph iteration had budgeted. This lead to a much longer queue of frames to process on the next iteration, and then longer, and then longer, until we ran out of memory.

This was fixed by doing multiple things:

- [Don't queue up frames on the input side if the MediaStreamGraph cannot keep up.](https://hg.mozilla.org/mozilla-central/rev/b06d6ff27862)
- [Only pass on one black frame after disabling happens, so we don't do new allocations on each frame.](https://hg.mozilla.org/mozilla-central/rev/469e29166c55)
- [Send frames for image conversion onto a queue, processed by a separate thread - and drop frames if the separate thread is busy (> 1 frame queued up).](https://hg.mozilla.org/mozilla-central/rev/da8d6c4eab61)

With that done, the intermittents were gone. Jolly good!

### <a name="tldr"></a> Wait what? What does this actually mean?

Here's a jsfiddle demonstrating what you can do with track cloning. Once you've gotten your getUserMedia camera feed going, you can click the "Clone it 100 times!" button and if track cloning is supported by your browser, a second video should appear, playing back the clone (to the power of 100) of the original VideoStreamTrack. The clone can now be disabled and stopped independently from the original. Try it yourself below!

<script async src="https://jsfiddle.net/pehrsons/tx4dfhcp/embed/js,html,result/"></script>

---

Andreas Pehrson works with Mozilla's WebRTC team for Telenor Digital, as part of the joint WebRTC Competency Center.
