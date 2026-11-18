# Twilio phone agent with AssemblyAI STT 3.7 Realtime Agent

Companion code for the blog post **[Twilio phone agent with AssemblyAI STT 3.7 Realtime Agent](https://www.assemblyai.com/blog/twilio-phone-agent-with-assemblyai)**.

Build an AI phone agent that answers real calls, using Twilio Voice and Media Streams for telephony and [AssemblyAI STT 3.7 Realtime Agent](https://www.assemblyai.com/products/streaming-speech-to-text) for real-time speech-to-text.

The detail that keeps this build short is the audio format. Twilio streams 8 kHz μ-law (mulaw) audio, and AssemblyAI's streaming API accepts `encoding=pcm_mulaw` at `sample_rate=8000` directly, so there's no resampling and no format conversion. [AssemblyAI Text-to-Speech](https://www.assemblyai.com/products/text-to-speech) closes the loop in the same format: ask it for `pcm_mulaw` at 8,000 Hz, and the agent's replies go back to Twilio with no conversion on your side either.

## Architecture

```text
    Incoming call
         │
      Twilio Voice
         │ TwiML → open Media Stream WebSocket
         ▼
    Your server (/media-stream WebSocket)
         │                          ▲
         │ mulaw 8 kHz audio        │ synthesized mulaw 8 kHz audio
         ▼                          │
    AssemblyAI STT 3.7          AssemblyAI TTS
    Realtime Agent                  ▲
         │ transcript + turn end    │ reply text
         ▼                          │
      OpenAI GPT-4o ────────────────┘
```

## Why AssemblyAI STT 3.7 Realtime Agent for phone calls?

Phone agents fail on the parts of a call that carry the most information: the policy number read out in chunks, the quiet "yes" after a confirmation question, the TV in the background. STT 3.7 Realtime Agent is the model the AssemblyAI docs recommend for [voice agents](https://www.assemblyai.com/solutions/voice-agents), and it's trained on real voice-agent and telephony conversations in which every speaker's role is labeled.

On AssemblyAI's benchmarks, compared with Universal-3.6 Pro Realtime:

<table style="width:100%; border-collapse:collapse; font-family:Arial, sans-serif; font-size:15px;">
  <thead>
    <tr style="background-color:#1E56A0; color:#fff;">
      <th style="padding:10px 14px; text-align:left; border:1px solid #ccc;">What goes wrong on a call (vs Universal-3.6 Pro Realtime)</th>
      <th style="padding:10px 14px; text-align:left; border:1px solid #ccc;">Universal-3.6 Pro Realtime</th>
      <th style="padding:10px 14px; text-align:left; border:1px solid #ccc;">AssemblyAI STT 3.7 Realtime Agent</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:10px 14px; border:1px solid #ccc;">Number errors (Voice Code Bench)</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">11.28%</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">2.27% (−80%)</td>
    </tr>
    <tr style="background-color:#f5f5f5;">
      <td style="padding:10px 14px; border:1px solid #ccc;">Digit-string errors</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">23.6%</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">3.6%</td>
    </tr>
    <tr>
      <td style="padding:10px 14px; border:1px solid #ccc;">Money-amount errors</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">13.3%</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">0.0%</td>
    </tr>
    <tr style="background-color:#f5f5f5;">
      <td style="padding:10px 14px; border:1px solid #ccc;">Recordings with at least one missed entity</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">71.3%</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">37.0%</td>
    </tr>
    <tr>
      <td style="padding:10px 14px; border:1px solid #ccc;">Words added while the caller is silent (per 1,000 words)</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">2.53</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">1.34 (−47%)</td>
    </tr>
    <tr style="background-color:#f5f5f5;">
      <td style="padding:10px 14px; border:1px solid #ccc;">One- and two-word replies dropped</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">0.29%</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">0.18% (−38%)</td>
    </tr>
    <tr>
      <td style="padding:10px 14px; border:1px solid #ccc;">False end-of-turn events</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">—</td>
      <td style="padding:10px 14px; border:1px solid #ccc;">−12%</td>
    </tr>
  </tbody>
</table>


*Source: the [AssemblyAI STT 3.7 Realtime Agent research post](https://www.assemblyai.com/blog/assemblyai-stt-3-7-realtime-agent-research), Nov 2026. The voice-agent benchmark is 12,460 scripted scenarios from 522 speakers, recorded as telephony audio with no, low or heavy background noise. Number and digit figures come from Voice Code Bench, a set dense in numbers and codes.*

Background speech falls by about half; it doesn't go to zero. Fewer false end-of-turn events means fewer callers cut off mid-thought.

## Prerequisites

- Python 3.11+
- An AssemblyAI API key ([free account](https://www.assemblyai.com/dashboard/signup)). One key covers speech-to-text and text-to-speech.
- A Twilio account with a phone number
- An OpenAI API key
- ngrok, for local development

## Quick start

```bash
git clone https://github.com/kelsey-aai/twilio-phone-agent-assemblyai
cd twilio-phone-agent-assemblyai

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt   # includes assemblyai>=1.10.1

cp .env.example .env
# Add ASSEMBLYAI_API_KEY and OPENAI_API_KEY to .env

uvicorn server:app --host 0.0.0.0 --port 8000
ngrok http 8000
```

### Configure Twilio

1. Open the Twilio Console and go to **Phone Numbers**.
2. Select your number, then **Voice Configuration**.
3. Set **A call comes in** to **Webhook**: `https://your-ngrok-url.ngrok.io/incoming-call`.
4. Call your Twilio number.

The `/incoming-call` route returns TwiML that tells Twilio to open a bidirectional Media Stream to your server:

```python
from fastapi import FastAPI, Request
from fastapi.responses import Response

app = FastAPI()


@app.post("/incoming-call")
async def incoming_call(request: Request):
    """Return TwiML that opens a Media Streams WebSocket to this server."""
    host = request.headers.get("host", "your-ngrok-url.ngrok.io")
    twiml = f"""<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Connect>
    <Stream url="wss://{host}/media-stream" />
  </Connect>
</Response>"""
    return Response(content=twiml, media_type="application/xml")
```

## AssemblyAI WebSocket parameters for Twilio

Twilio's Media Streams docs fix the inbound format: `audio/x-mulaw` at a sample rate of 8000, one channel. Match it on the AssemblyAI side and forward Twilio's audio as-is:

```python
# The API key goes in the Authorization header, not the URL:
# the `token` query param is only for temporary streaming tokens.
ASSEMBLYAI_WS_URL = (
    "wss://streaming.assemblyai.com/v3/ws"
    "?speech_model=assemblyai-stt-3-7-realtime-agent"
    "&encoding=pcm_mulaw"      # must match Twilio's audio format
    "&sample_rate=8000"        # must match Twilio's 8 kHz stream
)
```

Open the connection with your key in the `Authorization` header, then decode each Twilio `media` event and send the raw bytes:

```python
import base64
import json
import os

import websockets


async def bridge(twilio_ws):
    """Forward Twilio's mu-law audio to AssemblyAI unchanged."""
    async with websockets.connect(
        ASSEMBLYAI_WS_URL,
        additional_headers={"Authorization": os.environ["ASSEMBLYAI_API_KEY"]},
    ) as aai_ws:
        async for raw in twilio_ws.iter_text():
            msg = json.loads(raw)
            if msg.get("event") == "media":
                # Twilio sends base64-encoded mu-law; AssemblyAI wants the raw bytes
                await aai_ws.send(base64.b64decode(msg["media"]["payload"]))
            elif msg.get("event") == "stop":
                await aai_ws.send(json.dumps({"type": "Terminate"}))
                break
```

There's no `min_turn_silence` or `max_turn_silence` on that URL, on purpose. The [turn detection](https://www.assemblyai.com/docs/streaming/turn-detection) defaults now come from the `mode` preset (`balanced` unless you set one), and entity-aware endpointing holds a turn open while the caller reads out a phone number, code or email, without a longer silence timer. If your callers are deliberate speakers, add `&mode=max_accuracy` to the URL.

When your agent has just asked for something long, such as a card number, raise `min_turn_silence` for that one answer, then re-apply the mode to restore its defaults:

```python
import json


async def capture_long_entity(aai_ws):
    """Give the caller extra room for one answer, then restore the defaults."""
    await aai_ws.send(json.dumps({"type": "UpdateConfiguration", "min_turn_silence": 1000}))


async def restore_defaults(aai_ws):
    await aai_ws.send(json.dumps({"type": "UpdateConfiguration", "mode": "balanced"}))
```

> Previous model: `universal-3-6-pro` (Universal-3.6 Pro Realtime) is still fully supported. Moving a Twilio agent to 3.7 is a one-value change to `speech_model`; the encoding, sample rate and message types are the same.

## Speak replies with AssemblyAI Text-to-Speech

AssemblyAI Text-to-Speech is built for conversational voice agents and trained on millions of hours of audio, including real conversational data. For Twilio, configure the session for 8 kHz μ-law: `encoding="pcm_mulaw"` and `sample_rate=8000`. The model's native output is 24 kHz 16-bit PCM, and the server converts it before sending, so each chunk arrives raw, with no header, ready to base64-encode into a Twilio media event. Open the session when the call connects and reuse it for every reply:

```python
import base64
import json
import os

import assemblyai as aai
from fastapi import FastAPI, WebSocket

app = FastAPI()

# Twilio plays 8 kHz mu-law, so ask for exactly that: no transcoding on the way out
TTS_CONFIG = aai.TTSConfig(
    voice_id="aai-kyla",  # the default English voice
    language="english",
    encoding="pcm_mulaw",
    sample_rate=8000,
)


@app.websocket("/media-stream")
async def media_stream(ws: WebSocket):
    await ws.accept()
    # One TTS session per call: opened when the call starts, reused for every reply
    async with aai.AsyncTTSSession(
        TTS_CONFIG, api_key=os.environ["ASSEMBLYAI_API_KEY"]
    ) as tts:
        ...  # the AssemblyAI STT bridge; for each agent reply:
        # await speak_on_call(ws, tts, stream_sid, reply)


async def speak_on_call(ws: WebSocket, tts: aai.AsyncTTSSession, stream_sid: str, text: str):
    """Speak one reply on the call's TTS session and stream it into the Twilio call."""
    async for chunk in tts.speak(text):  # raw 8 kHz mu-law chunks
        await ws.send_text(json.dumps({
            "event": "media",
            "streamSid": stream_sid,
            "media": {"payload": base64.b64encode(chunk).decode()},
        }))
```

Keep one session open for the whole call. A session opened per reply pays the connection handshake on every turn and uses up your sessions-per-minute rate limit. The session closes after two minutes without a frame, so call `await tts.keep_alive()` during long caller silences; it lasts at most an hour. The [TTS accuracy and latency guide](https://www.assemblyai.com/docs/tts/getting-started/optimizing-accuracy-and-latency) has the full pattern.


## Let callers interrupt

To support barge-in, stop the reply as soon as the caller starts talking. On STT 3.7 Realtime Agent, the session sends a `SpeechStarted` message once per turn, just before the first partial. Background noise alone doesn't trigger it, so the docs recommend it as the barge-in signal. When it arrives during a reply, cancel synthesis and tell Twilio to drop the audio it has already buffered:

```python
import json

import assemblyai as aai


async def barge_in(twilio_ws, tts: aai.AsyncTTSSession, stream_sid: str):
    """The caller started talking over the agent: stop the reply everywhere."""
    await tts.cancel()  # stop synthesis of the reply in flight
    await twilio_ws.send_text(json.dumps({"event": "clear", "streamSid": stream_sid}))
```

Call it from your AssemblyAI receive loop when a message with `"type": "SpeechStarted"` arrives while a reply is playing. For that to work, run each reply as its own `asyncio` task so the loop keeps reading AssemblyAI messages while the agent talks.

## Extending the agent

### Give the model your agent's question

STT 3.7 Realtime Agent transcribes each caller turn with two kinds of context: what was said in earlier turns, and what your agent just said. After your agent asks "What's your account number?", the model is primed for a string of digits. Send your agent's last reply as `agent_context` after each agent turn:

```python
import json


async def send_agent_context(aai_ws, text: str):
    """Push the agent's last reply so the next caller turn is transcribed in context."""
    await aai_ws.send(json.dumps({
        "type": "UpdateConfiguration",
        "agent_context": text,  # up to 1,750 characters; each update replaces the last
    }))
```

The repo also sends the greeting as `agent_context` as soon as the session begins, so even the caller's first answer has context.

### Handle noisy lines with Voice Focus

[Voice Focus](https://www.assemblyai.com/docs/streaming/voice-focus) isolates the primary voice and suppresses background noise before transcription. It's available on the Agent model. Use `near-field` for close-talking mics, which covers most phone handsets and headsets, and `far-field` for distant mics, such as a speakerphone across a room:

```python
ASSEMBLYAI_WS_URL += "&voice_focus=near-field"
```

### Add keyterm prompting

Bias the model toward your brand, product and plan names with up to 100 keyterms. Each keyterm is its own `keyterms_prompt` query parameter. You can also update the list mid-call with `UpdateConfiguration`:

```python
ASSEMBLYAI_WS_URL += "&keyterms_prompt=YourBrand&keyterms_prompt=SpecialTerm"
```

### Add post-call transcription

For a full transcript after the call, run Twilio's recording through the pre-recorded API. Set the model explicitly:

```python
import assemblyai as aai

aai.settings.api_key = "YOUR_ASSEMBLYAI_API_KEY"

config = aai.TranscriptionConfig(speech_models=["universal-3-5-pro"])
transcript = aai.Transcriber(config=config).transcribe("https://example.com/recording.wav")
print(transcript.text)
```

If you already stream calls for transcription only, without an agent, our guide to [transcribing Twilio calls in real time](https://www.assemblyai.com/blog/transcribe-twilio-phone-calls-in-real-time-with-assemblyai) covers that simpler setup.


## Deploy to Railway or Render

```bash
# Railway
railway login && railway init && railway up

# Render: create a Web Service pointing to this repo
# Start command: uvicorn server:app --host 0.0.0.0 --port $PORT
```

Then point your Twilio number's webhook at `https://<your-host>/incoming-call`.

## What it costs

AssemblyAI STT 3.7 Realtime Agent is $0.45/hr, billed on session duration, with keyterm prompting included. AssemblyAI Text-to-Speech is $15 per 1M characters, billed on input characters, on the same account. Twilio Voice and your LLM bill separately through their own accounts. Current rates for every add-on are on the [pricing page](https://www.assemblyai.com/pricing).

## Where to go next

Twilio is one front door for an agent. To build the same loop without telephony, from a laptop mic or a browser tab, see the [Node.js voice agent tutorial](https://www.assemblyai.com/blog/node-js-voice-agent-with-assemblyai). If you'd rather hand telephony, orchestration and a visual builder to a platform, the [Vapi voice agent guide](https://www.assemblyai.com/blog/vapi-voice-agent-with-assemblyai) shows what Vapi's AssemblyAI transcriber supports today.

A phone agent is only as good as the numbers it hears. Transcription is the one stage where an error can't be fixed downstream, because the LLM never sees what the caller said. That's the case for spending your effort at the speech layer first.

## Resources

- [Companion repo](https://github.com/kelsey-aai/twilio-phone-agent-assemblyai)
- [Twilio Media Streams overview](https://www.twilio.com/docs/voice/media-streams)
- [Twilio Media Streams WebSocket messages](https://www.twilio.com/docs/voice/media-streams/websocket-messages)

## License

MIT
