# Firefox multistream and renegotiation for Jitsi Videobridge

## Introduction

Many of you WebRTC developers out there, have probably already crossed the name
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

In a Jitsi Videobridge based conference, all signaling goes through a separate
server-side application called the Focus. It is responsible for managing media
sessions between each of the participants and the videobridge. Communication
between the Focus and a given participant is done through Jingle and between
the Focus and the Jitsi Videobidge through COLIBRI.


### Unified Plan, Plan B and the answer to life, the universe and eveything!

When discussing interoperability between Firefox and Chome for multi-party
video conferences, it is impossible to not talk a little bit (or a lot!) about
[Unified Plan](https://tools.ietf.org/html/draft-roach-mmusic-unified-plan-00)
and [Plan B (https://tools.ietf.org/html/draft-uberti-rtcweb-plan-00). These
were two competing IETF drafts for the negotiation and exchange of multiple
media sources (AKA MediaStreamTracks, or MSTs) between WebRTC endpoints.
Unified Plan is being incorporated in the [JSEP
draft](https://tools.ietf.org/html/draft-ietf-rtcweb-jsep-09) and is on its way
to becoming an IETF standard while Plan B has expired in 2013 and nobody should
care about it anymore ... at least in theory

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
didn't have multistream support. As a result most of our abstractions were
built arround this assumption.

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

In our case, it was obvious from the beginning that all the magic should happen
in the client. The Focus communicates with the clients using Jingle which we
transform into SDP to feed to the browser. There's no SDP going around on the
wire. Furthermore, there's no signalling communication between the endpoints
and the Jitsi Videobridge, it's the Focus that mediates this procedure using
COLIBRI. So the question was _what's the easiest way to go from Jingle to
Unified Plan for Firefox, given that we have code that assumes Plan B in all
imaginable places_.

In our first few attempts we tried to provide general abstractions wherever
there was Plan B specific code. This could have worked, but at the same
period of time Jitsi Meet was undergoing some massive refactoring and our
Unified Plan patches were constantly broken. On top of that, with the
multisteam support in Firefox being in its very early stages, Firefox was
breaking more often than it worked. Put those two ingredients together and
you're left with pretty much _0_ progress. One could even argue that the
progress was actually negative because of the wasted time.

Change of course, we decided to give a much more general solution to the
problem and deal with it at a lower level. The idea was to build a
PeerConnection adapter that would feed the right SDP to the browser, i.e.
Unified Plan to Firefox and Plan B to Chrome and that would give a Plan B SDP
to the application. Enter [sdp-interop](https://github.com/jitsi/sdp-interop).

### An SDP interoperability layer

sdp-interop is a reusable npm module that offers the two simple methods:

* `toUnifiedPlan(sdp)` that takes an SDP string and transforms it into a Unified Plan
  SDP.
* `toPlanB(sdp)` that, not surprisingly, takes an SDP string and transforms it
  to Plan B SDP.

The PeerConnection adapter wraps the `setLocalDescription()`,
`setRemoteDescription()` methods and the success callbacks of the
`createAnswer()` and `createOffer()` methods. If the browser is Chrome, the
adapter does nothing. If, on the other hand, the browser is Firefox the
PeerConnection adapter...

* calls the `toUnifiedPlan()` method of the sdp-interop module prior to calling the
  `setLocalDescription()` or the `setRemoteDescription()` methods, thus
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
Unified Plan offer/answer. So, while it is easy to go from Plan B to Unified
Plan, the opposite is not possible without keeping some state.

Suppose, for example, that a Firefox client gets an offer from the Focus to
join a large call. In the _native_ create answer success callback you get a
Unified Plan answer that contains multiple m-lines. You convert it in a Plan B
answer using the sdp-interop module and hand it over to the app to do its
thing. At some point later-on, the app calls the adapter's
`setLocalDescription()` method. The adapter will have to convert the Plan B
answer back to a Unified Plan one to pass it to Firefox.

That's the tricky part because you can't naively put any SSRC in any m-line,
each SSRC has to be put back into the same m-line that it was in the original
answer from the native create answer success callback. The order of the m-lines
is important too, so each m-line has to be in the same position it was in the
original answer from the native create answer success callback (which matches
the position of the m-line in the Unified Plan offer). It is also forbidden to
remove an m-line, instead they must be marked as inactive, if they're no longer
used.  Similar considerations have to be taken into account when converting a
Plan B offer to a Unified Plan one when doing renegotiation, for example.

We solved this issue by caching both the most recent Unified Plan offer and the
most recent Unified Plan answer. When we go from Plan B to Unified Plan we use
the cached Unified Plan offer/answer and add the missing information from
there. You can see
[here](https://github.com/jitsi/sdp-interop/blob/d4569a12875a7180004726633793430eccd7f47b/lib/interop.js#L175)
how we do this exactly.

Another soft limitation (in the sense that it can be removed given enough
effort) is that we require bundle and rtcp-mux for both Chrome and Firefox
endpoints, so all the media whatever the channel is, go through a single port.
To completely remove this limitation it would require changes in our Focus
because it currently allocates channels in the Jitsi Videobridge the Plan B
way, i.e. one channel for audio, one for video and one for data while Firefox
would expect different channels for each stream it receives. We could however
partially remove this limitation by requiring bundle and rtcp-mux only from
Firefox and bundling video streams separately from audio streams.

One last soft limitation is that we have currently tested the interoperability
layer only when Firefox answers a call and not when it offers one because in
our architecture endpoints always get invited to join a call and never offer
one.

### Far, far beyond the basics

Even with the SDP interoperability layer in place there's been a number of
difficulties which we had to overcome to bring FF support in Jitsi Videobridge
and Mozilla has been a great help to solve all of them.

* We've had [DTLS negotiation
  failures](https://github.com/bcgit/bc-java/pull/111) soon after Mozilla
  [enabled DTLS 1.2 in
  Firefox](https://bugzilla.mozilla.org/show_bug.cgi?id=1132813).
* Firefox was [missing msids](https://bugzilla.mozilla.org/show_bug.cgi?id=1095218)
  but Mozilla kindly took care of that.
* We've had a very weird issue where the remote video playback froze or never
  started in Firefox and that was triggerd when goog-remb was signaled. As it
  turned out, this was because Firefox [doesn't currently support
  ulpfec/red](https://bugzilla.mozilla.org/show_bug.cgi?id=875922) and the
  packets Chrome was sending were being discarded by Firefox because they were
  encapsulated in RED and they had the wrong payload type. The Jitsi
  Videobridge now decapsulates VP8 when it streams to Firefox.
* We discovered a non-zero offset bug in our stack, probably inside the SRTP
  transformers, that was causing SRTP auth failures at the receiving side and
  for which we have provided an efficient workaround.

## Conclusion

It's been quite an interesting journey but we are almost there! One of the
things that we have left to tackle are the issues arising from  #1155246 . The
remote video freezes after a while in Firefox because it (Firefox) doesn't push
down the network the PLIs that the receive only video channels are generating.

We also don't have simulcast support in FF because it doesn't support
MediaStream constructors yet but our simulcast reception implementation relies
heavily on them. We are working on an alternative approach that doesn't require
MediaStream constructors.

One last major thing that we're missing is desktop sharing but that is also
currently baking!

In otherwords, Firefox and Jitsi are about to become best buddies!


