# AI-Chatbot-LineOA — Cheatsheet

## Architecture at a Glance

```
LINE User → LINE Platform → POST /webhook-dev → FastAPI
→ extract_line_event() → show_loading_animation()
→ load_session() → LangGraph.ainvoke()
→ session_gate → intent_classify / resume / handoff
→ [hospital_info | nurse | package | concierge] subgraph
→ save_session() → reply_to_line() → LINE User
```

---

## Key Files

| File | หน้าที่ |
|------|--------|
| `src/app/webhook_router.py` | FastAPI endpoints, lifespan, semaphore |
| `src/app/app_config.py` | Constants (model, log level, max concurrent) |
| `src/agent/graph_stub.py` | Main LangGraph definition |
| `src/agent/shared/state_stub.py` | AgentStateStub TypedDict |
| `src/agent/shared/llm_factory.py` | LLM provider factory (cached) |
| `src/agent/stub_nodes/session_gate_stub.py` | Node 1: prepare + routing |
| `src/agent/stub_nodes/intent_router/node.py` | Intent + route classification |
| `src/agent/stub_nodes/hospital_info_agent_module/` | Hospital info subgraph |
| `src/integrations/line/handler_stub.py` | LINE event → session → AI |
| `src/integrations/line/messaging.py` | Reply + loading animation |
| `src/integrations/line/utils.py` | Extract LINE event |
| `src/infra/session_model.py` | Session dataclass |
| `src/infra/session_store/memory.py` | In-memory session store |
| `src/infra/session_store/redis.py` | Redis session store |

---

## LangGraph Graph Structure

```
START
  └── session_gate
        ├── (HANDOFF) → handoff_mode → END
        ├── (ACTIVE)  → resume_router → [module] → END
        └── (else)    → intent_classify → [module] → END

Modules:
  hospital_info_agent_module (subgraph)
    START → route → extract_info → route → fetch_data → route → build_answer → END
  nurse_agent_module
  package_agent_module
  concierge_agent_module
```

---

## State Flow

```
state_in = {
  messages, task_status, conversation_mode,   ← from session
  latest_user_input, postback_data,           ← from LINE event
  line_user_id, line_account, line_token       ← LINE credentials
}

state_out fields of interest:
  final_answer    → text to reply
  line_messages   → Flex messages
  quick_replies   → Quick reply buttons
  task_status     → COMPLETED/ACTIVE
  conversation_mode → AI/HANDOFF
```

---

## ENV Variables

```bash
# Required
LINE_CHANNEL_ACCESS_TOKEN_DEV=xxx
LINE_CHANNEL_ACCESS_TOKEN_OFFICIAL=xxx
OPENAI_API_KEY=xxx

# Optional
SESSION_STORE_BACKEND=memory   # or redis
REDIS_URL=redis://localhost:6379/0
MAX_CONCURRENT_EVENTS=1
LOG_LEVEL=INFO
DEFAULT_LLM_MODEL=gpt-4o-mini
```

---

## Task Status Transitions

```
COMPLETED → (new message) → ACTIVE  [messages cleared]
ACTIVE    → (module done) → COMPLETED
ACTIVE    → (need more info) → ACTIVE  [stays active, asks user]
```

---

## Postback Actions

| Action | เส้นทาง |
|--------|---------|
| `concierge_menu` | intent_classify → fast-track → concierge |
| `hospital_agent` | intent_classify → fast-track → hospital_info |
| `nurse_agent` | intent_classify → fast-track → nurse |
| `ai_mode` | handoff_mode → switch to AI |

---

## Quick Reference: Adding a New Module

1. สร้าง folder ใหม่: `src/agent/stub_nodes/new_module/`
2. สร้าง `subgraph.py` กำหนด nodes + edges
3. เพิ่มใน `RouteModule` Literal ใน `state_stub.py`
4. เพิ่ม node ใน `graph_stub.py`
5. เพิ่ม route mapping ใน `intent_router/node.py`
