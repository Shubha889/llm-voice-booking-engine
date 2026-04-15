# LLM Voice Booking Engine

A production-grade, multi-turn voice booking engine powered by **AWS Bedrock (Claude)** and **LangGraph** state machines. Handles the full conversation lifecycle for home service bookings — from intent detection through vendor selection, slot confirmation, and payment — over a real-time WebSocket voice channel.

---

## What This Does

Users call in or use a voice interface to book home services. This engine handles the AI side of the conversation: it understands what the user wants across multiple turns, asks clarifying questions, resolves ambiguity, and drives the booking to completion — all in real time.

**Key behaviors:**
- Handles interruptions, corrections, and partial utterances gracefully
- Maintains session state across turns using LangGraph
- Processes audio through a 9-stage pipeline before generating responses
- Detects crisis signals and routes to a safety gate before proceeding

---

## Architecture

```
User Voice Input
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     9-Stage Processing Pipeline                  │
│                                                                  │
│  [1] Guardrails ──▶ [2] Role Resolver ──▶ [3] Context Aware    │
│       │                    │                      │             │
│       ▼                    ▼                      ▼             │
│  [4] Emotion/Tone ──▶ [5] Intent ──▶ [6] Compliance           │
│       │                    │                      │             │
│       ▼                    ▼                      ▼             │
│  [7] Decide Action ──▶ [8] Supervisor Router ──▶ [9] Guardrails│
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LangGraph State Machine                       │
│                                                                  │
│  IDLE ──▶ COLLECT_SERVICE ──▶ COLLECT_VENDOR                   │
│    │              │                    │                        │
│    │              ▼                    ▼                        │
│    │       COLLECT_DATE ──▶ COLLECT_SLOT                       │
│    │              │                    │                        │
│    │              ▼                    ▼                        │
│    └──── CONFIRM_BOOKING ◀── VERIFY_DETAILS                    │
│                   │                                             │
│                   ▼                                             │
│             PROCESS_PAYMENT ──▶ BOOKING_COMPLETE               │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
AWS Bedrock (Claude) ──▶ TTS Response ──▶ User
```

---

## The 9-Stage Pipeline

Each user utterance passes through all 9 stages sequentially before the engine decides how to respond:

| Stage | Name | What it does |
|-------|------|-------------|
| 1 | **Input Guardrails** | Blocks harmful, off-topic, or adversarial inputs before any LLM call |
| 2 | **Role Resolver** | Identifies if the speaker is a user, vendor, or admin |
| 3 | **Context Awareness** | Injects session history, user preferences, and prior booking context |
| 4 | **Emotion & Tone** | Detects user emotional state; adapts response tone accordingly |
| 5 | **Intent Classifier** | Classifies the utterance into a structured booking intent |
| 6 | **Compliance** | Checks for regulatory and business-rule violations |
| 7 | **Decide Action** | Chooses the next state machine transition based on intent + context |
| 8 | **Supervisor Router** | Routes to the correct LangGraph node or human escalation |
| 9 | **Output Guardrails** | Final safety check on the response before it reaches the user |

---

## LangGraph State Machine

The booking flow is modeled as a LangGraph state machine. Each state defines:
- What data to collect from the user
- Validation rules for the collected data
- Transition conditions to the next state
- Fallback behavior on invalid input

```python
# Simplified state definition example
class BookingState(TypedDict):
    session_id: str
    user_id: str
    current_state: BookingStateEnum
    service_type: Optional[str]
    vendor_id: Optional[str]
    booking_date: Optional[str]
    time_slot: Optional[str]
    confirmed: bool
    turn_count: int
    emotion_context: dict
    crisis_flag: bool
```

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| LLM inference | AWS Bedrock (Claude 3) |
| State machine | LangGraph |
| Real-time transport | WebSocket (FastAPI) |
| Session management | Redis (TTL-based) |
| Voice input | Whisper STT |
| Voice output | ElevenLabs TTS |
| API layer | FastAPI + Python |
| Deployment | Docker · AWS · PM2 |

---

## Quickstart

### Prerequisites

- Python 3.11+
- AWS account with Bedrock access (Claude 3 model enabled)
- Redis instance running locally or on AWS ElastiCache
- ElevenLabs API key (for TTS)

### 1. Clone and install

```bash
git clone https://github.com/Shubha889/llm-voice-booking-engine.git
cd llm-voice-booking-engine
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
# Fill in your values:
# AWS_REGION, AWS_BEDROCK_MODEL_ID
# REDIS_URL
# ELEVENLABS_API_KEY
```

### 3. Run locally

```bash
uvicorn src.main:app --reload --port 8000
```

### 4. Connect via WebSocket

```python
import websockets
import asyncio
import json

async def test_booking():
    uri = "ws://localhost:8000/ws/booking/session-123"
    async with websockets.connect(uri) as ws:
        # Send a user utterance
        await ws.send(json.dumps({
            "type": "user_utterance",
            "text": "I need a plumber tomorrow morning"
        }))
        response = await ws.recv()
        print(json.loads(response))

asyncio.run(test_booking())
```

---

## Project Structure

```
llm-voice-booking-engine/
├── src/
│   ├── main.py                        # FastAPI app + WebSocket handler
│   ├── pipeline/
│   │   ├── stage_01_input_guardrails.py
│   │   ├── stage_02_role_resolver.py
│   │   ├── stage_03_context_awareness.py
│   │   ├── stage_04_emotion_tone.py
│   │   ├── stage_05_intent_classifier.py
│   │   ├── stage_06_compliance.py
│   │   ├── stage_07_decide_action.py
│   │   ├── stage_08_supervisor_router.py
│   │   └── stage_09_output_guardrails.py
│   ├── state_machine/
│   │   ├── booking_graph.py           # LangGraph graph definition
│   │   ├── states.py                  # State enum + TypedDict
│   │   └── transitions.py             # Transition logic per state
│   ├── bedrock/
│   │   └── client.py                  # AWS Bedrock wrapper
│   ├── session/
│   │   └── redis_manager.py           # Session TTL management
│   └── voice/
│       ├── stt.py                     # Whisper STT wrapper
│       └── tts.py                     # ElevenLabs TTS wrapper
├── tests/
│   ├── test_pipeline.py
│   ├── test_state_machine.py
│   └── test_session.py
├── configs/
│   └── engine_config.yaml
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── README.md
```

---

## Configuration

```yaml
# configs/engine_config.yaml
bedrock:
  region: "ap-south-1"
  model_id: "anthropic.claude-3-sonnet-20240229-v1:0"
  max_tokens: 1024
  temperature: 0.3

session:
  redis_ttl_seconds: 3600
  max_turns_per_session: 50

pipeline:
  guardrails_strict_mode: true
  emotion_ema_alpha: 0.3        # smoothing factor for emotion scoring
  crisis_confidence_threshold: 0.85

tts:
  provider: "elevenlabs"
  voice_id: "your-voice-id-here"
  stability: 0.5
  similarity_boost: 0.75
```

---

## Running Tests

```bash
pytest tests/ -v --cov=src --cov-report=term-missing
```

---

## License

MIT License — see [LICENSE](LICENSE)

---

## Author

**Shubha K** · [LinkedIn](https://linkedin.com/in/shubha1111) · [Shubha889@gmail.com](mailto:Shubha889@gmail.com)
