# Voice-agent architecture diagrams

Companion diagrams for the [voice-agent interview preparation guide](./voice-agent-interview.md).
The diagrams are intentionally provider-agnostic; provider names illustrate
integration points, not required choices.

### How to read these diagrams

- Arrows show logical data/control flow; boxes do not necessarily mean separate
  deployable services.
- A dashed arrow is an alternative path or observation/control relationship,
  not an additional mandatory hop.
- The canonical design is a cascaded baseline. A realtime speech-to-speech
  provider can replace the STT/agent/TTS media path, but not the application's
  identity, tenant isolation, tool authority, durable state, or audit boundary.
- Provider adapters translate provider-specific wire messages into normalized
  session events. The `SessionRuntime` owns mutable state for one authenticated
  tenant/session.

## 1. Canonical production/interview architecture

```mermaid
flowchart LR
    subgraph CLIENTS[Channels]
        B[Browser / mobile]
        PH[Phone / PSTN]
    end

    subgraph EDGE[Public edge]
        G[Reverse proxy / load balancer<br/>TLS · WSS · rate limits]
        AD[Transport adapters<br/>WebSocket · WebRTC · SIP/media]
        ID[Identity + session issuance<br/>tenant binding · short-lived credentials]
    end

    subgraph RUNTIME[One SessionRuntime per tenant/session]
        IQ[(Bounded input queue)]
        RT[Realtime turn runtime<br/>normalized events · VAD · semantic turns<br/>cancellation · generation IDs]
        OQ[(Bounded output queue)]
    end

    subgraph ENGINE[Voice engine]
        STT[Streaming STT adapter]
        AG[Agent orchestrator<br/>prompt · workflow · streaming]
        TTS[Streaming TTS adapter]
        S2S[Optional realtime S2S adapter]
    end

    subgraph CONTROL[Control plane]
        TB[Tool broker<br/>schema · auth · confirmation · idempotency]
        WS[Durable workflow state]
        MEM[Short-term checkpoint<br/>long-term memory]
        EJ[(Event journal<br/>traces · metrics · redacted audit)]
    end

    APP[(Authoritative business systems<br/>orders · bookings · identity · payments)]
    JOBS[Async workers<br/>long-running tasks · CPU-heavy work]

    B --> G
    PH --> G
    G --> AD
    ID --> AD
    AD --> IQ
    IQ --> RT
    RT -->|audio frames| STT
    STT -->|partial/final events| RT
    RT -->|confirmed turn| AG
    AG -->|streamed text| TTS
    TTS -->|audio chunks| OQ
    OQ --> AD
    AD --> G
    G --> B
    G --> PH

    RT -.->|alternative media engine| S2S
    S2S -.->|normalized audio/events| RT

    AG -->|typed tool request| TB
    TB --> APP
    APP --> TB
    TB -->|verified result| AG
    AG --> WS
    RT --> WS
    AG <--> MEM
    TB --> JOBS
    RT -.-> EJ
    AG -.-> EJ
    TB -.-> EJ
    AD -.-> EJ
```

Read the architecture as two cooperating planes:

- **Media plane:** channels, adapters, audio queues, turn detection, model
  streaming, playback, and cancellation. It is time-sensitive and must not
  wait synchronously on slow business operations.
- **Control plane:** identity, tenant policy, tools, confirmations, durable
  workflow state, memory, billing, and audit. It is authoritative and must
  not trust raw audio, transcripts, caller IDs, or model output.

The diagram shows the **cascaded baseline**:

```text
audio -> STT -> agent/LLM -> TTS -> audio
```

For a speech-to-speech design, `S2S` replaces the STT/agent/TTS media path, but
the session runtime, tool broker, durable state, tenant isolation, and
observability boundaries remain.

The most important invariant is:

> One authenticated `SessionRuntime` owns mutable realtime state for exactly
> one tenant/session. Shared workers may multiplex I/O, but must not share
> decoder, VAD, playback, conversation, or tool state across sessions.

## 2. One confirmed turn and tool call

```mermaid
sequenceDiagram
    participant C as Browser / phone
    participant E as Edge + transport adapter
    participant R as SessionRuntime
    participant S as STT provider
    participant A as Agent / LLM
    participant T as TTS provider
    participant P as Tool broker
    participant D as Business system

    C->>E: Audio frame
    E->>R: Normalized AudioFrame
    R->>S: Stream audio
    S-->>R: Partial transcript
    R-->>C: Optional caption event
    S-->>R: Final transcript
    R->>R: Semantic turn confirmation
    R->>A: Finalized user turn + scoped context
    A-->>R: Optional spoken preamble
    R->>T: Stream speakable text
    T-->>R: Audio chunks
    R-->>E: Queue audio for playback
    E-->>C: Audio output
    A->>P: Structured tool request
    P->>P: Validate tenant, identity, policy, confirmation, idempotency
    P->>D: Authorized operation
    D-->>P: Authoritative result
    P-->>A: Typed tool result
    A->>T: Verified spoken result
```

The tool call may occur while audio continues, but the broker—not the model or
browser—owns authorization and side effects. If the user barges in, the
runtime cancels playback/generation where possible and reconciles any tool
operation that has already committed.

## 3. Cascaded streaming pipeline

```mermaid
flowchart LR
    M[Microphone PCM frames]
    Q1[(Bounded input queue)]
    STT[STT WebSocket]
    TE[Transcript events]
    TURN[Endpointing + semantic turn detection]
    LLM[Text agent / LLM]
    TOK[Streamed text chunks]
    TTS[TTS WebSocket]
    AQ[(Bounded playback queue)]
    SP[Speaker]

    M --> Q1 --> STT
    STT --> TE --> TURN
    TURN -->|turn confirmed| LLM
    LLM --> TOK --> TTS
    TTS --> AQ --> SP
```

The pipeline is incremental:

```text
audio arrives -> partial transcript -> finalized turn
              -> first agent text -> first TTS audio -> playback
```

Partial transcripts can support captions or low-risk speculation. Normal
reasoning and side effects should usually wait for a confirmed turn and pass
through the tool broker.

## 4. Speech-to-speech versus cascaded versus hybrid

```mermaid
flowchart TB
    subgraph S2S[Speech-to-speech]
        SA[User audio] --> SM[Realtime multimodal model] --> SO[Assistant audio]
    end

    subgraph CAS[Cascaded]
        CA[User audio] --> CSTT[STT] --> CLLM[Text agent] --> CTTS[TTS] --> CO[Assistant audio]
    end

    subgraph HYB[Hybrid]
        HA[Realtime voice surface] --> HDEC{Risk / workflow decision}
        HDEC -->|simple conversation| HOUT[Spoken response]
        HDEC -->|structured or high-risk task| HBACK[Auditable backend agent]
        HBACK --> HTOOLS[Authorized tools]
        HTOOLS --> HOUT
    end
```

| Architecture | Optimize for | Main cost |
| --- | --- | --- |
| Speech-to-speech | Naturalness and low latency | Less intermediate control and provider portability |
| Cascaded | Inspectability, replacement, and policy control | More stages, buffering, and failure boundaries |
| Hybrid | Fast conversation plus controlled high-risk workflows | More orchestration and synchronization |

## 5. Browser transports and provider adapters

```mermaid
flowchart LR
    B[Browser]
    SIG[HTTPS/WSS signaling<br/>auth · SDP · ICE]
    WSS[Browser WSS adapter<br/>binary audio or JSON/Base64]
    RTC[WebRTC adapter<br/>media tracks · Opus · ICE/TURN]
    TEL[Telephony adapter<br/>Twilio / Vonage / SIP]
    N[Normalized transport contract<br/>AudioFrame + TransportEvent]
    R[Session runtime]

    B --> SIG
    B --> WSS
    B --> RTC
    WSS --> N
    RTC --> N
    TEL --> N
    N --> R
    R --> N
    N --> WSS
    N --> RTC
    N --> TEL
```

The runtime should not depend on provider packet envelopes. Each adapter
authenticates the connection, validates sequence/timestamps and audio format,
then maps provider messages into internal events.

## 6. Multi-tenant session isolation

```mermaid
flowchart TB
    I[Validated identity<br/>subject · tenant · roles]
    C[Session creation<br/>short-lived credential]
    A[ASGI / media edge]
    W1[Shared worker]
    W2[Shared worker]

    subgraph SR1[Tenant A / Session 1]
        D1[Decoder + resampler + jitter state]
        Q1[Input/output queues]
        P1[VAD · STT · LLM · TTS · playback]
        K1[Tenant-scoped tools/memory]
    end

    subgraph SR2[Tenant B / Session 2]
        D2[Decoder + resampler + jitter state]
        Q2[Input/output queues]
        P2[VAD · STT · LLM · TTS · playback]
        K2[Tenant-scoped tools/memory]
    end

    I --> C --> A
    A --> W1 --> SR1
    A --> W2 --> SR2
    D1 --> Q1 --> P1 --> K1
    D2 --> Q2 --> P2 --> K2
```

Shared workers may multiplex I/O, but mutable decoder, VAD, playback,
conversation, and tool state belongs to one `SessionRuntime`. If a provider
multiplexes streams, demultiplex by a trusted provider stream ID, never by
arrival order.

## 7. User turn and barge-in state machine

```mermaid
stateDiagram-v2
    [*] --> Listening
    Listening --> SpeechStarted: VAD detects speech
    SpeechStarted --> UserSpeaking: audio continues
    UserSpeaking --> EndpointCandidate: silence / endpoint signal
    EndpointCandidate --> UserSpeaking: user continues
    EndpointCandidate --> TurnConfirmed: semantic detector accepts
    TurnConfirmed --> Thinking: start agent
    Thinking --> AgentSpeaking: first safe audio
    AgentSpeaking --> Listening: response complete
    AgentSpeaking --> Interrupted: user speech / explicit stop
    Interrupted --> Listening: stop playback + cancel generation
    Thinking --> Listening: provider failure / timeout
    Listening --> Reconnecting: transport disconnect
    Reconnecting --> Listening: authenticated resume
    Reconnecting --> Ended: resume failed
    Ended --> [*]
```

On interruption, invalidate the response generation, clear queued audio, and
discard late model/TTS events. A committed tool effect is reconciled rather
than pretending it was cancelled.

## 8. Structured tool-call flow

```mermaid
sequenceDiagram
    participant U as User
    participant V as Voice model
    participant B as Tool broker
    participant S as Business system
    participant P as Playback

    U->>V: Spoken request
    V->>P: Optional spoken preamble
    V->>B: Structured tool.requested
    B->>B: Validate schema, tenant, role, resource, confirmation
    alt confirmation required
        B-->>V: Need confirmation
        V->>P: Ask user to confirm exact action
        U->>V: Confirmation
        V->>B: Confirmed request
    end
    B->>S: Idempotent authorized operation
    S-->>B: Authoritative result
    B-->>V: Structured tool.result
    V->>P: Verified spoken response
```

The model proposes. The broker authorizes and executes. The application never
treats arbitrary spoken text as an executable tool call.

## 9. Short-term memory, compaction, and long-term memory

```mermaid
flowchart LR
    T[Current conversation turns]
    W[Typed workflow state<br/>slots · pending confirmation · tool status]
    C[Compaction<br/>rolling summary + recent turns]
    J[(Short-term checkpoint<br/>tenant + thread)]
    X[Memory promotion<br/>validate · redact · deduplicate]
    L[(Long-term memory<br/>facts · preferences · episodes)]
    R[Future-session retrieval<br/>relevance + authorization]
    A[Authoritative systems<br/>orders · identity · payments]

    T --> C --> J
    W --> J
    T --> X --> L
    L --> R
    R --> J
    A --> W
```

Compaction reduces context size; it does not replace authoritative state.
Long-term memory is promoted explicitly and must retain provenance, scope,
retention, and deletion behavior.

## 10. Python async execution model

```mermaid
flowchart TB
    EL[Event-loop thread<br/>asyncio]
    T1[Session task A]
    T2[Session task B]
    T3[Session task C]
    IO[OS socket readiness<br/>HTTP · WebSocket]
    WT[Worker thread<br/>blocking SDK / I/O]
    PR[Process / worker service<br/>CPU-heavy work]

    EL --> T1
    EL --> T2
    EL --> T3
    T1 -->|await async I/O| IO
    IO -->|ready| EL
    T2 -->|await asyncio.to_thread| WT
    WT -->|result| EL
    T3 -->|submit CPU work| PR
    PR -->|result| EL
```

`await` suspends the current coroutine at an async operation. A blocking
synchronous call on the event-loop thread stalls every task on that loop.
Threads isolate blocking I/O; processes or services provide CPU parallelism.
