# Chat Streaming Optimization Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** LLM 진짜 스트리밍 구현 + factcheck 조건부 실행 + 타이핑 인디케이터 추가로 체감 응답속도 개선.

**Architecture:**
- Backend: LangGraph `astream_events(version="v2")`로 persona 노드 LLM 토큰을 실시간 SSE 전송. factcheck는 context가 없거나 답변이 짧으면 스킵.
- Frontend: `isStreaming=true && content=''` 상태에서 바운스 점 3개 애니메이션 표시. 내용 도착 후 기존 `▋` 커서 유지.

**Tech Stack:** LangGraph astream_events v2, FastAPI SSE, Next.js React, Tailwind CSS

---

### Task 1: factcheck 조건부 스킵

**Files:**
- Modify: `backend/services/agents/nodes.py:142-185`

**현재 상태:**
```python
async def factcheck_node(state: ChatState) -> dict[str, Any]:
    context = state["context"]
    draft = state["draft_answer"]

    # 검색 컨텍스트 없으면 팩트체크 생략 (이미 있음)
    if not context or not context.strip():
        return { "fact_check_passed": True, ... }
```

**Step 1: nodes.py factcheck_node 상단에 짧은 답변 조건 추가**

`nodes.py:148` 아래에 다음 라인 추가:
```python
    # 답변이 짧으면 팩트체크 불필요
    if len(draft.strip()) < 100:
        return {
            "fact_check_passed": True,
            "fact_check_notes": "짧은 답변 — 팩트체크 생략",
            "final_answer": draft,
        }
```

최종 `factcheck_node` 시작부:
```python
async def factcheck_node(state: ChatState) -> dict[str, Any]:
    context = state["context"]
    draft = state["draft_answer"]

    # 검색 컨텍스트 없으면 팩트체크 생략
    if not context or not context.strip():
        return {
            "fact_check_passed": True,
            "fact_check_notes": "참고 문서 없음 — 팩트체크 생략",
            "final_answer": draft,
        }

    # 답변이 짧으면 팩트체크 불필요
    if len(draft.strip()) < 100:
        return {
            "fact_check_passed": True,
            "fact_check_notes": "짧은 답변 — 팩트체크 생략",
            "final_answer": draft,
        }
```

**Step 2: 백엔드 서버 실행해서 에러 없는지 확인**
```bash
cd /mnt/d/projects/persona-cv/backend && uv run python -c "from services.agents.nodes import factcheck_node; print('OK')"
```
Expected: `OK`

**Step 3: 기존 pytest 실행**
```bash
cd /mnt/d/projects/persona-cv/backend && uv run pytest tests/ -v -x 2>&1 | tail -20
```
Expected: 모든 테스트 pass (또는 기존 skip 포함)

---

### Task 2: LangGraph astream_events 기반 실시간 스트리밍

**Files:**
- Modify: `backend/routers/chat.py` (전체 재작성)

**핵심 변경:**
- `chat_graph.ainvoke()` → `chat_graph.astream_events(version="v2")`
- LangGraph 이벤트에서 토큰 추출 → 즉시 SSE 전송
- retrieval 완료 시 citations/graph_fallback 전송
- factcheck 완료 후 백그라운드 저장

**Step 1: chat.py 4번 섹션(LangGraph 실행)과 6번 섹션(SSE 스트리밍)을 하나로 합치기**

`chat.py` 의 `# ── 4. LangGraph 실행 ──` 부터 끝까지를 아래 코드로 교체:

```python
    # ── 4. SSE 스트리밍 (astream_events) ──────────────────────────────────
    async def generate():
        final_state: dict = {}
        draft_answer = ""

        try:
            async for event in chat_graph.astream_events(initial_state, version="v2"):
                kind = event["event"]
                name = event.get("name", "")
                meta = event.get("metadata", {})

                # retrieval 노드 완료 → citations + graph_fallback 전송
                if kind == "on_chain_end" and name == "retrieval":
                    output = event["data"].get("output", {})
                    final_state.update(output)
                    if output.get("graph_fallback"):
                        yield f"data: {json.dumps({'type': 'graph_fallback', 'used': True}, ensure_ascii=False)}\n\n"
                    citations = output.get("citations", [])
                    yield f"data: {json.dumps({'type': 'citations', 'sources': citations}, ensure_ascii=False)}\n\n"

                # persona 노드 LLM 토큰 스트리밍
                elif kind == "on_chat_model_stream" and meta.get("langgraph_node") == "persona":
                    chunk = event["data"].get("chunk")
                    if chunk and chunk.content:
                        draft_answer += chunk.content
                        yield f"data: {json.dumps({'type': 'text', 'content': chunk.content}, ensure_ascii=False)}\n\n"

                # factcheck 노드 완료 → fact_check_warn 전송
                elif kind == "on_chain_end" and name == "factcheck":
                    output = event["data"].get("output", {})
                    final_state.update(output)
                    if not output.get("fact_check_passed", True):
                        yield f"data: {json.dumps({'type': 'fact_check_warn', 'notes': output.get('fact_check_notes', '')}, ensure_ascii=False)}\n\n"

        except Exception as e:
            logger.error("astream_events failed: %s", e)
            yield f"data: {json.dumps({'type': 'error', 'message': '스트리밍 오류가 발생했습니다.'}, ensure_ascii=False)}\n\n"

        yield "data: [DONE]\n\n"

        # ── 5. 백그라운드 저장 ─────────────────────────────────────────────
        final_answer = final_state.get("final_answer") or draft_answer
        if final_answer:
            try:
                save_conversation(
                    user_id=user["id"],
                    session_id=req.sessionId,
                    question=req.question,
                    answer=final_answer,
                    interviewer_config=req.config,
                    question_embedding=query_embedding,
                )
            except Exception as e:
                logger.warning("save_conversation failed: %s", e)

    return StreamingResponse(
        generate(),
        media_type="text/event-stream; charset=utf-8",
        headers={"X-Content-Type-Options": "nosniff", "Cache-Control": "no-cache"},
    )
```

또한 `background_tasks` import와 파라미터 제거:
- `from fastapi import APIRouter, BackgroundTasks, HTTPException` → `from fastapi import APIRouter, HTTPException`
- `async def chat(req: ChatRequest, background_tasks: BackgroundTasks):` → `async def chat(req: ChatRequest):`

**Step 2: 백엔드 import 오류 없는지 확인**
```bash
cd /mnt/d/projects/persona-cv/backend && uv run python -c "from routers.chat import router; print('OK')"
```
Expected: `OK`

**Step 3: 서버 실행 후 curl로 스트리밍 동작 확인**
```bash
curl -s -N -X POST http://localhost:8001/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","question":"안녕하세요","config":{},"sessionId":"plan-test"}' \
  --max-time 30 | head -20
```
Expected: `data: {"type": "citations", ...}` 후 `data: {"type": "text", ...}` 청크들이 실시간 출력

---

### Task 3: 타이핑 인디케이터 (... 바운스 애니메이션)

**Files:**
- Modify: `frontend/src/components/persona/chat-bubble.tsx`

**현재 동작:**
- `isStreaming=true`일 때 content 유무 관계없이 `▋` 블링크 표시

**목표 동작:**
- `isStreaming=true && content === ''` → 바운스 점 3개 (`···`) 표시 (AI가 생성 준비 중)
- `isStreaming=true && content !== ''` → 기존 `▋` 커서 표시 (생성 중)

**Step 1: `TypingIndicator` 컴포넌트 추가 + `ChatBubble` 조건 수정**

`chat-bubble.tsx` 상단(imports 아래)에 컴포넌트 추가:
```tsx
function TypingIndicator() {
  return (
    <span className="flex items-center gap-1 py-1">
      {[0, 150, 300].map((delay) => (
        <span
          key={delay}
          className="size-2 rounded-full bg-zinc-400 animate-bounce"
          style={{ animationDelay: `${delay}ms` }}
        />
      ))}
    </span>
  )
}
```

`ChatBubble` 컴포넌트 내 기존 `{message.isStreaming && ...}` 부분 교체:
```tsx
        {/* 타이핑 인디케이터: 내용 도착 전 */}
        {message.isStreaming && !message.content && <TypingIndicator />}
        {/* 스트리밍 커서: 내용 도착 후 */}
        {message.isStreaming && message.content && (
          <span className="ml-1 inline-block animate-pulse text-zinc-400">▋</span>
        )}
```

**Step 2: 프론트엔드 빌드 오류 없는지 확인**
```bash
cd /mnt/d/projects/persona-cv/frontend && npx tsc --noEmit 2>&1 | tail -10
```
Expected: 오류 없음

**Step 3: Vitest 실행**
```bash
cd /mnt/d/projects/persona-cv/frontend && npm test -- --run 2>&1 | tail -15
```
Expected: 137 passed (또는 그 이상)

---

### Task 4: E2E 검증

**Step 1: 백엔드 서버 실행 확인**
```bash
curl -s http://localhost:8001/health
```
Expected: `{"status":"ok"}` 또는 200

**Step 2: 프론트엔드 → 백엔드 통합 스트리밍 검증**
```bash
curl -s -N -X POST http://localhost:3000/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","question":"자기소개해줘","config":{},"sessionId":"e2e-stream"}' \
  --max-time 30 | head -30
```
Expected:
1. `data: {"type": "citations", ...}` — 즉시
2. `data: {"type": "text", "content": "안"}` 등 짧은 청크들 — 실시간 스트리밍
3. `data: [DONE]` — 완료

**Step 3: 첫 토큰 도착 시간 측정 (선택)**
```bash
time curl -s -N -X POST http://localhost:8001/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","question":"자기소개","config":{},"sessionId":"ttfb-test"}' \
  --max-time 30 | head -3
```
Expected: 첫 text 청크까지 1~2초 이내 (기존 2~3초 대비 개선)

---

### 변경 요약

| 파일 | 변경 내용 |
|------|---------|
| `backend/services/agents/nodes.py` | factcheck: 짧은 답변(< 100자) 스킵 조건 추가 |
| `backend/routers/chat.py` | ainvoke → astream_events, 토큰 실시간 SSE, BackgroundTasks 제거 |
| `frontend/src/components/persona/chat-bubble.tsx` | TypingIndicator 컴포넌트 추가, isStreaming 조건 분기 |
