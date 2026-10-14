# Voice-agent architecture diagrams

Companion diagrams for the [voice-agent interview preparation guide](./voice-agent-interview.md).
The diagrams are intentionally provider-agnostic; provider names illustrate
integration points, not required choices.

## 1. End-to-end reference architecture

```mermaid
flowchart LR
    U[Browser or phone user]
    E[Transport edge<br/>WSS / WebRTC / SIP]
    R[Realtime session runtime<br/>VAD · turns · cancellation]
    S[Streaming STT<br/>partial + final events]
    A[Agent orchestrator<br/>LLM · memory · workflow]
    T[TTS provider<br/>streaming audio]
    P[Playback scheduler<br/>queue · barge-in]
    B[Tool broker<br/>auth · policy · idempotency]
    D[(Authoritative business systems)]
    J[(Event journal<br/>traces · metrics · redacted audit)]

    U -->|audio in| E
    E --> R
    R -->|audio frames| S
    S -->|transcript events| R
    R -->|finalized turn| A
    A -->|typed tool request| B
    B --> D
    D --> B
    B -->|verified result| A
    A -->|streamed text| T
    T -->|audio chunks| P
    P -->|audio out| E
    E --> U

    R -.-> J
    A -.-> J
    B -.-> J
    T -.-> J
```

The media path is optimized for low latency. The control path owns
authorization, durable state, tools, and audit evidence. A provider may
collapse several boxes into one managed realtime service, but the ownership
boundaries should remain explicit.

## 2. Cascaded streaming pipeline

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

## 3. Speech-to-speech versus cascaded versus hybrid

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

## 4. Browser transports and provider adapters

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

## 5. Multi-tenant session isolation

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

## 6. User turn and barge-in state machine

```mermaid
stateDiagram-v2
    [*] --> Listening
    Listening --> SpeechStarted: VAD detects speech
    SpeechStarted --> Speaking: audio continues
    Speaking --> EndpointCandidate: silence / endpoint signal
    EndpointCandidate --> Speaking: user continues
    EndpointCandidate --> TurnConfirmed: semantic detector accepts
    TurnConfirmed --> Thinking: start agent
    Thinking --> SpeakingAgent: first safe audio
    SpeakingAgent --> Listening: response complete
    SpeakingAgent --> Interrupted: user speech / explicit stop
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

## 7. Structured tool-call flow

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

## 8. Short-term memory, compaction, and long-term memory

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

## 9. Python async execution model

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
