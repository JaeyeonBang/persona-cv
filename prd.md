# [요구사항 정의서] PersonaID: 하이브리드 지식 엔진 기반 AI 페르소나 명함

## 1. 서비스 개요
- **서비스명**: **PersonaID** (가칭)
- **목표**: 사용자의 이력서, 포트폴리오, 가치관을 학습한 **'디지털 트윈'**을 구축하여, 모바일 환경에서 외부 방문자에게 사용자 본인을 완벽하게 대변하는 인터랙티브 AI 명함 서비스 제공.
- **핵심 차별점**: Vector와 Graph DB를 결합한 **하이브리드 검색(RAG)**을 통한 답변 정확도 극대화 및 사용자 친화적인 페르소나 설정 UX.

---

## 2. MVP 범위 (현재 개발 대상)

> Advanced Features (F-401~403)는 MVP 이후 단계에서 다룸.

### 2.0 인증 및 보안 (Auth & Security)
| ID | 요구사항 명칭 | 세부 내용 | 관련 기술/근거 |
|:---|:---|:---|:---|
| **F-001** | 회원가입 | 이메일 + 비밀번호(6자 이상) + username으로 계정 생성. username 중복 확인 후 즉시 사용. | Supabase Auth |
| **F-002** | 로그인 / 로그아웃 | 이메일/비밀번호 로그인. 로그아웃 시 세션 완전 제거 후 `/login` 리다이렉트. | Supabase Auth |
| **F-003** | 비밀번호 재설정 | 로그인 페이지에서 "비밀번호 찾기" → 이메일로 재설정 링크 발송. 링크 클릭 시 새 비밀번호 입력 후 변경. | Supabase Auth (resetPasswordForEmail) |
| **F-004** | 미들웨어 기반 라우트 보호 | `/dashboard/*` 접근 시 미인증이면 `/login`으로 리다이렉트. 이미 로그인한 사용자가 `/login`, `/signup` 접근 시 `/dashboard`로 리다이렉트. 세션 쿠키를 미들웨어에서 갱신(refresh). | Next.js Middleware + @supabase/ssr |
| **F-005** | Visitor 채팅 API 어뷰징 방지 | `/api/chat` 엔드포인트에 IP 기반 요청 제한 적용. 1분당 20회 초과 시 429 반환. 서버 재시작 시 카운터 초기화(in-memory). | Next.js Route Handler |

### 2.1 하이브리드 지식 베이스 구축 (Owner Side)
| ID | 요구사항 명칭 | 세부 내용 | 관련 기술/근거 |
|:---|:---|:---|:---|
| **F-101** | 멀티 소스 데이터 통합 | PDF 이력서, 노션/웹 포트폴리오 링크, GitHub URL, LinkedIn URL, 기타 링크를 입력받아 텍스트 추출 후 벡터화하여 저장. 업로드 상태(처리 중/완료/오류) 표시. | LangChain, pdf-parse, cheerio |
| **F-102** | Graph RAG 인덱싱 | 이력 데이터 내의 '프로젝트-기술스택-성과' 간의 관계를 Knowledge Graph로 자동 구조화. **Graphiti (getzep/graphiti)** 사용. | Graphiti, Docker |
| **F-103** | 가치관/말투 퀵 설정 | '전문적인', '친근한', '도전적인' 등 가치관과 말투를 대표하는 프리셋 버튼 제공. | Prompt Engineering |
| **F-104** | 상세 페르소나 보완 | 프리셋 외 자유 텍스트로 답변 가이드라인 직접 입력. Owner가 방문자 기본 설정값 지정 가능. | System Message Tuning |
| **F-105** | 프로필 기본 정보 입력 | 이름, 직책, 한줄 소개, 프로필 사진 업로드, 공개 username(명함 URL) 설정. | Supabase Storage |

### 2.2 지능형 답변 엔진 (AI Engine)
| ID | 요구사항 명칭 | 세부 내용 | 관련 기술/근거 |
|:---|:---|:---|:---|
| **F-201** | 신뢰도 기반 하이브리드 검색 | Vector 유사도 점수 < 0.7 시 Graphiti 그래프 쿼리 폴백. | Hybrid Search logic |
| **F-202** | 다단계 추론 답변 | 복합 관계 질문에 Graphiti를 활용한 Multi-hop reasoning. | Graphiti |
| **F-203** | 출처 명시 (Citation) | 답변 근거 문서/그래프 노드를 방문자에게 제시. 출처 표시 여부는 Interviewer가 설정 가능. | Source Attribution |
| **F-204** | Q&A 캐시 | 동일/유사 질문(Vector 유사도 > 0.95) 시 저장된 답변 즉시 반환하여 LLM 호출 최소화. | Supabase conversations |
| **F-205** | 대화 히스토리 저장 | 모든 Q&A를 DB에 저장. session_id, question, answer, 검색 컨텍스트, Interviewer 설정값 포함. | Supabase |

### 2.3 모바일 최적화 인터랙션 (Visitor Side)
| ID | 요구사항 명칭 | 세부 내용 | 관련 기술/근거 |
|:---|:---|:---|:---|
| **F-301** | 프로필 사진 표시 | 방문자 화면 상단 60%에 사용자 프로필 사진과 이름/직책 표시. (아바타 기능은 MVP 이후) | Next.js Image |
| **F-302** | 모바일 전용 바텀 시트 UI | 추천 질문 버튼을 화면 하단에 배치. Owner가 추천 질문 커스터마이징 가능. | shadcn/ui Sheet |
| **F-303** | 인라인 미디어 뷰어 | AI가 특정 프로젝트 설명 시 대화창 내에서 이미지/영상 슬라이드 표시. | Dynamic Carousel |
| **F-304** | Interviewer 설정 패널 | 방문자가 인터뷰 경험을 커스터마이징할 수 있는 설정. 아래 세부 항목 참조. | shadcn/ui |

#### F-304 Interviewer 설정 세부 항목
| 설정 | 옵션 |
|:---|:---|
| 답변 길이 | 간결 / 보통 / 상세 (슬라이더) |
| 언어 | 한국어 / English |
| 말투 | 격식체 / 반말 |
| 질문 스타일 | 자유형 / 면접관 모드 / 가벼운 대화 |
| 출처 표시 | ON / OFF 토글 |

---

## 3. 기술 스택 (MVP)

| Layer | Technology |
|:---|:---|
| Frontend | Next.js 15 (App Router, TypeScript) + Tailwind CSS + shadcn/ui |
| LLM | OpenRouter API — **GLM-4.7 Flash** 모델 |
| Vector DB | Supabase pgvector |
| Graph DB | **Graphiti** (getzep/graphiti) — Docker 실행 |
| Cache/History | Supabase (conversations 테이블) |
| Auth/Storage | Supabase Auth + Supabase Storage |
| Deploy | Vercel |

---

## 4. DB 스키마 (MVP)

```sql
-- 사용자 프로필
users (
  id uuid PRIMARY KEY,
  username text UNIQUE,        -- 명함 URL slug
  name text,
  title text,                  -- 직책
  bio text,                    -- 한줄 소개
  photo_url text,
  persona_config jsonb,        -- 페르소나 설정 (말투, 프리셋, 자유텍스트)
  default_interviewer_config jsonb,  -- Owner가 지정하는 방문자 기본 설정
  created_at timestamptz DEFAULT now()
)

-- 업로드 문서 및 임베딩
documents (
  id uuid PRIMARY KEY,
  user_id uuid REFERENCES users(id),
  type text,                   -- 'pdf' | 'url' | 'github' | 'linkedin' | 'other'
  source_url text,
  content text,
  embedding vector(1536),
  status text,                 -- 'processing' | 'done' | 'error'
  created_at timestamptz DEFAULT now()
)

-- Q&A 대화 히스토리 (캐시 겸용)
conversations (
  id uuid PRIMARY KEY,
  user_id uuid REFERENCES users(id),
  session_id text,
  question text,
  question_embedding vector(1536),
  answer text,
  context_used jsonb,          -- 검색에 사용된 문서/그래프 노드
  interviewer_config jsonb,    -- 해당 답변 시 Interviewer 설정값
  is_cached boolean DEFAULT false,
  created_at timestamptz DEFAULT now()
)
```

---

## 5. 기술 및 비기능 요구사항

### 5.1 인프라 및 아키텍처
- **하이브리드 검색 로직**: Vector 유사도 점수 < 0.7일 경우 자동으로 Graphiti 쿼리 실행.
- **캐시 로직**: 동일/유사 질문(유사도 > 0.95) 시 저장된 답변 즉시 반환.
- **Graphiti 연동**: Docker로 실행, Next.js API Route에서 REST 호출.
- **모바일 최적화 성능**: 첫 화면 로딩 2초 이내.

### 5.2 보안 및 데이터 관리
- **개인정보 비식별화**: 이력서 내 전화번호, 주소 등 민감 정보 마스킹 처리 후 벡터화.
- **환경변수 검증**: zod로 모든 환경변수 런타임 검증 (미설정 시 즉시 에러).
- **라우트 보호**: Next.js Middleware에서 세션 쿠키 갱신 + `/dashboard` 인증 게이트.
- **API 어뷰징 방지**: `/api/chat` IP 기반 rate limiting (1분 20회). Visitor는 비인증 허용이므로 서버 레벨에서 제한.
- **서버 액션 인증**: dashboard Server Actions는 Supabase session 검증 후 실행. adminClient는 RLS 우회이므로 액션 내부에서 반드시 호출자 인증 확인.

---

## 6. UI/UX 가이드라인 (Mobile-First)

- **Visitor 페이지**: 상단 60% 프로필 사진 + 이름/직책, 하단 채팅 UI + 바텀 시트 추천 질문.
- **상호작용**: 버튼 클릭만으로도 전체 커리어 스토리 파악 가능한 **'가이드 대화 방식'**.
- **Interviewer 설정**: 우상단 설정 아이콘으로 접근.
- **테마 커스터마이징** (MVP 이후):
  - 테크: 다크모드 기반 고대비 디자인
  - 크리에이티브: 컬러풀한 그라데이션
  - 비즈니스: 미니멀 디자인

---

## 7. Phase 10: 예상 Q&A (Pinned FAQ)

### 개요
Owner가 예상 질문 목록을 미리 등록하고 AI 초안 답변을 확인/수정한다. 방문자 채팅 시 해당 Q&A를 RAG보다 우선 활용한다.

### 요구사항

| ID | 요구사항 | 세부 내용 |
|:---|:---|:---|
| **F-501** | 예상 Q&A 온보딩 | 프로필 최초 저장 후 `/dashboard?tab=qa` 탭으로 이동. 질문 입력 → AI 답변 초안 생성 → 텍스트 편집 → 저장. |
| **F-502** | AI 초안 생성 | `POST /api/pinned-qa/generate` — 질문 + 페르소나 프로필을 LLM에 전달해 답변 초안 생성. |
| **F-503** | 텍스트 직접 편집 | 편집 버튼 없이 textarea가 항상 활성화된 인라인 편집. 변경 시 저장 버튼 노출. |
| **F-504** | 대시보드 탭 통합 | 대시보드에 "예상 Q&A" 탭 추가 — 탭 순서: 설정 / 예상 Q&A / 대화 히스토리. |
| **F-505** | 채팅 우선순위 | `retrieval_node`에서 해당 사용자의 pinned_qa를 시스템 프롬프트에 포함. LLM이 관련 질문을 사전 답변 기반으로 우선 응답. |
| **F-506** | AI 질문 자동 추천 | `POST /api/pinned-qa/suggest` — 업로드된 문서 + 프로필 분석 후 면접 예상 질문 5개와 답변을 2-step LLM으로 자동 생성. |
| **F-507** | 포트폴리오 인용 UI | 답변 작성 시 업로드된 문서 목록을 펼쳐 보고 "인용" 버튼으로 `[출처: 제목]` 텍스트를 커서 위치에 삽입. |

### DB 스키마 추가

```sql
-- 예상 Q&A
pinned_qa (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES users(id) ON DELETE CASCADE,
  question text NOT NULL,
  answer text NOT NULL,
  display_order int DEFAULT 0,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
)
```

---

## 8. MVP 이후 고려 기능 (현재 범위 외)

| ID | 기능 | 설명 |
|:---|:---|:---|
| F-401 | 에이전틱 액션 (Meeting) | Google/Outlook 캘린더 연동 미팅 예약 |
| F-402 | 실시간 다국어 대응 | 브라우저 언어 감지 자동 번역 |
| F-403 | 방문자 인사이트 분석 | 질문 패턴 분석, 주간 리포트 |
| F-404 | AI 아바타 | HeyGen/D-ID 실시간 Lip-sync 아바타 |
| F-405 | 테마 커스터마이징 | 전문 분야별 UI 테마 |
