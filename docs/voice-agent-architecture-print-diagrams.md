# Voice-agent architecture: print-friendly diagrams

Focused diagrams for the [voice-agent interview preparation guide](./voice-agent-interview.md).
Each diagram is intentionally standalone and can be printed independently.

The architecture uses WebRTC as the primary browser media transport. HTTPS/WSS
is used for authentication and signaling; STT/TTS provider streams are internal
application connections behind provider adapters.

## 1. Media plane

This diagram shows only the time-sensitive media path and its transport
boundaries. The media gateway owns WebRTC details; the session runtime receives
normalized frames and returns normalized output audio.

```mermaid
flowchart LR
    B[Browser / mobile]
    SIG[HTTPS/WSS signaling<br/>auth · SDP · ICE]
    ICE[STUN / regional TURN]
    GW[WebRTC media gateway<br/>DTLS-SRTP · RTP · Opus]
    TEL[Optional telephony/SIP adapter]
    IN[Normalized inbound<br/>PCM AudioFrame]
    RT[SessionRuntime<br/>per tenant/session]
    OUT[Normalized outbound<br/>AudioChunk]

    B -->|session/auth request| SIG
    SIG -->|short-lived provider token| B
    B -->|WebRTC media tracks| GW
    B -.->|ICE candidates| ICE
    GW -.->|connectivity checks / relay| ICE

    GW -->|decode, jitter, normalize| IN
    IN --> RT
    RT --> OUT
    OUT -->|pace, encode, publish track| GW
    GW -->|outbound WebRTC audio| B

    TEL -->|provider media| IN
    OUT -->|provider media| TEL
```

Key invariant:

```text
WebRTC packets are decoded and normalized at the media boundary.
The runtime never needs to know RTP, DTLS-SRTP, ICE, or Opus packet details.
```

## 2. Session runtime, voice engine, and external providers

This diagram shows the per-session processing path after the media gateway has
produced normalized audio frames. The adapter boxes are application-owned;
provider boxes perform external inference.

```mermaid
flowchart TB
    IQ[(Bounded input queue)]
    RT[Realtime turn runtime<br/>VAD · endpointing · semantic turns<br/>barge-in · cancellation]
    STTA[STT adapter]
    STTP[External STT provider]
    CTX[Context builder / policy<br/>tenant scope · redaction · token budget]
    AG[Agent orchestrator<br/>workflow · streaming · tool proposals]
    LLMP[External LLM / reasoning provider]
    TTSA[TTS adapter]
    TTSP[External TTS provider]
    OQ[(Bounded output queue)]
    S2S[Optional realtime S2S adapter]
    S2SP[External realtime S2S provider]

    IQ --> RT
    RT -->|audio frames| STTA
    STTA -->|audio stream| STTP
    STTP -->|partial/final transcript events| STTA
    STTA -->|normalized transcript events| RT
    RT -->|confirmed UserTurn| CTX
    CTX -->|bounded authorized context| AG
    AG -->|prompt + context| LLMP
    LLMP -->|tokens + structured tool proposals| AG
    AG -->|speakable text chunks| TTSA
    TTSA -->|text stream| TTSP
    TTSP -->|audio chunks| TTSA
    TTSA -->|normalized audio| OQ

    RT -.->|alternative media engine| S2S
    S2S <--> S2SP
    S2SP -.->|normalized audio/events| RT
```

The runtime is the session-scoped owner of mutable realtime state. The context
builder decides what the model is allowed to see; the agent orchestrator
coordinates model calls and tool proposals; the TTS adapter converts approved
speakable text into outbound audio.

## 3. Control plane

This diagram shows identity, tenant policy, tools, durable workflow state,
memory, asynchronous work, and observability. These components are authoritative
or durable; they should not be placed on the raw audio hot path unnecessarily.

```mermaid
flowchart TB
    USER[Authenticated user / provider callback]
    ID[Identity + session issuance<br/>tenant binding · roles<br/>short-lived credentials]
    POLICY[Tenant policy snapshot<br/>tools · model · voice · region<br/>retention · quotas]
    RUNTIME[SessionRuntime<br/>session_id · tenant_id · subject_id]
    CTX[Context builder / policy filter<br/>scope · redact · compact · budget]
    MEM[(Short-term checkpoints<br/>long-term memory)]
    WF[(Durable workflow state<br/>intent · slots · confirmations<br/>tool status · idempotency)]
    TB[Tool broker<br/>schema · authorization<br/>confirmation · reconciliation]
    APP[(Authoritative business systems<br/>orders · bookings · payments<br/>identity · CRM)]
    JOBS[Async workers / job queue<br/>long-running and CPU-heavy work]
    OBS[(Event journal / observability<br/>traces · metrics · redacted audit)]
    BILL[Usage / billing / quotas]

    USER --> ID
    ID -->|authenticated tenant/session context| RUNTIME
    ID -->|versioned configuration| POLICY
    POLICY --> RUNTIME
    RUNTIME --> CTX
    MEM <--> CTX
    WF <--> CTX
    CTX -->|bounded model context| RUNTIME
    RUNTIME -->|typed tool request| TB
    TB -->|authorized operation| APP
    APP -->|authoritative result| TB
    TB -->|tool result / status| RUNTIME
    TB -->|long-running work| JOBS
    JOBS -->|checkpoint / result| WF
    RUNTIME <--> WF
    RUNTIME -.-> OBS
    TB -.-> OBS
    ID -.-> OBS
    RUNTIME --> BILL
    TB --> BILL
    BILL --> POLICY
```

The control-plane rules are:

- tenant context comes from validated identity, not arbitrary client fields;
- model output is a proposal, not authorization;
- workflow/tool outcomes are persisted with idempotency and uncertainty;
- long-term memory is promoted and retrieved under tenant/subject policy;
- raw audio and sensitive transcripts are redacted and retained deliberately;
- the event journal supports tracing and audit without becoming the business
  source of truth.

## How the three diagrams compose

```text
Media plane
  -> normalized audio/events
  -> SessionRuntime and voice engine
  -> control-plane policy/tools/state
  -> normalized output audio/events
  -> media plane
```

The diagrams are logical boundaries. Depending on scale and provider choice,
the media gateway, session runtime, and adapters may be colocated in one worker,
split into services, or supplied by a managed platform.
