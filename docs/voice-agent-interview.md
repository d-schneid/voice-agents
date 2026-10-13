# Voice agent interview preparation guide

This guide is a provider-agnostic framework for designing, reviewing, and
explaining production voice agents. It is intentionally specific about
boundaries and failure modes without assuming a particular model vendor,
telephony carrier, WebRTC SDK, orchestration framework, or cloud.

The central design rule is:

> Treat conversation as a real-time, interruptible control loop, not as a
> sequence of independent speech-to-text, LLM, and text-to-speech requests.

That rule affects the media path, turn detection, cancellation, tool
authority, lifecycle handling, safety controls, observability, and cost model.

## 1. A platform-agnostic reference architecture

Separate the system into a **media plane**, a **control plane**, and an
**application/tool plane**. The names are conceptual; the implementation may
use one process or many services, but the contracts should remain explicit.

```text
User microphone / phone
        |
        | media plane: encoded audio, RTP/WebRTC frames, jitter, timestamps
        v
Ingress + session edge
        |
        | normalized audio frames and lifecycle events
        v
Realtime conversation runtime
  - VAD / endpointing / turn state
  - recognizer or speech-to-speech model
  - response generation
  - speech output scheduler
  - interruption and cancellation
        |
        | typed intents, tool calls, policy decisions, audit events
        v
Control plane
  - session/auth/policy
  - tool broker and confirmation
  - workflow state and idempotency
  - durable transcript/event journal
        |
        v
Application services and authoritative systems
  - CRM, booking, payment, identity, notifications
```

### Responsibilities and boundaries

| Area | Owns | Must not silently own |
| --- | --- | --- |
| Media ingress/egress | Audio transport, codec/sample-rate normalization, jitter handling, timestamps, connection state | Business authorization or irreversible side effects |
| Conversation runtime | Turn state, model streaming, output scheduling, cancellation, conversational context | Direct unrestricted database or payment access |
| Control plane | Authentication, authorization, policy, typed events, confirmations, idempotency, durable workflow state | Guessing facts from an untrusted transcript |
| Tool broker | Validating arguments, checking authority, executing approved operations, returning typed results | Bypassing application policy because a model requested it |
| Application systems | Source-of-truth records and business invariants | Trusting natural-language intent without validation |
| Observability | Correlation, metrics, traces, redacted audit evidence | Persisting raw sensitive audio by default |

The runtime can be colocated with the media edge for latency, while
authoritative tools should remain behind a narrow, authenticated interface.
Use explicit timeouts and backpressure at every streaming boundary.

### A typical turn

1. The ingress adapter emits timestamped audio frames and a `session.started`
   event.
2. VAD and endpointing classify speech activity without deciding business
   intent.
3. The runtime streams partial recognition or model input and emits only
   provisional text/events until a turn boundary is established.
4. The response planner produces incremental output. The speech scheduler
   starts playback as soon as a safe chunk is available.
5. A barge-in signal cancels generation and playback, invalidates stale output,
   and returns control to listening.
6. A tool request goes through the broker, where schema, identity, policy,
   confirmation, and idempotency are checked.
7. Durable state records the outcome before the agent claims success to the
   user.

## 2. WebRTC versus telephony

These transports solve different problems. WebRTC is generally the better
choice for an application-controlled browser or mobile experience; telephony
is necessary when the user should call an ordinary number or receive a PSTN
call.

| Dimension | WebRTC | Telephony/PSTN |
| --- | --- | --- |
| Access | Browser/mobile permissions and a signaling path | Carrier, SIP, trunk, or telephony provider |
| Audio | Usually low-latency packet media with negotiated codecs | Codec and transcoding constraints; often narrowband |
| Identity | Application session and account identity | Caller ID is a routing hint, not proof of identity |
| Network behavior | ICE, STUN/TURN, NAT traversal, changing networks | Carrier routing, SIP state, provider webhooks, call transfers |
| UX | Rich controls, mute, device selection, data channel | DTMF, prompts, holds, transfer, voicemail, disconnect |
| Compliance | Browser consent and app privacy controls | Recording announcements, regional calling/consent rules |
| Failure modes | Permission denied, ICE failure, device change | Busy, no-answer, carrier failure, fraud, call spoofing |

### WebRTC interview points

- Signaling authenticates and authorizes a session; it is not the media
  itself.
- ICE connectivity may require STUN and TURN. Treat TURN as a normal
  production path, not an exceptional failure.
- Handle device changes, mute state, network migration, and tab suspension.
- Keep signaling/control messages separate from audio frames. A data channel
  can carry typed events, but the server must validate them as untrusted input.
- A browser identity token should be bound to an application session and
  expire or be revocable.

### Telephony interview points

- A provider webhook may start or update a call, but webhook authenticity,
  replay protection, and call-state validation are still required.
- Caller ID does not establish the caller's account authority. Use a separate
  identity step for sensitive operations.
- Expect codec conversion, packet loss, silence suppression, DTMF, transfer,
  hold, voicemail, and one-way-audio conditions.
- Do not assume a phone number's country or area code is the user's location.
- Recording and transcription require an explicit, jurisdiction-aware policy.
- Put spend limits, destination restrictions, rate limits, and anomaly
  detection around outbound calls to reduce toll fraud.

### Transport adapter rule

The conversation runtime should consume a normalized interface such as:

```text
AudioFrame {
  session_id
  sequence
  captured_at
  sample_rate
  channels
  encoding
  pcm_or_encoded_bytes
}

TransportEvent =
  connected | disconnected | muted | unmuted | network_changed |
  dtmf_received | transfer_requested | transport_error
```

WebRTC and telephony adapters can then differ in codec and lifecycle details
without changing tool authorization or conversation policy.

### Browser-server WebSockets and transport adaptation

A demo may use a browser WebSocket as its real-time, bidirectional connection:

```text
browser microphone/audio player
          <==== WSS: audio + typed events ====> server
             -> agent runtime -> tools/providers
```

This is a reasonable teaching and prototype architecture because one
persistent connection can carry client events, server events, audio chunks,
heartbeats, and lifecycle notifications. It is not the only production
transport, and it is important to distinguish three cases:

| Connection | What it carries | Typical use |
| --- | --- | --- |
| Browser WebSocket | Application-defined audio/control messages over WSS | Simple demo, server-controlled media, captions and events |
| WebRTC | Negotiated realtime media plus optional data channel | Browser/mobile production audio with low-latency media features |
| Telephony provider WebSocket | Provider-defined audio and control protocol | PSTN/SIP calls through Twilio, Vonage, or another carrier |

The agent runtime should not depend on the demo's message format. Normalize
each transport into the `AudioFrame` and `TransportEvent` contracts above.
Then the same turn manager, tool broker, workflow state, and playback policy
can operate behind a browser WebSocket, a WebRTC media track, or a
telephony adapter.

#### Twilio and Vonage examples

- **Twilio Media Streams** supports unidirectional streams for receiving call
  audio and bidirectional streams for receiving caller audio while sending
  audio back to the call. Its WebSocket protocol has lifecycle, media, DTMF,
  mark, and stop messages; applications must validate the `X-Twilio-Signature`
  and use secure WebSockets. The documented bidirectional stream has provider
  constraints, including one inbound track and inbound-only DTMF.
- **Vonage Voice WebSockets** use a persistent full-duplex connection where
  binary messages carry audio and text messages carry JSON events/commands.
  An NCCO `connect` action selects the `wss://` endpoint and audio format.
  Vonage supports connection authorization, playback-buffer control, DTMF
  events, and event webhooks for connection failures/disconnects.

These are not identical protocols. They differ in codec/sample-rate
negotiation, message envelopes, authentication, playback buffering, DTMF,
disconnect semantics, and webhook behavior. The adapter owns those details;
the conversation runtime should see normalized events and must still apply
its own authorization, sequencing, backpressure, cancellation, and
idempotency rules.

For an interview, say:

> “The demo's browser WebSocket is a transport choice, not the agent
> architecture. I would keep a typed transport interface and swap the adapter
> for WebRTC in a browser production path or Twilio/Vonage media WebSockets
> for telephony. I would not forward provider packets directly into business
> logic: I would authenticate the connection, validate message types and
> sequence/timestamps, normalize audio, bound buffers, handle disconnects and
> barge-in, and keep tools behind the server-side policy boundary.”

Official references: [MDN WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API),
[WebRTC overview](https://webrtc.org/getting-started/overview),
[Twilio Media Streams](https://www.twilio.com/docs/voice/media-streams),
[Twilio WebSocket messages](https://www.twilio.com/docs/voice/media-streams/websocket-messages),
and [Vonage Voice WebSockets](https://developer.vonage.com/en/voice/voice-api/concepts/websockets).

## 3. Speech-to-speech, cascaded, and hybrid pipelines

There are three useful families. None is universally best.

### Direct speech-to-speech

A multimodal/realtime model consumes audio and emits audio, sometimes with
intermediate events or text.

**Strengths**

- Potentially shorter turn-taking path.
- Preserves prosody and acoustic context more naturally.
- Fewer separately managed model boundaries.

**Tradeoffs**

- Less transparent intermediate representation.
- Provider-specific session semantics may be stronger.
- Exact transcript, redaction, deterministic replay, and audit behavior can
  require extra side channels.
- Tool authorization is still an application responsibility; audio output is
  not evidence that a side effect occurred.

### Cascaded ASR -> text model -> TTS

Audio is recognized, text is planned, and text is synthesized.

**Strengths**

- Clear transcripts and inspectable intermediate decisions.
- Easy to swap recognizer, language model, or synthesizer independently.
- Mature controls for redaction, evaluation, deterministic workflows, and
  text-based policy checks.

**Tradeoffs**

- More buffering and serialization points.
- Errors can compound across recognition, reasoning, and synthesis.
- Prosody and emotion may be less continuous across components.
- Each component needs cancellation, timeout, and retry semantics.

### Hybrid

Use realtime speech interaction for low-risk conversational turns, while
routing identity, structured extraction, high-risk workflows, or audit-critical
steps through text and typed control paths.

Examples:

- Speech-to-speech for greetings and FAQ, cascaded transcription for a
  regulated disclosure.
- Streaming audio interaction, but a deterministic workflow state machine for
  a cancellation, refund, or account-change flow.
- Realtime model for candidate tool arguments, then a typed policy service for
  authorization and execution.

### How to choose in an interview

Start from requirements rather than a benchmark:

1. Is an exact transcript required?
2. Must a decision be reproducible from stored evidence?
3. Is the action reversible?
4. Is latency more important than inspection and portability?
5. What are the audio conditions, languages, accents, and interruption needs?
6. Does the provider expose enough lifecycle and cancellation primitives?
7. Can the application retain control of identity, tools, and policy?

Do not present a vendor's reported latency as a universal property of an
architecture. End-to-end delay depends on capture, codec, network, queueing,
VAD, model load, tokenization, synthesis, buffering, and playback.

### Provider examples: AssemblyAI and Cartesia

AssemblyAI and Cartesia are useful examples of specialized providers in a
cascaded voice stack. They are not interchangeable model vendors:

| Provider | Relevant capability | Typical place in the design |
| --- | --- | --- |
| AssemblyAI | Streaming and pre-recorded speech-to-text, turn events, speech understanding, and broader voice infrastructure | ASR/STT before the orchestrator |
| Cartesia | Realtime text-to-speech through Sonic, speech-to-text and turn detection through Ink, voice models, and managed agents | TTS after the orchestrator, or an integrated voice-agent platform |

The component-level flow is:

```text
incoming audio -> AssemblyAI streaming STT -> LLM/workflow/tools
              -> incremental text -> Cartesia streaming TTS -> playback
```

This design is valuable in an interview because each boundary is visible and
replaceable. The recognizer can emit partial and final transcripts; the
orchestrator can validate or redact text before reasoning; and the synthesizer
can start producing audio from incremental LLM text instead of waiting for a
complete answer. The cost is extra coordination: each WebSocket, queue,
timeout, cancellation path, audio format, and provider failure must be
managed.

Important provider-specific considerations:

- **AssemblyAI streaming:** treat the WebSocket as a billable session and
  explicitly terminate it on every normal and abnormal call path. For browser
  use, mint a short-lived/temporary token from the backend rather than
  exposing the API key. Preserve the distinction between provisional
  transcript updates and finalized turns.
- **Cartesia streaming TTS:** push text chunks over a streaming connection and
  consume audio chunks through a bounded playback queue. On barge-in, stop
  playback, invalidate the current generation, and discard queued audio from
  that generation. Keep the audio format and sample rate explicit at the
  provider boundary.
- **Latency claims:** treat provider-advertised first-audio or end-to-end
  numbers as vendor-reported. Measure time to first useful audio at actual
  playback under the chosen transport, model, voice, network, and buffering
  configuration.
- **Managed products:** both providers also expose broader voice-agent
  capabilities. Choosing a managed agent can reduce integration work; choosing
  separate STT/TTS components preserves more control over prompts, tools,
  policy, fallback providers, and intermediate evidence.

An interview-ready comparison is:

> “I would evaluate AssemblyAI and Cartesia as pipeline components, not choose
> them from a single benchmark. I would compare endpointing and interruption
> behavior, first partial/final transcript timing, first playable audio,
> language and accent coverage, codecs and sample rates, session limits,
> cancellation semantics, data retention and region, reliability, pricing,
> rate limits, and fallback options. I would keep their APIs behind internal
> STT/TTS interfaces so the business workflow does not depend on vendor event
> names.”

Official references: [AssemblyAI documentation](https://www.assemblyai.com/docs),
[AssemblyAI streaming STT](https://www.assemblyai.com/docs/streaming/getting-started/transcribe-streaming-audio),
[Cartesia overview](https://docs.cartesia.ai/get-started/overview.md), and
[Cartesia realtime TTS](https://docs.cartesia.ai/get-started/realtime-text-to-speech-quickstart).

### Async generators as streaming stage contracts

The LangChain tutorial expresses each pipeline stage as an asynchronous stream
transformation rather than a batch function:

```text
audio stream -> STT events -> agent text events -> TTS audio events
```

Conceptually, a stage has a contract like:

```python
async def stage(input_stream):
    async for item in input_stream:
        result = await process(item)
        yield result
```

`async for` waits for the next item without blocking the event loop. `yield`
emits that item immediately while preserving the generator's state so it can
continue when the next item arrives. In the tutorial:

1. The STT stage sends audio chunks to AssemblyAI and receives partial and
   finalized transcript events concurrently.
2. The agent stage starts after a finalized transcript and yields response
   tokens as the LangChain agent generates them.
3. The TTS stage sends text chunks to Cartesia while concurrently receiving
   synthesized audio chunks.
4. The WebSocket endpoint forwards each audio chunk to the browser for
   playback instead of waiting for a complete response.

The result is pipelining:

```text
without streaming:
record -> transcribe -> generate -> synthesize -> play

with streaming:
audio arrives -> transcript finalizes -> tokens arrive -> audio plays
                                      while later output is still generated
```

This reduces **time to first playable audio**, not necessarily total task
completion time. The tutorial does not begin agent reasoning from every
provisional transcript; it waits for a usable/finalized turn. That is a
reasonable correctness trade-off, while speculative generation is an
optional optimization with cancellation and wasted-work costs.

In the LangChain implementation, `RunnableGenerator` composes these
async-generator stages and `pipeline.atransform(...)` connects the browser
audio stream to the final event stream. That is framework-specific plumbing;
the durable architectural idea is a typed stream of normalized events between
replaceable stages.

### Python synchronous versus asynchronous handling

Python `asyncio` is a strong fit for the voice loop because the workload is
mostly concurrent I/O: WebSocket reads/writes, STT events, model tokens, TTS
audio, timers, and cancellation. It does not make CPU-bound Python code
parallel; it coordinates tasks while they wait for I/O.

Use asynchronous code for the realtime path:

```python
async def handle_session(websocket):
    async with provider_session() as stt:
        async for frame in websocket.iter_bytes():
            await stt.send_audio(frame)
```

Use synchronous code for short, deterministic, non-blocking operations such as
argument validation, state transitions, formatting, and pure policy checks.
Do not call blocking HTTP clients, synchronous SDKs, file operations, database
drivers, or slow CPU work directly from an async event loop. They pause every
session sharing that event loop.

If a synchronous dependency cannot be replaced, isolate it deliberately:

```python
result = await asyncio.to_thread(sync_lookup, customer_id)
```

For CPU-heavy work, prefer a process pool, a native implementation that
releases the GIL, or a separate worker service. `to_thread` is primarily an
I/O/blocking-call escape hatch, not a general CPU-parallelism solution.

### Structured concurrency and cancellation

The STT producer and consumer, TTS sender and receiver, and session lifecycle
tasks form one ownership tree. Use structured task management such as
`asyncio.TaskGroup` where the supported Python version permits it, and make
failure of one child cancel or reconcile its siblings. Use `asyncio.timeout`
for bounded provider operations rather than waiting indefinitely.

Every session should have a cancellation path:

1. Browser/phone disconnects or barge-in is detected.
2. Child tasks are cancelled.
3. `CancelledError` is allowed to propagate after `finally` cleanup.
4. Provider WebSockets, queues, timers, and playback handles are closed.
5. Durable state records whether work completed, was cancelled, or is unknown.

Do not swallow cancellation in a broad exception handler. Cancellation is a
control signal, not an ordinary provider failure. A synchronous side effect
already running in a worker thread may not stop when its awaiter is cancelled,
so the operation still needs an idempotency key and authoritative reconciliation.

### Sync/async boundaries for tools

Keep the voice loop async even when a business tool is implemented
synchronously. Wrap blocking reads or SDK calls at the boundary, but keep
authorization, confirmation, idempotency, and result validation in the
application policy layer. Do not let the model decide whether a blocking
operation is safe merely because it can call the tool.

For an interview, distinguish:

| Work | Preferred handling |
| --- | --- |
| WebSocket/audio/model I/O | `async`/`await`, async iterators, bounded queues |
| Short validation/state transition | synchronous function called from the event loop |
| Blocking HTTP/SDK/database call | async client, or `asyncio.to_thread` as a contained fallback |
| CPU-heavy denoising/encoding/ML | process/worker pool or dedicated service |
| Irreversible tool effect | idempotent durable operation with reconciliation |

This is more precise than saying “async is always better.” The decision is
about keeping the event loop responsive while making ownership, cancellation,
timeouts, and side-effect semantics explicit.

### Highest-value Python concurrency patterns

These are the common concepts most worth explaining in an interview:

**Bounded queues and backpressure**

```python
audio_queue = asyncio.Queue(maxsize=32)
```

The producer should wait, drop stale speculative work, or enter a documented
degraded mode when the queue is full. An unbounded queue turns a slow STT/TTS
provider or browser into a memory leak. Apply the same rule to audio frames,
transcript events, tool work, and playback.

**Structured task groups and bounded fan-out**

Use `asyncio.TaskGroup` for tasks with one lifecycle owner. Use
`asyncio.gather` only when operations are genuinely independent, and use an
`asyncio.Semaphore` to cap concurrent provider or tool calls:

```python
limit = asyncio.Semaphore(8)

async def bounded_lookup(item):
    async with limit:
        return await lookup(item)
```

Parallelize independent read-only work; serialize or transactionally protect
conflicting writes.

**One owner for mutable session state**

Give each live session one worker/actor that owns turn state, playback
generation, event ordering, and tool-result application. Other tasks send
typed messages to that owner. If shared mutation is unavoidable, use a
per-session `asyncio.Lock`; do not use one global lock for all calls.

**Graceful shutdown**

On disconnect, deployment shutdown, or provider failure: stop admitting work,
cancel child tasks, close provider sessions, drain or checkpoint only safe
work, flush audit/usage state, and release resources. A task that is cancelled
must not leave an open WebSocket, timer, queue consumer, or uncommitted
side-effect ambiguity.

The concise interview distinction is:

> “Asyncio gives me cooperative concurrency for I/O. Queues provide flow
> control, task groups provide lifecycle ownership, semaphores bound fan-out,
> and a per-session owner prevents races. Threads isolate blocking I/O,
> processes/services handle CPU-heavy work, and idempotency reconciles effects
> that cannot be safely cancelled.”

### WSGI versus ASGI

The tutorial's long-lived browser WebSocket endpoint is an **ASGI-shaped**
problem, not a traditional WSGI request:

| Interface | Core model | Voice-agent relevance |
| --- | --- | --- |
| **WSGI** | Synchronous HTTP request/response callable returning an iterable of bytes | Useful for conventional REST endpoints and existing synchronous Python applications; not a natural native interface for long-lived bidirectional WebSockets |
| **ASGI** | Async application receiving and sending protocol events for HTTP and WebSocket connection lifecycles | Natural fit for persistent voice sessions, streaming audio, disconnect events, cancellation, and concurrent connections |

ASGI provides the Python server/application boundary; it does not make the
voice agent correct or scalable. The application still needs bounded queues,
provider lifecycle management, authentication, backpressure, and durable
workflow state. Keep blocking WSGI-era libraries or synchronous SDKs out of
the ASGI event loop, or isolate them with an async client, `to_thread`, or a
worker service.

An interview-ready distinction is:

> “WSGI models a synchronous HTTP request and response. ASGI models a
> connection plus asynchronous receive/send events, including WebSockets.
> For a browser voice demo I would expose the realtime endpoint through an
> ASGI framework/server, while ordinary REST or legacy WSGI applications could
> remain separate or be mounted behind the same deployment boundary.”

Do not describe ASGI as a replacement for WebRTC. ASGI is the Python
application interface; WebSocket, WebRTC, SIP, or a telephony provider are
transport/media choices that can be terminated or adapted by the server.

The implementation's producer-consumer tasks and merged async iterators are
important: a provider can continue sending events while the application
forwards upstream events or sends new text downstream. In a production
version, define the behavior when one side is faster than the other:

- bound queues and apply backpressure;
- cancel provider tasks when the browser disconnects or the user barges in;
- attach a `generation_id` so stale TTS chunks cannot play after cancellation;
- close STT/TTS sessions explicitly on every exit path;
- validate audio encoding, sample rate, message type, and sequence;
- distinguish provisional, final, cancelled, and failed events.

An interview-ready explanation is:

> “Each stage is an async stream transformation. STT turns audio frames into
> transcript events, the agent turns finalized transcript events into streamed
> text, and TTS turns text chunks into audio events. Because each stage awaits
> network I/O and yields incrementally, the system is pipelined rather than
> batch-oriented, reducing time to first audio. In production I would add
> bounded queues, cancellation propagation, stale-generation rejection,
> authentication, and durable state.”

### STT as a long-lived producer-consumer session

The tutorial's STT step is not a single
`transcribe(complete_audio)` request. It is a stateful, bidirectional provider
session:

```text
browser audio -> producer -> STT WebSocket
                             |
                 transcript events <- consumer
```

The producer sends audio chunks while the consumer receives recognition
events. This allows transcription to begin before the user's entire utterance
has arrived.

Keep provisional and committed results separate:

```text
stt_chunk  -> partial/provisional transcript, useful for captions or hints
stt_output -> finalized turn, normally safe to pass to the agent
```

The finalized event is a semantic boundary, not merely another text message.
It determines when the agent starts reasoning. Starting tools from provisional
text can cause actions based on an incomplete phrase, while waiting for every
final result increases latency. The correct policy depends on the action risk:
captions may show partial text, read-only lookups may be speculative, and
irreversible tools should require finalized, validated intent plus the normal
authorization and confirmation checks.

The STT adapter should own provider-specific details:

- WebSocket connection and termination lifecycle;
- audio encoding, sample rate, channels, and chunk validation;
- provider turn-formatting/endpointing configuration and its semantics;
- provider message parsing and sequence handling;
- mapping provider events to normalized application events;
- timeout, reconnect, and provider-error behavior;
- idempotent cleanup when both the audio task and outer pipeline terminate.

In the tutorial's concrete example, the provider connection is configured for
16 kHz PCM with automatic turn formatting, and a provider `Turn` is mapped to
either a provisional transcript event or a finalized `stt_output` event.
Those values are tutorial/provider choices, not universal voice-agent
requirements.

The rest of the runtime should consume normalized events rather than AssemblyAI
or another provider's raw message schema. This keeps the agent workflow
portable and makes STT providers replaceable.

Production questions to answer in an interview include:

- What happens if the browser disconnects while audio is still being sent?
- Can the provider session be closed more than once safely?
- Is the queue bounded if recognition is slower than audio ingestion?
- How are duplicate, late, cancelled, or out-of-order events handled?
- Which events are visible to the user, and which can trigger reasoning/tools?
- How are final-transcript latency, endpointing accuracy, and critical-entity
  accuracy measured?

## 4. VAD, endpointing, and semantic turns

These are related but different:

- **Voice activity detection (VAD)** estimates whether audio contains speech.
- **Endpointing** decides when a speech segment or turn is complete.
- **Semantic turn detection** uses linguistic/contextual evidence to decide
  whether the speaker has yielded the floor.

### Why simple silence timers fail

A fixed silence threshold can cut off a slow speaker, wait too long after a
short answer, or mis-handle a backchannel such as “mm-hm.” No detector has
perfect knowledge, especially with noise, overlap, packet loss, or a user who
is thinking.

Use a layered policy:

1. VAD gates obvious non-speech and noise.
2. Endpointing combines speech duration, trailing silence, punctuation-like
   cues, and transport timing.
3. Semantic turn logic considers whether the answer is complete for the
   current prompt.
4. A maximum turn duration and an explicit “continue” path prevent indefinite
   waiting.

Expose confidence and reason codes in internal events. Keep provisional,
confirmed, and cancelled hypotheses distinct.

### Useful turn states

```text
LISTENING
SPEECH_STARTED
SPEECH_CONTINUING
ENDPOINT_CANDIDATE
USER_TURN_CONFIRMED
THINKING
SPEAKING
INTERRUPTED
RECOVERING
ENDED
```

A state machine is easier to test than scattered booleans such as
`isSpeaking`, `isListening`, and `shouldStop`.

### Barge-in and cancellation

Barge-in is a coordinated operation, not merely a VAD callback:

1. Detect speech or an explicit interrupt gesture.
2. Stop or duck audio playback immediately.
3. Cancel model generation and TTS at the earliest supported boundary.
4. Abort in-flight low-risk tool proposals that have not committed.
5. Increment a response generation/turn token so late chunks are discarded.
6. Preserve the user's new audio and the interruption reason in the journal.
7. Re-enter listening without repeating stale output.

Cancellation must be idempotent. A result arriving after cancellation must be
recognized as stale and must not be played or applied as a business action.
If a tool has already committed, cancellation cannot undo it; the workflow
must report that fact and use a compensating action if one exists.

Illustrative pseudocode:

```python
async def start_response(turn):
    token = response_tokens.issue(turn.id)
    playback.start(turn.id)
    try:
        async for chunk in planner.stream(turn):
            if response_tokens.is_current(token) is False:
                return
            await playback.enqueue(chunk, token=token)
    except CancelledError:
        raise
    finally:
        playback.close(turn.id, token=token)

async def interrupt(turn_id, reason):
    response_tokens.invalidate(turn_id)
    await playback.stop(turn_id)
    await planner.cancel(turn_id)
    await tts.cancel(turn_id)
    await journal.append(
        event="response.interrupted",
        session_id=current_session.id,
        turn_id=turn_id,
        reason=reason,
    )
```

## 5. LangChain agent stage in the cascaded pipeline

In the tutorial, the agent stage consumes the STT event stream and only starts
the LangChain agent for finalized `stt_output` events. Provisional
`stt_chunk` events can be shown as captions or used for low-risk speculation,
but they should not directly trigger irreversible tools.

The agent then streams response content incrementally:

```text
final transcript -> LangChain agent -> agent_chunk 1 -> agent_chunk 2 -> ...
```

This lets the TTS stage start synthesizing before the complete answer exists.
The agent stage therefore transforms a stream of voice events into a stream of
agent events rather than returning one completed string.

### Key concepts

- **Final-turn gating:** finalized speech is the normal boundary for reasoning;
  the policy can be more aggressive for read-only work and more conservative
  for high-risk actions.
- **Streamed agent output:** the tutorial uses LangChain's
  `astream_events(version="v3")` and `stream.messages` to expose incremental
  text content for downstream TTS. Treat the exact API version as
  framework-specific and verify it against the current LangChain release.
- **Voice-specific prompting:** responses should be concise and speakable;
  avoid markdown, emojis, tables, and other text that sounds unnatural when
  synthesized.
- **Conversation memory:** a checkpointer keyed by a stable `thread_id`
  preserves context across turns. The tutorial's in-memory saver is useful for
  a demo but loses state on process restart; production needs durable,
  tenant-scoped storage and a compaction strategy.
- **Agent construction:** the tutorial uses a LangChain `create_agent` with
  order tools and a text model. The model is replaceable, but the agent's
  tool boundary and durable application state must remain provider-independent.
- **Tool boundary:** the LangChain agent can propose tool calls, but the
  application must authenticate the user, validate arguments, authorize the
  action, require confirmation where appropriate, enforce idempotency, and
  verify the result before speaking success.
- **Tracing:** LangSmith or an equivalent system can trace model/tool steps and
  latency, but transcripts, audio, and tool arguments must be redacted and
  access-controlled according to the data policy.

The tutorial's order tools return illustrative strings. They do not demonstrate
transactionality, authorization, duplicate prevention, or a real kitchen
system. Call that out explicitly in an interview.

An interview-ready explanation is:

> “The agent stage is a stateful, streamed text workflow between finalized
> speech recognition and speech synthesis. It owns reasoning and orchestration,
> but not raw media transport or unrestricted business authority. I would use
> the checkpointer for conversational context, keep authoritative workflow
> state outside the prompt, and make every tool call pass through a typed,
> authenticated, idempotent policy boundary.”

## 6. TTS stage and playback scheduling

In the tutorial, the TTS stage consumes streamed `agent_chunk` events and
produces audio events for the browser:

```text
agent text chunks -> TTS provider -> audio chunks -> playback queue
```

It uses producer-consumer concurrency in the opposite direction from STT:

- one path forwards upstream events and sends agent text to the TTS provider;
- another receives synthesized audio chunks;
- the tutorial's `merge_async_iters` helper combines both streams for
  downstream consumers.

TTS is therefore not a final serialization step. It is a realtime consumer of
agent text and a producer of audio, with its own lifecycle, buffering, format,
context, cancellation, and playback-clock responsibilities.

### Key design decisions

- **Incremental synthesis:** starting from text chunks reduces time to first
  playable audio, but sending every token independently can sound choppy.
  Buffer to phrase or clause boundaries when naturalness matters.
- **Provider continuation/context:** verify that incremental chunks share the
  intended provider context or continuation. Otherwise each chunk may be
  synthesized as an unrelated utterance with inconsistent prosody.
- **Explicit audio contract:** configure and validate encoding, sample rate,
  channels, container, and frame boundaries at the provider boundary. The
  output must match the browser, WebRTC, telephony, or media-gateway playback
  path.
- **Playback is a separate clock:** audio generated or received is not
  necessarily audio the user heard. Track generated, queued, played, and
  discarded audio separately for latency, safety, and transcript alignment.
- **Barge-in:** stop or clear queued audio, cancel the corresponding
  generation, invalidate stale chunks, and resume listening. Never allow late
  audio from an interrupted response to play.
- **Output policy:** if assistant speech must be checked before playback, buffer
  audio until its transcript and policy check pass. This improves safety but
  adds latency and needs an explicit timeout fallback.

The tutorial's provider client is intentionally minimal. A production TTS
adapter should also handle provider connection errors, timeouts, explicit
completion signals, idempotent close, bounded playback queues, cancellation,
and fallback to captions, text, pre-recorded audio, or a human handoff.

In the concrete example, the Cartesia adapter sends text over a WebSocket,
decodes returned audio payloads, and emits raw PCM audio events using the
configured voice, model, encoding, and sample rate. The tutorial's exact
payload shape and audio settings belong in the provider adapter; the rest of
the runtime should consume normalized audio events. Verify that incremental
text uses the intended provider continuation/context semantics rather than
creating unrelated utterances for every token or chunk.

An interview-ready explanation is:

> “TTS is a streaming stage, not a final batch conversion. The agent emits text
> while it is still generating, TTS emits audio while it is still synthesizing,
> and the playback scheduler decides what the user actually hears. I would
> optimize time to first useful audio, but keep phrase-level chunking,
> generation-aware cancellation, bounded queues, explicit audio formats, and
> playback-based observability.”

## 7. Media plane and control plane

The media plane is optimized for continuous, time-sensitive data. The control
plane is optimized for typed, authenticated, durable decisions. Mixing them
creates subtle security and reliability problems.

### Media plane

- Audio frames, timestamps, sequence numbers, codec metadata.
- Jitter buffers and packet-loss handling.
- Playback queues and interruption markers.
- No direct SQL, payment, identity, or unrestricted tool calls.

### Control plane

- Session identity and authorization context.
- Conversation and workflow state.
- Typed tool requests/results.
- Confirmation and policy decisions.
- Reconnect/resume tokens.
- Audit and redaction metadata.

A dropped audio packet may be tolerated or concealed; a dropped
`payment.authorized` event cannot be treated the same way. Assign each event a
delivery and durability expectation.

## 8. Typed events, ownership, and invariants

Use a versioned event envelope rather than passing arbitrary dictionaries
between components:

```json
{
  "type": "tool.requested",
  "version": 1,
  "event_id": "evt_01...",
  "session_id": "sess_01...",
  "turn_id": "turn_42",
  "occurred_at": "2026-01-01T12:00:00Z",
  "producer": "conversation-runtime",
  "trace_id": "trace_...",
  "payload": {
    "tool": "appointment.cancel",
    "arguments": {
      "appointment_id": "apt_123"
    },
    "risk": "high",
    "confirmation_required": true
  }
}
```

Every event should have one authoritative owner for its meaning:

| Event or state | Owner |
| --- | --- |
| Audio delivery and sequence | Transport adapter |
| Turn boundary | Conversation runtime |
| User identity | Auth/session service |
| Permission to call a tool | Policy/tool broker |
| Appointment/payment/account record | Authoritative application service |
| Whether a side effect committed | The service that committed it |
| Audit record | Durable event/audit service |

The model can propose intent; it does not own the truth of a record. The
frontend can display a result; it does not own authorization. A transcript can
describe an action; it does not prove that action occurred.

## 9. Tools: authority, authentication, confirmation, and idempotency

Tool use is where a conversational system becomes an agent. Treat every model
tool call as untrusted input.

### Tool broker checklist

For each request:

1. Authenticate the caller and bind the request to the current session.
2. Authorize the user, tenant, role, and resource scope.
3. Validate the tool name and arguments against a strict schema.
4. Normalize identifiers and reject ambiguous or extra fields.
5. Apply policy, rate, amount, destination, and time-window limits.
6. Determine whether the operation is read-only, reversible, or high risk.
7. Obtain explicit confirmation when required.
8. Attach an idempotency key and a bounded deadline.
9. Execute against the authoritative service.
10. Persist the outcome and return a typed result.

Never let a prompt, transcript, caller ID, or model-generated “confirmed”
field substitute for an authenticated policy decision.

### Confirmation

Confirmation should be specific and comprehensible:

> “I will cancel appointment A-123 for Tuesday at 10:00, and the cancellation
> policy may charge EUR 20. Should I continue?”

Record what was displayed or spoken, the confirmation response, the identity
context, and the policy version. For high-risk actions, require step-up
authentication or a human approval path where appropriate.

### Idempotency

Retries are normal during reconnects and provider timeouts. Every
side-effecting request should carry an idempotency key derived from a stable
workflow/action identity, not from a newly generated retry UUID:

```text
idempotency_key = hash(tenant, authenticated_subject,
                       workflow_instance, action_name, action_version)
```

The authoritative service should atomically record the key and result. If a
client receives an unknown outcome, query by the key rather than blindly
executing again.

## 10. Multi-tenant voice-agent architecture

Multi-tenancy is more than putting a `tenant_id` column on a conversation
table. Tenant context must be a first-class security and routing invariant
across identity, realtime sessions, agent configuration, tools, storage,
providers, quotas, billing, and observability.

### Establish and propagate tenant context

Bind the authenticated subject to a tenant before opening the realtime session:

```text
subject_id + tenant_id + roles/scopes
        -> session_id + short-lived session credential
        -> tenant-scoped runtime, tools, storage, and providers
```

Do not trust a tenant ID supplied in an arbitrary browser message. Resolve it
from a validated identity token or server-side session record, and bind:

```text
session_id -> tenant_id -> authenticated_subject -> authorized roles
```

Re-check that binding at every sensitive boundary. A WebSocket or WebRTC
connection, telephony callback, tool request, reconnect token, and provider
event must not be able to select another tenant by changing a field in its
payload.

### Snapshot tenant policy at session start

Resolve and version the tenant's configuration before the conversation starts:

- enabled tools and role permissions;
- system instructions and agent version;
- model, voice, language, and provider routing;
- region and data-residency policy;
- retention, recording, and redaction rules;
- quotas, spend limits, and concurrency limits;
- escalation and human-handoff policy.

Keep this configuration application-controlled and immutable for the session,
or explicitly journal any policy change. Do not silently allow a tenant admin
or model-generated text to change security-critical policy mid-call.

### Enforce isolation at every data boundary

Include tenant ownership in durable records and event envelopes:

```text
tenant_id, subject_id, session_id, conversation_id,
turn_id, tool_call_id, trace_id
```

Apply defense in depth:

- tenant-scoped database queries, composite keys, and row-level policies;
- tenant-prefixed object-storage paths for audio and transcripts;
- tenant-specific vector/RAG namespaces;
- tenant-aware cache keys and checkpointer/thread IDs;
- tenant-scoped encryption, retention, deletion, and export;
- support/admin tools that require an explicit tenant scope;
- no raw transcript, audio, prompt, or credential leakage across tenants.

Do not rely on every caller remembering to add a tenant filter. Put isolation
in repository helpers, database policies, authorization middleware, and
automated negative tests.

### Scope tools, providers, and resources

The tool broker must authorize the tuple
`(tenant, subject, role, resource, action)` before execution. A model request
is only a proposal. Validate resource ownership, confirmation, idempotency,
and the authoritative result before speaking success.

Keep provider credentials server-side. Choose deliberately between pooled
platform credentials, per-tenant credentials, or dedicated provider projects.
If tenants bring their own keys, encrypt, rotate, scope, and audit them.
Never place secrets in prompts, transcripts, client code, or unrestricted
telemetry.

For telephony, map verified provider resources such as a signed webhook,
account/subaccount, phone number, call ID, or stream ID to the tenant. Caller
ID alone is not tenant authentication.

### Protect against noisy neighbors

Track and limit per tenant:

- active sessions and WebSocket connections;
- audio minutes and provider concurrency;
- STT/LLM/TTS usage and tool calls;
- queue depth, CPU/memory, and bandwidth;
- spend, rate, and outbound-call budgets.

Use admission control, fair scheduling, bounded queues, circuit breakers, and
tenant-specific rate limits. A single tenant's spike must not exhaust shared
provider quotas or event-loop capacity for other tenants.

The main deployment trade-off is:

| Model | Benefit | Cost |
| --- | --- | --- |
| Shared pool with logical isolation | Efficient and easy to operate | Highest blast radius; requires strong isolation and noisy-neighbor controls |
| Dedicated workers/provider projects | Better isolation, quotas, and customization | More operational cost and slower provisioning |
| Hybrid tiers | Match isolation to compliance and tenant value | More routing and configuration complexity |

### Tenant-aware observability and tests

Attach tenant and session attributes to traces, metrics, billing, and audit
events, but restrict access and redact sensitive content. Test explicitly for:

- cross-tenant reads, writes, memory retrieval, vector search, and cache hits;
- session-token swapping and confused-deputy tool calls;
- stale/replayed reconnect tokens and provider callbacks;
- duplicate effects with tenant-scoped idempotency keys;
- tenant deletion, export, retention, and legal hold;
- noisy-tenant starvation and quota bypass;
- tenant data appearing in prompts, logs, traces, recordings, or fallbacks.

An interview-ready explanation is:

> “I make tenant context part of identity, not user input. I bind every live
> session to one tenant, snapshot its policy, enforce tenant scope in the tool
> broker and storage layer, isolate memory/caches/providers, apply per-tenant
> quotas and fair scheduling, and test cross-tenant leakage as a security
> failure. Shared infrastructure is acceptable only with defense-in-depth
> isolation and an explicit path to dedicated tiers.”

## 11. Deterministic workflows for high-risk actions

Use a state machine or workflow engine for actions such as payments,
transfers, refunds, account recovery, destructive changes, regulated
disclosures, or emergency escalation. Let the model help with language and
slot filling; do not let it invent the transaction protocol.

Example:

```text
START
  -> IDENTIFY_SUBJECT
  -> COLLECT_REQUIRED_FIELDS
  -> SHOW_OR_SPEAK_SUMMARY
  -> EXPLICIT_CONFIRMATION
  -> STEP_UP_AUTHENTICATION
  -> POLICY_CHECK
  -> EXECUTE_WITH_IDEMPOTENCY
  -> VERIFY_COMMIT
  -> DISCLOSE_RESULT
  -> COMPLETE

Any state -> ABORTED
Any network interruption after commit attempt -> RECONCILE
```

Each state should define allowed inputs, timeouts, retry behavior, audit
requirements, and whether a transition is reversible. A workflow must never
say “done” merely because the model generated a successful-sounding sentence.

## 12. Lifecycle, reconnect, and resume

Treat the session lifecycle as a distributed-systems problem.

### Lifecycle events

At minimum model:

```text
session.created
transport.connected
transport.disconnected
transport.reconnected
turn.started
turn.ended
response.started
response.interrupted
tool.requested
tool.committed
tool.failed
session.suspended
session.resumed
session.ended
```

Use a monotonic sequence or journal position for control events. Persist enough
state to determine whether a response, tool request, or workflow transition
was already applied.

### Reconnect strategy

1. Give the client a short-lived resume token bound to the authenticated
   session and device/channel.
2. On reconnect, authenticate again and validate the token.
3. Exchange last-seen media/control sequence numbers.
4. Replay durable control events, not blindly replayed audio.
5. Reconcile in-flight tool operations by idempotency key.
6. Mark any uncertain side effect as `UNKNOWN` until the authoritative service
   answers.
7. Tell the user what is known and what needs retry or human follow-up.

Avoid pretending that a new connection is a continuation if identity,
authorization, or workflow state cannot be verified.

## 13. Fallbacks and graceful degradation

Plan fallbacks before production:

- If speech recognition confidence is low, ask a targeted clarification or
  offer DTMF/text/human transfer.
- If the realtime model is unavailable, switch to a cascaded or
  text-first path when safe.
- If TTS is unavailable, use a pre-recorded disclosure or transfer to a human;
  do not silently perform a high-risk action.
- If tools time out, report that the result is pending and reconcile by
  idempotency key.
- If the network degrades, reduce bitrate or disable nonessential features
  while preserving identity and policy.
- If the agent loses context, summarize verified state from the journal instead
  of guessing.

A fallback must preserve the same authorization, confirmation, audit, and
business invariants as the primary path.

## 14. Safety, privacy, consent, and telephony fraud

### Safety and prompt injection

- Treat user speech, retrieved text, web pages, voicemail, and tool output as
  untrusted content.
- Keep instructions, data, and tool results in separate typed fields.
- Use allowlisted tools and argument schemas.
- Do not allow a retrieved document to change policy or authorization.
- Apply output and action policy before a side effect, not only after it.
- Make human escalation available when the user is distressed, vulnerable, or
  the system is uncertain.

### Privacy

- Minimize collection and retention of audio, transcripts, embeddings, and
  metadata.
- Classify fields and redact secrets and payment data before logs.
- Encrypt in transit and at rest; control operator access.
- Define deletion, export, legal hold, and tenant-isolation behavior.
- Do not use production conversations for training or evaluation without the
  applicable consent and governance.
- Keep provider data-retention and subprocessors in the data-flow inventory.

### Consent and disclosure

Before recording or transcription, provide a clear, jurisdiction-aware
disclosure and a non-recorded alternative where required. The exact rule
depends on the participants' locations, business purpose, and applicable law;
legal review is not replaced by an agent prompt.

### Telephony fraud

Outbound voice systems are attractive fraud targets. Use:

- Destination allowlists and country/premium-rate blocks.
- Per-user, tenant, number, campaign, and time-window spend limits.
- Velocity and anomaly detection for call volume and duration.
- Verified webhook signatures and replay protection.
- Separate authorization for call initiation and call content.
- Restrictions on transferring to external numbers.
- Alerts for repeated failed identity checks, high-cost destinations, and
  unusual after-hours traffic.

## 15. Evaluation, observability, and cost

### Evaluation dimensions

Measure more than model quality:

| Dimension | Example measures |
| --- | --- |
| Turn taking | Endpoint false-cutoff rate, late-endpoint rate, barge-in success |
| Recognition | Word error rate by noise/accent/language; entity accuracy |
| Conversation | Task completion, clarification rate, groundedness, escalation |
| Tool safety | Unauthorized-call rate, confirmation correctness, duplicate side effects |
| Reliability | Reconnect success, unknown-outcome reconciliation, timeout rate |
| User experience | Time to first audio, response gap, interruption recovery |
| Equity | Performance by language, accent, device, network, disability needs |
| Operations | Error budget, queue depth, provider error classes |
| Cost | Audio minutes, model tokens, TTS characters, TURN/carrier charges |

Use scenario tests with scripted audio, noisy audio, interruptions, overlap,
silence, accents, network loss, duplicate webhooks, delayed tool results,
prompt injection, and ambiguous confirmations. Include human review for
high-risk outcomes.

### Observability

Propagate a correlation ID through transport, runtime, model, tool broker, and
authoritative services. Record:

- Session, turn, response, workflow, tool, and idempotency identifiers.
- Timestamps for capture, endpoint, first model output, first audio, playback,
  interruption, tool request, commit, and final result.
- Provider/model/version and configuration identifiers.
- State transitions and cancellation reasons.
- Redaction classifications and retention policy.

Do not log raw audio or sensitive transcripts by default. Prefer hashes,
redacted excerpts, structured reason codes, and short-lived secure samples.

### Latency

Report a component breakdown:

```text
time_to_first_audio =
  capture + transport + endpoint + queue +
  model_first_chunk + synthesis_first_chunk + playback_buffer
```

Any latency number in an interview answer should be labeled **illustrative** or
**vendor-reported**, tied to stated conditions, and never presented as a
guarantee. Compare the same workload, audio format, network, concurrency,
buffering, and measurement boundary.

### Cost

Model cost per completed task, not only per minute:

```text
total_cost =
  model_input + model_output + ASR + TTS +
  transport/TURN + telephony + storage + observability +
  retries + human_escalation
```

Track cost by tenant, workflow, provider, and fallback path. Add budgets and
rate limits that fail safely before an outage becomes an unexpectedly large
bill.

## 16. Implementation sketch

The following is intentionally provider-neutral pseudocode. It illustrates
ownership and cancellation, not a drop-in SDK:

```python
async def run_session(transport, auth, broker, journal):
    session = await auth.establish_session(transport.credentials())
    state = SessionState(session_id=session.id)
    await journal.append("session.created", session_id=session.id)

    try:
        async for event in transport.events():
            if event.type == "audio.frame":
                if state.mode == "SPEAKING" and vad.detect_speech(event.frame):
                    await state.interrupt("barge_in")
                await state.turns.accept_audio(event.frame)

            elif event.type == "transport.disconnected":
                await state.suspend()
                await journal.append("session.suspended", session_id=session.id)

            elif event.type == "turn.confirmed":
                turn = await state.turns.confirm(event.turn_id)
                response = await state.responses.start(turn)

                async for item in response:
                    if item.type == "audio":
                        await transport.play(item, generation=response.generation)
                    elif item.type == "tool.request":
                        result = await broker.handle(
                            session=session,
                            request=item,
                            idempotency_key=item.idempotency_key,
                        )
                        await response.resume_with_tool_result(result)

            elif event.type == "resume.requested":
                await auth.reauthorize(event.resume_token, session)
                await state.reconcile()
                await transport.replay_control_events(
                    after=event.last_control_sequence
                )
    finally:
        await state.cancel_active_work()
        await journal.append("session.ended", session_id=session.id)
        await transport.close()
```

Production code should make event ordering, ownership, timeouts, cancellation,
and error classes explicit. Avoid broad exception handlers that turn an
unknown result into a success-shaped response.

## 17. Interview questions and strong answer patterns

### Architecture

**How would you design a voice agent end to end?**  
Start with transport adapters, a realtime turn runtime, a separate control
plane, and an authoritative tool broker. Explain event ownership and where
state is durable.

**Why separate media and control planes?**  
Audio needs low-latency streaming and can tolerate different loss behavior;
authorization and side effects need typed, authenticated, durable semantics.

### Turn taking

**How do you detect the end of a turn?**  
Use VAD, endpointing, and semantic context together. Describe confidence,
maximum durations, backchannels, and how you measure false cutoffs.

**How do you implement barge-in?**  
Stop playback, cancel generation/TTS, invalidate stale chunks, preserve the
new user audio, and reconcile any tool that may already have committed.

### Model and pipeline choice

**When would you choose speech-to-speech over a cascade?**  
Choose it for natural low-latency interaction when the provider exposes the
needed control and audit surfaces. Choose a cascade or hybrid when exact
transcripts, portability, deterministic workflows, or inspection dominate.

**How do you compare latency claims?**  
Define the measurement boundary and workload, decompose the path, and label
numbers illustrative or vendor-reported rather than universal.

### Tools and safety

**Can the model call a payment/refund/cancellation tool?**  
It can propose a typed request. A broker authenticates, authorizes, validates,
confirms, applies idempotency, executes, and returns the authoritative result.

**What happens after a timeout?**  
Do not blindly retry. Query the authoritative service by idempotency key and
represent the result as unknown until reconciled.

**How do you defend against prompt injection?**  
Treat all content as untrusted, keep policy outside the prompt, allowlist tools,
validate schemas, and require independent authorization for side effects.

### Reliability

**How do you handle reconnects?**  
Reauthenticate, exchange sequence positions, replay durable control events,
reconcile in-flight operations, and explain uncertainty to the user.

**What do you log?**  
Correlated structured events and timings with redaction, not unrestricted raw
audio. Keep enough evidence to evaluate and audit without collecting data by
default.

### Product and operations

**What would you measure after launch?**  
Task completion, endpointing and barge-in quality, first-audio latency,
reconnects, tool safety, equity, escalation, provider failures, and cost per
completed task.

**What is your fallback if the realtime provider fails?**  
Switch only to a path with equivalent identity, policy, confirmation, audit,
and business invariants; otherwise transfer or defer safely.

## 18. Source links and scope notes

The links below are starting points for primary documentation and standards.
Provider capabilities and terms change, so verify the current version and
region before making an implementation or compliance commitment.

### Standards and platform references

- [WebRTC API (W3C)](https://www.w3.org/TR/webrtc/)
- [WebRTC 1.0 API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
- [Interactive Connectivity Establishment, RFC 8445](https://www.rfc-editor.org/rfc/rfc8445)
- [Session Traversal Utilities for NAT, RFC 8489](https://www.rfc-editor.org/rfc/rfc8489)
- [Traversal Using Relays around NAT, RFC 5766](https://www.rfc-editor.org/rfc/rfc5766)
- [RTP, RFC 3550](https://www.rfc-editor.org/rfc/rfc3550)
- [SIP, RFC 3261](https://www.rfc-editor.org/rfc/rfc3261)
- [OAuth 2.0, RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [NIST Digital Identity Guidelines](https://pages.nist.gov/800-63-4/)
- [AWS SaaS Lens: Tenant Isolation](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/tenant-isolation.html)
- [AWS SaaS Lens: Identity and Access Management](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/identity-and-access-management.html)
- [OWASP Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/)

### Realtime speech and model documentation

These are provider-specific examples, not endorsements or universal
guarantees. They are useful for comparing concepts such as streaming,
interruption, turn detection, and tool calling:

- [OpenAI Realtime API guide](https://platform.openai.com/docs/guides/realtime)
- [LangChain voice-agent tutorial](https://docs.langchain.com/oss/python/langchain/voice-agent)
- [LangChain voice sandwich demo](https://github.com/langchain-ai/voice-sandwich-demo)
- [LangChain streaming](https://docs.langchain.com/oss/python/langchain/streaming)
- [LangChain short-term memory/checkpointers](https://docs.langchain.com/oss/python/langchain/short-term-memory)
- [`asyncio` — Python asynchronous I/O](https://docs.python.org/3/library/asyncio.html)
- [PEP 3333 — WSGI](https://peps.python.org/pep-3333/)
- [ASGI 3.0 specification](https://asgi.readthedocs.io/en/latest/specs/main.html)
- [OpenAI function calling guide](https://platform.openai.com/docs/guides/function-calling)
- [Google Cloud Speech-to-Text streaming recognition](https://cloud.google.com/speech-to-text/docs/streaming-recognize)
- [Google Cloud Text-to-Speech streaming synthesis](https://cloud.google.com/text-to-speech/docs/streaming-synthesize)
- [Google Cloud Vertex AI function calling](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/function-calling)
- [Amazon Transcribe streaming](https://docs.aws.amazon.com/transcribe/latest/dg/streaming.html)
- [Amazon Polly developer guide](https://docs.aws.amazon.com/polly/latest/dg/what-is.html)
- [Azure Speech SDK real-time speech](https://learn.microsoft.com/azure/ai-services/speech-service/speech-sdk)
- [Azure OpenAI realtime audio](https://learn.microsoft.com/azure/ai-services/openai/realtime-audio-quickstart)

### Telephony, privacy, and security references

- [Twilio Voice security and webhook validation](https://www.twilio.com/docs/usage/security)
- [Twilio fraud prevention](https://www.twilio.com/docs/voice/troubleshooting/fraud)
- [FCC consumer guide to caller ID spoofing](https://www.fcc.gov/spoofing)
- [GDPR principles (European Commission)](https://commission.europa.eu/law/law-topic/data-protection/data-protection-law-applies_en)
- [EDPB guidelines on processing personal data](https://www.edpb.europa.eu/our-work-tools/general-guidance/guidelines-recommendations-best-practices_en)
- [PCI DSS documentation](https://www.pcisecuritystandards.org/standards/pci-dss/)

Source links document concepts and provider APIs; they do not transfer legal,
security, or operational responsibility to the provider. Keep vendor-specific
claims scoped to the cited product and version, and treat all latency examples
as **illustrative/vendor-reported** unless independently measured under stated
conditions.
