# Modifications to current proposal

## Problem to be solved

The current insertable streams proposal allows unlocking new use cases.
It has a number of weaknesses that could be adressed by updating the API to using TransformStream instead of ReadableStream/WritableStream.
Some advantages would be:
* Allow using native blocks for important features like SFrame.
* Simplify use of the API and avoid wrong use of the API.


## Approach

Current proposal is to directly expose ReadableStream and WritableStream on existing RTC objects.
This has some known drawbacks:
* The current API makes it natural to do processing on the main thread, which will lead to bad results for real time communication. The API should, by default, remove the need for web developers to do extra work to do the right thing (i.e. do processing on non main threads).
* ReadableStream can be cloned: page can quickly run out of memory if data is not consumed on all ReadableStream (original and cloned).
* Transfering a WebRTC ReadableStream may be difficult to optimize, in case ReadableStream is cloned, in case page started to read the ReadableStream. While it might not be impossible to optimize these cases, this adds to the complexity of supporting this API.

It seems best suited to expose TransformStream objects as the the goal of the API is to modify data being transmitted.
While it requires exposing more APIs, this should provide the following benefits:
* Follow a known pattern used by other APIs (https://encoding.spec.whatwg.org/#interface-mixin-generictransformstream, https://wicg.github.io/compression/#generic-transform-stream).
* Ability to add native transform streams, to implement SFrame in particular.
* Direct use of the API leads to optimal performances.

## Code Examples

1. Feature detection can be done as follows:

<pre>
const supportsInsertableStreams = window.RTCRtpSender && !!RTCRtpSender.prototype.transform;
</pre>

2. Create a MediaStreamTrack, add it to the RTCPeerConnection and connect the
SFrame Transform stream to the track's sender.

<pre>
const localStream = await navigator.mediaDevices.getUserMedia({video:true});
const [track] = stream.getTracks();
const pc = new RTCPeerConnection();
const videoSender = pc.addTrack(localStream.getVideoTracks()[0], localStream)
videoSender.transform = new SFrameSenderStream();

// Do ICE and offer/answer exchange.
...

// Do key exchange
const sframeKeys = ...

videoSender.transform.setEncryptionKey(sframeKeys);
</pre>

3. Do the corresponding operations on the receiver side.

<pre>
const sframeKeys = ...

const pc = new RTCPeerConnection();
pc.ontrack = e => {
  e.receiver.transform = new SFrameReceiverStream();
  e.receiver.transform.setEncryptionKey(sframeKeys);
}
</pre>

4. Do some JavaScript specific processing on the packets using RTCRtpReceiverStream/RTCRtpSenderStream

<pre>

// html file
const localStream = await navigator.mediaDevices.getUserMedia({video:true});
const [track] = stream.getTracks();

const pc = new RTCPeerConnection();
pc.worklet.addModule("worker-module.js")

const videoSender = pc.addTrack(localStream.getVideoTracks()[0], localStream)
videoSender.transform = pc.worlet.createSenderStream("myNoopSendTransformer");
videoSender.transform.postMessage("Hello controller");
videoSender.transform.onmessage = (e) => console.log("Message from video sender transform: " + e.data);

const pc = new RTCPeerConnection();
pc.ontrack = e => {
    e.receiver.transform = pc.worlet.createReceiverStream("myNoopReceiveTransformer");
}

// worker-module.js file content
function myNoopSendTransformer()
{
    return {
        start : (controller) => { controller.postMessage("Hello transform"); },
        transform : (frame, controller) => { controller.enqueue(frame) },
        flush : (controller) => { }
    };
}

function myNoopReceiveTransformer()
{
    return {
        start : (controller) => { },
        transform : (frame, controller) => { controller.enqueue(frame) },
        flush : (controller) => { }
    };
}
</pre>

## Additional API

### Extensions to RTC objects
<pre>
interface GenericRTCStream {
    readonly attribute ReadableStream readable;
    readonly attribute WritableStream writable;
};
partial interface RTCRtpSender {
    attribute GenericRTCStream? transform;
};
partial interface RTCRtpReceiver {
    attribute GenericRTCStream? transform;
};
</pre>

### SFrame additions
FIXME: Decide whether to expose explicit key IDs. Decide whether to expose options for encrypting only parts of the stream.
SFrame transforms that could either be used with key management implemented in JS or with some to-be-defined MLS-in-the-browser.
<pre>
// Sender
dictionary SFrameSenderOptions {
};
[Exposed=(Window,RTCWorklet)]
interface SFrameSenderStream : GenericRTCStream {
    constructor(optional SFrameSenderOptions options = { });
    Promise&lt;undefined&gt; setEncryptionKey(CryptoKey, optional unsigned long long keyID);
    Promise&lt;undefined&gt; ratchetEncryptionKey();
    Promise&lt;undefined&gt; setSigningKey(CryptoKey);
};

// Receiver
dictionary SFrameReceiverOptions {
};
[Exposed=(Window,RTCWorklet)]
interface SFrameReceiverStream : GenericRTCStream {
    constructor(optional SFrameReceiverOptions options = { });
    Promise&lt;undefined&gt; setEncryptionKey(CryptoKey, optional unsigned long long keyID);
    Promise&lt;undefined&gt; ratchetEncryptionKey();
    Promise&lt;undefined&gt; setSigningKey(CryptoKey);
};
</pre>

Note: this transform stream can be used in various contexts:
* With current RTCInsertableStreams proposal.
* With proposed RTCRtpSender and RTCRtpReceiver transform attribute.
* Used by JS RTC streams as one of the transform (with some limitations, requestIntraFrame in particular).

Plan: define SFrameReceiverStream with the hooks that JS RTC streams expose to web page.

### JS RTC Streams
FIXME: Need more thoughts. Think about moving messaging API to worklet, should we have all the APIs grouped in the controller...

<pre>
interface mixin MessageObject {
    undefined postMessage(any message, optional sequence<Transferable> transfer);
    attribute EventHandler onmessage;
};

// Sender
[Exposed=RTCWorklet]
interface RTCRtpSenderStreamController {
    undefined enqueue(RTCEncodedFrame chunk);
    undefined requestIntraFrame(); // to the local encoder.

};
RTCRtpSenderStreamController includes MessageObject;

interface RTCRtpSenderStream : GenericRTCStream {
};
RTCRtpSenderStream includes MessageObject;

// Receiver
[Exposed=RTCWorklet]
interface RTCRtpReceiverStreamController {
    undefined enqueue(RTCEncodedFrame chunk);
    undefined requestIntraFrame(); // to the remote encoder.
};
RTCRtpReceiverStreammController includes MessageObject;

interface RTCRtpReceiverStream : GenericRTCStream {
};
RTCRtpReceiverStream includes MessageObject;

// RTCPeerConnection
interface RTCWorklet {
    Promise&lt;RTCRtpSenderStream&gt; createSenderStream(DOMString name);
    Promise&lt;RTCRtpReceiverStream&gt; createReceiverStream(DOMString name);
};

partial interface RTCPeerConnection {
    Promise&lt;RTCWorklet&gt; createWorklet(DOMString scriptURL);
};
</pre>

## API already defined in insertable stream spec

The following are proposed interfaces defined by the insertable streams spec.

<pre>
// New enum for video frame types. Will eventually re-use the equivalent defined
// by WebCodecs.
enum RTCEncodedVideoFrameType {
    "empty",
    "key",
    "delta",
};

// New dictionaries for video and audio metadata.
dictionary RTCEncodedVideoFrameMetadata {
    long long frameId;
    sequence&lt;long long&gt; dependencies;
    unsigned short width;
    unsigned short height;
    long spatialIndex;
    long temporalIndex;
    long synchronizationSource;
    sequence&lt;long&gt; contributingSources;
};

dictionary RTCEncodedAudioFrameMetadata {
    long synchronizationSource;
    sequence&lt;long&gt; contributingSources;
};

// New interfaces to define encoded video and audio frames. Will eventually
// re-use or extend the equivalent defined in WebCodecs.
// The additionalData fields contain metadata about the frame and will
// eventually be exposed differently.
interface RTCEncodedVideoFrame {
    readonly attribute RTCEncodedVideoFrameType type;
    readonly attribute unsigned long long timestamp;
    attribute ArrayBuffer data;
    RTCVideoFrameMetadata getMetadata();
};

interface RTCEncodedAudioFrame {
    readonly attribute unsigned long long timestamp;
    attribute ArrayBuffer data;
    RTCAudioFrameMetadata getMetadata();
};

typedef (RTCEncodedAudioFrame or RTCEncodedVideoFrame) RTCEncodedFrame;
</pre>


