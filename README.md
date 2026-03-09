# Persona CV

AI 기반 명함 인터뷰 서비스. 방문자가 자연어로 질문하면 AI가 해당 인물처럼 1인칭으로 답변합니다.

## 데모

`/@username` 형태의 URL로 누구나 접근 가능. 로그인 없이 AI와 대화할 수 있습니다.

## 주요 기능

- **AI 명함 채팅** — 등록된 이력서·포트폴리오 문서를 기반으로 RAG 답변
- **멀티턴 대화** — 이전 대화 히스토리를 LLM에 전달해 맥락 유지
- **Citation** — 답변 근거 문서 인용 표시 (번호 클릭 시 원문 발췌 팝오버)
- **문서 포트폴리오** — PDF·GitHub·LinkedIn·URL 등록 및 3D 캐러셀 표시
- **Interviewer 설정** — 답변 길이·말투·언어·질문 스타일 실시간 조정
- **Owner 대시보드** — 프로필·문서·페르소나 설정·대화 히스토리 관리

## 기술 스택

| Layer | Tech |
|---|---|
| Frontend | Next.js 15 (App Router) · TypeScript · Tailwind CSS · shadcn/ui |
| Backend | FastAPI · uvicorn |
| LLM | OpenRouter `z-ai/glm-4.7-flash` |
| Embedding | OpenRouter `openai/text-embedding-3-small` (1536차원) |
| Vector DB | Supabase pgvector |
| Storage | Supabase Storage (비공개 버킷 · Signed URL) |
| Auth | Supabase Auth |

## 구조

```
persona-cv/
├── frontend/          # Next.js 15 앱
│   ├── src/app/
│   │   ├── [username]/    # 방문자 명함 페이지 (SSR)
│   │   ├── dashboard/     # Owner 대시보드
│   │   └── api/           # FastAPI 프록시 (chat, process-document)
│   └── src/components/
│       ├── persona/       # 방문자용 컴포넌트 (채팅, citation, 캐러셀)
│       └── dashboard/     # 대시보드 섹션 컴포넌트
└── backend/           # FastAPI 서버
    ├── routers/       # chat, documents
    └── services/      # embeddings, search, llm, document_processor
```

## 로컬 실행

### 사전 요구사항

- Node.js 18+
- Python 3.11+
- Supabase 프로젝트
- OpenRouter API 키

### 프론트엔드

```bash
cd frontend
npm install
cp .env.local.example .env.local   # 환경변수 설정
npm run dev                         # http://localhost:3000
```

### 백엔드

```bash
cd backend
uv sync                             # 또는 pip install -r requirements.txt
cp .env.example .env               # 환경변수 설정
uv run uvicorn main:app --port 8001 --reload
```

### 환경변수

**frontend/.env.local**

```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
FASTAPI_URL=http://localhost:8001
```

**backend/.env**

```env
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
OPENROUTER_API_KEY=
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
OPENROUTER_MODEL=z-ai/glm-4.7-flash
OPENROUTER_EMBEDDING_MODEL=openai/text-embedding-3-small
```

### DB 마이그레이션

```bash
cd frontend
npx supabase db push
```

## 문서 처리 흐름

```
PDF 업로드 → Supabase Storage → 텍스트 추출 (pdfplumber)
                                      ↓
                               청크 분할 + 임베딩
                                      ↓
                           document_chunks 테이블 저장
                                      ↓
                     질문 → 벡터 검색 → RAG 컨텍스트 → LLM 스트리밍
```

## 테스트

```bash
# 프론트엔드 (Vitest)
cd frontend && npm test

# 백엔드 (pytest)
cd backend && uv run pytest tests/
```
