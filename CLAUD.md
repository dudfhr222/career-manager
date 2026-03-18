# CLAUDE.md — Career Manager Orchestrator

## Project Overview

Career Manager는 4개의 통합 subagent로 구성된 커리어 관리 시스템이다.
메인 세션(이 CLAUDE.md)이 오케스트레이터로서 사용자 요청을 판단하고, 적절한 agent를 호출한다.

**Language**: 모든 사용자 대면 산출물은 한국어. 기술 용어는 영어 유지.

## Architecture

```
agent_prompts/   → 4개 통합 agent의 self-contained prompt
agents/          → 기존 개별 agent 정의 (참조용 보존)
rules/           → 기존 규칙 파일 (참조용, agent prompt에 이미 임베딩)
workflows/       → 기존 workflow 정의 (참조용, 이 파일의 판단 트리로 흡수)
inputs/          → 설정 파일 (target_roles, user_config, skill_tags, sources_map)
outputs/         → 생성된 산출물
scratch/         → 임시 작업 공간
```

## Agent System (5 Agents)

> **주의**: 자소서(cover_letter / full_application) 작성 완료 시 `cover_letter_evaluator`를 반드시 호출한다. document_writer 내부에 자동 호출 로직이 포함되어 있으며, 생략 불가.

| Agent | Prompt 경로 | 역할 | 통합 대상 |
|-------|-------------|------|-----------|
| `data_foundation` | `agent_prompts/data_foundation.md` | 경험 자산화 + JD 수집·분석 | experience_curator + market_analyst |
| `strategy_positioning` | `agent_prompts/strategy_positioning.md` | 포지셔닝 + 회사 분석 + 갭 코칭 | positioning_strategist + company_analyst + skill_gap_coach |
| `document_writer` | `agent_prompts/document_writer.md` | 경력기술서 + 자소서 + 평가 트리거 | resume_writer + cover_letter_writer |
| `cover_letter_evaluator` | `agent_prompts/cover_letter_evaluator.md` | 자소서 탈락 원인 분석·등급 평가 | document_writer 내 자동 호출 |
| `interview_prep` | `agent_prompts/interview_prep.md` | 면접 Q&A + 약점 시뮬레이션 | interview_simulator |

### Agent 호출 방법

각 agent prompt 파일을 Read한 뒤, 그 내용을 Agent tool의 prompt로 전달한다.
mode와 파라미터를 prompt 상단에 명시한다.

```
Agent tool 호출 예시:
prompt: |
  [agent_prompts/data_foundation.md 내용]

  ---
  ## 실행 지시
  mode: curate
  오늘 날짜: YYYY-MM-DD
  작업 디렉토리: [PROJECT_DIR]
```

## Data Source Convention

**원본 소스 (수정 금지)**:
- `[CAREER_DIR]/templates/Career_template.md` — 마스터 프로필
- `[CAREER_DIR]/08_Experiences/*.md` — 프로젝트별 상세 경험

**보조 소스 (읽기 전용, 수정 금지)**:
- `[PROJECTS_DIR]/Logs/` — 일자별 작업 기록 (경험 근거 추출용)

**프로젝트 격리 예외**:
- `[PROJECTS_DIR]/Blog Posting/` — dashboard_render_flow에서 블로그 후보 스캔 시 읽기 허용

**파생 산출물** (agent가 원본 기반으로 생성):
- `[CAREER_DIR]/00_Dashboard/` ~ `[CAREER_DIR]/07_Assets/`
- `outputs/`

**설정 파일**:
- `inputs/target_roles.md` — 타겟 직군 3개
- `inputs/user_config.md` — 지역/기업형태 필터
- `inputs/skill_tag_taxonomy.md` — 역량 태그 분류
- `inputs/career_sources_map.md` — 데이터 소스 계층

---

## 판단 트리

세션 시작 시 또는 사용자 요청 수신 시 아래 트리를 따른다.

### 세션 초기화

```
세션 시작
  └─ [STEP 0] 경로 설정 로드
      └─ inputs/user_config.md에서 career_dir, projects_dir 읽기
         → 이후 모든 경로 참조 시 [CAREER_DIR] → career_dir 값으로, [PROJECTS_DIR] → projects_dir 값으로 resolve
         → 설정 미존재 시: "inputs/user_config.md의 career_dir / projects_dir를 먼저 설정하세요" 안내 후 중단

  └─ [STEP 1] 기본 컨텍스트 로드
      ├─ [CAREER_DIR]/templates/Career_template.md 읽기 — 프로필/강점/약점/타겟 파악
      ├─ [CAREER_DIR]/08_Experiences/ 목록 확인 — 프로젝트 현황
      ├─ [CAREER_DIR]/00_Dashboard/weekly_focus.md 읽기 — 이번 주 집중 사항
      ├─ 파생 파일(01~07) 빈 상태 감지 → bootstrap: data_foundation[full] 트리거
      └─ [PROJECTS_DIR]/Logs/ 최근 30일 기록 확인 — 신규 로그 존재 시 data_foundation[curate]에 logs_scan 플래그 전달

  └─ [STEP 2] 회사별 파이프라인 상태 스캔 ★ Skill 1
      [CAREER_DIR]/05_Applications/ Glob으로 회사명 목록 추출 후 아래 7단계 체크:
      ┌──────────────────┬─────────────────────────────────────────────────────────┬────────┐
      │ 단계             │ 판단 기준 (파일 존재 여부)                              │ 아이콘 │
      ├──────────────────┼─────────────────────────────────────────────────────────┼────────┤
      │ JD 수집          │ [CAREER_DIR]/05_Applications/[회사명]_*.md                   │ ✅/❌  │
      │ 회사 분석        │ [PROJECTS_DIR]/Career Manager/outputs/company_profiles/[회사명]_*.md                 │ ✅/❌  │
      │ 포지셔닝         │ [PROJECTS_DIR]/Career Manager/outputs/positioning/[회사명]_*.md                      │ ✅/❌  │
      │ 맞춤 이력서      │ [PROJECTS_DIR]/Career Manager/outputs/resumes/resume_[회사명]_*.md                   │ ✅/❌  │
      │ 자소서           │ [PROJECTS_DIR]/Career Manager/outputs/cover_letters/[회사명]_자소서_*.md             │ ✅/❌  │
      │ 평가             │ [PROJECTS_DIR]/Career Manager/outputs/cover_letters/eval_[회사명]_*.md               │ ✅/❌  │
      │ 면접 준비        │ [PROJECTS_DIR]/Career Manager/outputs/interview_packs/[회사명]_*.md                  │ ✅/❌  │
      └──────────────────┴─────────────────────────────────────────────────────────┴────────┘
      출력: 회사별 진행 테이블 + 각 회사의 "다음 권장 액션" 1줄 요약

  └─ [STEP 3] 마감일 긴급도 스캔 ★ Skill 2
      [CAREER_DIR]/05_Applications/*.md 각 파일에서 마감일 필드 파싱
      오늘 날짜 기준 D-day 계산:
        🔴 D-3 이내  → 긴급
        🟡 D-4~7    → 주의
        🟢 D-8 이상  → 여유
        ⚪ 미확인    → 확인 필요
      파이프라인 상태(STEP 2)와 교차 경고:
        → "마감 임박(D-N)인데 [단계] 미완성" 형태로 복합 경고 출력
        → 경고가 없으면 출력 생략

  └─ [STEP 4] 산출물 버전 중복 감지 (경량) ★ Skill 4 경량
      [PROJECTS_DIR]/Career Manager/outputs/ 하위 디렉토리 스캔 — 동일 회사에 2개 이상 버전 파일 발견 시
      → "[회사명] [유형]에 N개 버전 존재 — 세션 중 /cleanup 으로 정리 가능" 안내

  └─ [STEP 5] 세션 시작 요약 출력
      회사별 파이프라인 상태 테이블 (STEP 2)
      마감 긴급도 테이블 (STEP 3, 건이 있을 경우)
      복합 경고 목록 (STEP 3)
      버전 중복 안내 (STEP 4, 해당 시)
      → 이후 dashboard_render_flow 실행 — 대시보드 최신화
```

### 요청 분류

```
사용자 요청 수신
  │
  ├─ [주간 루틴]
  │   └─ "이번 주 뭐 할까?" / "주간 리뷰" → 직접 처리 (상태 조회 로직)
  │
  ├─ [분석 계열]
  │   ├─ "JD 분석해줘" / JD 첨부 → data_foundation[analyze_jds]
  │   ├─ "채용 트렌드" / "시장 분석" → data_foundation[analyze_jds]
  │   └─ "스킬 갭" / "뭘 공부해야" → strategy_positioning[gap_coaching]
  │
  ├─ [산출물 계열]
  │   ├─ "이력서 써줘" / "경력기술서" → project_to_resume_flow
  │   ├─ "자소서 써줘" / "자기소개서" → document_writer[cover_letter]
  │   ├─ "경험 정리" / "asset 생성" → data_foundation[curate]
  │   ├─ "포트폴리오 정리" → document_writer[base_resume] (전제: asset 3개+)
  │   ├─ "로그 반영" / "최근 작업 반영" → data_foundation[curate] (logs_scan: true)
  │   └─ "최신화해줘" / "업데이트" → 갱신 로직 (아래 참조)
  │
  ├─ [지원 계열]
  │   ├─ "[회사명] 지원 준비" → company_targeting_flow
  │   ├─ "[회사명] 회사 분석" → strategy_positioning[company_full] (포지셔닝 생략 가능)
  │   ├─ "포지셔닝" / "어필 포인트" → strategy_positioning[universal 또는 company_full]
  │   └─ "[A] vs [B] 비교" → data_foundation[analyze_jds] + strategy_positioning 병렬
  │
  ├─ [면접 계열]
  │   ├─ "면접 준비" / 면접 일정 → interview_prep_flow
  │   └─ "모의 면접" / "예상 질문" → interview_prep (standalone)
  │
  ├─ [상태 확인]
  │   ├─ "진행상황" / "현재 상태" → 직접 처리 (상태 조회 로직)
  │   └─ "대시보드" / "대시보드 갱신" / "현황판" → dashboard_render_flow
  │
  └─ [미인식]
      └─ 유사 의도 추론 → 가능 작업 목록 제시
```

---

## Workflow 디스패치 패턴

### 에이전트 의존성 맵 ★ Skill 3

```
기본 체인 (순차 의존):
  data_foundation → strategy_positioning → document_writer → cover_letter_evaluator
                                                           ↗ (자동 호출)
  document_writer 내부: resume → cover_letter (resume 먼저, 순차)
```

**병렬 가능 조합** — 아래 조합은 Agent tool 동시 호출로 처리한다:

| 조합 | 조건 | 비고 |
|------|------|------|
| `data_foundation[analyze_jds]` + `strategy_positioning[company_full]` | JD와 회사 정보가 각각 독립적으로 존재 | `company_targeting_flow`에서 기본 적용 |
| 복수 회사 `data_foundation[analyze_jds]` | 회사 A, B의 JD가 각각 존재 | 별개 Agent 인스턴스로 동시 호출 |
| 복수 회사 `strategy_positioning[company_full]` | data_foundation 완료 후 각 회사 독립 | 완료 확인 후 병렬 호출 |
| `strategy_positioning[interview_positioning]` + 기존 `interview_prep` 리뷰 | 면접팩 갱신 시 | 신규 포지셔닝 + 기존 팩 검토 병렬 |

**순차 강제 조합** (병렬 불가):

| 조합 | 이유 |
|------|------|
| `data_foundation` → `strategy_positioning` | positioning이 JD 분석 결과에 의존 |
| `strategy_positioning` → `document_writer` | 문서 작성이 포지셔닝 시트에 의존 |
| `document_writer[resume]` → `document_writer[cover_letter]` | 자소서가 이력서 내용 참조 |
| `document_writer` → `cover_letter_evaluator` | 평가 대상 자소서 필요 |

---

### 1. project_to_resume_flow (이력서 생성)

```
전제조건 가드 체크
  ↓
data_foundation[curate]
  ↓
strategy_positioning[universal]
  ↓
document_writer[base_resume]
  ↓
결과 요약 출력
  ↓ (완료 후)
  dashboard_render_flow 자동 실행
```

### 2. jd_to_gap_analysis_flow (JD 분석)

```
JD 소스 확인 (파일 / 사용자 제공 / 웹검색)
  ↓
data_foundation[analyze_jds]
  ↓
리포트 요약 출력
  ↓ (고심각도 갭 발견 시)
strategy_positioning[gap_coaching] (선택)
  ↓ (완료 후)
  dashboard_render_flow 자동 실행
```

### 3. company_targeting_flow (회사 지원 준비) ★ 가장 큰 개선

```
전제조건 가드 체크
  ↓
┌─────────────────── 병렬 실행 ───────────────────┐
│ data_foundation[analyze_jds]                     │
│ (JD 미분석 시)                                    │
│                                                   │
│ strategy_positioning[company_full]                │
│ (회사 분석 + 포지셔닝 + 갭 코칭)                   │
└───────────────────────────────────────────────────┘
  ↓ (둘 다 완료 후)
document_writer[full_application]
  ↓
결과 요약 출력
  ↓ (완료 후)
  dashboard_render_flow 자동 실행
```

### 4. interview_prep_flow (면접 준비)

```
면접 정보 확인 (회사, 단계, 날짜)
  ↓
strategy_positioning[interview_positioning]
  ↓
interview_prep
  ↓
결과 요약 출력
  ↓ (완료 후)
  dashboard_render_flow 자동 실행
```

### 5. weekly_operating_cycle (주간 루틴) — Agent 호출 없음

```
직접 처리:
  ├─ [CAREER_DIR]/07_Assets/ → asset 파일 수/목록
  ├─ [CAREER_DIR]/03_Narrative/ → positioning 완성 여부
  ├─ [CAREER_DIR]/05_Applications/ → JD 수집 현황
  ├─ [PROJECTS_DIR]/Career Manager/outputs/ 하위 디렉토리별 → 산출물 존재 여부 + 최종 수정일
  └─ weekly_focus.md → 이번 주 focus 현황

출력: 상태 테이블 + 추천 다음 액션
  ↓ (완료 후)
  dashboard_render_flow 자동 실행
```

### 6. dashboard_render_flow (대시보드 생성) — Agent 호출 없음

```
직접 처리:
  ├─ [CAREER_DIR]/05_Applications/*.md → 지원 건 목록 (회사/포지션/상태/마감일 파싱)
  ├─ [PROJECTS_DIR]/Career Manager/outputs/jd_reports/*.md → JD 리포트 목록
  ├─ [PROJECTS_DIR]/Career Manager/outputs/resumes/*.md → 이력서 버전 목록
  ├─ [PROJECTS_DIR]/Career Manager/outputs/cover_letters/*.md → 자소서 + 평가 목록 (eval_ 접두사 = 평가)
  ├─ [PROJECTS_DIR]/Career Manager/outputs/positioning/*.md → 포지셔닝 문서 목록
  ├─ [CAREER_DIR]/04_Upskill/gap_analysis.md → 스킬 갭 TOP 5 추출
  ├─ [CAREER_DIR]/04_Upskill/portfolio_build_plan.md → 업스킬 항목 + 상태
  ├─ [CAREER_DIR]/04_Upskill/execution_log.md → 실행 이력
  ├─ [PROJECTS_DIR]/Naver Blog Posting/ → 블로그 후보 스캔 (프로젝트 격리 예외)
  ├─ [CAREER_DIR]/00_Dashboard/weekly_focus.md → 주간 포커스 임베딩
  └─ 액션 추천 로직 적용

출력: [CAREER_DIR]/00_Dashboard/dashboard.md 덮어쓰기 (자동 생성 문서)
```

#### 대시보드 출력 구조

```markdown
---
updated: YYYY-MM-DD
generated_by: career_manager_orchestrator
---

# Career Operating Dashboard

> [!info] 마지막 갱신: YYYY-MM-DD | 자동 생성 문서 — 직접 수정 시 다음 갱신에서 덮어씁니다

## 1. 진행 중인 지원 건
| 회사 | 포지션 | 상태 | 마감일 | 산출물 |
([CAREER_DIR]/05_Applications/*.md에서 파싱 — 기본 정보 테이블의 회사/포지션/상태/마감일 추출)
(관련 outputs/ 파일을 wiki link로 연결)

> [!warning] 마감 임박 알림 (D-7 이내 건이 있을 경우만 표시)

## 2. 부족 스킬 TOP 5
| 순위 | 스킬 영역 | 심각도 | 보완 계획 링크 |
([CAREER_DIR]/04_Upskill/gap_analysis.md에서 5개 항목 추출)

## 3. 이력서 / 자소서 버전
| 유형 | 파일 | 생성일 | 비고 |
([PROJECTS_DIR]/Career Manager/outputs/resumes/*.md, [PROJECTS_DIR]/Career Manager/outputs/cover_letters/*.md 스캔)
(eval_ 접두사 파일은 비고에 "평가 보고서" 표기)

## 4. JD 분석 리포트
| 리포트 | 날짜 | 포함 내용 |
([PROJECTS_DIR]/Career Manager/outputs/jd_reports/*.md 스캔)

## 5. 블로그 후보
| 주제 | 출처 | 상태 |
([PROJECTS_DIR]/Naver Blog Posting/ 스캔)

## 6. 업스킬링 진행
| 항목 | 우선순위 | 상태 | 다음 단계 |
(portfolio_build_plan.md + execution_log.md에서 추출)

## 7. 다음 액션 추천
> [!important] 오늘 해야 할 일
(액션 추천 우선순위 로직에 의해 생성 — 최대 5개)

## 주간 포커스
(weekly_focus.md 내용 요약 임베딩)
```

#### 액션 추천 우선순위 로직

다음 순서로 평가하여 최대 5개 액션을 추천한다:

1. **마감 긴급** — [CAREER_DIR]/05_Applications/에서 마감일 파싱, D-7 이내 → `[긴급]`, D-3 이내 → `[긴급!!!]`
2. **미완성 워크플로우** — 회사별로 산출물 존재 여부 교차 비교:
   - JD 있으나 포지셔닝 없음 → "strategy_positioning 실행 권장"
   - 포지셔닝 있으나 자소서 없음 → "document_writer[cover_letter] 실행 권장"
   - 자소서 있으나 평가(eval_) 없음 → "cover_letter_evaluator 실행 권장"
3. **산출물 노후** — outputs/ 파일 생성일이 14일+ 경과 + 활성 지원 존재 → 갱신 권장
4. **갭→액션 매핑** — gap_analysis TOP 항목 vs execution_log/portfolio_build_plan 진행 현황 비교 → 미착수 항목 표면화
5. **블로그 주기** — Naver Blog Posting에서 최근 7일 게시물 없으면 스킬갭 연계 주제 제안
6. **주간 포커스 정합** — weekly_focus 목표 vs 실제 산출물 비교 → 미달성 항목 리마인드

---

## 전제조건 가드

각 agent 호출 전 아래 조건을 Glob으로 체크한다. 미충족 시 명시된 행동을 먼저 수행.

| 호출 대상 | 전제조건 | 미충족 시 |
|---|---|---|
| document_writer (resume) | `[CAREER_DIR]/07_Assets/asset_*.md` 존재 | data_foundation[curate] 먼저 (자동) |
| strategy_positioning (회사별) | JD report 또는 JD 파일 존재 | data_foundation[analyze_jds] 먼저 (자동) |
| interview_prep | positioning sheet 존재 | strategy_positioning 먼저 (자동) |
| strategy_positioning (gap) | market report 또는 positioning gaps 존재 | data_foundation[analyze_jds] 먼저 (자동) |
| document_writer (custom) | positioning sheet 존재 | strategy_positioning 먼저 (자동) |
| document_writer (cover_letter) | positioning sheet + 자소서 항목 제공 | strategy_positioning + 항목 요청 |
| strategy_positioning (company) | 회사명 명시 | "어떤 회사를 분석할까요?" 사용자 확인 |
| data_foundation (analyze) | JD 데이터 존재 | "JD 제공 or 웹 검색?" 사용자 확인 |
| 모든 agent | Career_template.md 비어있음 | "Career_template.md 작성 필요" 안내 후 중단 |

### 부분 진입

워크플로우의 특정 단계만 실행 가능. 단, 전제조건 가드는 반드시 통과.
```
"[특정 단계]만 해줘" → 해당 agent 식별 → 가드 체크 → 실행
```

---

## 갱신 로직

```
"최신화해줘" / "다시 만들어줘" / "업데이트해줘"
  └─ 대상 식별
      ├─ asset → data_foundation[curate]
      ├─ 이력서 → document_writer[base_resume 또는 custom_resume]
      ├─ 자소서 → document_writer[cover_letter]
      ├─ positioning → strategy_positioning[universal 또는 company_full]
      ├─ JD report → data_foundation[analyze_jds]
      ├─ 면접팩 → interview_prep
      └─ "전부" / 미지정 → 의존 체인 순서:
          data_foundation[full] → strategy_positioning[universal] → document_writer[base_resume]
```

- 원본(Career_template.md, 08_Experiences/) 변경 없으면 → "원본 변경 없음, 그래도 갱신할까?" 확인
- 날짜 태그로 기존 파일 버전 구분

---

## 상태 조회 로직

Agent 호출 없이 직접 처리:

```
스캔 대상:
  ├─ [CAREER_DIR]/07_Assets/ → asset 파일 수/목록
  ├─ [CAREER_DIR]/03_Narrative/ → positioning 완성 여부
  ├─ [CAREER_DIR]/05_Applications/ → JD 수집 현황
  ├─ [PROJECTS_DIR]/Career Manager/outputs/ 하위 디렉토리별 → 산출물 존재 여부 + 최종 수정일
  └─ [CAREER_DIR]/00_Dashboard/weekly_focus.md → 이번 주 focus

출력:
  | 영역 | 상태 | 최종 갱신일 | 비고 |

미완성 영역 발견 시 → 해당 workflow/agent 실행 제안
```

---

## 에러 처리

### 전제조건 미충족 시 자동 체이닝
```
agent 호출 요청 → 가드 체크
  ├─ 충족 → 정상 실행
  └─ 미충족
      ├─ 자동 해결 가능 → 선행 agent 자동 실행 후 원래 요청 이어서 실행
      └─ 자동 해결 불가 → 사용자에게 필요 조건 안내 + 선택지 제시
```

### 인식 불가 요청
```
매칭 안 됨
  ├─ 유사 분기 발견 → "혹시 [X] 작업을 원하시나요?"
  └─ 유사 없음 → 가능 작업 목록 제시:
      1. 주간 계획 수립
      2. JD 분석 및 갭 분석
      3. 이력서/경력기술서 작성
      4. 회사별 지원 전략
      5. 면접 준비
      6. 경험 정리 / 포지셔닝 / 스킬 갭 코칭
      7. 진행상황 확인
```

### 실행 중 실패
```
agent 단계 실패
  ├─ 입력 데이터 부족 → 데이터 확보 방법 안내
  ├─ 출력 품질 미달 → 해당 agent 재실행 1회
  │   └─ 재실행 후에도 미달 → 사용자 수동 검토 요청 + 위반 내역
  └─ 의존 산출물 손상 → 의존 agent부터 재실행
  └─ 실패 지점 기록: weekly_focus.md에 "[중단] [workflow명] Step N - 사유: ..."
```

---

## 산출물 신선도 관리

주간 루틴 시 첫 단계로 체크:

**갱신 트리거**:
- 원본 소스 수정일 > 파생 산출물 생성일
- 산출물 생성 후 2주 경과 + 활성 지원 진행 중
- 새 JD 추가 시 기존 JD report 갱신 필요

**갱신 우선순위**: asset → positioning → resume → interview_pack (의존 체인 순)

---

## 유틸리티: 산출물 버전 정리 ★ Skill 4

트리거: 사용자가 "정리해줘" / "버전 정리" / `/cleanup` 입력 시, 또는 세션 시작 STEP 4에서 중복 감지 시.

```
[실행 절차]
1. [PROJECTS_DIR]/Career Manager/outputs/ 하위 전체 디렉토리 Glob 스캔:
   company_profiles/, positioning/, resumes/, cover_letters/, interview_packs/, jd_reports/

2. 각 디렉토리별 파일 목록에서 회사명 기준 그룹핑

3. 동일 회사 그룹에 복수 파일 있으면:
   ├─ 파일명 내 날짜 패턴(YYYYMMDD 또는 YYYY-MM-DD) 추출
   ├─ 날짜 없으면 파일 수정일 비교
   └─ 최신 파일 식별 + 이전 버전 목록화

4. 출력 예시:
   [PROJECTS_DIR]/Career Manager/outputs/resumes/
     최신: resume_[Company]_20260315.md ✅
     이전: resume_[Company]_20260301.md, resume_[Company]_20260210.md

5. 사용자에게 제안:
   "이전 버전 N개를 [PROJECTS_DIR]/Career Manager/outputs/archive/ 로 이동할까요? (자동 삭제 아님)"

6. 사용자 승인 후에만 파일 이동 실행
   → 자동 삭제 절대 금지
   → 아카이브 경로: [PROJECTS_DIR]/Career Manager/outputs/archive/[원본디렉토리명]/
```

---

## 호출하지 않는 상황

- 단순 정보 조회 ([CAREER_DIR]/ 파일 직접 참조)
- 이미 완성된 산출물 재활용 (outputs/ 기존 파일 안내)
- 주간 상태 확인 (직접 스캔)
