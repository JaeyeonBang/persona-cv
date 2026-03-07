# persona-cv (모노리포 루트)

AI 명함 인터뷰 서비스. 방문자가 자연어로 질문하면 AI가 해당 인물처럼 답변.

## 구조

```
persona-cv/
  frontend/   → persona-cv-frontend (Next.js 15)
  backend/    → persona-cv-backend (FastAPI)
```

## 서브모듈 관리

```bash
# 최초 클론 시
git clone --recurse-submodules https://github.com/JaeyeonBang/persona-cv.git

# 서브모듈 업데이트
git submodule update --remote --merge

# 서브모듈 커밋 포인터 업데이트 후 부모 커밋
git add frontend backend
git commit -m "chore: update submodule refs"
```

## 로컬 실행

```bash
# 프론트엔드 (포트 3000)
cd frontend && npm install && npm run dev

# 백엔드 (포트 8001)
cd backend && pip install -r requirements.txt && uvicorn main:app --port 8001 --reload
```

## 환경변수

- `frontend/.env.local` — Supabase, OpenRouter, FASTAPI_URL
- `backend/.env` — Supabase, OpenRouter

## 기술 스택

| Layer | Tech |
|---|---|
| Frontend | Next.js 15 (App Router) + TypeScript + Tailwind + shadcn/ui |
| Backend | FastAPI + uvicorn |
| LLM | OpenRouter `z-ai/glm-4.7-flash` |
| Embedding | OpenRouter `openai/text-embedding-3-small` (dim: 1536) |
| Vector DB | Supabase pgvector |
| Storage | Supabase Storage (bucket: documents) |
| Auth | Supabase Auth |
