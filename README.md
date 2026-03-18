# Career Manager — LLM Multi-Agent System

Claude Code 위에서 동작하는 커리어 관리 멀티에이전트 시스템.
사용자 요청을 오케스트레이터가 판단하고 적절한 subagent를 호출하여 이력서·자소서·면접 준비 산출물을 자동 생성한다.

---

## 시스템 개요

### 설계 철학

- **오케스트레이터 패턴**: `CLAUD.md`가 메인 세션의 context로 로드되어 판단 트리 역할을 수행. 사용자 요청을 분류하고 적절한 agent를 호출한다.
- **Self-contained Agent Prompt**: 각 agent 파일은 모든 규칙을 임베딩한 완전한 명세. 별도 rules 파일 없이 단독 실행 가능.
- **데이터 분리**: 원본 경험 데이터는 절대 수정하지 않음. Agent는 원본 기반으로 파생 산출물만 생성.
- **병렬/순차 혼합 오케스트레이션**: 독립 작업(JD 분석 + 회사 분석)은 Claude의 Agent tool로 병렬 실행, 의존 관계가 있는 작업은 순차 강제.
- **자동 품질 검증**: 자소서 작성 완료 시 `cover_letter_evaluator`가 항상 자동 호출됨. 탈락 패턴 분석 기반 8대 항목 평가.

---

## 에이전트 구조

```
CLAUD.md (Orchestrator)
│
├── data_foundation          — 경험 자산화 + JD 수집·분석
│   ├── mode: curate         — 경험 파일 → asset 변환
│   ├── mode: analyze_jds    — JD 수집·분석·갭 도출
│   └── mode: full           — curate + analyze_jds
│
├── strategy_positioning     — 포지셔닝 전략 + 회사 분석 + 갭 코칭
│   ├── mode: universal      — 범용 커리어 포지셔닝
│   ├── mode: company_full   — 회사 분석 + 포지셔닝 + 갭 코칭
│   ├── mode: gap_coaching   — 스킬 갭 → 결과물 기반 액션 플랜
│   └── mode: interview_positioning — 면접용 포지셔닝 재조정
│
├── document_writer          — 경력기술서 + 자소서 생성
│   ├── mode: base_resume    — 범용 경력기술서
│   ├── mode: custom_resume  — 회사 맞춤 경력기술서
│   ├── mode: cover_letter   — 맞춤 자소서
│   └── mode: full_application — 경력기술서 + 자소서 통합
│
├── cover_letter_evaluator   — 자소서 탈락 원인 분석·등급 평가
│   └── (document_writer 내 자동 호출, 독립 실행도 가능)
│
└── interview_prep           — 면접 Q&A + 압박 질문 시뮬레이션
```

### 에이전트 의존성 맵

```
기본 체인 (순차 의존):
  data_foundation → strategy_positioning → document_writer → cover_letter_evaluator

병렬 가능 조합:
  data_foundation[analyze_jds] ──┐
                                  ├─ 완료 후 → document_writer[full_application]
  strategy_positioning[company_full] ─┘

순차 강제:
  document_writer[resume] → document_writer[cover_letter]
  document_writer → cover_letter_evaluator (자동)
```

---

## 주요 워크플로우

### 1. company_targeting_flow — 회사 지원 준비

```
전제조건 가드 체크
  ↓
┌─────────────────── 병렬 실행 ───────────────────┐
│ data_foundation[analyze_jds]                     │
│ strategy_positioning[company_full]                │
└───────────────────────────────────────────────────┘
  ↓ (둘 다 완료 후)
document_writer[full_application]
  → cover_letter_evaluator (자동 호출)
  ↓
dashboard_render_flow
```

### 2. project_to_resume_flow — 이력서 생성

```
data_foundation[curate]
  ↓
strategy_positioning[universal]
  ↓
document_writer[base_resume]
  ↓
dashboard_render_flow
```

### 3. jd_to_gap_analysis_flow — JD 분석

```
data_foundation[analyze_jds]
  ↓
리포트 요약 출력
  ↓ (고심각도 갭 발견 시)
strategy_positioning[gap_coaching] (선택)
```

### 4. interview_prep_flow — 면접 준비

```
strategy_positioning[interview_positioning]
  ↓
interview_prep
  ↓
dashboard_render_flow
```

### 5. dashboard_render_flow — 대시보드 갱신

```
직접 처리 (Agent 호출 없음):
  지원 현황 + 산출물 목록 + 갭 현황 + 업스킬 진행 + 액션 추천
  ↓
[CAREER_DIR]/00_Dashboard/dashboard.md 생성
```

---

## 디렉토리 구조

```
career-manager/
├── CLAUD.md                    # 오케스트레이터 명세 (판단 트리 + 워크플로우 디스패치)
├── agents/
│   ├── data_foundation.md      # 경험 자산화 + JD 분석 agent
│   ├── strategy_positioning.md # 포지셔닝 + 회사 분석 + 갭 코칭 agent
│   ├── document_writer.md      # 경력기술서 + 자소서 작성 agent
│   ├── cover_letter_evaluator.md # 자소서 평가 agent
│   └── interview_prep.md       # 면접 준비 agent
├── inputs/
│   ├── user_config.example.md       # 지역·기업형태 필터 설정 예시
│   ├── target_roles.example.md      # 타겟 직군 설정 예시
│   ├── career_sources_map.example.md # 데이터 소스 계층 예시
│   └── skill_tag_taxonomy.example.md # 역량 태그 분류 체계 예시
├── outputs/                    # agent 생성 산출물 (gitignore)
│   └── .gitkeep
└── architecture/
    └── system_design.md        # 아키텍처 상세 문서
```

---

## Setup 가이드

### 1. 사전 요구사항

- Claude Code CLI 설치
- 경험 데이터 디렉토리 준비 (외부 `Career/08_Experiences/*.md` 형태의 폴더 구조)
- 마스터 프로필 작성 (외부 `Career/templates/Career_template.md`)

> `[CAREER_DIR]`과 `[PROJECTS_DIR]`은 이 프로젝트 외부에 있는 실제 디렉토리를 가리키는 플레이스홀더입니다.
> 예: Obsidian vault 내의 `Career/`, `Projects/` 폴더

### 2. 설정 파일 작성

`inputs/` 폴더의 `.example.md` 파일을 복사하여 개인 설정 파일 생성:

```bash
cp inputs/user_config.example.md inputs/user_config.md
cp inputs/target_roles.example.md inputs/target_roles.md
cp inputs/skill_tag_taxonomy.example.md inputs/skill_tag_taxonomy.md
```

### 3. 경로 설정 (필수)

`inputs/user_config.md`의 경로 설정 섹션을 실제 경로로 수정:

```
career_dir: /path/to/your/Career      → 경험 데이터가 있는 Career 폴더 절대 경로
projects_dir: /path/to/your/Projects  → 작업 로그·산출물이 있는 Projects 폴더 절대 경로
```

세션 시작 시 CLAUD.md가 이 값을 읽어 `[CAREER_DIR]` / `[PROJECTS_DIR]` 플레이스홀더를 자동으로 resolve합니다.

나머지 설정도 본인 상황에 맞게 수정:
- `user_config.md`: 희망 근무 지역, 기업 형태 우선순위
- `target_roles.md`: 타겟 직군 3개
- `skill_tag_taxonomy.md`: 본인 도메인 역량 태그 분류

### 4. 실행

Claude Code를 `career-manager/` 디렉토리에서 실행:

```bash
cd career-manager
claude
```

세션 시작 시 `CLAUD.md`가 자동 로드되어 파이프라인 상태 스캔 + 대시보드 갱신이 실행된다.

---

## 트리거 치트시트

| 목적 | 입력 예시 | 실행 흐름 |
|------|----------|----------|
| **풀 지원 준비** | `[회사명] 지원 준비해줘` + JD | company_targeting_flow |
| **범용 이력서** | `경력기술서 작성해줘` | project_to_resume_flow |
| **JD 분석** | `이 공고 분석해줘: [URL]` | jd_to_gap_analysis_flow |
| **면접 준비** | `[회사명] 면접 준비해줘` | interview_prep_flow |
| **주간 루틴** | `이번 주 뭐 할까?` | weekly_operating_cycle |
| **대시보드 갱신** | `대시보드 갱신` | dashboard_render_flow |
| **경험 정리** | `경험 정리해줘` | data_foundation[curate] |
| **회사 분석만** | `[회사명] 회사 분석해줘` | strategy_positioning[company_full] |
| **스킬 갭** | `뭘 공부해야 할까?` | strategy_positioning[gap_coaching] |
| **버전 정리** | `/cleanup` | 유틸리티 |

---

## 설계 상세

→ [architecture/system_design.md](architecture/system_design.md)
