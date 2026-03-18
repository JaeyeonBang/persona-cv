# PersonaID MVP — 개발 계획 (prompt_plan.md)

> 최종 확정: 2026-03-07 | 최종 업데이트: 2026-03-11
> 빌드 순서: Phase 1 → 2 → 3 → 4 → 5 → ... → Phase 10

---

## 확정 스택

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router, TypeScript) + Tailwind CSS + shadcn/ui |
| LLM | OpenRouter API — GLM-4.7 Flash |
| Vector DB | Supabase pgvector |
| Graph DB | Graphiti (getzep/graphiti) — Docker 실행 |
| Cache/History | Supabase (conversations 테이블) |
| Auth/Storage | Supabase Auth + Supabase Storage |
| Deploy | Vercel |

---

## Phase 1: 프로젝트 기반 + 라우팅

**목표**: 인증, 스토리지, DB 연결이 동작하는 골격

### Tasks
- [ ] Next.js 15 (TypeScript, App Router) 프로젝트 생성
- [ ] Supabase 클라이언트 연결 (Auth + pgvector extension 활성화)
- [ ] 환경변수 스키마 정의 (zod, `src/lib/env.ts`)
- [ ] Supabase DB 스키마 마이그레이션 실행 (users, documents, conversations)
- [ ] 기본 라우팅 구조 생성:
  - `/` — 랜딩 페이지
  - `/dashboard` — Owner 대시보드 (auth guard)
  - `/[username]` — Visitor 명함/인터뷰 페이지
- [ ] Next.js Middleware (`src/middleware.ts`) — 세션 갱신 + 라우트 보호

### 완료 기준
- `npm run dev` 정상 실행
- Supabase 연결 확인 (ping)
- 3개 라우트 접근 가능

---

## Phase 2: Visitor 인터뷰 UI (더미 데이터)

**목표**: `/[username]` 페이지 완성 — 더미 데이터로 UI 전체 동작

### 더미 데이터 (`src/lib/dummy-data.ts`)
```typescript
export const DUMMY_PERSONA = {
  name: "김민준",
  title: "Frontend Engineer @ Kakao",
  photo: "/dummy/profile.jpg",
  bio: "사용자 경험을 최우선으로 생각하는 프론트엔드 개발자",
  suggestedQuestions: [
    "주요 프로젝트를 소개해줘",
    "협업 스타일이 어떻게 돼?",
    "가장 자신 있는 기술 스택은?",
    "개발자로서의 강점이 뭐야?",
  ],
}
```

### Tasks
- [ ] Visitor 페이지 레이아웃:
  - 상단 60%: 프로필 사진 (Next.js Image) + 이름/직책/bio
  - 하단 40%: 채팅 UI
- [ ] 채팅 버블 컴포넌트 (질문/답변, 스트리밍 텍스트 애니메이션)
- [ ] 바텀 시트 추천 질문 버튼 (shadcn/ui Sheet)
- [ ] Interviewer 설정 패널 (우상단 설정 아이콘):
  - 답변 길이: 간결 / 보통 / 상세 (Slider)
  - 언어: 한국어 / English (Select)
  - 말투: 격식체 / 반말 (ToggleGroup)
  - 질문 스타일: 자유형 / 면접관 모드 / 가벼운 대화 (RadioGroup)
  - 출처 표시: ON/OFF (Switch)
- [ ] 더미 응답 함수 (실제 API 연동 전 목업)
- [ ] 모바일 반응형 최적화 (375px ~ 430px 기준)

### 완료 기준
- 더미 데이터로 전체 UI 동작
- 추천 질문 버튼 클릭 → 채팅 버블 표시
- Interviewer 설정 변경 → 상태 유지

---

## Phase 3: Owner 데이터 입력 대시보드

**목표**: `/dashboard` 완성 — Owner 정보 등록 및 페르소나 설정

### Tasks
- [ ] 프로필 기본 정보 섹션:
  - 이름, 직책, 한줄 소개 입력
  - 프로필 사진 업로드 (Supabase Storage, drag & drop)
  - 공개 username 설정 (중복 확인)
- [ ] 문서 업로드 섹션:
  - PDF 이력서 드래그&드롭 (업로드 상태: 처리 중 / 완료 / 오류)
  - 노션/웹 포트폴리오 URL 입력
  - GitHub URL, LinkedIn URL
  - 기타 링크 (동적 추가/삭제)
- [ ] 페르소나 설정 섹션:
  - 말투 프리셋 버튼 (전문적 / 친근한 / 도전적)
  - 자유 텍스트 보완 필드
  - 방문자 기본 설정값 지정 (Owner용 Interviewer 기본값)
  - 추천 질문 커스터마이징 (추가/삭제)
- [ ] "방문자로 보기" 버튼 → `/[username]` 이동
- [ ] Supabase에 데이터 저장 (Server Actions)

### 완료 기준
- 모든 입력 폼 동작
- 파일 업로드 → Supabase Storage 저장 확인
- 저장된 데이터 → Visitor 페이지에서 반영

---

## Phase 4: RAG 파이프라인 (실제 AI 연동)

**목표**: 문서 인덱싱 + OpenRouter 답변 생성

### Tasks
- [ ] 문서 처리 파이프라인 (Next.js API Route):
  - PDF 파싱 (`pdf-parse`)
  - 웹 URL 스크래핑 (`cheerio`)
  - 텍스트 청킹 + 임베딩 → Supabase pgvector 저장
- [ ] Graphiti Docker 설정:
  - `docker-compose.yml` 작성
  - Graphiti REST API 브릿지 (`/api/graphiti/*`)
  - 텍스트 → Knowledge Graph 변환 (프로젝트-기술-성과 관계)
- [ ] 하이브리드 검색 로직 (`src/lib/search.ts`):
  - Vector 검색 (Supabase pgvector)
  - Confidence score < 0.7 → Graphiti 폴백
- [ ] OpenRouter 연동 (`src/lib/llm.ts`):
  - 모델: GLM-4.7 Flash
  - 시스템 프롬프트 동적 생성 (페르소나 + Interviewer 설정 반영)
  - 스트리밍 응답 (Vercel AI SDK)
- [ ] Citation 포함 응답 포맷 (출처 표시 ON일 때)

### 완료 기준
- PDF 업로드 → 임베딩 저장 확인
- 질문 → Vector 검색 → OpenRouter 답변 스트리밍
- 복합 질문 → Graphiti 폴백 동작 확인

---

## Phase 5: Q&A 캐시 & 히스토리

**목표**: 대화 저장 + 유사 질문 캐시 반환

### Tasks
- [ ] 모든 Q&A → `conversations` 테이블 저장 (session_id, 설정값 포함)
- [ ] 캐시 로직:
  - 질문 임베딩 생성
  - Vector 유사도 > 0.95 → 저장된 답변 즉시 반환
  - 캐시 히트 시 `is_cached: true` 표시
- [ ] Owner 히스토리 대시보드:
  - 최근 방문자 질문 목록
  - 자주 묻는 질문 Top 5
  - 방문 세션 수 통계

### 완료 기준
- 동일 질문 2회 → 2번째는 캐시 반환 확인
- 대시보드에서 히스토리 조회 가능

---

---

## Phase 6: FastAPI 백엔드 분리 (리팩터링)

**목표**: RAG·AI 연산을 Next.js에서 분리 → FastAPI로 이관, Next.js는 순수 UI/BFF 역할

### 배경
- Phase 4·5의 RAG 파이프라인(임베딩 생성, LLM 호출, Graphiti 통신)은 Next.js Server Actions에서 구현
- 연산이 무거워질수록 Vercel Edge 제약에 걸리고 재사용성이 떨어짐
- FastAPI로 분리해 독립 배포 가능하고 Python 생태계(langchain, sentence-transformers 등) 활용

### 아키텍처 변경

```
[ Next.js 15 (Vercel) ]
  └─ UI + Server Actions (auth, profile, document CRUD)
  └─ API calls → FastAPI

[ FastAPI (별도 서버/Docker) ]
  └─ POST /api/chat          — RAG 검색 + LLM 스트리밍 응답
  └─ POST /api/documents/process  — PDF 파싱 + 임베딩 저장
  └─ GET  /api/documents/search   — Vector 검색
  └─ POST /api/graphiti/*    — Graphiti 브릿지
  └─ GET  /api/conversations/cache — Q&A 캐시 조회
```

### Tasks

#### 6-1. FastAPI 프로젝트 구조
- [ ] `backend/` 디렉터리 생성
  ```
  backend/
  ├── main.py
  ├── requirements.txt
  ├── Dockerfile
  ├── routers/
  │   ├── chat.py          # /api/chat (스트리밍)
  │   ├── documents.py     # /api/documents/*
  │   ├── graphiti.py      # /api/graphiti/*
  │   └── conversations.py # /api/conversations/*
  ├── services/
  │   ├── llm.py           # OpenRouter 클라이언트
  │   ├── embeddings.py    # 임베딩 생성
  │   ├── search.py        # 하이브리드 검색 (pgvector + Graphiti)
  │   └── document_processor.py  # PDF 파싱, URL 스크래핑
  └── db/
      └── supabase.py      # Supabase Python 클라이언트
  ```
- [ ] `docker-compose.yml` — FastAPI + Graphiti 통합

#### 6-2. 핵심 엔드포인트 구현
- [ ] `POST /api/chat` — StreamingResponse (SSE)
  - 입력: `{ username, question, interviewer_config, session_id }`
  - 하이브리드 검색 → 프롬프트 조립 → OpenRouter 스트리밍
- [ ] `POST /api/documents/process` — 문서 처리 워커
  - PDF 파싱 (`pdfplumber`) / URL 스크래핑 (`playwright` or `httpx`)
  - 청킹 → 임베딩 → pgvector 저장
  - document.status 업데이트 (pending→done|error)
- [ ] `POST /api/conversations` — Q&A 저장 + 캐시 체크
  - 유사도 > 0.95 → 캐시 히트 반환

#### 6-3. Next.js 마이그레이션
- [ ] Phase 4에서 구현한 Next.js API Routes → FastAPI 호출로 교체
- [ ] `src/lib/api-client.ts` 생성 — FastAPI base URL 래핑
- [ ] 환경변수 추가: `FASTAPI_URL=http://localhost:8001`
- [ ] Visitor 페이지 스트리밍: `fetch` SSE → ReadableStream 처리

#### 6-4. 배포
- [ ] FastAPI → Docker 이미지 빌드
- [ ] Vercel 환경변수에 `FASTAPI_URL` 추가 (프로덕션 FastAPI URL)
- [ ] CORS 설정 (Vercel 도메인만 허용)

### 완료 기준
- FastAPI 독립 실행: `uvicorn main:app --reload`
- Next.js → FastAPI 채팅 스트리밍 동작
- 문서 처리 FastAPI에서 완료, status 업데이트 확인
- docker-compose로 전체 스택 로컬 실행

---

## 리스크 및 대응

| Level | Risk | Mitigation |
|---|---|---|
| HIGH | Graphiti ↔ Next.js REST 통신 | docker-compose로 로컬 테스트 후 진행 |
| MEDIUM | GLM-4.7 Flash 응답 포맷 차이 | 모델 추상화 레이어로 스왑 가능하게 |
| MEDIUM | PDF 이미지 기반 파싱 실패 | 텍스트 직접 입력 폴백 UI 제공 |
| LOW | pgvector 캐시 임계값 튜닝 | 0.95로 시작, 실측 후 조정 |

---

---

## Phase 7: Auth 완성

**목표**: 인증 플로우 완성 + 라우트 보호 강화 + API 어뷰징 방지

### 배경
- Phase 1~6에서 로그인/회원가입은 구현됐으나, 미들웨어 기반 보호·비밀번호 재설정·API rate limiting이 누락
- `/api/chat`이 완전 무인증 공개 상태 → LLM 비용 어뷰징 가능
- 현재 각 페이지에서 `redirect('/login')` 패턴 → 미들웨어로 일원화 필요

### Tasks

#### 7-1. Next.js Middleware
- [x] `src/middleware.ts` 생성
  - `@supabase/ssr` `createServerClient`로 세션 쿠키 갱신
  - `/dashboard` 이하: 미인증 → `/login` 리다이렉트
  - `/login`, `/signup`, `/forgot-password`: 인증됨 → `/dashboard` 리다이렉트
  - `/[username]`, `/api/*`, 정적 파일: 통과
- [x] `src/lib/supabase/middleware.ts` — Middleware 전용 Supabase 클라이언트

#### 7-2. 비밀번호 재설정
- [x] `src/app/(auth)/forgot-password/page.tsx` — 이메일 입력 폼 (발송 완료 시 확인 메시지)
- [x] `src/app/(auth)/reset-password/page.tsx` — 새 비밀번호 + 확인 입력 폼
- [x] `(auth)/actions.ts` — `requestPasswordReset()`, `resetPassword()` 추가
- [x] 로그인 페이지에 "비밀번호를 잊으셨나요?" 링크 추가 + `next` param 처리
- [x] `src/app/auth/callback/route.ts` — `type=recovery` 파라미터 → `/reset-password` 분기

#### 7-3. Visitor 채팅 API Rate Limiting
- [x] `src/app/api/chat/route.ts` — IP 기반 in-memory rate limiter 추가
  - 1분당 20회 초과 시 429 반환
  - `X-Forwarded-For` 헤더 기준 IP 추출
  - Map<ip, {count, resetAt}> 구조로 구현

#### 7-4. 버그 수정
- [x] 랜딩 페이지 `/demo-user` → `/demo` 링크 수정

### 완료 기준
- 미인증 상태로 `/dashboard` 접근 → `/login` 리다이렉트 (미들웨어 레벨)
- 로그인 상태로 `/login` 접근 → `/dashboard` 리다이렉트
- 비밀번호 재설정 이메일 발송 → 링크 클릭 → 새 비밀번호 저장 → 로그인 성공
- `/api/chat` 1분 21회 요청 시 429 응답 확인

---

## 현재 진행 상태

- [x] PRD 확정
- [x] 개발 계획 확정
- [x] Phase 1: 프로젝트 기반 (Next.js 15, Supabase 클라이언트, zod env, DB 스키마)
- [x] Phase 2: Visitor UI (더미 데이터) — /demo 에서 확인 가능
- [x] Phase 3: Owner 대시보드 — /dashboard 완성, 저장 데이터 → /[username] 반영
- [x] Phase 4: RAG 파이프라인 (FastAPI document_processor, embeddings, vector search, LLM streaming)
- [x] Phase 5: Q&A 캐시/히스토리 (conversations 저장, 유사도 캐시 체크)
- [x] Phase 6: FastAPI 백엔드 분리 (routers/chat, documents, Next.js 프록시)
- [x] Phase 7: Auth 완성 (Middleware, 비밀번호 재설정, Rate Limiting) — 2026-03-09
- [x] Phase 8: Graphiti 연동 (Graph RAG, 하이브리드 검색 폴백) — 2026-03-09
- [x] Phase 9: Citation UI + Owner 히스토리 대시보드 — 2026-03-09
- [x] 버그 픽스 & 기술 부채 정리 — 2026-03-18
- [x] Phase 12: 방문자 피드백 + 인사이트 차트 — 2026-03-18
- [x] Phase 13: QR코드 / 명함 공유 — 2026-03-18
- [x] Phase 14: 방문자 페이지 테마 커스터마이징 — 2026-03-18

## Phase 8 완료 내역 (2026-03-09)

- graphiti_client.py: is_available(), add_episode(), search(), graph_results_to_context() — Neo4j 미연결 시 graceful fallback
- services/search.py: hybrid_search() — vector 유사도 < 0.7 시 Graphiti 폴백
- routers/chat.py: hybrid_search 연동, graph_fallback SSE 이벤트 emit, citations에 source 필드 추가
- routers/graphiti.py: GET /api/graphiti/status, POST /api/graphiti/search, POST /api/graphiti/ingest
- docker-compose.yml: Neo4j + FastAPI 서비스 구성
- Pytest: 18 tests 통과 (test_graphiti.py 11개 + test_search.py 7개)

## Phase 9 완료 내역 (2026-03-09)

- Citation type: source 필드 추가 ('vector' | 'graph')
- ChatBubble: graphFallback 프로퍼티 추가, "Graph AI 보강" 보라색 배지 표시
- visitor-page: graph_fallback SSE 이벤트 처리
- proxy.ts: Next.js 16.1+ 요구사항 맞게 export 함수명 middleware → proxy로 수정
- HistorySection: 통계 카드(총 대화 수, 방문 세션 수, 최다 질문 횟수) + Top 5 FAQ + 최근 50건 목록
- Vitest: 137 tests 통과

## Phase 3 완료 내역 (2026-03-07)

- 프로필 섹션: 이름/직책/bio/사진(drag&drop)/username 저장
- 문서 섹션:
  - 타입 버튼 통합: [PDF 파일 첨부] [노션/웹] [GitHub] [LinkedIn] [기타]
  - PDF: storage 선업로드 → 미저장 이탈 시 자동 삭제 (storage 절약)
  - URL: 로컬 스테이징 → "저장 (N개)" 일괄 저장
  - 한글 파일명 버그 수정 (`sanitizeStorageKey`)
- 페르소나 설정: 말투 프리셋 / 커스텀 프롬프트 / 방문자 기본값
- 추천 질문: 추가/삭제/저장
- 방문자로 보기 → /[username] (실제 DB 데이터 반영)
- Vitest 테스트: 38개 전부 통과

## Phase 6 완료 내역 (2026-03-07)

- FastAPI backend: routers/chat (SSE), routers/documents, services/llm, services/search, services/embeddings, services/document_processor, services/conversations
- Next.js → FastAPI 프록시: /api/chat, /api/process-document
- Pytest: 32개 테스트 통과

---

## Phase 10: 예상 Q&A (Pinned FAQ)

**목표**: Owner가 예상 Q&A를 미리 등록하고 AI 초안 답변을 확인/수정. 채팅 시 우선 활용.

### 배경
- 방문자가 자주 하는 질문에 대해 Owner가 직접 답변을 준비하고 싶음
- AI가 RAG로 생성하는 답변보다 Owner가 직접 큐레이션한 답변이 더 정확할 수 있음
- 온보딩 UX로 프로필 설정 직후 Q&A 등록을 유도

### 완료 기준
- [ ] 프로필 저장 후 `/dashboard/onboarding` 페이지로 이동
- [ ] 질문 입력 → "AI 답변 생성" → 답변 프리뷰 → 텍스트 편집 → 저장
- [ ] 대시보드에서도 언제든 Q&A 수정/추가/삭제
- [ ] 방문자 채팅 시 pinned_qa가 시스템 프롬프트에 포함되어 우선 활용
- [ ] Pytest 신규 테스트 통과
- [ ] Playwright 스크린샷으로 UI 동작 확인

### Tasks

#### 10-1. DB 스키마
- [ ] `frontend/supabase/migrations/004_pinned_qa.sql`
  ```sql
  CREATE TABLE pinned_qa (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id uuid REFERENCES users(id) ON DELETE CASCADE,
    question text NOT NULL,
    answer text NOT NULL,
    display_order int DEFAULT 0,
    created_at timestamptz DEFAULT now(),
    updated_at timestamptz DEFAULT now()
  );
  ALTER TABLE pinned_qa ENABLE ROW LEVEL SECURITY;
  CREATE POLICY "read_all" ON pinned_qa FOR SELECT USING (true);
  CREATE POLICY "owner_write" ON pinned_qa FOR ALL USING (
    user_id IN (SELECT id FROM users WHERE auth_id = auth.uid())
  );
  ```

#### 10-2. 백엔드 API
- [ ] `backend/routers/pinned_qa.py`
  - `GET  /api/pinned-qa?username=xxx` — Q&A 목록 조회 (공개)
  - `POST /api/pinned-qa` — 새 Q&A 저장 (username으로 user 찾기)
  - `PUT  /api/pinned-qa/{id}` — 답변 수정
  - `DELETE /api/pinned-qa/{id}` — 삭제
  - `POST /api/pinned-qa/generate` — AI 답변 초안 생성 (LLM 호출)
- [ ] `backend/main.py` — 라우터 등록

#### 10-3. 채팅 우선순위 연동
- [ ] `backend/services/agents/state.py` — `pinned_qa_context: str` 필드 추가
- [ ] `backend/services/agents/nodes.py` — `retrieval_node`에서 pinned_qa 조회 후 state에 추가
- [ ] `backend/services/llm.py` — `build_system_prompt`에 "## 사전 준비된 답변" 섹션 추가

#### 10-4. 프론트엔드
- [ ] `frontend/src/lib/types.ts` — `PinnedQA` 타입 추가
- [ ] `frontend/src/app/api/pinned-qa/[[...path]]/route.ts` — FastAPI 프록시
- [ ] `frontend/src/app/dashboard/onboarding/page.tsx` — 온보딩 Q&A 등록 페이지
- [ ] `frontend/src/components/dashboard/pinned-qa-section.tsx` — 대시보드 섹션
- [ ] `frontend/src/app/dashboard/page.tsx` — PinnedQASection 추가

#### 10-5. 테스트
- [ ] `backend/tests/test_pinned_qa.py` — CRUD + generate 엔드포인트 테스트
- [ ] Playwright 스크린샷 검증

### 파일 변경 목록 (12개)
| 분류 | 파일 |
|---|---|
| 신규 | `frontend/supabase/migrations/004_pinned_qa.sql` |
| 신규 | `backend/routers/pinned_qa.py` |
| 신규 | `frontend/src/app/dashboard/onboarding/page.tsx` |
| 신규 | `frontend/src/components/dashboard/pinned-qa-section.tsx` |
| 신규 | `frontend/src/app/api/pinned-qa/[[...path]]/route.ts` |
| 신규 | `backend/tests/test_pinned_qa.py` |
| 수정 | `backend/main.py` |
| 수정 | `backend/services/agents/nodes.py` |
| 수정 | `backend/services/agents/state.py` |
| 수정 | `backend/services/llm.py` |
| 수정 | `frontend/src/app/dashboard/page.tsx` |
| 수정 | `frontend/src/lib/types.ts` |

---

## Phase 11: 예상 Q&A UX 개선 + 포트폴리오 인용

**목표**: 예상 Q&A를 대시보드 탭으로 통합하고, 답변 작성 시 포트폴리오 문서를 직접 인용할 수 있는 인터페이스 추가.

### 11-1. 대시보드 탭 재설계
- [x] `/dashboard?tab=qa` 탭으로 예상 Q&A 통합 (별도 페이지 → 탭 전환)
- [x] 탭 순서 변경: 설정 / 예상 Q&A / 대화 히스토리
- [x] 설정 완료 버튼 → `?tab=qa` 이동
- [x] PC UI 양 열 하단 라인 맞춤 (`flex-1 [&>section]:h-full`)

### 11-2. 인라인 편집 + AI 자동 추천
- [x] 편집 버튼 제거 — textarea 항상 활성화 (인라인 편집)
- [x] Dirty state 감지 → 저장 버튼 노출
- [x] `POST /api/pinned-qa/suggest` 엔드포인트 — 2-step LLM
  - Step 1: 문서 + 프로필 분석 → 면접 예상 질문 5개 (plain text)
  - Step 2: 질문별 병렬 답변 생성 (`asyncio.gather`)
  - GLM-4.7-flash max_tokens: suggest=8000, generate=4000

### 11-3. 포트폴리오 인용 UI (TDD)
- [x] `frontend/src/lib/qa-citation.ts` — 순수 함수 3개
  - `insertCitationAtCursor(current, cursorPos, doc)` — 커서 위치에 `[출처: 제목]` 삽입
  - `formatCitation(doc)` — `[출처: ${doc.title}]` 반환
  - `filterCitableDocs(docs)` — `status === 'done'` 필터
- [x] `frontend/src/lib/__tests__/qa-citation.test.ts` — 13개 단위 테스트 (전체 통과)
- [x] `frontend/src/components/dashboard/portfolio-citation-panel.tsx`
  - 접이식 패널 (기본 닫힘)
  - 완료 상태 문서만 표시, 타입 아이콘 포함
  - "인용" 버튼 클릭 → `onCite(doc)` 콜백 + 패널 닫힘
- [x] `qa-client.tsx` — `portfolio: Document[]` prop 추가
  - 각 답변 textarea에 `PortfolioCitationPanel` 추가
  - 커서 위치 추적 (`useRef<Map<string, HTMLTextAreaElement>>`)
  - `handleCite(doc, itemId)` — `insertCitationAtCursor` 호출 후 커서 복원
  - "직접 추가" 폼에도 인용 패널 적용
- [x] `dashboard/page.tsx` — `portfolio={docs}` prop 전달

### 파일 변경 목록 (6개)
| 분류 | 파일 |
|---|---|
| 신규 | `frontend/src/lib/qa-citation.ts` |
| 신규 | `frontend/src/lib/__tests__/qa-citation.test.ts` |
| 신규 | `frontend/src/components/dashboard/portfolio-citation-panel.tsx` |
| 수정 | `frontend/src/app/dashboard/qa/qa-client.tsx` |
| 수정 | `frontend/src/app/dashboard/page.tsx` |
| 수정 | `frontend/src/app/dashboard/profile-section.tsx` |

---

## 버그 픽스 & 기술 부채 정리 (2026-03-18)

### 수정 내역
- [x] `backend/CLAUDE.md` — 실제 엔드포인트 전체 반영 (pinned_qa, graphiti, conversations)
- [x] `backend/services/conversations.py` — `save_conversation`에 `conversation_id` 파라미터 추가 (uuid pre-generate 지원)
- [x] `frontend/src/lib/dummy-data.ts` — `DUMMY_PERSONA`에 `theme: 'default'` 추가 (TS 타입 정합성)

---

## Phase 12: 방문자 인사이트 + 피드백 (2026-03-18)

**목표**: 방문자 답변에 좋아요/별로예요 피드백, 대시보드에 일별 추이 차트 + 만족도 통계.

### 완료 내역
- [x] `frontend/supabase/migrations/005_feedback_theme.sql` — conversations.feedback 컬럼, users.theme 컬럼 추가
- [x] `backend/routers/conversations.py` (신규) — `POST /api/conversations/{id}/feedback`
- [x] `backend/routers/chat.py` — SSE 첫 이벤트로 `conversation_id` 방출, 백그라운드 저장 시 동일 ID 사용
- [x] `backend/main.py` — conversations 라우터 등록
- [x] `frontend/src/app/api/conversations/[...path]/route.ts` (신규) — Next.js 프록시
- [x] `frontend/src/components/persona/chat-bubble.tsx` — 피드백 버튼(👍/👎), `conversationId`/`feedback` 필드 추가
- [x] `frontend/src/components/persona/chat-messages.tsx` — `onFeedback` prop 전달
- [x] `frontend/src/components/persona/visitor-page.tsx` — `conversation_id` SSE 수신, 피드백 API 호출
- [x] `frontend/src/components/dashboard/history-section.tsx` — 일별 차트(7일), 만족도 통계 카드, 피드백 이모지 표시

---

## Phase 13: QR코드 / 명함 공유 (2026-03-18)

**목표**: 대시보드에서 QR코드 생성 + URL 복사 + 소셜 공유, 오프라인 네트워킹 지원.

### 완료 내역
- [x] `npm install qrcode @types/qrcode`
- [x] `frontend/src/components/dashboard/qr-share-section.tsx` (신규) — QR 코드(200px), URL 복사, Web Share API, QR 이미지 다운로드
- [x] `frontend/src/app/dashboard/page.tsx` — 설정 탭 하단에 QRShareSection 추가

---

## Phase 14: 방문자 페이지 테마 커스터마이징 (2026-03-18)

**목표**: Owner가 4가지 테마(기본/테크/크리에이티브/비즈니스) 중 선택해 방문자 명함 페이지 색상 변경.

### 완료 내역
- [x] `frontend/supabase/migrations/005_feedback_theme.sql` — users.theme 컬럼 추가 (위와 동일 파일)
- [x] `frontend/src/lib/types.ts` — `Theme` 타입 추가, `User`/`Persona`에 theme 필드
- [x] `frontend/src/lib/user-to-persona.ts` — theme 매핑 추가
- [x] `frontend/src/app/dashboard/actions.ts` — `saveTheme()` Server Action 추가
- [x] `frontend/src/components/dashboard/theme-section.tsx` (신규) — 4가지 테마 카드 UI, dirty state 저장 버튼
- [x] `frontend/src/app/dashboard/page.tsx` — 설정 탭에 ThemeSection 추가
- [x] `frontend/src/components/persona/visitor-page.tsx` — THEME_STYLES 맵으로 테마별 색상 적용

### 테마 스펙
| 테마 | 배경 | 카드 | 용도 |
|---|---|---|---|
| default | zinc-50 | white | 기본 |
| tech | gray-950 | gray-900 | 개발자/테크 |
| creative | violet→pink 그라데이션 | white/80 backdrop | 크리에이터 |
| business | slate-100 | white | 비즈니스/컨설턴트 |

