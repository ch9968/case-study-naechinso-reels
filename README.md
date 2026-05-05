# case-study-naechinso-reels

> Engineering case study: end-to-end Instagram Reels content automation pipeline for a Korean creator client (내친소).

내친소 인스타 릴스 콘텐츠를 매주 7개씩 자동 생성하는 파이프라인 case study.

원본 코드는 클라이언트 IP라 비공개 (`nextxalliance/naechinso_reels`, private). 이 repo는 의사결정 흐름과 architecture 기록용.

## What it does

콘텐츠 운영팀이 매주 콘텐츠 기획에 쓰는 시간을 줄이기 위한 6단계 자동화 시스템:

1. **아이데이션** — 이번 주 후보 주제 10개 생성 (Track A 자료 기반 + Track B 트렌드 입력)
2. **자료조사** — Tavily 검색으로 매니챗 메인 자료 작성
3. **대본 작성** — 제미나이로 릴스 대본 (Hook + Body + CTA) 생성
4. **썸네일 후킹 자막** — 시리즈별 훅 패턴 적용
5. **인스타그램 캡션** — 브랜드 톤 + KPI 목적 구조 반영
6. **Notion DB 기록** — 주간 7개 row 자동 push, 슬랙 알림

매주 사람이 하던 콘텐츠 기획 루틴을 **AI 결과 검토·수정·승인**으로 압축.

## Why this project

내친소는 Threads/Instagram에서 콘텐츠 운영 중인 크리에이터 계정. 콘텐츠 제작 파이프라인이 사람 의존도가 높아 일관성·속도가 떨어지는 문제. 클라이언트 측 요청으로 nextxalliance가 자동화 시스템을 담당.

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | TypeScript, Node 22+, ESM |
| Job orchestration | [Trigger.dev](https://trigger.dev) v4.4.5 |
| LLMs | Vertex AI Gemini (`@google/genai`), Anthropic Vertex SDK |
| Embeddings | `gemini-embedding-001` (Vertex AI) |
| Search | [Tavily](https://tavily.com) |
| Datastore | Notion API (`@notionhq/client`) |
| Cloud | Google Cloud Platform (Vertex AI, GitHub App auto-deploy) |
| Validation / config | Zod, dotenv |

## Architecture

```mermaid
flowchart TD
    Trend[매주 트렌드 키워드<br/>Track B: 사람 입력] --> Step1
    Refs[(references/<br/>9개 정적 문서<br/>brand_kpi · target_psychology<br/>performance · script_templates ...)] -.manual routing.-> Step1
    Archive[(references/05_archive.json<br/>105+ 콘텐츠 + Gemini embeddings)] -.cosine similarity<br/>≥0.85 중복.-> Step1

    Step1[Step 1: 아이데이션<br/>10 후보 → 7 selected<br/>Track A 자료 + Track B 트렌드] --> Step2[Step 2: 자료조사<br/>Tavily 검색 → 매니챗 자료]
    Step2 --> Step3[Step 3: 대본 작성<br/>Gemini, references 주입]
    Step3 --> Step4[Step 4: 썸네일 후킹<br/>훅 패턴 + 자막 분리]
    Step4 --> Step5[Step 5: 인스타 캡션<br/>브랜드 톤 + CTA]
    Step5 --> Step6[Step 6: Notion 기록<br/>주간 7개 row push]

    HardRules[HARD_RULES.md<br/>guardrail 규칙] -.각 step 주입.-> Step3
    HardRules -.-> Step4
    HardRules -.-> Step5
```

## Key Design Decisions

### 1. Hybrid retrieval architecture (정적 도메인 vs 동적 도메인)

검색해서 LLM에 컨텍스트 주입하는 방식을 **두 가지로 분리**했다.

| 도메인 | 크기 | dynamism | retrieval 방식 |
|---|---|---|---|
| `references/` (브랜드 KPI, 타겟 심리, 시리즈 템플릿 등) | 9개 정적 문서 | 거의 변하지 않음 | **Manual routing matrix** (`00_index.md`) — 어느 step에 어느 문서를 주입할지 명시 |
| `references/05_archive.json` (성과 좋았던 콘텐츠 아카이브) | 105+ 항목, 매주 추가 | 자주 변함 | **Embedding + cosine similarity** (`embed_archive.ts`, `check_duplicate.ts`) |

작은 정적 도메인에서 vector retrieval은 **noise risk + 인프라 코스트가 큼** — 9개 chunk에서 similarity로 검색하면 무관한 문서가 retrieve되어 LLM이 헷갈릴 수 있다. Manual routing이 더 정확하고 cheap. 반면 105+ 동적 archive는 매주 새 항목이 들어가고 의미적 중복 검사가 필요해서 embedding이 적절했다.

**확장 path**: 정적 도메인이 100+ docs로 커지면 vector retrieval 도입 검토.

### 2. Embedding-based duplicate detection

`check_duplicate.ts` — 신규 후보 주제가 기존 archive와 의미적으로 중복되는지 검사:

```
Verdict 분류:
  cosine similarity ≥ 0.85  →  🚫 duplicate (제외)
  0.75 ~ 0.85               →  ⚠️  review (사람 검토)
  < 0.75                    →  ✅ ok
```

- Vertex AI Gemini `gemini-embedding-001` 사용
- Cosine similarity 직접 구현 (라이브러리 의존성 추가 안 함 — 코어 함수 10줄)
- top-K nearest neighbor 반환
- CLI + module 인터페이스 둘 다 지원 (step1 task에서 import 가능)

### 3. Multi-LLM strategy

| Provider | 사용처 |
|---|---|
| Vertex AI Gemini | 대본/캡션/아이데이션 generation, embedding |
| Anthropic Vertex SDK | (보조 generation, 향후 task별 분기) |

Vertex로 통일한 이유는 GCP credentials 관리·과금·로깅을 한 곳으로 모으려고. Anthropic SDK도 Vertex 경로로 동일하게 호출.

### 4. Production reliability patterns

핸드오프 대상이라 reliability 신경 씀:

- **Idempotent**: `embed_archive.ts`는 이미 embedding된 항목은 스킵, 매 10개마다 disk flush — 중단되면 이어서 진행
- **Crash on error**: 응답 이상하거나 네트워크 실패 시 silent skip 금지, 즉시 종료
- **Validation retry + quota guard**: step별 LLM call에 retry / rate-limit 가드
- **Step idempotencyKey**: Trigger.dev sub-task에 idempotency key 부여 (재실행 시 cache hit)
- **Self-audit**: 코드 review를 Critical / High / Medium / Low로 분류해 직접 fix (commit 메시지에 `fix(critical):` `fix(high):` 등으로 트래킹)

### 5. Guardrail by document injection

`HARD_RULES.md` — 모든 step의 system prompt에 자동 주입되는 절대 규칙 (예: "직접 데이터 멘트 금지", "5/4/1 시리즈 비율 유지"). 클라이언트 브랜드 가이드를 LLM이 매번 참조하도록 강제. `04_script_templates.md`, `06_script_prompt.md`도 동일하게 step별로 주입.

### 6. WAT framework v2 (Track A/B split)

콘텐츠 아이데이션을 **두 트랙으로 분리**:

- **Track A** — 정적 자료 기반 (브랜드 KPI, 타겟 심리, 시리즈 템플릿). AI가 프레임워크에 맞춰 생성.
- **Track B** — 매주 트렌드 키워드 (사람이 손으로 입력). 시의성 보강.

10개 후보 → 5/4/1 시리즈 분포 + 중복 검사 통과한 7개 selected.

## Outcomes

- **매주 약 10시간의 운영 시간 절약** — 사람이 하던 기획 루틴이 AI 검토·승인으로 압축
- **매주 마케팅 콘텐츠 7개 자동 생성** (10 후보 → 중복 검사·시리즈 분포 통제 후 7개 selected)
- **클라이언트 핸드오프 단계** (production-ready, 운영 인수인계 진행 중)

## Lessons Learned

- **작은 도메인에 vector RAG는 over-engineering** — manual routing이 정확도 + 비용 측면 우월. dynamism이 retrieval 방식을 결정해야 한다.
- **Self-audit 단계 분류(C/H/M/L)는 효과적** — fix 우선순위가 명확해야 머지 결정이 빨라진다.
- **AI 출력에 guardrail은 system prompt 주입이 가장 robust** — 후처리 검증보다 사전 제약이 retry 비용을 줄인다.
- **Trigger.dev의 idempotencyKey + sub-task cache**는 LLM 호출 비용을 실질적으로 줄여준다 (재실행 시 step2 cache hit).

## Notes

- 원본 repo: `nextxalliance/naechinso_reels` (private, 클라이언트 IP)
- 클라이언트 동의하에 case study 공개. 개인정보(이메일/전화/매니챗 ID/GCP credentials/Notion DB ID 등)는 일체 노출하지 않음.
- 코드 스니펫은 의사결정을 보여주기 위한 발췌이며, 원본 구현은 추가 운영 자원·secret 관리·테스트 인프라를 포함한다.
