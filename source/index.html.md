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
    Our API only supports the Websocket protocol as defined in the rfc <a href='https://datatracker.ietf.org/doc/html/rfc6455'>here.</a>
</aside>

## Authentication and initialisation

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

We will provide you with an `API_KEY` required to access our endpoints.

Authentication is done via an "INIT" type message which must be sent as the first message once the socket connection is established. Anything else will result in errors and the incoming connection will be terminated.

In addition to the API key, you must also pass the [language](#language-codes), audio, and format configurations. If you omit the latter two, defaults will be chosen for you, which may result in a poor transcription quality.

The audio configuration must contain the sample rate of your audio input together with the [encoding](#supported-encodings)

And example of the INIT message, using the default values looks like this:

As of version 1.0, we only support the transcription format.

## Using the transcription API

> Example B: Sending and receiving messages

```javascript
// example only for node clients.
const WebSocket = require("ws");
const client = new WebSocket("wss://api.amberscript.com/real-time/");
client.on("open", () => {
  client.alive = true;
  client.send(
    JSON.stringify({
      messageType: "INIT",
      language: "en",
      apiKey: "my-api-key",
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

function exitHandler(options, exitCode) {
  client.send(JSON.stringify({ messageType: "TRANSCRIPTION_FINISHED" }));
}
```

> Example C: Message structure

```json
{
  "type": "<String; one of the types from the table on the left>",
  "message": "<String | Object<message: Array<Object>>>",
  "ready": "<boolean; not present on result type messages>",
  "messageCode": "<String; not present on result type messages",
}
```

<aside class="warning">
    Your client will be getting regular ping messages as defined in the <a href='https://datatracker.ietf.org/doc/html/rfc6455#page-37'>protocol specification</a>. Make sure that your client will respond with pongs, or your connection will be terminated. If you are using a browser client, your client will automatically reply to pings. If you are using a Websocket library for your back-end client, then please make sure that the library is compliant with the protocol.
</aside>

After sending the initialisation message, you can then start sending the audio packets. However, it is recommended that your client waits until a message with the ready flag is set to true (see [short string message codes and ready indicators](#potential-errors-and-troubleshooting))

> Example D: Info message sequence

```json
{
  "type":"Info",
  "message":"Input configuration set to sample rate of 48000 for encoding s32le",
  "ready":false,
  "sampleRate":48000,
  "encoding":"s32le",
  "messageCode":"inputConfigurationInfo"
}

{
	"type": "INFO",
	"message": "Recognition started",
	"ready": true,
	"messageCode": "recognitionStartedInfo"
}
```

If you are not using a browser client, we recommend setting up a ping/pong mechanism so your client can detect connection drops and reconnect.

Once you start sending audio data, you should expect to receive tanscriptions.

If something goes wrong during setup, or if the server loses its connection to one of our workers, you will receive warnings. This does not mean that your session is lost, but it is possible that you will have to wait until connections are reestablished. You will be informed of the current retry count, maximum retries, and the time until a connection is retried.

Once your client is finished with streaming data, they should always send a "TRANSCRIPTION_FINISHED" message to our service. See example B, exitHandler function.

### Return message types

> Example E: Warning example

```json
{
  "type": "Warning",
  "current_counter": 1,
  "max_retries": 4,
  "current_timer": 10,
  "message": "Workers are not available. Retrying connection in 10 seconds...",
  "messageCode": "workerAcquisitionWarning",
  "ready": false
}

{
  "type": "WARNING",
	"current_counter": 1,
	"max_retries": 4,
	"message": "The worker has disconnected. The connection will be retried 4 times, with current counter at 1",
	"messageCode": "connectionLostWorkerAcquisitionWarning",
	"ready": false
}
```

> Example F: Partial result type

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

> Example G: Final result type

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

This service will output different types of messages:

| Type          | Description                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Error         | Standard auth, bad input errors along with connectivity issues, such as failure to connect to workers, or transcription errors. In the case of worker errors, clients will get the errors that were sent by the workers.                                                                                                                                                                                               |
| Info          | General information about audio configuration                                                                                                                                                                                                                                                                                                                                                                          |
| Warning       | Used when any disruptions of service occur (errors on the part of the workers/connection issues). You will also get warning if bad input was detected in your configuration.                                                                                                                                                                                                                                           |
| PartialResult | Partial transcription received from the workers. Partial transcripts will not be having alignment information, nor will it have any confidence information. However, you can expect to receive partial transcriptions almost immmediately. Do be aware that these transcriptions are likely to change until a final transcription is received. You can use the unique identifier for a segment to track these changes. |
| FinalResult   | Final transcription for that segment of speech. Will not be recieving any more updates for this segment. Will contain alignment and confidence information                                                                                                                                                                                                                                                             |

Clients should expect the format of the messages to follow the pattern visible in Example C (do note that some messages may or may not contain extra properties that clients may choose to ignore).

If you set the partials configured to `true`, then you will also receive partial transcriptions. You can see in the examples below that each message will have an id attached to it.

This can help you identify the sequence of the results. Each partial result will share the same id, until a final transcription is received. The final transcription will also share that id. Once you receive the final transcription, the id will reset and a new segment will start.

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

## Potential errors and Troubleshooting

| Error message                                                                                                                                                                            | Description                                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Something went wrong when setting up a connection to our services. Please try again later, or contact support.                                                                           | This error indicates that workers are not available right now. You should generally attempt new connections in 1-2 minutes after this error occurs. If the problem persists, please do contact us.                                             |
| Your client seems to have been inactive for more than one minute. If this is a mistake, try again or contact support                                                                     | This error is very unlikely to occur. Your implementation might not be following the WebSocket protocol, and is not responding to ping control frames as detailed in the warning [here](#using-the-transcription-API).                         |
| Something went wrong when processing your input. Please try again later, or contact support.                                                                                             | This error might be because of faulty input. Please evaluate your input configuration and the raw output of your microphone. If the issue cannot be resolved, please do communicate this information if you decide to ask for techical support |
| Your language configuration is missing. Please send your API key and language under the format {language: <langugage>, apiKey: &lt;your-api-key&gt;}. This connection will now be closed | Make sure your JSON message is stringified and that the language configuration uses the correct key                                                                                                                                            |
| Your api key is missing. Please send your API key and language under the format {language: &lt;langugage&gt;, apiKey: &lt;your-api-key&gt;}. This connection will now be closed          | Make sure your JSON message is stringified and that the API key uses the correct key                                                                                                                                                           |
| This language is not supported.                                                                                                                                                          | See [languages](#language-codes) for our supported languages. If you are interested in additional languages, please contact our support.                                                                                                       |
| This API key is not recognised. Closing connection...                                                                                                                                    | Double check your API key. If there are no typos, then please contact support in order to get your API key again.                                                                                                                              |
| Your status is &lt;different status than ACTIVE&gt;. Closing connection...                                                                                                               | Your account has been disabled. Contact support in order to clarify the issues with your accound                                                                                                                                               |
| You must send your messages under a JSON format. This connection will now be closed                                                                                                      | Please stringify your JSON messages before sending them.                                                                                                                                                                                       |
| Something went wrong                                                                                                                                                                     | This error message is the equivalent of a 500 status code. If the situation is not resolved after reattempting connections, then please contact our support.                                                                                   |
| Something went wrong when transcribing your input.                                                                                                                                       | This error message is the equivalent of a 500 status code, indicating that something went wrong with the processing of our models' outputs. Please contact support if you ever encounter this message.                                         |

### Short string message codes and ready indicators

Each client message will be accompanied by a short string message code in order to aid clients in identifying the information that each message contains. These message codes are only available for **INFO**, **WARNING**, and **ERROR** message types.

**INFO**, **WARNING**, and **ERROR** message types will always hold a ready flag. However, there is only one message that will report if a worker is ready to start transcribing input. While it is recommended that you only start streaming when the ready flag is true, it is not mandatory. While not common, it is possible that there are delays between the worker acquisition and the start of the recognition process, which can lead to data loss.

| messageCode                            | Message type | Explanation                                                                                                                                                                                                                                                |
| -------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| noWorkerAvailableError                 | ERROR        | This message code indicates that no transcription was found, even after attempting reconnection                                                                                                                                                            |
| audioProcessingError                   | ERROR        | This message code indicates that a fatal error occurred when processing your audio input.                                                                                                                                                                  |
| initialisationFailedError              | ERROR        | This message code indicates that a fatal error occurred when setting up the connection to our transcription workers.                                                                                                                                       |
| inactiveClientError                    | ERROR        | This message code indicates that your client is not alive. If this message is received, make sure that your client is responding to pings coming from our service.                                                                                         |
| transcriptionError                     | ERROR        | This message code indicates that the worker has encountered an issue when transcribing your audio input.                                                                                                                                                   |
| noLanguagePresentError                 | ERROR        | This message code indicates that your client is not specifying any language information.                                                                                                                                                                   |
| missingOrInvalidKeyError               | ERROR        | This message code indicates an authentication error.                                                                                                                                                                                                       |
| languageNotAvailableError              | ERROR        | This message code indicates that the language you have selected is not available.                                                                                                                                                                          |
| inactiveUserError                      | ERROR        | This message code indicates that the user is not active.                                                                                                                                                                                                   |
| messageFormatNotJSONError              | ERROR        | This message code indicates that the INIT message format cannot be parsed.                                                                                                                                                                                 |
| defaultInputWarning                    | WARNING      | This message code indicates that there was no input configuration provided in the INIT message, and certain defaults have been chosen for you. The transcription you receive is likely to have quality issues, or you will not receive any results at all. |
| defaultOutputWarning                   | WARNING      | This message code indicates that there was no output configuration provided in the INIT message. You will not receive partial results, and the transcription format is set to "transcription".                                                             |
| noPartialConfigurationFoundWarning     | WARNING      | This message code indicates that an output configuration was found, but there was no partial configuration. You will only receive final results.                                                                                                           |
| noTranscriptionFormatFoundWarning\*    | WARNING      | This message code indicates that an output configuration was found, but there was no transcription format configuration. The transcription format is set to "transcription".                                                                               |
| workerAcquisitionWarning               | WARNING      | This message code indicates that no transcription worker was found on initialisation, and the service is currently searching for available workers.                                                                                                        |
| connectionLostWorkerAcquisitionWarning | WARNING      | This message code indicates that the connection to the transcription worker was lost, and the service is currently searching for a new worker.                                                                                                             |
| connectionReestablishedInfo            | INFO         | This message code indicates that a new worker was acquired.                                                                                                                                                                                                |
| recognitionStartedInfo\*\*             | INFO         | This message code indicates that the worker has started the recognition process.                                                                                                                                                                           |
| inputConfigurationInfo                 | INFO         | This message code indicates that your INIT message was received. Your input configuration will be reported back to the client.                                                                                                                             |

\* This message can be safely ignored at v1.0, as the only supported format is "transcription".

\** This is the only message that will have the ready flag set to **true**.

## Extra message metadata

Apart from the standard message structure (see Example C), some messages will contain additional information in order to aid clients in dealing with certain situations, such as the **WARNING** types. Some messages will only contain information that will confirm client settings (i.e **INFO** types). All the metadata in the table below can be safely ignored by clients if chosen so.

| messageCode                           	| Message type 	| Additional metadata                                                                                                                                                                                                                                                                                         	|
|---------------------------------------	|--------------	|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|
| inactiveUserError                     	| ERROR        	|  - userStatus: String type indicating the current status of the user used by the client.                                                                                                                                                                                                                    	|
| defaultInputWarning                   	| WARNING      	|  - sampleRate: int type indicating the default sample rate that will be used for your input<br/> - encoding: String type indicating the default encoding that will be used for your input                                                                                                                       	|
| defaultOutputWarning                  	| WARNING      	|  - partials: boolean type indicating the default partial configuration for your output<br/>  - format: String type indicating the default transcription output format                                                                                                                                            	|
| workerAcquisitionWarning              	| WARNING      	|  - current_counter: int type indicating the current attempt counter<br/>  - max_retries: int type indicating the maximum amount of retries until the client connection is terminated (once current_counter > max_retries)<br/>  - current_timer: int type indicating when the next attempt will be made (in seconds)  	|
| connectionLostWorkerAquisitionWarning 	| WARNING      	| Same metadata as the previous message, with the exception of the current_timer property; when a connection is lost, attempt #1 is made immediately                                                                                                                                                          	|
| inputConfigurationInfo                	| INFO         	|  - sampleRate: int type indicating the sample rate received from the client in the INIT message<br/>  - encoding: String type indicating the encoding received from the client in the INIT message                                                                                                               	|


## Support

If you need any technical assistance, feel free to contact `info (at) amberscript (dot) com`.
