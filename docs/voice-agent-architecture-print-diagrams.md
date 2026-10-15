# Voice-agent architecture: print-friendly diagrams

Focused extracts of the canonical architecture in
[voice-agent-architecture-diagrams.md](./voice-agent-architecture-diagrams.md).
The component boxes and labels below are unchanged from the canonical diagram;
only the relevant parts and arrows are extracted for printing.

## 1. Media plane

```mermaid
flowchart LR
    subgraph CLIENTS[Channels]
        B[Browser / mobile]
        PH[Phone / PSTN]
    end

    subgraph EDGE[Public edge and signaling]
        G[Reverse proxy / load balancer<br/>HTTPS · WSS signaling · rate limits]
        ID[Identity + session issuance<br/>tenant binding · short-lived credentials]
        ICE[ICE service<br/>STUN · regional TURN]
        AD[WebRTC/media gateway<br/>tracks · DTLS-SRTP · Opus]
        TEL[Optional telephony/SIP adapter]
    end

    subgraph RUNTIME[One SessionRuntime per tenant/session]
        IQ[(Bounded input queue)]
        RT[Realtime turn runtime<br/>normalized events · VAD · semantic turns<br/>cancellation · generation IDs]
        OQ[(Bounded output queue)]
    end

    B --> G
    G -->|auth/session request| ID
    ID -->|short-lived provider token| B
    G -->|SDP/ICE signaling| AD
    B -->|WebRTC media tracks| AD
    B -.->|ICE candidates| ICE
    AD -.->|STUN/TURN| ICE
    ID -->|server-side session binding| AD
    PH --> TEL
    ID -->|tenant binding| TEL
    AD --> IQ
    TEL --> IQ
    IQ --> RT
    RT --> OQ
    OQ --> AD
    AD -->|outbound WebRTC track| B
    OQ --> TEL
    TEL -->|telephony media| PH
```

## 2. Session runtime, voice engine, and external providers

```mermaid
flowchart TB
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

    subgraph PROVIDERS[External inference providers]
        STTP[STT inference provider]
        LLMP[LLM / reasoning provider]
        TTSP[TTS inference provider]
        S2SP[Realtime S2S provider]
    end

    IQ --> RT
    RT -->|audio frames| STT
    STT -->|audio stream| STTP
    STTP -->|partial/final events| STT
    STT -->|partial/final events| RT
    RT -->|confirmed turn| AG
    AG -->|prompt + scoped context| LLMP
    LLMP -->|tokens + structured proposals| AG
    AG -->|streamed text| TTS
    TTS -->|speakable text| TTSP
    TTSP -->|audio chunks| TTS
    TTS -->|audio chunks| OQ
    RT -.->|alternative media engine| S2S
    S2S <--> S2SP
    S2SP -.->|normalized audio/events| RT
```

## 3. Control plane

```mermaid
flowchart TB
    subgraph EDGE[Public edge and signaling]
        ID[Identity + session issuance<br/>tenant binding · short-lived credentials]
    end

    subgraph RUNTIME[One SessionRuntime per tenant/session]
        RT[Realtime turn runtime<br/>normalized events · VAD · semantic turns<br/>cancellation · generation IDs]
    end

    subgraph ENGINE[Voice engine]
        AG[Agent orchestrator<br/>prompt · workflow · streaming]
    end

    subgraph CONTROL[Control plane]
        TB[Tool broker<br/>schema · auth · confirmation · idempotency]
        WS[Durable workflow state]
        MEM[Short-term checkpoint<br/>long-term memory]
        EJ[(Event journal<br/>traces · metrics · redacted audit)]
    end

    APP[(Authoritative business systems<br/>orders · bookings · identity · payments)]
    JOBS[Async workers<br/>long-running tasks · CPU-heavy work]

    ID -->|tenant policy snapshot| RT
    RT -->|confirmed turn| AG
    AG -->|typed tool request| TB
    TB --> APP
    APP --> TB
    TB -->|verified result| AG
    AG --> WS
    RT --> WS
    AG <--> MEM
    TB -->|long-running / CPU work| JOBS
    JOBS -->|checkpoint / result| WS
    TB -->|idempotency + outcome| WS
    RT -.-> EJ
    AG -.-> EJ
    TB -.-> EJ
    ID -.-> EJ
```

These are logical extracts. Depending on deployment, the canonical components
may be colocated in one worker, split into services, or supplied by a managed
platform; the component contracts and arrows remain the same.
