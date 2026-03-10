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

## 개발 규칙

### 기능 추가 후 Playwright 테스트 필수 (CRITICAL)

**기능을 추가한 뒤에는 무조건 Playwright로 스크린샷을 찍어 실제 동작을 확인한다.**

```bash
# Playwright 스크린샷 스크립트 예시 (Node.js)
const { chromium } = require('./frontend/node_modules/playwright')
;(async () => {
  const browser = await chromium.launch()
  const page = await browser.newPage()
  await page.goto('http://localhost:3000/<username>', { waitUntil: 'networkidle' })
  await page.screenshot({ path: '/tmp/feature-test.png', fullPage: true })
  // 인터랙션 후 추가 스크린샷
  await browser.close()
})()
```

검증 항목:
- 컴포넌트가 실제로 렌더링되는가
- 인터랙션(클릭, 입력, 네비게이션)이 정상 동작하는가
- 레이아웃이 데스크톱/모바일 모두 정상인가
- 콘솔 에러가 없는가

### 수정 후 보고 전 필수 E2E 검증

**UI/기능 수정 시 반드시 curl로 실제 렌더 확인 후 보고한다.**

```bash
# 레이아웃 변경 검증 예시
curl -s http://localhost:3000/<username> | python3 -c "
import sys
html = sys.stdin.read()
pos_portfolio = html.find('포트폴리오')
pos_chat      = html.find('내 명함 AI')
print('portfolio pos:', pos_portfolio, '/ chat pos:', pos_chat)
print('순서:', 'OK' if pos_portfolio < pos_chat else '오류')
"

# API 동작 검증
curl -s -X POST http://localhost:3000/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","question":"테스트","config":{},"sessionId":"e2e"}' \
  --no-buffer -m 15 | head -5
```

stale SSR, 캐시, 파일 미저장 모두 코드만 보면 못 잡는다. 반드시 실행해서 확인 후 보고한다.

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
