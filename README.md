# Lib-Kurento

A typescript library for simplifying the use of Kurento in Node.js.

## Motivation

Kurento Media server is controlled through the API it exposes, so we, the application developers use client implementations like `kurento-client-js` to interact with the media server.
The problem with `kurento-client-js` is that the package was automatically generated and therefore the source code is not the most readable. In addition the API might be difficult to handle as your application becomes larger and make use of more then one streaming protocol. Therefore I have created a simple library which simplifies the initialization process of the common endpoints types (And I even fixed some bugs).

## Install

```
npm i --save lib-kurento
```

## Usage

```typescript
import * as libKurento from 'lib-kurento';

const kmsUri = "ws://192.168.32.130:8888/kurento";
const rtpSdpOffer: string = "";     // get sdp from somewhere
const clientSdpOffer: string = "";  // get sdp from client using any kind of a signaling communication
const socket: WebSocket;

function sendServerIceCandidate(candidate) {
    // send ice candidate to client
    // for example:
    socket.send(JSON.stringify( { candidate: candidate } ))
}

function async main(){
    // connect to kurento
    const kurentoClient = await libKurento.connectToKurentoServer(kmsUri);

    // create a pipeline
    const pipeline = await libKurento.createPipeline(kurentoClient);

    // create RTSP and RTP endpoints
    const rtpEndpoint = new libKurento.RtpEndpointWrapper(pipeline, rtpSdpOffer);
    const rtspEndpoint = new libKurento.PlayerEndpointWrapper(pipeline, { uri: 'rtsp://192.168.1.100/stream1', networkCache: 0 /* low latency */ });

    // initialization simplified!
    await rtpEndpoint.init();
    await rtspEndpoint.init();

    // Accessing kurento-client`s instances is allowed as follows:
    await (rtpEndpoint.endpoint as any).setMaxOutputBitrate(0); // unlimited bitrate

    // start receiving feed from the rtsp source
    await rtspEndpoint.play();

    // create a WebRTC endpoint
    let webRtcEndpoint = new libKurento.WebRtcEndpointWrapper(pipeline, clientSdpOffer);

    // when the server's ice candidates are collected send them to the client
    webRtcEndpoint.on("ServerIceCandidate", sendServerIceCandidate);

    // init the WebRTC endpoint
    await webRtcEndpoint.init();

    // receive client ice candidates
    socket.addEventListener('message', (msg: any) => {
        const parsedMsg = JSON.parse(msg);

        webRtcEndpoint.addClientIceCandidate(parsedMsg.candidate);
    })

    await rtspEndpoint.connect(webRtcEndpoint as libKurento.WebRtcEndpointWrapper);
    await rtpEndpoint.connect(webRtcEndpoint as libKurento.WebRtcEndpointWrapper);
});
```
