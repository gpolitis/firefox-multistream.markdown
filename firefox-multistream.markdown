# Firefox multistream and renegotiation in Jitsi

## Introduction

Jitsi Meet is an OpenSource (MIT) WebRTC JavaScript application that uses Jitsi
Videobridge to provide high quality, scalable video conferences. Jitsi
Videobridge is the server that receives video from every participant and then
relays it to everyone else. The IETF term for Jitsi Videobridge is a Selective
Forwarding Unit (SFU).  More commonly, however, such servers are known as
_video routers_ or MCUs. It is pretty much the same piece of tech as the one
Google use for Hangouts. In a Jitsi Videobridge based conference, all the
signaling goes through the _Focus_. It is responsible for managing media
sessions between each of the participants and the videobridge. Communication
between the Focus and a given participant is done through Jingle and between
the Focus and the Jitsi Videobidge through COLIBRI.

### How Jitsi Meet Works

## Implementation

### Battling with signaling

#### Plan A vs Plan B
     FF has started implementing support for [Plan A/Plan
     Unified](plan-unified) for their multi stream support while in Jitsi Meet
     we’ve implemented [Plan B](plan-b). Plan Unified has prevailed and it is
     incorporated in the [JSEP draft](jsep-draft).

#### Going berserk is not smart
     We're implementing the necessary
     translation logic in our focus so that it serves Plan A/Plan Unified SDP
     to FF (and Plan B to Chrome).

#### sdp-interop in theory

#### sdp-interop in practice
     getting it right was not instantaneous. Lots of debugging native code and
     help from Mozilla.

     Mozilla has helped in a number of occasions..
     * FF doesn't like it when we add/remove m-lines from its local SDP but
       further investigation is necessary.

     unfortunately, a Plan B offer/answer doesn't have enough information to
     rebuild an equivalent Plan A offer/answer.

     For example, if we want to convert a local answer in Plan A style to Plan
     B prior to handing it over to the application (we are called for example
     after a successful createAnswer), then we want to remember the m-line at
     which we've seen the local SSRCs. That's because when the application
     wants to call the SLD method, forcing us to do the inverse transformation
     (from Plan B to Plan A), we need to know to which m-line to assign the
     local SSRCs. We also need to know all the other m-lines that the original
     answer had and include them in the transformed answer as well.

     We resolved the above issue in an elegant way by caching/storing in Plan A
     style both the offer and the answer right before we transform an Plan A
     style offer/answer to Plan B and right after we transform a Plan B style
     offer/answer to Plan A. When we go from Plan B to Plan A we use the cached
     Plan A offer/answer and add the missing information from there.

### beyond signaling
    * dtls negotiation failures
      https://bugzilla.mozilla.org/show_bug.cgi?id=1132813
      https://github.com/bcgit/bc-java/pull/111
    * missing msids
    * goog-remb (and ulpfec/red)

     We've investigated what breaks the remote video playback in FF the
     issue/debugged FF and, as it turns out, we hit
     ​https://bugzilla.mozilla.org/show_bug.cgi?id=875922.

     With ulpfec/red enabled (jicofo/jvb enables it) Chrome will be sending
     packets with a different payload type and content (typically 116 red
     instead of 100 VP8) and FF is discarding those packets because it doesn't
     understand that content. This makes FF not to send PLIs because the jitter
     buffer remains empty all the time (FF is discarding most packets) and so
     request_key_frame = have_non_empty_frame = false all the time (in
     jitter_buffer.cc:1013).

     We have asked the FF team to enable ulpfec/red because

     * It fixes the video playback in Jitsi Meet
     * It improves the quality on unreliable connections
     * It simplifies a lot the use of ulpfec/red in scenarios with a server
       routing the traffic to multiple destinations (otherwise we have to
       translate the content in the server)
     * It should be supported in WebRTC core and probably it is not difficult
       to enable it

     Despite the above reasons, we need to think about alternative solutions,
     in case Mozilla doesn't want to enable ulpfec/red for whatever reason. So
     to support FF we 1. either have to implement some magic in the JVB to hide
     ulpfec/red when it talks to FF or 2. we completely disable both mechanisms
     (we have tested the latter and it works). We're thinking what's the best
     approach.

### bugs in our implementation

 We digged further into the supposedly synchronization issue we discovered
 earlier and it turns out it's not a synchronization problem but a non-zero
 offset bug somewhere in libjitsi, probably inside the SRTP transformers. The
 problem is described ​here. We have provided an efficient workaround and
 it should not affect the work on FF support any further.

 This finding does not change the state of the current ticket, we're still
 blocked by ​#1155246.

## Conclusion

### things left to do

#### simulcast
     * FF doesn't support MediaStream constructors yet (see ​here and
       ​here) but our simulcast reception implementation relies heavily on
       them. We are working on an alternative approach that doesn't require
       MediaStream constructors.
     * Unfortunately, FF doesn't support the RTCPeerConnectio.getStreamById()
       method but we rely on this method. We provided a JS implementation of
       the method.
  - some unresolved issues with FF

### Desktop sharing
### ORTC

