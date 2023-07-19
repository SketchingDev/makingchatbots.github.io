---
layout: blog-post
title:  "Automating how IVR call flows are tested"
date:   2021-03-03 00:00:00
categories: ivr call flow automation testing
image-base: /assets/images/posts/2021-03-03-automating-how-ivr-call-flows-are-tested
---

We are all familiar with the automated systems that answer our calls then ask a list of predefined questions to try and
resolve simple queries. They're known as Interactive Voice Response (IVR) call flows and are incredibly useful at
allowing call-centre staff to spend more time answering the difficult queries.

Having recently helped develop such IVR flows, I've been surprised by the lack of freely available tooling to automate
testing them. I've instead been subjected to manually calling the flow each time I wanted to test a change - a slow and
erroneous process.

To ease my suffering, I'm creating [IVR Tester](https://github.com/SketchingDev/ivr-tester), a tool that automates
calling IVR flows and traversing them based on what it hears. The benefits of such a tool are:

- Customers get value quicker. Those developing flows don't have to waste time manually calling them each time they them
  want to regression test it (reducing [cycle time](https://www.davefarley.net/?p=218))
- [Synthetic testing](https://en.wikipedia.org/wiki/Synthetic_monitoring) can be set up to routinely test that
  production call flows are working as expected from the customer's perspective
- End-to-end tests can test flows and their integrations, as part of a continuous delivery pipeline

Developing this tool has been an interesting journey and one I'd like to share with you from a high-level (hopefully
non-technical) perspective.

Let us begin by observing a human interacting with a call flow...

## How a human interacts with an IVR call flow

From the customer's perspective they:

1. Call the number from their phone
2. Hear a (often computer generated) voice asking a question, e.g. "Press 1 to make an appointment or press 2 to ..."
3. Respond by either speaking a word, or more likely by pressing a key on the keypad
4. Repeat from step 2 until their query is resolved, they're forwarded to a human, or they hang up in frustration

![Human calling IVR call flow and pressing keypad in response to questions]({{ page.image-base }}/human-interaction.png)

## Simulating a human interacting with an IVR call flow

Let's tackle each step and build out our application as we go. We start with a blank application that runs on any
computer.

<p style="text-align: center">
  <img class="ui image" alt="Blank application ready for functionality to be added" src="{{ page.image-base }}/application-1.png" />
</p>

### Calling the IVR flow

Calling a phone number is a complicated business, but thankfully companies like [Twilio](http://twilio.com/) abstract
away this complexity. In Twilio's case, they offer their
[MediaStream API](https://www.twilio.com/blog/media-streams-public-beta), which can call a phone number and then
establish a bi-directional audio stream with our application. Our application is then capable of listening to a call's
audio, and responding with its own audio.

Once plumbed in, it is a 3-step process to establish a call:

1. Our application requests that Twilio makes the phone call for us
2. Twilio does the complicated business of making the call
3. Twilio then establishes the bi-directional audio stream with our application

![Application establishing bi-directional audio stream with IVR flow]({{ page.image-base }}/application-2.png)

If you're interested, you can see what the code looks like to achieve this: [TwilioCaller.ts](https://github.com/SketchingDev/ivr-tester/blob/4d85b12d4d1187072145690e70f4a6a456401119/packages/ivr-tester/src/call/TwilioCaller.ts#L40-L61)

We now have our call. The next problem is trying to understand what the IVR call flow is asking us...

### Hearing the IVR flow

Millions of years of evolution and a basic education means that us humans can understand the flow's audio prompts with
ease, however computers haven't been so fortunate. To provide the application with a basic recognition capability, we're
going to transcribe the audio, as text is much easier to perform matches against (which we'll do later on).

The two main providers for transcribing audio streams are
[Google's Speech-to-Text](https://cloud.google.com/speech-to-text) and
[Amazon's Transcribe](https://aws.amazon.com/transcribe/). Although both APIs are very similar, Google's is more geared
towards telephony, as it allows you to provide the telephony standard audio format of 8-bit PCM mono uLaw
(8Khz sampling rate) and has a model pre-trained for phone calls.

![Application transcribing audio from Twilio]({{ page.image-base }}/application-3.png)

Once again, here's the code if you're interested: [GoogleSpeechToText.ts](https://github.com/SketchingDev/ivr-tester/blob/4d85b12d4d1187072145690e70f4a6a456401119/packages/transcriber-google-speech-to-text/src/GoogleSpeechToText.ts)

Now when we establish a call we can forward the audio to Google's Speech-to-Text service and receive a transcription
like below:

> Please enter the numbers from your post code. For example, if your post code is a 12 8 B c. He would enter 12 indeed.
> Please. Breath has when you are finished

The observant amongst you will have noticed the last part makes no sense at all, and herein lies the Achilles heel of
this approach. Transcriptions aren't perfect (although they can be trained), so we have to be lenient to these mistakes,
e.g. by matching on the section of text, "Please enter the numbers from your postcode".


### Responding to a question

Navigating a call flow is more reliably performed using your phone's keypad - we've all had a frustrating experience
with voice-recognition. Pressing your keypad works by producing a tone within the voice-frequency band that is unique
to each key - known as DTMF or touch tones.

Producing them programmatically is beyond my expertise, however [creating audio files that can be played back
isn't](https://github.com/SketchingDev/ivr-tester/tree/4d85b12d4d1187072145690e70f4a6a456401119/packages/ivr-tester/src/call/dtmf/raw).
These audio files can be easily created using [Sound eXchange (SoX)](http://sox.sourceforge.net/):

```shell
# Generates tone for '1' by combining 1209 Hz and 697 Hz sine waves
sox -n -r 8000 -t raw -e u-law -c 1 -b 8 1.raw synth 0.5 sin 1209 sin 697 pad 0 0.5
```

![Application responding to a call with DTMF tones]({{ page.image-base }}/application-4.png)

The application can now push the contents of these files onto the call's audio stream to simulate a phone's keypad
being pressed.

### When to play a response

Our application can now establish calls, transcribe their audio and simulate a keypad being pressed. However, it doesn't
yet know how to recognise an instruction in the transcription and what it should respond with - without this it cannot
traverse an IVR flow.

Naturally, we don't want to entangle these instructions within the application since they'll be different for each flow,
so instead we'll create external files that define how our application should traverse a flow, like so:

```typescript
const test = {
  name: "Customer is asked for their account number",
  instructions: inOrder([
    {
      whenPrompt: contains("please enter your account number"),
      then: press("1234567"),
    },
    // ...
  ]),
}
```

This example 'definition file' tells our application that when the call's transcription contains *"please enter your
account number"*, it should then simulate pressing the keys '1234567' on the keypad. Hopefully our definition is also
fluent enough to act as documentation for the flow too!


### Putting it all together

![Application that can establish, translate and respond to calls]({{ page.image-base }}/application-5.png)

After gluing it all together, and adding a few bells and whistles, you end up with something like
[IVR Tester](https://github.com/SketchingDev/ivr-tester).

![Terminal running a test with IVR Tester]({{ page.image-base }}/terminal-recording.gif)

## Conclusion

This article has focused on just one of many interesting problems which come with making a tool like this. However,
the most exciting of these will be integrating it into the testing of call flows used by hundreds of customers - a
process that I will be sure to blog about!
