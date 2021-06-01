---
title: AmberScript Live Transcription API

language_tabs: # must be one of https://git.io/vQNgJ
  - javascript: NodeJS

toc_footers:
  - <a href='https://www.amberscript.com/en/speech-to-text-api'>Sign Up for our Transcription API</a>

search: true

code_clipboard: true
---

# Introduction

Welcome to AmberScript's Realt Time Transcription API!

This page will guide you through setting up a websocket connection to our Real Time Transcription endpoint.

<aside class="notice">
    Our API only supports the Websocket protocol as defined in the rfc [here](https://datatracker.ietf.org/doc/html/rfc6455)
</aside>

## Authentication and initialisation

We will provide you with an `API_KEY` required to access our endpoints.

Authentication is done via an "INIT" type message which must be sent as the first message once the socket connection is established. Anything else will result in errors and the incoming connection will be terminated.

In addition to the API key, you must also pass the language, audio, and format configurations. If you omit the latter two, defaults will be chosen for you, which may result in a poor transcription quality.

The audio configuration must contain the sample rate of your audio input together with the [encoding](#supported-encodings)

And example of the INIT message, using the default values looks like this:

> Example A: INIT message

```json
{
  "messageType": "INIT",
  "language": "en",
  "apiKey": "<your-api-key>",
  "audioConfig": {
    "sample_rate": 16000,
    "encoding": "s16le"
  },
  "outputConfig": {
    "format": "transcription",
    "partials": false
  }
}
```

As of version 1.0, we only support the transcription format.

## Using the transcription API

<aside class="warning">
    Your client will be getting regular ping messages as defined in the [protocol specification](https://datatracker.ietf.org/doc/html/rfc6455#page-37). Make sure that your client will respond with pongs, or your connection will be terminated. If you are using a browser client, your client will automatically reply to pings. If you are using a Websocket library for your back-end client, then please make sure that the library is compliant with the protocol.
</aside>

```javascript
const SERVER_IP = "api.amberscript.com/real-time/";
// example only for node clients.
const WebSocket = require("ws");
const client = new WebSocket("wss://api.amberscript.com/real-time/");
client.on("open", () => {
  client.alive = true;
  client.send(
    JSON.stringify({
      messageType: "INIT",
      language: process.env.LANGUAGE,
      apiKey: process.env.API_KEY,
      audioConfig: {
        sample_rate: 48000,
        encoding: "s32le",
      },
      outputConfig: {
        partials: true,
      },
    })
  );
  interval();
});

client.on("message", (message) => onMessage(message));
client.on("pong", () => {
  client.alive = true;
});

function sendAudio(audioBuffer) {
  if (client.readyState) {
    cleint.send(audioBuffer);
  }
}
// set up ping pong from your client to the service
let interval = () =>
  setInterval(() => {
    if (!client.alive) {
      client.terminate();
      client = new WebSocket("wss://api.amberscript.com/real-time/");
    } else {
      if (client.readyState) {
        client.ping();
        client.alive = false;
      }
    }
  }, 30000);

function onMessage(data) {
  let objMsg = JSON.parse(data);
  switch (objMsg.type) {
    case "PartialResult":
      // object.message has the partial transcription for this segment
      break;
    case "FinalResult":
      // object.message has the final transcription for this segment
      break;
    case "Info":
      // object.message contains information regarding your configuration
      break;
    case "Warning":
      // object.message contains data regarding the warning and a reason
      break;
    case "Error":
      // object.message contains a reason for the error
      break;
  }
  return false;
}
```

After sending the initialisation message, you can then start sending the audio packets.

If you are not using a browser client, we recommend setting up a ping/pong mechanism so your client can detect connection drops and reconnect.

Once you start sending audio data, you should expect to receive tanscriptions.

If something goes wrong during setup, or if the server loses its connection to one of our workers, you will receive warnings. This does not mean that your session is lost, but it is possible that you will have to wait until connections are reestablished. You will be informed of the current retry count, maximum retries, and the time until a connection is retried.

### Return message types

This service will output different types of messages:


| Type   | Description                              |
|------- |----------------------------------------  |
| Error  | Standard auth, bad input errors along with connectivity issues, such as failure to connect to workers, or transcription errors. In the case of worker errors, clients will get the errors that were sent by the workers |
| Info   | General information about audio configuration |
| Warning | Only used when bad input was detected when setting up audio config. Will notify users of fallback configurations |
| PartialResult | Partial transcription received from the workers. It may, or may not be accurate, but it is quite fast |
| FinalResult | Final transcription for that segment of speech. Will not be recieving any more updates. |

Clients should expect the format of the messages to follow this pattern:

```json
{
  "type": "<String; one of the above types>",
  "message": "<String | Object<message: Array<Object>>>"
}
```

If you set the partials configured to `true`, then you will also receive partial transcriptions. You can see in the examples below that each message will have an id attached to it.

This can help you identify the sequence of the results. Each partial result will share the same id, until a final transcription is received. The final transcription will also share that id. Once you receive the final transcription, the id will reset and a new segment will start.

> Example B: Warning example

```json
{
  "type": "Warning",
  "message": {
    "type": "Warning",
    "current_counter": 1,
    "max_retries": 4,
    "current_timer": 10000,
    "message": "Workers are not available. Retrying connection in 10 seconds..."
  }
}
```

> Example C: Partial result type

```json
{
  "type": "PartialResult",
  "message": {
    "id": "682758c0-47bb-44b4-b4d3-93952580aa25",
    "version": "1.0",
    "segments": [
      {
        "words": [
          {
            "word": "Testing"
          },
          {
            "word": "testing"
          },
          {
            "word": "testing"
          },
          {
            "word": "testing"
          }
        ]
      }
    ],
    "transcript": "Testing testing testing testing"
  }
}
```

> Example D: Final result type

```json
{
  "type": "FinalResult",
  "message": {
    "id": "682758c0-47bb-44b4-b4d3-93952580aa25",
    "version": "1.0",
    "segments": [
      {
        "words": [
          {
            "word": "Testing,",
            "start": 0,
            "end": 1.23,
            "length": 0.6,
            "confidence": 1
          },
          {
            "word": "testing,",
            "start": 2.67,
            "end": 3.21,
            "length": 0.54,
            "confidence": 1
          },
          {
            "word": "testing,",
            "start": 3.21,
            "end": 3.72,
            "length": 0.51,
            "confidence": 1
          },
          {
            "word": "testing,",
            "start": 3.72,
            "end": 4.23,
            "length": 0.51,
            "confidence": 1
          }
        ],
        "start": 0,
        "end": 8.28,
        "length": 8.28
      }
    ],
    "transcript": "Testing, testing, testing, testing, "
  }
}
```

## Language Codes

| Language | Code |
| -------- | ---- |
| Dutch    | nl   |
| English  | en   |
| French   | fr   |
| German   | de   |

## Supported encodings

| Format | Description                             |
| ------ | --------------------------------------- |
| f32be  | PCM 32-bit floating-point big-endian    |
| f32le  | PCM 32-bit floating-point little-endian |
| s16be  | PCM signed 16-bit big-endian            |
| s16le  | PCM signed 16-bit little-endian         |
| s24be  | PCM signed 24-bit big-endian            |
| s24le  | PCM signed 24-bit little-endian         |
| s32be  | PCM signed 32-bit big-endian            |
| s32le  | PCM signed 32-bit little-endian         |
| u16be  | PCM unsigned 16-bit big-endian          |
| u16le  | PCM unsigned 16-bit little-endian       |
| u24be  | PCM unsigned 24-bit big-endian          |
| u24le  | PCM unsigned 24-bit little-endian       |
| u32be  | PCM unsigned 32-bit big-endian          |
| u32le  | PCM unsigned 32-bit little-endian       |

## Support

If you need any technical assistance, feel free to contact `info (at) amberscript (dot) com`
