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
* ReadableStream can be cloned using ReadableStream.tee: page can quickly run out of memory if data is not consumed on all ReadableStream (original and cloned).
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
videoSender.transform = new SFrameTransform();

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
  e.receiver.transform = new SFrameTtansform();
  e.receiver.transform.setEncryptionKey(sframeKeys);
}
</pre>

4. Do some JavaScript specific processing on the packets using RTCRtpReceiverStream/RTCRtpSenderStream

<pre>

// html file
const localStream = await navigator.mediaDevices.getUserMedia({video:true});
const [track] = stream.getTracks();

const pc = new RTCPeerConnection();
pc.addModule("worker-module.js")

const videoSender = pc.addTrack(localStream.getVideoTracks()[0], localStream)
videoSender.transform = new RTCRtpScriptTransform(pc, "myNoopTransformer");
videoSender.transform.port.postMessage("Hello transformer");
videoSender.transform.onmessage = (e) => console.log("Message from video sender transform: " + e.data);

const pc = new RTCPeerConnection();
pc.ontrack = e => {
    e.receiver.transform = new RTCRtpScriptTransform(pc, "myNoopTransformer");
}

// worker-module.js file content
class myNoopTransformer extends RTCRtpScriptTransformer {
{
    constructor() { }
    start(readableStream, writableStream) {
        this.port.postMessage("Hello Transform");
        readableStream.pipeTo(writableStream);
    }
};
registerRTCRtpScriptTransformer("myNoopTransformer", myNoopTransformer);
</pre>

## Additional API

### Extensions to RTC objects
<pre>
typedef (SFrameTransform or RTCRtpScriptTransform) RTCRtpTransform;
partial interface RTCRtpSender {
    attribute RTCRtpTransform? transform;
};
partial interface RTCRtpReceiver {
    attribute RTCRtpTransform? transform;
};
</pre>

### SFrame additions
SFrame transforms that could either be used with key management implemented in JS or with some to-be-defined MLS-in-the-browser.
<pre>
dictionary SFrameTransformOptions {
};
[Exposed=(Window,DedicatedWorker)]
interface SFrameTransform {
    constructor(optional SFrameTransformOptions options = { });

    readonly attribute ReadableStream readable;
    readonly attribute WritableStream writable;

    Promise&lt;undefined&gt; setEncryptionKey(CryptoKey key, optional unsigned long long keyID);
    Promise&lt;undefined&gt; ratchetEncryptionKey();
};
</pre>

Note: this transform stream can be used in various contexts:
* Directly with proposed RTCRtpSender and RTCRtpReceiver transform attribute.
* By JS RTC streams as one of the transform.

#### Advantages

A native SFrame transform has a number of advantages compared to a JavaScript specific implementation:
* Security: a SFrame native implementation does not require exposing encryption keys to JavaScript.
  Secure key managers built on frameworks like MLS could for instance generate CryptoKey that are not extractable.
* Simplicity: end-to-end encryption is a known use case that is expected to be deployed widely.
  Providing a native API will be beneficial
for web developers in terms of ease of use, interoperability and maintenance.
* Efficiency: introducing JavaScript introduces more flexibility with a potential memory cost.
  A native SFrame implementation will allow optimizing the implementation in terms of memory and processing.

### JS RTC Streams
<pre>
[Exposed=Window]
interface RTCRtpScriptTransform {
    constructor(Worker context, DOMString name, optional object transfromOptions);

    readonly attribute MessagePort port;
    attribute EventHandler onerror;
};

[Exposed=DedicatedWorker]
interface RTCRtpScriptTransformer {
    constructor(optional object transformOptions);

    undefined start(ReadableStream readableStream, WritableStream writableStream, RTCRtpScriptTransformerContext context);

    readonly attribute MessagePort port;
};

enum RTCRtpScriptTransformerContextSide { "receiver", "sender" };
enum RTCRtpScriptTransformerContextMediaType { "audio", "video" };

[Exposed=DedicatedWorker]
interface RTCRtpScriptTransformerContext {
    readonly attribute RTCRtpScriptTransformerContextSide side;
    readonly attribute RTCRtpScriptTransformerContextMediaType mediaType;

    undefined requestIntraFrame();
};

callback RTCRtpScriptTransformerConstructor = RTCRtpScriptTransformer(optional object transformOptions);

[Exposed=DedicatedWorker]
partial interface DedicatedWorkerGlobalScope {
    undefined registerRTCRtpScriptTransformer(DOMString name, RTCRtpScriptTransformerConstructor processorConstructor);
};
</pre>

Potential additions for Worklets:
<pre>
partial interface RTCRtpScriptTransform {
    constructor(RTCPeerConnection context, DOMString name, optional object transfromOptions);
};

partial interface RTCPeerConnection {
    Promise&lt;undefined&gt; addModule(DOMString scriptURL);
};

[Exposed=RTCRtpWorklet]
partial interface RTCRtpWorkletGlobalScope {
    undefined registerRTCRtpScriptTransformer(DOMString name, RTCRtpScriptTransformerConstructor processorConstructor);
};
</pre>

### Using SFrame transform within a JS transform

A SFrame transform can be used as part of a JS transform since it can be created in a script trsansform.
The SFrame transform can expose additional information through the RTCEncodedVideoFrame objects it produces.
This allows JS applications to do specific processing, for instance in error cases.
<pre>
enum SFrameState { "valid", "missing-key", "invalid", "need-key-frame" };
partial interface RTCEncodedAudioFrame {
    readonly attribute SFrameState? sframeState;
};
partial interface RTCEncodedVideoFrame {
    readonly attribute SFrameState? sframeState;
};
</pre>
<pre>
// worker-module.js file content
class mySFrameDecryptionTransformer extends RTCRtpScriptTransformer {
    constructor() {
    }
    start(readable, writable, context) {
        this._sframeTransform = new SFrameTransform();
        this._sframeTransform.readable.pipeTo(new WritableStream({ write : (frame) => this.processPostDecryption(writable, context, frame) }));
        readable.pipeTo(this._sframeTransform.writable);
    }
    processPostDecryption(writable, context, frame) => {
        if (frame.sframeState === "valid") {
            // Do specific decrypted frame processing before sending to decoder.
            ...
            writable.enqueue(frame);
            return;
        }
        if (frame.sframeState === "need-key-frame") {
            context.requestIntraFrame();
            return;
        }
    }
}
registerRTCRtpScriptTransformer("mySFrameDecryptionTransformer", mySFrameDecryptionTransformer);
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