# Firefox multistream and renegotiation for Jitsi Videobridge

## Introduction

Many of you WebRTC developers out there have probably already crossed the name
Jitsi Videobridge. Multi-party video conferencing is arguably one of the most
popular use cases for WebRTC and once you start looking for servers that allow
you to implement it, Jitsi's name is among the first you stumble upon.

For a while now, a number of JavaScript applications have been using WebRTC and
Jitsi Videobridge to deliver a rich conferencing experience to their users. The
bridge offers a very light-weight way (think routing vs. mixing) of conducting
high quality video conferences, so it has received it's fair share of
popularity.

The problem was that, until recently, applications using Jitsi Videobridge only
worked on a limited set of browsers: Chromium, Chrome and Opera.

This limitation is now gone!

After a few months of hard work from the Mozilla and Jitsi developers, both
Firefox and Jitsi have added the missing pieces and can now work with each
other.

While this wasn't the most difficult project on Earth, it wasn't quite a walk
in the park either, so in this post we'll try to tell you more about the dirty
details of this adventure!

## Some basics

Jitsi Videobridge is an OpenSource (LGPL) light-weight video conferencing
server. WebRTC JavaScript applications such as Jitsi Meet use Jitsi Videobridge
to provide high quality, scalable video conferences. Jitsi Videobridge receives
video from every participant and then relays some or all of it to everyone
else. The IETF term for Jitsi Videobridge is a Selective Forwarding Unit (SFU).
Sometimes such servers are also referred to as video routers or MCUs. The same
technology is used by most modern video conferencing systems like Google
Hangouts, Skype, Vidyo and many others.

From a WebRTC perspective, every browser establishes exactly one PeerConnection
with the videobridge. It sends and receives all audio and video data to and
from the bridge over that one PeerConnection.

![Jitsi Videobridge based 3-way call](images/201401-Jitsi-videobridge-diagram-FF.png)

In a Jitsi Videobridge based conference, all signaling goes through a separate
server-side application called the _Focus_. It is responsible for managing media
sessions between each of the participants and the videobridge. Communication
between the Focus and a given participant is done through Jingle and between
the Focus and the Jitsi Videobidge through
_[COLIBRI](http://xmpp.org/extensions/xep-0340.html)_.


### Unified Plan, Plan B and the answer to life, the universe and everything!

When discussing interoperability between Firefox and Chrome for multi-party
video conferences, it is impossible to not talk a little bit (or a lot!) about
[Unified Plan](https://tools.ietf.org/html/draft-roach-mmusic-unified-plan-00)
and [Plan B](https://tools.ietf.org/html/draft-uberti-rtcweb-plan-00). These
were two competing IETF drafts for the negotiation and exchange of multiple
media sources (AKA MediaStreamTracks, or MSTs) between WebRTC endpoints.
Unified Plan has been incorporated into the [JSEP
draft](https://tools.ietf.org/html/draft-ietf-rtcweb-jsep-09) and [Bundle
negotiation draft]
(https://tools.ietf.org/html/draft-ietf-mmusic-sdp-bundle-negotiation-19),
which are on their way to becoming IETF standards, while Plan B has expired in
2013 and nobody should care about it anymore ... at least in theory.

In reality Plan B lives on in Chrome and its derivatives, like Chromium and
Opera.  There's actually an issue in the Chromium bug tracker to add support
for [Unified Plan in
Chromium](https://code.google.com/p/chromium/issues/detail?id=465349), but
that'll take some time. Firefox, on the other hand, has, [as of
recently](https://hacks.mozilla.org/2015/03/webrtc-in-firefox-38-multistream-and-renegotiation/),
implemented Unified Plan.

Developers that want to support both Firefox and Chrome have to deal with this
situation and implement some kind of interoperability layer between Chrome and
and Firefox. Jitsi Meet is no exception of course; in the beginning it was a
no-brainer to assume Plan B because that's what Chrome implements and Firefox
didn't have [multistream
support](https://bugzilla.mozilla.org/show_bug.cgi?id=1095218).  As a result,
most of Jitsi's abstractions were built around this assumption.

The most substantial difference between Unified Plan and Plan B is how they
represent media stream tracks. Unified Plan extends the standard way of
encoding this information in SDP which is to have each RTP flow (i.e., SSRC)
appear on its own m-line. So, each media stream track is represented by its own
unique m-line.  This is a strict one-to-one mapping; a single media stream
track cannot be spread across several m-lines, nor may a single m-line
represent multiple media stream tracks.

Plan B takes a different approach, and creates a hierarchy within SDP; a m=
line defines an "envelope", specifying codec and transport parameters, and
a=ssrc lines are used to describe individual media sources within that
envelope. So, typically, a Plan B SDP has three channels, one for audio, one
for video and one for the data.

## Implementation

On the Jitsi side of things, it was obvious from the beginning that all the
magic should happen in the client. The Focus communicates with the clients
using Jingle which is in turn transformed into SDP and then handed over to the
browser. There's no SDP going around on the wire. Furthermore, there's no
signalling communication between the endpoints and the Jitsi Videobridge, it's
the Focus that mediates this procedure using
[COLIBRI](http://xmpp.org/extensions/xep-0340.html). So the question for the
Jitsi team was _what's the easiest way to go from Jingle to Unified Plan for
Firefox, given that we have code that assumes Plan B in all imaginable places_.

In its first few attempts the Jitsi team tried to provide general abstractions
wherever there was Plan B specific code. This could have worked, but at the
same period of time Jitsi Meet was undergoing some massive refactoring and the
inbound Unified Plan patches were constantly broken. On top of that, with the
multisteam support in Firefox being in its very early stages, Firefox was
breaking more often than it worked. Put those two ingredients together and
you're left with pretty much _0_ progress. One could even argue that the
progress was actually negative because of the wasted time.

Change of course, the Jitsi team decided to give a much more general solution
to the problem and deal with it at a lower level. The idea was to build a
PeerConnection adapter that would feed the right SDP to the browser, i.e.
Unified Plan to Firefox and Plan B to Chrome and that would give a Plan B SDP
to the application. Enter
[sdp-interop](https://www.npmjs.com/package/sdp-interop).

### An SDP interoperability layer

sdp-interop is a reusable npm module that offers the two simple methods:

* `toUnifiedPlan(sdp)` that takes an SDP string and transforms it into a
  Unified Plan SDP.
* `toPlanB(sdp)` that, not surprisingly, takes an SDP string and transforms it
  into a Plan B SDP.

The PeerConnection adapter wraps the `setLocalDescription()`,
`setRemoteDescription()` methods and the success callbacks of the
`createAnswer()` and `createOffer()` methods. If the browser is Chrome, the
adapter does nothing. If, on the other hand, the browser is Firefox the
PeerConnection adapter...

* calls the `toUnifiedPlan()` method of the sdp-interop module prior to calling
  the `setLocalDescription()` or the `setRemoteDescription()` methods, thus
  converting the Plan B SDP from the application to a Unified Plan SDP that
  Firefox can understand.
* calls the `toPlanB()` method prior to calling the `createAnswer()` or the
  `createOffer()` success callback, thus converting the Unified Plan SDP from
  Firefox to a Plan B SDP that the application can understand.

Here's a sample PeerConnection adapter:

    function PeerConnectionAdapter(ice_config, constraints) {
        var RTCPeerConnection = navigator.mozGetUserMedia
          ? mozRTCPeerConnection : webkitRTCPeerConnection;
        this.peerconnection = new RTCPeerConnection(ice_config, constraints);
        this.interop = new require('sdp-interop').Interop();
    }

    PeerConnectionAdapter.prototype.setLocalDescription
      = function (description, successCallback, failureCallback) {
        // if we're running on FF, transform to Unified Plan first.
        if (navigator.mozGetUserMedia)
            description = this.interop.toUnifiedPlan(description);

        var self = this;
        this.peerconnection.setLocalDescription(description,
            function () { successCallback(); },
            function (err) { failureCallback(err); }
        );
    };

    PeerConnectionAdapter.prototype.setRemoteDescription
      = function (description, successCallback, failureCallback) {
        // if we're running on FF, transform to Unified Plan first.
        if (navigator.mozGetUserMedia)
            description = this.interop.toUnifiedPlan(description);

        var self = this;
        this.peerconnection.setRemoteDescription(description,
            function () { successCallback(); },
            function (err) { failureCallback(err); }
        );
    };

    PeerConnectionAdapter.prototype.createAnswer
      = function (successCallback, failureCallback, constraints) {
        var self = this;
        this.peerconnection.createAnswer(
            function (answer) {
                if (navigator.mozGetUserMedia)
                    answer = self.interop.toPlanB(answer);
                successCallback(answer);
            },
            function(err) {
                failureCallback(err);
            },
            constraints
        );
    };

    PeerConnectionAdapter.prototype.createOffer
      = function (successCallback, failureCallback, constraints) {
        var self = this;
        this.peerconnection.createOffer(
            function (offer) {
                if (navigator.mozGetUserMedia)
                    offer = self.interop.toPlanB(offer);
                successCallback(offer);
            },
            function(err) {
                failureCallback(err);
            },
            constraints
        );
    };


### Beyond the basics

Like everything in life, sdp-interop is not "perfect", it makes certain
assumptions and it has some limitations. First and foremost, unfortunately, a
Plan B offer/answer does not have enough information to rebuild an equivalent
Unified Plan offer/answer. So, while it is easy (with some limitations) to go
from Unified Plan to Plan B, the opposite is not possible without keeping some
state.

Suppose, for example, that a Firefox client gets an offer from the Focus to
join a large call. In the _native_ create answer success callback you get a
Unified Plan answer that contains multiple m-lines. You convert it in a Plan B
answer using the sdp-interop module and hand it over to the app to do its
thing. At some point later-on, the app calls the adapter's
`setLocalDescription()` method. The adapter will have to convert the Plan B
answer back to a Unified Plan one to pass it to Firefox.

That's the tricky part because you can't naively put any SSRC in any m-line,
each SSRC should be put back into the same m-line that it was in the original
answer from the native create answer success callback. The order of the m-lines
is important too, so each m-line has to be in the same position it was in the
original answer from the native create answer success callback (which matches
the position of the m-line in the Unified Plan offer). It is also forbidden to
remove an m-line, instead they must be marked as inactive, if they're no longer
used.  Similar considerations have to be taken into account when converting a
Plan B offer to a Unified Plan one when doing renegotiation, for example.

sdp-interop solves this issue by caching both the most recent Unified Plan
offer and the most recent Unified Plan answer. When one goes from Plan B to
Unified Plan, sdp-interop uses the cached Unified Plan offer/answer and adds
the missing information from there. You can see
[here](https://github.com/jitsi/sdp-interop/blob/d4569a12875a7180004726633793430eccd7f47b/lib/interop.js#L175)
how exactly this is done.

Another limitation is that in some cases, a unified plan SDP cannot be mapped
to a plan B SDP. If the unified SDP has two audio m-lines (for example) that
have different media or transport attributes, these cannot be reconciled when
trying to squish them together in a single plan B m-section. This is why
sdp-interop can only work if the transport attributes are the same (ie; bundle
and rtcp-mux are being used), and if all codec attributes are exactly the same
for each m-line of a given media type. Fortunately, both Chrome and Firefox do
both of these things by default. (This is probably also part of the reason why
implementing unified plan won't be trivial for Chrome)

One last soft limitation is that the SDP interoperability layer has only been
tested when Firefox answers a call and not when it offers one because in the
Jitsi architecture the endpoints always get invited by the Focus to join a call
and never offer one. It would

### Far, far beyond the basics

Even with the SDP interoperability layer in place there's been a number of
difficulties which had to be overcome to bring FF support in Jitsi Videobridge
and Mozilla has been a *great* help to solve all of them (kudos to them!). In
most cases, the problem was easy to fix but it required time and effort to
identify what it was. For reference (and for fun!) we'll briefly describe a few
of those problems here.

One of the first unpleasant surprises was, for instance, that one day the Jitsi
prototype implementation decided to stop working all of a sudden. The DTLS
negotiation started failing soon after Mozilla [enabled DTLS 1.2 in
Firefox](https://bugzilla.mozilla.org/show_bug.cgi?id=1132813), and, as it
turned out, there was a problem in the DTLS version negotiation between Firefox
and our Bouncy Castle based stack. The RFCs are a little ambiguous in relation
to the record layer versions, but we assumed the openssl rules to be the
standard and [patched](https://github.com/bcgit/bc-java/pull/111) our stack to
behave according to those rules.

Another minor issue was that Firefox was [missing
msids](https://bugzilla.mozilla.org/show_bug.cgi?id=1095218) but Mozilla kindly
took care of that.

Next, the Jitsi team faced a very weird issue where the remote video playback
on the Firefox side froze or never started. The decoder was stalling. The weird
thing about this was that, in the test environment (LAN conditions), the
problem appeared to be triggered only when goog-remb was signaled in the SDP.
After some digging, it turned out that the problem had nothing to do with
goog-remb. The real issue was that the Jitsi Videobridge was relaying RED to
Firefox but the latter [doesn't currently support
ulpfec/red](https://bugzilla.mozilla.org/show_bug.cgi?id=875922) so nothing
made it through to the decoder. Signaling goog-remb probably tells Chrome to
encapsulate VP8 into RED right from the beginning of the streaming, even before
packet loss is detected (it's usually a good idea to activate it only when the
network conditions require it due to the overhead introduced by adding any
redundant data). The Jitsi Videobridge now decapsulates RED into plain VP8 when
it streams to Firefox (or any other client that doesn't support ULPFEC/RED).

The Jitsi team has also discovered and fixed a few issues in the Jitsi code
base, including a non-zero offset bug in our stack, probably inside the SRTP
transformers, that was causing SRTP auth failures.

Finally, and maybe most importantly, in a typical multistream enabled
conference, FF will be creating two (potentially three) sendrecv channels (one
for audio, one for video and potentially one for data) and N recvonly channels,
some for incoming audio and some for incoming video. Those recvonly channels
will send RTCP feedback with an internally generated SSRC. Here's where the
trouble begun.

Those internally generated SSRCs of the recvonly channels are known
*only* to FF. They're not known neither to the client app (they're not
included in the SDP), nor to the Jitsi Videobridge, nor to the other endpoints,
notably Chrome.

When using bundle, Chrome will discard RTCP traffic coming from
[unannounced SSRCs](https://code.google.com/p/webrtc/issues/detail?id=1772) as
it uses SSRCs to decide if an RTCP packet should go the the sending Audio or
the sending Video channel. If it can't find where to dispatch an RTCP packet,
it drops it. Firefox is not affected as it handles this differently. The webrtc
code that does the filtering is in [bundlefilter.cc](https://chromium.googlesource.com/external/webrtc/trunk/talk/+/6a741d7299ddc9eaa9c989729e02d5fdd145bc6e/session/media/bundlefilter.cc) which is not included in
mozilla-central. Unfortunately we have the same filtering/demux logic
implemented in our gateway.

This is hugely important because PLIs/RRs/NACKs/etc from recvonly channels
although they might reach Chrome, they're discarded, so the typical result
is a stalled decoder on the Firefox side. Mozilla fixed this in
[1160280](https://bugzilla.mozilla.org/show_bug.cgi?id=1160280) by exposing in
the SDP the SSRC for recvonly channels.

## Conclusion

It's been quite an interesting journey but we are almost there! One of the last
things that we have left to tackle is simulcast support in Firefox. Our
simulcast implementation relies heavily on MediaStream constructors but they're
not available in Firefox atm. The Jitsi team is working on an alternative
approach that doesn't require MediaStream constructors. One last major thing
that we're missing is desktop sharing but that is also currently baking!

In otherwords, Firefox and Jitsi are about to become best buddies!


